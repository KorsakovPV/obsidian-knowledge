---
project: comet-backend
created: 2026-07-21
updated: 2026-07-21
source: docs/lkm_role_model.md
tags: [project, research, lkm, permissions, rbac]
---

# ЛКМ: Ролевая модель — Архитектурный анализ

#research #project

> [!note] Целевой дизайн vs. текущий код
> Эта заметка — перенос исследования БТ02 (целевая модель, **8 ролей**, синк
> пермиссий из enum на старте). Текущий код в репозитории реализует **MVP-срез**:
> 4 роли (`manager / sales_lead / director / admin`), 16 пермиссий, сид ролей и
> пермиссий через **миграцию** (а не enum-sync на старте), а guard `require_admin`
> пока проверяет имя роли. Актуальную реализацию см. в [[Architecture]] (раздел
> «Права доступа») и связанной [[Offer Actions Rules]].

Источник: БТ02 «Ролевая модель» (Жуков К.С.), обновление от 15.07.2026.

> Источник истины по модели прав — **Таблица ролей (раздел 1.3)**. В ней
> согласование КП разбито на отдельные стадии (по одной пермиссии на согласующего).
> Чек-лист администратора (раздел 5.2) будет приведён аналитиком в соответствие с
> Таблицей 1.3, поэтому ориентируемся только на 1.3.

## Роли

БТ02 определяет **8 ролей**. Роль «Менеджер продаж» — базовая, каждая следующая
роль включает функционал предыдущих. Администратор ЛКМ включает функционал всех
бизнес-ролей плюс управление пользователями.

| Роль | codename | Назначение | Способ назначения |
|------|----------|-----------|-------------------|
| Менеджер продаж | `manager` | Базовая роль: сделки + чат с согласующими | Автоматически при первом входе |
| Пресейл | `presale` | + согласование КП на стадии пресейла + валидация БЗ | Администратором ЛКМ |
| Руководитель по продажам | `sales_lead` | + согласование КП руководителя + валидация БЗ | Администратором ЛКМ |
| Директор по продажам | `sales_director` | + согласование КП директора по продажам | Администратором ЛКМ |
| Финансовый директор | `finance_director` | + согласование кастомных тарифов + чат с менеджерами | Администратором ЛКМ |
| Владелец продукта | `product_owner` | Согласование КП по технической возможности услуг | Администратором ЛКМ |
| Юрист | `lawyer` | Согласование КП по нестандартным условиям (юридическим) | Администратором ЛКМ |
| Администратор ЛКМ | `admin` | Управление пользователями/ролями + функционал всех бизнес-ролей | Другим администратором |

> Роли Пресейл, Директор по продажам, Владелец продукта и Юрист добавлены в БТ02
> 17.06.2026. Ранее модель состояла из 4 ролей (manager / sales_lead / director / admin).

## Матрица пермиссий

Столбцы соответствуют ролям из раздела 1.3 БТ02.

| Пермиссия (codename) | Менеджер | Пресейл | Руководитель | Директор по продажам | Финдиректор | Владелец продукта | Юрист | Админ |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `view_deals` — Просмотр всех сделок | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `edit_deal` — Изменение данных сделки | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ |
| `change_deal_status` — Изменение статуса сделки | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ |
| `create_kp` — Создание КП | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ | ✓ |
| `send_kp` — Отправка КП в имплементацию | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ | ✓ |
| `create_deal` — Создание новой сделки | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ | ✓ |
| `import_deal` — Импорт новой сделки | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ | ✓ |
| `view_kp` — Просмотр всех КП | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `approve_kp_presale` — Согласование КП от пресейла | ✗ | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ |
| `approve_kp_lead` — Согласование КП от руководителя | ✗ | ✗ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ |
| `approve_kp_sales_director` — Согласование КП от директора по продажам ¹ | ✗ | ✗ | ✗ | ✓ | ✓ | ✗ | ✗ | ✓ |
| `approve_kp_finance_director` — Согласование КП от финансового директора ¹ | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✓ |
| `approve_kp_product_owner` — Согласование КП от владельца продукта | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ |
| `approve_kp_lawyer` — Согласование КП от юриста ¹ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| `validate_bz` — Валидация БЗ ² | ✗ | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| `chat_with_approvers` — Чат с согласующими | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| `chat_with_managers` — Чат с менеджерами продаж | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `add_users` — Добавление пользователей | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| `change_user_roles` — Изменение роли пользователям | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| `change_permissions` — Изменение набора полномочий | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

