---
title: "How Postgres Makes Transactions Atomic"
source: https://brandur.org/postgres-atomicity
author: Brandur Leach
company: Personal blog (author formerly of Heroku / Stripe)
date_posted: 2017-09-12
date_digested: 2026-09-26
---

# How Postgres Makes Transactions Atomic

## What's new to learn

1. **xmin/xmax in-heap versioning**: Every PostgreSQL row stores two hidden system columns — `xmin` (the transaction ID that created this version) and `xmax` (the transaction ID that deleted or superseded it) — encoding the row's full version history *inline in the heap*, so any snapshot visibility check is a pure arithmetic comparison with no separate version store.

2. **The Commit Log (CLOG / pg_xact)**: A 2-bit-per-XID status file that is the authoritative oracle for "did this transaction commit?" — xmin and xmax are just *keys* used to query this oracle; the MVCC visibility rule is computed as `(xmin's entry in CLOG) AND (xmax's entry in CLOG)`.

3. **VACUUM as garbage collector**: Because every `UPDATE` is implemented as a logical `DELETE` (set xmax) plus `INSERT` (new xmin), dead tuple versions accumulate in the heap indefinitely until VACUUM reclaims them — making PostgreSQL's storage fundamentally append-only at the tuple level, the same pattern as LSM trees and immutable object stores.

## Prerequisites

- What a database transaction and transaction ID (XID) are
- The basic notion of MVCC: readers don't block writers; each transaction sees a consistent snapshot
- Familiarity with transaction isolation levels (read committed, repeatable read, serializable)
- What a "heap file" is in database storage: rows packed into fixed-size pages (8 KB by default in PostgreSQL)
- Not required, but useful: the ARIES WAL mechanism (PostgreSQL uses both WAL *and* MVCC — they solve different problems)

## The core idea

