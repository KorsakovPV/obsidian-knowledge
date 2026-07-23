---
project: comet-backend
created: 2026-07-23
updated: 2026-07-23
source: docs/lkm_role_model.md
tags: [project, design, approval, lkm, rbac]
---

# Последовательное согласование КП

#project #design #approval #lkm #rbac

Связано: [[LKM Role Model]], [[BT02 Role Model Gaps]], [[Sequential KP Approval Tickets]].

## Контекст

БТ02 фиксирует не одну общую пермиссию согласования КП, а набор независимых стадий:

1. `approve_kp_presale` — пресейл;
2. `approve_kp_lead` — руководитель по продажам;
3. `approve_kp_sales_director` — директор по продажам;
4. `approve_kp_finance_director` — финансовый директор;
5. `approve_kp_product_owner` — владелец продукта;
6. `approve_kp_lawyer` — юрист.

Текущая реализация не моделирует последовательность стадий. `approval` — одна запись
на оффер, с одним общим `status`, `decision`, `email_token` и JSONB `approvers`.
`DealService.request_approval()` выбирает одного согласующего через
`max(approval.approvers, key=lambda approver: approver["level"])`, отправляет одно
письмо и переводит весь approval в `pending`.

Это плохо масштабируется на БТ02: невозможно явно понять текущую стадию, хранить
решение каждой стадии, пропустить этап, переназначить согласующего, отличить skip от
approve и безопасно обработать старые email-token'ы.

## Варианты реализации

### Вариант 1 — Расширить JSONB `approvers`

Хранить внутри `approvers` поля `stage`, `status`, `decision`, `decided_at`,
`skipped_at`, `skip_reason`, `token`.

Плюсы:

- минимум миграций;
- быстрее сделать MVP поверх текущего поля.

Минусы:

- сложно искать активные/pending стадии SQL-запросами;
- сложнее обеспечить уникальность token'ов и идемпотентность;
- нет нормальной ссылочной целостности на пользователя/permission;
- история skip/reassign/decision будет хрупкой;
- код быстро уйдёт в ручную работу с JSON.

Вывод: годится только как временный прототип, не как целевая модель.

### Вариант 2 — `approval` как процесс + `approval_stages`

`approval` остаётся агрегатом процесса согласования. Новая таблица
`approval_stages` хранит отдельные стадии, их порядок, required permission,
назначенного согласующего, token и статус стадии.

Плюсы:

- явная state machine;
- понятный audit trail на уровне каждой стадии;
- легко находить текущие задачи согласующего;
- email-token относится к конкретной стадии, а не ко всему процессу;
- skip/reassign/escalation становятся обычными доменными действиями;
- удобно тестировать переходы состояния.

Минусы:

- нужна миграция;
- нужно переписать отправку письма, приём ответа и сбор actions;
- нужно решить lifecycle старых approvals.

Вывод: рекомендуемый вариант.

### Вариант 3 — Event sourcing

Хранить события `stage_created`, `sent`, `approved`, `rejected`, `skipped`,
`reassigned`, а текущее состояние вычислять или материализовать отдельно.

Плюсы:

- лучший аудит;
- можно восстановить всю историю процесса.

Минусы:

- дороже реализация;
- нужен projector/current state;
- для текущей стадии проекта избыточно как первая итерация.

Вывод: можно добавить `approval_stage_events` позже, если простой audit в
`approval_stages` окажется недостаточным.

### Вариант 4 — Workflow/rules engine

Описывать маршрут согласования конфигом или DSL: стадии, условия включения, SLA,
fallback rules.

Плюсы:

- максимальная гибкость;
- бизнес-маршруты можно менять без кода.

Минусы:

- риск построить внутренний workflow-фреймворк;
- сложнее отладка и тестирование;
- пока нет evidence, что маршруты меняются настолько часто.

Вывод: не начинать с этого. Сначала сделать явную модель стадий и простой builder
маршрута.

## Рекомендуемая модель

Добавить таблицу `approval_stages`:

```sql
CREATE TABLE approval_stages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    approval_id UUID NOT NULL REFERENCES approval(id) ON DELETE CASCADE,
    stage_code TEXT NOT NULL,
    position INTEGER NOT NULL,
    required_permission TEXT NOT NULL,
    status TEXT NOT NULL,
    assigned_user_id UUID NULL REFERENCES lkm_users(id) ON DELETE SET NULL,
    assigned_email TEXT NULL,
    requested_at TIMESTAMPTZ NULL,
    due_at TIMESTAMPTZ NULL,
    decided_at TIMESTAMPTZ NULL,
    decision TEXT NULL,
    decision_comment TEXT NULL,
    email_token UUID UNIQUE NULL,
    email_token_expires_at TIMESTAMPTZ NULL,
    skipped_by_user_id UUID NULL REFERENCES lkm_users(id) ON DELETE SET NULL,
    skipped_at TIMESTAMPTZ NULL,
    skip_reason TEXT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NULL,
    UNIQUE (approval_id, position),
    UNIQUE (approval_id, stage_code)
);
```

