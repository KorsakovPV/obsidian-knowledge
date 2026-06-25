---
project: order-processing
created: 2026-06-25
tags: [project, research]
---

#research

> Поэтапная поддержка дробного количества в тарифах. Связано: [[Overview]],
> [[API#Количество тарифа (quantity / quantity_decimal)]], [[Architecture]].
> Источник истины — код; заметка описывает дизайн и этапы.

# Дробное количество в тарифах

## Зачем

Бизнесу нужно продавать дробные количества (напр. ИТ-специалист в объёме 0,5 часа).
Раньше количество тарифа (`quantity`) было целым `int`. Тарифы хранятся в JSONB
(`order_v2.tariffs`, `offer_v2.tariffs`, `tariff_commercial_offers.tariffs`,
`approval.tariffs`), где `quantity` — ключ внутри каждого объекта тарифа.

## Модель данных и схема

- Новый ключ `quantity_decimal: str` рядом с `quantity` — **канонический** источник
  количества, дробное строкой (напр. `"0.5"`).
- `quantity: int` — **DEPRECATED** (помечено в OpenAPI), для обратной совместимости.
- `OrderTariffsSchema` (`app/schemas/orders.py`):
  - `set_quantity_decimal` (pre-root-валидатор) — бэкфилл `quantity_decimal` из
    `quantity` (чтобы попадало в `__fields_set__` и сохранялось при
    `dict(exclude_unset=True)` в `order_v2_crud.create_order`);
  - property `quantity_value -> Decimal` — единый числовой аксессор для расчётов
    (приоритет `quantity_decimal`, затем `quantity`, иначе `0`).
- Поле добавлено и в `TariffsToAPISchema` (`schemas/v2/tariffs.py`) и в схему
  продюсера событий `TariffSchema` (`schemas/v2/producers.py`).

## Миграция БД

Alembic `2026_06_25_1200-a1b2c3d4e5f6_add_quantity_decimal_to_tariffs.py` — проходит
по JSONB-тарифам `order_v2`, `offer_v2`, `tariff_commercial_offers`, `approval` и
бэкфиллит `quantity_decimal = str(quantity)`. Идемпотентна, есть `downgrade`.

## Decimal-fallback в пайплайне

Расчёты/валидаторы v2-пайплайна читают `quantity_value` (не устаревшее `quantity`):
- `tariffs_validator.py` — диапазон и кастомные валидаторы;
- `custom_tariff_validators.py` — суммы Sum/Group;
- `tech_params_validator.py` — сумма linked-ТП;
- `order_transform_manager.py` — PDF (`quantity_decimal` → `TariffSchemaPDF.quantity`)
  и Kafka; `order_price_manager.py` уже считал из строки.

## Integer-доменные правила (MDM / PostgreSQL)

Для тарифов правил Cloud MDM (`cloud_mdm_admins`) и PostgreSQL (`postgresql_rule`)
дробное количество бессмысленно (0,5 администратора / дробное число узлов).

- `RuleManager.integer_only_tariff_pks(rule, groups)` отдаёт pk integer-доменных
  тарифов: MDM — ключи `rule.data.mapping`; PG — `tariff_ids` зависимого ТП.
- `tech_params_validator.validate_integer_only_tariffs()` (выполняется **до**
  `validate_rules()`) запрещает дробное для таких тарифов →
  `FractionalQuantityNotAllowedError`.
- Сами rule-менеджеры тоже переведены на `quantity_value` (+ `int()`), чтобы не
  падать на `None` до срабатывания запрета.

## API / Swagger

`quantity_decimal` имеет `description`/`example`, `quantity` помечен `deprecated=True`
(pydantic v1 `Field` → OpenAPI). Пример продюсера в `example_responses.py` обновлён.

## Вне объёма (следующие этапы)

- Legacy `forms.py` (~50 мест `int(tariff.get('quantity'))`), `helpers/validators.py`,
  `helpers/form_loader.py`, `actualize_order` — миграция на Decimal.
- Физическое удаление поля `quantity` после перехода всех потребителей.
