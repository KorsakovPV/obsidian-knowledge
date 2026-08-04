---
project: comet-backend
created: 2026-07-28
updated: 2026-08-04
source: docs/approval_cutover_runbook.md
status: historical
tags: [project, research, approval, runbook, migration, historical]
---

# Approval Cutover Runbook

#research #approval #runbook #historical

Связано: [[Sequential KP Approval]], [[Sequential KP Approval Tickets]], [[Architecture]],
[[Domain Model]].


> **Статус: архивный документ.** Cutover выполнен, прод работает на staged workflow.
> Ticket 15 удалил и инструменты, и схему, которые здесь описаны: команды
> `python -m app.cli.approval_cutover ...`, таблицы `approval_maintenance_state` и
> `approval_cutover_deals`, флаг `APP_APPROVAL_WORKFLOW_V2_ENABLED` и legacy-колонки
> `approval` больше не существуют (миграция `c9d0e1f2a3b5`). Ни одну команду из этого
> документа выполнить нельзя — он сохранён только как запись о том, как переводили
> данные, и как источник SQL-проверок для разбора старых сделок.

Документ описывал операционный порядок перевода legacy approvals на deal-level
staged workflow. Все команды выполнялись внутри пода приложения: директория
`scripts/` в образ не попадает, поэтому команда жила в `app/cli`.

## Модель развёртывания

Используется expand / migrate / contract:

1. **Expand** — миграция `d4e5f6a7b8ca` добавляет только новые таблицы
   (`approval_maintenance_state`, `approval_cutover_deals`). Существующие колонки
   `approval` не получают `NOT NULL` и новых уникальных ограничений, поэтому
   предыдущий релиз работает с этой схемой без изменений.
2. **Migrate** — preflight, dry-run и сама миграция данных (этот документ).
3. **Contract** — физическое удаление legacy-колонок (`approvers`,
   approval-level `email_token`, обязательный `offer_id`) выполняется отдельной
   cleanup migration в Ticket 15, после периода наблюдения.

## Защита от гонок

Записи согласования на время cutover закрываются двумя независимыми механизмами,
которые проверяются **внутри write-транзакции**:

- persisted flag `approval_maintenance_state.approval_cutover` — переживает
  restart приложения и виден всем worker-ам;
- PostgreSQL advisory lock `21642164` — cutover держит его в exclusive-режиме на
  отдельном соединении, а каждая write-транзакция берёт его в shared-режиме
  через `pg_try_advisory_xact_lock_shared`.

Проверка встроена в:

- `POST /api/v1/deals/{deal_id}/approval/request` и любой rebuild маршрута
  (`ApprovalLifecycleService`);
- изменение сделки и офферов (`ApprovalSubjectGuard`): запрос отклоняется до записи,
  иначе deal/offer закоммитились бы без пересборки маршрута;
- API стадий: decision, skip, reassign;
- обработка входящего решения по stage token (`ApprovalEmailDecisionService.process`);
- outbox worker доставки писем (`ApprovalNotificationWorker.process_pending`);
- IMAP consumer (`DealService.receive_mail`).

Одного HTTP-флага недостаточно, поэтому останавливать ручку без включения
persisted flag нельзя.

## Порядок выполнения

### 1. Backup

```bash
pg_dump --format=custom --file=comet_before_cutover.dump "$DB_DSN"
```

Снимок обязателен: после появления deal-level строк откат старого релиза
возможен только через восстановление БД (см. раздел «Rollback»).

### 2. Запрет новых отправок

```bash
python -m app.cli.approval_cutover maintenance-on \
  --reason "DFDEV-2164 cutover" --actor <логин>
python -m app.cli.approval_cutover status
```

После включения флага новые запросы согласования, решения по стадиям, доставка
писем и IMAP-приём отвечают ошибкой `approval_maintenance` (HTTP 503).

### 3. Preflight

```bash
python -m app.cli.approval_cutover preflight
```

Команда печатает `approval_id` и `deal_id` всех согласований в статусах
`pending` и `blocked` и возвращает код `1`, если такие записи есть. Их нужно
дождаться или закрыть операционно. На `revoke_approval()` полагаться нельзя:
метод остаётся заглушкой.

