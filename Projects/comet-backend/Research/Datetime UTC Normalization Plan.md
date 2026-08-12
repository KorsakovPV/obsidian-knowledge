---
project: comet-backend
created: 2026-08-11
source: docs/datetime_utc_normalization_plan.md
tags: [project, research, datetime, utc, api-contract]
---

# План правок: единый aware-UTC формат datetime во всех ответах API

#research #datetime #utc

Связано: [[Overview]], [[API]].

Основание — аудит от 2026-08-06 (три прохода: поля схем и их источники, точки
создания дат и таймзоны окружения, форматы внешних сервисов).

Целевое состояние: каждая datetime-строка в ответах API имеет суффикс таймзоны,
канонично `Z`. Ключевой факт из аудита: БД полностью на `timestamptz` и отдаёт
aware, Pydantic v2 сериализует aware-UTC как `...Z` сам — поэтому чинится не
сериализация, а **значения**: naive-даты рождаются в шести известных точках, и
допущение «naive = UTC» верно для нашего кода, но неверно для границ (Bitrix
naive = МСК).

## Этап 0. Живые проверки — ВЫПОЛНЕНО 2026-08-06 (стенды из configmap-comet-backend-test)

| # | Проверка | Результат |
|---|---|---|
| 0.1 | Формат дат customers | ✅ **aware `+00:00`** и в `created_at`, и в `contract_date` (`GET https://customers-t.datafort.ru/api/v1/contracts`). Запись: naive `contract_date: "2030-01-01T00:00:00"` сохраняется как `"...+00:00"` — **customers трактует naive как UTC** (проверено POST + DELETE пробного договора). Значит переход прокси-пути на aware-UTC семантику не меняет. Попутно: при `is_test: true` customers игнорирует переданный `contract_number` и генерирует свой `TST*` |
| 0.2 | Формат дат ОП | ✅ **aware `+00:00`**: `created_at: "2026-07-28T07:08:09.359115+00:00"`, `ready_at: "2025-11-17T10:43:56+00:00"`; `order_date` / `date_of_technological_readiness_of_services` / `billing_start_date` — date-only (`"2025-11-17"`). **Этап 3.3 упрощается: граничный валидатор для ОП не нужен**, naive от ОП не приходит |
| 0.3 | TZ базового образа | ✖ снаружи недоступно: registry `df-ors-registry` требует авторизацию (образ `ors/base-bookworm`), а API-утечка naive `now()` закрыта правами (клиентский Keycloak-токен → 403). Доделать одной командой при доступе к кластеру: `kubectl exec -n comet-backend-test deploy/comet-backend -- date`. Компенсация: этап 5.1 (`TZ=UTC` явно) закрывает вопрос навсегда; текущий TZ влияет только на `datetime.now()` в client_price и ничего не хранит |
| 0.4 | `Date:`-заголовки почты | ✖ порт 993 `post.datafort.ru` фильтруется с рабочей машины (timeout); проверка возможна из кластера. Компенсация: фикс 2.3 (naive → UTC) корректен по RFC 5322 независимо от фактических заголовков — naive у `parsedate_to_datetime` бывает только при `-0000`, что по RFC и означает UTC |

## Этап 1. Фундамент: глобальная нормализация на валидации — ВЫПОЛНЕНО

1.1. `app/schemas/base.py` — `ApiSchema(BaseModel)` c wildcard-валидатором
(`@field_validator("*", mode="after")`): `datetime` naive → `replace(tzinfo=utc)`,
aware → `astimezone(utc)`; рекурсивный обход list/dict для контейнеров с датами.
Сериализатор не нужен: aware-UTC Pydantic отдаёт как `Z`.

1.2. Механический прогон: все схемы в `app/schemas/` наследуются от `ApiSchema`
вместо `BaseModel` (включая внутренние write-схемы — вреда нет, выгода в
единообразии сравнений).

1.3. Тесты-критерии приёмки:
- юнит на базу: naive / aware-МСК / aware-UTC → на выходе строка с `Z`;
- интеграционный: собрать `DealSchema` с датами всех видов (заказ, согласование,
  kp_phase, контракт), сериализовать в JSON, рекурсивно проверить каждую
  datetime-строку на `(Z|[+-]\d{2}:\d{2})$` — naive = фейл. Это критерий
  «в GET /lkm/deals/{id} ни одной строки без суффикса», живущий в CI.

## Этап 2. Точечные фиксы naive-источников — ВЫПОЛНЕНО (кроме SQL-ревизии данных: нужен доступ к БД стенда)

2.1. **client_price** (худшее место аудита):
- `app/services/client_price.py:113` `datetime.now()` → `datetime.now(timezone.utc)`
  (симметрично bulk-пути `:141`);
- `app/cruds/client_price.py:156-157` — `replace(tzinfo=utc)` больше не нужен
  (вход нормализован), заменить на assert/удалить;
- `app/api/v1/client_price/client_price.py:80` — naive `now()` в тексте 404;
- **риск**: если TZ хоста не UTC (этап 0.3), правка `:113` меняет результат
  `TSTZRANGE @>` запроса на величину смещения — это исправление бага, но
  задокументировать в ПР;