Статусы стадии:

- `waiting` — стадия создана, но ещё не активна;
- `pending` — письмо/задача отправлены текущему согласующему;
- `approved` — стадия согласована;
- `rejected` — стадия отклонена;
- `skipped` — стадия пропущена уполномоченным пользователем;
- `canceled` — стадия отменена из-за отмены всего процесса или пересоздания КП.

Статус `approval` остаётся агрегатным:

- `draft` — маршрут подготовлен, но ещё не запущен;
- `pending` — есть активная стадия `pending` или следующая `waiting`;
- `answered` + `decision=approve` — все обязательные стадии `approved` или `skipped`;
- `answered` + `decision=reject` — любая стадия `rejected`;
- `canceled` — процесс отменён.

## State machine

Запуск:

1. При создании/обновлении оффера builder строит маршрут стадий.
2. Маршрут зависит от причины согласования: скидка, техническая возможность,
   нестандартные условия, тарифы и т.д.
3. Не все 6 стадий обязательны для каждого КП.
4. Первая стадия получает `pending`, token, `requested_at`, `due_at`; остальные — `waiting`.

Approve:

1. Ответ по email или API ищет stage по `email_token`.
2. Если stage не `pending`, ответ идемпотентно игнорируется или возвращает понятную ошибку.
3. Stage переводится в `approved`.
4. Если есть следующая `waiting` stage — она становится `pending`, генерируется новый token,
   отправляется письмо/уведомление.
5. Если стадий больше нет — approval закрывается как approved.

Reject:

1. Текущая stage переводится в `rejected`.
2. Approval закрывается как rejected.
3. Оставшиеся `waiting` stages переводятся в `canceled`.

Skip:

1. Skip — отдельное доменное действие, не approve.
2. Skip разрешён только пользователю с отдельной permission, например
   `skip_kp_approval_stage`, либо явно согласованной администраторской permission.
3. Skip требует `skip_reason`.
4. Stage переводится в `skipped`, сохраняются `skipped_by_user_id`, `skipped_at`,
   `skip_reason`.
5. Процесс активирует следующую стадию.

Reassign:

1. Reassign предпочтительнее skip, если согласующий отсутствует.
2. Новый согласующий должен иметь required permission стадии.
3. Старый token инвалидируется, новый token генерируется для этой же stage.
4. В истории stage фиксируются `reassigned_by`, old/new assignee и reason, если будет
   отдельная таблица событий.

Escalation:

1. Для просроченных стадий лучше сначала эскалировать, а не автоматически skip'ать.
2. Авто-skip допустим только для явно optional стадий.
3. Mandatory стадии требуют ручной skip/reassign с reason.

## Правила пропуска этапов

Пропуск нужен для кейса, когда согласующий долго отсутствует. Но skip меняет смысл
процесса: это не бизнес-согласование. Поэтому:

- skip всегда аудируется;
- skip не удаляет stage;
- skip не должен выглядеть как `approved` в истории;
- причина обязательна;
- UI должен показывать, что стадия была пропущена, кем и почему;
- downstream-интеграции должны получать различимые состояния `approved` и `skipped`.

Рекомендуемый порядок реакции на отсутствие согласующего:

1. `reassign` другому пользователю с той же permission;
2. `escalate` по согласованному правилу;
3. `skip` как ручной override с reason.

## Подводные камни

- Старое письмо не должно закрывать новую стадию: token должен принадлежать stage.
- Нужна транзакционная блокировка активной stage при approve/reject/skip/reassign.
- Нужна идемпотентность повторных email-ответов.
- Нужно решить, может ли один пользователь согласовать несколько последовательных стадий,
  если у него несколько permissions.
- Нужно решить, что делать, если permission/роль согласующего изменилась после отправки
  stage: сохранять исходного assignee или валидировать заново перед decision.
- Нужно решить, кто видит список pending задач: по assigned user, по permission или оба режима.
- Единоличные permissions БТ02 должны приводить к конкретному assignee; без assignee нельзя
  отправить stage.
- `deal.approval` сейчас возвращает первое согласование из списка, это нужно заменить
  выбором актуального approval-процесса.
- При изменении оффера во время `pending` нужно отменять текущие стадии и пересобирать маршрут.
- Email и API decisions должны использовать одну доменную state machine, а не две разные ветки.

## Открытые вопросы

1. Какие стадии обязательны для какого типа КП: скидка, тарифы, техническая возможность,
   нестандартные юридические условия?
2. Кто имеет право skip/reassign/escalate?
3. Какой SLA у стадии и что происходит при просрочке?
4. Может ли старшая роль согласовать младшую стадию вместо назначенного пользователя?
5. Если один пользователь обладает несколькими permissions, нужно ли авто-approve следующих
   стадий или каждую стадию подтверждать отдельно?
6. Нужно ли хранить полную event history отдельной таблицей `approval_stage_events`?
7. Как показывать skipped stages в PDF/письмах/интеграции с Order Processing?
