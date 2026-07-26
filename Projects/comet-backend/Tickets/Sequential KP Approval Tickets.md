---
project: comet-backend
created: 2026-07-23
updated: 2026-07-26
source: docs/lkm_role_model.md
tags: [project, tickets, approval, lkm, rbac]
---

# Sequential KP Approval Tickets

#project #tickets #approval #lkm #rbac

Связано: [[Sequential KP Approval]], [[LKM Role Model]], [[BT02 Role Model Gaps]].

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

Цель: строить набор стадий для КП на основании причины согласования и данных оффера.

Что сделать:

- Вынести построение маршрута из `OfferService._build_approval_create_data`.
- На вход builder принимает offer/deal/reason context.
- На выходе отдаёт ordered list стадий с `stage_code`, `required_permission`, optional/mandatory.
- Для первой итерации поддержать маршруты из БТ02: presale, lead, sales director,
  finance director, product owner, lawyer.

DoD:

- Маршрут не создаёт ненужные стадии.
- Для каждого stage указана required permission.
- Есть тесты на несколько типов маршрутов.

Статус на 2026-07-26: в текущем дереве route builder не подключён. `OfferService`
по-прежнему создаёт draft approval с mock approver через `_build_approval_create_data()`;
стадии при создании approval не строятся.

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

- Особые условия есть, если `offer.special_conditions` не `null` и после `strip()`
  содержит хотя бы один символ.
- При наличии особых условий builder добавляет одну обязательную стадию
  `ApprovalStageCode.LAWYER` с `required_permission=Permission.APPROVE_KP_LAWYER`.
- Юридическая стадия не заменяет стадии, выбранные по скидке: итоговый маршрут —
  объединение скидочной цепочки из Ticket 2 и стадии `lawyer`.
- `lawyer` располагается после всех стадий скидочной цепочки. Если скидочного
  согласования нет, но особые условия есть, маршрут состоит только из `lawyer`.
- При `null`, пустой или состоящей только из пробелов строке стадия `lawyer` не создаётся.

Что сделать:

- Расширить builder маршрута из Ticket 2: он сам читает `offer.special_conditions`,
  не получает отдельный `requires_lawyer` или другой флаг от вызывающего кода.
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
- Скидочные стадии сохраняются и идут перед юридической стадией.
- КП без скидки, но с особыми условиями, не проходит без согласования.
- Пустые особые условия не добавляют стадию юриста.
- Тесты фиксируют состав, порядок и `required_permission` маршрута.

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

## Ticket 6 — Реализовать state machine стадий

Цель: реализовать единый транзакционный сервис переходов состояния approval и его
стадий.

Граница процесса:

- Один `ApprovalModel` относится к одному offer.
- Deal может содержать несколько независимых approval-процессов по своим offers.
- `POST /api/v1/deals/{deal_id}/approval/request` запускает все draft approvals
  офферов сделки. Между offers процессы идут параллельно, внутри одного approval
  стадии идут строго по `position`.
- Offer без стадий не создаёт/не запускает approval и не блокирует сделку.

Что сделать:

- Создать `ApprovalWorkflowService` с операциями `start`, `approve`, `reject`,
  `skip`, `reassign`, `cancel`.
- Добавить в `approval_stages` persisted-поле `is_optional BOOLEAN NOT NULL
  DEFAULT false`; обновить модель, migration и Pydantic-схемы. Route builder должен
  сохранять в БД optional/mandatory признак.
- Все переходы выполнять в одной DB-транзакции с `SELECT ... FOR UPDATE` для
  approval и его активной стадии.
- Добавить partial unique index, запрещающий более одной стадии `pending` внутри
  одного approval.
- `start`: разрешён только для `approval=draft`; активирует первую `waiting`
  стадию. При ошибке выбора assignee ничего не меняет.
- `approve`: разрешён только для `pending`; сохраняет `approved`, `decision=approve`,
  `decided_at`, `decision_comment`, инвалидирует token и активирует следующую
  `waiting` stage.
- Если после approve/skip стадий больше нет, установить
  `approval.status=answered`, `approval.decision=approve`, `decided_at`.
- `reject`: переводит активную стадию в `rejected`, сохраняет решение и comment,
  остальные `waiting` стадии — в `canceled`, approval — в
  `answered/reject`.
