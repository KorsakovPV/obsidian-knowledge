---
project: comet-backend
created: 2026-06-25
updated: 2026-08-05
tags: [project, architecture, fastapi]
---

# Architecture

#project

Архитектура [[Overview|comet-backend]]. Классическое слоёное FastAPI-приложение:
**API → Services → CRUD → Models (БД)**, со схемами Pydantic для валидации на границах.

## Слои и структура `app/`

```
app/
├── main.py          # сборка FastAPI: middleware, CORS, Sentry, OpenAPI, lifespan
├── core/            # config.py (pydantic-settings), log_settings.py
├── api/             # HTTP-слой: роутеры по версиям и доменам (см. [[API]])
│   ├── api_router.py       # /api + /healthcheck
│   ├── v1/v1_router.py     # сборка всех роутеров /v1
│   ├── dependencies/       # permissions.py — FastAPI-зависимости проверки прав
│   ├── docs/               # теги, примеры запросов/ответов для OpenAPI
│   └── exception_handlers.py
├── middleware/      # auth_middleware.py, skip_paths.py
├── services/        # бизнес-логика и внешние интеграции
├── cruds/           # доступ к данным (SQLAlchemy)
├── models/          # ORM-модели (см. [[Domain Model]])
├── schemas/         # Pydantic-схемы запросов/ответов
├── db/              # db.py — engine и сессии
├── helpers/         # constants.py, errors.py, http_utils.py
└── templates/       # email-шаблоны (Jinja2)
```

### Поток запроса

1. **Middleware** (`AuthMiddleware`) проверяет JWT через `AuthService`/Keycloak,
   кладёт пользователя в `request.scope["user"]`. Публичные пути пропускаются по
   `middleware/skip_paths.py:is_skipped`. Затем по `ad_login` подтягивает
   LKM-пользователя и его эффективные права и кладёт в `request.state`
   (`lkm_user`, `lkm_role`, `lkm_permissions`, `lkm_is_permissions_extended`); если
   записи в `lkm_users` нет — 403. При обновлении access-токена он возвращается
   клиенту в cookie. Подробности проверки прав — в разделе «Права доступа».
2. **Роутер** (`app/api/v1/...`) валидирует вход Pydantic-схемой, проверяет права
   через `Depends(require_permission(...))` и вызывает сервис через
   `Depends(get_*_service)`.
3. **Сервис** (`app/services/...`) выполняет бизнес-логику и обращается к CRUD
   и/или внешним системам.
4. **CRUD** (`app/cruds/...`) работает с БД через async SQLAlchemy-сессию.

## Конфигурация

`app/core/config.py` — набор классов на `pydantic-settings`, каждый со своим
`env_prefix`, значения берутся из `.env`:

- `AppSettings` (`APP_`), `DatabaseSettings` (`DB_`), `RedisSettings` (`REDIS_`).
- `KeycloakSettings` (`KEYCLOAK_`) — аутентификация.
- `ClassificatorSettings` (`CLASSIFICATOR_`), `CustomersSettings` (`CUSTOMERS_`),
  `OrderProcessingSettings` (`Order_Processing_`), `Bitrix24Settings` (`BITRIX_`) — интеграции.
- `S3Settings` (`S3_`), `SMTPSettings` (`SMTP_`), `IMAPSettings` (`IMAP_`).
- `ApprovalWorkflowSettings` (`APPROVAL_WORKFLOW_`) — Fernet-ключ шифрования outbox,
  TTL stage token (14 дней), SLA по stage code (3 дня), размер пачки и backoff worker-а.
  Некорректный ключ отклоняется при загрузке настроек, `ensure_ready()` не даёт
  запустить v2 без него.
- Флаги: `APP_DEBUG` + `APP_DEBUG_EMAIL_REDIRECT` — вместе (`AppSettings.
  redirects_email_to_initiator`) включают переадресацию писем согласования инициатору и
  послабление авторизации решений на тестовом контуре (см. [[Approval Debug Mode]]);
  на прод- и stage-контурах `APP_DEBUG_EMAIL_REDIRECT` выставлен `False` явно;
  `ORDER_PROCESSING_SEND_EXTERNAL_CONTEXT` — трассировка approval/offer в payload ОП;
  `ORDER_PROCESSING_STATUS_BATCH_ENABLED` — батчевое чтение статусов заказов через
  фильтр `id__in` вместо запроса на заказ (включается стендом, когда фильтр доехал
  до ОП). Рядом — таймаут чтения статусов, размер батча, лимит параллельности и
  пороги предохранителя, см. [[Offer Order Status]].
  Флага `APP_APPROVAL_WORKFLOW_V2_ENABLED` больше нет: staged workflow —
  единственный, включать его нечем.
