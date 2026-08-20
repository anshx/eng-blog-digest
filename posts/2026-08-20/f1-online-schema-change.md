---
title: "Online, Asynchronous Schema Change in F1"
source: https://dl.acm.org/doi/10.14778/2536222.2536230
author: Ian Rae, Eric Rollins, Jeff Shute, Sukhdeep Sodhi, Radek Vingralek
company: Google
date_posted: 2013-08-26
date_digested: 2026-08-20
---

# Online, Asynchronous Schema Change in F1

## What's new to learn

1. **Schema state machine with adjacent-compatibility** — A schema element (index, column, constraint) can pass through a sequence of intermediate states where any two *consecutive* states are safe to run simultaneously in a cluster, even on different servers.

2. **Delete-only / write-only as the two canonical intermediate states** — Before an index can be used for reads it must go through a delete-only phase (prevents orphan index entries) then a write-only phase with a full backfill (makes the index complete for existing rows), then public. Every state transition waits one full lease period so no server is ever more than one state behind.

3. **Schema lease as the distributed coordination primitive** — Servers cache the schema descriptor under a time-bounded lease. The protocol advances the state machine by mutating the descriptor in the shared metadata store and waiting ≥ one lease period before advancing again. This turns a global coordination problem into a bounded-wait local check.

## Prerequisites

- Basic understanding of database indexes: why a secondary index duplicates key→row-pointer mappings for fast lookup
- MVCC or row-version awareness: that a DELETE on a row and a DELETE on an index entry are separate operations
- High-level grasp of distributed systems with caching: that "everyone switches at the same instant" is not achievable without a global stop

## The core idea

Every database eventually needs to add a column, create an index, or change a constraint — without taking the system offline. In a single-server database this is hard enough (table lock, full rewrite). In a globally distributed database with hundreds of stateless servers each caching the schema locally, it's a different class of problem: you can't coordinate a simultaneous switch, so at any point in time some servers are running schema version N and others have already advanced to version N+1.

F1's key realization: rather than trying to make the switch simultaneous, **design the state sequence so that any two consecutive states are safe when used by different servers at the same time**. Call this property *adjacent-compatibility*.

For adding an index the safe sequence is:

```
absent  →  delete-only  →  write-only  →  public
```

Each arrow is a state the entire cluster eventually reaches before the next arrow is crossed. Between arrow crossings, the cluster may have servers at state N and state N+1 simultaneously — and the protocol guarantees that no server is ever more than one state behind the global leader.

