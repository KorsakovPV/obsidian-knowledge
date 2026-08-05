---
project: comet-backend
created: 2026-08-04
updated: 2026-08-05
tags: [project, tickets, deal, contracts, customers, bt03]
---

# DFDEV-2257 Deal Contract Selection

#project #tickets #deal #contracts #customers

Связано: [[Domain Model]], [[API]], [[Architecture]],
[[DFDEV-2052 Deal and Offer Status Contract]].

Постановка: БТ03 «Функционал управления сделками», раздел «Создание новой сделки»
(на 2026-08-04 не заполнен). Аналитика по типам договора взята из ОП
(`order-processing`, `app/services/contracts/contract_helpers.py`).

## Проблема

Сейчас менеджер ищет клиента по ИНН и создаёт сделку, а договор ему никто не даёт
выбрать. Договор проставляется бэком молча и поздно — на имплементации:

`DealService.implement_deal()` (`app/services/deal.py:579`) берёт
`SELLER_PARENT_CLIENT_IDS[deal.seller]`, тянет договоры клиента из customers,
фильтрует по префиксу `TST`, при отсутствии создаёт новый тестовый договор
(`CustomerContractCreateSchema(..., is_test=True)` по умолчанию) и затирает
`deal.contract_id` через `DealUpdateSchema(contract_id=..., updated_by='updated_by')`
(`app/services/deal.py:637-640`).

Следствия:

- тип сделки (тестовая/коммерческая) не виден в карточке до имплементации;
- коммерческую сделку через API создать нельзя — даже переданный `contract_id`
  будет перезаписан TST-договором на имплементации;
- менеджер не управляет ни договором, ни, соответственно, типом сделки.

Тестовый договор при этом остаётся внутренней абстракцией: клиент о нём не знает,
менеджер его не выбирает. Меняется только вход в эту ветку и момент её выполнения.

## Что уже есть в коде (проверено 2026-08-04)

- `DealModel.contract_id` — nullable-колонка (`app/models/deal.py:26`);
  `DealBase.contract_id` (`app/schemas/deal.py:21`) наследуется `DealCreateSchema`
  и `DealCreateDBSchema`, то есть **поле уже принимается на создании и сохраняется**
  (`DealService.create` → `DealCreateDBSchema(**params.model_dump())`,
  `app/services/deal.py:353-355`). Отдельная миграция не нужна.
- `DealSchema.contract_id` уже отдаётся наружу (`app/schemas/deal.py:115`).
- `DealUpdateSchema.contract_id: UUID | None` (`app/schemas/deal.py:81`) — сейчас
  **обязательное** поле апдейта (без default), при `model_config extra="forbid"`.
- Прокси в customers: `GET /api/v1/customers/contracts`
  (`app/api/v1/customers/contract.py`), сервис `CustomersContractsService`,
  фильтры `CustomerContractFilters` — есть `child_client_id`, `parent_client_id`,
  `is_deleted`, `id__in`, `contract_number__in`.
- `SELLER_PARENT_CLIENT_IDS` / `SELLER_PARENT_CONTRACT_IDS` лежат прямо в
  `app/services/deal.py:73-85` (Датафорт: parent client есть, parent contract — `None`;
  Вымпелком: parent contract `DF0432`).

Чего нет:

- `CustomerContractSchema` (`app/schemas/customers/contracts.py:89`) отдаёт только
  базовые поля + `id`. **Ни типа договора, ни `is_deleted` в выдаче нет** — это важно
  для валидации (см. Ticket 4).
- Никакой классификации договоров в comet-backend нет вообще; вся логика — в ОП.

## Тип договора: что берём из ОП

`ContractFullType.get()` (ОП, `app/services/contracts/contract_helpers.py`) считает
две независимые оси. В скоуп входит только `use_case_type`:

| Условие по номеру | Тип | Подпись (`CONTRACT_TYPES_MAPPING`) |
|---|---|---|
| `contract_number.startswith('TST')` | `test` | Тестовый |
| `contract_number == 'DF0001'` (`DATAFORT_INNER_CONTRACT_NUMBER`) | `inner` | Внутренний |
| иначе | `commercial` | Коммерческий |

