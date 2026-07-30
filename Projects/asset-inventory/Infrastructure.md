---
project: asset-inventory
created: 2026-07-30
updated: 2026-07-30
tags: [project, infrastructure, kubernetes, postgresql, pgpool]
---

# Инфраструктура и эксплуатация

Состояние на 2026-07-30, снято с живых стендов. Точка входа — [[Overview]],
устройство сервиса — [[Architecture]].

## Стенды

| | прод (`ns asset-inventory`) | тест (`ns asset-inventory-test`) |
|---|---|---|
| Ноды Postgres | **2** (`postgresql-0`, `-1`) | **1** (`postgresql-0`) |
| pgpool | 1 под | 1 под |
| Поды приложения | 3 | 4 (есть ещё `api`) |
| Redis | 1 | 1 |

Деплои приложения поднимаются из одного образа разными `args`: основной
(`app/main.py`), `asset-inventory-worker`
(`app.tasks.scheduled.uncertainty_report_worker`) и `allocation-recovery-worker`
(`app.tasks.allocation_recovery_worker`). Манифесты — `k8s/manifests/`, конфигурация —
`k8s/configmaps/`. Пода `asset-inventory-api` с тестового стенда в репозитории нет.

**Primary на проде — `postgresql-1`, а не `-0`.** Команды вида
`kubectl exec ...postgresql-0` попадают на standby.

## pgpool и Postgres

Конфигурация **идентична на обоих стендах** и дефолтам чарта bitnami **не
соответствует** — devops донастраивал и базу, и pgpool. Не рассуждать по дефолтам.

pgpool-II 4.6: `num_init_children = 64`, `max_pool = 4`, `reserved_connections = 1`,
`child_life_time = 300`, `connection_life_time = 0`, `client_idle_limit = 1800`,
`connection_cache = on`, `load_balance_mode = on`,
`disable_load_balance_on_write = 'transaction'`, `statement_level_load_balance = 'off'`,
оба function-списка пустые.

Postgres: `max_connections = 400`, `superuser_reserved_connections = 3`,
`synchronous_standby_names` пустое (**репликация асинхронная**),
`synchronous_commit = on`.

### Бюджет соединений

Считается по **всем деплоям сразу**, а не по репликам одного: `engine` в
`app/db/db.py` создаётся на уровне модуля, поэтому у каждого пода свой пул.

```
клиентская сторона:  Σ_деплоев (replicas × (pool_size + max_overflow)) ≤ num_init_children − reserved
backend-сторона:     num_init_children × max_pool                      ≤ max_connections − reserved
```

Backend-сторона сходится с запасом (`64 × 4 = 256` против `400 − 3 = 397`).
Узкое место — клиентская: доступно 63 соединения на все поды вместе.

Расчёт продублирован комментарием в `app/core/config.py` над `pool_size`.

### Ловушки pgpool

- `client_idle_limit = 1800` рвёт клиента после 30 минут простоя **даже внутри
  открытой транзакции**. Отсюда `pool_recycle = 1500` в настройках движка.
- В pgpool 4.2+ `black_function_list` переименован в `write_function_list` /
  `read_only_function_list`. Оба пустые, а в этом случае pgpool смотрит на
  `provolatile` и volatile-функцию считает пишущей → `pg_advisory_xact_lock`
  (объявлен VOLATILE) уходит на primary. Плюс `disable_load_balance_on_write =
  'transaction'`. То есть балансировка advisory-локи не ломает.
- `PGPOOL SHOW` не работает из JDBC-клиентов (DataGrip): свои админ-команды pgpool
  распознаёт только в simple query mode. Читать `pgpool.conf` проще.

### История отказов по соединениям

Конфиг с тех пор менялся, поэтому текущими числами эти инциденты объяснять нельзя:

- pgpool на тесте — 819 `Sorry, too many clients already` (янв 2026: 720, фев: 99);
- Postgres на проде — 1054 `FATAL: sorry, too many clients already`, все в январе,
  пик 2026-01-21 15:49;
- `QueuePool` timeout в приложении — 16276 событий, только на тесте, недели
  2026-02-02 и 2026-02-09.

