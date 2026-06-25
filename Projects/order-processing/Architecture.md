---
project: order-processing
created: 2026-06-25
updated: 2026-06-25
tags: [project, architecture]
---

# Architecture

Точка входа — [[Overview]]. Здесь — структура приложения внутри `app/`.

## Точка входа

`app/main.py` создаёт `CustomFastAPI` (наследник `FastAPI`) и подключает:

- `api_router` — API v1 (`api/api_v1`), legacy «ручка для БЗ».
- `api_router_v2` — API v2 (`api/api_v2`), основной набор эндпоинтов (см. [[API]]).
- `url_router` (`urls/urls.py`) — серверный рендеринг форм заказа/TCO по продуктам
  (десятки `*_order_url` / `*_tco_url`, например `cloud_compute_order_url`).
- Middleware: `ConditionalMiddleware`, `check_api_protected_paths`.

`CustomFastAPI.url_path_for_with_query_param` достраивает URL форм с query-параметрами
(`tenant_id`, `contract_number`, `contract_id`, `parent_contract_id`).

## Слои приложения

```
app/
├── main.py            ← точка входа FastAPI
├── api/               ← роутеры и эндпоинты (api_v1, api_v2)
├── urls/              ← серверные формы заказов/TCO по продуктам
├── services/          ← бизнес-логика (см. ниже)
├── models/            ← SQLAlchemy-модели (order_v2, project_v2, approval, offers, …)
├── schemas/           ← Pydantic-схемы запросов/ответов
├── adapters/          ← интеграции (ITSM, push_order)
├── tasks/             ← фоновые задачи taskiq + broker/scheduler
├── producers/         ← публикация событий (op_orders_producer)
├── core/              ← config, БД (db), логирование
├── middleware/        ← middleware
├── forms/ filters/ helpers/ templatetags/ templates/  ← рендеринг и утилиты
├── migrations/        ← Alembic-миграции (alembic.ini, test_*alembic.ini)
└── static/            ← статика (в т. ч. css для PDF)
```

## Сервисы (`app/services/`)

- `orders/` — ядро: жизненный цикл бланка заказа (см. ниже).
- `offers/` — коммерческие предложения.
- `projects/`, `order_projects/` — проекты и проекты заказов.
- `products/` — продукты, тех. параметры, тарифы.
- `contracts/`, `clients/`, `staffers/` — контракты, клиенты, стафферы (подписанты).
- `billing/` — данные для биллинга (`billing_service.py`, `bs2.py`).
- `itsm_document_v2/`, `itsm_implement_order_v2/` — имплементация заказов через ITSM.
- `actualize_order/` — актуализация заказов/контрактов.
- `reports/` — отчёты. `regions/`, `info/` — справочники/общая информация.
- `bitrix24/` — интеграция с Bitrix24. `managers/`, `squirrel/` — вспомогательные.
- `keycloak.py` — работа с Keycloak.

## Жизненный цикл заказа (`services/orders/`)

- `order_service.py` (`OrderService`) — основной сервис.
- `order_handlers_factory.py` (`OrderHandlersFactory`) — по типу заказа выбирает
  обработчик из `order_handlers/`:
  - `NewOrderHandler` — новый заказ;
  - `ConnectedOrderHandler` — подключение;
  - `TestToProdOrderHandler` — перевод test → prod;
  - `DisbOrderHandler` — расформирование (disbandment).
- `order_state_manager.py` (`OrderStateManager`) — статус заказа; мэппинг состояния
  ITSM на статус через `ORDER_STATUS_MAPPING` по ключу `(order_category, order_action_type)`.
- `order_price_manager.py`, `order_transform_manager.py`, `order_actions_manager.py`,
  `file_manager.py`, `order_pdf_service.py`, `template_service.py` — расчёт цены,
  трансформация, действия, файлы, PDF, шаблоны.
- `order_validators/` — валидация тарифов и тех. параметров:
  - `rules_manager.py` — `RuleManager` (базовый) и наследники
    (`CloudMDMAdminRuleManager`, `PostgreSQLRuleManager`, `DisableFieldRuleManager`);
    методы `validate` / `calculate` / `get_excluded_tp` / `integer_only_tariff_pks`
    для правил тех. параметров (`TechParamsRulesSchema`). Расчёты используют
    `quantity_value` (Decimal-fallback), не устаревшее `quantity`.
  - `tariffs_validator.py` — диапазон и кастомные валидаторы (через `quantity_value`).
  - `tech_params_validator.py` — `validate_integer_only_tariffs()` (перед `validate_rules()`)
    запрещает дробное количество для integer-доменных тарифов (MDM/PG).
  - `custom_tariff_validators.py`, `constants.py`, `utils.py`.
  - Дробное количество тарифа — см. [[Fractional-quantity]].
- `preapproved_order_service.py` — создание предварительно согласованного заказа
  (см. [[Preapproved-order]]).

## Модель данных (`models/order_model_v2.py`)

Основные таблицы (`Base`, SQLAlchemy):

- `order_v2` (`OrderModelV2`) — бланк заказа: продукт, регион, контракт, тарифы,
  тех. параметры, стафферы (contractor / customer / partner_customer), подписанты,
  состояние и `state_log`, ссылки на оффер (`offer`), предыдущий/следующий заказ,
  PDF, ITSM uuid, связь `approvals`.
- `project_v2` (`ProjectModelV2`), `region` (`Region`),
  `itsm_implement_order_v2` (`ItsmImplementOrderModelV2`).
- Прочие модели: `approval_models.py` (согласования), `offer_models_v2.py`,
  `billing_services_model.py`, `tariff_commercial_offer_model.py`, `helper_models.py`.

## Фоновые задачи (`app/tasks/`)

- `broker.py` — taskiq на Redis: `ListQueueBroker` (очередь `TASKS_QUEUE_NAME`),
  `TaskiqScheduler` (`RedisScheduleSource` + `LabelScheduleSource`),
  `RedisAsyncResultBackend`.
- `itsm/` — задачи интеграции с ITSM (например, `create_service_call_from_itsm`,
  `actualize_contract`).
- Ручной перезапуск упавших задач — через `LPUSH` в Redis (см. `tasks/README.md`).

## Интеграции (`adapters/`)

- `itsm_adapter.py` + `itsm_adapters/` — обмен с ITSM (op↔itsm адаптеры, схемы,
  синхронизация заказов).
- `push_order.py` — отправка заказа во внешнюю систему. `base.py`, `exceptions.py`.
- `producers/op_orders_producer.py` — публикация событий по заказам.