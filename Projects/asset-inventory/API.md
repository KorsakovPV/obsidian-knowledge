---
project: asset-inventory
created: 2026-07-30
updated: 2026-07-30
tags: [project, api, kafka, contracts]
---

# Публичный интерфейс: Kafka

HTTP API у сервиса нет (`app/api/` пуст), поэтому публичный интерфейс — это набор
Kafka-топиков, которые он слушает. Точка входа — [[Overview]], внутреннее устройство —
[[Architecture]].

Все подписчики регистрируются в `app/main.py`, имена топиков и группа берутся из
конфигурации (`app/core/config.py`, префикс `KAFKA_`). На тестовом стенде к именам
добавляется суффикс `-test`.

## Топики

| Назначение | Переменная конфигурации | Значение на проде | Обработчик |
|---|---|---|---|
| Заказы | `KAFKA_ORDERS_TOPIC_NAME` | `orders` | `handle_order_message` |
| Статусы заказов | `KAFKA_ORDER_STATUS_TOPIC_NAME` | `order_state` | `handle_order_status_message` |
| Досыл статистики | `KAFKA_STATISTIC_RECOVERY_TOPIC_NAME` | `statistic-recovery` | `handle_statistic_recovery_message` |
| VCloud | `KAFKA_VCLOUD_TOPIC_NAME` | `statistic-vcloud` | `handle_vcloud_message` |
| DFCloud | `KAFKA_DFCLOUD_TOPIC_NAME` | `statistic-dfcloud` | `handle_dfcloud_message` |
| Platformcraft | `KAFKA_PLATFORMCRAFT_TOPIC_NAME` | `statistic-platformcraft` | `handle_platformcraft_message` |
| Veeam Backup | `KAFKA_VEEAM_BACKUP_TOPIC_NAME` | `statistic-veeam-backup` | `handle_veeam_backup_message` |
| Veeam CC | `KAFKA_VEEAM_CC_TOPIC_NAME` | `statistic-veeam-cc` | `handle_veeam_cc_message` |
| BeeChat | `KAFKA_BEECHAT_TOPIC_NAME` | `statistic-beechat` | `handle_beechat_message` |
| Exchange | `KAFKA_EXCHANGE_TOPIC_NAME` | `statistic-exchange` | `handle_exchange_message` |
| Workspace | `KAFKA_WORKSPACE_TOPIC_NAME` | `statistic-workspace` | `handle_workspace_message` |

Группа консьюмера — `KAFKA_GROUP_ID` (`ai-consumer-group`, на тесте
`ai-consumer-group-test`).

Все одиннадцать подписчиков объявлены с **`max_workers=1`**. Это значит, что внутри
топика сообщения обрабатываются строго последовательно, а параллелизм есть только
между разными топиками. От этого зависят выводы про порядок сообщений в
[[Scaling-phase-1]] и про семафоры в [[Scaling-roadmap-tasks]].

Общие параметры консьюмеров (`consumer_retry_params`): `max_poll_interval_ms=300000`,
`session_timeout_ms=10000`, `heartbeat_interval_ms=3000`.

## Схемы сообщений

Схемы Pydantic v2 лежат в `app/schemas/`, по файлу на источник: `vcloud.py`,
`dfcloud.py`, `platformcraft.py`, `veeam_backup.py`, `veeam_cc.py`, `beechat.py`,
`exchange.py`, `workspace.py`, плюс общий `base_cloud.py` и `protocols.py`.

Соглашение об именовании: `*SchemaIn` — входящее сообщение из Kafka, `*SchemaDbIn` —
то, что уходит в БД. Заказная часть описана в `order.py`, `order_resource.py`,
`tariff.py`, `client.py`, `contract.py`, `product.py`, `project.py`.

Ключевые поля статистики, на которые опирается логика:

- **идентификатор** — по нему ищется заказ (`pool_name`, `tenant_name`, `fingerprint`
  в зависимости от источника; конкретное поле задаётся `IDENTIFIER_FIELD` сервиса);
- **`collect_time`**, а также `time_start` / `time_end` — время снятия статистики,
  из них выводятся `start_date` и `last_collected_at` аллокации;
- **количество** — сравнивается с текущей аллокацией и решает, закрывать ли версию.

## Обработка сбоев

- Поток статусов заказа имеет собственный ретрай с backoff
  (`app/helpers/order_status_retry.py`), число попыток — `KAFKA_ORDER_STATUS_MAX_RETRIES`
  (сейчас 5). Метаданные повтора хранятся в самом сообщении (`_retry_count`).
- Для статистики отдельного ретрая нет: обработчики ловят исключение и логируют его,
  при этом offset уже закоммичен (`auto_commit=True`). То есть **сбой обработки
  означает потерю сообщения**. Это осознанная известная проблема, тикет
  `AI-SCALE-P0-5` в [[Scaling-roadmap-tasks]].
- Статистика, для которой не нашёлся заказ, должна была бы уходить в `LostResource`,
  но путь выключен флагом `APP_ENABLE_LOST_RESOURCES=False` (см. [[Architecture]]).

## Исходящие взаимодействия

Сервис ничего не публикует в Kafka в штатном режиме (кроме перепубликации сообщения
статуса заказа при ретрае). Наружу ходит по HTTP:

- **Classificator** (`CLASSIFICATOR_ENDPOINT`) — маппинг ресурсов на тарифы; до
  четырёх последовательных GET при промахе кэша, что заметно в латентности
  (тикет `AI-SCALE-P0-6`);
- **Hydra** (`HYDRA_URL`) — авторизация;
- **Order Processing** (`ORDER_PROCESSING_URL`) — данные по заказам;
- **Kafka SASL OAuth** — токен-провайдер в `app/consumers/auth/`.