- Таймзона по умолчанию — `Europe/Moscow`.

## Аутентификация

- JWT (Bearer) поверх **Keycloak**. Логика в `app/services/auth.py`
  (`AuthService`, `ApiAuthUserKeycloak`) и `app/services/keycloak.py`.
- `AuthMiddleware` применяется ко всем путям, кроме «skipped» (публичные ручки —
  например, callbacks/health). OpenAPI помечает защищённые пути «замком» через
  кастомный `custom_openapi()` в `main.py`.
- Ролевая модель ЛКМ (LKM): роли `manager / presale / sales_lead / sales_director / finance_director / product_owner / lawyer / admin`,
  таблицы `lkm_users`, `lkm_roles`, `lkm_permissions` и связки. `/auth/user`
  автоматически создаёт пользователя с ролью `manager` при первом входе. Для
  остальных защищённых ручек middleware кладёт LKM-пользователя и effective
  permissions в request context; если записи в `lkm_users` нет — возвращается 403.

## Права доступа (permissions)

Разделение ответственности: **middleware загружает** права, **ручка проверяет**.

- **Загрузка** (`app/middleware/auth_middleware.py`): на каждый запрос
  `AuthMiddleware` через `lkm_user_service.get_request_lkm_data(ad_login)` считает
  эффективный набор прав (**пермиссии роли ∪ персональные пермиссии**, UNION-запрос)
  и кладёт в `request.state.lkm_permissions` (+ `lkm_role`, `lkm_user`,
  `lkm_is_permissions_extended`).
- **Проверка** (`app/api/dependencies/permissions.py`) — FastAPI-зависимости,
  навешиваемые на ручки через `Depends(...)`:
  - `require_permission(Permission.X)` — нужен конкретный пермиссен, иначе
    `LkmForbiddenError` (403);
  - `require_any_permission(Permission.X, Y, ...)` — достаточно любого из списка;
  - `require_admin` — только роль `UserRole.ADMIN` (временный MVP-механизм для
    справочников/внутренних ручек, помечен TODO — исключение из принципа «проверять
    пермиссии, а не роль»);
  - `get_lkm_permissions(request)` — прочитать набор без проверки (решение в теле).
- **Зона ответственности**: персональное назначение может быть ограничено продуктом
  (`lkm_user_permissions.scope_product_id`). Это влияет на выбор согласующих: групповая
  стадия уходит holders без scope и holders нужных продуктов.
- **Каталог**: `Permission(StrEnum)` (24 пермиссии: 22 из БТ02 плюс
  `skip_kp_approval_stage` и `reassign_kp_approval_stage`) и `UserRole(StrEnum)`
  (`app/models/lkm_permission.py`, `app/models/lkm_user.py`). Пермиссии и роли, их
  наборы (`lkm_role_permissions`) и связки сейчас **засеваются миграцией**
  (`migrations/versions/..._add_lkm_permissions.py`).
- **Действия сделок/офферов**: доступность конкретных действий (edit/delete/…)
  дополнительно вычисляется resolver'ами `deal_actions` / `offer_actions` с учётом
  состояния согласования — см. раздел «Согласование» и [[Offer Actions Rules]].

Наборы прав ролей сужены миграцией `2026_08_05_..._narrow_approval_roles.py`:
постадийная `approve_kp_*` остаётся ровно у своей роли (`approve_kp_presale` →
`presale`, `approve_kp_lead` → `sales_lead`, `approve_kp_product_owner` →
`product_owner`). До этого `approve_kp_presale` входила в наборы пяти ролей, и письмо
групповой стадии уходило десятку адресатов. Единоличные `approve_kp_sales_director`,
`approve_kp_finance_director`, `approve_kp_lawyer` в наборы ролей не входят вовсе —
только персональные назначения.

