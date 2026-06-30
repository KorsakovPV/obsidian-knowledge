---
project: comet-backend
created: 2026-06-25
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
│   ├── api_router.py    # /api + /healthcheck
│   ├── v1/v1_router.py  # сборка всех роутеров /v1
│   ├── docs/            # теги, примеры запросов/ответов для OpenAPI
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
   `middleware/skip_paths.py:is_skipped`. При обновлении access-токена он
   возвращается клиенту в cookie.
2. **Роутер** (`app/api/v1/...`) валидирует вход Pydantic-схемой, вызывает сервис
   через `Depends(get_*_service)`.
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
- Таймзона по умолчанию — `Europe/Moscow`.

## Аутентификация

- JWT (Bearer) поверх **Keycloak**. Логика в `app/services/auth.py`
  (`AuthService`, `ApiAuthUserKeycloak`) и `app/services/keycloak.py`.
- `AuthMiddleware` применяется ко всем путям, кроме «skipped» (публичные ручки —
  например, callbacks/health). OpenAPI помечает защищённые пути «замком» через
  кастомный `custom_openapi()` в `main.py`.
- Ролевая модель ЛКМ (LKM): роли `manager / sales_lead / director / admin`,
  таблицы `lkm_users`, `lkm_roles`, `lkm_permissions` и связки. `/auth/user`
  автоматически создаёт пользователя с ролью `manager` при первом входе. Для
  остальных защищённых ручек middleware кладёт LKM-пользователя и effective
  permissions в request context; если записи в `lkm_users` нет — возвращается 403.
  Проверки действий сделок/офферов используют `deal_actions` / `offer_actions`,
  которые учитывают permissions, когда они переданы в context. Подробная
  спецификация — в `docs/lkm_role_model.md` репозитория.

## Фоновые задачи

- В `lifespan` приложения запускается бесконечный цикл `poll_imap_mailbox()`
  (`main.py`): раз в 2 минуты `DealService.receive_mail()` забирает письма-ответы
  согласования из IMAP-ящика. Ошибки логируются, цикл не падает; завершается через
  `CancelledError` при остановке приложения.

## Согласование (approvals) и «действия»

Ключевой бизнес-процесс — согласование оффера/сделки перед имплементацией:

- Модель `ApprovalModel` хранит статус (`draft/pending/answered/canceled`),
  решение (`approve/reject`), причину (`discount/product_rule`) и источник
  (`email` / `auto_related_deals` / `auto_rule`). См. [[Domain Model]].
- Ответы согласующих приходят по email и разбираются IMAP-поллером
  (`approval_email_composer.py`, `email_contents.py`, `email_transport.py`).
- **Доступность действий** над сделкой/оффером вычисляется паттерном resolver'ов:
  `app/services/deal_actions/` и `app/services/offer_actions/`
  (`context.py` — состояние, `resolvers.py` — правила, `manager.py` — оркестрация).
  Пример: `EditDealActionResolver` блокирует редактирование, пока согласование
  `REQUIRED`, и сбрасывает его в `DRAFT` при изменении одобренного оффера.
  Правила офферов задокументированы в `docs/offer_actions_rules.md`.

## Внешние интеграции (`app/services/`)

| Сервис | Назначение |
|--------|-----------|
| `classifier.py` (`ClassifierService`) | Классификатор продуктов/услуг |
| `customers/` (`CustomersClientService`, `CustomersContractsService`, `CustomersStaffersService`) | Клиенты, договоры, сотрудники |
| `order_processing.py` (`OrderProcessing`) | Создание заказов в ОП по офферам |
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
- **Kubernetes**: манифесты в `k8s/`.
- **CI (GitLab)**: стадии `builds → tests → release → deploy` (`.gitlab-ci.yml`),
  публикация образа в `df-ors-registry.datafort.ru`.
- **Качество**: pre-commit (`make check`) — black (line-length 120), isort, flake8,
  mypy, bandit; тесты — pytest (+asyncio, xdist, cov).
