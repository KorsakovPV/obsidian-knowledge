---
project: comet-backend
created: 2026-07-23
updated: 2026-07-28
source: docs/lkm_role_model.md
tags: [project, tickets, approval, lkm, rbac]
---

# Sequential KP Approval Tickets

#project #tickets #approval #lkm #rbac

Связано: [[Sequential KP Approval]], [[LKM Role Model]], [[BT02 Role Model Gaps]].

## Граница завершённых и последующих тикетов

Tickets 1–5 фиксируют уже реализованные промежуточные шаги offer-level workflow.
Их не нужно переоткрывать, переписывать задним числом или изменять уже применённые
миграции. Уточнения бизнес-процесса, появившиеся после Ticket 5, реализуются
forward-изменениями в Ticket 6 и следующих тикетах:

- Ticket 6 переводит процесс на один версионируемый deal-level approval, адаптирует
  builder и активацию стадий к данным всех offers сделки;
- Ticket 7 добавляет stage token и transactional outbox;
- Tickets 8–11 добавляют API, аудит, административные операции и API summaries;
- Ticket 12 выполняет cutover legacy данных;
- Ticket 13 адаптирует создание заказов в ОП;
- Tickets 14–15 закрывают интеграционные проверки и удаление legacy contract.

Код и тесты из Tickets 1–5 нужно переиспользовать там, где контракт сохранился.
Несовместимые offer-level части заменяются в соответствующем последующем тикете,
а не отдельными исправлениями завершённых тикетов.

## Ticket 1 — Спроектировать и добавить `approval_stages`

Цель: перейти от одной записи `approval` с JSONB `approvers` к явным стадиям
согласования.

Что сделать:

- Добавить модель `ApprovalStageModel`.
- Добавить Alembic migration `approval_stages`.
- Ввести enum/константы `ApprovalStageCode`, `ApprovalStageStatus`.
- Добавить индексы по `approval_id`, `status`, `assigned_user_id`, `email_token`.
- Сохранить совместимость чтения старых `approval.approvers` на период миграции, если нужна.

DoD:

- Миграция применяется и откатывается.
- У stage есть отдельный token и статус.
- Для одного approval порядок стадий уникален.

Статус на 2026-07-26: модель `ApprovalStageModel`, enum'ы, Pydantic-схемы,
миграция `approval_stages` и модельные тесты есть. Workflow ещё не переведён на stage-level
token'ы.

## Ticket 2 — Реализовать builder маршрута согласования

Цель: строить единый набор стадий для сделки на основании причины согласования и
данных всех её офферов.

Что сделать:

- Вынести построение маршрута из `OfferService._build_approval_create_data`.
- На вход builder принимает deal/reason context со всеми офферами сделки.
- Скидочный маршрут определяется максимальной скидкой среди всех тарифов всех
  офферов сделки.
- На выходе отдаёт ordered list стадий с `stage_code`, `required_permission`, optional/mandatory.
- Для первой итерации поддержать маршруты из БТ02: presale, lead, sales director,
  finance director, product owner, lawyer.

DoD:

- Маршрут не создаёт ненужные стадии.
- Для каждого stage указана required permission.
- Есть тесты на несколько типов маршрутов.

Статус на 2026-07-27: offer-level route builder и сохранение стадий подключены как
промежуточная реализация Ticket 2. Перевод builder-а на единый маршрут сделки по
данным всех offers входит в Ticket 6; Ticket 2 задним числом не переоткрывается.

## Ticket 3 — Добавить в оффер особые условия и передавать их в ОП

Цель: менеджер должен иметь возможность указать в КП текст особых условий, а при
создании БЗ этот текст должен без изменений уходить в Order Processing (ОП).

Контекст:

- У `OfferModel` и схем оффера поля ещё нет.
- В ответе ОП поле уже называется `special_conditions` (`OrderOutSchema`).
- Следующий тикет использует это поле как условие для юридической стадии согласования.

Что сделать:

- Добавить в `offer` nullable текстовое поле `special_conditions` через Alembic migration.
  Nullable нужен для существующих офферов и для КП без особых условий.
- Добавить `special_conditions: str | None = None` в create-, update- и read-схемы
  оффера. В update поле должно очищаться явным `null`; текущая сериализация с
  `exclude_none=True` не должна скрыть это изменение.
- Убедиться, что `POST /api/v1/offers`, `PATCH /api/v1/offers/{offer_id}` и ответы
  `GET` принимают или возвращают поле через схемы оффера. Обновить OpenAPI-примеры,
  если они показывают payload оффера.
- Добавить `special_conditions` в `OrderCreateSchema` и в `OrderService.create()`
  передавать туда значение `offer.special_conditions`. `OrderProcessing.create()`
  сериализует `OrderCreateSchema` и должен отправлять это поле в ОП без преобразований.
- Пустое значение не заменять служебной строкой вроде `"Без особых условий"`:
  `null` остаётся `null`, непустой текст передаётся как введён пользователем.
- Добавить тесты модели/схем, CRUD или сервиса оффера и `OrderService`: проверить
  создание, чтение, обновление, очистку поля и payload, отправляемый в ОП.

DoD:

- Поле `offer.special_conditions` хранится в БД, доступно в create/read/update API
  оффера и может быть очищено через `PATCH` с `null`.
- При создании БЗ в ОП передаётся ровно значение `offer.special_conditions`.
- Существующие офферы и создание оффера без поля продолжают работать.
- Есть миграция и тесты на create/update/clear и передачу в ОП.

## Ticket 4 — Добавить юридическую стадию по особым условиям

Зависимости: Ticket 2 и Ticket 3.

Цель: особые условия должны запускать обязательное согласование юристом в том же
процессе `approval`, что и согласование скидки.

Правило маршрута:

- Особые условия есть, если хотя бы у одного оффера сделки
  `offer.special_conditions` не `null` и после `strip()` содержит хотя бы один символ.
- При наличии особых условий builder добавляет одну обязательную стадию
  `ApprovalStageCode.LAWYER` с `required_permission=Permission.APPROVE_KP_LAWYER`.
- Юридическая стадия не заменяет стадии, выбранные по скидке: итоговый маршрут —
  объединение скидочной цепочки из Ticket 2 и стадии `lawyer`.
- Позиция `lawyer` задаётся отдельной конфигурацией порядка стадий, а не условием
  включения. Начальный порядок: `presale → lead → product_owner → lawyer →
  sales_director → finance_director`, чтобы директора получали документ после визы
  юриста. Это исторический scope Ticket 4; обязательная стадия `product_owner`
  перед `lawyer` добавлена уточнением Ticket 6.
- При `null`, пустой или состоящей только из пробелов строке стадия `lawyer` не создаётся.

Что сделать:

- Расширить builder маршрута из Ticket 2: он сам проверяет `special_conditions`
  всех офферов сделки и не получает отдельный `requires_lawyer` или другой флаг.
