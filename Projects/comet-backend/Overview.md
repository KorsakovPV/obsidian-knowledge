---
project: comet-backend
created: 2026-06-25
updated: 2026-07-21
tags: [project, backend, fastapi, python]
---

# Comet-backend

#project

Бэкенд сервиса **электронных коммерческих предложений** (внутреннее название — **ETK**,
«Сервис электронных коммерческих предложений»). Менеджеры формируют коммерческое
предложение клиенту, проводят его через согласование и превращают в заказы во внешней
системе обработки заказов.

## Доменная суть

- **Сделка (deal)** — коммерческое предложение клиенту. Может содержать несколько продуктов.
- **Оффер (offer)** — составная часть сделки: `1 оффер == 1 продукт == 1 бланк заказа`.
- Согласованная сделка превращается в несколько **заказов (orders)** во внешней системе ОП
  (Order Processing).

Подробнее о сущностях и их связях — в [[Domain Model]].

## Стек

- **Python 3.13**, **FastAPI**, **Uvicorn** (ASGI).
- **SQLAlchemy 2.0** (async, `asyncpg`) + **Alembic** для миграций.
- **PostgreSQL** как основная БД, **Redis** для кэша/токенов.
- **Pydantic v2** / `pydantic-settings` для схем и конфигурации.
- **Keycloak** — аутентификация (JWT, Bearer).
- **Sentry** — мониторинг ошибок.
- Интеграции по HTTP (`httpx`): классификатор, Customers, Order Processing, Bitrix24.
- Почта: **SMTP** (`aiosmtplib`) и **IMAP** (`aioimaplib`) — для процесса согласования по email.
- Хранение файлов: **S3** (`aioboto3`).
- Управление зависимостями — **Poetry**; деплой — Docker + Kubernetes, CI в GitLab.

## Как запустить

Код проекта: `/home/pavel/PycharmProjects/comet-backend`.

```bash
poetry install
cp .env.example .env          # заполнить креды БД, Keycloak, S3, SMTP/IMAP и т.д.
alembic upgrade head          # применить миграции
python app/main.py            # uvicorn на localhost:8022
```

- Swagger UI отдаётся на корне `/` (`docs_url='/'`).
- Все ручки под префиксом `/api` (далее `/v1`), см. [[API]].
- Проверки качества: `make check` (pre-commit: black, isort, flake8, mypy, bandit).

## Карта документации

- [[Architecture]] — слои приложения, middleware, права доступа, фоновые задачи, интеграции.
- [[API]] — публичные HTTP-эндпоинты по доменам.
- [[Domain Model]] — сущности БД (deal, offer, approval, …) и их связи.

## Исследования

Перенесены из `docs/` репозитория (одностороннее зеркало, источник — код):

- [[LKM Role Model]] — ролевая модель ЛКМ (БТ02): роли, пермиссии, RBAC + персональные права.
- [[Offer Actions Rules]] — правила доступности действий над сделкой/оффером по состоянию согласования.
- [[Preapproved Order Integration]] — интеграция с Order Processing после согласования оффера (DFDEV-1908).

## Заметки на полях

- Часть ручек — временные/технические (`/deals/check_send_email`,
  `/deals/check_fetch_unseen_emails`, `start-implementation` как заглушка).
  Это видно по коду и помечено в [[API]].
