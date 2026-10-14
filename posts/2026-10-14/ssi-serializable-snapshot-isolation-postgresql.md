---
title: "Serializable Snapshot Isolation in PostgreSQL"
source: https://arxiv.org/pdf/1208.4179
author: Dan R. K. Ports, Kevin Grittner
company: MIT CSAIL / PostgreSQL Project
date_posted: 2012-08-21
date_digested: 2026-10-14
---

# Serializable Snapshot Isolation in PostgreSQL

## What's new to learn

1. **Conflict graphs and the three conflict types** — Every concurrent execution history corresponds to a directed graph over transactions. A history is serializable if and only if its conflict graph is acyclic. The three edge types are: wr (dirty read: T1 wrote X, T2 read X), ww (dirty write: T1 and T2 both wrote X), and rw (anti-dependency: T1 read X, T2 later wrote X). Snapshot Isolation (SI) blocks wr and ww via MVCC snapshots — but rw edges pass through invisibly.

2. **The dangerous structure theorem** — Fekete et al. (2005) proved that every non-serializable SI execution must contain a *dangerous structure*: three transactions T_in →^{rw} T_pivot →^{rw} T_out where T_in and T_out overlap in time. Two consecutive rw-anti-dependency edges are the necessary and sufficient condition for a non-serializable execution under SI. SSI is therefore complete: catch every dangerous structure, prevent every anomaly.

3. **SIREAD locks** — A new lock mode that records "this transaction read these rows" without ever blocking concurrent writes. Unlike S or X locks, a SIREAD lock is invisible to the writer — the writer just records the rw edge T_reader →^{rw} T_writer in a conflict table. This is the key mechanism: serializability checking without any blocking.

## Prerequisites

- What Snapshot Isolation (SI) is and why it permits write skew; the archive's 2026-05-18 entry on "Snapshot Isolation vs Serializability" is the direct prerequisite.
- MVCC basics: rows have xmin/xmax version stamps; each transaction reads a snapshot at start time. The 2026-09-26 "How Postgres Makes Transactions Atomic" covers this.
- What a conflict graph is: nodes = transactions, directed edges = conflicts. A cycle means no serial order is consistent with the execution.

## The core idea

The existing archive entry on SI explains *why* write skew exists: SI's OCC commit rule checks write-write conflicts but not read-write conflicts. The engineers who built PostgreSQL 9.1's serializable mode faced a concrete question: how do you detect the read-write conflicts you need to prevent anomalies without killing performance?

The naive answer — track full read sets and abort whenever any read was overwritten — is prohibitively expensive: reads outnumber writes by 10–100× and tracking them would require a global read-set registry under constant contention.

The Fekete theorem provides a better answer. You don't need to catch *every* rw conflict — you only need to catch pairs of consecutive ones. The theorem says:

> Every non-serializable SI execution contains three transactions T_in, T_pivot, T_out where T_in has an rw-anti-dependency on T_pivot and T_pivot has an rw-anti-dependency on T_out, and T_in and T_out overlap in time.

This "dangerous structure" is necessary — if you kill it, you kill all anomalies. SSI's strategy is:

1. Track rw edges using non-blocking SIREAD locks.
2. After every write, check if a dangerous structure has formed.
3. If one has formed, abort one transaction from the pattern — typically the pivot.
4. Everything else runs exactly as it would under SI.

The result: serializability that is only slightly slower than SI and dramatically faster than strict two-phase locking (S2PL), especially on read-heavy workloads where S2PL's read locks cause cascading waits.

## Mechanics

### SIREAD locks: tracking reads without blocking

When transaction T reads tuple X (or a range of tuples via an index scan), the system acquires a `SIREAD` lock on X. Unlike a normal `S` lock, a SIREAD lock:
- Does **not** conflict with `X` locks from other transactions — writers are never blocked.
- Is recorded in a separate predicate lock manager, not the main lock table.
- Persists until T commits or aborts (and for read-only transactions, sometimes longer).

When transaction T' *writes* to X, the system scans the SIREAD lock table for any lock on X held by a transaction T ≠ T'. If found, it records an rw-anti-dependency edge T →^{rw} T' in a small conflict table. No blocking — T' just annotates the graph.