¹ Пермиссия может быть назначена **только одному пользователю** одновременно
  (ограничение из БТ02). Требует контроля уникальности на уровне сервиса при
  назначении кастомных пермиссий / смене роли.
² Пермиссия «Валидация БЗ» добавлена в БТ02 15.07.2026.

### Тарифные пермиссии

В Таблице 1.3 строк по тарифам нет, но пермиссии присутствуют в чек-листе
администратора (раздел 5.2) и в описании роли «Финансовый директор» («согласование
кастомных тарифов»). Приводятся здесь для полноты; точное распределение по ролям
требует подтверждения аналитика (см. «Открытые вопросы»).

| Пермиссия (codename) | Кому | Комментарий |
|---|---|---|
| `create_tariff` — Создание тарифа | Менеджер и выше (роли, работающие со сделками) + Админ | Создавать предтариф может менеджер и выше |
| `approve_tariffs` — Согласование тарифов | Финансовый директор + Админ | Согласование кастомных тарифов |

## Тип модели: RBAC + персональные пермиссии

Роль задаёт базовый набор пермиссий (хранится в БД в `lkm_role_permissions`). Кроме того пользователю можно назначить персональные пермиссии (`lkm_user_permissions`). Эффективный набор прав = **union** пермиссий роли и персональных пермиссий.

### Принципы хранения (важно для расширяемости)

Модель спроектирована так, чтобы **новые роли добавлялись без изменения кода**, а
новые пермиссии — с минимальной церемонией. Ключевые принципы:

1. **Пермиссии — enum-каталог в коде.** Канонический список всех пермиссий — это
   `Permission(StrEnum)`. При старте приложения значения enum идемпотентно
   «апсертятся» в таблицу `lkm_permissions` (insert отсутствующих). Таблица —
   зеркало кода, не источник истины. Отдельная миграция на каждую пермиссию не
   нужна. Новая пермиссия = добавить член enum + написать guard, который её
   проверяет (пермиссия без проверки в коде бессмысленна).

2. **Роли — данные, а не код.** В коде **нет** перечисления ролей и нет маппинга
   «роль → набор прав». Роль — это строка в `lkm_roles`, а её набор прав — строки
   в `lkm_role_permissions`. Добавить роль или изменить её набор прав можно правкой
   данных (сид/миграция сейчас; при необходимости — админ-API позже) **без деплоя**,
   потому что все точки проверки уже существуют.

3. **Guard'ы проверяют только пермиссии, никогда роль.** Это линчпин расширяемости.
   Даже «системные» роли не имеют спец-логики по имени:
   - управление пользователями закрыто пермиссиями `add_users` / `change_user_roles`
     / `change_permissions` (а не `role == 'admin'`);
   - функции менеджера закрыты `view_deals` / `create_deal` / … (а не `role == 'manager'`).

   ```python
   # ✅ роль можно добавить/переименовать — код не трогаем
   if Permission.APPROVE_KP_LEAD in user.effective_permissions: ...
   # ❌ каждая новая роль потребует правки кода
   if user.role in ("sales_lead", "director"): ...
   ```

4. **Единственная привязка к имени роли** — какая роль выдаётся при первом входе.
   Это codename-константа в конфиге (`DEFAULT_ROLE = "manager"`), без завязанного на
   неё поведения. Роли `admin` и `manager` — обычные строки в `lkm_roles`, которым
   просто засеян нужный набор существующих пермиссий, как и любой другой роли.

### Два вида кастомизации прав

| Вид | Таблица | Кого затрагивает | Пример |
|---|---|---|---|
| **Кастом набора роли** | `lkm_role_permissions` | всех пользователей роли | добавить всем «Пресейлам» право `approve_kp_lead` |
| **Персональные пермиссии** | `lkm_user_permissions` | одного пользователя | выдать конкретному пользователю `approve_tariffs` сверх роли |

Оба вида независимы и складываются: эффективные права = набор роли ∪ персональные пермиссии.

> **Правило для единоличных пермиссий.** Пермиссии с ограничением «только один
> пользователь» (`approve_kp_sales_director`, `approve_kp_finance_director`,
> `approve_kp_lawyer`) назначаются **только как персональные** (`lkm_user_permissions`)
> и **не включаются в наборы ролей** — иначе право автоматически получат все носители
> роли, что нарушает ограничение единоличности.

## Схема данных

