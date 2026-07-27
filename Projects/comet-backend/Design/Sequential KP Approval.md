---
project: comet-backend
created: 2026-07-23
updated: 2026-07-27
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

В текущем коде добавлены `approval_stages`, offer-level builder, selector assignee и
активация первой waiting stage из `DealService.request_approval()`. Исходящее письмо
всё ещё содержит legacy `approval.email_token`, а приём ответа изменяет approval
напрямую. Кроме того, `OfferService` создаёт отдельный approval каждому offer, а
`DealModel.approval` возвращает первый процесс из списка.

Оставшийся разрыв — переход к единому deal-level approval, stage-level email token
и транзакционной state machine.

Tickets 1–5 считаются завершёнными промежуточными этапами и не переоткрываются
после уточнения границы агрегата. Переход к deal-level модели, адаптация builder-а,
единой юридической стадии и activation orchestration выполняются в Ticket 6 новыми
миграциями и forward-изменениями. Stage token и outbox относятся к Ticket 7,
остальные части перехода распределены по Tickets 8–15.

Принятые решения для реализации:

- approval-процесс принадлежит сделке; все offers рассматриваются совместно в одном
  маршруте и одном письме активной стадии;
- скидочный маршрут определяется максимальной скидкой среди всех тарифов всех offers;
- особые условия любого offer добавляют обязательную цепочку
  `product_owner → lawyer`;
- единоличная стадия назначается одному персональному holder-у permission;
- групповая стадия отправляется всем holders effective permission, первый ответ
  закрывает stage;
- один пользователь отдельно подтверждает каждую последовательную stage, даже если
  имеет permissions нескольких стадий;
- внутри approval одновременно может быть только одна `pending` stage;
- все переходы фиксируются в `approval_stage_events`;
- v2 workflow включается feature flag только после готовности audit, stage-token
  processing и outbox delivery;
- cutover выполняется только при отсутствии legacy approvals в `pending`.

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

Вывод: полное event sourcing для процесса не требуется, но для целевого workflow
используется отдельный append-only audit `approval_stage_events`.

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

### Deal-level approval и версии

`approval` — версия процесса согласования сделки, а не одного offer. У сделки может
быть история процессов, но не более одного текущего:

- `deal_id` — владелец агрегата;
- `version` — монотонный номер версии внутри сделки;
- `is_current`, `superseded_at` — выбор текущей версии без эвристики «первый в списке»;
- `workflow_version` — разделение legacy и staged records;
- `subject_snapshot`, `subject_hash` — неизменяемый после start предмет решения;
- `route_context` — причины маршрута, max discount/source, offers с особыми
  условиями, версии builder и stage order.

Ограничения: `UNIQUE(deal_id, version)` и partial unique index по
`deal_id WHERE is_current`. Обычный `UNIQUE(deal_id)` недопустим, поскольку уничтожает
возможность повторного согласования с сохранением истории.

Новые staged records не принадлежат одному offer. Nullable `offer_id` сохраняется
только на период чтения legacy history. `DealModel.approvals` остаётся ordered
history, а `current_approval` определяется отдельной relationship/query.

Snapshot фиксирует все значения, показанные согласующему и влияющие на маршрут:
offers, продукты, тарифы/цены/скидки, технические параметры, особые условия и
deal-level поля. Email, API detail и создание заказов после approve используют этот
snapshot, а не изменяемые live rows.

Таблица `approval_stages` добавлена миграцией `2026_07_23_1200-a7b8c9d0e1f2_add_approval_stages.py`:

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

- `not_required` — route policy рассчитана для snapshot, обязательных стадий нет;
- `draft` — маршрут подготовлен, но ещё не запущен;
- `pending` — есть активная стадия `pending` или следующая `waiting`;
- `blocked` — предыдущее решение сохранено, но следующую stage нельзя активировать
  из-за отсутствующего assignee;
- `answered` + `decision=approve` — все обязательные стадии `approved` или `skipped`;
- `answered` + `decision=reject` — любая стадия `rejected`;
- `canceled` — процесс отменён.

### Token history и transactional outbox

Bearer token нельзя удалять после первого ответа, иначе повторный ответ невозможно
отличить от неизвестного token и идемпотентно вернуть `already_processed`.

Целевая таблица `approval_stage_tokens` хранит hash token, stage, issued/expires,
used и invalidated timestamps. Reassign инвалидирует старую запись и создаёт новую.

SMTP не вызывается внутри workflow-транзакции. В той же транзакции, что переводит
stage, создаются одна логическая `approval_stage_notification` и N
`approval_stage_notification_deliveries`. Notification временно хранит raw token
только в зашифрованном виде; после завершения доставок secret редактируется. Outbox
worker после commit отправляет deliveries с retry/backoff и
`FOR UPDATE SKIP LOCKED`. Для group permission stage/token/message общие, recipients
и статусы доставки раздельные.

## State machine

Целевой запуск:

1. При создании/обновлении состава сделки builder строит единый маршрут стадий.
2. Маршрут зависит от максимальной скидки среди всех offers сделки, технической
   возможности, особых условий и других deal-level правил.
3. Не все 6 стадий обязательны для каждого КП.
4. Первая стадия получает `pending`, token, `requested_at`, `due_at`; остальные — `waiting`.
5. Переход и notification outbox фиксируются одной транзакцией; письмо отправляется
   после commit.

### Особые условия и юридическая стадия

Особые условия — отдельное поле оффера `special_conditions`, а не значение,
выведенное из скидки. Непустое после `strip()` значение хотя бы у одного offer
добавляет в единый маршрут сделки одну обязательную стадию `lawyer` с пермиссией
`approve_kp_lawyer`. Пустые значения и `null` стадию не создают.