### Dangerous structure detection

After recording T →^{rw} T', the system immediately checks for a dangerous structure by looking for adjacent rw edges:

```
Check outbound edges from T:  is there T'' →^{rw} T?
  → yields structure T'' →^{rw} T →^{rw} T' (T is the pivot)

Check inbound edges to T':   is there T' →^{rw} T'''?
  → yields structure T →^{rw} T' →^{rw} T''' (T' is the pivot)
```

A dangerous structure is confirmed when the two "endpoints" (T'' and T' in the first case, T and T''' in the second) overlap in time — i.e., both are still running or one committed after the other started. If they don't overlap, no cycle is possible.

When a dangerous structure is confirmed, one transaction must be aborted. PostgreSQL's heuristic prefers aborting the pivot, but the system can abort any transaction in the structure. The aborted transaction returns a serialization failure error; the application retries.

### The doctor-on-call example revisited

Using the write-skew example from the archive:

```
T_alice: SIREAD(on_call_count=2)  →  write alice=off_call  [commit]
T_bob:   SIREAD(on_call_count=2)  →  write bob=off_call    [commit]
```

Under SI: both commit, leaving 0 doctors on call.

Under SSI:
1. T_alice reads `on_call_count` → acquires SIREAD(alice, on_call_count).
2. T_bob reads `on_call_count` → acquires SIREAD(bob, on_call_count).
3. T_alice writes `alice=off_call`. No SIREAD on alice; no rw edge.
4. T_alice writes `on_call_count=1`. The system finds SIREAD(bob, on_call_count) → records `T_bob →^{rw} T_alice`.
5. T_alice commits. Checks for dangerous structure: no inbound rw edge to T_alice. No structure yet.
6. T_bob writes `bob=off_call`. No SIREAD on bob; no rw edge.
7. T_bob writes `on_call_count=0`. Finds SIREAD(alice, on_call_count) → records `T_alice →^{rw} T_bob`.
8. Checks: T_alice →^{rw} T_bob, and T_bob →^{rw} T_alice (from step 4). Dangerous structure detected. T_bob is aborted.

Result: only one doctor goes off-call. The anomaly is prevented.

Note: T_alice already committed in step 5 — SSI sometimes aborts the transaction that tries to commit *second*, which is why the error message says "could not serialize access due to concurrent update."

### Lock granularity escalation and memory management

A key practical constraint: SIREAD locks must not consume unbounded memory. If a transaction reads 10 million tuples, 10 million SIREAD locks is untenable.

The system uses **lock escalation**: when a transaction holds SIREAD locks on more than a threshold of tuples in a page, it collapses them into a single SIREAD lock on the entire page. Similarly, many page-level locks escalate to a single relation-level lock. Escalation is conservative — it may detect false positives (non-existent rw edges) and cause spurious aborts, but it is never incorrect. The threshold is tunable (`max_pred_locks_per_transaction`).

### Read-only transaction optimization

A read-only transaction (ROT) cannot have *outbound* rw edges — it never writes, so no writer can record a conflict on its reads from the ROT's perspective. In a dangerous structure T_in →^{rw} T_pivot →^{rw} T_out, an ROT can only appear as T_out.

The key insight: if a ROT reads only data committed before any currently-running read-write transaction started, then no running transaction can create a new rw edge pointing at the ROT's reads (those records are already committed and stable). Such a snapshot is called a **safe snapshot**.

A ROT that reads a safe snapshot can skip all SIREAD lock acquisition entirely — it cannot be the T_out in any dangerous structure, so it never needs tracking. PostgreSQL detects safe snapshots automatically: if the ROT's snapshot was taken before all currently-active read-write transactions started, it's safe.

This optimization dramatically reduces overhead for read-heavy workloads (analytics, reporting queries) which are exactly the cases where SSI overhead would otherwise be most painful.

### Implementation artifacts