- Сохранять результат в том же `approval` и тех же `approval_stages`; не создавать
  отдельный approval-процесс для юриста.
- Использовать существующие `ApprovalStageCode.LAWYER` и
  `Permission.APPROVE_KP_LAWYER`. Не создавать новые stage code или permission.
- Не создавать дубликат стадии: в одном маршруте не более одной `lawyer` stage.
- Добавить unit-тесты builder-а: только скидка, только особые условия, скидка вместе
  с особыми условиями, а также `null`, пустая и whitespace-строка.

DoD:

- Непустые особые условия создают ровно одну mandatory `lawyer` stage с
  `approve_kp_lawyer`.
- Скидочные стадии сохраняются; позиция юридической стадии определяется конфигурацией
  порядка и может меняться независимо от правил включения стадий.
- КП без скидки, но с особыми условиями, не проходит без согласования.
- Пустые особые условия не добавляют стадию юриста.
- Тесты фиксируют состав, порядок и `required_permission` маршрута.

Статус на 2026-07-27: offer-level юридическая стадия реализована как промежуточный
шаг. Проверка особых условий всех offers сделки, единая стадия `lawyer` и
конфигурируемый порядок переносятся в deal-level builder Ticket 6.

## Ticket 5 — Реализовать выбор assignee для стадии

Цель: подключить выбор получателей активной стадии так, чтобы её нельзя было
запустить без согласующего.

Контекст:

- В проекте уже есть `ApprovalStageAssigneeSelector`.
- Единоличные permissions: `approve_kp_sales_director`,
  `approve_kp_finance_director`, `approve_kp_lawyer`.
- Для единоличных permissions источник — только персональные назначения
  `lkm_user_permissions`.
- Для остальных permissions принято групповое правило: стадия отправляется всем
  сохранённым пользователям с effective permission. Первый валидный ответ закрывает
  стадию, последующие ответы обрабатываются идемпотентно.
- `approval_stages.assigned_user_id` / `assigned_email` заполняются только для
  единоличной стадии. Для групповой стадии они остаются `null`, а доступ и список
  получателей определяются по `required_permission`.

Что сделать:

- Исправить типизацию dependency `lkm_user_crud` в `ApprovalStageAssigneeSelector`,
  чтобы `poetry run mypy .` проходил без `type: ignore`.
- Для единоличной permission требовать ровно одного персонального holder-а.
  Ноль или больше одного — доменная ошибка `ApprovalAssigneeSelectionError`.
- Для групповой permission возвращать всех сохранённых пользователей с effective
  permission, полученной из role permissions и personal permissions.
- Подключить selector при активации стадии, а не только вызывать его в unit-тестах.
- Если список пуст или единоличность нарушена, не менять статус стадии и approval,
  не генерировать token и не отправлять email.
- Ошибка API должна содержать пользовательское сообщение и codename permission,
  для которой не найден согласующий.
- Добавить тесты selector-а и orchestration activation для single-holder, group,
  empty result и multiple single holders.

DoD:

- Единоличная стадия имеет ровно одного assignee из `lkm_user_permissions`.
- Групповая стадия отправляется всем текущим holders effective permission.
- При ошибке назначения стадия остаётся `waiting`, approval не переходит в `pending`.
- В ответе нет общей 500: пользователь получает понятную доменную ошибку с permission.
- `poetry run pytest` и `poetry run mypy .` проходят.

Статус на 2026-07-27: selector assignee и промежуточная активация offer-level stage
реализованы. Сам selector переиспользуется; выбор current deal-level approval,
транзакционная активация и одно логическое уведомление по сделке входят в Ticket 6.
Stage token и надёжная доставка уведомлений остаются ответственностью Ticket 7.

## Ticket 6 — Реализовать state machine стадий

Цель: реализовать единый транзакционный процесс согласования сделки и её стадий.

Зависимости: Ticket 1–5.

Ticket 6 является точкой перехода от промежуточной offer-level реализации
Tickets 1–5 к целевому deal-level workflow. Все изменения ранее реализованных
builder-а, юридической стадии и activation orchestration, необходимые из-за новой
границы агрегата, выполняются здесь forward-изменениями. Ticket 3 сохраняет поле
`offer.special_conditions` без изменения его offer-level семантики.

Размер задачи: Ticket 6 меняет границу агрегата, builder и state machine. Перед
разработкой рекомендуется разделить реализацию минимум на три последовательных PR:

1. deal-level schema, versioning и snapshot;
2. deal-level builder и lifecycle draft/rebuild;
3. транзакционная state machine и переключение request endpoint.

Граница процесса:

- Одна сделка имеет не более одного текущего `ApprovalModel`, но может иметь историю
  завершённых/отменённых версий. Все офферы сделки рассматриваются в одном текущем
  approval-процессе и одном письме активной стадии.
- Маршрут по скидке определяется максимальной скидкой среди всех тарифов всех
  офферов сделки. Скидки не суммируются и не усредняются: самый строгий необходимый
  маршрут применяется ко всей сделке.
- Если хотя бы у одного оффера есть непустые после `strip()` особые условия, общий
  маршрут сделки содержит две обязательные стадии: `product_owner`, затем `lawyer`.
- Согласующий получает контекст всей сделки и всех её офферов, чтобы оценивать их
  совместно, включая компенсацию низкомаржинального оффера прибыльными офферами.
- Внутри approval стадии выполняются последовательно по сохранённому `position`.
  На каждой активной стадии отправляется одно письмо по сделке, а не отдельные
  письма по офферам. Для group permission одно и то же письмо отправляется каждому
  holder-у, но относится к одной stage и имеет один stage token.
- Сделка с пустым маршрутом сохраняет current approval version со статусом
  `not_required`, snapshot/route context и без stages. Это фиксирует результат
  расчёта; `start` для такой версии не выполняется.

Версии и предмет согласования:

- Нельзя делать обычный `UNIQUE(approval.deal_id)`: он запретит хранить историю
  повторных согласований сделки. Добавить `version`, `is_current`,
  `superseded_at`, `workflow_version` и ограничения:
  `UNIQUE(deal_id, version)` плюс partial unique index по `deal_id WHERE is_current`.
- Новые staged approvals имеют `offer_id = NULL`; legacy `offer_id` временно
  сохраняется nullable для чтения старой истории до cleanup migration.
- Approval хранит immutable `subject_snapshot` сделки и всех её offers, а также
  `subject_hash`. Snapshot содержит значения, по которым принято решение: offer id,
  product id, тарифы/цены/скидки, технические параметры, особые условия и необходимые
  deal-level поля.
- Хранить `route_context`: причины включения стадий, максимальную скидку и источник
  этой скидки, offers с особыми условиями, версию builder-а и версию `STAGE_ORDER`.
  Legacy `reason_type` не подходит для одновременных причин и остаётся только для
  совместимости.