```sql
-- Каталог всех пермиссий. Зеркало Permission(StrEnum) из кода:
-- идемпотентно синкается на старте приложения (insert отсутствующих codename).
CREATE TABLE lkm_permissions (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codename   TEXT UNIQUE NOT NULL,  -- например: 'view_deals'
    name       TEXT NOT NULL,         -- например: 'Просмотр всех сделок'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ
);

-- Роли — данные, а не enum в коде. Дефолтные роли засеваются (insert-if-absent),
-- дальше набор ролей редактируется как данные (сид/миграция; при необходимости — API).
CREATE TABLE lkm_roles (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codename   TEXT UNIQUE NOT NULL,  -- например: 'manager'
    name       TEXT NOT NULL,         -- например: 'Менеджер продаж'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ
);

-- Наборы пермиссий ролей. Редактируемые данные: изменение набора роли вступает
-- в силу сразу (guard'ы проверяют пермиссии, а не имя роли), без изменения кода.
CREATE TABLE lkm_role_permissions (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id       UUID NOT NULL REFERENCES lkm_roles(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES lkm_permissions(id) ON DELETE CASCADE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ,
    UNIQUE (role_id, permission_id)
);

-- Пользователи ЛКМ
CREATE TABLE lkm_users (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ad_login         TEXT UNIQUE NOT NULL,
    full_name        TEXT NOT NULL,
    role             TEXT NOT NULL DEFAULT 'manager',  -- codename из lkm_roles; дефолт = config DEFAULT_ROLE
    first_login_at   TIMESTAMPTZ NULL,  -- NULL = ни разу не логинился
    created_by_admin UUID REFERENCES lkm_users(id) ON DELETE SET NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ
);

-- Персональные пермиссии пользователя (назначаются администратором сверх ролевых)
CREATE TABLE lkm_user_permissions (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID NOT NULL REFERENCES lkm_users(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES lkm_permissions(id) ON DELETE CASCADE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ,
    UNIQUE (user_id, permission_id)
);
```

### Пермиссии «на одного пользователя»

Пермиссии `approve_kp_sales_director`, `approve_kp_finance_director` и
`approve_kp_lawyer` в каждый момент времени должны принадлежать не более чем одному
пользователю. Чтобы это ограничение было выполнимо, такие пермиссии **выдаются
только персонально** (`lkm_user_permissions`) и **не входят в наборы ролей**
(`lkm_role_permissions`) — иначе право получили бы все носители роли сразу.

Уникальность контролируется на уровне сервиса при назначении персональной пермиссии
(проверка, что она ещё ни за кем не закреплена). Так как эти пермиссии не приходят от
ролей, достаточно контроля по таблице `lkm_user_permissions`.

## Логика разрешения пермиссий

Эффективный набор прав пользователя вычисляется сервисом одним запросом через UNION:

```python
async def get_effective_permissions(self, user_id: UUID, role: str) -> set[str]:
    result = await self.session.execute(
        # пермиссии роли
        select(LkmPermissionModel.codename)
        .join(LkmRolePermissionModel, LkmRolePermissionModel.permission_id == LkmPermissionModel.id)
        .join(LkmRoleModel, LkmRoleModel.id == LkmRolePermissionModel.role_id)
        .where(LkmRoleModel.codename == role)
        .union(
            # персональные пермиссии пользователя
            select(LkmPermissionModel.codename)
            .join(LkmUserPermissionModel, LkmUserPermissionModel.permission_id == LkmPermissionModel.id)
            .where(LkmUserPermissionModel.user_id == user_id)
        )
    )
    return set(result.scalars().all())
```

Наборы пермиссий ролей хранятся в `lkm_role_permissions` (дефолтные засеваются
insert-if-absent при инициализации). Изменение набора роли — это правка данных в
`lkm_role_permissions` (сид/миграция сейчас; при необходимости — админ-API позже),
без изменения кода: эффективный набор пересчитывается тем же UNION-запросом. Список
самих пермиссий (`lkm_permissions`) — зеркало `Permission(StrEnum)`, синкается на
старте приложения.

## Многоступенчатое согласование КП

Согласование КП в БТ02 (Таблица 1.3) — это цепочка независимых стадий, каждая со
своей пермиссией:

1. `approve_kp_presale` — пресейл;
2. `approve_kp_lead` — руководитель по продажам;
3. `approve_kp_sales_director` — директор по продажам;
4. `approve_kp_finance_director` — финансовый директор;
5. `approve_kp_product_owner` — владелец продукта (техническая возможность услуг);
6. `approve_kp_lawyer` — юрист (нестандартные/юридические условия).

