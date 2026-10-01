---
project: comet-backend
created: 2026-09-28
updated: 2026-09-28
tags: [project, ticket, deals, api, pagination, filters]
---

# DFDEV-2561 — список сделок для рабочего места менеджера

#project #ticket #deals #api

Связано: [[Overview]], [[API]], [[Domain Model]],
[[DFDEV-2052 Deal and Offer Status Contract]].

Тикеты:

- https://btask.beeline.ru/browse/DFDEV-2561 — головная задача;
- https://btask.beeline.ru/browse/DFDEV-2659 — пагинация, сортировка, scope и фильтры;
- https://btask.beeline.ru/browse/DFDEV-2658 — обязательный summary, число БЗ и регулярная сумма;
- https://btask.beeline.ru/browse/DFDEV-2657 — контрагент, менеджер и последнее изменение.

## Статус

DFDEV-2659, DFDEV-2658 и DFDEV-2657 реализованы локально и прошли полный набор
тестов и статических проверок. `GET /api/v1/deals` возвращает страницу
`DealListPageSchema` с обязательным `summary`, контрагентом, ответственным
менеджером и данными последнего изменения.

## Рамки

В ближайший backend-срез входят только DFDEV-2659, DFDEV-2658 и DFDEV-2657.
DFDEV-2660 (статус/этап БТ01), DFDEV-2661 (`products_preview`) и DFDEV-2662
(контакт, запросы и обращения) остаются отдельным бэклогом.

Детальная ручка `GET /api/v1/deals/{deal_id}` сохраняет `DealSchema`. Новый контракт
нужен только списку.

## Порядок поставки

### 1. DFDEV-2659 — страница, сортировка, scope и фильтры

Этот тикет делается первым: от page-контракта и фильтров зависит фронт.

Контракт ответа:

```json
{
  "items": [],
  "total": 0,
  "limit": 20,
  "offset": 0
}
```

- Добавить `DealListPageSchema` и облегчённый `DealListItemSchema`.
- Не включать в строку тяжёлые `offers`, `attachments` и `current_approval`.
- Сохранить существующие фильтры и добавить `manager_id`, `q`, `scope`, `limit`,
  `offset`, `sort`.
- Default: `limit=20` (`1..100`), `offset=0` (`>=0`), сортировка `updated_at DESC`, где list-значение
  `updated_at = coalesce(deal.updated_at, deal.created_at)`; разрешить только
  согласованный allowlist полей (`updated_at`, `created_at`, `number`) и направлений.
- Любую сортировку завершать стабильными tie-breaker: `number` в том же направлении,
  затем `id`, чтобы страницы не дублировались при одинаковых значениях основного ключа.
- `scope=all` не добавляет ограничений. `scope=my` означает: текущий пользователь —
  ответственный менеджер сделки или доверенный автор последнего изменения.
- `manager_id` фильтруется по доверенному FK; неизвестный UUID даёт пустую страницу.
- `q` ищет по названию сделки, названию/ИНН контрагента через Customers. Остальные
  фильтры объединяются через `AND`.
- `total` считается тем же WHERE, что и выборка страницы, но без eager load, сортировки,
  limit и offset.
- Permission остаётся `view_deals`; identity для `scope=my` берётся только из
  аутентифицированного principal.

Для доверенной семантики «мои» нужны серверные поля сделки:

- `manager_id -> lkm_users.id` — ответственный, назначается создателю сделки;
- `last_changed_by_user_id -> lkm_users.id` и `last_changed_at` — последнее
  пользовательское изменение;
- FK nullable, `ON DELETE SET NULL`, с индексами;
- legacy `created_by`/`updated_by` разрешено использовать только как отображаемый
  fallback, но не для прав, scope или фильтра менеджера;
- автоматический backfill назначает пользователя только при одном однозначном совпадении
  по нормализованным email/ad_login. Неоднозначные строки не угадываются и попадают в
  отдельный отчёт перед релизом.
- `last_changed_at` у legacy-строк остаётся `NULL`: массовый `UPDATE` исключён из
  миграции, чтобы не блокировать живую таблицу; для отображения предусмотрен fallback
  `coalesce(last_changed_at, updated_at, created_at)`. Новые записи получают значение
  из сервиса и DB default.

