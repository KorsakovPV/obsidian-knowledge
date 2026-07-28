---
project: comet-backend
created: 2026-06-25
updated: 2026-07-28
tags: [project, domain, database, sqlalchemy]
---

# Domain Model

#project

Доменная модель [[Overview|comet-backend]]. ORM-модели — в `app/models/`, общий `Base`,
PostgreSQL. Ниже основные таблицы и связи. Бизнес-смысл сущностей см. в [[Overview]],
процессы — в [[Architecture]].

## Связи (кратко)

```
Deal 1───* Approval (версии; ровно одна is_current)
  │            ├──* ApprovalStage ──* ApprovalStageToken ──* ApprovalStageNotification ──* Delivery
  │            │                   └─* ApprovalStageDecisionRequest / ApprovalStageMutationRequest
  │            ├──* ApprovalStageEvent   (ordered audit по sequence)
  │            └──* OfferImplementation  (по одному на offer одобренного snapshot)
  ├──* Offer 1───0..1 OfferOrder  (заказ в ОП)
  │        └───0..1 Approval      (legacy offer-level связь, удаляется в Ticket 15)
  └──* DealAttachment

PreTariff 1───* PreTariffComment
          1───* PreTariffAttachment

ClientPrice            (цены клиента)
LkmUser / LkmRole / LkmPermission  (ролевая модель ЛКМ)
ApprovalMaintenanceState / ApprovalCutoverDeal  (служебные таблицы cutover)
```

## Deal — `deal`

Коммерческое предложение (сделка).

- `client_id`, `contract_id?`, `region_id` — UUID заказчика / договора / региона.
- `title`, `description?`.
- `seller` — организатор продаж, enum `Seller`: `datafort` | `vimpelcom`.
- `staffers_contractor`, `staffers_customer`, `staffers_partner_customer` —
  списки UUID сотрудников.
- `number` — порядковый номер сделки; `deal_date`, `created_by`, `updated_by?`.
- Связи: `offers` (cascade delete-orphan), `attachments`, `approvals`.

## Offer — `offer`

Оффер: один продукт = один бланк заказа. Принадлежит сделке.

- `product_id` (int) — продукт/услуга; `deal_id` → `deal`.
- `technical_parameters` (JSONB), `tariffs` (JSONB) — тех. параметры и тарифы бланка заказа.
  Тариф хранит присланные клиентом `pk`, `quantity`, `price` (цена продажи),
  `partner_client_price` (цена конечного потребителя, в бизнес-логике не участвует),
  `group_code` и рассчитанные сервером `base_price`, `base_price_source`,
  `discount_percent` — по ним route builder выбирает скидочный маршрут.
- `created_by`, `updated_by?`.
- Связи: `deal`, `order` (0..1, `OfferOrder`); `approval` (0..1) — legacy offer-level
  связь, помеченная TODO Ticket 15.
- `special_conditions?` — nullable особые условия КП; передаются без преобразований
  в создаваемый заказ ОП.

## Approval — `approval`

Процесс согласования сделки: владелец — `deal_id`, все офферы рассматриваются вместе.

- `deal_id` → `deal`; `offer_id?` → `offer` — legacy-связь, у staged-версий `NULL`.
- Версии: `version`, `is_current`, `superseded_at`, `workflow_version`
  (`1` — legacy, `2` — staged). Ограничения: `UNIQUE(deal_id, version)` и partial
  unique index по `deal_id WHERE is_current` — у сделки не больше одной текущей версии.
- Предмет согласования: `subject_snapshot` (JSONB со сделкой и всеми офферами),
  `subject_hash`, `route_context` (триггеры маршрута, максимальная скидка и её источник,
  версии builder-а и `STAGE_ORDER`).
- `status` (`ApprovalStatus`):
  `draft` | `pending` | `blocked` | `answered` | `canceled` | `not_required`.
- `decision?` (`ApprovalDecision`): `approve` | `reject`; `decision_comment?`.
- `blocked_permission?`, `blocked_at?` — стадия, для которой не нашёлся согласующий.
- `requested_at?`, `requested_by_user_id?` → `lkm_users` — когда и кем сделка отправлена
  на согласование (событие `started` создаётся системой и actor не содержит), `decided_at?`.