- Пока approval `draft`, изменение сделки пересобирает snapshot и route в текущей
  версии. После `start` snapshot и route неизменяемы. Новый цикл создаёт следующую
  версию approval, а предыдущую помечает `is_current=false`.
- Отсутствие approval не является доказательством `not_required`: рассчитанный
  пустой маршрут представлен current version `status=not_required`.

Правила построения и порядка:

- До реализации аналитик должен утвердить полную матрицу trigger-ов, включая
  product owner и комбинации нескольких причин. Builder не выбирает одну ветку по
  scalar `reason_type`: он независимо вычисляет все причины и объединяет стадии.
- Builder получает deal со всеми актуальными offers и формирует единый набор стадий.
- Определение состава стадий и их порядок разделены. Условия включения не должны
  зависеть от позиции стадии.
- Порядок задаётся отдельной конфигурацией `STAGE_ORDER`. Начальное значение:
  `presale → lead → product_owner → lawyer → sales_director → finance_director`.
- `lawyer` должен быть перемещаемым изменением конфигурации без изменения скидочных
  и юридических правил builder-а.
- После создания approval итоговый порядок фиксируется в `approval_stages.position`;
  изменение конфигурации влияет только на новые или явно пересобранные маршруты.

Что сделать:

- Перевести модель с offer-level на deal-level approval:
  - `approval.deal_id` становится владельцем процесса;
  - удалить обязательную принадлежность процесса одному `offer_id`;
  - оставить `DealModel.approvals` как ordered history и добавить явное
    `current_approval`, не вычисляемое через первый элемент списка;
  - schema должна поддерживать legacy rows и expand/contract cutover из Ticket 12.
- Делать новую forward Alembic revision поверх уже применённой migration
  `approval_stages`; не переписывать существующую revision.
- Перенести создание/пересборку approval из `OfferService` в deal-level orchestration.
  Создание, изменение или удаление любого оффера должно пересобирать draft-маршрут
  всей сделки. Изменение оффера при pending обрабатывается через cancel/rebuild policy.
- Переделать route builder: вычислять максимальную скидку по всем офферам, добавлять
  `lawyer` по особым условиям любого оффера, объединять несколько причин маршрута,
  применять `STAGE_ORDER` и возвращать `route_context`.
- Создать `ApprovalWorkflowService` с операциями `start`, `approve`, `reject`,
  `skip`, `reassign`, `cancel`.
- Добавить в `approval_stages` persisted-поле `is_optional BOOLEAN NOT NULL
  DEFAULT false`; обновить модель, migration и Pydantic-схемы. Route builder должен
  сохранять в БД optional/mandatory признак.
- Все переходы выполнять в одной DB-транзакции и одной переданной `AsyncSession`.
  Запрещены CRUD-методы, открывающие вложенные независимые sessions внутри workflow.
  Использовать единый lock order: deal → current approval → active stage.
- Создание и пересборку текущей версии сериализовать блокировкой строки deal, чтобы
  параллельные create/update/request не создали два current approvals.
  При переносе offer между deals блокировать обе сделки в детерминированном порядке.
- Добавить partial unique index, запрещающий более одной стадии `pending` внутри
  одного approval.
- `start`: разрешён только для `approval=draft`; активирует первую `waiting`
  стадию. `POST /api/v1/deals/{deal_id}/approval/request` запускает единственный
  draft approval сделки. При ошибке выбора assignee ничего не меняет и не отправляет
  письмо. Перед стартом повторно вычислить `subject_hash`; несовпадение требует
  rebuild или понятную conflict-ошибку.
- `approve`: разрешён только для `pending`; сохраняет `approved`, `decision=approve`,
  `decided_at`, `decision_comment`, инвалидирует token и активирует следующую
  `waiting` stage.
- Если после approve/skip стадий больше нет, установить
  `approval.status=answered`, `approval.decision=approve`, `decided_at`.
- Если решение текущей стадии принято, но у следующей mandatory stage не найден
  assignee, не откатывать уже принятое решение: сохранить stage/event, оставить
  следующую stage `waiting`, перевести approval в `blocked` с
  `blocked_permission/blocked_at` и не создавать token/outbox. Операция
  `retry_activation` после исправления permissions идемпотентно продолжает процесс.
- `reject`: переводит активную стадию в `rejected`, сохраняет решение и comment,
  остальные `waiting` стадии — в `canceled`, approval — в
  `answered/reject`.
- `skip`: переводит стадию в `skipped` с actor/time/reason и активирует следующую.
  Skip разрешён только для `is_optional=true`; mandatory stage, включая `lawyer`,
  нельзя пропустить. Авторизация skip реализуется в Ticket 10.
- `reassign`: оставляет стадию `pending`, меняет assignee, инвалидирует старый token
  и выпускает новый. Reassign применим только к single-holder stage; для group stage
  назначение определяется permission. Авторизация реализуется в Ticket 10.
- `cancel`: переводит approval и все `waiting/pending` стадии в `canceled`,
  инвалидирует активные token'ы.
- Повторное действие над уже закрытой стадией не меняет состояние и возвращает
  предсказуемый результат `already_processed`.
- Email context активной стадии содержит данные сделки и ordered list всех её
  офферов из frozen `subject_snapshot`, а не повторно читает изменяемые live rows.
  В одном письме должны быть видны как минимум продукт, цены, скидка и особые
  условия каждого оффера.
- State machine не отправляет SMTP внутри DB-транзакции. Она атомарно сохраняет
  переход и durable notification intent; transactional outbox и доставка
  реализуются в Ticket 7.
- Все v2 writes и фоновые consumer-ы закрыть feature flag
  `approval_workflow_v2_enabled`. Включение разрешено только после Ticket 7 и Ticket 9.

DoD:

- У сделки не более одного актуального approval-процесса; отдельные approval для
  каждого оффера больше не создаются.
- Пустой маршрут представлен сохранённой current version `not_required`, а не
  двусмысленным отсутствием approval.
- Повторное согласование создаёт новую version и не удаляет историю предыдущего
  процесса; partial unique index не допускает две current versions.
- Решение относится к immutable snapshot; route context объясняет, почему каждая
  стадия попала в маршрут.
- Максимальная скидка любого оффера определяет единый скидочный маршрут сделки.
- Особые условия любого оффера добавляют ровно одну mandatory stage `product_owner`
  и ровно одну mandatory stage `lawyer`.
- Для одной активной стадии формируется одно письмо с контекстом всей сделки и всех
  офферов; отдельные письма по офферам не создаются.
- Порядок `lawyer` меняется через конфигурацию `STAGE_ORDER`; директора получают
  документ после юридической визы при начальной конфигурации.
- В одном approval не может быть больше одной `pending` stage.
- Невалидные переходы не записываются в БД.
- Изменение состояния, token/notification intent и audit event выполняются одной
  транзакцией; rollback любой части откатывает всё.
- Полная цепочка approve завершается агрегатным `answered/approve`.
- Ошибка assignee следующей стадии даёт восстанавливаемый `blocked`, не теряет
  предыдущее решение и не отправляет письмо в никуда.