Публичные create/update-схемы больше не должны принимать `created_by`/`updated_by`:
actor и trusted FK выставляет service из `LkmUserModel`. Внутренние системные изменения
не подменяют автора пользовательского изменения.

Изменения по слоям:

- `app/models/deal.py`, миграция Alembic — trusted FK/time и индексы;
- `app/schemas/deal.py` — query enums, page и list item;
- `app/api/v1/deals/deals.py` — новый response model и передача текущего пользователя;
- `app/cruds/deal.py` — общий WHERE, отдельные count/page statements;
- `app/services/deal.py` — list-specific builder без сборки полного `DealSchema`;
- `app/cruds/lkm_user.py` — distinct список назначенных менеджеров;
- `GET /api/v1/deals/managers` — минимальный справочник `{id,name,email}` под
  `view_deals`, если фронту нужен отдельный источник опций фильтра.

Миграции разделены: первая добавляет nullable FK и валидирует их через PostgreSQL
`NOT VALID`/`VALIDATE CONSTRAINT`, вторая создаёт индексы `CONCURRENTLY`. Для default
сортировки добавлен expression index по `coalesce(updated_at, created_at), number, id`.
Добавление FK выполняется в отдельном autocommit-блоке: короткий
`SHARE ROW EXCLUSIVE` освобождается до backfill и `VALIDATE`, поэтому миграция не
блокирует запись в `deal` на всё время обработки legacy-строк.

List-query не загружает `Offer.order` и историю согласований: читается только current
approval с минимальным набором полей и его стадии. По офферам и начатым имплементациям
выбираются только нужные идентификаторы; наличие активных сканов определяется отдельным
`SELECT DISTINCT offer_id`, без загрузки всех attachment rows. Метаданные уникальных
продуктов страницы читаются из классификатора chunked-запросами по `id__in`, а не
отдельным HTTP-запросом на каждый продукт.

Деградация поиска: обычная выдача не должна падать из-за обогащения Customers;
`client_title/client_inn` ещё не входят в этот тикет. Для `q`, где Customers участвует
в самом predicate, ошибка upstream должна возвращаться явно, иначе ответ был бы ложно
неполным.

### 2. DFDEV-2658 — гарантированный summary

Реализован после page-контракта: от этих данных зависит счётчик БЗ в строке.

- Сделать `DealSchema.summary` обязательным и гарантировать объект также в
  `DealListItemSchema`.
- Добавить `total_orders_count`: число офферов с локальной связью `offer.order`.
- Добавить `total_recurring_with_vat` как
  `total_flat_rate_with_vat + total_usage_based_with_vat`.
- Для сделки без офферов вернуть нулевые счётчики и денежные значения `0.00`.
- Счётчик БЗ не требует live-запроса статусов в Order Processing: считается наличие
  локальной связи, а не текущее состояние заказа.
- Оставить один чистый калькулятор summary для detail и list, чтобы формулы не
  расходились.

List pipeline не запускает N+1:

- уникальные tariff ids всех офферов страницы собираются до расчёта;
- тарифы загружаются одним chunked batch-запросом к классификатору;
- наличие БЗ для всех офферов страницы читается одним запросом к `offer_order`;
- суммы считаются из сохранённых в оффере цен и заранее загруженных тарифов без
  построения `OfferReadSchema`, подписания attachment URL и live-запросов в Order
  Processing;
- detail и list используют общую формулу расчёта summary и общую логику расчёта
  тарифных сумм.

Изменения по слоям:

- `app/schemas/deal.py` — новые обязательные поля summary;
- `app/cruds/deal.py` — batch-проверка локальных связей с БЗ;
- `app/services/offer.py` — отделён расчёт по уже загруженным тарифам классификатора;
- `app/services/deal.py` — единый калькулятор summary и batch pipeline для страницы.

### 3. DFDEV-2657 — плоские поля контрагента и менеджера

Реализован третьим и добавляет в строку списка:

```json
{
  "client_title": "ООО Ромашка",
  "client_inn": "5400000000",
  "manager": {"id": "...", "name": "Иван Иванов", "email": "..."},
  "last_change": {"at": "2026-09-28T10:00:00Z", "by_name": "Пётр Петров"}
}
```

- уникальные `client_id` страницы загружаются из Customers batch-запросами по
  `id__in`, без запроса на каждую сделку;
