---
project: comet-backend
created: 2026-06-25
updated: 2026-08-05
tags: [project, api, rest, fastapi]
---

# API

#project

Публичный HTTP-API [[Overview|comet-backend]]. Все ручки версионированы и собираются в
`app/api/v1/v1_router.py`. Общий префикс — **`/api/v1`**. Swagger UI — на корне `/`.

Аутентификация — **Bearer JWT** (Keycloak) для всех путей, кроме «skipped» (см.
[[Architecture]]). Ниже пути указаны относительно `/api/v1`.

Многие ручки дополнительно закрыты проверкой прав ЛКМ через
`Depends(require_permission(...))` — механизм описан в [[Architecture]] (раздел
«Права доступа»). В таблицах ниже требуемый пермиссен указан в колонке «Право».

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

## Approval stages — `/approval-stages`

| Метод | Путь | Назначение |
|-------|------|-----------|
| GET | `/approval-stages/pending?limit=50&offset=0` | Доступные текущему LKM user pending-стадии |
| POST | `/approval-stages/{stage_id}/decision` | Решение `approve/reject`; требует `Idempotency-Key` |
| POST | `/approval-stages/{stage_id}/skip` | Пропустить необязательную стадию (`reason`), право `skip_kp_approval_stage` |
| POST | `/approval-stages/{stage_id}/reassign` | Переназначить единоличную стадию (`user_id`, `reason`), право `reassign_kp_approval_stage` |

Ручки не принимают email token или user id: actor берётся только из principal middleware.
Доступ и статус стадии проверяются повторно в locked workflow-транзакции; результат
команды по `Idempotency-Key` сохраняется в БД, повтор возвращает его без нового перехода.
Skip доступен только для optional stage, reassign — только для единоличной (для групповой
возвращается `reassign_not_supported_for_group_stage`).

## Deals (сделки) — `/deals`

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST   | `/deals` | Создать сделку (необязательный `contract_id`, см. ниже) |
| GET    | `/deals` | Список сделок (с фильтрами `DealFilters`) |
| GET    | `/deals/{deal_id}` | Получить сделку |
| PATCH  | `/deals/{deal_id}` | Обновить сделку (`contract_id` не принимается → 422) |
| DELETE | `/deals/{deal_id}` | Удалить сделку |
| POST   | `/deals/{deal_id}/start-implementation` | Создать заказы в ОП по офферам сделки (`change_deal_status`) |
| POST   | `/deals/{deal_id}/approval/request` | Отправить сделку на согласование (инициатор сохраняется в согласовании) |
| POST   | `/deals/{deal_id}/approval/revoke` | Отозвать согласование |
| GET    | `/deals/{deal_id}/approvals?limit=20&offset=0` | История версий согласования сделки (без token-полей) |
| POST   | `/deals/check_send_email` | ⚙️ Техническая: тест отправки письма |
| POST   | `/deals/check_fetch_unseen_emails` | ⚙️ Техническая: тест чтения писем |

`start-implementation` доступна только для сделки с пройденным маршрутом согласования
(одобренной либо не требующей согласования). Для одобренной заказы создаются по
одобренному снимку, по одному на оффер, с идемпотентным ключом; если часть заказов не
создана — 409, сделка не считается переданной.

### Договор сделки

`contract_id` принимается при создании и после создания не меняется. Договор
пришёл — сделка коммерческая; не пришёл — тестовая, бэк сам подставляет тестовый
(`TST*`) договор клиента. Сделка отдаёт вычисляемый тип договора:
`contract_number`, `contract_type` (`test` / `inner` / `commercial`),
`contract_type_title` — и в карточке, и в списке. Подробно —
[[Deal Contract Selection]] и [[DFDEV-2257 Deal Contract Selection]].

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

### Статусы в ответе оффера

Оффер отдаёт два независимых статуса, оба в общей форме `code` / `title` / `kind`:

- `status` — **КП-фаза**: `in_progress` / `formed` / `approval` / `approved` /
  `returned`. Вычисляется из состояния оффера и согласования сделки, не хранится.
  В фазе `approval` добавляется `stage_code` активной стадии — русских подписей стадий
  бэк не отдаёт, их рисует фронт.
- `order.status` — **статус БЗ в ITSM**: `code` (сырой `itsm_state`), `title` и `link`
  от ОП, `kind = external`, `synced_at`. Читается из ОП на каждый запрос сделки, а не
  сохраняется однократно при создании заказа.

Рядом в блоке заказа: `order.order_type` (`test` / `inner` / `commercial`, приходит из
ОП) и `order.kp_phase` — снимок КП-фазы на момент создания заказа, первая точка
мини-таймлайна.

При недоступном ОП карточка и список не падают: приходит последнее сохранённое
значение, признак — `synced_at`. Поля `order.order_status` и `order.synced_at`
помечены deprecated и дублируют `order.status.*` на переходный период.

Подробно, включая режимы чтения и предохранитель — [[Offer Order Status]].