Вторая ось (`ContractClientType`: прямой / партнёрский / для внешних пользователей API,
считается по `parent_contract_id` и `is_cloud`) в скоуп **не входит**.

### Про `INNER_DF_CONTRACT`

В ОП `app/constants.py:159` есть `INNER_DF_CONTRACT = ('DF0001', 'TST0563', 'DF1302')` —
три «внутренних договора Датафорта». Расхождение с `ContractFullType` кажущееся:

- `TST0563` в `ContractFullType` всё равно попадёт в `test`, потому что проверка
  префикса `TST` идёт первой — добавление кортежа ничего не изменит;
- реальная разница только по `DF1302`: в `ContractFullType` это `commercial`;
- сам кортеж используется исключительно в ITSM-задачах
  (`app/tasks/itsm/contracts.py:217,370,384`) как защита «от внутренних договоров
  не отвязываем услуги и пулы». Это операционный carve-out синхронизации, а не
  бизнес-классификация для UI.

**Рекомендация:** повторяем поведение `ContractFullType` (внутренний — только `DF0001`),
чтобы подпись типа в ЛКМ совпадала с ОП. Список внутренних номеров выносим в
константу (`INNER_CONTRACT_NUMBERS = ('DF0001',)`), чтобы при обратном решении
аналитика правка была однострочной. Требует подтверждения аналитика (см. Открытые вопросы).

## Целевое поведение

Создание сделки:

1. Пришёл `contract_id` → **коммерческая сделка**. Валидируем (принадлежит клиенту
   сделки, не удалён), сохраняем как есть. Ветка TST не выполняется, `contract_id`
   никогда не перетирается.
2. `contract_id` не пришёл → **тестовая сделка**. Логика прежняя: ищем TST-договор
   у родительского клиента продавца, при отсутствии создаём. Отличие — момент:
   договор проставляется при создании сделки, а не на имплементации, поэтому тип
   виден в карточке сразу.
3. После создания договор не меняется — по аналогии с контрагентом (БТ03, раздел
   «Общие данные», требование 5: «После создания сделки контрагента изменить нельзя»).

Тип договора — вычисляемый, не хранимый: отдельной колонки на сделке не заводим,
считаем по номеру договора.

## Порядок этапов

| Этап | Тикеты | Что даёт | Статус |
|---|---|---|---|
| 1 | 1–3 | Фронт может показать список договоров клиента с типом | сделано 2026-08-04 |
| 2 | 4–7 | Выбор договора на создании, неизменяемость, тип в сделке | сделано 2026-08-04 |
| 3 | 8–9 | Легаси-сделки, тесты, документация контракта | сделано 2026-08-04, кроме текста в БТ03 |

**Принято на стенде 2026-08-05** — все сценарии прошли живьём, включая оба продавца,
переиспользование TST-договора и все отказы валидации. Один дефект найден и исправлен
(customers отдаёт 404 на пустую выборку — ломало валидацию и первую сделку клиента без
договоров). Протокол — [[Stand Testing 2026-08-05]].

---

## Ticket 1 — Модуль типов договора в comet-backend

Цель: перенести классификацию `use_case_type` из ОП в comet-backend как чистую
функцию от номера договора.

Что сделать:

- Добавить `app/helpers/contract_types.py` (или `app/services/contracts/`):
  - `ContractUseCaseType(StrEnum)`: `test` / `commercial` / `inner`;
  - `CONTRACT_TYPE_LABELS` — русские подписи из `CONTRACT_TYPES_MAPPING` ОП;
  - `TEST_CONTRACT_PREFIX = 'TST'`, `INNER_CONTRACT_NUMBERS = ('DF0001',)`;
  - `resolve_contract_use_case_type(contract_number: str | None) -> ContractUseCaseType`.
- Порядок проверок ровно как в ОП: сначала префикс `TST`, затем внутренние номера,
  иначе `commercial`.