С февраля 2026 — ничего. Примечательно, что прод-приложение в тот период не
залогировало ни `QueuePool`, ни `OperationalError`: либо отказы получал не сервис,
либо исключения проглатывались (см. `AI-SCALE-P0-5` в [[Scaling-roadmap-tasks]]).

### История failover

`received promote request` на проде — 21 событие с января по май 2026 (9 на `-0`,
12 на `-1`), из них 10 за пять дней 24–28 мая: флаппинг двухнодового repmgr без
кворума. Оба пода Postgres созданы ~29 мая, сразу после последнего переключения,
и с тех пор переключений не было. На тесте promote-событий нет.

Вывод по pgpool: **с прода не убирать** — это единственный источник failover для двух
нод, и он реально срабатывал. Проблемы решаются настройкой, а не удалением.

## Логи

Логи обоих стендов лежат в Elasticsearch (`https://elastic-dev.datafort.ru`,
Kibana — `kibana-dev.datafort.ru`), индекс `logs-*`, ретенция минимум с 2025-09-22.

Имена полей нестандартные: `namespace`, `container`, `log_message` (текст сообщения
приложения), `json_data.module` / `.function` / `.line` / `.git_tag`, `level`.
У логов Postgres и pgpool полей `namespace`/`container` **нет** — фильтровать только
по `log.file.path`. Осторожно: имя пода pgpool содержит подстроку `postgresql`,
поэтому паттерн `*postgresql*_asset-inventory_*` захватывает и его; правильно —
`*-pgpool-*` и `*-ha-postgresql-*`.

⚠️ Запрос вида `{"size": N}` по широкому окну возвращает не выборку, а произвольный
непрерывный кусок. Для перцентилей обязательны `function_score` + `random_score` по
`_seq_no`, иначе получаются ложные картины. Также нужен `"track_total_hits": true` —
иначе счётчик режется на 10000.

## Производительность

Замер 2026-07-30, случайная выборка, n=7200 на стенд, окно 36 часов, поле `total=`
из строки `<service> timing:`.

| стенд | med | p90 | p99 | max |
|---|---|---|---|---|
| прод | **123 ms** | 336 ms | 580 ms | 866 ms |
| тест | **53 ms** | 156 ms | 203 ms | 310 ms |

Разложение по фазам (окно 6 часов): `order_lookup` 23 против 11 мс,
`sync_allocations` внешний 113 против 31 мс, а внутренний
`sync_allocations breakdown total` — **24 против 17 мс**. Session acquire — 0 мс на
медиане и p99 на обоих стендах.

Главное: сама транзакция записи под advisory-локом почти не отличается, а пул не
контендится. Значит разрыв не про пул, не про семафоры и не про pgpool.

Проверены и отброшены: pgpool; отсутствие индекса под `latest_recovered_through`
(индекс `ix_allocation_recovery_run_completed` есть с 2026-06-23, `EXPLAIN` даёт
0.086 мс); синхронная репликация (её нет); фоновые задачи; версия «тест быстрее,
потому что ничего не делает» (объёмы сопоставимы).

Осталось два кандидата: **два COMMIT'а на сообщение** (`synchronous_commit = on`,
fsync на локальный диск, на проде объём WAL больше) и **общий эффект объёма данных**
(`slow_sqlalchemy_query` за 36 ч: 220 на проде против 3 на тесте, из них 98 по
`allocate_resource` и 67 по `order_resource`). Ради этого в код добавлены
`recovery_phase breakdown` и `write_phase breakdown` — они меряют в том числе время
COMMIT.

## Прочее

- `APP_LOG_LEVEL = DEBUG` на **обоих** стендах: ~1.5 млн строк за 6 часов на проде
  (~70 строк/с). Переводить прод в INFO имеет смысл, но только после снятия замеров
  по новым breakdown-строкам.
- `DEBUG: "True"` в проде против `"False"` на тесте — **мёртвая настройка**:
  `AppSettings` объявлен с `env_prefix="APP_"`, то есть читает `APP_DEBUG`, и
  `settings.app.debug` в коде не используется.
- `POSTGRES_SQL_LOGS` не задан нигде, `echo` у движка выключен.
- `flake8` в локальном окружении падает: плагин `flake8_return` использует
  `ast.NameConstant`, удалённый в Python 3.12+, а venv на 3.14. `make check` из-за
  этого не проходит независимо от изменений.