### 4. Dry-run reconciliation

```bash
python -m app.cli.approval_cutover dry-run > cutover-dry-run.json
```

Для каждой сделки отчёт содержит legacy id и статусы, выбранную новую версию,
максимальную скидку и её источник, офферы с особыми условиями, новый маршрут и
`subject_hash`. Отчёт прикладывается к задаче cutover.

Сделки со `state = manual_review_required` в cutover не попадают: миграция не
угадывает legacy trigger (например, `product_rule`, который не восстанавливается
из актуальных данных). Их разбирают вручную и мигрируют точечно через
`--deal-id`.

### 5. Миграция данных

```bash
python -m app.cli.approval_cutover migrate > cutover-migrate.json
```

Что делает команда:

- группирует legacy `draft` версии по сделке и пересобирает единый deal-level
  маршрут по актуальным данным всех офферов;
- при пустом маршруте создаёт current v2 версию `not_required` и увеличивает
  счётчик `approval_not_required`;
- помечает legacy `answered` / `canceled` версии `workflow_version=1`,
  `is_current=false`, `superseded_at`, сохраняя `offer_id`; строки физически не
  удаляются;
- пишет журнал `approval_cutover_deals`, благодаря которому повторный запуск
  идемпотентен и не создаёт дубли.

### 6. Проверки количества

```sql
-- Активных согласований не осталось
SELECT count(*) FROM approval WHERE status IN ('pending', 'blocked');

-- Ровно одна current версия на сделку
SELECT deal_id, count(*) FROM approval WHERE is_current GROUP BY deal_id HAVING count(*) > 1;

-- Legacy строки сохранены и помечены
SELECT status, count(*) FROM approval WHERE workflow_version = 1 GROUP BY status;

-- Итоги cutover
SELECT state, count(*) FROM approval_cutover_deals GROUP BY state;
```

Числа должны совпадать со счётчиками `migrated`, `approval_not_required`,
`manual_review_required` из отчёта команды.

### 7. Deploy и включение записей

1. Выкатить релиз с `APP_APPROVAL_WORKFLOW_V2_ENABLED=true` и заданным
   `APPROVAL_WORKFLOW_OUTBOX_ENCRYPTION_KEY`.
2. Выключить режим обслуживания:

   ```bash
   python -m app.cli.approval_cutover maintenance-off --actor <логин>
   python -m app.cli.approval_cutover status
   ```

3. Проверить, что outbox worker разбирает очередь, а `GET
   /api/v1/approval-stages/pending` отвечает.

После включения v2 approval-level `email_token` больше не принимается: такие
письма помечаются прочитанными и логируются как legacy.

## Rollback (актуально было на момент cutover)

- **До включения v2 write traffic** (флаг выключен, миграция данных не
  запускалась) — обычный откат релиза на offer-level версию.
- **После появления deal-level строк** откат старого кода невозможен: он не умеет
  nullable `offer_id` и несколько версий согласования у сделки. Допустимы только
  два пути:
  1. forward-compatible релиз v2 с выключенным `approval_workflow_v2_enabled`;
  2. восстановление снимка БД из шага 1 (теряются изменения, сделанные после
     cutover).

Аварийная остановка без отката релиза: `maintenance-on` — записи закрываются, а
уже принятые решения сохраняются.

## Что изменилось после Ticket 15

- Флага `approval_workflow_v2_enabled` больше нет: staged workflow — единственный
  процесс, откат на offer-level версию невозможен ни при каком значении настроек.
- `approval.offer_id`, `approval.approvers`, approval-level `email_token` и
  `email_token_expires_at`, а также `approval_stages.email_token` удалены из схемы
  миграцией `c9d0e1f2a3b5`.
- Сводка сделки строится только по current версии. Legacy-строки остались в таблице
  `approval` и видны в истории (`GET /deals/{deal_id}/approvals`), но сделку больше
  не представляют: сделка без current версии считается `not_required`.
- Инструменты cutover (`app/cli/approval_cutover`, `ApprovalCutoverService`,
  `ApprovalMaintenanceService`) и обе служебные таблицы удалены.