- Reject закрывает процесс и отменяет необработанные стадии.
- Cancel и repeated decision инвалидируют возможность сдвинуть процесс старым token.
- Тесты покрывают start, approve chain, reject, skip, reassign, cancel,
  no-next-stage и concurrent/repeated decision.

Статус реализации на 2026-07-27:

- добавлена forward migration deal-level полей, version/current constraints,
  `is_optional` и partial unique index одной pending stage;
- builder переведён на все offers сделки; особые условия любого offer добавляют
  mandatory `product_owner → lawyer`;
- добавлены immutable snapshot/hash, route context и lifecycle draft/rebuild;
- добавлен `ApprovalWorkflowService` с переходами start/approve/reject/skip/reassign/
  cancel/retry_activation и recoverable `blocked`;
- request endpoint переключён на current deal-level approval, SMTP внутри workflow
  не вызывается;
- v2 writes закрыты `approval_workflow_v2_enabled`, по умолчанию флаг выключен;
- persisted stage token/transactional outbox выполняются в Ticket 7, audit events —
  в Ticket 9.

## Ticket 7 — Перевести email approval на stage token

Цель: email-ответ должен закрывать конкретную стадию, а не весь approval.

Зависимости: Ticket 6.

Контракт письма:

```json
{
  "deal_id": "uuid",
  "deal_number": 123,
  "stage_id": "uuid",
  "email_token": "opaque-secret",
  "decision": "approve | reject"
}
```

Источник истины — запись истории stage token. Остальные идентификаторы проверяются
на соответствие найденной стадии и используются для понятной диагностики.

Принятые решения:

- В `lkm_users` добавляется nullable `email`, валидируемый через `EmailStr`. Для legacy users разрешён fallback на `ad_login` только когда он является валидным email; иначе activation блокируется понятной доменной ошибкой. Администратор может задать или очистить delivery email отдельной ручкой.
- Начальный `stage_token_ttl` — 14 календарных дней. Начальный SLA всех stage codes — 3 календарных дня; настройки разделены по stage code и могут меняться без изменения workflow.
- Raw token шифруется в outbox через Fernet. При включённом workflow v2 обязателен отдельный production secret `APPROVAL_WORKFLOW_OUTBOX_ENCRYPTION_KEY`; token hash и raw token запрещено логировать или возвращать API.
- Fernet key валидируется при загрузке settings; с некорректным ключом v2 не запускается. Повреждённый ciphertext считается permanent delivery error: delivery отменяется с `InvalidEncryptedToken` и не блокирует остальную outbox batch.
- Permanent email errors (`invalid_token`, `payload_mismatch`, `permission_denied`, `token_expired`) помечают IMAP message как seen. Только transient infrastructure/database errors остаются для повторной обработки.

Что сделать:

- Добавить `approval_stage_tokens`: `id`, `stage_id`, уникальный `token_hash`,
  `issued_at`, `expires_at`, `used_at`, `invalidated_at`. В email передавать
  криптографически случайный raw token достаточной энтропии; lookup выполнять по
  hash с constant-time comparison.
- Stage хранит ссылку на текущий active token либо получает его отдельным запросом.
  Старые token rows не удалять: это позволяет отличать `already_processed` от
  неизвестного/поддельного token после закрытия или reassign.
- При активации стадии выпускать token и устанавливать `requested_at`/`due_at`
  согласно конфигурации `stage_token_ttl` и SLA по stage code. Значения обязательны
  в production settings и инъецируются в тестах.
- Истёкший token не закрывает stage; stage остаётся pending. Операция `resend`
  инвалидирует старый token, выпускает новый и создаёт новую generation
  notifications без повторной активации stage.
- Для единоличной стадии отправлять письмо `assigned_email`; для групповой —
  каждому holder `required_permission` одинаковое логическое письмо с одним stage
  token. Не создавать отдельную stage на каждого recipient.
- Зафиксировать канонический delivery email пользователя. Если `ad_login` не
  гарантированно является email, добавить отдельное валидируемое поле. При разборе
  входящего `From` использовать `parseaddr(...)[1]`, trim и casefold; не сравнивать
  display-name строку целиком.
- Добавить transactional outbox из двух уровней:
  - `approval_stage_notifications` — одна логическая notification на generation:
    stage/token, immutable payload snapshot и encrypted raw token;
  - `approval_stage_notification_deliveries` — N recipients: user/email, status
    (`pending/sent/failed/canceled`), attempts, `next_attempt_at`, `sent_at`,
    `last_error`.
- Уникальность `(notification_id, canonical_recipient_email)` защищает от дублей.
  Raw token никогда не логировать и не возвращать API; encrypted secret в outbox
  удалить/редактировать после завершения всех deliveries.
- Workflow в одной транзакции сохраняет stage/token/outbox rows. Отдельный worker
  отправляет после commit, использует `FOR UPDATE SKIP LOCKED`, retry/backoff и
  идемпотентную обработку. SMTP не вызывается внутри workflow-транзакции.
- Email строить только из `approval.subject_snapshot`; все recipients одной stage
  получают одинаковую версию документа.
- Расширить `ReceivedApprovalDecisionSchema` полем `stage_id`.
- Добавить CRUD-запрос стадии по token с eager load approval/deal и всех offers сделки.
- `DealService.receive_mail()` после валидации payload должен вызывать только
  `ApprovalWorkflowService.approve/reject`, не менять approval напрямую.
- Принимать решение только для стадии `pending`, с неистёкшим token и совпадающими
  `deal_id`, `deal_number`, `stage_id`.
- Для единоличной стадии `ReceivedEmailSchema.from_email` должен совпадать с
  `assigned_email`. Для групповой стадии отправитель должен быть сохранённым LKM user
  с effective `required_permission` на момент обработки ответа.
- После approve/reject/skip/cancel помечать token использованным/инвалидированным,
  но сохранять запись. После reassign выпускать другой token и инвалидировать
  notification/deliveries старой generation.
- Ответ по закрытой стадии или старому token считать `already_processed`; он не
  должен повторно отправлять следующую стадию.
- Удалить использование approval-level `email_token` из нового workflow. Legacy
  обработка остаётся только для записей без stages до выполнения Ticket 12.
- Во всех публичных response schemas, OpenAPI examples и логах исключить raw/hash
  approval/stage token. Наличие stages не является discriminator workflow:
  переключение выполняется только по persisted `workflow_version`.

DoD:

- Email однозначно закрывает только указанную stage соответствующей сделки.
- Старый, истёкший или не совпадающий с payload token не меняет БД.
- Повторный старый token находится по истории и возвращает `already_processed`,
  неизвестный token не раскрывает существование approval.