> [!note] Код vs. БТ02
> Реализация уже содержит 8 ролей и 22 пермиссии БТ02, а сид БТ02 применён через
> миграцию `2026_07_21_1200-f1a2b3c4d5e6_bt02_roles_permissions.py`. Оставшиеся
> расхождения с целевой моделью: `require_admin` проверяет роль, roles всё ещё
> enum в коде, нет контроля единоличных permissions, отключён фильтр ephemeral,
> pre-tariffs без permission guards. Подробно — [[BT02 Role Model Gaps]].

## Фоновые задачи

- В `lifespan` приложения запускается бесконечный цикл `poll_imap_mailbox()`
  (`main.py`): раз в 2 минуты `DealService.receive_mail()` забирает письма-ответы
  согласования из IMAP-ящика. Ошибки логируются, цикл не падает; завершается через
  `CancelledError` при остановке приложения.
- Там же безусловно стартует `poll_approval_outbox()`:
  `ApprovalNotificationWorker.process_pending()` разбирает transactional outbox
  (`FOR UPDATE SKIP LOCKED`, retry с экспоненциальным backoff) и отправляет SMTP уже
  после commit workflow-транзакции. Пауза между итерациями — 1 с, если что-то
  обработано, иначе 10 с.

## Согласование (approvals) и «действия»

Ключевой бизнес-процесс — согласование оффера/сделки перед имплементацией:

- Целевая граница процесса: одна сделка имеет один актуальный approval, который
  рассматривает все offers совместно. Завершённые процессы сохраняются как версии;
  partial unique index допускает только один current approval. Маршрут выбирается
  по максимальной скидке среди всех цен сделки; особые условия любого offer
  добавляют mandatory stages `product_owner → lawyer`.
- Скидку считает сервер при создании и изменении оффера, а не route builder: builder
  работает под блокировкой сделки, где сетевой вызов недопустим. База — прайс
  классификатора (`tariff_element.price`), сравнивается с `price` тарифа (цена продажи);
  `partner_client_price` и `beeline_price` в расчёте не участвуют. Результат
  (`base_price`, `base_price_source`, `discount_percent`) сохраняется в тарифе и попадает
  в `subject_snapshot`. Тариф без цены продажи или без прайса отклоняет сохранение оффера
  доменной ошибкой, а не даёт тихую скидку 0.
- Стадия `product_owner` адресуется владельцам продуктов, вызвавших её включение:
  builder кладёт их в `approval_stages.scope_product_ids`, selector отбирает holders
  этих продуктов плюс holders без scope. Скидочные и единоличные стадии к продукту
  не привязаны.
- Approval фиксирует immutable `subject_snapshot` и `route_context`, поэтому email,
  история решения и создание заказов используют согласованные, а не текущие live data.
- Активная стадия формирует одно письмо со всей сделкой и всеми offers. Для group
  permission письмо доставляется всем holders, но относится к одной stage/token.
- Состав маршрута и порядок разделены; `STAGE_ORDER` позволяет перемещать `lawyer`.
  Начальный порядок ставит юриста перед sales/finance directors.
- State machine, audit, token history и notification outbox используют одну
  DB-транзакцию/`AsyncSession`; SMTP выполняется outbox worker-ом после commit.

- Модель `ApprovalModel` хранит статус
  (`draft/pending/blocked/answered/canceled/not_required`), решение (`approve/reject`),
  версию процесса и предмет согласования, инициатора и источник
  (`email` / `auto_related_deals` / `auto_rule`). Legacy scalar `reason_type`
  (`discount/product_rule`) сохранён для совместимости, полный набор причин лежит в
  `route_context`. См. [[Domain Model]].
- `approval_stages` хранит стадию процесса: stage code, position, required permission,
  optional-признак, assignee, сроки и поля skip/решения. Активная стадия в процессе
  одна — это гарантирует partial unique index.
- `DealService.request_approval()` запускает deal-level state machine и сохраняет
  инициатора процесса в `approval.requested_by_user_id`: событие `started` создаётся
  системой и actor не содержит. Инициатор берётся только из trusted principal middleware.
  Активация сохраняет stage token и transactional outbox в той же транзакции; SMTP
  выполняется отдельным worker после commit.
