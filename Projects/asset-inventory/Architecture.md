---
project: asset-inventory
created: 2026-07-30
updated: 2026-07-30
tags: [project, architecture, kafka, postgresql]
---

# Архитектура Asset Inventory

Точка входа — [[Overview]]. Контракты сообщений — [[API]]. Стенды и БД — [[Infrastructure]].

## Потоки данных

Два независимых входящих потока сходятся в одной модели данных.

**1. Заказы.** Из системы оформления заказов приходят события по заказу. Сервис
создаёт и обновляет цепочку `Client → Contract → Project → Product → Order →
OrderTariff → OrderResource`. Отдельный поток статусов заказа обновляет состояние.

**2. Статистика потребления.** Коллекторы восьми облаков присылают фактическое
потребление. Каждое сообщение сопоставляется с существующим заказом и создаёт или
обновляет записи `AllocateResource`.

**3. Восстановление несопоставленного.** Статистика, для которой не нашёлся заказ,
складывается в `LostResource` и переигрывается позже. Для «бумажных» заказов (без
тарифов) на лету создаются временные тарифы.

> ⚠️ Путь lost-ресурсов сейчас **выключен** флагом `APP_ENABLE_LOST_RESOURCES=False`
> на обоих стендах. Он срезается в двух независимых местах: `order.py:123` (до
> `asyncio.create_task`) и `lost_resource_revivor.py:532`.

## Модули

| Путь | Назначение |
|------|------------|
| `app/main.py` | точка входа, регистрация всех Kafka-подписчиков |
| `app/services/` | бизнес-логика: заказы, аллокация ресурсов, приём статистики |
| `app/services/statistics_backfill/` | досыл исторической статистики по источникам |
| `app/cruds/` | работа с БД, у каждой модели свой CRUD с паттерном `get_or_upsert()` |
| `app/models/` | SQLAlchemy ORM (общая база: UUID pk, `created_at`, `updated_at`) |
| `app/schemas/` | Pydantic v2: `SchemaIn` для входящих сообщений, `SchemaDbIn` для записи |
| `app/handlers/` | обработчики, куда делегирует `main.py` |
| `app/managers/matcher.py` | `MatchManager` — сопоставление ресурсов коллектора с заказом |
| `app/tasks/` | фоновые задачи: ревайвор lost-ресурсов, воркер восстановления |
| `app/tasks/scheduled/` | периодические задачи на APScheduler |
| `app/reports/` | отчёты (в частности, отчёт по неопределённостям) |
| `app/cache/` | Redis-кэш, декоратор `async_cache` и класс `RedisCache` |
| `app/consumers/auth/` | авторизация Kafka (SASL OAuth Bearer, token provider) |
| `app/db/db.py` | движок и сессии async SQLAlchemy, логирование медленных SQL |
| `app/core/config.py` | вся конфигурация на `BaseSettings`, все env-переменные |
| `app/helpers/` | логирование, типы ошибок, работа с датами |

Каталог `app/api/` в репозитории **пуст** — HTTP API здесь нет. При этом на тестовом
стенде крутится под `asset-inventory-api`, манифеста которого в репозитории нет.
Это расхождение стоит держать в голове (см. [[Infrastructure]]).

## Модель данных

Заказная часть: `client`, `contract`, `project`, `product`, `order`, `order_tariff`,
`order_resource`.

Аллокации: общая `allocate_resource` плюс таблица на каждый источник —
`dfcloud_`, `beechat_`, `vcloud_`, `platformcraft_`, `veeam_backup_`, `veeam_cc_`,
`workspace_`, `exchange_allocate_resource`. Аналогичный набор для `lost_resource`.

Служебные: `projection_sync_state` (курсоры проекции в общую таблицу),
`allocate_resource_deletion`, `allocation_recovery_run` и `allocation_recovery_chunk`
(очередь восстановления), `uncertainty_event`, `dwh_deletion_outbox`.

### Версионирование ресурсов

