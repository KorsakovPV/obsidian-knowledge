---
project: customers
created: 2026-08-05
updated: 2026-08-05
tags: [project, api]
---

# API

Публичные интерфейсы сервиса. Точка входа — [[Overview]], структура — [[Architecture]].
Роутеры подключаются в `app/api/api_router.py` → `app/api/v1/v1_router.py`.

Общий префикс — `/api/v1`. Swagger отдаётся на корне (`/`).
Служебная ручка: `GET /api/healthcheck` — проверка связности с Sentry.

## Роутеры

Версия одна (v1). Тег → роутер (`api/v1/<домен>/<домен>_router.py`), теги описаны
в `api/docs/tags.py` (`ApiTags`):

| Тег (Swagger) | Префикс | Файл |
|---|---|---|
| API V1 / Clients | `/api/v1/clients` | `clients/clients_router.py` |
| API V1 / Данные из ЕГРЮЛ | `/api/v1/egruls` | `egruls/egruls_router.py` |
| API V1 / Contracts | `/api/v1/contracts` | `contracts/contracts_router.py` |
| API V1 / Payment credentials | `/api/v1/payment_credentials` | `payment_credentials/payment_credentials_router.py` |
| API V1 / Staffers | `/api/v1/staffers` | `staffers/staffers_router.py` |
| API V1 / Managers | `/api/v1/managers` | `managers/managers_router.py` |
| API V1 / Info | `/api/v1/info` | `info/info_router.py` |

Примеры запросов и описания ответов вынесены в `api/docs/v1/`
(`requests.py`, `open_api_docs.py`, `conflicts.py`) — например,
`INVALID_INN_422`, `CLIENT_ALREADY_EXISTS_409`.

## Клиенты (`/api/v1/clients`)

| Метод | Путь | Назначение |
|---|---|---|
| GET | `` | список клиентов / поиск (`search`, `is_deleted`) |
| GET | `/detailed` | список с детализацией, фильтры `ClientModelFilter` |
| POST | `` | создание клиента (упрощённая схема + подтяжка ЕГРЮЛ) |
| GET | `/{client_id}` | клиент по id |
| GET | `/by_external_id/{external_id}` | клиент по `external_id` |
| PUT | `/{client_id}` | обновление |
| PATCH | `/deanon/{client_id}` | деанонимизация клиента |
| DELETE | `/{client_id}` | мягкое удаление (архивация) |
| PATCH | `/restore/{client_id}` | восстановление из архива |
| POST/PUT | `/{client_id}/egruls`, `/{client_id}/egruls/{egrul_id}` | привязка/обновление данных ЕГРЮЛ |
| POST | `/{client_id}/payment_credentials` | добавить платёжные реквизиты |
| POST | `/{client_id}/staffers` | добавить контактное лицо |
| POST | `/{client_id}/managers` | добавить менеджера Датафорта |

Схемы ответов — `ClientSchemaOutSimplified` (список) и `ClientSchemaOutExtended`
(детально), см. `app/schemas/clients.py`.

### Поиск (`?search=`)

`GET /api/v1/clients?search=…` работает в три этапа:

1. точное совпадение по `inn` / `title` / `short_name_with_opf`;
2. поиск по `contract_number` среди договоров;
3. если точных совпадений нет — нечёткий полнотекстовый поиск (pg_trgm,
   см. [[Architecture#Поиск]]).

Время выполнения хендлера логируется; дольше 4 секунд — уровень `error`.

### Авторство изменений

Мутирующие ручки принимают `created_by` / `updated_by` (`EmailStr | ActionSourceTags`),
по умолчанию — `DEFAULT_CREATED_BY` из `core/config.py`. `ActionSourceTags`
позволяет отличить правки адаптеров (ITSM, OP, Dadata) от пользовательских.

## Остальные домены

- **Contracts** — `GET ''`, `GET /{contract_id}`, `POST ''`, `PUT`/`PATCH /{contract_id}`,
  `DELETE /{contract_id}`, `PATCH /restore/{contract_id}`.
- **Staffers** — `GET ''`, `GET /{staffer_id}`, `PUT`/`PATCH /{staffer_id}`,
  `DELETE /{staffer_id}`, `PATCH /restore/{staffer_id}`. Создаются через клиента.
- **Payment credentials** — `GET ''`, `GET /{id}`, `PUT`/`PATCH /{id}`, `DELETE /{id}`.
- **Egruls** — `GET ''`, `GET /{egrul_id}`, `DELETE /{egrul_id}`.
- **Managers** — `GET ''`, `GET /{manager_email}` (поиск по email),
  `POST /synchronize` (синхронизация менеджеров клиента по `client_id`).
- **Info** — `GET /contract_number_patterns`: допустимые паттерны и примеры номеров
  договоров в разрезе ИНН (`VALID_CONTRACT_NUMBER_PATTERNS_MAPPING`
  в `helpers/constants.py`).

## Фильтры и пагинация

Списочные ручки с детализацией используют `fastapi-filter`
(`FilterDepends`, фильтры в `app/filters/`: `clients.py`, `contracts.py`,
`staffers.py`, `managers.py`). Например, `ClientModelFilter` даёт
`is_deleted` и `external_id__in`.

## Мягкое удаление

`DELETE` не удаляет строку, а ставит `is_deleted = true` (поле есть в `Base`).
Архивные записи возвращаются только при явном `is_deleted=true`,
восстановление — через `PATCH /restore/{id}`.

## Аутентификация

**HTTP Basic**, собственные учётки в таблице `users` — не Keycloak
(Keycloak в конфиге нужен только для исходящих запросов в Order Processing).

- `AuthService` (`services/auth_service.py`) — зависимость `Security(HTTPBasic(auto_error=False))`,
  каждая ручка явно вызывает `await auth_service.authenticate()`.
- Пароли — bcrypt. `verify_password` обёрнут в `lru_cache`: `bcrypt.checkpw`
  синхронный и стоит ~0.2 с на запрос, поэтому результат кешируется
  (осознанный компромисс — сервис во внутреннем контуре). Альтернатива
  без кеша — `verify_password_async` через `ThreadPoolExecutor`.
- `auto_error=False` оставлен временно, до перевода всех зависимых сервисов
  (TODO в коде).

### Создание пользователя

Учётки заводятся только из кода — API для этого нет. Функция
`create_user(username, password)` (`services/auth_service.py`) хеширует пароль
через `bcrypt.gensalt()` / `bcrypt.hashpw()` и пишет запись в `users`.

Интерактивного ввода нет: блок `if __name__ == '__main__'` хардкодит
`username='test', password='password'`, поэтому прямой запуск файла заведёт
именно тестовую учётку. Правильный способ — разовый вызов функции:

```bash
export PYTHONPATH=$PWD
poetry run python3 -c "
import asyncio
from app.services.auth_service import create_user
asyncio.run(create_user(username='ЛОГИН', password='ПАРОЛЬ'))
"
```

Подводные камни:

- БД берётся из `db_context()`, то есть из `.env` (`POSTGRES_*`) — перед запуском
  проверить, что это нужный контур.
- `created_by` захардкожен внутри функции (`sonosov@datafort.ru`); если нужен
  свой email в аудите — править функцию или обновлять поле после вставки.
- Пароль в открытом виде нигде не сохраняется, восстановить его нельзя —
  только перезаписать хеш.