The protocol doesn't require distributed locks or global barriers. It requires only:
1. A shared durable metadata store (Spanner in F1's case) that holds the current schema state
2. A schema lease: each server caches the schema for at most L seconds
3. A rule: wait at least L seconds at each state before advancing to the next

That's it. Every server will have seen the new state within L seconds of the advance, and the adjacent-compatibility property guarantees safety throughout.

## Mechanics

### The four states

| State | Reads from element | Writes to element | Deletes from element |
|---|---|---|---|
| **absent** | No | No | No |
| **delete-only** | No | No | Yes |
| **write-only** | No | Yes | Yes |
| **public** | Yes | Yes | Yes |

"Element" means an index, a column, or a table-level constraint — any schema entity that maps rows to auxiliary storage.

### Forward protocol: adding an index

**Step 1 — absent → delete-only** (wait ≥ L seconds)

All servers start in absent: they ignore the index entirely. The global schema descriptor is updated to delete-only. Within L seconds, all servers have refreshed their cached descriptor and now execute DELETEs against the index whenever they delete a row.

Cluster state during this window:
- absent servers: insert rows without writing to index, delete rows without touching index
- delete-only servers: insert rows without writing to index, delete rows by removing the index entry

Can this corrupt the index? No. Neither state ever *adds* an entry. absent ignores the index on insert; delete-only also ignores the index on insert. So the index can only contain entries that a delete-only server already removed — meaning the index is either empty or has entries from before this server reached delete-only. Either way, no orphan entries (entries pointing to non-existent rows) are created.

**Step 2 — delete-only → write-only** (wait ≥ L seconds, then run backfill)

The descriptor advances to write-only. Servers at write-only now INSERT into the index on row inserts and UPDATE the index on row updates, in addition to deleting on row deletes.

Cluster state during this window:
- delete-only servers: only remove index entries on row deletes
- write-only servers: fully maintain the index for new writes

Safety: if a write-only server inserts row X (adds to index), and a delete-only server then deletes row X (removes from index), consistency is preserved. If a delete-only server inserts row X (does NOT add to index), and a write-only server later queries the index, it won't find X — but write-only servers can't use the index for reads (read is forbidden in write-only), so no incorrect query results are returned.

After the entire cluster reaches write-only, a **backfill** job runs: it scans all existing table rows in small batches and inserts the missing index entries. Because write-only is already active everywhere, any concurrent writes during backfill correctly update the index; the backfill just needs to catch up on rows that existed before write-only started. After backfill completes, the index is complete and consistent.

**Step 3 — write-only → public** (wait ≥ L seconds)

The descriptor advances to public. Servers at public use the index to answer queries. Servers still in write-only continue to maintain the index but don't query it. Since both maintain the index identically, the index is always consistent from a write perspective, and read-from-index is safe for public servers.

### Reverse protocol: dropping an index

The reverse is:

```
public  →  write-only  →  delete-only  →  absent
```

- **public → write-only**: servers stop using the index for reads. Index is still maintained, so no inconsistency.
- **write-only → delete-only**: servers stop inserting/updating the index on new writes. Rows inserted after this point won't have index entries.
- **delete-only → absent**: servers stop deleting index entries on row deletes. The index is now fully orphaned (contains stale entries), but since absent servers never read from it and it's about to be physically deleted, this is safe.

### Why the "one version behind" invariant is load-bearing

Skip delete-only and go directly absent → write-only:
- absent servers don't add to index on insert, don't remove from index on delete
- write-only servers fully maintain the index
- If an absent server deletes row X (doesn't touch index), the index retains a stale entry for row X
- When write-only advances to public, queries using the index will return row X's stale pointer — data corruption

The delete-only state exists precisely to neutralize this: during the absent+delete-only window, no server ever adds an entry, so no stale entry can appear from a row that was later deleted by an absent server.

### Schema leases in practice

F1's leases are configurable; in production they use 10 seconds. Each state advance therefore costs 10 seconds of wall-clock time. A three-step index creation (absent → delete-only → write-only → public) takes ~30 seconds of protocol overhead, plus the backfill time (which can be hours for large tables, but runs concurrently with live traffic).

The metadata store (Spanner) is the authority. Servers check the descriptor on every lease renewal. If a server can't contact the metadata store before its lease expires, it refuses to serve queries — a safety-over-availability decision.

### Handling multiple concurrent schema changes

F1 allows multiple schema changes in flight simultaneously on different tables. Schema changes to the same table are serialized (each change must reach public before the next one starts). The descriptor version is a monotonic integer; a server running version N is safe if the global descriptor is at most at version N+1.

## Where it breaks

**Backfill latency is unbounded.** For a billion-row table, the backfill step can take hours. The index isn't usable until backfill completes, so long-running schema changes require careful planning. Tools like gh-ost and pt-online-schema-change address this with chunk-based backfill and throttling, but the core protocol doesn't change.

**The two-state maximum doesn't protect against logical errors.** The protocol prevents physical corruption (orphan entries, missing entries). It can't prevent an index that is physically correct but semantically wrong — for example, if your application logic silently writes inconsistent data during the transition period, the index will faithfully record that inconsistent data.

**Lease length is a latency/safety tradeoff.** Shorter leases mean faster state advancement but more metadata store load. Longer leases mean less load but slower schema changes. F1 uses 10 seconds; a global system with higher metadata access cost might use longer leases, proportionally increasing schema change time.

**Constraints (NOT NULL, CHECK) are harder than indexes.** An index is physically separate storage; you can add it without touching the main heap. Adding a NOT NULL constraint requires validating every existing row satisfies it — which is equivalent to a backfill but with a correctness check. If the validation fails mid-flight, the state machine rolls back. F1 handles this by running validation during write-only and aborting to absent on failure.

**Doesn't extend easily to cross-table constraints.** Foreign keys reference two tables. Adding a FK constraint requires coordinating the state machine across both tables simultaneously. F1 does support this but the safety argument becomes more complex — you need the delete-only/write-only sequence on both sides, and the backfill validates cross-table references.

## Why it works

The deeper principle is that **a distributed migration is safe if and only if you can construct a monotone path through schema states where every adjacent pair is semantically compatible**.

The word "compatible" has a precise meaning: for any pair of concurrent servers at states (N, N+1), the union of all writes they produce cannot create a database state that is inconsistent with respect to the schema at state N+1 after the migration completes.

This is the same invariant — in different vocabulary — that governs every distributed migration:

- **Blue-green deployments**: your application at version v1 and version v2 must share the same database schema and emit the same external events. If they don't, you need an intermediate schema state that both accept.
- **Kubernetes rolling updates**: old pod + new pod share the same ConfigMap, the same downstream API contract. Rolling update works safely only if the two pod versions are "adjacent-compatible".
- **API versioning**: you don't delete a field immediately; you deprecate it first (mark as write-ignored by new servers), then after all clients are on the new version, remove it from reads. This is delete-only → absent.
- **Database migration tools** (Alembic, Flyway, gh-ost, PlanetScale): they all implement a variant of this state machine. The "shadow table" approach in gh-ost is literally a write-only index (shadow = index on a different table), populated via backfill, then cut over to public.

The state machine insight unifies all of these: **the number of intermediate states you need is equal to the number of "incompatible operation pairs" you need to prevent**. For an index, there are two incompatible pairs to prevent (absent-deletes creating orphans, write-only-inserts missing from absent-deletes) — hence two intermediate states.

Algebraically: you're constructing a commutative migration where the transition graph has the property that `T(state_N) ∘ T(state_{N+1}) = T(state_{N+1}) ∘ T(state_N)` for all valid DML. The intermediate states are the scaffolding that makes this commutativity true.

This is the same scaffolding principle that ARIES uses for recovery (redo-then-undo rather than undo-then-redo, because redo is idempotent), and that Kafka uses for exactly-once delivery (write idempotency via PID+sequence before enabling cross-partition atomicity via 2PC).

## Going deeper

1. **Original paper (VLDB 2013)**: Ian Rae, Eric Rollins, Jeff Shute, Sukhdeep Sodhi, and Radek Vingralek. "Online, Asynchronous Schema Change in F1." *PVLDB* 6(11):1045–1056, 2013. The full correctness proof is in Section 4 — it's the most rewarding part of the paper and short enough to read in one sitting.

2. **CockroachDB's RFC implementing the same protocol**: [github.com/cockroachdb/cockroach/blob/master/docs/RFCS/20151014_online_schema_change.md](https://github.com/cockroachdb/cockroach/blob/master/docs/RFCS/20151014_online_schema_change.md) — a readable engineering document showing how a production database adapted the F1 protocol. Discusses the edge cases around lease timing and concurrent DDL.

3. **TiDB's Online DDL**: TiDB's blog has a detailed post on how they adapted the F1 algorithm with a distributed DDL owner (a single elected node) and how they accelerate backfill using parallel scan workers. Searching "TiDB Online DDL F1 schema change" finds the current version. Seeing how the abstract protocol gets implemented under real write pressure is the best test of understanding the core invariant.
