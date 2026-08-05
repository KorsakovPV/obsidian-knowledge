---
project: customers
created: 2026-08-05
updated: 2026-08-05
tags: [project, architecture]
---

# Architecture

Точка входа — [[Overview]]. Здесь — структура приложения внутри `app/`.

## Точка входа

`app/main.py` создаёт `FastAPI` и подключает:

- `api_router` (`api/api_router.py`, префикс `/api`) — включает `v1_router`
  и служебный `GET /api/healthcheck` (пишет тестовую ошибку в Sentry).
- Теги Swagger — `api/docs/tags.py` (`ApiTags`, порядок задаётся списком `api_tags`),
  сам Swagger отдаётся на `/` (`docs_url='/'`).
- Зависимость `fastapi_context` — кладёт `X-Request-ID` в `starlette_context`.
- Sentry: `SentryAsgiMiddleware` + middleware, проставляющий тег `request_id`.

## Слои приложения

```
app/
├── main.py            ← точка входа FastAPI
├── api/               ← роутеры v1 по доменам + docs (теги, примеры ответов)
├── services/          ← бизнес-логика, репозитории БД и Redis, интеграции
├── adapters/          ← синхронизация с внешними системами (ITSM, OP, Dadata)
├── tasks/scheduled/   ← APScheduler-воркеры, запускающие адаптеры
├── models/            ← SQLAlchemy-модели (clients, auth, counters)
├── schemas/           ← Pydantic-схемы запросов/ответов
├── filters/           ← fastapi-filter фильтры для списочных ручек
├── db/                ← Base, engine, naming convention
├── core/              ← config (Settings), db/session, log_settings
└── helpers/           ← константы, валидаторы (например, InnStr)
```

Миграции — `migrations/` + `alembic.ini` (в корне проекта, не внутри `app/`).

## Модель данных (`models/clients.py`)

Базовый класс `Base` (`app/db/db.py`) — `MappedAsDataclass` + `DeclarativeBase`,
даёт всем таблицам общие поля: `id` (uuid6), `is_deleted` (мягкое удаление),
`created_by/created_at`, `updated_by/updated_at`. Там же — соглашение об именовании
индексов и констрейнтов (`ix__`, `uq__`, `fk__`, `pk__`) и логирование медленных
запросов (> 1 сек → `logger.error`).

Таблицы:

- `client` (`ClientModel`) — юрлицо: `inn`, `title`, `short_name_with_opf`, `status`,
  `client_type`, `external_id` (автоинкремент через sequence `external_id_seq`),
  `order_ids` (список БЗ из Order Processing), `ou_company_itsm_uuid`
  (метакласс ITSM `ou$company`), `is_anonymous`, `comment`, `link_wiki`.
  Уникальный индекс `uq_inn_title` по `coalesce(inn,'') + title`.
  Свойство `key_identifier` — идентификация по `inn` → `title` → `short_name_with_opf`.
- `contract` (`ContractModel`) — договор: `contract_number` (unique), `contract_date`,
  `is_partner`, самоссылка `parent_contract_id` и две связи с клиентом —
  `parent_client_id` (кто оказывает услуги) и `child_client_id` (конечный получатель).
- `staffers` (`StafferModel`) — контактные лица клиента: ФИО, должность, email
  (в ITSM **не уникален** — архивный и живой стафферы могут делить почту),
  телефоны, `is_decision_maker` (ЛПР), `is_cloud_user`,
  `ecp_itsm_uuid` (метакласс `employee$contactPerson`).
- `managers` (`ManagerModel`) — сотрудники Датафорта, привязанные к клиенту:
  ФИО, `login`, `role`, `ee_itsm_uuid` (метакласс `employee$employee`).
- `payment_credentials` (`PaymentCredentialsModel`) — банк, БИК, счета, `is_default`.
- `egrul` (`EgrulModel`) — выписка из ЕГРЮЛ (one-to-one с клиентом): ОГРН, КПП,
  ОКВЭД/ОКПО/ОКАТО и прочие коды, руководитель, адрес, `raw_response`.
- `users` (`UserModel`) — учётки для Basic-аутентификации (`username`, `hashed_password`).
- `counters` (`CountersModel`) — счётчики по префиксу (`CounterService.get_new_count_value`).

## Поиск (`services/search_service.py`)

Нечёткий поиск на `pg_trgm`:

- `FuzzySearchService` → `FuzzySearchServiceMultiColumns` — `clients_search`
  по `inn`/`title`/`short_name_with_opf` с GIN-индексом `client_trgm_idx`.
- `MaterializedSearchService` — `full_clients_search` поверх материализованного
  представления `full_clients_search_view` (только неудалённые клиенты).

> Если в БД нет функции `idx(anyarray, anyelement)`, поиск не работает — SQL для её
> создания лежит в `README.md` проекта.

## Сервисы (`app/services/`)

- `base.py` — `BaseService[_CrudType]`: общий конструктор (request + session)
  и доступ к CRUD-репозиторию.
- Доменные: `client_service.py`, `contracts_service.py`, `staffers_service.py`,
  `managers_service.py`, `payment_credentials_service.py`, `egrul_service.py`,
  `info_service.py` (паттерны номеров договоров), `counters_service.py`.