- При недоступности Customers обычный список остаётся доступен, а клиентские поля
  возвращаются `null`; сбой логируется один раз на batch.
- `manager` строится только из `deal.manager` и не подменяется последним изменившим.
- `last_change.at = coalesce(last_changed_at, updated_at, created_at)`.
- `last_change.by_name` берётся из trusted пользователя; для legacy-строки допустим
  display fallback на `updated_by/created_by` без влияния на scope и права.
- Все новые поля и варианты деградации описываются по-русски в OpenAPI.

Изменения по слоям:

- `app/schemas/deal.py` — `DealManagerSchema`, `DealLastChangeSchema`, поля list item;
- `app/services/deal.py` — batch enrichment клиентов, безопасная деградация и сборка
  плоских полей;
- `app/cruds/deal.py` — eager load manager/last changer только для списка.

## Совместимость и релиз

DFDEV-2659 меняет верхний уровень ответа `GET /api/v1/deals` с массива на объект страницы.
Это breaking change и требует согласованного релизного окна с фронтом. Если атомарная
поставка невозможна, до реализации нужно выбрать versioned endpoint; временную
backwards-compatible обёртку в v1 не добавлять.

DFDEV-2658 и DFDEV-2657 расширяют уже введённый list item и поставляются следующими
отдельными MR. Они не должны блокировать первую поставку page/filter contract.

Offset pagination допускает сдвиг строк при конкурентных изменениях; стабильный
tie-breaker устраняет неоднозначность порядка, но не создаёт snapshot isolation. Для
сценария «Показать ещё» это принимается; cursor — отдельное улучшение.

## Тесты

### DFDEV-2659

- default `limit=20`, точный `total`, непересекающиеся offset-страницы;
- стабильная default-сортировка и все разрешённые sort/direction;
- 422 для неизвестных sort/scope и неверных limit/offset;
- сохранение смысла старых фильтров;
- `q`: title/название клиента/ИНН, остальные фильтры через `AND`;
- `scope=my` только по trusted FK; поддельный legacy `updated_by` не влияет;
- `manager_id`, включая неизвестный UUID;
- count и page используют одинаковый predicate;
- миграция: однозначный, отсутствующий и неоднозначный backfill;
- 403 без `view_deals` и отсутствие query-параметра, способного подменить current user.

### DFDEV-2658

- summary пустой сделки: counts `0`, суммы `0.00`;
- recurring = flat + usage, one-time не входит;
- `total_orders_count` считает только офферы с order;
- detail и list дают одинаковый summary;
- mock call-count подтверждает отсутствие запросов на каждый оффер/сделку;
- list не вызывает Order Processing status loader.

### DFDEV-2657

- сделка создана A и изменена B: `manager=A`, `last_change.by_name=B`;
- manager берётся только из FK;
- Customers outage даёт 200 и nullable client fields для обычного списка;
- legacy display fallback не влияет на scope;
- русские descriptions новых полей присутствуют в OpenAPI.

Общие проверки каждого MR:

```bash
poetry run pytest <релевантные тесты>
poetry run pytest
poetry run make check
```

## Приёмка

- Проверить страницу минимум на 25 сделках: пустая, без БЗ, с несколькими КП и частью
  БЗ, созданная одним пользователем и изменённая другим.
- Проверить «Показать ещё», `total`, `scope`, фильтр менеджера и все сортировки.
- Отключить Customers: обычный список остаётся доступен с `null` в клиентских полях;
  поиск по контрагенту возвращает диагностируемую upstream-ошибку.
- По логам и метрикам подтвердить, что число обращений к Customers/LKM/классификатору
  ограничено batch/chunk size, а Order Processing в list endpoint не вызывается.
- Перед включением нового endpoint проверить нулевой необработанный остаток legacy
  `manager_id` либо письменно принять nullable manager для этих строк.

## Ревью плана

План проверен Codex, Claude и Kimi. Критичные и major-замечания были закрыты решениями
про trusted identity, стабильную пагинацию, batch enrichment, fail-closed поиск и
разделение detail/list контрактов. После уточнения владельца бэкенда поставка разбита на
три последовательных MR: DFDEV-2659 → DFDEV-2658 → DFDEV-2657.
