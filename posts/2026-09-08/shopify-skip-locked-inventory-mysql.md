---
title: "We replaced Redis with MySQL for inventory reservations—and it scaled"
source: https://shopify.engineering/scaling-inventory-reservations
author: Emilie Noel
company: Shopify
date_posted: 2026-05-12
date_digested: 2026-09-08
---

# We replaced Redis with MySQL for inventory reservations—and it scaled

## What's new to learn

1. **SKIP LOCKED as a concurrency primitive** — `SELECT … FOR UPDATE SKIP LOCKED` lets multiple concurrent transactions each claim a different row without waiting, turning any relational table into a multi-producer, multi-consumer work queue.
2. **Row-per-unit data model** — encoding "one claimable resource" as one row, rather than tracking a mutable count column, shifts contention from a shared scalar to per-row locks that never collide.
3. **InnoDB's secondary-index double-dip** — a `SELECT FOR UPDATE` on a secondary index acquires two locks per row (one on the secondary index entry, one on the clustered PK leaf); placing filter columns in the PK collapses this to one lock and eliminates phantom contention on unrelated rows.

## Prerequisites

- InnoDB clustered index: rows are physically stored in primary-key order; every secondary index stores PK values as row pointers, not physical page offsets.
- MySQL transaction isolation levels: `REPEATABLE READ` (default) uses next-key locks to prevent phantoms; `READ COMMITTED` uses only record locks.
- Gap locks: in `REPEATABLE READ`, InnoDB locks the gap *between* index entries on range queries, preventing inserts into that range even when no matching rows exist.

## The core idea

The old Redis design treated inventory as a shared mutable counter: "this item has 42 units available" was a single Redis key. Reserving three units meant `DECRBY 42 3 → 39`. Redis's single-threaded command execution made this safe, but it came at a cost: the reservation lived in Redis while the authoritative inventory record lived in MySQL, so every checkout required writes to two systems that could never be wrapped in a single atomic transaction.

The MySQL redesign flips the data model. Instead of one row per item with a `quantity` column, the system stores **one row per reservable unit**. An item with 100 units has 100 rows in the `reservation_units` table. Reserving three units means selecting three rows and marking them reserved — a single SQL transaction touching only MySQL.

The concurrency problem shifts: instead of "how do we atomically decrement a shared counter?" it becomes "how do two concurrent buyers avoid claiming the same row?" The answer is `FOR UPDATE SKIP LOCKED`: each transaction tries to lock its chosen rows and, when it finds a row already locked by another transaction, skips it rather than waiting. Two concurrent buyers naturally land on different rows.

## Mechanics

### Schema

```sql
CREATE TABLE reservation_units (
  shop_id          BIGINT NOT NULL,
  item_id          BIGINT NOT NULL,
  group_id         BIGINT NOT NULL,
  id               BIGINT NOT NULL,
  status           ENUM('available', 'reserved') NOT NULL DEFAULT 'available',
  reservation_id   BIGINT,
  PRIMARY KEY (shop_id, item_id, group_id, id)
);
```

The composite PK `(shop_id, item_id, group_id, id)` contains every column used to filter when claiming units. Because InnoDB stores rows in PK order, a filter on all four columns is a clustered-index point lookup: one B+ tree traversal, one row lock, done.

### Claiming units

```sql
SELECT id
FROM   reservation_units
WHERE  shop_id = ? AND item_id = ? AND group_id = 0 AND status = 'available'
LIMIT  3
FOR UPDATE SKIP LOCKED;
```

InnoDB evaluates each candidate row in PK order. For each row it tries to acquire an exclusive lock. If the row is already locked by another transaction, `SKIP LOCKED` advances to the next candidate without waiting. The result is a set of row IDs that this transaction now owns exclusively.

A second query then updates those IDs to `status = 'reserved'`, and the transaction commits. The whole claim is one round-trip that contends only with transactions claiming the exact same rows — which almost never happens because the candidate pool is large.

### Composite PK vs secondary index — why it matters

Without the composite PK, the `WHERE` clause would hit a secondary index on `(shop_id, item_id, group_id, status)`. In `REPEATABLE READ`, InnoDB acquires:

1. A shared next-key lock on the secondary index entry (covering the gap before it).
2. An exclusive lock on the corresponding PK leaf page.

That's two locks per row, and the gap lock on step 1 prevents *any* other transaction from inserting a new `available` row into the scanned range — directly blocking the replenishment process that refills the pool. With the composite PK, InnoDB goes straight to the clustered index: one lock per row, no gap locks on the range.

### Transaction isolation level