- ревизия данных: `scripts/price_filler.py` писал naive-границы `valid_period`
  (локальная полночь) и naive-сентинел `datetime(2099, 12, 31)` — одноразовый
  SQL-аудит границ + фикс скрипта.

2.2. **Единая «дата заказа»**: `datetime.date.today()` (локальная) в
`app/services/order.py:89` и `offer_implementation_service.py:360` против
UTC-даты в `offer.py:402`. Привести к одной конвенции —
`datetime.now(timezone.utc).date()` во всех трёх (сдвиг около полуночи,
согласовать с ОП не требуется: формат не меняется, только источник часов).

2.3. **Почта**: `app/services/email_transport.py:386` — после
`parsedate_to_datetime` нормализовать: naive (заголовок `-0000`) → UTC по RFC.
Снимает живой риск `TypeError` в `deal.py:739`.

## Этап 3. Граничные нормализаторы — ВЫПОЛНЕНО

3.1. **Bitrix24**: naive у него — московское время. В валидаторах
`app/schemas/bitrix24.py` (`LAST_COMMUNICATION_TIME` :107-122, :369-382,
`MOVED_TIME`, `BEGINDATE`) — naive → `Europe/Moscow` → UTC. Использовать
`settings.msk_tz` (`app/core/config.py:52`, сейчас мёртвый код — оживает здесь).
Разобрать union `CLOSEDATE: datetime | str`.

3.2. **Клиентский ввод** (`as_of`, `valid_from/to`, `contract_date`, `deal_date`,
даты в PATCH): принимаем naive и трактуем как UTC (это дефолт нормализации
этапа 1), решение фиксируется в `description` полей. Альтернатива — 422 на
naive — отвергнута: ломает текущий фронт. Отдельно: прокси-путь `contract_date`
теперь отправит в customers aware-UTC вместо naive (сверить с 0.1, что customers
это переваривает).

3.3. **Classifier / ОП**: `ClassifierTariff.*: datetime | str` — internal-only,
нормализуются этапом 1 там, где распарсились; рассинхрон `archived_at`
(datetime у classifier / str в offer-слоях) — отдельный тикет, не блокер.
`OrderOutSchema.ready_at` — по результату 0.2 ОП шлёт aware `+00:00`,
граничный валидатор не нужен, этап 1 покрывает с запасом.

## Этап 4. Семантические даты → `date` (отдельный ПР, после согласования)

Поля-дни, где UTC-нормализация может сместить календарное число
(`00:00 МСК → 21:00Z предыдущего дня`):

| Поле | Решение |
|---|---|
| `contract_date` (схемы контрактов + `CustomerClientContractSchema`) | → `date`; согласовать с customers и фронтом; `deal.py:482` штампует дату, а не момент |
| `deal_date` (`DealBase`/`DealUpdateSchema`/`DealSchema`) | → `date` в API, колонка `timestamptz` остаётся (без миграции) |
| Bitrix `BEGINDATE` / `CLOSEDATE` | → `date` (у Bitrix это дни) |
| classifier `start_at` / `end_at` | → `date`, internal-only, низкий приоритет |
| client_price `valid_from` / `valid_to` | оставить datetime (границы `TSTZRANGE`, полуинтервалы) |

Образец уже в коде: биллинговые даты `OrderOutSchema` типизированы `date`.
Этап 4 не блокирует этапы 1–3: без него формат станет aware-UTC, риск смены
числа остаётся и явно проговаривается фронту.

## Этап 5. Окружение и гигиена — ВЫПОЛНЕНО

5.1. `ENV TZ=UTC` в `Dockerfile_base`, `TZ: UTC` в пять k8s-конфигмапов —
чтобы смысл naive-значений впредь не зависел от невидимого базового образа.
Postgres не трогаем (дефолт образа — UTC).

5.2. Консольный лог-форматтер (`app/core/log_settings.py:103-104`) — UTC или
явное смещение в `datefmt`; сейчас JSON-логи в UTC, консольные — в локальном
без пометки.

5.3. Попутные баги из аудита (чинятся почти бесплатно):
- `app/schemas/client_price.py:15-21` — naive/aware сравнение в валидаторе
  кидало `TypeError` (500 вместо 422); этап 1 чинит автоматически — добавить
  регресс-тест;
- `app/services/approval_lifecycle.py:193-201` — snapshot оффера не пишет
  `created_at`, сортировка в `approval_stage_tasks.py:129-136` всегда падает в
  `datetime.min`-фоллбек — маленький фикс + тест.

## Правки по ревью 2026-08-06 — ВЫПОЛНЕНО

Все восемь пунктов внесены. Дополнительно по ходу:
- guard-тест на весь `app/` реализован без allowlist — 13 моделей вне
  `app/schemas` (`order_processing`, `approval_workflow`, роутеры, api/docs)
  переведены на `ApiSchema`, pydantic-settings исключены по базовому классу;