- `DealService.receive_mail()` для `workflow_version >= 2` находит историю stage token по SHA-256 hash, проверяет stage/deal/sender и вызывает общие `ApprovalWorkflowService.approve/reject`. Письмо без `stage_id` — ответ на legacy-рассылку: оно только помечается прочитанным (approval-level token удалён из схемы).
- Delivery email хранится в nullable `lkm_users.email`; валидный `ad_login` используется только как legacy fallback. Token TTL по умолчанию 14 дней, SLA стадии — 3 дня с отдельными настройками по stage code.
- Некорректный Fernet key отклоняется при загрузке settings; повреждённый outbox ciphertext отменяет только соответствующую delivery. Permanent ошибки входящего stage email помечаются seen, transient ошибки повторяются.
- API задач согласующего фильтрует pending stages в SQL по trusted LKM principal. API decision повторно проверяет assignee/effective permission под lock и хранит persisted idempotency result в `approval_stage_decision_requests`.
- Skip и reassign вынесены в отдельные ручки с правами `skip_kp_approval_stage` /
  `reassign_kp_approval_stage`: skip доступен только для optional stage, reassign — только
  для единоличной, каждая команда принимает idempotency key и пишет audit event.
- Каждый фактический переход пишет event в `approval_stage_events` в той же транзакции;
  история упорядочена по `sequence` внутри approval.
- Любое чтение под `SELECT ... FOR UPDATE` выполняется с `populate_existing=True`:
  иначе identity map SQLAlchemy вернёт объекты, прочитанные до блокировки, и проигравшая
  в гонке команда применит переход по устаревшему состоянию.
- Переход на staged workflow (cutover Ticket 12) **завершён**, а инструментарий удалён
  миграцией `2026_08_06_..._drop_legacy_approval_contract.py`: нет ни CLI
  `app.cli.approval_cutover`, ни `approval_cutover.py` / `approval_maintenance.py`,
  ни таблиц `approval_cutover_deals` и `approval_maintenance_state`. Исторический
  порядок операций сохранён в [[Approval Cutover Runbook]].
- Той же миграцией снят весь legacy-контракт offer-level workflow: `approval.offer_id`,
  `approval.approvers`, approval-level `email_token`/`email_token_expires_at` и
  `approval_stages.email_token`/`email_token_expires_at`. Строки старых согласований
  остались — история сделки читается через `GET /deals/{deal_id}/approvals`.
- После полного `answered/approve` `OfferImplementationService` создаёт по одному заказу
  ОП на каждый offer одобренного snapshot; ключ идемпотентности — детерминированный
  UUIDv5 от пары `(approval_id, offer_id)`, состояние хранится в `offer_implementations`.
  В момент создания заказа в `offer_order` сохраняется снимок КП-фазы оффера — первая
  точка мини-таймлайна карточки.
- Статус созданного заказа дальше живёт в ITSM и читается из ОП **на каждый запрос
  сделки**, а не хранится: заказы всей страницы забираются одним вызовом, недоступность
  ОП деградирует до сохранённого значения и прикрыта предохранителем. Подробно —
  [[Offer Order Status]].
- На контуре с переадресацией (`APP_DEBUG` и `APP_DEBUG_EMAIL_REDIRECT` вместе) письма
  согласования уходят инициатору процесса, а не реальным
  согласующим: адрес резолвится от `approval.requested_by_user_id` в момент создания
  уведомления и кладётся в его payload, потому что outbox отправляет письмо вне
  запроса. Если инициатор неизвестен или у него нет email, доставка отменяется —
  реальный согласующий письма не получает. Вместе с переадресацией снимается проверка
  прав на решение: инициатор может закрыть любую стадию своего согласования
  (`approval_debug_access.py`), иначе маршрут не пройти. Оба послабления включаются
  одним и тем же условием `redirects_email_to_initiator`, поэтому там, где письма
  уходят настоящим согласующим, инициатор без прав получает `permission_denied`.
  Токен, статус стадии,
  актуальность снапшота и идемпотентность проверяются всегда, каждое такое решение
  пишется в лог warning-ом. Детали и порядок использования — [[Approval Debug Mode]].