- Первый ответ на групповую стадию закрывает её, остальные ответы идемпотентны.
- Ответ с email пользователя, которому стадия недоступна, не меняет БД.
- После решения автоматически отправляется только следующая стадия общего approval сделки.
- Сбой SMTP не теряет уведомление: outbox повторяет доставку без повторной активации
  stage и без дублей одному recipient.
- Частичная group-доставка видна по delivery rows; закрытие stage отменяет ещё не
  отправленные deliveries, а уже отправленные ответы остаются идемпотентными.
- Тесты покрывают valid decision, stale/expired token, payload mismatch,
  repeated group reply и переход к следующей стадии.

## Ticket 8 — API для задач согласующего

Цель: дать фронту список стадий, ожидающих решения текущего пользователя, и единый
API-вход в ту же state machine, которую использует email consumer. API не принимает
email token и не создаёт отдельную логику переходов.

Зависимости: Ticket 6 и Ticket 7.

Принятые архитектурные решения:

- Identity берётся только из trusted principal (`request.state.lkm_user`),
  установленного auth middleware. `user_id`, email или login из query/body не
  принимаются и не используются для авторизации.
- Список задач фильтруется в SQL. Single-holder stage доступна только при
  `assigned_user_id=current_user.id`; group stage с `assigned_user_id IS NULL` —
  при наличии `required_permission` в effective permissions principal-а.
- Decision выполняется в одной workflow-транзакции и под существующим lock order
  `deal → current approval → active stage`. Router guard не считается достаточной
  защитой: после lock single assignee или effective group permission читаются и
  проверяются повторно по актуальному состоянию БД.
- Идемпотентность API-команды хранится отдельно от будущего audit log в таблице
  `approval_stage_decision_requests`. Ключ имеет область
  `(stage_id, actor_user_id, idempotency_key)`, поэтому одинаковая строка ключа у
  разных пользователей или стадий не конфликтует.
- Первая команда сохраняет decision и полный стабильный response. Повтор того же
  `Idempotency-Key` с тем же decision возвращает сохранённый response без нового
  перехода. Тот же ключ с противоположным decision возвращает
  `decision_conflict`. Изменение comment при повторе не создаёт новую команду:
  источником истины остаётся результат первого запроса.
- `approval_stage_decision_requests` обеспечивает command replay после restart и
  между несколькими application workers. Ticket 9 отдельно добавляет immutable
  transition events; event не заменяет сохранённый API response.

Что сделать:

- Добавить `GET /api/v1/approval-stages/pending`.
- Одновременно фильтровать `approval.status=pending` и `stage.status=pending`.
- Сортировать по `due_at ASC NULLS LAST`, затем `requested_at ASC NULLS LAST`, затем
  `stage_id ASC`. Поддержать offset pagination: `limit` default `50`, maximum `100`,
  `offset` default `0`. `stage_id` является обязательным tie-breaker-ом.
- Response item должен содержать `stage_id`, `stage_code`, `required_permission`,
  `status`, `requested_at`, `due_at`, `approval_id`, `deal_id`, `deal_number`,
  `deal_title`, `client_id`, `approval_version`, `subject_hash` и краткий ordered
  list offers/products только из immutable `approval.subject_snapshot`.
- Не читать изменяемые live offers для построения ответа. Не возвращать raw token,
  token hash, `email_token` или данные notification outbox.
- Добавить `POST /api/v1/approval-stages/{stage_id}/decision` с обязательным header
  `Idempotency-Key` длиной 1–255 символов и body
  `{"decision": "approve | reject", "comment": "..."}`. Comment nullable,
  maximum 4000 символов.
- Endpoint вызывает `ApprovalWorkflowService.approve/reject`, используемый email
  consumer-ом, и возвращает `stage_id`, `approval_id`, `outcome` и стабильный code.
- Поддержать codes: `processed`, `already_processed`, `decision_conflict`,
  `invalid_status`, `not_assigned`, `permission_denied`.
- Для нового запроса сначала проверить persisted idempotency result, затем под lock
  проверить status и доступ actor-а. Закрытая stage не выполняет переход повторно;
  допустимый повтор исходной команды возвращает `already_processed` или ранее
  сохранённый результат.
- После decision/skip stage автоматически исчезает из pending query. После reassign
  single-holder stage исчезает у прежнего assignee и появляется у нового. Group
  stage остаётся permission-based и через reassign не изменяется.
- Добавить API, CRUD, model и workflow tests для single-holder, group permission,
  постороннего пользователя, закрытой стадии, reassign visibility, стабильной
  пагинации, replay и противоположного решения с тем же idempotency key.

Отложенные решения:

- TODO (forward ticket после Ticket 8): snapshot Ticket 6 пока не гарантирует
  `offer.created_at`. Offers сортируются по `(created_at, id)`, когда timestamp есть,
  и стабильно по `id` для legacy snapshot. Уже frozen snapshots не переписывать.
- TODO (Ticket 9): email/API используют общую state machine, но одинаковые audit
  events обоих каналов появляются только после добавления
  `approval_stage_events`. API idempotency table из Ticket 8 при этом сохраняется.

DoD:

- Пользователь видит только доступные ему pending stages; фильтр одновременно
  ограничивает status approval и stage.
- Закрытые стадии и любые token-поля отсутствуют в pending response.
- Порядок и offset pagination детерминированы обязательным `stage_id` tie-breaker-ом.
- Decision повторно авторизуется внутри locked workflow-транзакции и использует те
  же переходы, что email decision.
- Persisted idempotency переживает restart/несколько workers: повтор возвращает
  исходный response, противоположное решение даёт `decision_conflict`.
- Невалидный actor не меняет stage, approval, token или outbox.
- Тесты фиксируют permissions/isolation, pagination, visibility после закрытия и
  reassign, а также idempotent replay/conflict.

## Ticket 9 — Добавить аудит переходов workflow

Цель: хранить неизменяемую ordered историю каждого фактического перехода стадии.

Зависимости: Ticket 6–8. Feature flag v2 нельзя включать, пока state machine не
пишет audit events из этого тикета.

Принятые архитектурные решения:

- Добавить `approval_stage_events`: `id`, `stage_id`, `approval_id`,
  `actor_user_id` nullable для системного действия, `action`, `old_status`,
  `new_status`, `reason`, `metadata JSONB`, `sequence`, `source`, `source_id`,
  `created_at`.
- `source`: `system | api | email`. Для первичного API event
  `source_id=Idempotency-Key`, для email event — `ReceivedEmailSchema.id`.
- Гарантировать partial unique index `(approval_id, stage_id, actor_user_id,
  source, source_id) WHERE source_id IS NOT NULL`. Область соответствует API-команде
  конкретного actor-а по конкретной stage. Отдельное поле `idempotency_key` в event не хранить:
  persisted response API остаётся в `approval_stage_decision_requests` Ticket 8.
- Одно внешнее решение может создать несколько events. Все изменённые им стадии
  сохраняют одинаковые actor/source/source_id и различаются stage_id; причинный source
  дополнительно сохраняется в metadata. Полностью автоматические переходы имеют
  `source=system` и `source_id=null`.