- `skip`: переводит стадию в `skipped` с actor/time/reason и активирует следующую.
  Авторизация skip реализуется в Ticket 10.
- `reassign`: оставляет стадию `pending`, меняет assignee, инвалидирует старый token
  и выпускает новый. Авторизация реализуется в Ticket 10.
- `cancel`: переводит approval и все `waiting/pending` стадии в `canceled`,
  инвалидирует активные token'ы.
- Повторное действие над уже закрытой стадией не меняет состояние и возвращает
  предсказуемый результат `already_processed`.

DoD:

- В одном approval не может быть больше одной `pending` stage.
- Невалидные переходы не записываются в БД.
- Полная цепочка approve завершается агрегатным `answered/approve`.
- Reject закрывает процесс и отменяет необработанные стадии.
- Cancel и repeated decision инвалидируют возможность сдвинуть процесс старым token.
- Тесты покрывают start, approve chain, reject, skip, reassign, cancel,
  no-next-stage и concurrent/repeated decision.

## Ticket 7 — Перевести email approval на stage token

Цель: email-ответ должен закрывать конкретную стадию, а не весь approval.

Контракт письма:

```json
{
  "deal_id": "uuid",
  "deal_number": 123,
  "offer_id": "uuid",
  "stage_id": "uuid",
  "email_token": "uuid",
  "decision": "approve | reject"
}
```

Источник истины — уникальный `approval_stages.email_token`. Остальные идентификаторы
проверяются на соответствие найденной стадии и используются для понятной диагностики.

Что сделать:

- При активации стадии генерировать новый `email_token`, устанавливать срок действия,
  `requested_at` и `due_at`.
- Для единоличной стадии отправлять письмо `assigned_email`; для групповой —
  каждому holder `required_permission` с одинаковым stage token.
- Расширить `ReceivedApprovalDecisionSchema` полями `offer_id` и `stage_id`.
- Добавить CRUD-запрос стадии по token с eager load approval/offer/deal.
- `DealService.receive_mail()` после валидации payload должен вызывать только
  `ApprovalWorkflowService.approve/reject`, не менять approval напрямую.
- Принимать решение только для стадии `pending`, с неистёкшим token и совпадающими
  `deal_id`, `deal_number`, `offer_id`, `stage_id`.
- Для единоличной стадии `ReceivedEmailSchema.from_email` должен совпадать с
  `assigned_email`. Для групповой стадии отправитель должен быть сохранённым LKM user
  с effective `required_permission` на момент обработки ответа.
- После approve/reject/skip/cancel обнулять token и expires_at. После reassign
  выпускать другой token.
- Ответ по закрытой стадии или старому token считать `already_processed`; он не
  должен повторно отправлять следующую стадию.
- Удалить использование approval-level `email_token` из нового workflow. Legacy
  обработка остаётся только для записей без stages до выполнения Ticket 12.

DoD:

- Email однозначно закрывает только указанную stage соответствующего offer.
- Старый, истёкший или не совпадающий с payload token не меняет БД.
- Первый ответ на групповую стадию закрывает её, остальные ответы идемпотентны.
- Ответ с email пользователя, которому стадия недоступна, не меняет БД.
- После решения автоматически отправляется только следующая стадия того же approval.
- Тесты покрывают valid decision, stale/expired token, payload mismatch,
  repeated group reply и переход к следующей стадии.

## Ticket 8 — API для задач согласующего

Цель: дать фронту список КП, ожидающих решения текущего пользователя.

Что сделать:

- Добавить `GET /api/v1/approval-stages/pending`.
- Текущего LKM user определять по аутентифицированному AD login/email.
- Единоличную stage показывать только при `assigned_user_id=current_user.id`.
- Групповую stage (`assigned_user_id IS NULL`) показывать, если пользователь имеет
  effective `required_permission`.
- Возвращать только `status=pending`; сортировать по `due_at NULLS LAST`,
  затем `requested_at`, поддержать `limit/offset`.
- Response item должен содержать: `stage_id`, `stage_code`, `required_permission`,
  `status`, `requested_at`, `due_at`, `approval_id`, `offer_id`, `product_id`,
  `deal_id`, `deal_number`, `deal_title`, `client_id`.
- Не возвращать `email_token` в API.
- Реализовать SQL-фильтрацию в CRUD, а не загружать все стадии в память.
- Добавить API-тесты для single-holder, group permission, постороннего пользователя,
  закрытой стадии и пагинации.

