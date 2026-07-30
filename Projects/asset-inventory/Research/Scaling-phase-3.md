---
project: asset-inventory
created: 2026-07-30
tags: [project, research, scaling]
source: docs/scaling-phase-3.md
---

#research

> Фаза 3 — декомпозиция сервиса и масштаб БД.
> Связано: [[Overview]], [[Architecture]]. Источник истины — код;
> заметка отражает состояние на 2026-07-30.

# Scaling Phase 3 — декомпозиция по ролям и масштаб БД

> Цель: изоляция blast-radius (медленный/тяжёлый путь не валит live ingestion) и независимое
> масштабирование ролей; плюс масштаб БД под кратный рост записи/чтения.
>
> Зависимости: пункты 1–3 можно начинать **после Фазы 1** (не требуют Фазы 2). Пункт 4
> (БД) независим, но партиционирование таблиц — крупный отдельный эффорт.

---

## 1. Разделение деплоев по ролям (из одного образа)

### Проблема
Сейчас все 10 сабскрайберов — module-level декораторы `@broker.subscriber` в `app/main.py`,
регистрируются при импорте. `python -m app.main` поднимает **всё** в одном процессе: orders,
7 статистик, recovery, фоновые задачи. Любая роль влияет на остальные (общий event loop, пул,
group rebalance).

### Решение: один образ, роль через env
Ввести `AI_ROLE` (или список топиков) и регистрировать только нужные сабскрайберы.

**Рефактор регистрации** — обернуть декораторы в функции и вызывать по роли:

```python
# app/main.py
def _register_orders(broker): ...        # handle_order_message, order_state
def _register_statistics(broker, sources: set[str]): ...   # выбранные statistic-*
def _register_recovery(broker): ...      # handle_statistic_recovery_message

ROLE_REGISTRARS = {
    "orders":     lambda b: _register_orders(b),
    "statistics": lambda b: _register_statistics(b, sources=settings.app.enabled_sources),
    "recovery":   lambda b: _register_recovery(b),
    "revivor":    lambda b: None,        # не Kafka-сабскрайбер, см. п.2
}

def build_app() -> FastStream:
    role = settings.app.role
    ROLE_REGISTRARS[role](broker)
    return FastStream(broker)
```

Фоновую `resource_projection_sync` запускать только в её роли (вне рассмотрения по решению
команды — оставляем как есть, просто не тащим в чужие роли).

### Деплои
- `asset-inventory-orders` (orders + order_state) — низкий объём, важен порядок по pool/client.
- `asset-inventory-stat-{source}` — по источнику (или сгруппировать), каждый со своим KEDA и
  пулом.
- `asset-inventory-recovery` — Kafka `statistic-recovery`.
- `asset-inventory-revivor` — см. п.2.
- `asset-inventory-worker` (reports) — уже есть.

Каждый деплой — свой configmap (свои `max_workers`, `pool_size`) и свои ресурсы. Образ один,
разница только в `AI_ROLE`/args.

---

## 2. Вынос ревайвора в отдельный деплой (событийно)

### Проблема
Сейчас `OrderService` на каждый заказ кладёт запрос в `_pending_lost_resource_revivals` и через
`schedule_pending_lost_resource_revivals()` → `asyncio.create_task(run_lost_resource_revivor)`
(`app/services/order.py:120–133`). Ревайвор крутится **в том же процессе/loop/пуле**, что
live ingestion — прямой источник прошлого инцидента (тяжёлый скан подвесил Kafka).

### Решение
1. Вместо `create_task` — **публиковать команду ревайва** в топик (`statistic-recovery` или
   новый `lost-resource-revival`) с `key = identifier`:
   ```python
   await broker.publish(
       {"command_type": LOST_RESOURCE_REVIVAL, "pool_name": ..., "tenant_name": ...,
        "fingerprint": ..., "order_id": ...},
       topic=settings.kafka.lost_resource_revival_topic,
       key=identifier.encode(),
   )
   ```
   (механизм `broker.publish` уже используется в `app/helpers/order_status_retry.py:38`).
2. Отдельный деплой `revivor` потребляет этот топик, со **своим пулом БД** и троттлингом.
3. **Дебаунс по identifier:** много заказов на один пул триггерят избыточные полные сканы.
   Схлопывать запросы по ключу в окне — через keyed-топик + воркер-side dedup или Redis-ключ
   `revival:debounce:{identifier}` с TTL.
4. Удалить inline-путь: `_pending_lost_resource_revivals` и `schedule_pending_lost_resource_revivals`
   из `OrderService`, и вызов в `handle_order_message` (`main.py:452`).