Более старшая роль наследует пермиссии согласования более младших стадий
(руководитель может согласовать и на стадии пресейла и т.д.), а администратор имеет
все стадии. Согласующие стадии директора по продажам, финансового директора и юриста
единоличны (см. сноску ¹). Точная последовательность/условия прохождения стадий
описываются в отдельном БТ по согласованию КП; здесь фиксируется только модель прав.

## Бизнес-правила

- При первом входе пользователь автоматически получает роль по умолчанию (codename из конфига `DEFAULT_ROLE`, сейчас `manager`).
- **Проверки доступа — только по пермиссиям, никогда по имени роли** (включая `admin` и `manager`). Управление пользователями закрыто пермиссиями `add_users` / `change_user_roles` / `change_permissions`.
- Пользователь с пермиссиями управления **не может** менять роль своей собственной УЗ (проверка на уровне сервиса).
- Пользователь с пермиссиями управления **не может** задавать кастомные пермиссии для своей собственной УЗ.
- Роль `admin` — это обычная роль, которой засеян полный набор пермиссий (в т.ч. все бизнес-пермиссии), а не спец-случай в коде.
- При смене роли все персональные пермиссии пользователя удаляются из `lkm_user_permissions` (новая роль = чистый старт с набором роли).
- Единоличные пермиссии согласования (директор по продажам / финансовый директор / юрист) выдаются только персонально и в один момент времени принадлежат не более чем одному пользователю.
- Ephemeral-пользователь: добавлен, роль не менялась (осталась дефолтной), ни разу не логинился → не отображается в списке и не сохраняется после обновления страницы.

```python
def is_visible_in_lkm(user: LkmUser) -> bool:
    return user.first_login_at is not None or user.role != settings.DEFAULT_ROLE
```

- Поиск пользователей для добавления идёт по AD ДФ (Фамилия/Имя/Отчество), минимум 3 символа, русские буквы + пробел/дефис/точка. Поиск повторяется при вводе каждого следующего символа.
- Уже добавленные пользователи подсвечиваются в результатах поиска, повторно добавить нельзя.
- Сортировка списка пользователей ЛКМ по полям «ФИО» и «Роль» (по возрастанию/убыванию по клику на заголовок).
- Аудит смен ролей рекомендуется через отдельную таблицу `user_role_history(user_id, old_role, new_role, changed_by, changed_at)`.

## Декомпозиция: Jira-тикеты

### Тикет 1 — [ЛКМ] Создать модель пользователя ЛКМ

**Цель**: завести таблицу `lkm_users` и SQLAlchemy-модель. Роль хранится как
codename-строка (FK/ссылка на `lkm_roles`), **enum ролей в коде не заводим** — роли
это данные (см. «Принципы хранения»).

**Что делать**:
- `app/models/lkm_user.py`:
  - `LkmUserModel(Base)`: поля `ad_login VARCHAR(255) UNIQUE NOT NULL`, `full_name VARCHAR(500) NOT NULL`, `role VARCHAR(50) NOT NULL DEFAULT '<DEFAULT_ROLE>'` (codename из `lkm_roles`), `first_login_at TIMESTAMP NULL`, `created_by_admin UUID FK → lkm_users(id) SET NULL`
  - **Не** объявлять `UserRole(StrEnum)`. Дефолтная роль берётся из конфига `DEFAULT_ROLE`.
- Добавить в конфиг константу `DEFAULT_ROLE` (codename роли для первого входа, сейчас `manager`).
- Миграция Alembic: создать таблицу `lkm_users`
- Зарегистрировать модель в `app/db/__init__.py`

**DoD**:
- Миграция применяется и откатывается без ошибок
- Модель импортируется без ошибок
- В коде нет перечисления ролей (grep по `UserRole` пуст)

---

### Тикет 2 — [ЛКМ] Создать справочник пермиссий, справочник ролей и связи (Django-style)

**Цель**: реализовать хранение пермиссий по аналогии с `auth_permission` в Django — каталог всех пермиссий (зеркало enum) + таблица ролей (данные) + M2M роль→пермиссия (редактируемые наборы) + M2M пользователь→пермиссия (персональные).