- `reason_type` (`ApprovalReasonType`): `discount` | `product_rule` — legacy scalar-причина;
  полный набор причин лежит в `route_context`.
- `source` (`ApprovalSource`): `email` | `auto_related_deals` | `auto_rule`.
- Legacy-поля до Ticket 15: `approvers` (JSON-список), `email_token?`,
  `email_token_expires_at?`.
- Связи: `stages` (сортировка по `position`), `events` (сортировка по `sequence`).
- Маршрут строится по максимальной скидке среди всех offers сделки; особые условия
  любого offer добавляют обязательную цепочку `product_owner → lawyer`.
- После `start` snapshot неизменяем; повторное согласование создаёт следующую version,
  предыдущая помечается `is_current=false`.

## ApprovalStage — `approval_stages`

Явная стадия последовательного согласования.

- `approval_id` → `approval`, `stage_code` (`ApprovalStageCode`):
  `presale` | `lead` | `sales_director` | `finance_director` | `product_owner` | `lawyer`.
- `position` — порядковый номер стадии внутри approval; `required_permission` — permission,
  необходимая для закрытия стадии.
- `status` (`ApprovalStageStatus`): `waiting` | `pending` | `approved` | `rejected` |
  `skipped` | `canceled`.
- `is_optional` — можно ли пропустить стадию; mandatory-стадии нельзя skip даже с правом.
- `scope_product_ids` (JSONB) — продукты, вызвавшие включение стадии. Заполняется только
  у `product_owner` и фиксируется вместе с маршрутом: активация происходит позже, и
  повторный вывод причины дал бы другой состав получателей. Пусто — стадия к продукту
  не привязана.
- `assigned_user_id?` → `lkm_users`, `assigned_email?` — заполняются только для
  единоличной стадии; у групповой остаются `NULL`, получатели считаются по
  `required_permission`.
- `requested_at?`, `due_at?` (SLA стадии), `decided_at?`, `decision?`, `decision_comment?`.
- `skipped_by_user_id?` → `lkm_users`, `skipped_at?`, `skip_reason?`.
- Legacy до Ticket 15: `email_token?`, `email_token_expires_at?`.
- Ограничения: уникальны `(approval_id, position)`, `(approval_id, stage_code)` и
  `email_token`; partial unique index по `approval_id WHERE status = 'pending'` не
  допускает две активные стадии в одном процессе.

## ApprovalStageToken — `approval_stage_tokens`

История bearer token стадии: `stage_id`, уникальный `token_hash`, `issued_at`,
`expires_at`, `used_at`, `invalidated_at`. Raw token в БД не хранится. Старые записи
сохраняются для идемпотентной обработки повторных ответов.

## ApprovalStageNotification — `approval_stage_notifications`

Одна логическая notification generation: `stage_id`, `token_id`, immutable payload
snapshot и временно encrypted raw token.

## ApprovalStageNotificationDelivery — `approval_stage_notification_deliveries`

Получатели transactional outbox: `notification_id`, recipient user/email, delivery
status (`pending/sent/failed/canceled`), attempts, `next_attempt_at`, `sent_at`,
`last_error`. Уникальна пара `(notification_id, canonical_recipient_email)`. Для group
stage несколько delivery rows относятся к одной notification/stage/token.

## ApprovalStageEvent — `approval_stage_events`

Неизменяемая ordered история переходов: `stage_id`, `approval_id`, `actor_user_id?`,
`action` (`started` | `approved` | `rejected` | `skipped` | `reassigned` | `canceled` |
`superseded` | `activation_failed` | `activation_retried`), `old_status`, `new_status`,
`reason?`, `metadata` (JSONB), `sequence`, `source` (`system` | `api` | `email`),
`source_id?`. Уникальны `(approval_id, sequence)` и — при заполненном `source_id` —
`(approval_id, stage_id, actor_user_id, source, source_id)`.

## Идемпотентность команд — `approval_stage_decision_requests`, `approval_stage_mutation_requests`