- Обработать `contract_number is None` / пустую строку (в customers номер nullable
  на уровне create-схемы) — считаем `commercial` или заводим явный fallback.
- Заменить хардкод `contract_number.startswith("TST")` в `implement_deal`
  (`app/services/deal.py:605`) на вызов хелпера.

DoD:

- Юнит-тесты на все ветки: `TST0563` → `test`, `DF0001` → `inner`, `DF1302` →
  `commercial`, `DF0432` → `commercial`, пустой номер → фиксированный результат.
- Ни одного строкового литерала `'TST'` вне модуля.

Статус на 2026-08-04: **сделано.** `app/helpers/contract_types.py` —
`ContractUseCaseType`, `CONTRACT_TYPE_LABELS`, `TEST_CONTRACT_PREFIX`,
`INNER_CONTRACT_NUMBERS = ('DF0001',)`, `resolve_contract_use_case_type`,
`resolve_contract_type_label`, `is_test_contract_number`. Договор без номера →
`commercial`. Хардкод в `implement_deal` заменён на `is_test_contract_number`.
Тесты: `tests/helpers/test_contract_types.py`.

## Ticket 2 — Тип договора в выдаче договоров customers

Цель: `CustomerContractSchema` отдаёт тип договора, чтобы фронт не считал его сам.

Что сделать:

- Добавить в `CustomerContractSchema` (`app/schemas/customers/contracts.py:89`)
  computed-поля: `contract_type: ContractUseCaseType` и
  `contract_type_title: str` (русская подпись).
- Считать через `resolve_contract_use_case_type(self.contract_number)` —
  `is_test` из customers **не тянем**.
- Проверить, что поля появляются во всех ручках прокси
  (`GET /customers/contracts`, `GET /customers/contracts/{id}`, POST/PUT).

DoD:

- Тест API-прокси: в ответе есть `contract_type` и `contract_type_title`.
- OpenAPI обновлён, поля описаны.

Статус на 2026-08-04: **сделано.** В `CustomerContractSchema` добавлены
`computed_field` `contract_type` и `contract_type_title`; поля попадают во все
ручки прокси, задать их снаружи нельзя (`extra="ignore"` + computed).
Тесты: `tests/schemas/test_customers_contracts.py`,
`tests/api/test_customers_contracts_api.py`.

## Ticket 3 — Список договоров клиента для выбора с учётом продавца

Цель: фронт получает все договоры клиента, доступные для выбора, одним запросом,
не зная про родительских клиентов Датафорта и Вымпелкома.

Что сделать:

- Вынести `SELLER_PARENT_CLIENT_IDS` и `SELLER_PARENT_CONTRACT_IDS` из
  `app/services/deal.py:73-85` в `app/helpers/constants.py` рядом с `Seller`
  (их начнут использовать минимум три модуля).
- Добавить в `CustomerContractFilters` параметр `seller: Seller | None`, который
  на бэке резолвится в `parent_client_id` (явно переданный `parent_client_id`
  имеет приоритет либо конфликт — 400, решить в тикете).
- Всегда подставлять `is_deleted=false`, если фильтр не задан явно.
- Договоры отдаём **все** (тестовые, внутренние, коммерческие) — фильтрацию/
  группировку по типу делает фронт, тип уже приходит из Ticket 2.

DoD:

- `GET /api/v1/customers/contracts?child_client_id=<uuid>&seller=datafort` возвращает
  договоры клиента у родительского клиента Датафорта, каждый с типом.
- Тест на резолв `seller` → `parent_client_id` для обоих продавцов.
- Тест, что удалённые договоры не попадают в выдачу.

Статус на 2026-08-04: **сделано.** `SELLER_PARENT_CLIENT_IDS` и
`SELLER_PARENT_CONTRACT_IDS` переехали в `app/helpers/constants.py`
(`app/services/deal.py` импортирует их оттуда). В `CustomerContractFilters`
добавлен `seller`, `model_validator(mode="after")` разворачивает его в
`parent_client_id`, `to_params()` исключает `seller` из запроса в customers.
`is_deleted` теперь по умолчанию `False` — **изменение поведения ручки**: чтобы
получить удалённые договоры, нужен явный `is_deleted=true`.

