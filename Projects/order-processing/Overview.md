---
project: order-processing
created: 2026-06-25
updated: 2026-06-25
tags: [project, backend, fastapi]
---

# Order processing

Сервис для оформления и обсчёта услуг (бланки заказа, БЗ). Позволяет в формах
ввести данные, необходимые для обсчёта услуги или подключения, выполнить расчёт
и отправить заказ во внедрение (ITSM). Используется в экосистеме DataFort.

Репозиторий: `gitlab.datafort.ru/dev/order-processing`.

## Стек

- **Python + FastAPI** — HTTP API (две версии: v1 legacy для БЗ, v2 — основная).
- **SQLAlchemy** (async + sync) + **Alembic** — БД PostgreSQL и миграции.
- **Pydantic v1** — схемы и настройки (`core/config.py`, `BaseSettings`).
- **taskiq + Redis** — фоновые задачи и планировщик (см. [[Architecture#Фоновые задачи]]).
- **Keycloak** (`datafort_utils`) — аутентификация/авторизация.
- **ITSM** — интеграция для имплементации заказов и получения статусов.
- **Bitrix24** — интеграция (сервис `services/bitrix24`).
- **wkhtmltopdf + Jinja2** — генерация PDF бланков заказа.
- **Sentry** — мониторинг ошибок.
- Внешний сервис **Comet** — источник предварительно согласованных заказов
  (см. [[Preapproved-order]]).

## Как запустить

Зависимости (см. папку `requirements/`):

```bash
pip install -r requirements/dev.txt
pre-commit install
```

Приложение — FastAPI (`app/main.py`, объект `app`). Конфигурация через env
(`env.stage_dev` и т. п.), деплой в Kubernetes (`order-processing*.yaml`,
`Dockerfile`, `docker-compose.yaml`).

Воркер и планировщик фоновых задач (taskiq) — см. `app/tasks/README.md`:

```bash
export PYTHONPATH=app
taskiq scheduler --fs-discover --tasks-pattern 'app/tasks/**/*.py' tasks.broker:scheduler
taskiq worker   --fs-discover --tasks-pattern 'app/tasks/**/*.py' tasks.broker:broker
```

Линтеры/тесты — через `Makefile` (`make run_linters`, `make check`).

## Оглавление

- [[Architecture]] — структура приложения, основные модули и их связи.
- [[API]] — версии API, эндпоинты v2, аутентификация.

## Исследования

- [[Preapproved-order]] — DFDEV-1908: создание предварительно согласованного
  заказа из внешнего сервиса (Comet).
- [[Fractional-quantity]] — поэтапная поддержка дробного количества в тарифах
  (`quantity_decimal`, Decimal-fallback, запрет дробного для MDM/PG).