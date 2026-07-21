---
project: comet-backend
created: 2026-07-21
source: docs/preapproved_order_integration.md
tags: [project, research, order-processing, integration, approval]
---

# DFDEV-1908: Интеграция с ОП после согласования оффера

#research #project

> Интеграция с Order Processing по офферу. Сущность `OfferOrder` описана в
> [[Domain Model]], процесс согласования — в [[Architecture]] и [[Offer Actions Rules]].

## Цель

После перехода approval оффера в состояние `answered / approve` — автоматически создавать
предварительно согласованный заказ в Order Processing (ОП). (До уточнения у системного аналитика не реализовывать этот пункт)

---

## Триггер

Вызов `POST /api/v2/internal/orders/preapproved/new` происходит когда:

```
approval.status   = answered
approval.decision = approve
```

---

## Что передаётся в ОП

```json
{
  "external_approval_id": "<UUID approval из Comet>",
  "order": {
    "...": "данные оффера в формате OrderNewCreateSchemaIn"
  }
}
```

`external_approval_id` — это `approval.id` из Comet. Используется как ключ идемпотентности в ОП.

---

## Что сохраняется в Comet

После успешного ответа ОП в `OfferOrderModel` сохраняются:

| Поле | Источник |
|---|---|
| `order_id` | `response.id` — UUID заказа в ОП |
| `order_number` | `response.order_number` |
| `order_status` | `response.itsm_state` |

---

## Идемпотентность и обработка ошибок

| Ситуация | Действие |
|---|---|
| ОП вернул `201` | Сохранить `order_id` и `order_number` в `OfferOrderModel` |
| ОП вернул `200` (повтор) | Сохранить данные из ответа, не является ошибкой |
| Timeout / сетевая ошибка | Выполнить `GET /api/v2/internal/orders/by-external-approval/{external_approval_id}` |
| `GET` вернул `404` | Повторить создание |
| `GET` вернул `200` | Сохранить данные из ответа |

---

## Связь с `OfferOrderModel`

`OfferOrderModel` (`app/models/offer_order.py`) хранит результат имплементации оффера в ОП:

```
offer_id      FK → offer.id (CASCADE)
order_id      UUID БЗ в ОП
order_number  номер БЗ
order_status  itsm_state из ОП (кешируется)
```

Поле `order_status` обновляется при получении актуального статуса из ОП.

---

## Сброс согласования при редактировании одобренного оффера

Если оффер со статусом `APPROVED` редактируется, approval сбрасывается в `DRAFT`:

```
status          → DRAFT
decision        → null
decision_comment → null
decided_at      → null
```

После сброса оффер требует повторной отправки на согласование перед новой имплементацией.
Ранее созданный `OfferOrderModel` остаётся без изменений — он отражает факт прошлой имплементации.

---

## Логирование

При каждом вызове логировать:
```
external_approval_id
offer_id
order_id (из ответа ОП)
order_number (из ответа ОП)
result: created | reused | failed
```