Сохранённый результат API-команды в области `(stage_id, actor_user_id, idempotency_key)`:
`decision` / `action` + `payload_hash`, `result_code` и полный `response_payload`.
Повтор возвращает сохранённый ответ и не выполняет переход заново; та же строка ключа с
другим решением даёт `decision_conflict`.

## OfferImplementation — `offer_implementations`

Результат передачи одного оффера в ОП внутри одной одобренной версии согласования.

- `approval_id` → `approval`, `deal_id`, `offer_id`, `approval_version`, `subject_hash?`.
- `external_approval_id` — детерминированный UUIDv5 от пары `(approval_id, offer_id)`:
  ключ идемпотентности заказа в ОП.
- `status` (`pending` | `succeeded` | `failed`), `attempts`, `last_error?`,
  `order_id?`, `order_number?`, `order_status?`, `succeeded_at?`.
- Ограничения: уникальны `(approval_id, offer_id)` и `external_approval_id`.

## Служебные таблицы cutover — `approval_maintenance_state`, `approval_cutover_deals`

- `approval_maintenance_state`: persisted maintenance flag (`key`, `is_active`, `reason?`,
  `activated_at?`, `activated_by?`), проверяется внутри write-транзакций вместе с
  advisory lock.
- `approval_cutover_deals`: журнал миграции по сделке (`deal_id`, `state`, `reason?`,
  `legacy_approval_ids`, `target_approval_id?`, `subject_hash?`, `report`, `migrated_at?`),
  делает повторный запуск идемпотентным. Порядок операций — [[Approval Cutover Runbook]].

## OfferOrder — `offer_order`

Связь оффера с заказом во внешней системе ОП (Order Processing).

- `offer_id` → `offer`, `order_id` (UUID в ОП), `order_number`, `order_status?`.

## DealAttachment — `deal_attachment`

Вложение сделки (файл в S3).

- `deal_id` → `deal`, `file_name`, `storage_key` (ключ в S3), `content_type?`,
  `size?`, `uploaded_by?`.

## PreTariff — `pre_tariff` (+ комментарии и вложения)

Пре-тариф (заготовка тарифа) с обсуждением.

- `pre_tariff`: `description`, `created_by`, `updated_by?`; связи `comments`, `attachments`.
- `pre_tariff_comment`: `pre_tariff_id`, `comment`, `created_by`, `updated_by?`.
- `pre_tariff_attachment`: `pre_tariff_id`, `file_name`, `storage_key`, `content_type?`,
  `size?`, `uploaded_by?`.

## ClientPrice — `client_price`

Индивидуальная цена клиента на тариф.

- `client_id`, `tariff_id`, `price` (Decimal), `valid_period` (диапазон дат).

## Ролевая модель ЛКМ (LKM)

Личный кабинет менеджера: пользователи, роли, права.

- `lkm_users`: `ad_login`, `email?` (канонический delivery email писем согласования),
  `full_name`, `role` (`UserRole`:
  `manager` | `presale` | `sales_lead` | `sales_director` | `finance_director` | `product_owner` | `lawyer` | `admin`), `first_login_at?`, `created_by_admin?`.
- `lkm_permissions` (`codename`, `name`, 22 записи), `lkm_roles` (`codename`, `name`).
- Связки: `lkm_role_permissions` (набор прав роли), `lkm_user_permissions` (персональные права).
- `lkm_user_permissions.scope_product_id` — зона ответственности назначения: `NULL` даёт
  право на все продукты, заполненный ограничивает его одним продуктом. Уникальность
  задана двумя частичными индексами: обычный ключ с nullable-колонкой не запретил бы два
  глобальных назначения, потому что NULL в Postgres не сравнивается сам с собой.
- Пермиссии/роли и наборы засеваются миграциями `..._add_lkm_permissions.py` и `..._bt02_roles_permissions.py`.
- Эффективные права = пермиссии роли ∪ персональные права. Полная спецификация —
  в исследовании [[LKM Role Model]]; проверка прав — в [[Architecture]].

---

Миграции схемы — Alembic (`migrations/versions/`), см. [[Architecture]].