Результат: ревайвор физически не может подвесить live ingestion; масштабируется/троттлится
независимо.

---

## 3. Удаление `_AsyncSharedExclusiveLock` и паузы консьюмеров

### Когда безопасно
После Фазы 1 (advisory-лок на ревайворе + event-time guard) **и** выноса ревайвора (п.2).
Тогда in-process пауза избыточна для корректности и сама является источником инцидентов
(зависший recovery держит consumers на паузе).

### Что удалить (`app/main.py`)
- `_AsyncSharedExclusiveLock`, `_statistic_recovery_gate`;
- `_ordinary_statistic_processing`, `_exclusive_statistic_recovery`;
- `_pause_statistic_consumers` / `_resume_statistic_consumers` + `_paused_statistic_partitions_by_consumer`;
- обёртки `async with _ordinary_statistic_processing()` в statistic-хендлерах (просто прямой
  вызов сервиса);
- `handle_statistic_recovery_message` обрабатывает команды как обычный хендлер (запись и так
  идёт через advisory-лок).

> Это финально закрывает класс инцидентов «recovery держит паузу / rebalance-шторм».

---

## 4. Масштаб БД

### 4.1 Read-реплики — ТОЛЬКО для read-only нагрузки, НЕ для горячего пути ingestion

⚠️ **Горячий путь ingestion (`order_lookup`, гидрация графа заказа) оставляем на primary.**
Эти чтения происходят в контексте сообщения, которое затем пишет allocation. Если заказ только
что создан на primary, а реплика отстаёт, `order_lookup` на реплике вернёт `None` → ресурс
уйдёт в `lost_resource` (ложная потеря + лишняя нагрузка на ревайвор). Read-after-write делает
реплику на горячем пути опасной; синхронная репликация, которая это снимает, слишком дорога.

Реплику применяем **только к чисто read-only** операциям, где лаг в секунды безопасен:
- отчёты (`app/reports/`, уже отдельный worker);
- recovery/revivor-сканирование (`get_lost_resources` и аналитика);
- внешние API/выгрузки, если появятся.

- Добавить `POSTGRES_READ_HOST` и **второй движок/фабрику сессий** на DSN реплики:
  ```python
  read_engine = create_async_engine(settings.db.read_dsn, pool_pre_ping=True, ...)
  read_session_factory = async_sessionmaker(read_engine, expire_on_commit=False, class_=AsyncSession)
  ```
  и использовать `read_session_factory` **только** в перечисленных read-only местах.
- Альтернатива «через pgpool»: автобалансировка SELECT'ов pgpool ненадёжна при явных
  транзакциях (`session.begin()` всё уводит на primary) → app-level роутинг предсказуемее.

### 4.2 Тайм-партиционирование `*_allocate_resource`
Append-only история растёт неограниченно (см. CLAUDE.md «Resource versioning»). Нативное
declarative RANGE-партиционирование по `start_date` (или `created_at`), помесячно; управление —
`pg_partman` либо вручную.

**Онлайн-миграция большой таблицы (безопасно):**
1. создать партиционированную таблицу-двойник `*_allocate_resource_p`;
2. перелить данные пачками (или `ATTACH` существующей как партиции, если структура позволяет);
3. переключить запись (rename в транзакции);
4. построить индексы на партициях `CONCURRENTLY`.

Это крупный отдельный эффорт — выносить в свой проект/PR, не смешивать с остальным.

### 4.3 Ретеншн/архивация и autovacuum
- Старые партиции `DETACH` + архив в холодное хранилище / `DROP` по политике.
- Тюнинг autovacuum под high-write таблицы (scale factor ниже, чаще).

---

## Порядок внутри Фазы 3
1. Разделение деплоев по ролям (п.1) — после Фазы 1, даёт изоляцию сразу.
2. Вынос ревайвора (п.2).
3. Удаление паузы/`_AsyncSharedExclusiveLock` (п.3).
4. Read-реплики (4.1).
5. Тайм-партиционирование + ретеншн (4.2–4.3) — отдельным крупным эффортом.

## Риски/откат
- Разделение ролей: чистый рефактор регистрации; откат = вернуть один `AI_ROLE=all`.
- Вынос ревайвора: на переходе оставить feature-flag (старый inline ↔ новый топик), убрать
  inline после стабилизации.
- Удаление лока: делать **только** после п.2 и подтверждённой Фазы 1; иначе вернётся гонка #2.
- Read-реплики: начать с роутинга одного-двух заведомо безопасных запросов, расширять по факту.
- Партиционирование: необратимо по структуре — отдельный проект с бэкапом и репетицией на тесте.
