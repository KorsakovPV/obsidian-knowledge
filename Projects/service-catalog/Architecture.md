---
project: service-catalog
created: 2026-08-20
tags: [project, architecture, django]
---

# Architecture

Точка входа — `backend/manage.py`, настройки `backend/app/settings.py`
(наследуются от `garpixcms.settings`), корневой роутер `backend/app/urls.py`.
Подробнее об эндпоинтах — [[API]]. Общее описание проекта — [[Overview]].

## Приложения Django

```
backend/
├── app/          настройки проекта, админ-сайт, аутентификация, права, celery
├── classifier/   основной домен: услуги, тарифы, продукты, Excel
├── changelog/    журнал изменений и рассылка уведомлений
├── listapi/      read-only API для сторонних сервисов
└── user/         пользователи, AD/LDAP, Keycloak, настройки таблиц
```

Дополнительно к `INSTALLED_APPS` из garpixcms подключаются `classifier`,
`changelog`, `listapi`, `django_filters`, `minio_storage`, `cuser`,
`nested_admin`.

## Доменная модель (classifier)

Иерархия каталога — четыре уровня, у каждого свой числовой код:

```
ServiceBusiness (LOB)
└── ServiceLine (Service Line)
    └── Service
        └── ServiceElement
            └── Tariff
```

- `Tariff` (`classifier/models/tariff.py`) — центральная сущность: ссылка на
  `ServiceElement` и на `TariffElement`, даты старта/завершения, `access_period`
  (срок подключения), `related_tariffs` (M2M на себя — связанные тарифы),
  `location` (M2M на `Location`), `num_clients`, `is_price_protected`,
  `price_code` и вычисляемый составной код (`current_concated_code`,
  `update_price_code`, `get_tariff_with_max_price_code`).
- `TariffElement` — то, что тарифицируется: `unit` (`Unit`),
  `charge_type` (`ChargeType`), цены `price` / `cost_price` / `beeline_price`,
  `currency`, `flat_rate`, `update_frequency`; свойства `default_price`,
  `min_default_price`, `partner_price`, `min_partner_price`, расчёт
  `get_max_discount` и округление денег `money_rounding`.
- `UpdateTariffElementFrequency` — частота обновления данных; при сохранении
  пересчитывает `fact_freq` и человекочитаемое `readable_freq`.
- `ResourceMapping` + `TariffToResources` — привязка тарифа к ресурсам с
  множителем (уникальная пара тариф-ресурс).
- `Location` — локации предоставления услуги.

### Продукты и тех-параметры

`classifier/models/product.py` описывает витрину для КП:

```
ProductGroup → Product → GroupLevelOne → GroupLevelTwo → GroupLevelThree
                                                          └── GroupLevelFour → GroupTariff → Tariff
Product → GroupProductTechParams → ProductTechParams
```

- Группы наследуют `BaseGroup` (`name`, `action_type`, `position`,
  `enable_for` / `disable_for` в JSON) и связаны через `parent`/`children`.
- `GroupLevelFour` — тарифная группа: M2M на `Tariff` через `GroupTariff`
  (`params` в JSON, `position`), плюс `validators`.
- `ProductTechParams` — технический параметр: `tariff_ids`,
  `tariff_group_ids`, `depend_on_ids` (`ArrayField`) и правила связи
  (`one_from` / `all`), `field_validation` (regex), `itsm_key` для передачи в
  ITSM.

Карточка продукта в админке рендерит 4 уровня вложенных инлайнов
(`nested_admin`). Из-за этого в `settings.py` поднят
`DATA_UPLOAD_MAX_NUMBER_FIELDS = 10000`, а пустые инлайн-формы выбрасываются
из POST скриптом `classifier/static/classifier/js/skip_empty_inline_forms.js`
(правка DFDEV-2196 — лечила `Bad Request 400` на сохранении продукта).

## Правила создания и изменения тарифа

Логика живёт в `classifier/serializers/tariff_serializer.py::TariffSerializer`
и в `Tariff` — это фактический бизнес-контракт каталога.

### Тарифный элемент шарится

`TariffElement` создаётся через `get_or_create` по всем полям сразу (единица,
тип потребления, цены, валюта, частота, `flat_rate`). Два тарифа с одинаковыми
параметрами ссылаются на одну строку, поэтому мутация элемента задела бы чужие
тарифы — при изменении цены серверу нужно подменять элемент, а не править его.
То же самое касается ветки услуги: она создаётся каскадным `get_or_create`
(линия бизнеса → линейка → услуга → элемент услуги).

### Валидации

- Коды всех пяти уровней обязательны и лежат в диапазоне 0–32767
  (`check_codes`); поля `PositiveSmallIntegerField`.
- Цены: `price ≥ cost_price ≥ 0`, длина `cost_price` до 11 символов, даты —
  `start_at` обязательна и строго раньше `end_at`.
- Особый случай: если тип тарификации равен `SOR`
  (`settings.CHARGE_TYPE_WITHOUT_PRICE`), себестоимость и базовая цена
  принудительно обнуляются — такие услуги не имеют цены.
- `max_discount` элемента считается автоматически (`get_max_discount`).

### Версионирование и дубли

`is_code_valid` при создании проверяет, нет ли уже **доступной** услуги
(`is_available=True`) с тем же составным кодом и той же ценой — если есть,
возвращается 400 «уже существует». Версия цены `price_code` берётся как
`max + 1` по тому же составному коду **с учётом архивных** тарифов, иначе после
архивирования версия переиспользовалась бы. Составной код — пять кодов ветки
услуги плюс `price_code` шестым сегментом (`current_concated_code`).

