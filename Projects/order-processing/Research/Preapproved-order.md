---
project: order-processing
created: 2026-06-25
source: docs/preapproved_order.md
tags: [project, research]
---

#research

> Перенесено из репозитория (`docs/preapproved_order.md`). Источник истины —
> файл в репозитории; здесь — копия с обсидиановым оформлением.
> Связано: [[Overview]], [[API#Internal-эндпоинты]], [[Architecture]].

# DFDEV-1908: Создание предварительно согласованного заказа

## Цель

Добавить возможность создать заказ, который уже был согласован во внешнем доверенном сервисе (Comet).
В результате одной атомарной операции создаются заказ и локальный approval со статусом `Согласован`.

---

## Изменения модели данных

### Новый источник согласования

В `ApprovalSource` (`approval_helpers.py`) добавлен:

```
EXTERNAL_SERVICE = 'ExternalService'
```

### Новое поле в таблице `approval`

```
external_approval_id  uuid  nullable
```

UUID согласования из внешнего сервиса. Null для всех существующих типов согласований.

### Уникальный частичный индекс

```sql
CREATE UNIQUE INDEX uq_approval_external_approval_id
ON approval (external_approval_id)
WHERE external_approval_id IS NOT NULL;
```

Гарантирует, что одно внешнее согласование не создаёт два заказа.

### CHECK-ограничение

```sql
CHECK (source != 'ExternalService' OR external_approval_id IS NOT NULL)
```

---

## Новые ручки

### POST /api/v2/internal/orders/preapproved/new

Создаёт заказ с предварительно согласованным approval.

**Тело запроса:**
```json
{
  "external_approval_id": "uuid",
  "order": { "...": "поля OrderNewCreateSchemaIn" }
}
```

**Ответ:**
- `201 Created` — заказ создан, тело `OrderV2OutSchema`
- `200 OK` — идемпотентный повтор, возвращён существующий заказ

В `approval` дополнительно возвращаются `source` и `external_approval_id`.

**Правила создания approval:**
```
status               = Согласован
source               = ExternalService
external_approval_id = UUID из запроса
requested_from       = []
tariffs              = tariffs_for_approve или []
created_by           = order.created_by
```

### GET /api/v2/internal/orders/by-external-approval/{external_approval_id}

Lookup-ручка для восстановления после timeout. Возвращает заказ по `external_approval_id` согласования.

- `200 OK` — заказ найден, тело `OrderV2OutSchema`
- `404 Not Found` — заказ не найден

---

## Транзакционность

Заказ и approval создаются в одной транзакции:
- либо созданы оба объекта;
- либо не создано ничего.

---

## Идемпотентность

`external_approval_id` — бизнес-ключ идемпотентности.

| Сценарий | HTTP-код | Поведение |
|---|---|---|
| Первый запрос | `201` | Создать заказ и approval |
| Повторный запрос | `200` | Вернуть существующий заказ без создания |

---

## Авторизация (следующий тикет)

На первом этапе ручки доступны без дополнительных ограничений доступа.
Планируется: Keycloak service account + scope `orders:create-preapproved` / `orders:read-preapproved`.
