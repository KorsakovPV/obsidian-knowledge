---
project: comet-backend
created: 2026-06-25
tags: [project, domain, database, sqlalchemy]
---

# Domain Model

#project

Доменная модель [[Overview|comet-backend]]. ORM-модели — в `app/models/`, общий `Base`,
PostgreSQL. Ниже основные таблицы и связи. Бизнес-смысл сущностей см. в [[Overview]],
процессы — в [[Architecture]].

## Связи (кратко)

```
Deal 1───* Offer 1───0..1 Approval
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

## Approval — `approval`

Согласование оффера/сделки.

- `offer_id` → `offer`, `deal_id` → `deal`.
- `status` (`ApprovalStatus`): `draft` | `pending` | `answered` | `canceled`.
- `decision?` (`ApprovalDecision`): `approve` | `reject`; `decision_comment?`.
- `reason_type` (`ApprovalReasonType`): `discount` | `product_rule`.
- `source` (`ApprovalSource`): `email` | `auto_related_deals` | `auto_rule`.
- `approvers` (JSON-список), `requested_at?`, `decided_at?`.
- `email_token?`, `email_token_expires_at?` — для подтверждения согласования по ссылке из письма.

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
  `manager` | `sales_lead` | `director` | `admin`), `first_login_at?`, `created_by_admin?`.
- `lkm_permissions` (`codename`, `name`), `lkm_roles` (`codename`, `name`).
- Связки: `lkm_role_permissions`, `lkm_user_permissions`.
- Полная спецификация прав — `docs/lkm_role_model.md` в репозитории кода.

---

Миграции схемы — Alembic (`migrations/versions/`), см. [[Architecture]].
