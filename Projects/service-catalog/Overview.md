---
project: service-catalog
created: 2026-08-20
tags: [project, backend, django]
---

# Service catalog

Классификатор услуг DataFort. Хранит каталог услуг компании: стоимость, сроки
подключения, единицы и типы тарификации, связи между тарифами (какую услугу
нельзя подключить без другой), локации предоставления и сборку услуг в
продукты для КП. Является источником данных для соседних сервисов экосистемы —
[[Projects/order-processing/Overview|order-processing]] и comet-backend
берут из него тарифы и продукты (см. [[API#Кто потребляет API]]).

Репозиторий: `gitlab.datafort.ru/dev/service-catalog` (переменная CI `$CATALOG`).

## Стек

- **Python 3.8 + Django 3.1** поверх CMS **garpixcms 2.18** (`garpix_notify`,
  `garpix_auth`, `django-solo`) — база проекта и админка.
- **Django REST Framework 3.13** + **django-filter** + **drf-spectacular** —
  API и Swagger по пути `/api/docs/`.
- **PostgreSQL** — основная БД; **Redis** — кэш (`django-redis`) и брокер
  **Celery** для фоновых задач.
- **LDAP/Active Directory** (`ldap3`) и **Keycloak** — аутентификация и
  раздача групп-прав; `django-cuser` — текущий пользователь в сигналах.
- **MinIO** (`django-minio-storage`) — файловое хранилище (`USE_S3_STORAGE`).
- **pandas + openpyxl + xlsxwriter** — импорт/экспорт Excel.
- **django-nested-admin** — 4 уровня вложенных инлайнов в карточке продукта.
- **Sentry** — мониторинг ошибок (DSN зависит от `DEBUG`).
- Зависимости — **pipenv** (`Pipfile`), деплой — Docker + Kubernetes
  (`classifier*.yaml`), CI — GitLab.

## Как запустить

```bash
cp example.env .env          # заполнить SECRET_KEY, POSTGRES_*, REDIS_HOST
pip install pipenv
pipenv install
docker-compose up -d         # postgres + redis
pipenv run python backend/manage.py migrate
pipenv run python backend/manage.py createsuperuser
pipenv run python backend/manage.py runserver
```

Проверки как в CI (flake8 + bandit + radon, пакет `garpix_qa`):

```bash
pipenv run python backend/manage.py qa
pipenv run python backend/manage.py test
```

Первичное наполнение каталога из Excel:
`pipenv run python backend/manage.py load_excel -f fill.xlsx`.

В контейнере приложение поднимается через uwsgi (`uwsgi.ini`,
`app.wsgi:application`), статика админки раздаётся `static-map`.

## Оглавление

- [[Architecture]] — приложения Django, модели, права, логирование, кэш.
- [[API]] — внутреннее API фронта, внешнее `listapi`, аутентификация.

## Исследования

Папки `docs/` в репозитории нет — переносить в `Research/` пока нечего.