**Что делать**:
- `app/models/lkm_permission.py`:
  - `LkmPermissionModel(Base)`: `codename TEXT UNIQUE NOT NULL`, `name TEXT NOT NULL`
  - `LkmRoleModel(Base)`: `codename TEXT UNIQUE NOT NULL`, `name TEXT NOT NULL`
  - `LkmRolePermissionModel(Base)` (M2M): `role_id UUID FK → lkm_roles`, `permission_id UUID FK → lkm_permissions`, уникальная пара
  - `LkmUserPermissionModel(Base)` (M2M): `user_id UUID FK → lkm_users`, `permission_id UUID FK → lkm_permissions`, уникальная пара
- `Permission(StrEnum)` — **канонический каталог** пермиссий (codename = значение enum): `view_deals`, `edit_deal`, `change_deal_status`, `create_kp`, `send_kp`, `create_deal`, `import_deal`, `view_kp`, `approve_kp_presale`, `approve_kp_lead`, `approve_kp_sales_director`, `approve_kp_finance_director`, `approve_kp_product_owner`, `approve_kp_lawyer`, `validate_bz`, `chat_with_approvers`, `chat_with_managers`, `create_tariff`, `approve_tariffs`, `add_users`, `change_user_roles`, `change_permissions`. Роли enum'ом **не** объявляются.
- **Синк пермиссий на старте**: при инициализации приложения идемпотентно апсертить все значения `Permission` в `lkm_permissions` (insert отсутствующих codename). Отдельной миграции на каждую пермиссию не требуется.
- **Сид ролей**: дефолтные роли и их наборы (`lkm_roles`, `lkm_role_permissions`) засеваются insert-if-absent (не перезатирать существующие). Единоличные пермиссии (`approve_kp_sales_director` / `approve_kp_finance_director` / `approve_kp_lawyer`) **не включать** в наборы ролей — только персонально.
- Миграция Alembic: создать 4 таблицы (`lkm_permissions`, `lkm_roles`, `lkm_role_permissions`, `lkm_user_permissions`).

**Логика эффективных пермиссий**:
- Эффективные пермиссии = пермиссии роли (`lkm_role_permissions`) ∪ персональные пермиссии (`lkm_user_permissions`)
- Персональные пермиссии добавляются к ролевым, не заменяют их
- При смене роли персональные пермиссии пользователя удаляются (чистый старт)

**DoD**:
- После инициализации в `lkm_permissions` есть все codename из `Permission(StrEnum)`; повторный старт не создаёт дублей
- Дефолтные роли и их наборы засеяны; повторная инициализация не перезатирает изменённые вручную наборы
- Единоличные пермиссии отсутствуют в `lkm_role_permissions`
- Миграция применяется и откатывается без ошибок

---

### Тикет 3 — [ЛКМ] CRUD и сервис управления пользователями ЛКМ

**Цель**: реализовать операции с пользователями и бизнес-правила ролевой модели.

**Что делать**:
- `app/cruds/lkm_user.py`:
  - `get_by_id(user_id)`, `get_by_ad_login(ad_login)`
  - `list_visible()` — только пользователи, у которых `first_login_at IS NOT NULL` или `role != DEFAULT_ROLE`
  - `search_by_full_name(query)` — ILIKE по `full_name`, минимум 3 символа
  - `create(data)`, `update_role(user_id, new_role)`, `set_custom_permissions(user_id, permission_ids)`
- `app/services/lkm_user.py`:
  - `get_effective_permissions(user) -> frozenset[Permission]` — union набора роли и персональных пермиссий
  - `has_permission(user, perm) -> bool`
  - `is_visible_in_lkm(user) -> bool`
  - `change_role(admin_ad_login, target_user_id, new_role)`:
    - запрещает смену собственной роли (`admin_ad_login == target.ad_login`)
    - сбрасывает персональные пермиссии (удаляет записи из `lkm_user_permissions`)
    - проверка единоличных пермиссий не нужна: их нет в наборах ролей (только персонально)
  - `set_custom_permissions(admin_ad_login, target_user_id, permission_codenames)` — полная замена персонального набора; запрет для собственной УЗ; проверка единоличных пермиссий (нельзя выдать право второму пользователю)
- Ошибки в `app/helpers/errors.py`: `LkmUserCrudError`, `LkmUserNotFoundError`, `LkmForbiddenError`, `LkmSingleHolderPermissionError`

**DoD**:
- Правило "нельзя менять свою роль" проверяется на уровне сервиса
- Правило "нельзя задавать себе кастомные пермиссии" проверяется на уровне сервиса
- При смене роли кастомные пермиссии сбрасываются
- Единоличные пермиссии нельзя выдать второму пользователю

---