Конфликт `seller` + чужой `parent_client_id` отклоняется как `BadRequestError`
(HTTP 400 с русским `user_message`), а не как `ValueError`: `ValueError` внутри
`model_validator` модели, подключённой через `Depends()`, FastAPI в 422 не
конвертирует — исключение улетает наружу как 500.

## Ticket 4 — Валидация переданного `contract_id` при создании сделки

Цель: нельзя привязать к сделке чужой или удалённый договор.

Что сделать:

- В `DealService.create` (`app/services/deal.py:353`), если `params.contract_id`
  задан, проверить договор перед вставкой сделки:
  - договор существует;
  - `child_client_id == params.client_id`;
  - договор не удалён.
- **Важно:** `CustomerContractSchema` не отдаёт `is_deleted`, поэтому `get_by_id`
  для проверки удалённости не годится. Рекомендуемый способ — один запрос
  `contracts_service.list(params={'id__in': str(contract_id),
  'child_client_id': str(client_id), 'is_deleted': 'false'})` и проверка, что
  вернулась ровно одна запись. Альтернатива — добавить `is_deleted` в
  `CustomerContractSchema` (требует, чтобы customers его отдавал — проверить).
- Ошибку возвращать как 400/422 с человекочитаемым `user_message`
  («Договор не принадлежит выбранному клиенту» / «Договор удалён»).
- Решить, проверяем ли дополнительно `parent_client_id ==
  SELLER_PARENT_CLIENT_IDS[params.seller]` (см. Открытые вопросы).

DoD:

- Тесты: договор другого клиента → ошибка; удалённый договор → ошибка;
  несуществующий `contract_id` → ошибка; валидный договор → сделка создана
  с этим `contract_id`.
- При ошибке валидации сделка в БД не создаётся.

Статус на 2026-08-04: **сделано.** `DealService._validate_deal_contract` делает один
запрос `CustomerContractFilters(id__in, child_client_id, seller, is_deleted=False)`
и дополнительно проверяет наличие договора в ответе **по id** — так результат не
зависит от того, поддерживает ли customers фильтр `id__in`. Ошибка —
`BadRequestError` → 400, «Договор не найден у выбранного клиента или удалён»
(общая формулировка для «чужой» и «удалён»: различать их наружу незачем).
Проверка продавца включена (открытый вопрос 2 закрыт в пользу строгой проверки):
любой договор из списка выбора её проходит, спотыкается только вызов мимо ЛКМ.
`is_deleted` в `CustomerContractSchema` **не добавлялся** — не понадобился.

## Ticket 5 — Перенос TST-ветки из имплементации в создание сделки

Цель: тип сделки известен сразу после создания.

Что сделать:

- В `DealService.create`, если `contract_id` не передан, выполнить прежнюю логику
  до вставки сделки: `SELLER_PARENT_CLIENT_IDS[seller]` → `contracts_service.list`
  по `parent_client_id` + `child_client_id` + `is_deleted=false` → фильтр по типу
  `test` (через хелпер из Ticket 1) → взять первый либо создать новый через
  `CustomerContractCreateSchema(parent_contract_id=SELLER_PARENT_CONTRACT_IDS[seller],
  ...)`.
- Проставить полученный `contract_id` в `DealCreateDBSchema`.
- Из `implement_deal` (`app/services/deal.py:579-640`) удалить блок поиска/создания
  TST-договора и вызов `deal_crud.update(..., contract_id=...)`.
  Fallback для сделок без договора — Ticket 8.
- Логи сохранить, поменяв контекст с «deal implementation» на «deal creation».
- Учесть порядок операций: договор в customers создаётся **до** вставки сделки,
  поэтому при падении вставки останется «осиротевший» TST-договор. Это безопасно
  (следующая попытка его переиспользует), но должно быть отражено в логе и в тесте.

DoD:

- Сделка без `contract_id` на входе создаётся сразу с TST-договором,
  `implement_deal` его не трогает.