- попутно найден и убран дубль `_list_contracts` в `deal.py` (артефакт merge);
- ограничение по `subject_snapshot` пересмотрено по ревью: **хранение** и хеш
  неприкосновенны, но **отдача** нормализуется — targeted-сериализатор на
  `subject_snapshot`/`route_context` приводит ISO-строки к aware-UTC только в
  ответе (`normalize_datetime_strings_to_utc`). Закрывает naive-строки и в
  легаси-снапшотах, и в новых снапшотах старых офферов (тарифы копируются из
  JSONB как есть). Тесты: legacy-снапшот, неизменность хранимого dict,
  прозрачность обычных строк;
- `_parse_lenient_datetime` пишет warning при отбрасывании мусора — потеря
  значения видна в логах.

Из двух ревью (жёсткое P1-ревью и approve-ревью DFDEV-2267_2 → DFDEV-2267_1):

1. **mypy** — 4 ошибки в `base.py` / `log_settings.py` / `approval_lifecycle.py`;
   `mypy .` — шаг CI, блокер.
2. **black** — 3 файла не отформатированы (в CI black нет, но репозиторий
   отформатирован — конвенция).
3. **classifier `datetime | str`** — smart-union оставляет строку строкой, naive
   `archived_at` доезжает до тарифов в `GET /lkm/deals` и в снапшот согласования.
   Было «отдельный тикет» — повышено до ПР-1: `ClassifierTariff.*` → `datetime |
   None` с before-парсером + согласовать `TariffSchemaOfferOut.archived_at: str`
   по цепочке. Нарушает критерий приёмки, поэтому не остаточный случай.
4. **Hardening сериализации** — валидация не покрывает `model_copy(update=...)`,
   `model_construct` и присваивания (живых naive-путей нет, но класс обходов
   существует): добавить в `ApiSchema` wrap-сериализатор вторым эшелоном.
5. **Acceptance-тест полного `DealSchema`** — текущий критерий собирается из
   малых схем; полный ответ сделки поймал бы и снапшот, и тарифы (п. 3).
6. **Guard-тест на весь `app/`** — Pydantic-модели вне `app/schemas`
   (`order_processing.py`, `approval_workflow.py`, `api/docs`, роутеры) сейчас без
   datetime-полей, но и без защиты; расширить свип с allowlist осознанных
   исключений.
7. **Guard на naive-дефолты** — `validate_default=False` в Pydantic v2: дефолт
   `field: datetime = datetime(...)` ушёл бы naive мимо валидатора; тест-скан
   `model_fields` всех схем.
8. **Комментарий в `hash_subject_like_stored`** — почему `created_at` снимается
   со всех офферов разом (снапшот атомарен).

## Этап 6. Релиз (~0.5 дня)

- bump версии openapi; ченж-лог фронту: формат строк меняется (breaking по
  букве, фронт — заказчик); перечислить поля, которые были naive, и поля-даты
  с риском смены числа (пока этап 4 не сделан);
- проверка на стенде: `GET /lkm/deals/{id}`, client_price ручки, кейс из
  постановки (`offers[].order.created_at` vs `approval_summary.requested_at`
  больше не расходятся на ±3 ч);
- **согласовать с аналитикой/ОП календарь «даты заказа»** (по ревью, ОТКРЫТО —
  блокер релиза: это семантика `date`, а не форматирование date-time): после
  перехода на UTC до 03:00 МСК `order_date` — «вчерашняя» по московскому
  календарю; дата видна бизнесу (БЗ, биллинг, документы). Если решат МСК —
  правка тривиальна: `ZoneInfo('Europe/Moscow')` вместо UTC в трёх местах,
  конвенция остаётся единой;
- ~~зачистить рукописные примеры дат~~ — ВЫПОЛНЕНО: 15 naive и 21 строка со
  смещениями в `responses.py` приведены к `Z`, регресс-тест
  `test_openapi_examples_are_canonical_utc` обходит `app.openapi()`;
- в ченж-лог фронту включить и **смены типов**, не только формата:
  `archived_at` (тарифы офферов, продукты) `string` → `date-time | null`,
  Bitrix `CLOSEDATE` `datetime | str` → `datetime | null`.

## Нарезка на ПР и оценка

| ПР | Состав | Оценка |
|---|---|---|
| ПР-1 «ядро» | этапы 1 + 2 + 3 + 5 | ~3 дня |
| ПР-2 «семантические даты» | этап 4 | ~1 день + согласования |

Этап 0 — перед ПР-1, результаты вписать в этот документ.

## Чего не делаем и почему

- Не чиним на уровне response-класса FastAPI: к моменту сборки JSON Pydantic уже
  превратил datetime в строку — поздно.
- Не вводим per-field аннотированный тип вместо базового класса: одна точка
  правды в `ApiSchema` дешевле сопровождения, а критерий приёмки в CI ловит
  пропуски.
- Не делаем 422 на naive-ввод — сломало бы текущий фронт; трактовка «naive = UTC»
  зафиксирована в описаниях полей.
