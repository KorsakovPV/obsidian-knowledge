---
project: comet-backend
created: 2026-07-23
updated: 2026-07-23
source: docs/lkm_role_model.md
source_pdf: Attachments/БТ02_ Ролевая модель - Datafort DEV - Confluence.pdf
tags: [project, tickets, lkm, permissions, rbac]
---

# BT02 Role Model Gaps

#project #tickets #lkm #permissions #rbac

Источник: [[БТ02_ Ролевая модель - Datafort DEV - Confluence.pdf]], [[LKM Role Model]].

Проверка выполнена по PDF БТ02 от 15.07.2026 и текущему коду `comet-backend` на 2026-07-23.
Код уже содержит базу БТ02: 8 ролей, 22 permissions и миграцию пересборки матрицы
ролей. Ниже — оставшиеся расхождения, которые нужно закрыть отдельными задачами.

## Ticket 1 — Перевести справочники ролей/permissions с `require_admin` на permission guards

Проблема: `GET /lkm/roles` и `GET /lkm/permissions` закрыты `require_admin`, который
проверяет `request.state.lkm_role == admin`. Это противоречит принципу БТ02:
проверки доступа должны идти по permissions, а не по имени роли.

Evidence: `app/api/dependencies/permissions.py`, `app/api/v1/lkm/lkm_role_router.py`,
`app/api/v1/lkm/lkm_permission_router.py`.

DoD:

- Завести отдельные permissions для чтения справочников или использовать существующие
  `change_user_roles` / `change_permissions` по согласованному правилу.
- Убрать role-based проверку из справочников.
- Добавить тесты на доступ к справочникам по permissions.

## Ticket 2 — Запретить администратору менять персональные permissions собственной УЗ

Проблема: БТ02 запрещает менять собственный набор permissions. Смена роли это уже
проверяет, но `PATCH /lkm/users/{user_id}/permissions` не передаёт `admin_ad_login`
в сервис, поэтому self-change не проверяется.

Evidence: `app/api/v1/lkm/lkm_user_router.py`, `app/services/lkm_user.py`.

DoD:

- Передавать текущего администратора в `set_personal_permissions`.
- Если target user совпадает с текущим пользователем — вернуть 403.
- Добавить тест self-change запрета.

## Ticket 3 — Контролировать единоличные permissions

Проблема: БТ02 требует, чтобы `approve_kp_sales_director`,
`approve_kp_finance_director`, `approve_kp_lawyer` принадлежали не более чем одному
пользователю. Сейчас `set_personal_permissions` просто удаляет старый персональный
набор target user и вставляет найденные permissions без проверки других владельцев.

Evidence: `app/cruds/lkm_user.py`, `app/models/lkm_permission.py`.

DoD:

- Перед назначением единоличной permission проверять, что она не выдана другому пользователю.
- При конфликте возвращать доменную ошибку с понятным сообщением.
- Покрыть тестами успешное назначение и конфликт второго владельца.

## Ticket 4 — Вернуть фильтр и очистку ephemeral-пользователей

Проблема: по БТ02 пользователь, добавленный через поиск, но не вошедший и оставшийся
с ролью `manager`, не должен сохраняться в списке после обновления. В коде фильтр
`list_visible()` закомментирован; `delete_ephemeral()` есть, но не используется при
получении списка.

Evidence: `app/cruds/lkm_user.py`, `app/services/lkm_user.py`.

DoD:

- Вернуть фильтр `first_login_at IS NOT NULL OR role != manager`.
- Вызвать очистку ephemeral-пользователей перед выдачей списка или реализовать
  согласованный lifecycle очистки.
- Добавить тесты для видимого пользователя, скрытого ephemeral и удаления ephemeral.

## Ticket 5 — Закрыть pre-tariff ручки permissions из БТ02

Проблема: permissions `create_tariff` / `approve_tariffs` есть в коде, но ручки
`/pre-tariffs` пока не проверяют LKM permissions.

Evidence: `app/api/v1/pre_tariffs/pre_tariff_router.py`.

DoD:

- Навесить `create_tariff` на создание, просмотр и редактирование pre-tariffs.
- Навесить `approve_tariffs` на endpoint согласования тарифов после появления/уточнения endpoint.
- Добавить API-level tests на 403 без permission и успешный доступ с permission.

## Ticket 6 — Передавать permissions в actions при сборке API-ответов

Проблема: action resolvers умеют учитывать permissions, но при сборке `DealSchema` и
`OfferReadSchema` context создаётся без `permissions`. Поэтому фронт видит actions
по состоянию объекта, но не по правам текущего пользователя.

Evidence: `app/services/deal_actions/context.py`, `app/services/offer_actions/context.py`,
`app/services/deal.py`, `app/services/offer.py`.

DoD:

- Протащить `request.state.lkm_permissions` в сборку actions для deal/offer responses.
- Сохранить старое поведение только там, где request context действительно отсутствует.
- Добавить тесты, что action скрывается/запрещается без нужной permission в API-ответе.

## Ticket 7 — Решить расхождение целевого дизайна `roles as data` и текущего `UserRole` enum

Проблема: целевой дизайн в [[LKM Role Model]] описывает роли как данные, чтобы новые
роли добавлялись без изменения кода. Текущая реализация содержит `UserRole(StrEnum)`
и использует его в схемах, сервисах и проверках.

Evidence: `app/models/lkm_user.py`, `app/schemas/lkm_user.py`, `app/services/lkm_user.py`.

DoD:

- Принять архитектурное решение: оставить enum как сознательное ограничение или
  убрать enum и валидировать роли по `lkm_roles`.
- Если убираем enum — обновить схемы, service/crud, миграции/seed и тесты.
- Если оставляем enum — обновить [[LKM Role Model]] как accepted deviation от БТ02.

## Ticket 8 — Добавить тесты для LKM service и BT02 migration matrix

Проблема: текущие тесты покрывают middleware и базовые action permissions, но не
проверяют сервис управления LKM-пользователями, матрицу ролей БТ02, single-holder
permissions, ephemeral lifecycle и pre-tariff guards.

Evidence: `tests/test_action_permissions.py`, `tests/middleware/test_auth_middleware.py`.

DoD:

- Добавить unit/integration tests для `LkmUserService.change_role`,
  `set_personal_permissions`, `list_visible/delete_ephemeral`.
- Добавить тест матрицы `lkm_role_permissions` из BT02 migration/seed.
- Добавить tests для новых guards из tickets 1, 3, 5, 6.