Состав стадий и их порядок определяются раздельно. Начальная конфигурация порядка:
`presale → lead → product_owner → lawyer → sales_director → finance_director`.
Так директора получают документ после юридической визы. Позицию `lawyer` можно
изменить в `STAGE_ORDER` без изменения правил включения; созданный маршрут фиксирует
порядок в `approval_stages.position`.

### Граница deal-level approval

- Одна сделка имеет один актуальный approval.
- Builder анализирует все актуальные offers и выбирает скидочную цепочку по
  максимальной скидке среди всех цен.
- На активной стадии формируется одно письмо со сделкой и всеми offers. Для group
  permission оно доставляется каждому holder-у, но stage и token остаются общими.
- Согласующий принимает решение по сделке целиком и видит прибыльные и
  низкомаржинальные offers вместе.
- Изменение любого offer пересобирает draft-маршрут; изменение при pending требует
  отмены текущего процесса и явного rebuild.

Approve:

1. Ответ по email или API ищет stage по `email_token`.
2. Если stage не `pending`, ответ идемпотентно игнорируется или возвращает понятную ошибку.
3. Stage переводится в `approved`.
4. Если есть следующая `waiting` stage — она становится `pending`, генерируется новый token,
   отправляется письмо/уведомление.
5. Если стадий больше нет — approval закрывается как approved.
6. Если решение сохранено, но у следующей stage нет assignee, approval получает
   `blocked`, следующая stage остаётся waiting; retry activation продолжает процесс
   после исправления permissions без отмены принятого решения.

Reject:

1. Текущая stage переводится в `rejected`.
2. Approval закрывается как rejected.
3. Оставшиеся `waiting` stages переводятся в `canceled`.

Skip:

1. Skip — отдельное доменное действие, не approve.
2. Skip разрешён только пользователю с permission `skip_kp_approval_stage`.
3. Skip требует `skip_reason`.
4. Stage переводится в `skipped`, сохраняются `skipped_by_user_id`, `skipped_at`,
   `skip_reason`.
5. Процесс активирует следующую стадию.

Reassign:

1. Reassign предпочтительнее skip, если согласующий отсутствует.
2. Новый согласующий должен иметь required permission стадии.
3. Старый token инвалидируется, новый token генерируется для этой же stage.
4. В `approval_stage_events` фиксируются actor, old/new assignee и reason.

Для целевого workflow используется `approval_stage_events`; изменение stage и запись
event выполняются в одной транзакции.

Escalation и автоматический skip не входят в первую итерацию. Просроченная стадия
остаётся `pending`, пока не будет выполнен approve/reject/reassign либо ручной skip
с обязательным reason.

## Правила пропуска этапов

Пропуск нужен для кейса, когда согласующий долго отсутствует. Но skip меняет смысл
процесса: это не бизнес-согласование. Поэтому:

- skip всегда аудируется;
- skip не удаляет stage;
- skip не должен выглядеть как `approved` в истории;
- причина обязательна;
- UI должен показывать, что стадия была пропущена, кем и почему;
- downstream-интеграции должны получать различимые состояния `approved` и `skipped`.

Порядок реакции на отсутствие согласующего:

1. `reassign` другому пользователю с той же permission;
2. `skip` как ручной override с reason.

## Подводные камни

- Старое письмо не должно закрывать новую стадию: token должен принадлежать stage.
- Нужна транзакционная блокировка активной stage при approve/reject/skip/reassign.
- Нужна идемпотентность повторных email-ответов.
- Один пользователь отдельно подтверждает каждую последовательную stage, даже если
  обладает несколькими permissions.
- Для единоличной stage сохраняется назначенный assignee; для групповой доступ и
  отправитель повторно проверяются по effective permission.
- Единоличные pending задачи видит assigned user, групповые — пользователи с
  effective permission стадии.
- Единоличные permissions БТ02 должны приводить к конкретному assignee; без assignee нельзя
  отправить stage.
- `deal.approval` сейчас возвращает первое согласование из списка; модель нужно
  мигрировать к единственному актуальному deal-level approval.
- При изменении любого оффера во время `pending` нужно отменять текущие стадии и
  пересобирать общий маршрут сделки.
- Draft rebuild сериализуется блокировкой deal. После start snapshot неизменяем;
  новый цикл создаёт следующую approval version и supersede предыдущей.
- Все workflow CRUD принимают одну `AsyncSession`; вложенные автономные sessions
  нарушат атомарность state/event/token/outbox. Lock order:
  deal → current approval → active stage.
- Skip разрешён только optional stage; mandatory `lawyer` не пропускается.
- Reassign имеет смысл только для single-holder stage. Group membership определяется
  effective permission.
- Email и API decisions должны использовать одну доменную state machine, а не две разные ветки.

## Интеграция с Order Processing

Одно одобрение сделки должно породить отдельный заказ для каждого offer. Текущий
контракт использует `approval.id` как единственный `external_approval_id`, поэтому
после deal-level перехода он конфликтует для второго заказа.

Предпочтительный контракт — составной `(approval_id, offer_id)`. Если ОП принимает
только UUID, Comet вычисляет стабильный UUIDv5 от пары approval/offer. Payload каждого
заказа строится из approved snapshot. Частичные ошибки повторяются per offer и не
создают заново уже успешные заказы.

## Открытые вопросы

1. Какие стадии обязательны для какого типа КП: скидка, тарифы, техническая возможность,
   нестандартные юридические условия?
2. Какой SLA у стадии и что происходит при просрочке?
3. Как показывать skipped stages в PDF/письмах/интеграции с Order Processing?
