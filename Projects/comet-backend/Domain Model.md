---
project: comet-backend
created: 2026-06-25
updated: 2026-07-27
tags: [project, domain, database, sqlalchemy]
---

# Domain Model

#project

Доменная модель [[Overview|comet-backend]]. ORM-модели — в `app/models/`, общий `Base`,
PostgreSQL. Ниже основные таблицы и связи. Бизнес-смысл сущностей см. в [[Overview]],
процессы — в [[Architecture]].

## Связи (кратко)

```
Deal 1───* Offer 1───0..1 Approval 1───* ApprovalStage
  │           └──────0..1 OfferOrder  (заказ в ОП)
  └──* DealAttachment

PreTariff 1───* PreTariffComment
          1───* PreTariffAttachment

ClientPrice            (цены клиента)
LkmUser / LkmRole / LkmPermission  (ролевая модель ЛКМ)
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
- `created_by`, `updated_by?`.
- Связи: `deal`, `approval` (0..1), `order` (0..1, `OfferOrder`).
- `special_conditions?` — nullable особые условия КП; передаются без преобразований
  в создаваемый заказ ОП.

## Approval — `approval`

Текущее состояние кода хранит согласование оффера, но целевая модель workflow —
единое согласование сделки со всеми её офферами.

- Сейчас: `offer_id` → `offer`, `deal_id` → `deal`.
- Цель Ticket 6: владелец процесса — `deal_id`; у сделки не более одного current
  approval, но сохраняется ordered history версий. Обязательная связь нового
  процесса с одним `offer_id` удаляется.
- Целевые поля: `version`, `is_current`, `superseded_at`, `workflow_version`,
  `subject_snapshot`, `subject_hash`, `route_context`.
- Ограничения целевой модели: `UNIQUE(deal_id, version)` и partial unique
  `deal_id WHERE is_current`; legacy `offer_id` временно nullable.
- `status` (`ApprovalStatus`), цель v2:
  `not_required` | `draft` | `pending` | `blocked` | `answered` | `canceled`.
- `decision?` (`ApprovalDecision`): `approve` | `reject`; `decision_comment?`.
- `reason_type` (`ApprovalReasonType`): `discount` | `product_rule`.
- `source` (`ApprovalSource`): `email` | `auto_related_deals` | `auto_rule`.
- `approvers` (JSON-список), `requested_at?`, `decided_at?`.
- `email_token?`, `email_token_expires_at?` — для подтверждения согласования по ссылке из письма.
- Связь `stages` загружается как список `ApprovalStageModel`, отсортированный по `position`.
- Маршрут строится по максимальной скидке среди всех offers сделки; особые условия
  любого offer добавляют обязательную цепочку `product_owner → lawyer`;
- После start snapshot неизменяем; повторное согласование создаёт следующую version.

## ApprovalStage — `approval_stages`

Явная стадия последовательного согласования.

- `approval_id` → `approval`, `stage_code` (`ApprovalStageCode`):
  `presale` | `lead` | `sales_director` | `finance_director` | `product_owner` | `lawyer`.
- `position` — порядковый номер стадии внутри approval; `required_permission` — permission,
  необходимая для закрытия стадии.
- `status` (`ApprovalStageStatus`): `waiting` | `pending` | `approved` | `rejected` |
  `skipped` | `canceled`.
- `assigned_user_id?` → `lkm_users`, `assigned_email?`, `requested_at?`, `due_at?`.
- `decided_at?`, `decision?`, `decision_comment?`.
- `email_token?`, `email_token_expires_at?` — stage-level token для будущего workflow.
- `skipped_by_user_id?` → `lkm_users`, `skipped_at?`, `skip_reason?`.
- Ограничения: уникальны `(approval_id, position)`, `(approval_id, stage_code)` и
  `email_token`.

## ApprovalStageToken — `approval_stage_tokens` (целевая модель)

История bearer token стадии: `stage_id`, уникальный `token_hash`, `issued_at`,
`expires_at`, `used_at`, `invalidated_at`. Raw token в БД не хранится. Старые записи
сохраняются для идемпотентной обработки повторных ответов.

## ApprovalStageNotification — `approval_stage_notifications` (целевая модель)

Одна логическая notification generation: `stage_id`, `token_id`, immutable payload
snapshot и временно encrypted raw token.

## ApprovalStageNotificationDelivery — `approval_stage_notification_deliveries`

Получатели transactional outbox: `notification_id`, recipient user/email, delivery
status, attempts, `next_attempt_at`, `sent_at`, `last_error`. Для group stage
несколько delivery rows относятся к одной notification/stage/token.

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

- `lkm_users`: `ad_login`, `full_name`, `role` (`UserRole`:
  `manager` | `presale` | `sales_lead` | `sales_director` | `finance_director` | `product_owner` | `lawyer` | `admin`), `first_login_at?`, `created_by_admin?`.
- `lkm_permissions` (`codename`, `name`, 22 записи), `lkm_roles` (`codename`, `name`).
- Связки: `lkm_role_permissions` (набор прав роли), `lkm_user_permissions` (персональные права).
- Пермиссии/роли и наборы засеваются миграциями `..._add_lkm_permissions.py` и `..._bt02_roles_permissions.py`.
- Эффективные права = пермиссии роли ∪ персональные права. Полная спецификация —
  в исследовании [[LKM Role Model]]; проверка прав — в [[Architecture]].

---

Миграции схемы — Alembic (`migrations/versions/`), см. [[Architecture]].