История аллокаций **append-only**. При изменении количества текущая запись
`AllocateResource` получает `end_date`, и создаётся новая. Количество на месте
не обновляется никогда.

## Конкурентность

Горячий путь записи статистики сериализован **между подами** транзакционным
advisory-локом Postgres. Перед чтением и записью аллокаций `_sync_allocations`
вызывает `_lock_order_resources`, который берёт `pg_advisory_xact_lock(hashtext(id))`
на каждый `order_resource_id` (отсортированные, чтобы исключить дедлок), после чего
**весь read-modify-write идёт в одной транзакции**. Два обработчика, даже на разных
подах, сериализуются на этом локе.

Второй лок — `acquire_allocation_recovery_lock` по ключу
`allocation-recovery:{source}:{order_id}` — разделяет живой путь и восстановление;
его же берёт `historical_allocation_rebuilder`.

Внутрипроцессный `asyncio.Semaphore` (`_MAX_CONCURRENT_WRITE_PHASES = 10`)
дополнительно ограничивает число одновременных write-фаз на под. Отдельный
`_AsyncSharedExclusiveLock` в `app/main.py` разводит живую статистику (shared) и
backfill-обработчик `statistic-recovery` (exclusive, ставит консьюмеры на паузу) —
**этот лок только внутрипроцессный** и между репликами не синхронизирует.

Все подписчики объявлены с `max_workers=1`, поэтому внутри топика обработка
последовательная, а параллелизм есть только между топиками.

## Известные гонки

Гонки в asyncio возможны только между двумя `await` — синхронные участки атомарны
в пределах одного event loop.

1. **TOCTOU в `get_or_upsert` / `get_or_create` — РЕШЕНО.** Все CRUD используют
   `pg_insert(...).on_conflict_do_update(...)` поверх уникальных ограничений
   (миграция `e4f1b2c9d7a3`). Атомарно и безопасно между подами.

2. **`get_last → create` для аллокаций — РЕШЕНО для живого пути, ОТКРЫТО для ревайвора.**
   `_sync_allocations` берёт advisory-лок и делает весь цикл в одной транзакции.
   Но `LostResourceRevivor._process_revived_resource` делает `get_last → update/create`
   **без** этого лока и может гонять с живым обработчиком по одному `order_resource_id`.
   Сейчас замаскировано тем, что всё живёт в одном поде (`replicas: 1`).

3. **Многоподовое восстановление / `_AsyncSharedExclusiveLock` — ЧАСТИЧНО.**
   Внутрипроцессный лок паузы не синхронизирует реплики, но advisory-лок из п. 2 уже
   сериализует сами записи. Остаётся экспозиция через ревайвор и п. 4.

4. **Применение статистики не по порядку — ОТКРЫТО (латентно).** `_sync_allocations`
   применяет любое пришедшее сообщение, сравнивая только `quantity` и проставляя
   `last_collected_at`. При однопартиционных топиках и `max_workers=1` порядок
   сегодня сохраняется. При репартиционировании или `max_workers > 1` более старый
   `collect_time` сможет перезаписать более свежую запись. Нужен event-time guard —
   см. [[Scaling-phase-1]].

## Наблюдаемость

- Медленные SQL логируются в `app/db/db.py` при превышении 0.5 с
  (`slow_sqlalchemy_query`, с текстом запроса и параметрами для `EXPLAIN`).
- Обработка статистики разложена по фазам: строки `<service> timing:`
  (`order_lookup`, `collector_resources`, `order_resources`, `match`,
  `lost_resources`, `sync_allocations`, `total`) и `sync_allocations breakdown:`
  (`get_active`, `index`, `process_loop`, `bulk_update`, `bulk_create`).
- Добавлены `recovery_phase breakdown:` и `write_phase breakdown:` — они раскладывают
  промежуток между внешним и внутренним таймером, включая время COMMIT.
- Логи в JSON, уровень задаётся `APP_LOG_LEVEL` (на обоих стендах сейчас `DEBUG`).

Как читать эти логи и что с ними уже намеряно — в [[Infrastructure]].