PostgreSQL achieves both atomicity (either all of a transaction's writes are visible, or none are) and isolation (each transaction sees a consistent snapshot) through a **single shared mechanism: stamping every tuple with the transaction IDs that created and destroyed it, and applying a per-snapshot visibility test at read time**.

The rules are refreshingly simple:

| Operation | What PostgreSQL actually does |
|-----------|-------------------------------|
| `INSERT`  | Write new row with `xmin = current_txn_id`, `xmax = 0` |
| `DELETE`  | Mark the row: `xmax = current_txn_id` |
| `UPDATE`  | Mark old row's `xmax = current_txn_id`; write new row with `xmin = current_txn_id`, `xmax = 0` |
| `SELECT`  | For each row, evaluate: *is xmin committed before my snapshot? is xmax either 0, aborted, or after my snapshot?* |

Nothing is ever overwritten in-place. Atomicity falls out for free: until a transaction commits, its XID is not marked COMMITTED in the Commit Log, so none of its writes are visible to other transactions regardless of what's already on disk.

## Mechanics

### Tuple header layout

Every PostgreSQL row ("heap tuple") carries hidden metadata prepended to the user-visible columns:

```c
typedef struct HeapTupleHeaderData {
    TransactionId   t_xmin;      // inserting transaction's XID
    TransactionId   t_xmax;      // deleting/locking transaction's XID
    CommandId       t_cid;       // cmin and cmax packed together
    ItemPointerData t_ctid;      // self-pointer, or pointer to newer version
    uint16          t_infomask2; // attribute count + hint bits
    uint16          t_infomask;  // HEAP_XMIN_COMMITTED, HEAP_XMAX_COMMITTED, etc.
    uint8           t_hoff;      // offset to start of user data
} HeapTupleHeaderData;
```

The critical fields: `t_xmin` (who created me), `t_xmax` (who deleted me; 0 means alive), and `t_ctid` (my location, or the ctid of my replacement in a HOT chain).

### The Commit Log (CLOG)

The key question for visibility — "did xmin's transaction commit before my snapshot?" — is answered by the **Commit Log**, stored as files under `$PGDATA/pg_xact/`. Each XID gets exactly 2 bits:

```
00 = IN_PROGRESS
01 = COMMITTED
10 = ABORTED
11 = SUB_COMMITTED (subtransaction)
```

This is a compact, append-only file: committing a transaction is essentially `pg_xact[xid] = COMMITTED` — a single 2-bit write, followed by an `fsync()`. Aborts are `pg_xact[xid] = ABORTED`. The CLOG files are mmap'd into the shared buffer pool for fast random access.

### Hint bits: caching CLOG lookups

A naïve implementation would consult the CLOG on every tuple read, which is expensive for hot rows. PostgreSQL caches the result of the CLOG lookup in the tuple header's `infomask` bits (`HEAP_XMIN_COMMITTED`, `HEAP_XMIN_INVALID`, `HEAP_XMAX_COMMITTED`, `HEAP_XMAX_INVALID`). The first backend that reads a tuple whose XID status is not yet cached checks the CLOG, confirms the status, and **sets the hint bits in-place on the heap page**. Subsequent readers find the bits already set and skip the CLOG entirely.

This is a write to the heap without a WAL entry — hint bits are not crash-safe to set but also don't need to be, since the CLOG is the authoritative record and bits can always be recomputed. This gives row-level CLOG-lookup amortization without coordination.

### Transaction snapshots

When a transaction begins, PostgreSQL takes a snapshot by recording:

```c
typedef struct SnapshotData {
    TransactionId xmin;    // lowest XID still active
    TransactionId xmax;    // next XID to be assigned
    TransactionId *xip;    // list of active XIDs in [xmin, xmax)
    uint32         xcnt;   // length of xip
    ...
} SnapshotData;
```

A tuple with `t_xmin = X` is visible if:
- `X < snapshot.xmin` (committed long before snapshot) **or**
- `snapshot.xmin ≤ X < snapshot.xmax` **and** `X` is not in `xip` (committed while snapshot was being taken)

A tuple is dead if its `t_xmax` meets the same "committed before snapshot" test.

Under READ COMMITTED, a fresh snapshot is taken on every statement. Under REPEATABLE READ / SERIALIZABLE, the same snapshot is held for the entire transaction.

### UPDATE is DELETE + INSERT

The most important consequence: an `UPDATE` leaves the original row in the heap with its `xmax` set, and adds a brand-new row. Two versions of the row coexist on disk until VACUUM. Running `pageinspect` shows this directly:

```sql
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('users', 0));

 lp | t_xmin | t_xmax | t_ctid
----+--------+--------+--------
  1 |   1234 |   1235 | (0,2)   ← dead: xmax=1235 committed
  2 |   1235 |      0 | (0,2)   ← live current version
```

Row 1 is the pre-UPDATE ghost. Row 2 is the post-UPDATE version. Both live in the same 8 KB heap page.

### VACUUM: the garbage collector

VACUUM scans the heap looking for "dead" tuples — rows whose `xmax` is committed and whose XID is older than the oldest live snapshot in the system (the `oldestXmin` horizon). For each dead tuple it finds, VACUUM:

1. Marks the tuple's line pointer as dead (LP_DEAD)
2. Updates the **Free Space Map (FSM)** to record available space on that page for future inserts
3. Updates the **Visibility Map (VM)** — one bit per heap page — to signal "all tuples on this page are visible to all current and future transactions" once a page is clean; index-only scans can skip the heap entirely for pages marked in the VM

VACUUM does *not* truncate the file or return space to the OS (that's `VACUUM FULL`, which locks the table and rewrites it). Normal VACUUM only reclaims space for *reuse within the same table*.

### HOT (Heap Only Tuples)

When an UPDATE touches only columns that are not covered by any index, PostgreSQL can avoid creating a new index entry for the new tuple. Instead, `t_ctid` of the dead tuple is set to point to the new tuple, forming a **HOT chain**. An index lookup that lands on the old (dead) tuple follows the ctid chain to the new version — no index entry needed. This greatly reduces index bloat and index write amplification on high-churn tables.

### Transaction ID wraparound

XIDs are 32-bit integers, wrapping at ~4.3 billion. PostgreSQL uses modular arithmetic so each XID is "in the future" for XIDs more than 2^31 away — meaning if a table hasn't been vacuumed and 2 billion transactions have elapsed, old rows would appear to be in the future and become invisible. PostgreSQL prevents this by **freezing** old tuples during VACUUM: tuples with xmin older than `vacuum_freeze_min_age` transactions get their xmin replaced with the special `FrozenTransactionId` (XID 2), which is always considered committed and visible. If a database approaches the wraparound horizon without being frozen, PostgreSQL enters **wraparound protection mode** and halts all writes — a production emergency.

## Where it breaks

**Table bloat**: High-`UPDATE`-rate tables that are not vacuumed frequently enough accumulate dead tuples. PostgreSQL cannot return this space to the OS without `VACUUM FULL` (which takes an exclusive lock). A table that was once large and has since been heavily updated will occupy far more disk than its live rows require.

**Long-running transactions block VACUUM**: VACUUM only reclaims tuples dead before the `oldestXmin` horizon — the xmin of the oldest live snapshot. A single `BEGIN` left idle (e.g., an application holding a connection open) freezes the horizon and prevents the entire cluster from reclaiming dead tuples, causing unbounded table growth.

**Index bloat**: PostgreSQL indexes hold a pointer to each tuple version, not just the latest. A dead heap tuple with many covering-index pointers is not cleaned up by VACUUM until the index entries are also reclaimed (in a separate `VACUUM` phase). High-churn indexed columns accumulate index bloat even with healthy table vacuum.

**Snapshot too long-lived (REPEATABLE READ)**: Long-running REPEATABLE READ transactions hold a snapshot from their start time, preventing VACUUM from advancing past it. OLAP queries that run for hours against an OLTP database are a common culprit.

**XID wraparound** (described above): requires periodic maintenance and monitoring of `age(datfrozenxid)`.

## Why it works

The deeper principle: **immutable-append storage at the finest practical granularity, plus deferred garbage collection, gives you MVCC without distributed coordination.**

By never overwriting in-place, PostgreSQL sidesteps the need for read-write locking entirely — a writer adding a new row version never conflicts with a reader examining an old one. The only coordination needed is a tiny commit record in the CLOG, which is a sequential append. This is why PostgreSQL's write path is: (1) write row versions to heap buffers, (2) write WAL records, (3) flip 2 bits in pg_xact.

The same pattern appears everywhere in systems design:

| System | "Tuple" = | "CLOG" = | "VACUUM" = |
|--------|-----------|----------|------------|
| LSM tree (RocksDB) | SSTable entry | MANIFEST (latest seqnum is current) | Compaction |
| Kafka | Log offset | Consumer group offset commit | Log segment deletion |
| Git object store | Blob/tree/commit | Refs (HEAD, branches) | `git gc` |
| Apache Iceberg | Data file | Manifest + snapshot log | Expiration job |
| Percolator (CockroachDB) | Key-version pair | Lock + commit column | GC background worker |

The crucial difference from Oracle/SQL Server's model: Oracle stores the *undo* image in a separate *undo tablespace*, updates rows in-place, and reconstructs old versions on demand. PostgreSQL stores *all versions* in the heap. Oracle's model gives a smaller heap but requires undo lookups under read pressure; PostgreSQL's model makes reads fast (the old version is right there) but makes heaps grow under write pressure.

The CLOG-as-oracle concept appears in every distributed MVCC database too: Percolator uses a "lock column" + "write column" in Bigtable (the write column is the commit record); CockroachDB uses a MVCC timestamp written atomically to the primary key's intent resolution record; Aurora DSQL uses an OCC adjudicator. In all cases, the row-level metadata is just a key, and the authoritative commit status lives in a separate, durable, simply-updated store.

## Going deeper

1. **"The Internals of PostgreSQL" by Hironobu Suzuki** (https://www.interdb.jp/pg/) — free online book; Chapter 5 (heap storage and tuples), Chapter 6 (vacuum), and Chapter 2 (process + shared buffer architecture) are directly relevant.

2. **"Every UPDATE Leaves a Ghost: MVCC, Bloat, and VACUUM in PostgreSQL"** — PlanetScale blog (https://planetscale.com/blog/postgresql-mvcc) — shows live `pageinspect` output demonstrating dead tuples in heap pages, and explains how to diagnose and mitigate bloat in production.

3. **"Percolator: Large-scale Incremental Processing Using Distributed Transactions and Notifications"** (Google Research) — the same xmin/xmax pattern generalised to Bigtable; reading both back-to-back makes the architectural continuity obvious.
