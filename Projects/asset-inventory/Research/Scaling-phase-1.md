---
project: asset-inventory
created: 2026-07-30
tags: [project, research, scaling]
source: docs/scaling-phase-1.md
---

#research

> Фаза 1 — корректность (гейт перед параллелизмом).
> Связано: [[Overview]], [[Architecture]]. Источник истины — код;
> заметка отражает состояние на 2026-07-30.

# Scaling Phase 1 — close the remaining correctness gaps

> Goal: make `replicas > 1` and/or `max_workers > 1` **safe**, so Phase 2 (repartition
> topics + horizontal scale) can proceed without data corruption.
>
> Context: as of `git_tag 3.55` the hot path is already mostly multi-pod-safe —
> `get_or_upsert` uses `ON CONFLICT`, and `_sync_allocations` serializes the
> read-modify-write of `AllocateResource` with `pg_advisory_xact_lock(order_resource_id)`.
> Two gaps remain (see CLAUDE.md "Known race conditions" #2 and #4). Phase 1 closes
> exactly those two. No new architecture, no topic changes.

There are **two** independent changes. Both are small and shippable on the current
single-pod deployment without behaviour change, so they de-risk the later scale-out.

---

## 1. Run the revivor through the same advisory lock as the live path

### Problem
`LostResourceRevivor._process_revived_resource`
(`app/tasks/lost_resource_revivor.py:272–314`) does `get_last → update / close → bulk_create`
**without** taking `pg_advisory_xact_lock(order_resource_id)`. The live statistics path
(`optimized_base_resource._sync_allocations`) does take it. So revivor vs. live handler for
the same `order_resource_id` are **not** serialized against each other — today this is masked
only because everything runs in one pod (`replicas: 1`). Under multiple pods (or once the
revivor moves to its own deployment) this races: double "open" allocations, conflicting
`end_date`.

### Fix
Make both paths acquire the **same** lock, derived **identically**.

**Step 1 — extract the lock helper** so the key derivation can't drift between callers.
New `app/helpers/db_locks.py`:

```python
from uuid import UUID
from sqlalchemy import text

async def advisory_xact_lock_order_resources(session, order_resource_ids: list[UUID]) -> None:
    """Transaction-scoped Postgres advisory lock per order_resource_id.

    Sorted acquisition order prevents deadlocks on overlapping id sets.
    Lock is held until the *outer* transaction commits/rolls back (xact-scoped),
    so callers should keep that transaction short.
    """
    if not order_resource_ids:
        return
    sorted_ids = [str(uid) for uid in sorted(order_resource_ids)]
    await session.execute(
        text(
            """
            SELECT pg_advisory_xact_lock(hashtext(id))
            FROM unnest(CAST(:ids AS text[])) AS t(id)
            ORDER BY id
            """
        ),
        {"ids": sorted_ids},
    )
```

**Step 2 — use it in the live path.** Replace the body of
`OptimizedBaseResourceService._lock_order_resources` (`optimized_base_resource.py:686`)
with a call to the helper (keeps the existing public method, just delegates). This guarantees
the live path and the revivor compute the identical `hashtext(str(uuid))` key.

**Step 3 — use it in the revivor.** In `_process_revived_resource`, before the
`get_last → update/create` block, acquire the lock for that `order_resource.id`:

```python
await advisory_xact_lock_order_resources(self.db_session, [order_resource.id])
```

**Note on resource creation.** The lock is keyed by `order_resource_id`, which already exists
at this point: the revivor resolves an order → its `order_resource`s and matches by
`resource_id` (`_process_revived_resource`). Where the revivor/order path *creates* an
`order_resource` (temp tariffs for paper orders), that creation is already atomic via
`ON CONFLICT` (`app/cruds/order_resource.py:60,78`) — it does **not** rely on this advisory
lock. So the lock is correct to take by `order_resource_id` after the resource is known; no
separate "create under lock" step is required.

### Important: transaction granularity
The advisory lock is **xact-scoped** — released only when the revivor's transaction
commits. The revivor currently commits **per batch** (`batch_size=100`, one outer
transaction per batch with `begin_nested()` savepoints per resource). If you take the lock
inside that batch transaction, locks accumulate for up to 100 `order_resource_id`s and are
held until the batch commit — increasing contention with live handlers.

Pick one:
- **(recommended)** Commit per resource in the revivor (drop the batch-level transaction;
  keep a small batch only for logging/progress). Lock is then held for one resource at a
  time. Slightly more commits, but the revivor is a background/throttled job — throughput is
  not its priority, and this is the cleanest correctness story.
- Or keep batching but **lower `batch_size`** (e.g. 10–20) to bound lock hold time.

### Defense-in-depth — "Phase 1.5", recommended fail-safe
A partial unique index makes a missed lock a hard error instead of silent duplication.

Timing note (two reviewers disagreed; resolved): the write-contention worry is largely **moot
once the advisory lock exists** — concurrent writes to the same active-allocation key are
already serialized by the lock, so the index adds only normal maintenance cost, not hot-key
contention. The **only** real prerequisite is a one-time dedup pass over existing
`end_date IS NULL` duplicates. So this does **not** need to wait for Phase 2 — ship it as soon
as dedup is clean if you want the fail-safe early.

Apply it to **all** allocate tables (the source-specific `*_allocate_resource` and the general
`allocate_resource`), not just one. The index itself:

```sql
-- per source-specific *_allocate_resource table AND general allocate_resource
CREATE UNIQUE INDEX CONCURRENTLY uq_<table>_active_allocation
ON <table> (order_resource_id, instance_name, instance_id)
WHERE end_date IS NULL;
```

⚠️ Will fail if duplicate active allocations already exist (from past races). Sequence:
1. dedup query to find/resolve existing `end_date IS NULL` duplicates,
2. then `CREATE UNIQUE INDEX CONCURRENTLY`,
3. handle the resulting `IntegrityError` in `bulk_create` as "already created → reload".

This is belt-and-suspenders; the lock alone is sufficient for correctness.

---

## 2. Event-time guard in `_sync_allocations`

### Problem
`_sync_allocations` (`optimized_base_resource.py:626–651`) applies whatever message it gets,
comparing only `quantity` and stamping `last_collected_at`. It never checks whether the
incoming message is **newer** than what is already stored. With single-partition topics +
`max_workers=1`, order is preserved, so this is latent. The moment topics are repartitioned
or `max_workers > 1`, an **older** `collect_time` can arrive after a newer one and:
- move `last_collected_at` **backwards** (same-quantity branch), or
- close a newer allocation and recreate an older one (changed-quantity branch).

### Fix
The incoming event time is already computed in the function:
- `allocation_start = data_in.time_start or data_in.collect_time`
- `allocation_last_collected = data_in.time_end or data_in.collect_time`

The stored row has `start_date` and `last_collected_at`. Add a staleness check **inside the
existing per-resource loop** (already under the advisory lock, so read+decision+write is
atomic per `order_resource_id`):

```python
allocate_resource_db = latest_allocations_by_key.get(allocation_key)

# Event-time guard: ignore messages older than what we already recorded.
# Makes reprocessing (at-least-once / rebalance) a no-op and protects against
# out-of-order delivery once topics are repartitioned or max_workers > 1.
if allocate_resource_db is not None and (
    allocation_last_collected <= allocate_resource_db.last_collected_at
):
    continue

if not allocate_resource_db and allocate_resource.quantity != 0:
    allocate_resources_to_create.append(allocate_resource)
elif allocate_resource_db and allocate_resource_db.quantity == allocate_resource.quantity:
    same_quantity_update_ids.append(allocate_resource_db.id)
    ...
```

Notes:
- **Per allocation key, not per message.** One statistics message (one VM) matches several
  `order_resource_id`s (cpu/ram/disk) under a single `collect_time`. The guard is evaluated
  inside the per-resource loop, keyed by `(order_resource_id, instance_name, instance_id)`, and
  the `continue` skips a stale resource entirely — so it never reaches `same_quantity` /
  `bulk_close` / `bulk_create`. Some resources in the same message can be stale while others
  are fresh; each is decided independently. The guard MUST sit before all three branches (it
  does, in the code above).
- Compare against `last_collected_at` (monotonic "we have data up to here"), not
  `start_date`.
- The "no existing allocation" branch is intentionally left as-is: a first-ever message
  should always create, regardless of age.
- `time_start`/`time_end` are optional on some sources; the `or collect_time` fallback keeps
  the comparison well-defined for all of them.
- **Tie on equal event-time.** `<=` makes an exact replay a no-op, but two *different*
  messages with the **same** `collect_time` processed concurrently (only possible with
  `max_workers > 1` on one partition) would be a nondeterministic last-write-wins. Mitigation,
  in order of preference: (a) keep strict per-partition serialization (`max_workers=1` per
  partition) so equal timestamps resolve by Kafka offset order deterministically; (b) if you
  must run `max_workers > 1`, extend the guard with a secondary key (`collect_time` + Kafka
  offset / collector `sequence`). Whether (b) is needed depends on the collector message
  format — see open questions in `docs/scaling-roadmap.md`.

---

## What Phase 1 does NOT include
- No new Kafka partitions, no repartitioning, no `max_workers`/`replicas` bump — those are
  Phase 2, and they are only **safe** once the two changes above are merged.
- No removal of `_AsyncSharedExclusiveLock` yet. Once the revivor is advisory-locked (#1)
  and the event-time guard is in (#2), the in-process pause becomes redundant for
  correctness and can be removed in Phase 3 when the revivor moves to its own deployment.

## Tests to add
- `_sync_allocations`: older `collect_time` after a newer one → no write (guard works);
  exact replay → no-op; newer message → applied.
- revivor: asserts `advisory_xact_lock_order_resources` is called for each processed
  `order_resource_id` before `get_last`/create (spy/mock).
- helper: identical lock key for the same UUID across both call sites.

## Suggested order of merge
1. Extract `advisory_xact_lock_order_resources` helper + delegate the live path to it (pure
   refactor, no behaviour change).
2. Add the event-time guard (#2) + tests — safe on single pod, immediately makes reprocessing
   idempotent.
3. Add the advisory lock to the revivor (#1) + per-resource commit + tests.
4. (Phase 1.5, separate PR) dedup + partial unique index (fail-safe).