- **`src/backend/storage/lmgr/predicate.c`**: ~5,000 lines, the entire SSI lock manager. The `CheckForSerializableConflictOut` and `CheckForSerializableConflictIn` functions handle the dangerous-structure detection.
- **`src/backend/storage/lmgr/README-SSI`**: A detailed design document in the PostgreSQL source tree explaining the algorithm, edge cases, and implementation choices in prose. Worth reading as a companion to the paper.
- PostgreSQL introduced SSI in version 9.1 (September 2011). It was the first production database to implement the Cahill/Fekete algorithm.

## Where it breaks

**False positives are unavoidable.** SSI is conservative: lock escalation and the dangerous-structure check can abort transactions that *would have* been serializable. Under high contention or with long-running transactions, false positive rates can be significant.

**Long-running read transactions hold SIREAD locks for their entire duration.** An OLAP query scanning a large table keeps SIREAD locks active for minutes. This increases both memory pressure (forcing escalation → false positives) and the probability of rw conflicts with ongoing writers.

**Write-heavy workloads suffer disproportionately.** If many transactions write to the same "hot" rows, each write triggers a scan of the SIREAD lock table for that row. The constant conflict detection becomes a bottleneck. S2PL (strict two-phase locking) queues writers; SSI burns CPU on detection and aborts.

**Retries are necessary but not free.** Every serialization failure requires the application to catch the error and retry. If the application uses connection pooling and retry logic isn't carefully designed, retries can pile up and worsen congestion.

**Not all cycles are caught eagerly.** A dangerous structure is only detected when the second rw edge is created. If both edges were created by already-committed transactions, the system may detect the cycle only at commit time of the remaining transaction — or miss it if insufficient history is retained.

## Why it works

The deeper principle: **serializability is cycle detection in the conflict graph, and SSI instruments only the edges that SI misses**.

SI already prevents wr edges (MVCC snapshots prevent dirty reads) and ww edges (OCC write-write conflict detection). The only thing SI doesn't track is rw edges — anti-dependencies — because they don't involve a direct "concurrent modification" but rather the invisible situation of "you read what I then changed."

Fekete's theorem tells us that not *all* rw edges matter: only consecutive pairs. This is the same insight as "you don't need to solve the full NP-hard cycle detection problem for all graph types — for this specific graph structure, you only need to watch for the one subgraph shape that serializability anomalies require." The theorem converts an intractable problem (check all possible serialization orders) into a tractable one (scan for one graph motif).

This is structurally identical to how SSI relates to 2PL:
- S2PL prevents cycles by *blocking* any write that would create an rw edge.
- SSI prevents cycles by *aborting* one transaction when a dangerous structure forms.
- Blocking is pessimistic (no false positives but high latency); aborting is optimistic (false positives but low blocking latency). The same pessimistic-vs-optimistic tradeoff that distinguishes 2PL from OCC generally.

The "X is just Y" insight: **SSI is optimistic concurrency control applied to cycle detection. It runs transactions speculatively under SI (the optimistic assumption), tracks rw edges lazily (the minimal instrument needed), and aborts only when the specific graph pattern that makes speculation unsafe appears — exactly as OCC aborts only when the write set overlaps.**

This is the same design pattern as:
- CPU branch predictors (execute speculatively, squash on mispredict)
- Optimistic locking in ORMs (read, write, check version, retry on conflict)
- Multi-version GC (let transactions run, defer cleanup until safe to reclaim)

The theorem does the work: it tells you *what* to check. The implementation does the rest.

## Going deeper

1. **Cahill, Röhm, Fekete — "Serializable Isolation for Snapshot Databases" (SIGMOD 2008)** — The theory paper that proves the dangerous-structure theorem and proposes SSI as a practical algorithm. Shorter and more rigorous than the VLDB 2012 engineering paper; the proof is in Section 4.

2. **`src/backend/storage/lmgr/README-SSI`** (in the PostgreSQL source tree, available at github.com/postgres/postgres) — The implementors' design document. Explains each data structure, the memory-management strategy, and subtle edge cases like "what happens when a transaction commits before we detect the structure?"

3. **Andy Pavlo — "CMU 15-721 Lecture 17: MVCC Protocols"** — A graduate database course lecture that walks through the conflict graph formalism, explains the connection between Herbrand semantics and rw-anti-dependencies, and compares SSI against timestamp ordering and S2PL. The slides and video are publicly available.
