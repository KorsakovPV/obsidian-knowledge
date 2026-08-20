---
project: service-catalog
created: 2026-08-20
tags: [project, api, django]
---

# API

Схема и Swagger — `/api/docs/` (drf-spectacular). Разбор моделей и прав — в
[[Architecture]], общее описание — в [[Overview]].

Роутинг верхнего уровня (`backend/app/urls.py`):

| Префикс | Приложение | Назначение |
| --- | --- | --- |
| `/api/` | `classifier` | основное API классификатора (для фронта) |
| `/api/` | `changelog` | журнал изменений |
| `/api/` | `user` | текущий пользователь, права, настройки таблицы |
| `/api/listapi/` | `listapi` | read-only API для сторонних сервисов |
| `/admin/` | `app` | кастомный админ-сайт со входом через AD |

`/api/auth/login/` из CMS намеренно закрыт (`forbidden_view`).

## Аутентификация

- Токен: `POST /api/ldap_login/` — логин/пароль проверяются в AD/Keycloak
  (`LdapLoginView`, `user/utils/utils.py`), в ответ — access/refresh.
- Обновление: `POST /api/ldap_refresh/`.
- Далее — заголовок с токеном (`app/authentication.py::Authentication`) либо
  сессия Django.
- TTL токенов: `GARPIX_ACCESS_TOKEN_TTL_SECONDS = 36000`,
  `GARPIX_REFRESH_TOKEN_TTL_SECONDS = 5 дней`.
- Права на конкретные операции считаются от групп AD (см. [[Architecture#Права доступа]]).

## Внутреннее API классификатора (`/api/`)

ViewSet-ы (DRF router):

| Эндпоинт | Класс | Возможности |
| --- | --- | --- |
| `/api/tariffs/` | `TariffListViewSet` | CRUD тарифов, фильтры `TariffFilter`, сортировка по составному коду (`ConcatCodeOrderingFilter`), пагинация; `DELETE` запрещён (только псевдоудаление) |
| `/api/products/` | `ProductListViewSet` | read-only дерево продукта с группами и тарифами, кэш в Redis, экшены сброса кэша |
| `/api/products_groups/` | `ProductGroupListViewSet` | список групп продуктов |

Справочники (`classifier/views/list_api_views.py`, `ListCreateAPIView` —
GET-список и создание без дублей, права `ViewModelPermissions`):

`/api/tariff_element/`, `/api/tariff_short/`, `/api/unit/`,
`/api/charge_type/`, `/api/service_element/`, `/api/service/`,
`/api/service_line/`, `/api/service_business/`, `/api/currency/`,
`/api/update_frequency/`, `/api/location/`.

Экшены `TariffListViewSet` (`classifier/views/tariff_view_set.py`):

| Эндпоинт | Метод | Назначение |
| --- | --- | --- |
| `/api/tariffs/query_tariff/` | POST | выборка тарифов с фильтрами в теле запроса, ответ кэшируется в Redis, `count_all` в ответе; логирование изменений выключается на время запроса |
| `/api/tariffs/delete/` | POST, DELETE | псевдоудаление массивом `pk` |
| `/api/tariffs/to_archive/` | PUT, PATCH | архивирование массивом `pk` |
| `/api/tariffs/<pk>/set_related_tariffs/` | PUT, PATCH | задать связанные тарифы |
| `/api/tariffs/<pk>/related_tariffs/` | GET | дерево связанных тарифов |
| `/api/tariffs/<pk>/flat_related_tariffs/` | GET | плоский список связанных тарифов |

Экшены `ProductListViewSet`: `DELETE /api/products/<product_id>/drop_cache` и
`DELETE /api/products/drop_cache` — сброс кэша одного продукта или всех.

Дополнительно:

- `/api/filter/` (`FilterView`) — наборы значений для фильтров таблицы фронта.
- `/api/excel/` (`ExcelView`, GET и POST) — выгрузка Excel с учётом фильтров и
  пользовательских настроек таблицы; датафрейм собирается pandas, права —
  `IsInADGroupPermission`.
- `/api/logs/` (`LogView`) — журнал изменений, только чтение, пагинация,
  фильтры по модели/автору/дате.

### Валидации при записи тарифа

`classifier/serializers/tariff_serializer.py::TariffSerializer`:
`validate()` проверяет логику цен и дат, `check_concated_code_unchanged` —
неизменность составного кода при обновлении, `check_service_line_unchaged` —
что не подменили линию услуг. Связанные тарифы и локации проставляются через
`Tariff.set_related_tariffs()` и `set_locations()` с асинхронным логированием.

## Внешнее API для сторонних сервисов (`/api/listapi/`)

Только чтение, `IsAuthenticated`. `GET /api/listapi/` отдаёт список доступных
путей.

| Эндпоинт | Методы |
| --- | --- |
| `/api/listapi/tariffs/`, `/tariffs/<pk>/` | list, retrieve |
| `/api/listapi/query_tariff/` | GET и POST — выборка тарифов по набору условий (POST для длинных запросов) |
| `/api/listapi/tariff_element/`, `/tariff_short/`, `/unit/`, `/charge_type/` | list, retrieve |
| `/api/listapi/service/`, `/service_element/`, `/service_line/`, `/service_business/` | list, retrieve |
| `/api/listapi/resource_mapping/` | list |
| `/api/listapi/tariff_to_resources/` | list |
| `/api/listapi/tariff_to_resources/by_tariff_id/<tariff_id>/` | ресурсы конкретного тарифа |

Именно этот префикс используют соседние сервисы (order-processing,
comet-backend), поэтому его контракт менять осторожно.

## API пользователя (`/api/`)

| Эндпоинт | Назначение |
| --- | --- |
| `GET /api/userdata/` | данные текущего пользователя и его права для фронта |
| `POST /api/table_settings/` | сохранить настройку таблицы |
| `DELETE /api/delete_setting/<pk>` | удалить свою настройку (проверка владельца) |
| `POST /api/ldap_login/`, `POST /api/ldap_refresh/` | токены |
| `GET /api/send_test_error_to_sentry/` | проверка интеграции с Sentry |

## Форматы

Заданы в `REST_FRAMEWORK` (`app/settings.py`): пагинация
`PageNumberPagination` с `PAGE_SIZE = 10`, даты — `%d.%m.%Y`, дата-время —
`%d.%m.%YT%H:%M%z`.

## Кто потребляет API

- **comet-backend** (`app/services/classifier.py`) ходит в
  `/api/products/`, `/api/products_groups/`, `/api/listapi/query_tariff/`,
  авторизуется через `/api/ldap_login/` и `/api/ldap_refresh/`.
- **order-processing** (`app/helpers/service_catalog.py`, переменная
  `SERVICE_CATALOG_BASE_URL`) ходит POST-ом в `/api/tariffs/query_tariff/`
  и разбирает дерево `tariff → service_element → service → line → line_business`.

Поэтому контракты `query_tariff`, `/api/products/` и всего `/api/listapi/`
менять осторожно — это внешние потребители.