Even with the composite PK, `REPEATABLE READ` applies a gap lock on the empty range when the pool is exhausted — blocking replenishment inserts and causing deadlocks. Switching to `READ COMMITTED` disables gap locking for non-unique scans. The only cost is that phantom rows are now theoretically visible within a transaction, which is acceptable here because each claim ends in a short-lived transaction.

### Bounded pool and replenishment

Materializing every unit as a row would create unbounded table growth. Shopify caps the pool at **1,000 rows per item** and runs a background replenishment process: whenever the available count falls below a threshold, it inserts new rows up to the cap. On most items this is invisible latency; for flash-sale items it requires tuning the cap and trigger threshold.

The pool cap also serves as a natural rate limiter: if 1,000 concurrent claims arrive simultaneously, the 1,001st gets an empty result set rather than fighting over one overloaded counter.

### At peak load

The system survived Shopify's Black Friday 2025 traffic spike with merchant sales peaking at $5.1 million per minute — a record. At those rates, multiple checkout processes are simultaneously claiming units for the same hot items; `SKIP LOCKED` ensures they each get distinct rows with zero blocking between them.

## Where it breaks

**No FIFO guarantee.** `SKIP LOCKED` returns rows in primary-key order, skipping locked ones. The n-th buyer does not necessarily get the n-th unit in arrival order. For inventory this is fine; for ticket seat-selection or waitlist management it may not be.

**Replenishment lag.** If demand spikes faster than the background process can insert rows, the pool empties and buyers see "out of stock" for items that physically exist. The 1,000-row cap must be tuned per-item for flash sales.

**Cross-database portability.** `SKIP LOCKED` is available in MySQL 8+ and PostgreSQL 9.5+, but not in SQLite, SQL Server (not standard), or many NoSQL stores. The pattern is InnoDB/PostgreSQL-specific.

**Discrete units only.** The model requires that "a unit" is a well-defined indivisible thing. Fractional inventory (e.g., weight-based goods where a merchant has 3.75 kg and a buyer wants 1.2 kg) cannot be modeled as rows without a more complex assignment algorithm.

**Table size scales with reservation volume, not item count.** A high-throughput item with 1,000 rows that turns over rapidly generates heavy InnoDB page churn. Monitoring pool fill rate and row churn is necessary at scale.

## Why it works

The deeper principle: **shift contention from shared mutable state to parallel exclusive claims**.

The Redis counter `DECRBY` requires exclusive access to a single key — the concurrency domain is the entire item. Any two simultaneous purchasers of the same item must serialize. The row-per-unit model makes the concurrency domain the individual row. Two purchasers of the same item can proceed in parallel as long as they land on different rows, which `SKIP LOCKED` arranges atomically.

This is the same insight that appears everywhere in high-concurrency systems:

- **Database sharding**: instead of one table-wide lock, one lock per shard. Shards don't conflict.
- **MVCC (multi-version concurrency control)**: instead of locking the current version, readers get their own snapshot version. Readers never block writers.
- **The LMAX Disruptor**: instead of a shared queue with a lock, pre-allocate a ring buffer; producers and consumers each own a cursor, and no lock is taken if they stay apart.
- **Per-CPU variables in the Linux kernel**: instead of one global counter, one counter per CPU. Cross-CPU operations are rare, so the data structure is almost always lock-free.

In all these cases the pattern is identical: **decompose shared state into N independent units, assign each concurrent actor one unit, let them proceed without coordination**. `FOR UPDATE SKIP LOCKED` is the SQL spelling of "try-lock, and if it fails, try the next one" — the same work-stealing loop that makes Cilk and Go's goroutine scheduler scale across cores.

The additional composite-PK insight — that the schema itself determines lock granularity — illustrates that concurrency is not just a runtime choice but a design-time one. InnoDB's locking behavior is mechanically determined by index structure. Choosing the wrong indexes doesn't just affect query speed; it changes which rows get locked, which transactions block, and which deadlocks become possible. The schema IS the concurrency contract.

## Going deeper

1. **37signals' Solid Queue** — the inspiration Shopify cites. Solid Queue is a Rails background-job backend built on MySQL with `SKIP LOCKED`. Its README explains the pattern for job queues, which is structurally identical to inventory reservation: [github.com/rails/solid_queue](https://github.com/rails/solid_queue).

2. **PostgreSQL's SKIP LOCKED documentation** — covers the same primitive in Postgres, including the job-queue use case explicitly. Also explains why `NOWAIT` (fail immediately if locked) is the complementary primitive: [postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE).

3. **InnoDB Locking (MySQL Reference Manual)** — the authoritative source on gap locks, next-key locks, and how isolation level selects among them. Understanding this is prerequisite to reasoning about any schema that uses `SELECT FOR UPDATE`: [dev.mysql.com/doc/refman/8.0/en/innodb-locking.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html).