Тариф оффера обязан содержать `price` — цену продажи, от которой считается скидка и
строится маршрут согласования. Тариф без неё отклоняется с кодом
`offer_tariff_price_required`, тариф без цены в прайсе классификатора —
`offer_tariff_price_not_found` (оба 400). `partner_client_price` остаётся в контракте как
цена конечного потребителя и в бизнес-логике согласования не участвует.

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

На 2026-07-23 permissions из БТ02 на этих ручках ещё не навешаны. Целевое состояние:
`create_tariff` для создания/просмотра/редактирования, `approve_tariffs` для
согласования тарифов; см. [[BT02 Role Model Gaps]].

### Комментарии пре-тарифов — `/pre-tariff-comments`

CRUD-набор: POST / GET (список) / GET `/{id}` / PATCH `/{id}` / DELETE `/{id}`.

### Вложения пре-тарифов — `/pre-tariffs/...`

POST загрузка, GET список, GET метаданные, DELETE, GET скачивание файла (S3).

## Customers (прокси к сервису Customers) — `/customers`

Собирает три под-роутера:

- `/customers/client` — клиенты: GET список / `search` / `detailed`, получение по
  `{client_id}`, `by-inn/{inn}`, `by-external-id/{external_id}`, POST создание.
- `/customers/contracts` — договоры (GET/POST/PUT/DELETE). В выдаче есть
  вычисляемый тип договора (`contract_type`, `contract_type_title`). Для выбора
  договора при создании сделки: `?child_client_id=<клиент>&seller=<продавец>` —
  `seller` разворачивается в `parent_client_id` продавца, удалённые договоры по
  умолчанию не отдаются. См. [[Deal Contract Selection]].
- `/customers/staffers` — сотрудники (GET/POST/PUT/PATCH/DELETE).

## Bitrix24 — `/bitrix`

| Метод | Путь | Назначение |
|-------|------|-----------|
| GET | `/bitrix/deal/deals_and_company_by_inn/{inn}` | Сделки и компания из Bitrix24 по ИНН |

## ЛКМ: управление пользователями — `/lkm/users`

Ролевая модель ЛКМ (`app/api/v1/lkm/`). Guard'ы — по пермиссиям. Подробно — в
[[Architecture]] и исследовании [[LKM Role Model]].

| Метод | Путь | Назначение | Право |
|-------|------|-----------|-------|
| GET   | `/lkm/users` | Список видимых пользователей | любой из `add_users` / `change_user_roles` / `change_permissions` |
| GET   | `/lkm/users/search?q=` | Поиск в Keycloak по логину/ФИО (мин. 3 символа) | `add_users` |
| POST  | `/lkm/users` | Добавить пользователя из AD в ЛКМ | `add_users` |
| GET   | `/lkm/users/{user_id}` | Карточка пользователя + эффективные пермиссии | любой из `add_users` / `change_user_roles` / `change_permissions` |
| PATCH | `/lkm/users/{user_id}/role` | Сменить роль (не свою УЗ; сбрасывает персональные права) | `change_user_roles` |
| PATCH | `/lkm/users/{user_id}/email` | Задать канонический email доставки писем согласования (или очистить через `null`) | `add_users` |
| PATCH | `/lkm/users/{user_id}/permissions` | Заменить персональные пермиссии: `permission_codenames` — на все продукты, `scoped_permissions` (`codename` + `product_id`) — на конкретный продукт | `change_permissions` |

## ЛКМ: справочники — `/lkm/roles`, `/lkm/permissions`

| Метод | Путь | Назначение | Право |
|-------|------|-----------|-------|
| GET | `/lkm/roles` | Справочник ролей (8 ролей) с дефолтными наборами прав | `require_admin` (роль `admin`, временный gap) |
| GET | `/lkm/permissions` | Справочник пермиссий (24 шт.: 22 из БТ02 + `skip_kp_approval_stage`, `reassign_kp_approval_stage`) | `require_admin` (роль `admin`, временный gap) |

Справочники кэшируются (данные статичны после seeding).

## Права на ключевых доменных ручках

Проверяются через `require_permission(...)` (см. `app/api/v1/deals/deals.py`,
`app/api/v1/offers/offers.py`):

| Ручка | Право |
|-------|-------|
| `POST /deals` | `create_deal` |
| `GET /deals`, `GET /deals/{id}` | `view_deals` |
| смена статуса сделки | `change_deal_status` |
| отправка КП (`send_kp`-действия) | `send_kp` |
| `POST /offers`, edit/delete оффера | `create_kp` |
| `GET /offers`, `GET /offers/{id}` | `view_kp` |
| внутренние/справочные ручки сделок | `require_admin` |
| `POST /approval-stages/{id}/skip` | `skip_kp_approval_stage` |
| `POST /approval-stages/{id}/reassign` | `reassign_kp_approval_stage` |

---

⚙️ — временные/технические ручки, не часть стабильного контракта (видно по коду:
docstring «Временная техническая ручка»). Обе закрыты `require_admin`.

Доменные сущности, фигурирующие в запросах/ответах, описаны в [[Domain Model]].