- `resend` не создаёт stage event, потому что status стадии не меняется. История
  resend хранится в token и notification generations Ticket 7.

Что сделать:

- Ввести actions: `started`, `approved`, `rejected`, `skipped`, `reassigned`,
  `canceled`, `superseded`, `activation_failed`, `activation_retried`.
- Гарантировать `UNIQUE(approval_id, sequence)`. Историю сортировать по sequence;
  sequence вычислять только под существующим lock approval.
- Добавить индексы `(stage_id, created_at)` и `(approval_id, created_at)`.
- Каждая операция `ApprovalWorkflowService` создаёт events в той же транзакции,
  что stage/token/outbox transition.
- `approve/reject` сохраняют actor, comment и source. Автоматическая активация
  следующей стадии создаёт отдельный system event.
- `reject/cancel` создают отдельный `canceled` event каждой реально изменённой
  waiting/pending стадии.
- `skip` сохраняет actor и reason. `reassign` сохраняет old/new user id и email в
  metadata. `retry_activation` создаёт `activation_retried`; recoverable ошибка
  назначения следующей стадии — `activation_failed`.
- Lifecycle supersede создаёт `superseded` для каждой стадии, статус которой реально
  изменился на canceled. Завершённые стадии не получают фиктивного события.
- Во всех metadata хранить `schema_version`, approval version и subject hash.
- Event rows не обновлять и не удалять отдельно от cascade удаления всего approval.
  Routine rebuild/supersede сохраняет историю завершённых версий. Hard delete
  допускается только отдельной retention policy всей сделки.
- Добавить read schema и ordered `events` в детальный `ApprovalSchema`; CRUD должен
  eager-load history до закрытия async session.
- Добавить model/schema/workflow/lifecycle/email tests для каждого action,
  source/source_id, sequence, metadata и идемпотентного повтора.

DoD:

- Для каждого фактического перехода есть ровно один event; repeated decision не
  создаёт второй event.
- Два конкурентных действия сериализованы approval lock и защищены уникальностью
  `(approval_id, sequence)`.
- API/email events содержат actor и source id для каждой изменённой stage и не
  конфликтуют благодаря stage/actor scope partial unique index.
- По reassign видны прежний/новый assignee, по skip — actor/reason.
- Reject/cancel/supersede оставляют отдельную историю каждой изменённой стадии.
- Изменение stage, token/outbox и event атомарны в одной `AsyncSession` transaction.
- Детальный response approval возвращает события строго по sequence.

## Ticket 10 — API для skip и reassign

Цель: поддержать отсутствие согласующего без ручной правки БД.

Зависимости: Ticket 7 и Ticket 9.

Принятые permissions:

- `skip_kp_approval_stage`;
- `reassign_kp_approval_stage`.

Permissions добавить в `Permission`; они синхронизируются с `lkm_permissions`.
Проверки выполнять только по permissions, без проверки имени роли. Escalation и
автоматический skip в этот тикет не входят.

Bootstrap policy: миграция добавляет permissions в справочник и выдаёт их только
роли `admin` (если аналитик не утвердит personal-only). Обычным согласующим права
skip/reassign автоматически не выдаются.

Что сделать:

- Добавить `POST /api/v1/approval-stages/{stage_id}/skip` с body
  `{"reason": "непустая строка"}`.
- Добавить `POST /api/v1/approval-stages/{stage_id}/reassign` с body
  `{"user_id": "uuid", "reason": "непустая строка"}`.
- Skip/reassign разрешены только для `pending` stage и пользователя с
  соответствующей permission. Каждый mutation request принимает idempotency key.
- Skip разрешён только для optional stage. Mandatory stages нельзя пропустить даже
  при наличии `skip_kp_approval_stage`.
- Reassign разрешён только для single-holder stage и должен менять assignee на
  другого пользователя. Для group stage возвращать HTTP 409 с code
  `reassign_not_supported_for_group_stage`.
- Новый assignee должен существовать в `lkm_users` и иметь effective `required_permission`.
  Для единоличной permission он также должен быть её единственным персональным holder.
  Практический порядок: администратор сначала меняет personal holder permission,
  затем вызывает reassign на нового единственного holder-а; старый пользователь
  после смены permission не может принять решение.
- Skip вызывает state machine, сохраняет `skipped_by_user_id`, `skipped_at`,
  `skip_reason`, создаёт audit event и активирует следующую stage.
- Reassign меняет assignee, создаёт audit event и новое письмо/token. Старый token
  и его неотправленные notifications инвалидируются в той же транзакции; новый
  token/outbox создаются атомарно.
- Для 403, stage not found, invalid status, invalid target и empty reason добавить
  отдельные пользовательские ошибки со стабильными machine-readable codes.
- Публичные `retry_activation` для `blocked` approval и `resend` для истёкшего
  token отложены до отдельного административного API-тикета. До утверждения
  отдельных permissions и audit-контракта Ticket 10 не публикует эти endpoints;
  внутренние workflow operations сохраняются для будущего orchestration.
- Добавить API и service tests.

DoD:

- Skip и reassign нельзя вызвать без соответствующей permission.
- Skip без reason и reassign неподходящему пользователю отклоняются без изменения БД.
- Mandatory stage нельзя skip, group stage нельзя reassign.
- Reassign инвалидирует старый token, отправляет новый и сохраняет old/new assignee.
- Skip активирует следующую стадию и остаётся отличимым от approve.
- Все действия отражены в `approval_stage_events`.

## Ticket 11 — Пересобрать actions и статусы API под stage workflow

Цель: backend response и action availability должны отражать единый deal-level
stage workflow, а не первое legacy approval из списка.

Зависимости: Ticket 6–10.

Что сделать:

- Удалить использование `DealModel.approval`, возвращающего первый approval.
  Возвращать `current_approval` отдельно от ordered `approval_history`.
- Для сделки вычислять `approval_summary`: `approval_id`, aggregate `status`,
  `version`, `subject_hash`, `decision`, `active_stage`, `completed_stages`,
  `skipped_stages`, `canceled_stages`, `stage_counts_by_status`, `total_stages`,
  notification delivery summary. `completed_stages` включает только
  `approved/rejected/skipped`; canceled показываются отдельно.
- `active_stage` содержит `stage_id`, `stage_code`, `required_permission`,
  `assigned_user_id`, `assigned_email`, `requested_at`, `due_at`, но не token.
- Для deal возвращать один `approval_summary` со статусом:
  `not_required | draft | pending | blocked | approved | rejected | canceled`.
- Обновить `deal_actions` / `offer_actions`: изменение любого оффера запрещено при
  pending approval сделки; request approval доступен для единственного draft approval.
- Для draft изменение deal/offer пересобирает snapshot и route под блокировкой deal.
  Для pending изменение полей, входящих в snapshot, возвращает 409. Для
  approved/rejected/canceled изменение создаёт новую current version либо
  `not_required`, сохраняя предыдущую в history.