- Сделка с переданным `contract_id` не вызывает ни `list`, ни `create` договоров.
- Тест: второй раз TST-договор не создаётся, переиспользуется существующий.
- Тест обоих продавцов (у Датафорта `parent_contract_id=None`, у Вымпелкома — задан).

Статус на 2026-08-04: **сделано.** Логика вынесена в
`DealService._get_or_create_test_contract`, вход в неё — `_resolve_deal_contract_id`
(договор передан → валидация, не передан → тестовый). Из `implement_deal` блок
поиска/создания договора и `deal_crud.update(contract_id=...)` удалены, на их месте
комментарий о новом месте логики. Тест
`test_implementation_does_not_touch_deal_contract` фиксирует, что имплементация
не дёргает `contracts_service.list/create` и `deal_crud.update`.

**Внимание:** сделки с `contract_id IS NULL`, созданные до этого изменения, теперь
дойдут до имплементации без договора — страховку добавляет Ticket 8, он должен
попасть в тот же релиз.

## Ticket 6 — Запретить смену договора после создания сделки

Цель: требование БТ03 по аналогии с контрагентом.

Что сделать:

- Убрать `contract_id` из `DealUpdateSchema` (`app/schemas/deal.py:81`).
  При `extra="forbid"` попытка передать его в `PATCH/PUT /deals/{id}` даст 422 —
  это и есть требуемый запрет. Заодно чинится текущая аномалия: сейчас
  `contract_id` в апдейте **обязателен** (объявлен без default).
- Починить внутреннее использование: `implement_deal` больше договор не пишет
  (Ticket 5), но если внутренний путь записи всё же нужен (Ticket 8) — завести
  отдельную внутреннюю схему (`DealInternalUpdateSchema`) или метод CRUD
  `set_contract`, не выставляя поле в публичный API.
- Проверить `approval_subject_guard.ensure_deal_mutable(changed_fields=...)`
  (`app/services/deal.py:413`) — из набора изменяемых полей `contract_id` уходит.
- Обновить `update_deal_examples` в `app/api/docs/`.
- Проверить тесты, которые конструируют `DealUpdateSchema(contract_id=...)`
  (`tests/services/test_deal_approval_lifecycle.py:23`).

DoD:

- `PATCH /deals/{id}` с `contract_id` → 422 с внятным сообщением.
- Остальные поля апдейта работают как раньше.
- Фронт уведомлён об изменении контракта ручки.

Статус на 2026-08-04: **сделано.** Поле убрано из `DealUpdateSchema` (на его месте
комментарий с ссылкой на требование БТ03), `PATCH /deals/{id}` с `contract_id`
возвращает 422 — тест `test_deal_update_rejects_contract_id`. Внутренняя схема
записи договора **не заводилась**: после Ticket 5 писать договор после создания
некому. Она понадобится Ticket 8. `update_deal_examples` правки не потребовали —
`contract_id` там и не было. Поправлены тесты, конструировавшие
`DealUpdateSchema(contract_id=None)`.

## Ticket 7 — Тип договора в сделке (карточка и список)

Цель: тип сделки виден и в карточке, и в списке сделок, без хранимой колонки.

Что сделать:

- Добавить в `DealSchema` (`app/schemas/deal.py:114`) поля
  `contract_type: ContractUseCaseType | None` и `contract_type_title: str | None`
  (и, вероятно, `contract_number: str | None` — фронту он нужен для отображения;
  подтвердить с фронтом).
- Источник — номер договора из customers, который в сделке не хранится. Для списка
  сделок нельзя дёргать customers на каждую сделку (N+1 по HTTP):
  **рекомендуемый способ** — собрать уникальные `contract_id` по странице и сделать
  один запрос `contracts_service.list(params={'id__in': ','.join(ids)})`, затем
  раздать по сделкам в `_build_deal_schema`.
- Предусмотреть деградацию: customers недоступен или договор не найден → поля
  `None`, сделка отдаётся, ошибка логируется. Список сделок не должен падать
  из-за внешнего сервиса.