DoD:

- Пользователь видит только доступные ему стадии.
- Stage исчезает из pending после decision/skip/reassign.
- Закрытые стадии и token не попадают в response.
- Пагинация и порядок стабильны.
- Есть API tests на permissions и isolation между пользователями.

## Ticket 9 — Добавить аудит переходов workflow

Цель: хранить неизменяемую историю каждого перехода стадии до появления управляющих
API.

Что сделать:

- Добавить `approval_stage_events`: `id`, `stage_id`, `approval_id`, `actor_user_id`
  nullable для системного действия, `action`, `old_status`, `new_status`,
  `reason`, `metadata JSONB`, `created_at`.
- Ввести enum action: `started`, `approved`, `rejected`, `skipped`, `reassigned`,
  `canceled`.
- Добавить индексы `(stage_id, created_at)` и `(approval_id, created_at)`.
- Каждая операция `ApprovalWorkflowService` должна создавать event в той же
  транзакции, что и изменение stage.
- Для reassign сохранять old/new user id и email в `metadata`; для email decision —
  `ReceivedEmailSchema.id`.
- Event-записи не обновлять и не удалять отдельно от cascade удаления всего approval.
- Добавить read schema и включить ordered history в детальный response approval.
- Добавить model, CRUD и integration tests для каждого action.

DoD:

- Для каждого фактического перехода есть ровно один event.
- Повторный идемпотентный запрос не создаёт второй event.
- По reassign видно прежнего и нового assignee, по skip — actor и reason.
- Изменение stage и создание event атомарны.

## Ticket 10 — API для skip и reassign

Цель: поддержать отсутствие согласующего без ручной правки БД.

Принятые permissions:

- `skip_kp_approval_stage`;
- `reassign_kp_approval_stage`.

Permissions добавить в `Permission`; они синхронизируются с `lkm_permissions`.
Проверки выполнять только по permissions, без проверки имени роли. Escalation и
автоматический skip в этот тикет не входят.

Что сделать:

- Добавить `POST /api/v1/approval-stages/{stage_id}/skip` с body
  `{"reason": "непустая строка"}`.
- Добавить `POST /api/v1/approval-stages/{stage_id}/reassign` с body
  `{"user_id": "uuid", "reason": "непустая строка"}`.
- Skip/reassign разрешены только для `pending` stage и пользователя с
  соответствующей permission.
- Новый assignee должен существовать в `lkm_users` и иметь effective `required_permission`.
  Для единоличной permission он также должен быть её единственным персональным holder.
- Skip вызывает state machine, сохраняет `skipped_by_user_id`, `skipped_at`,
  `skip_reason`, создаёт audit event и активирует следующую stage.
- Reassign меняет assignee, создаёт audit event и новое письмо/token. Старый token
  становится недействительным до отправки нового письма.
- Для 403, stage not found, invalid status, invalid target и empty reason добавить
  отдельные пользовательские ошибки.
- Добавить API и service tests.

DoD:

- Skip и reassign нельзя вызвать без соответствующей permission.
- Skip без reason и reassign неподходящему пользователю отклоняются без изменения БД.
- Reassign инвалидирует старый token, отправляет новый и сохраняет old/new assignee.
- Skip активирует следующую стадию и остаётся отличимым от approve.
- Все действия отражены в `approval_stage_events`.

## Ticket 11 — Пересобрать actions и статусы API под stage workflow

Цель: backend response и action availability должны отражать offer-level stage
workflow, а не первое legacy approval сделки.

Что сделать:

- Удалить использование `DealModel.approval`, возвращающего первый approval.
- Для offer вычислять `approval_summary`: `approval_id`, aggregate `status`,
  `decision`, `active_stage`, `completed_stages`, `skipped_stages`,
  `total_stages`.
- `active_stage` содержит `stage_id`, `stage_code`, `required_permission`,
  `assigned_user_id`, `assigned_email`, `requested_at`, `due_at`, но не token.
- Для deal возвращать список `offer_approvals` и агрегат:
  `not_required | draft | pending | approved | rejected | canceled`.
- Deal aggregate: `rejected`, если отклонён хотя бы один offer; `pending`, если
  хотя бы один процесс pending; `draft`, если есть draft и нет pending/rejected;
  `canceled`, если pending/draft/rejected отсутствуют и отменён хотя бы один процесс;
  `approved`, если все требующие согласования offers approved; `not_required`, если
  ни один offer не требует согласования.