- TODO: добавить `offer.created_at` в будущие snapshot для полного ключа сортировки; до этого используется fallback по offer id.
- Ответы согласующих приходят по email и разбираются IMAP-поллером
  (`approval_email_composer.py`, `email_contents.py`, `email_transport.py`).
- **Доступность действий** над сделкой/оффером вычисляется паттерном resolver'ов:
  `app/services/deal_actions/` и `app/services/offer_actions/`
  (`context.py` — состояние, `resolvers.py` — правила, `manager.py` — оркестрация).
  Пример: `EditDealActionResolver` блокирует редактирование, пока согласование
  `REQUIRED`, и сбрасывает его в `DRAFT` при изменении одобренного оффера.
  Правила офферов задокументированы в [[Offer Actions Rules]].

## Внешние интеграции (`app/services/`)

| Сервис | Назначение |
|--------|-----------|
| `classifier.py` (`ClassifierService`) | Классификатор продуктов/услуг |
| `customers/` (`CustomersClientService`, `CustomersContractsService`, `CustomersStaffersService`) | Клиенты, договоры, сотрудники. Договор сделки выбирается при её создании, тип договора считается по номеру — [[Deal Contract Selection]]. Поиск договоров идёт через `DealService._list_contracts`: customers на выборку без совпадений отвечает 404, и для поиска это «ничего не найдено», а не сбой |
| `order_processing.py` (`OrderProcessing`) | Создание заказов в ОП по офферам и чтение их актуальных статусов — [[Offer Order Status]] |
| `order_status.py` (`OrderStatusService`) | Собирает статусы заказов страницы одним походом в ОП и подставляет в офферы |
| `kp_phase.py` | Вычисление КП-фазы оффера из состояния согласования сделки |
| `bitrix24.py` | Интеграция с Bitrix24 (сделки) |
| `keycloak.py` / `auth.py` | Аутентификация и токены |
| `s3.py` | Хранение вложений сделок и пре-тарифов |
| `price_manager.py`, `tariff.py`, `client_price.py` | Цены и тарифы |
| `email_transport.py`, `email_contents.py`, `approval_email_composer.py` | SMTP/IMAP, письма согласования |

## Данные и миграции

- ORM-модели — `app/models/`, общий `Base`. Связи описаны в [[Domain Model]].
- Миграции — **Alembic** (`migrations/versions/`, конфиг `alembic.ini`).
  `alembic upgrade head` для применения.

## Эксплуатация

- **Docker**: `Dockerfile` (+ `Dockerfile_base` для базового образа),
  `docker-compose.yaml` (host network, env из `.env`).
- **Kubernetes**: манифесты в `k8s/manifests/`, переменные — в `k8s/configmaps/`
  (по одной конфигмапе `comet-backend-env` на контур: `comet-backend-test`,
  `comet-backend-stage`, прод `comet-backend`, плюс варианты `*-gw`). Отладочные
  флаги согласования различаются по контурам — таблица в [[Approval Debug Mode]].
- **Разовые скрипты**: `scripts/` в образ **не попадает** (`scripts/README.md`) —
  код копируется в под руками и запускается там. Сейчас в директории:
  `backfill_deal_contracts.py` (проставляет договор легаси-сделкам с
  `contract_id IS NULL`: без флагов только считает объём, с `--apply` пишет, есть
  `--limit`, идемпотентен — [[DFDEV-2257 Deal Contract Selection]]),
  `approval_route_probe.py`, `price_filler.py`.
- **CI (GitLab)**: стадии `builds → tests → release → deploy` (`.gitlab-ci.yml`),
  публикация образа в `df-ors-registry.datafort.ru`.
- **Качество**: pre-commit (`make check`) — black (line-length 120), isort, flake8,
  mypy, bandit; тесты — pytest (+asyncio, xdist, cov).
- **Тесты**: unit-набор запускается без внешних зависимостей, интеграционные тесты
  workflow согласования (`tests/integration/`) помечены маркером `integration`,
  поднимают схему Alembic-миграциями на PostgreSQL и очищают её между тестами.
  Без базы — `pytest -m "not integration"`. Набор делит одну тестовую БД, поэтому
  запускается последовательно; CI прогоняет unit с `-n auto`, интеграционные — отдельным
  шагом против сервиса `postgres`.
