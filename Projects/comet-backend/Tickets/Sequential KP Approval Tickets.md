---
project: comet-backend
created: 2026-07-23
updated: 2026-07-23
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

## Ticket 3 — Реализовать выбор assignee для стадии

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

## Ticket 4 — Реализовать state machine стадий

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

## Ticket 5 — Перевести email approval на stage token

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

## Ticket 6 — API для задач согласующего

Цель: дать фронту список КП, ожидающих решения текущего пользователя.

Что сделать:

- Добавить endpoint pending approval stages текущего пользователя.
- Фильтровать по `assigned_user_id` и/или effective permissions по принятому правилу.
- Отдавать stage metadata: stage code, due_at, offer/deal summary, status.

DoD:

- Пользователь видит только доступные ему стадии.
- Stage исчезает из pending после decision/skip/reassign.
- Есть API tests на permissions.

## Ticket 7 — API для skip/reassign/escalate

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

## Ticket 8 — Пересобрать actions и статусы UI под stage workflow

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

## Ticket 9 — Аудит workflow-переходов

Цель: иметь объяснимую историю согласования для спорных кейсов и skip/reassign.

Что сделать:

- Минимум: хранить audit fields на `approval_stages`.
- Если нужен полный аудит: добавить `approval_stage_events`.
- Логировать actor, action, old_status, new_status, reason, created_at.

DoD:

- По каждому skip/reassign видно кто, когда и почему сделал действие.
- Историю можно показать в карточке КП или админском представлении.

## Ticket 10 — Миграция текущих approvals и backward compatibility

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

## Ticket 11 — Тесты workflow

Цель: покрыть рискованные переходы до подключения к реальным письмам и UI.

Что сделать:

- Unit tests state machine.
- Integration tests route builder + DB stages.
- Email tests для approve/reject by token.
- Race/idempotency tests для повторных ответов и параллельных actions.

DoD:

- Покрыты approve chain, reject chain, skip, reassign, stale token, repeated token.
- `poetry run pytest` проходит.