- Рассмотреть короткий кэш номеров договоров (договор у сделки неизменяем —
  Ticket 6, значит кэш безопасен).

DoD:

- `GET /deals/{id}` и `GET /deals` отдают тип договора.
- Тест: список из N сделок делает **один** запрос в customers, а не N.
- Тест деградации при ошибке customers.

Статус на 2026-08-04: **сделано.** В `DealSchema` добавлены `contract_number`,
`contract_type`, `contract_type_title` (все nullable). `DealService._load_deal_contracts`
собирает уникальные `contract_id` и делает один запрос по `id__in`; результат
прокидывается в `_build_deal_schema` через параметр `contracts`. Вызывается из
`create`, `list` и `get_by_id`. При исключении из customers — `logger.exception`
и пустая карта, сделки отдаются с пустым типом. Кэш номеров **не добавлялся** —
преждевременно, договор и так тянется одним запросом на страницу.
Обновлён `deal_example` в `app/api/docs/responses.py`.

## Ticket 8 — Легаси-сделки без договора

Цель: сделки, созданные до изменения, не ломаются на имплементации.

Что сделать:

- В `implement_deal` оставить страховку: если `deal.contract_id is None` —
  выполнить прежнюю TST-ветку (переиспользуя общий метод из Ticket 5) и записать
  договор внутренним путём (Ticket 6). Пометить блок как временный с ссылкой на
  задачу удаления.
- Оценить объём: посчитать сделки с `contract_id IS NULL`, не дошедшие до
  имплементации.
- Решить, нужен ли разовый бэкфилл (скрипт в `scripts/`), который проставит
  TST-договор всем незакрытым сделкам без договора, чтобы страховку можно было
  удалить сразу после прогона.

DoD:

- Тест: сделка с `contract_id=None` имплементируется как раньше.
- В логе видно, что сработал легаси-путь (для мониторинга остаточного объёма).

Статус на 2026-08-04: **сделано.** `DealService._ensure_legacy_deal_contract`
переиспользует `_get_or_create_test_contract`, пишет WARNING «Legacy deal without
contract» и проставляет договор внутренней схемой `DealContractUpdateSchema`
(`updated_by="system"`); `DealCrud.update` принимает её наравне с `DealUpdateSchema`.
Блок помечен как временный.

Бэкфилл — `scripts/backfill_deal_contracts.py`: без флагов только считает
`contract_id IS NULL`, с `--apply` проставляет договоры (есть `--limit`,
идемпотентен, сбой по одной сделке не останавливает прогон). Объём на проде не
измерен: доступа к БД из рабочего окружения нет, скрипт для этого и написан —
первый прогон без `--apply` и даёт число.

## Ticket 9 — Тесты, документация и контракт с фронтом

Что сделать:

- Сквозной интеграционный тест: создание коммерческой сделки (с `contract_id`) и
  тестовой (без) → проверка типа в карточке до имплементации → имплементация не
  меняет договор.
- Обновить `docs/` в репозитории и заметки хранилища ([[API]], [[Domain Model]]).
- Зафиксировать для фронта:
  - `contract_id` опционален на создании и запрещён в апдейте;
  - список договоров: `GET /customers/contracts?child_client_id&seller`;
  - `contract_type` / `contract_type_title` в договоре и в сделке;
  - три значения типа и их русские подписи.
- Отдать формулировки в БТ03, раздел «Создание новой сделки» — он пока пустой.

DoD:

- `make check` и `pytest` зелёные.
- Раздел БТ03 заполнен согласованным текстом.

Статус на 2026-08-04: **сделано, кроме БТ03.** Сквозной набор на реальной БД —
`tests/integration/test_deal_contract_db.py` (7 кейсов: коммерческая и тестовая
сделка, переиспользование TST-договора вторым созданием, отказ по чужому договору,
тип в списке одним запросом, деградация при недоступном customers, легаси-бэкфилл).
Заглушен только customers. Документация: `docs/deal_contract_selection.md` в
репозитории, зеркало [[Deal Contract Selection]] в хранилище, обновлены [[API]] и
[[Domain Model]].