### Тикет 4 — [ЛКМ] Поиск пользователей в Keycloak по частичному совпадению логина или ФИО

**Цель**: расширить `KeycloakService` методом поиска пользователей через Keycloak Admin REST API, чтобы администратор мог найти нужного человека по неполному логину или части имени перед добавлением его в ЛКМ.

**Контекст**:
- Файл `app/services/keycloak.py` содержит `KeycloakService` с методом `get_all_users()` (полная загрузка всех пользователей постранично).
- Для поиска нужен другой подход: Keycloak Admin REST API поддерживает параметр `search`, который делает частичное совпадение (case-insensitive) одновременно по `username`, `firstName`, `lastName`, `email`.
- API-эндпоинт: `GET /admin/realms/{realm}/users?search={q}&max={max}&first=0`

**Что делать**:

**`app/services/keycloak.py`** — добавить метод:
```python
def search_users(self, query: str, max_results: int = 50) -> list[dict[str, Any]]:
    """Поиск по частичному совпадению username / firstName / lastName / email."""
```
- Принимает строку `query` (минимум 3 символа — валидация на вызывающей стороне)
- Отправляет `GET /admin/realms/{realm}/users?search={query}&max={max_results}`
- Возвращает список пользователей в формате Keycloak: `[{id, username, firstName, lastName, email, ...}]`
- Следует существующему паттерну сервиса (синхронный `httpx.Client`)

**`app/schemas/lkm_user.py`** — добавить схему ответа:
```python
class KeycloakUserSchema(BaseModel):
    id: str                     # keycloak UUID
    username: str               # AD-логин
    first_name: str | None
    last_name: str | None
    email: str | None
    already_added: bool = False # True если username уже есть в lkm_users
```

**`app/services/lkm_user.py`** — добавить метод сервиса:
```python
async def search_in_keycloak(self, query: str) -> list[KeycloakUserSchema]:
```
- Вызывает `keycloak_service.search_users(query)`
- Загружает из `lkm_users` все `ad_login` из результатов (batch-запрос)
- Проставляет `already_added=True` для тех, кто уже добавлен

**DoD**:
- `search_users("Иванов")` возвращает только совпадающих пользователей, не всю базу
- `already_added` корректно проставляется
- При `query` короче 3 символов сервис поднимает ошибку (валидация до вызова Keycloak)

---

### Тикет 5 — [ЛКМ] Перевести KeycloakService на полностью асинхронный httpx

**Цель**: убрать блокирующие синхронные HTTP-вызовы (`httpx.Client`) из `KeycloakService`, заменив их на `httpx.AsyncClient`, чтобы сервис не блокировал event loop FastAPI при каждом обращении к Keycloak.

**Контекст**: сейчас все методы `KeycloakService` (`get_all_users`, `search_users`, `create_user` и др.) используют синхронный `httpx.Client` и не являются корутинами. При вызове из async-контекста FastAPI они блокируют event loop на время HTTP-запроса. `search_in_keycloak` в `LkmUserService` вызывает `keycloak_service.search_users(query)` напрямую — это блокирующий вызов внутри `async def`.

**Что делать**:
- Заменить `httpx.Client` на `httpx.AsyncClient` во всех методах
- Сделать все публичные методы `async def` (`get_all_users`, `get_local_users`, `search_users`, `create_user`, `make_password_permanent` и др.)
- `_get_service_token` также сделать `async def` (использует `httpx.Client` для получения токена)
- Обновить все места вызова: `search_in_keycloak` в `LkmUserService` заменить `keycloak_service.search_users(query)` на `await keycloak_service.search_users(query)`
- Обновить существующие вызовы в других частях приложения (если есть)

**DoD**:
- Все методы `KeycloakService` — `async def`
- `mypy` и тесты проходят
- `search_in_keycloak` вызывает `await keycloak_service.search_users(query)`

---

### Тикет 6 — [ЛКМ] API-эндпоинты для управления пользователями

**Цель**: REST API для CRUD-операций с пользователями ЛКМ.

Guard'ы — по пермиссиям, не по роли (см. «Принципы хранения»).

**Эндпоинты** (`/api/v1/lkm/users`):
- `GET /` — список видимых пользователей (`list_visible`)
- `GET /search?q=<строка>` — поиск по ФИО для добавления нового пользователя
- `POST /` — добавить пользователя (пермиссия `add_users`)
- `GET /{user_id}` — карточка пользователя с эффективными пермиссиями
- `PATCH /{user_id}/role` — сменить роль (пермиссия `change_user_roles`, не свою УЗ)
- `PATCH /{user_id}/permissions` — задать персональные пермиссии (пермиссия `change_permissions`, не свою УЗ)