- Одинаково применять invalidation policy к offer endpoints и deal fields, влияющим
  на предмет согласования; нельзя оставить обходной update endpoint.
- Доступность `skip/reassign` рассчитывать по active stage и permissions текущего
  пользователя.
- Удалить token-поля из всех публичных approval/offer/deal schemas, а не только из
  pending-list. Для истории добавить отдельный paginated endpoint без bearer secrets.
- Offer response больше не содержит собственный approval; он ссылается на
  deal-level summary/version.
- Добавить response schemas и tests для сделки с несколькими offers в одном процессе.

DoD:

- API не выбирает случайный/первый approval сделки.
- История версий доступна отдельно, а summary всегда относится к current version.
- Frontend получает active stage и состав всех offers без дополнительных запросов.
- Actions соответствуют stage state и effective permissions.
- Router-level availability не заменяет повторную авторизацию command handler-а
  внутри транзакции.
- Тесты покрывают сделки с несколькими offers, pending, approved, skipped, rejected
  и canceled.

## Ticket 12 — Миграция текущих approvals и backward compatibility

Цель: безопасно перейти от текущей модели к stage workflow.

Зависимости: Ticket 6–11.

Принятая стратегия:

- Перед cutover временно запретить новые отправки на согласование.
- На момент миграции в БД не должно быть `approval.status=pending`. Preflight
  проверка прерывает операцию и выводит ID таких approvals; их нужно дождаться или
  закрыть операционной командой. На текущий `revoke_approval()` полагаться нельзя,
  пока он остаётся заглушкой.
- `draft` approvals группируются по deal и пересобираются builder-ом в один
  deal-level approval по актуальным данным всех offers.
- `answered` и `canceled` approvals не получают искусственных stages и остаются
  readonly legacy history.
- Признак нового workflow — только `workflow_version=2`, а не наличие stages:
  transitional legacy rows уже могут иметь stages, а v2 `not_required` version
  законно не имеет стадий.

Что сделать:

- Использовать expand/migrate/contract deployment: сначала nullable/new columns и
  dual-read код, затем preflight/data migration, после проверки — переключение
  writes. Не добавлять новые NOT NULL/partial unique ограничения до backfill.
- На время cutover установить persisted maintenance flag и взять PostgreSQL advisory
  lock, которые проверяются внутри write-транзакций. Остановить request approval,
  IMAP consumer и outbox worker; одного HTTP-флага недостаточно.
- Добавить preflight-команду, проверяющую pending approvals и печатающую их
  `approval_id`, `deal_id`.
- Добавить идемпотентный data migration/service command, объединяющий draft approvals
  по deal и пересобирающий единый маршрут. Повторный запуск не должен создавать дубликаты.
- Для draft с пустым маршрутом создать current v2 version `not_required` и увеличить
  счётчик `approval_not_required` в отчёте. Legacy rows физически не удалять до
  подтверждённого cutover; помечать superseded.
- Legacy answered/canceled rows пометить `workflow_version=legacy`,
  `is_current=false` и сохранить `offer_id`. Для их отображения реализовать
  совместимый deal-level aggregate; отсутствие current staged approval нельзя
  автоматически трактовать как `not_required`.
- Для каждого deal формировать dry-run reconciliation: legacy ids/statuses,
  выбранная current version, max discount/source, offers с особыми условиями,
  новый route и subject hash.
- Если trigger product owner или другая причина не восстанавливается из актуальных
  данных, migration не угадывает её: помечает deal `manual_review_required` и
  исключает его из cutover до ручного решения.
- В read schemas сохранить отображение answered/canceled legacy approvals без stages.
- После успешного cutover перестать создавать и читать `approval.approvers` и
  approval-level email token в новом workflow. Физическое удаление legacy columns
  вынести в отдельную cleanup migration после стабилизации.
- Описать runbook: backup, запрет новых request, preflight, migration, проверки
  количества, deploy, включение request, rollback.
- Rollback старого offer-level кода разрешён только до включения v2 write traffic.
  После появления deal-level rows rollback означает forward-compatible v2 release
  либо восстановление DB snapshot; старый код не умеет nullable `offer_id` и новые
  current versions.

DoD:

- Миграция не теряет старые решения.
- Deal-level summary после cutover сохраняет смысл legacy approved/rejected history
  и не превращает её в `not_required`.
- Cutover не начинается при наличии pending legacy approvals.
- Все оставшиеся draft approvals имеют корректный маршрут без дублей; удалённые
  пустые маршруты учтены в `approval_not_required`.
- Answered/canceled legacy approvals продолжают читаться.
- Повторный запуск migration идемпотентен.
- Есть проверенный dry-run, rollback и операционный runbook.
- Cutover защищён от гонки между preflight и первым v2 write advisory lock/maintenance
  flag; background consumers гарантированно остановлены.

Статус реализации на 2026-07-28:

- expand-миграция `d4e5f6a7b8ca` добавляет `approval_maintenance_state` и
  `approval_cutover_deals`; существующие колонки `approval` не получают NOT NULL и новых
  ограничений;
- `ApprovalMaintenanceService` реализует persisted flag и advisory lock `21642164`
  (exclusive у cutover, `pg_try_advisory_xact_lock_shared` в write-транзакциях);
  проверки встроены в request approval, rebuild, мутации deal/offer (`ApprovalSubjectGuard`,
  отказ до записи), API стадий, обработку stage token из письма, outbox worker и IMAP consumer;
- `ApprovalCutoverService` выполняет preflight, dry-run reconciliation и идемпотентную
  миграцию: draft-версии сделки заменяются одним deal-level маршрутом, пустой маршрут даёт
  версию `not_required`, legacy answered/canceled помечаются `workflow_version=1`,
  `is_current=false` с сохранением `offer_id`, невосстановимый `product_rule` даёт
  `manual_review_required`;
- операционная команда `python -m app.cli.approval_cutover` (внутри `app`, потому что
  `scripts/` не попадает в образ); runbook — `docs/approval_cutover_runbook.md`;
- deal-level сводка не превращает legacy approved/rejected историю в `not_required`:
  при отсутствии current staged версии берётся последняя завершённая legacy версия с
  признаком `is_legacy`;
- после включения v2 approval-level `email_token` больше не принимается; физическое
  удаление legacy-колонок остаётся за Ticket 15.

## Ticket 13 — Адаптировать создание заказов в ОП к deal-level approval

Цель: после единого решения по сделке создать отдельный заказ для каждого
согласованного оффера без конфликта idempotency key.

Зависимости: Ticket 6, Ticket 11 и контракт Order Processing.

Контекст:

- Сейчас `OfferImplementationService` ищет approval по `offer_id`, использует
  `approval.offer` и отправляет `approval.id` как `external_approval_id`.
