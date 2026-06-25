---
project: comet-backend
created: 2026-06-25
tags: [project, api, rest, fastapi]
---

# API

#project

Публичный HTTP-API [[Overview|comet-backend]]. Все ручки версионированы и собираются в
`app/api/v1/v1_router.py`. Общий префикс — **`/api/v1`**. Swagger UI — на корне `/`.

Аутентификация — **Bearer JWT** (Keycloak) для всех путей, кроме «skipped» (см.
[[Architecture]]). Ниже пути указаны относительно `/api/v1`.

## Служебное

| Метод | Путь | Назначение |
|-------|------|-----------|
| GET | `/api/healthcheck` | Проверка связанности с Sentry (вне `/v1`) |

## Auth — `/auth`

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST | `/auth/login` | Аутентификация пользователя |
| GET  | `/auth/user` | Текущий пользователь |
| POST | `/auth/logout` | Выход |

## Deals (сделки) — `/deals`

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST   | `/deals` | Создать сделку |
| GET    | `/deals` | Список сделок (с фильтрами `DealFilters`) |
| GET    | `/deals/{deal_id}` | Получить сделку |
| PATCH  | `/deals/{deal_id}` | Обновить сделку |
| DELETE | `/deals/{deal_id}` | Удалить сделку |
| POST   | `/deals/{deal_id}/start-implementation` | Запустить имплементацию сделки *(сейчас заглушка)* |
| POST   | `/deals/{deal_id}/approval/request` | Отправить сделку на согласование |
| POST   | `/deals/{deal_id}/approval/revoke` | Отозвать согласование |
| POST   | `/deals/check_send_email` | ⚙️ Техническая: тест отправки письма |
| POST   | `/deals/check_fetch_unseen_emails` | ⚙️ Техническая: тест чтения писем |

## Вложения сделок — `/deals/{deal_id}/attachments`

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST   | `/deals/{deal_id}/attachments` | Загрузить вложение |
| GET    | `/deals/{deal_id}/attachments` | Список вложений |
| GET    | `/deals/{deal_id}/attachments/{attachment_id}` | Метаданные вложения |
| DELETE | `/deals/{deal_id}/attachments/{attachment_id}` | Удалить вложение |
| GET    | `/deals/{deal_id}/attachments/{attachment_id}/file` | Скачать файл (S3) |

## Offers (офферы) — `/offers`

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST   | `/offers` | Создать оффер |
| GET    | `/offers/{offer_id}` | Получить оффер |
| GET    | `/offers` | Список офферов |
| PATCH  | `/offers/{offer_id}` | Обновить оффер |
| DELETE | `/offers/{offer_id}` | Удалить оффер |
| DELETE | `/offers/by-deal/{deal_id}` | Удалить офферы сделки |

## Orders (заказы) — `/orders`

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST | `/orders/{offer_id}` | Создать заказ в ОП по офферу |

## Products (продукты) — `/products`

| Метод | Путь | Назначение |
|-------|------|-----------|
| GET | `/products/products_names` | Список названий продуктов |
| GET | `/products/{product_id}` | Продукт по id |

## Client prices (клиентские цены) — `/client-prices`

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST | `/client-prices` | Создать клиентскую цену |
| GET  | `/client-prices/effective` | Действующая цена |
| GET  | `/client-prices/{price_id}` | Цена по id |
| GET  | `/client-prices/client/{client_id}/prices` | Цены клиента |

## Pre-tariffs (пре-тарифы) — `/pre-tariffs`

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST   | `/pre-tariffs` | Создать пре-тариф |
| GET    | `/pre-tariffs` | Список пре-тарифов |
| GET    | `/pre-tariffs/{pre_tariff_id}` | Получить пре-тариф |
| PATCH  | `/pre-tariffs/{pre_tariff_id}` | Обновить пре-тариф |
| DELETE | `/pre-tariffs/{pre_tariff_id}` | Удалить пре-тариф |

### Комментарии пре-тарифов — `/pre-tariff-comments`

CRUD-набор: POST / GET (список) / GET `/{id}` / PATCH `/{id}` / DELETE `/{id}`.

### Вложения пре-тарифов — `/pre-tariffs/...`

POST загрузка, GET список, GET метаданные, DELETE, GET скачивание файла (S3).

## Customers (прокси к сервису Customers) — `/customers`

Собирает три под-роутера:

- `/customers/client` — клиенты (GET-поиск/получение, POST).
- `/customers/contracts` — договоры (GET/POST/PUT/DELETE).
- `/customers/staffers` — сотрудники (GET/POST/PUT/PATCH/DELETE).

## Bitrix24 — `/bitrix`

| Метод | Путь | Назначение |
|-------|------|-----------|
| GET | `/bitrix/deal/...` | Получение данных сделки из Bitrix24 |

---

⚙️ — временные/технические ручки, не часть стабильного контракта (видно по коду:
docstring «Временная техническая ручка» / `start-implementation` — заглушка до
реализации бизнес-логики).

Доменные сущности, фигурирующие в запросах/ответах, описаны в [[Domain Model]].
