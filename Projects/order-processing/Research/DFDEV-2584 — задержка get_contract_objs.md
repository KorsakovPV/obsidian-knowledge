---
project: order-processing
created: 2026-09-24
tags: [project, research, customers, performance, reliability]
---

#project #research

# DFDEV-2584 — задержка get_contract_objs

Связано: [[Overview]], [[Architecture#Customers и bulk-загрузка контрактов]],
[[API#Исходящий контракт с Customers]].

Тикет: `https://btask.beeline.ru/browse/DFDEV-2584`.

## Статус

Исследование и план исправления завершены. Изменения в код пока не внесены.
План прошёл независимое ревью Claude, Kimi и Codex: финальный круг для PLAN V4 —
`PASS`, без Critical/Major у всех трёх.

## Симптом

Порог `get_contract_objs` — 0,5 с. Наблюдения из production-логов:

- 2026-09-23 21:13: пустой запрос сначала получил `httpx.ConnectTimeout`, после
  retry дошёл до Customers и получил `422`; OP измерил 5,528 с;
- валидный запрос с одним UUID: Customers залогировал 1,160382 с SQL, OP —
  1,593 с end-to-end.

Это два разных источника задержки:

1. OP делает бессмысленный внешний запрос для пустой пачки.
2. Непустой запрос может быть медленным внутри Customers и требует отдельного
   измерения SQL/serialization/network.

## Текущий поток

Задача `op_itsm_adapter` запускается по cron `*/1 * * * *`:

```text
retrieve local + remote
  → match
  → retrieve_external_data
  → resolve
  → perform
  → report
```

В `retrieve_external_data()` список `contract_id` передаётся в
`OrderTransformManager.get_contract_objs()` даже при отсутствии локальных
заказов. `ContractFilter.dict()` превращает `[]` в пустую строку:

```text
id__in=
```

Customers валидирует `id__in` как список UUID, поэтому пустая строка даёт `422`.

## Подтверждённые проблемы в OP

### Потеря семантики ошибок

`get_contract_objs()` возвращает `{}` для:

- `404` — ни один контракт не найден;
- неожиданных `4xx`;
- `5xx` и исчерпанных сетевых попыток;
- `200` с невалидным JSON/schema.

Из-за этого доменное отсутствие данных неотличимо от отказа зависимости.

### Ошибка при неполной пачке

В OP→ITSM adapter результат `contracts.get(contract_id)` используется без
проверки. Если Customers недоступен или не вернул контракт, выполнение может
упасть на `contract.parent_client_id` до `resolve`/`perform`.

### Кэш

Локальный `AsyncRequestServiceBase`:

- считает cache hit только truthy-значение;
- вызывает синхронный Redis из async-кода;
- не обрабатывает ошибку Redis read как fallback к remote;
- создаёт новый `httpx.AsyncClient` на каждый запрос;
- делает до двух попыток на `ReadTimeout`, `ConnectTimeout`, `ConnectError`.

Кэш — отдельный контур работ: общая база используется несколькими интеграциями,
поэтому её нельзя менять только под Customers без инвентаризации потребителей.

## Вызовы get_contract_objs

На момент исследования найдено семь production callsites:

1. OP→ITSM adapter.
2. Смена контракта в base order handler.
3. Test→prod handler.
4. Список заказов: основные контракты.
5. Список заказов: родительские контракты.
6. Отчёт: основные контракты.
7. Отчёт: родительские контракты.

Для каждого вызова до изменения нужно тестом зафиксировать существующий внешний
status/body/errors. Текущая false-not-found семантика публичных ответов сохраняется
ради совместимости; её исправление требует отдельного согласования.

## HTTP-контракт OP → Customers

Базовый вызов:

```text
GET /api/v1/contracts
id__in=<UUID,UUID,...>
relationship=true
is_deleted=<omitted|true|false>
```

Зафиксировать executable contract tests:

- один CSV `id__in`, не repeated query params;
- `relationship=true/false`;
- `is_deleted` omitted/true/false;
- полный и частичный `200`;
- `404` при полном отсутствии результатов;
- `422` для пустого CSV и невалидного UUID;
- фактические поля, типы и nullability raw JSON;
- текущее auth-поведение.

Наличие `child_client` нельзя выводить только из OpenAPI. Customers объявляет
базовый `response_model`, сервис условно создаёт схему с relationship, а mixin
содержит собственную настройку extra fields. Источник истины — реальный
wire-response на точной production-сборке.

## План исправления

### 0. Идентифицировать production bytes

- OP tag `22.53`: `bf3e2b74377bf35d2b2d28ffbfb1fdf8334c4c1c`.
- Customers tag `4.88`: `cc2402d6f295961eeb3ce49af8fb8a3601efe99b`.
- Перед разработкой сравнить с digest реально развёрнутых образов и embedded
  version metadata.
- При расхождении строить baseline по развёрнутому образу, а не по тегу.

### 1. Сначала contract suite

Один и тот же black-box suite запускается против baseline и candidate. Golden
формируется из наблюдаемого baseline, без предположений о `child_client`.
Любое изменение path, query semantics, auth, статуса, поля, типа или nullability
блокирует поставку.

Дополнительно измерить максимальное число UUID и длину request line во всех семи
вызовах вместе с ingress-лимитами. Chunking и silent truncation не входят в этот
тикет.

### 2. Атомарное исправление в order-processing

Одним MR и одним deployment без промежуточной версии:

- `get_contract_objs([])` немедленно возвращает `{}` без Redis/HTTP;
- adapter делает `retrieve_local` первым и возвращает typed `no_work` до ITSM и
  Customers;
- `404` становится доменным отсутствием;
- неожиданный `4xx`/malformed `200` → `ContractsProtocolError`;
- `5xx`/исчерпанные network retries → `ContractsDependencyUnavailable`;
- каждый contract dereference предваряется явной проверкой;
- enrichment fail-closed для contract/client/managers/staffers/subscribers/product;
- исключения из `asyncio.gather(return_exceptions=True)` обрабатываются до
  распаковки;
- `LocalExtendData` публикуется только после успешного построения всей пачки;
- `run_for_one` получает typed outcomes для отсутствующего local/remote заказа,
  без `IndexError` и без внешних side effects.

При любой технической ошибке adapter не переходит к `resolve`/`perform` и не
обновляет ITSM.

### 3. Отдельный MR для cache safety

Сначала инвентаризировать по полному имени класса и source path:

- локальный `app.helpers.crud.v2.request.AsyncRequestServiceBase` и все его
  потребители, включая Bitrix;
- внешний `datafort_utils.AsyncRequestServiceBase`;
- `ExtendedServiceCatalog.get_classifier_paper_products` с собственным Redis;
- `ItsmAdapterBase.customers_service`;
- все прямые `redis.get/set/delete`.

Для локальной базы: `cached_response is not None`, Redis read fail-open к remote,
best-effort write/invalidation, конечные измеренные таймауты и bounded metrics.
Миграция на `redis.asyncio` — отдельное решение после измерения event-loop p99.

### 4. Customers — только по результатам измерений

После OP-фикса и 24 часов baseline измерить непустые запросы для размеров
1/25/100/250/observed-max, разделив:

- pool wait;
- SQL;
- relationship loading;
- Pydantic/FastAPI serialization;
- network/DNS/TLS/ingress.

Изменять Customers только если bottleneck подтверждён. Допустимы лишь внутренние
оптимизации с идентичным raw-wire контрактом. Не менять response model, схему,
query, auth и статусы; не маскировать проблему увеличением timeout/retry.

## Порядок rollout

1. `O`: observability, затем не менее 24 часов baseline.
2. `A`: атомарное изменение поведения OP.
3. `C`: cache safety.
4. `D`: внутренний Customers tuning, если подтверждён.
5. `N`: отдельные network-исправления.

Немедленный rollback текущей волны при contract diff, новом пустом outbound,
ITSM side effect после dependency failure, новом необработанном исключении либо
изменении auth/status/body.

## Проверка

- focused и full pytest, lint, typecheck в CI-окружениях обоих сервисов;
- raw-wire suite против baseline и candidate;
- staging replay/load без PII;
- повторная проверка семи callsites перед merge;
- fault injection каждой зависимости с утверждением «ITSM update не вызван»;
- отдельные тесты `run` и `run_for_one`.

Во время анализа существующий focused baseline OP дал `80 passed`; независимый
Codex review дополнительно получил `53 passed` для `OrderTransformManager` и
`130 passed` на расширенном релевантном наборе. Повторный запуск Customers и
adapter-suite в параллельном окружении не состоялся из-за ошибки создания
временного файла; это не считается успешным прогоном.

## Граница изменений

Минимальное обязательное исправление находится в `order-processing`. Изменения
в Customers условны и нужны только при подтверждённой задержке непустых запросов.
Контракты между приложениями не меняются: прекращается только ошибочный запрос с
пустым `id__in`.