- `auth_service.py` — `AuthService`: HTTP Basic + bcrypt (см. [[API#Аутентификация]]).
- `repository_db/` — слой доступа к БД: `crud.py` (`RepositoryDB`,
  `RepositoryDBwithDelete`), `cruds.py` (`RepositoryClient`, `RepositoryEgrul`,
  `RepositoryContract`, `RepositoryPaymentCredentials`), плюс
  `staffers_crud.py`, `managers_crud.py`, `users_crud.py`.
- `repository_redis/` — `RedisHashRepository` и `ConflictsRepository`:
  хранение конфликтов синхронизации (`ConflictMessageKey/Prototype/Full`,
  `ConflictStatus`).
- `request/` — `BaseRequestService` поверх httpx + `RetryTransport` (ретраи).
- `egrul_provider/` — клиент Dadata: `EgrulDataProvider`, схемы-парсеры
  (`DadataEgrulParserBase/Explicit`) и типизированные ошибки
  (`NoDadataSuggestionsError`, `ManyDadataSuggestionsError`, `ClientDepartmentError`, …).
- `order_processing_service.py` — HTTP-клиент к Order Processing
  (`AsyncKeycloakTokenProvider` для токена, схемы заказов по договорам).
- `grafana_service.py` — отправка метрик, `raising_http_excp.py` — хелперы 4xx/5xx.

## Адаптеры (`app/adapters/`)

Общий базис — `adapters/base.py`:

- `AdapterBase` — сессия БД, репозиторий конфликтов (`ConflictsRepository`),
  тег источника действия (`ActionSourceTags`).
- `LocalRemoteMatchBase[remote, create, update]` — пара «локальная запись ↔ удалённая»
  со схемами на создание/обновление с каждой стороны; на ней строится сравнение
  и разрешение конфликтов.

Адаптеры:

- `itsm_adapter/` — `ItsmAdapter`, двусторонняя синхронизация с ITSM 365
  (`ITSM ↔ ItsmService ↔ ItsmAdapter ↔ Customers`): компании, договоры
  (`AgreementEntity`), контактные лица, сотрудники. Проставляет `*_itsm_uuid`
  в локальные записи; `schemas.py` — матчи (`ClientMatch`, `ContractMatch`,
  `StafferMatch`, `ManagerMatch`), `utils.py` — `async_timed`, backoff, обработка ошибок ITSM.
- `order_process_adapter.py` — `OrderProcessAdapter`, односторонний
  (`Order Processing → Customers`): обновляет `order_ids` и `is_deleted`,
  автоархивация клиентов без активных заказов (`ARCHIVE_THRESHOLD_WEEKS`),
  защищённые клиенты — `PROTECTED_CLIENTS_INN` / `PROTECTED_CLIENTS_ITSM_UUID`.
- `dadata_adapter.py` — `DadataAdapter`: обогащение клиентов данными ЕГРЮЛ,
  разрешение конфликтов (`INN_DUPLICATE`, `NOT_FOUND`, `MANY_MATCHES`, `FIELD_CONFLICT`).

## Фоновые задачи (`app/tasks/scheduled/`)

APScheduler (`AsyncIOScheduler`), без брокера очередей — каждый воркер это
отдельный процесс со своим env-файлом рядом с модулем (`SCHEDULERS_DIR`):

- `base.py` — `WorkerBase` + `WorkerSettingsBase` (`WORKER_NAME`, `RUNS_AT_HOUR`/
  `RUNS_AT_MINUTE`, `MAX_INSTANCES`, `MISFIRE_GRACE_TIME`, `RUN_IMMEDIATELY`);
  ошибки джоб уходят в Sentry через listener на `EVENT_JOB_ERROR`.
- `itsm_adapter.py` — `WorkerItsmAdapter`, cron `*/1` мин, `max_instances=1`.
- `order_process_adapter.py` — `WorkerOrderProcessingBase`, cron `*/3` мин.
- `dadata_adapter.py` — базовый `WorkerBase`, cron `*/3` мин.

В k8s каждому воркеру соответствует свой манифест: `worker-itsm.yaml`,
`worker-op.yaml`, `worker-dadata.yaml`.

## Конфигурация (`core/config.py`)

`Settings(BaseSettings)` читает `.env` из корня проекта. Группы:
приложение (`APP_TITLE`, `DEBUG`, `SEND_TO_GRAFANA`, `STRESS_TEST`), Postgres
(DSN собирается валидатором в `postgresql+asyncpg`), логи (`JSON_LOGS`, `SQL_LOGS`,
`CRUD_LOGS`), Redis, Dadata (`DADATA_TOKEN`, `DADATA_SECRET_KEY`), ITSM
(`ITSM_BASE_URL/LOGIN/PASSWORD/ACCESS_KEY`), `ORDER_PROCESSING_URL`, Grafana,
Keycloak (для похода в Order Processing), `PROXY`, `DEFAULT_TIMEZONE = UTC`.

Здесь же создаётся глобальный `itsm_auth: ItsmAuth` — access key берётся из
`ITSM_ACCESS_KEY` (автообновление ключа закомментировано).