`pytest` целиком (включая `integration`) — 435 passed; `pre-commit` — чисто.

Осталось: заполнить раздел «Создание новой сделки» в БТ03 — это не код, текст ниже
можно отдать аналитику как есть.

## Текст для БТ03, раздел «Создание новой сделки»

1. При создании сделки менеджер выбирает договор из списка договоров клиента.
   Список отдаёт бэк с учётом организатора продаж (Датафорт / Вымпелком).
2. Каждый договор в списке имеет тип: **Тестовый**, **Внутренний** или
   **Коммерческий**. Тип определяется по номеру договора и в ЛКМ не хранится:
   номер с префиксом `TST` — тестовый, `DF0001` — внутренний, остальные —
   коммерческие. Подписи совпадают с Order Processing.
3. Если договор выбран — сделка коммерческая. Договор должен принадлежать клиенту
   сделки и не быть удалённым, иначе создание отклоняется с ошибкой «Договор не
   найден у выбранного клиента или удалён».
4. Если договор не выбран — сделка тестовая. Тестовый договор клиента подставляется
   автоматически, при отсутствии создаётся. Клиенту он не показывается, менеджер им
   не управляет.
5. Тип сделки виден в карточке и в списке сделок сразу после создания, а не после
   передачи в имплементацию.
6. После создания сделки договор изменить нельзя — по аналогии с контрагентом
   (раздел «Общие данные», требование 5).

## Правка после релиза: 404 customers ≠ ошибка (2026-08-05)

Customers на выборку без совпадений отвечает **404**, а не пустым массивом, и
`CustomerContractsAPI.list` превращает это в `CustomersNotFoundError`. Для поиска
договоров это не сбой, а «искать нечего», поэтому оба вызова `contracts_service.list`
в `DealService` (валидация переданного договора и поиск TST-договора) идут через новый
`DealService._list_contracts`, который ловит `CustomersNotFoundError` и возвращает
пустой список; решение принимает вызывающий.

Что чинилось:

- несуществующий `contract_id` отдавал наружу 404 customers с его собственным текстом
  вместо нашей 400 «Договор не найден у выбранного клиента или удалён»;
- клиент вообще без договоров (первая сделка) ломал создание тестовой сделки — вместо
  создания TST-договора улетала ошибка customers.

Тесты: `tests/services/test_deal_contract.py` —
`test_customers_404_is_treated_as_not_found`,
`test_client_without_any_contracts_gets_test_contract`.

---

## Открытые вопросы

1. **`INNER_DF_CONTRACT`.** Повторяем ОП (внутренний — только `DF0001`) или считаем
   внутренними все три номера? Рекомендация — повторять ОП: кортеж из ОП
   используется только как ITSM-carve-out, а по `TST0563` результат всё равно
   совпадает; расходится только `DF1302`. Нужно подтверждение аналитика.
   В коде рекомендация уже реализована (`INNER_CONTRACT_NUMBERS = ('DF0001',)`);
   при обратном решении правка — один кортеж в `app/helpers/contract_types.py`.
2. ~~**Валидация продавца.**~~ Закрыт: строгая проверка включена (Ticket 4).
   Договор из списка выбора её проходит всегда, потому что список фильтруется по
   тому же продавцу. Если аналитика подтвердит существование валидных договоров с
   другим родительским клиентом — убрать `seller` из фильтра в
   `_validate_deal_contract`.
3. ~~**`is_deleted` в customers.**~~ Снят: валидация сделана через `list` с
   `is_deleted=false`, отдельное поле в схеме не понадобилось.
4. ~~**Номер договора во фронте.**~~ Закрыт: `contract_number` добавлен в
   `DealSchema` вместе с типом — фронту не нужен второй запрос ради номера.
5. **Смена клиента у сделки.** `client_id` в `DealUpdateSchema` отсутствует уже
   сейчас, то есть контрагент неизменяем — значит связка «договор ↔ клиент» после
   создания разъехаться не может. Проверить, что это действительно так и нет
   обходного пути через другие ручки.