**Что делать**:
- `app/api/v1/lkm/__init__.py` + `app/api/v1/lkm/lkm_user_router.py`
- Подключить роутер в `app/api/v1/v1_router.py`
- Добавить тег `V1_LKM_USERS = 'API V1 / lkm / users'` в `ApiTags`

**DoD**:
- Все эндпоинты отображаются в Swagger
- Роутер подключён и отвечает на запросы

---

### Тикет 7 — [ЛКМ] Расширить `GET /api/v1/auth/user` данными роли и пермиссий

**Цель**: при аутентификации возвращать текущему пользователю его роль в ЛКМ и полный список доступных пермиссий.

**Что делать**:
- Добавить в ответную схему `/api/v1/auth/user`:
  ```json
  {
    "lkm_role": "manager | presale | sales_lead | sales_director | finance_director | product_owner | lawyer | admin",
    "lkm_permissions": ["view_deals", "create_deal", ...],
    "lkm_is_permissions_extended": false
  }
  ```
- В обработчике ручки: lookup по `ad_login` из JWT-токена → `LkmUserModel`
- Если пользователь не найден в `lkm_users`, автоматически создать его с ролью по умолчанию (`DEFAULT_ROLE`)
- При первом обращении (`first_login_at IS NULL`): проставить `first_login_at = now()` и вернуть effective permissions

**DoD**:
- Ответ содержит `lkm_role` и `lkm_permissions`
- Первый вызов `/api/v1/auth/user` создаёт пользователя с ролью по умолчанию (`DEFAULT_ROLE`), если его ещё нет в `lkm_users`
- `first_login_at` проставляется автоматически при первом вызове

---

### Тикет 8 — [ЛКМ] Аудит смен ролей

**Цель**: логировать каждое изменение роли пользователя в отдельную таблицу для последующего аудита.

**Контекст**: в разделе «Бизнес-правила» документа явно рекомендована таблица `user_role_history`.

**Что делать**:
- `app/models/lkm_user_role_history.py`:
  - `LkmUserRoleHistoryModel(Base)`: `user_id UUID FK → lkm_users`, `old_role VARCHAR`, `new_role VARCHAR`, `changed_by UUID FK → lkm_users` (администратор, совершивший смену), `changed_at TIMESTAMP`
- Миграция Alembic: создать таблицу `lkm_user_role_history`
- В `app/cruds/lkm_user.py` добавить `create_role_history_record(user_id, old_role, new_role, changed_by)`
- Вызывать из `LkmUserService.change_role(...)` сразу после успешного обновления роли

**DoD**:
- После каждой смены роли в `lkm_user_role_history` появляется новая запись
- Запись содержит корректные `old_role`, `new_role`, `changed_by`

---

### Тикет 9 — [ЛКМ] Очистка ephemeral-пользователей

**Цель**: автоматически удалять из `lkm_users` пользователей, добавленных администратором, но ни разу не вошедших в систему и не получивших смену роли.

**Контекст**: бизнес-правило из документа — «Ephemeral-пользователь: добавлен администратором, роль не менялась, ни разу не логинился → не отображается в списке **и не сохраняется после обновления страницы**».

**Критерий ephemeral**: `first_login_at IS NULL AND role = DEFAULT_ROLE` (роль осталась дефолтной)

**Что делать**:
- В `app/cruds/lkm_user.py` добавить метод `delete_ephemeral()` — удаляет все записи по критерию ephemeral
- В `app/services/lkm_user.py` добавить вызов `delete_ephemeral()` перед возвратом результата в методе `list_visible()` — очистка происходит при каждом открытии/обновлении списка пользователей

**DoD**:
- Пользователь, добавленный admin'ом, но не вошедший и с неизменённой ролью, исчезает из БД после следующего вызова `GET /lkm/users`
- Пользователь с изменённой ролью (`role != DEFAULT_ROLE`) не удаляется, даже если не логинился

---

### Тикет 10 — [ЛКМ] Эндпоинт справочника пермиссий

**Цель**: предоставить фронтенду список всех возможных пермиссий для построения UI выбора кастомных пермиссий (список чекбоксов при редактировании пользователя).

**Что делать**:
- Добавить эндпоинт в роутер из Тикета 6:
  - `GET /api/v1/lkm/permissions` — возвращает все записи из таблицы `lkm_permissions`
