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

Цель: определить, кому отправлять активную стадию.

Что сделать:

- Для единоличных permissions выбирать пользователя из `lkm_user_permissions`.
- Для групповых permissions определить правило: конкретный assignee, все пользователи с
  permission или ручной выбор инициатором.
- Если assignee не найден — stage не отправлять, возвращать понятную ошибку или статус
  `blocked` после согласования с аналитиком.

DoD:

- Stage нельзя отправить в никуда.
- Единоличные permissions приводят к одному assignee.
- Ошибка отсутствующего согласующего понятна пользователю.

## Ticket 6 — Реализовать state machine стадий

Цель: единый сервис переходов `start`, `approve`, `reject`, `skip`, `reassign`, `cancel`.

Что сделать:

- Создать `ApprovalWorkflowService`.
- Все переходы выполнять транзакционно.
- После `approve` активировать следующую waiting stage.
- После `reject` закрывать approval как rejected и отменять следующие стадии.
- После завершения всех стадий закрывать approval как approved.

DoD:

- Невалидные переходы запрещены.
- Повторный approve/reject идемпотентен или даёт предсказуемую доменную ошибку.
- Тесты покрывают happy path, reject, no next stage.

## Ticket 7 — Перевести email approval на stage token

Цель: email-ответ должен закрывать конкретную стадию, а не весь approval.

Что сделать:

- Генерировать `email_token` на `approval_stages`.
- В payload письма передавать `stage_id`/stage token.
- `receive_mail()` должен искать pending stage по token.
- Старые token'ы после reassign/cancel должны становиться недействительными.

DoD:

- Старое письмо не может закрыть новую стадию.
- Повторное письмо по закрытой stage не двигает процесс повторно.
- Тесты покрывают stale token и repeated email.

## Ticket 8 — API для задач согласующего

Цель: дать фронту список КП, ожидающих решения текущего пользователя.

Что сделать:

- Добавить endpoint pending approval stages текущего пользователя.
- Фильтровать по `assigned_user_id` и/или effective permissions по принятому правилу.
- Отдавать stage metadata: stage code, due_at, offer/deal summary, status.

DoD:

- Пользователь видит только доступные ему стадии.
- Stage исчезает из pending после decision/skip/reassign.
- Есть API tests на permissions.

## Ticket 9 — API для skip/reassign/escalate

Цель: поддержать отсутствие согласующего без ручной правки БД.

Что сделать:

- Добавить permission `skip_kp_approval_stage` или согласовать использование существующей admin permission.
- Добавить endpoint `skip` с обязательным reason.
- Добавить endpoint `reassign` с проверкой permission нового assignee.
- Опционально добавить escalation job/rule по `due_at`.

DoD:

- Skip сохраняет `skipped_by`, `skipped_at`, `skip_reason`.
- Reassign инвалидирует старый token и создаёт новый.
- Unauthorized user получает 403.

## Ticket 10 — Пересобрать actions и статусы UI под stage workflow

Цель: frontend actions должны отражать текущий процесс стадий, а не один общий
`approval.status`.

Что сделать:

- Обновить `deal_actions` / `offer_actions` context: учитывать active stage.
- В response добавить summary текущего approval process: active stage, status,
  assignee, due_at, skipped stages.
- Сохранить блокировки редактирования/удаления при pending stage.

DoD:

- UI видит текущую стадию и доступные действия.
- Permissions текущего пользователя учитываются при сборке actions.
- Тесты покрывают actions при pending/approved/skipped/rejected stages.

## Ticket 11 — Аудит workflow-переходов

Цель: иметь объяснимую историю согласования для спорных кейсов и skip/reassign.

Что сделать:

- Минимум: хранить audit fields на `approval_stages`.
- Если нужен полный аудит: добавить `approval_stage_events`.
- Логировать actor, action, old_status, new_status, reason, created_at.

DoD:

- По каждому skip/reassign видно кто, когда и почему сделал действие.
- Историю можно показать в карточке КП или админском представлении.

## Ticket 12 — Миграция текущих approvals и backward compatibility

Цель: безопасно перейти от текущей модели к stage workflow.

Что сделать:

- Определить поведение существующих `approval` в `draft/pending/answered`.
- Для draft создать stages заново builder'ом.
- Для pending либо создать одну legacy stage, либо запретить миграцию при pending approvals.
- Для answered создать закрытую legacy stage или оставить readonly mapping.

DoD:

- Миграция не теряет старые решения.
- Pending approvals обрабатываются по явно выбранной стратегии.
- Есть rollback/операционный план.

## Ticket 13 — Тесты workflow

Цель: покрыть рискованные переходы до подключения к реальным письмам и UI.

Что сделать:

- Unit tests state machine.
- Integration tests route builder + DB stages.
- Email tests для approve/reject by token.
- Race/idempotency tests для повторных ответов и параллельных actions.

DoD:

- Покрыты approve chain, reject chain, skip, reassign, stale token, repeated token.
- `poetry run pytest` проходит.