- После перехода `1 deal = 1 approval` один `approval.id` относится к нескольким
  offers, но ОП по-прежнему создаёт отдельный заказ на каждый offer. Повторять один
  `external_approval_id` для разных заказов нельзя.

Что сделать:

- Запускать создание заказов только после полного `answered/approve` текущей version
  сделки и только для offers из одобренного `subject_snapshot`.
- Согласовать с ОП составной idempotency contract `(approval_id, offer_id)`.
  Если API ОП принимает только один UUID, использовать документированный
  детерминированный UUIDv5 от `approval.id + offer.id`; не генерировать случайный
  ключ при retry.
- Для трассировки передавать deal approval id/version и offer id отдельно, если
  контракт ОП позволяет.
- Формировать payload заказа из одобренного snapshot либо проверять совпадение
  `subject_hash` с live data перед отправкой. Нельзя создать заказ из данных,
  отличающихся от согласованных.
- Создавать/восстанавливать заказы независимо по каждому offer. Partial failure
  одного заказа не должен повторно создавать уже успешные.
- Хранить per-offer implementation status, attempts, last_error и использованный
  external id. Deal считается переданным в имплементацию только после успеха всех
  требуемых заказов.
- Передавать `special_conditions` каждого offer без преобразований.
- Добавить tests на два offers одного approval, retry после timeout, partial failure,
  повторный запуск и несовпадение snapshot/live data.

DoD:

- Один approved deal-level approval создаёт ровно один заказ на каждый offer.
- У каждого заказа стабильный уникальный idempotency key, повторный запуск не
  создаёт дублей.
- В ОП уходят именно согласованные данные и особые условия соответствующего offer.
- Partial failure восстанавливается повторным запуском только для неуспешных offers.

Статус реализации на 2026-07-28:

- миграция `e5f6a7b8c9db` добавляет `offer_implementations` с границей
  `UNIQUE(approval_id, offer_id)` и уникальным `external_approval_id`; `offer_order`
  не меняется и остаётся контрактом ответа оффера;
- `build_external_approval_id` сворачивает составной ключ `(approval_id, offer_id)` в
  детерминированный UUIDv5 от фиксированного namespace; retry после таймаута и повторный
  запуск используют то же значение;
- `OfferImplementationService.create_for_deal` требует `answered/approve` текущей staged
  версии, обходит offers одобренного `subject_snapshot`, строит payload из snapshot и
  сверяет живой оффер с одобренным предметом (`ApprovalSubjectMismatchError`);
  операционные поля (`contract_id`, стафферы) берутся из актуальной сделки;
- per-offer хранятся status/attempts/last_error/external id/данные БЗ; успешные строки
  переживают повтор, ошибка одного оффера не отменяет остальные;
- `DealService.implement_deal` выбирает deal-level путь по `approval_summary`, оставляет
  legacy offer-level ветку до Ticket 15 и при partial failure бросает
  `DealImplementationIncompleteError` (HTTP 409) со списком неуспешных offers;
- `special_conditions` уходит в ОП без преобразований; отдельная трассировка
  approval/offer в payload закрыта настройкой `ORDER_PROCESSING_SEND_EXTERNAL_CONTEXT`
  (по умолчанию выключена до согласования контракта с ОП).

## Ticket 14 — Интеграционные и конкурентные тесты workflow

Цель: проверить совместную работу builder, assignee selector, state machine, email,
API, Order Processing и БД после unit-тестов отдельных тикетов.

Что сделать:

- Добавить DB integration test полного маршрута
  `presale -> lead -> product_owner -> lawyer -> sales_director -> finance_director`
  при одновременных discount/product/legal triggers.
- Проверить отдельные маршруты: no approval, цепочка `product_owner → lawyer` без
  скидки и скидочная цепочка без особых условий.
- Проверить deal с несколькими offers: создаётся одна current approval version,
  максимальная скидка выбирает общий маршрут, а особые условия любого offer
  добавляют одну цепочку `product_owner → lawyer`.
- Проверить, что одна активная stage использует один snapshot/token и одно логическое
  письмо со всеми offers; group holders получают копии одного сообщения.
- Проверить single-holder и group assignee, включая первый ответ из группы.
- Проверить email approve/reject, expired/stale token, payload mismatch и repeated reply.
- Проверить гонки email decision против API decision и decision против
  skip/reassign: ровно одна команда меняет состояние, противоположная получает
  предсказуемый conflict.
- Проверить skip/reassign с audit events и permissions.
- Конкурентно отправить два решения одной pending stage: только один запрос меняет
  состояние, создаёт event и активирует следующую стадию.
- Конкурентно запустить один approval дважды: остаётся ровно одна pending stage и
  одно стартовое событие.
- Добавить тесты rollback транзакции при ошибке assignee и при ошибке записи event.
- Проверить конкурентные rebuild/start одного deal, partial unique current approval,
  неизменность snapshot после start и supersede/version history.
- Проверить outbox retry/idempotency и создание отдельных OP-заказов для нескольких
  offers без конфликтов `external_approval_id`.
- Проверить частичную group-доставку, изменение group membership во время pending,
  blocked/retry activation и пользователя с permissions нескольких последовательных
  stages.
- Запускать тесты на PostgreSQL, потому что `FOR UPDATE`, partial index и concurrency
  нельзя достоверно проверить только unit-моками.

DoD:

- Покрыты все перечисленные маршруты, переходы и race conditions.
- После каждого сценария проверяются stage, approval, token и events в БД.
- Нет flaky ожиданий через `sleep`; конкурентные тесты синхронизируются barrier-ом.
- `poetry run pytest` и `poetry run make check` проходят.

## Ticket 15 — Удалить legacy approval contract после стабилизации

Цель: завершить expand/contract migration только после успешной эксплуатации v2.

Зависимости: Ticket 12–14, подтверждённый период наблюдения и отсутствие rollback
на offer-level release.

Что сделать:

- Проверить, что нет `workflow_version=1` records, необходимых online API. Перед
  удалением legacy columns перенести требуемую историю в snapshot/history schemas.
- Удалить создание/чтение `approval.approvers`, approval-level `email_token`,
  `email_token_expires_at`, scalar legacy `reason_type` и обязательную связь
  `approval.offer_id`.
- Удалить `OfferModel.approval`, `ApprovalCrud.get_by_offer_id`,
  `_build_approval_create_data()` в `OfferService` и legacy ветки
  `DealService.request_approval()/receive_mail()`.
- Сделать v2 fields/constraints NOT NULL там, где это безопасно после backfill.
- Удалить feature-flag fallback, но сохранить аварийное выключение новых starts и
  background delivery.
- Добавить migration tests и сверку публичной OpenAPI schema на отсутствие token и
  offer-level approval contract.

DoD:

- В runtime нет двух конкурирующих approval workflows.
- Legacy bearer token/approvers не читаются и не возвращаются API.
- История решений сохранена, migrations применяются/откатываются в допустимой
  contract-границе.