- Обновить `deal_actions` / `offer_actions`: edit/delete запрещены только для
  соответствующего pending approval; request approval доступен, если в сделке есть
  хотя бы один draft approval и нет pending.
- Доступность `skip/reassign` рассчитывать по active stage и permissions текущего
  пользователя.
- Добавить response schemas и tests для нескольких offers с разными состояниями.

DoD:

- API не выбирает случайный/первый approval сделки.
- Frontend получает active stage и агрегат по всем offers без дополнительных запросов.
- Actions соответствуют stage state и effective permissions.
- Тесты покрывают mixed offer states, pending, approved, skipped, rejected и canceled.

## Ticket 12 — Миграция текущих approvals и backward compatibility

Цель: безопасно перейти от текущей модели к stage workflow.

Принятая стратегия:

- Перед cutover временно запретить новые отправки на согласование.
- На момент миграции в БД не должно быть `approval.status=pending`. Preflight
  проверка прерывает операцию и выводит ID таких approvals; их нужно дождаться или
  отозвать штатным способом.
- `draft` approvals пересобираются builder-ом по актуальным данным offer/deal.
- `answered` и `canceled` approvals не получают искусственных stages и остаются
  readonly legacy history.
- Новый workflow работает только с approvals, у которых есть stages.

Что сделать:

- Добавить preflight-команду, проверяющую pending approvals и печатающую их
  `approval_id`, `offer_id`, `deal_id`.
- Добавить идемпотентный data migration/service command для пересборки stages всех
  draft approvals. Повторный запуск не должен создавать дубликаты.
- Для draft с пустым маршрутом удалить approval и увеличить счётчик
  `approval_not_required` в отчёте.
- В read schemas сохранить отображение answered/canceled legacy approvals без stages.
- После успешного cutover перестать создавать и читать `approval.approvers` и
  approval-level email token в новом workflow. Физическое удаление legacy columns
  вынести в отдельную cleanup migration после стабилизации.
- Описать runbook: backup, запрет новых request, preflight, migration, проверки
  количества, deploy, включение request, rollback.
- Rollback возвращает старый код, не удаляя legacy columns; созданные stages можно
  оставить неиспользуемыми до повторного cutover.

DoD:

- Миграция не теряет старые решения.
- Cutover не начинается при наличии pending legacy approvals.
- Все оставшиеся draft approvals имеют корректный маршрут без дублей; удалённые
  пустые маршруты учтены в `approval_not_required`.
- Answered/canceled legacy approvals продолжают читаться.
- Повторный запуск migration идемпотентен.
- Есть проверенный dry-run, rollback и операционный runbook.

## Ticket 13 — Интеграционные и конкурентные тесты workflow

Цель: проверить совместную работу builder, assignee selector, state machine, email,
API и БД после unit-тестов отдельных тикетов.

Что сделать:

- Добавить DB integration test полного маршрута
  `presale -> lead -> sales_director -> finance_director -> lawyer`.
- Проверить отдельные маршруты: no approval, только lawyer, только product owner,
  скидочная цепочка без lawyer.
- Проверить deal с несколькими offers: независимые approval-процессы стартуют
  параллельно и корректно агрегируются.
- Проверить single-holder и group assignee, включая первый ответ из группы.
- Проверить email approve/reject, expired/stale token, payload mismatch и repeated reply.
- Проверить skip/reassign с audit events и permissions.
- Конкурентно отправить два решения одной pending stage: только один запрос меняет
  состояние, создаёт event и активирует следующую стадию.
- Конкурентно запустить один approval дважды: остаётся ровно одна pending stage и
  одно стартовое событие.
- Добавить тесты rollback транзакции при ошибке assignee и при ошибке записи event.
- Запускать тесты на PostgreSQL, потому что `FOR UPDATE`, partial index и concurrency
  нельзя достоверно проверить только unit-моками.

DoD:

- Покрыты все перечисленные маршруты, переходы и race conditions.
- После каждого сценария проверяются stage, approval, token и events в БД.
- Нет flaky ожиданий через `sleep`; конкурентные тесты синхронизируются barrier-ом.
- `poetry run pytest` и `poetry run make check` проходят.
