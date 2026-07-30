---
project: asset-inventory
created: 2026-07-30
updated: 2026-07-30
tags: [project, backend, kafka, event-driven]
---

# Asset Inventory

Событийно-ориентированный микросервис учёта облачных ресурсов. Слушает Kafka,
сопоставляет статистику потребления от облачных коллекторов с заказами из системы
оформления заказов и ведёт историю выделенных ресурсов, на которой строится
начисление. Используется в экосистеме DataFort.

Собственного HTTP API у сервиса нет — это чистый consumer, публичный интерфейс
описан в [[API]].

## Стек

- **Python 3.11+** — рантайм (`pyproject.toml`; mypy настроен на 3.13).
- **FastStream + aiokafka** — Kafka-консьюмеры, точка входа `app/main.py`.
- **SQLAlchemy 2 (async)** + **Alembic** — PostgreSQL и миграции.
- **Pydantic v2** + **pydantic-settings** — схемы сообщений и конфигурация.
- **Redis** — кэш (`app/cache/`), декоратор `async_cache`.
- **APScheduler** — периодические задачи (`app/tasks/scheduled/`).
- **Sentry** — мониторинг ошибок.
- Внешние интеграции: **Classificator** (маппинг ресурсов на тарифы),
  **Hydra** (авторизация), **Order Processing** (источник заказов).

## Как запустить

```bash
poetry install --no-root
python app/main.py                 # с авторизацией Kafka
python app/main.py --without-auth  # локально / тестовый режим
```

Проверки и тесты:

```bash
make check                                              # все pre-commit хуки
python -m pytest -sv tests/                             # тесты
python -m pytest -sv --cov=app/ --cov-report=term-missing tests/
alembic upgrade head                                    # миграции
```

Системные тесты (`tests/test_system_tests.py`) требуют живых Kafka и PostgreSQL.

## Оглавление

- [[Architecture]] — потоки данных, модули, конкурентность, известные гонки.
- [[API]] — Kafka-топики, контракты сообщений, источники статистики.
- [[Infrastructure]] — стенды, pgpool, Postgres, бюджет соединений, логи.

## Исследования

Материалы из `docs/` репозитория, перенесены как есть:

- [[Scaling-roadmap]] — общая карта работ по масштабированию, фазы и гейты.
- [[Scaling-roadmap-tasks]] — декомпозиция на тикеты `AI-SCALE-*`, рабочий бэклог.
- [[Scaling-phase-1]] — фаза 1, корректность (гейт перед параллелизмом).
- [[Scaling-phase-1-RFC]] — RFC к фазе 1.
- [[Scaling-phase-2]] — фаза 2, параллелизм: партиции, реплики, бюджет соединений.
- [[Scaling-phase-3]] — фаза 3, декомпозиция сервиса и масштаб БД.

## Важно

Источник истины — код. Эти заметки описывают состояние на 2026-07-30 и могут
устареть; при расхождении доверяй коду. Актуальные рабочие инструкции для агентов
лежат в `CLAUDE.md` репозитория.
