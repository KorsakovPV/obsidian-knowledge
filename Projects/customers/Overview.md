---
project: customers
created: 2026-08-05
updated: 2026-08-05
tags: [project, backend, fastapi]
---

# Customers («Карточка клиента»)

Сервис-хранилище клиентов экосистемы DataFort: юрлица, их договоры, контактные
лица (стафферы), менеджеры Датафорта, платёжные реквизиты и данные ЕГРЮЛ.
Выступает мастер-системой по клиентам для остальных сервисов и синхронизируется
с ITSM, Order Processing и Dadata.

Репозиторий: `gitlab.datafort.ru/dev/customers`.

## Стек

- **Python 3.11 + FastAPI** — HTTP API, одна версия (v1). Swagger на корне (`/`).
- **SQLAlchemy 2.0 (async, `MappedAsDataclass`)** + **Alembic** — PostgreSQL и миграции.
- **Pydantic v1** — схемы и настройки (`core/config.py`, `BaseSettings`).
- **APScheduler + Redis** — фоновые воркеры-адаптеры (см. [[Architecture#Фоновые задачи]]).
- **HTTP Basic + bcrypt** — собственная аутентификация (не Keycloak, см. [[API#Аутентификация]]).
- **ITSM 365** (`datafort_utils.utils.itsm`) — двусторонняя синхронизация клиентов,
  договоров, стафферов и менеджеров.
- **Order Processing** — источник заказов (`order_ids`) и признака архивации.
- **Dadata** — обогащение клиента данными ЕГРЮЛ.
- **PostgreSQL pg_trgm** — нечёткий полнотекстовый поиск (`services/search_service.py`).
- **Grafana**, **Sentry** — метрики и мониторинг ошибок.

## Как запустить

```bash
poetry install --with dev
pip install git+https://…@gitlab.datafort.ru/dev/datafort_utils.git
poetry shell
pre-commit install
export PYTHONPATH=$PWD
python3 app/main.py      # uvicorn на localhost:8020
```

Конфигурация — через `.env` (см. `.env.example`, класс `Settings` в `app/core/config.py`).
БД должна работать в UTC+0: `SELECT current_setting('TIMEZONE')` / `SET timezone = 'UTC'`.

Пользователя для Basic-аутентификации заводит функция `create_user`
в `app/services/auth_service.py` — интерактивного ввода нет, см. [[API#Создание пользователя]]:

```bash
export PYTHONPATH=$PWD
poetry run python3 -c "
import asyncio
from app.services.auth_service import create_user
asyncio.run(create_user(username='ЛОГИН', password='ПАРОЛЬ'))
"
```

Воркеры запускаются как отдельные процессы, каждый со своим env-файлом
в `app/tasks/scheduled/`:

```bash
python3 app/tasks/scheduled/itsm_adapter.py
python3 app/tasks/scheduled/order_process_adapter.py
python3 app/tasks/scheduled/dadata_adapter.py
```

Линтеры и форматирование — через `Makefile`:

```bash
make check     # isort --check-only, flake8, mypy, bandit
make format    # isort, black, flake8
```

Тесты — `pytest` (маркеры `e2e`, `redis`, `slow` по умолчанию отключены,
см. `[tool.pytest.ini_options]` в `pyproject.toml`).

Деплой — Kubernetes: `k8s/manifests/` (отдельные манифесты на приложение
и на каждый воркер), `k8s/configmaps/` по контурам dev/test/prod,
пайплайны в `.gitlab-ci.yml` и `k8s/pipelines/`.

## Оглавление

- [[Architecture]] — структура приложения, модель данных, адаптеры, воркеры.
- [[API]] — эндпоинты v1, поиск, фильтры, аутентификация.

## Исследования

Пока нет — заметки по конкретным задачам класть в `Research/`.