- Схема ответа:
  ```python
  class LkmPermissionSchema(BaseModel):
      id: UUID
      codename: str   # "view_deals"
      name: str       # "Просмотр всех сделок"
  ```
- Кэшировать ответ (данные статичны после seeding): `@lru_cache` на уровне сервиса или HTTP-заголовок `Cache-Control: max-age=3600`

**DoD**:
- `GET /api/v1/lkm/permissions` возвращает объекты с `codename` и `name` по всем пермиссиям
- Эндпоинт отображается в Swagger

---

### Тикет 11 — [ЛКМ] Проверка permissions в API-ручках

**Цель**: закрыть backend-ручки проверкой effective permissions пользователя.

**Правила**:
- `AuthMiddleware` продолжает отвечать за аутентификацию:
  - нет валидного токена → `401 Unauthorized`;
  - токен валиден → пользователь проходит дальше.
- `/api/v1/auth/user` — входная ручка ЛКМ:
  - создаёт пользователя с ролью по умолчанию (`DEFAULT_ROLE`), если его нет в `lkm_users`;
  - возвращает роль и effective permissions.
- Остальные защищённые ручки:
  - middleware по `ad_login` получает LKM-пользователя, роль и effective permissions;
  - если пользователя нет в `lkm_users`, вернуть `403 {"detail": "Доступ запрещён"}`;
  - если пользователь найден, положить данные в request context.
- `deal_actions` и `offer_actions` должны учитывать permissions пользователя:
  - если permissions не переданы в context, сохраняется старое поведение для API-ответов actions;
  - если permissions переданы, action resolver проверяет право и состояние объекта.
- Backend guard на ручках использует action resolver/helper:
  - если action недоступен, райзится `LkmForbiddenError`;
  - если есть человекочитаемое сообщение, оно отдаётся в `403`.
- `GET /api/v1/lkm/roles` и `GET /api/v1/lkm/permissions` для MVP закрыты пермиссиями управления пользователями (`change_permissions` / `change_user_roles`), а не проверкой роли.
  - TODO: при необходимости завести отдельные пермиссии `get_user_roles` и `get_permissions`.
- Внутренние/сервисные ручки для MVP закрываются пермиссиями управления пользователями.
  - TODO: заменить на отдельный permission или отдельный механизм сервисного доступа.

**DoD**:
- Неаутентифицированный пользователь получает `401`.
- Аутентифицированный пользователь без записи в `lkm_users` получает `403` на всех защищённых ручках, кроме `/auth/user`.
- `/auth/user` создаёт пользователя с ролью `DEFAULT_ROLE` при первом входе.
- Ручки сделок/офферов проверяют нужные permissions.
- Многоступенчатые пермиссии согласования КП проверяются по соответствующей стадии.
- LKM-админские справочники и управление пользователями закрыты пермиссиями (`add_users` / `change_user_roles` / `change_permissions`), а не проверкой имени роли.
- Нигде в бизнес-логике нет ветвления по имени роли (grep по сравнению `role ==` / `role in` пуст, кроме `DEFAULT_ROLE`).
- Есть тесты middleware и action resolver permissions.

---

## Открытые вопросы (уточнить у аналитика)

1. ~~Расхождение между Таблицей 1.3 и чек-листом 5.2~~ — **решено:** источник истины
   — Таблица 1.3 (многоступенчатое согласование КП), чек-лист 5.2 будет приведён
   аналитиком к 1.3. Ориентируемся только на 1.3.
2. **Тарифные пермиссии** (`create_tariff`, `approve_tariffs`) отсутствуют в Таблице 1.3,
   но фигурируют в описании роли «Финансовый директор» («согласование кастомных
   тарифов»). Нужно подтвердить, останутся ли они как отдельные пермиссии после
   приведения 5.2 к 1.3, и их точное распределение по ролям.
3. **Единоличные пермиссии**: для `approve_kp_sales_director`,
   `approve_kp_finance_director`, `approve_kp_lawyer` БТ02 требует «только один
   пользователь». Уточнить поведение при попытке выдать право второму пользователю
   (ошибка / автоматическое снятие с предыдущего).
4. ~~Поведение `custom_permissions` при смене роли~~ — **решено:** при смене роли
   `custom_permissions` всегда сбрасывается. Дефолтный набор новой роли вступает в
   силу автоматически. При необходимости администратор настраивает кастомные
   пермиссии заново.
