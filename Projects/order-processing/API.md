---
project: order-processing
created: 2026-06-25
tags: [project, api]
---

# API

Публичные интерфейсы сервиса. Точка входа — [[Overview]], структура — [[Architecture]].
Роутеры подключаются в `app/main.py`.

## Версии API

### API v1 (`api/api_v1`) — legacy

Минимальный набор, «ручка для БЗ» и проверочный эндпоинт Sentry
(`endpoints/order.py`, `endpoints/sentry_api.py`).

### API v2 (`api/api_v2`) — основной

Подключается в `api/api_v2/api.py` как `api_router_v2`. Роутеры по доменам
(тег → файл в `api/api_v2/endpoints/`):

| Тег (Swagger) | Эндпоинты |
|---|---|
| API V2: Аутентификация | `api_auth.py` |
| API V2: продукты | `products.py` |
| API V2: проекты | `projects.py` |
| API V2: коммерческие предложения | `offers.py` |
| API V2: бланки заказа | `orders.py` |
| API V2: Имплементация бланков заказа через itsm | `itsm_implement_order.py` |
| API V2: Клиенты | `clients.py` |
| API V2: Контракты | `contracts.py` |
| API V2: Стафферы | `staffers.py` |
| API V2: Общая информация | `info.py` |
| API V2: Проекты заказов | `order_projects.py` |
| API V2: Информация для биллинга | `billing.py` |
| API V2: Управление кэшем | `cache.py` |
| API V2: Создание PDF | `pdf.py` |
| API V2: Тарифы | `tariffs.py` |
| API V2: Услуги | `services.py` |
| API V2: Internal | `internal_orders.py` |

> Также есть `endpoints/regions.py`. Обработка ошибок и примеры ответов —
> `api/api_v2/error_handling.py`, `example_responses.py`, `exceptions.py`
> (в т. ч. `RulesTPValidationError` для валидации тех. параметров).

## Формы заказов / TCO (`urls/`)

Помимо JSON-API, сервис рендерит серверные формы заказа и расчёта TCO по каждому
продукту. Для каждого продукта — пара роутов `*_order_url` и `*_tco_url`
(например `cloud_compute_order_url` / `cloud_compute_tco_url`, `veeam_backup_*`,
`cloud_ngfw_*`, `df_workspace_*` и т. д.). Сборка — в `urls/urls.py` (`url_router`),
PDF — `urls/create_pdf.py`.

## Internal-эндпоинты

Группа `internal` (`endpoints/internal_orders.py`) — служебные ручки. В частности,
создание предварительно согласованного заказа из внешнего сервиса:

- `POST /api/v2/internal/orders/preapproved/new`
- `GET /api/v2/internal/orders/by-external-approval/{external_approval_id}`

Подробнее — [[Preapproved-order]].

## Аутентификация

Через Keycloak (`datafort_utils.auth`, `core/config.py: KeycloakSettings`,
`services/keycloak.py`). Защита путей — middleware `check_api_protected_paths`
(`middleware/middleware_path_pass.py`).