### Что менять нельзя

- Базовую цену после создания (`check_price_unchanged`) — новая цена означает
  новую версию тарифа.
- Составной код и ветку услуги при обновлении
  (`check_concated_code_unchanged`, `check_service_line_unchaged`).
- Любой тариф с подключёнными клиентами: `update()` и `set_related_tariffs`
  падают при `num_clients != 0`.

### Архивирование и удаление

`to_archive` не удаляет запись: проставляет `archived_at` и перевешивает тариф
на копию ветки услуги с `is_available=False` (`Service`/`ServiceElement`
создаются через `get_or_create`). Тариф остаётся читаемым по pk, поэтому
внешние потребители, которые ссылаются на него, не ломаются. Мягкое удаление
(`delete`) убирает тариф из списочных выдач, включая `query_tariff`.

### Видимость в каталоге продуктов

В дерево продукта тариф попадает только через привязку к тарифной группе
(`GroupLevelFour` → `GroupTariff`). Ограничения количества (`min`, `max`,
`step`) — свойство именно этой связки: `GroupTariff.params` по умолчанию
хранит тройку отдельно для каждого типа бланка заказа — `inner`, `commercial`,
`test` (`default_tariff_params`).

## Псевдоудаление

`classifier/mixins/delete_mixin.py`: `DeleteMixin` переопределяет `delete()`
на проставление флага, а `NotDeletedManager` отдаёт живые записи через
`Model.not_deleted_objects`. Реальное удаление — только `hard_delete()`;
через API оно запрещено (`TariffListViewSet.destroy`).

## Журнал изменений (changelog)

- `ChangeLogMixin` (`changelog/mixins/log_mixin.py`) запоминает значения полей
  при `__init__` и отдаёт диф через `get_changed_fields()`; примешан почти во
  все модели каталога.
- Сигналы `changelog/signals/changes_signals.py` (`pre_save`, `post_save`,
  `pre_delete`) пишут `ChangeLog` — модель, `record_id`, автор (`django-cuser`),
  действие и JSON с изменениями.
- `ChangeLog.add()` дополнительно рассылает уведомления через
  `garpix_notify` всем активным пользователям группы из синглтона
  `MailGroupConfiguration` (событие `settings.CHANGES_EVENT`).
- `LogMiddleware` (`changelog/middleware/log_middleware.py`) хранит флаг
  логирования в текущем потоке — логирование отключают на тяжёлых операциях,
  чтобы не просаживать производительность.
- Изменения M2M (связанные тарифы, локации) логируются асинхронно задачами
  `classifier/tasks.py`: `log_matrix_change`, `log_locations_change`.

## Права доступа

`app/permissions/model_permission.py`:

- `ViewModelPermissions` — `DjangoModelPermissions` с добавленной проверкой
  права на `GET`.
- `TariffViewModelPermissions` — объектные права на тариф: при `PUT`/`PATCH`
  общая проверка пропускается, а `has_object_permission` сравнивает
  изменяемые поля (`get_diff_tariff`) с правами пользователя — отдельно
  контролируются цена и комментарий.
- `IsInADGroupPermission` — минимальное право на чтение (пользователь состоит
  в нужной группе AD).

Группы приходят из AD/Keycloak: `user/models/user_model.py`
(`get_user_ad_group`, `get_user_keycloak_group`, `is_user_active_and_ad_admin`),
настройки серверов — синглтоны `ActiveDirectoryServer`, `ActiveDirectoryPath`,
`AdminGroupSetting` в `user/models/ldap_model.py` (правятся через админку).
Периодическая задача `user/tasks.py::get_active_directory_groups` обновляет
список групп.

Аутентификация DRF — `app/authentication.py` (`Authentication`
поверх `TokenAuthentication`) плюс сессионная. Штатный эндпоинт получения
токенов CMS отключён (`forbidden_view` на `/api/auth/login/`).

## Админка

`app/admin.py` собирает кастомный `ADLoginAdminSite` с формой входа через AD
(`app/forms`) и регистрирует модели garpix_notify. `classifier/admin.py` —
админки тарифов и справочников, импорт/экспорт продуктов и Excel-формы.

## Фоновые задачи и кэш

- Celery (`app/celery.py`) — брокер и бэкенд Redis (`redis://$REDIS_HOST:6379/1`).
- `classifier/tasks.py::update_product_cache` пересобирает кэш продуктов;
  ручной сброс — экшены `drop_cache` / `drop_all_cache` в
  `ProductListViewSet`. Утилиты кэша — `classifier/utils.py`
  (`cache_response`, `update_cache`).

## Прочие утилиты

`classifier/utils.py`: `NoWhiteSpaceSearchFilter` (поиск без разбиения по
пробелам и по датам в формате таблицы), `flatten` и
`make_related_tariffs_dict` (плоское и древовидное представление связанных
тарифов), исключения `ObjectAlreadyExist`, `ObjectWithTheseCodeExist`,
`retry()` для HTTP-сессий.

## Тесты

Тесты лежат внутри приложений: `classifier/tests/` (`test_models.py`,
`test_views.py`, `test_load.py`), `changelog/tests/`, `user/tests/test_ad.py`.
CI (`.gitlab-ci.yml`, job `Test 01`) поднимает postgres 13 и redis 7 и гоняет
`manage.py migrate` + `manage.py qa`.
