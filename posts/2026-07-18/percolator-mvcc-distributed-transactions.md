---
title: "Large-scale Incremental Processing Using Distributed Transactions and Notifications (Percolator)"
source: https://research.google/pubs/large-scale-incremental-processing-using-distributed-transactions-and-notifications/
author: Daniel Peng and Frank Dabek
company: Google
date_posted: 2010-10-04
date_digested: 2026-07-18
---

# Percolator: MVCC Snapshot Isolation Composed from a Row-Atomic Key-Value Store

## What's new to learn

1. **Shadow-column MVCC**: you can layer multi-row snapshot isolation on top of a single-row-atomic store by adding two extra "shadow" columns per data column — one to hold in-progress lock metadata, one to hold a committed version pointer — and never modifying the actual data in place.

2. **Client-coordinated 2PC with a primary lock as decision point**: two-phase commit does not require a separate coordinator process. The client acts as the coordinator; the "primary" cell of the first written key acts as the single decision point, so any client that encounters a stale lock can complete (or roll back) the transaction just by inspecting the primary.

3. **Timestamp Oracle**: a centralized monotonic counter that is the *sole* source of ordering in the system — contacting it twice per transaction (once for start timestamp, once for commit timestamp) is sufficient to make snapshot isolation fully correct across an arbitrary number of rows and tables.

## Prerequisites

- **Bigtable's data model**: rows keyed by a byte string; each row can have multiple columns; reads and writes of an entire row are atomic.
- **MVCC (multi-version concurrency control)**: the database keeps multiple timestamped versions of each cell; readers pick a snapshot by choosing the version at their start timestamp.
- **Snapshot isolation (SI)**: each transaction sees the committed state of the world as of its start timestamp, and aborts on write-write conflicts (two concurrent transactions writing the same cell).
- **Two-phase commit (2PC)**: a commit protocol split into "prewrite" (check for conflicts + acquire locks) and "commit" (mark as committed + release locks), with an abort if anything goes wrong in prewrite.

## The core idea

Bigtable gives you atomic reads and writes of a single row. Google needed to index the web incrementally: every crawled document must atomically update both its forward index *and* the inverted index entries that link back to it. If you write these two rows non-atomically, a crash leaves the system in an inconsistent state.

Percolator's insight: **you don't need a separate lock manager.** You can store the lock itself as a timestamped cell in the same key-value store. Similarly, you don't need the data to be updated in place — you can store a "version pointer" cell that records, at commit time, which data timestamp to read. The original data cell is written at the transaction's start timestamp, remains invisible to readers until the version pointer appears, and is cleaned up lazily.

The net result is that every Bigtable table in Percolator gets two extra shadow columns per data column, and the client library handles the prewrite/commit choreography. There is no separate coordinator process; the client *is* the coordinator, and a single designated "primary" lock cell doubles as the durable decision record.

## Mechanics

### The five-column layout

For each logical column `c` in a Percolator table, Bigtable physically stores three columns:

| Physical column | What's stored |
|---|---|
| `c:data` | The actual value, written at start timestamp `T_start` |
| `c:lock` | Opaque lock metadata; present only during in-progress transactions |
| `c:write` | Commit pointer: at timestamp `T_commit`, stores "data lives at `T_start`" |

Readers look at `c:write` to find the latest committed version, then read `c:data` at the timestamp that `c:write` points to. As long as `c:write` is absent, the data cell is invisible — it's a phantom write.

### Reading

```
get(key, T_start):
  // Check for stale locks from other transactions
  for lock in key:lock at timestamps [0, T_start]:
    backoff_or_cleanup(lock)
  // Find the latest committed version at or before our snapshot
  latest = max version in key:write with timestamp <= T_start
  if latest is None: return null
  // Follow the pointer to the actual data
  return key:data at timestamp latest.data_ts
```

A read aborts if it finds a lock covering its snapshot. This is the only blocking operation: live readers wait for in-progress writers, or roll them back if they appear expired.

### Prewrite phase (writing the locks and data)

One key among all the cells being written is designated the *primary*. All other cells are *secondaries*, each of whose lock cell stores a reference back to the primary's location.

```
prewrite(writes, primary, T_start):
  for (key, value) in writes:
    // Conflict check 1: write-write conflict
    if key:write has any version in [T_start, ∞): return ABORT
    // Conflict check 2: concurrent lock
    if key:lock has any version in [0, ∞): return ABORT
    // Write data (invisible until commit)
    key:data @ T_start = value
    // Write lock; secondary locks reference the primary's location
    if key == primary:
      key:lock @ T_start = {is_primary}
    else:
      key:lock @ T_start = {primary_location: primary}
```

Both conflict checks and the two writes to a single row happen as a single Bigtable conditional mutation (a row-level compare-and-swap), which is the only atomicity Percolator needs from Bigtable.

### Commit phase

```
commit(writes, primary, T_start):
  T_commit = oracle.get_timestamp()
  // Commit the primary (single atomic Bigtable row operation)
  if primary:lock @ T_start is absent: return ABORT  // rolled back by another client
  delete primary:lock @ T_start
  primary:write @ T_commit = {data_ts: T_start}
  // Secondaries can be written lazily (even after client death)
  for (key, _) in writes \ {primary}:
    delete key:lock @ T_start
    key:write @ T_commit = {data_ts: T_start}
```

The critical observation: writing `primary:write` and deleting `primary:lock` happen in a single atomic Bigtable row operation. The moment that succeeds, the transaction is durably committed. All secondary cleanup is idempotent and can be completed by *any* client that encounters the stale lock later.

### Lazy lock cleanup (transaction recovery)

When any reader finds a lock, it checks whether the transaction is still live:

```
resolve(lock):
  primary_lock = fetch lock at lock.primary_location
  if primary_lock absent:
    // Primary is committed (write pointer exists) or rolled back
    primary_write = fetch primary:write at any timestamp
    if primary_write exists:
      roll_forward(all_secondaries, primary_write.commit_ts)  // complete the commit
    else:
      roll_back(all_secondaries)  // erase all data and locks
  else:
    if lock is expired:
      roll_back(all_secondaries)  // coordinator died mid-prewrite
    else:
      wait or retry
```

The primary cell is the single decision point. Any client can inspect it to determine whether to commit or abort the stale transaction. This eliminates the coordinator single-point-of-failure problem: if the original client crashes after committing the primary, any other client can complete the secondary cleanup.

### Timestamp Oracle

The Oracle is a single highly available service that hands out monotonically increasing 64-bit timestamps. Performance optimization: the Oracle pre-allocates ranges of timestamps (writing the high watermark to stable storage), then serves individual timestamps from memory without disk I/O. A single Oracle can sustain millions of requests per second.

Every transaction contacts the Oracle exactly twice: once at start (to get `T_start`) and once at commit (to get `T_commit` > `T_start`). The guarantee `T_commit > T_start` is sufficient for snapshot isolation: your commit is guaranteed to be ordered *after* your reads.

### Observers

Percolator adds a reactive layer on top of transactions: *observers* are user-defined functions that fire when specified columns change. The indexing pipeline registers observers on document content columns; when a crawler writes a new version, observers are triggered to update forward-index and backlink tables. Observers run inside Percolator transactions, so observer-triggered writes also get ACID guarantees.

## Where it breaks

**Timestamp Oracle is a global bottleneck.** Every transaction in the system contacts the Oracle twice. Google mitigates this with in-memory batching, but it's still a single logical bottleneck — an argument that drove the design of Spanner (which uses bounded physical clock intervals via TrueTime to avoid needing a global Oracle).

**Two round trips to Oracle add latency.** Each transaction has at least two Oracle round trips, even in the best case. Percolator was designed for batch-like workloads (web indexing, not OLTP), where tens-of-millisecond latency is acceptable.

**Locks are still pessimistic during prewrite.** Percolator is *optimistic* in the sense that reads don't acquire locks, but prewrite is fully pessimistic: if two transactions both try to prewrite the same key, one will abort. High contention on hot rows is not solved.

**No cross-datacenter support.** The Oracle requires linearizable access; deploying across regions would require a strongly consistent Oracle itself, which brings back the latency problem at planetary scale. Spanner's TrueTime approach solves this by bounding clock uncertainty rather than enforcing a global total order.

**Lazy cleanup can delay reads.** A reader that encounters an expired lock from a crashed client has to do the cleanup work (inspecting the primary, completing or rolling back secondaries) before it can return. Under high crash rates, this adds significant read latency.

## Why it works

**The primary cell as a distributed commit flag.** The entire correctness of Percolator hinges on the fact that atomic row operations in Bigtable make "check lock + delete lock + write version pointer" a single uninterruptible step on the primary row. Once that step succeeds, the transaction is committed — not when the last secondary is cleaned up, but right at that moment. This is the "you don't need a coordinator" trick: instead of a coordinator deciding "committed/aborted," the primary cell *is* the decision, stored durably in the same system as the data.

This is the same insight behind WAL-based crash recovery (ARIES, covered elsewhere in this archive): you commit by appending to a log, not by updating pages; the log record is the durable decision. Percolator's primary lock cell is that log record, embedded in the data store.

**MVCC makes reads non-blocking.** Because data is written at `T_start` and made visible only when the `write` pointer appears at `T_commit`, readers at any snapshot between 0 and `T_start - 1` are completely unaffected by an in-progress writer. There is no lock taken during the read phase at all. This is why Percolator can achieve high read throughput even under heavy write concurrency.

**Monotonic timestamps are enough for SI.** The Timestamp Oracle does not need to know anything about what keys each transaction reads or writes. It just needs to produce monotonically increasing integers. Snapshot isolation's correctness follows from the rule `T_commit > T_start` and from the prewrite conflict checks — not from any global knowledge at the Oracle. The Oracle is *ordering infrastructure*, not *transaction management*.

**Lazy cleanup is idempotent by design.** Every step in the cleanup procedure is idempotent: checking the primary, writing a version pointer, and deleting a lock are all re-runnable without side effects. This is the same property that makes 2PC recovery correct: you can replay the commit phase as many times as needed.

The deeper principle: **transaction state is data.** By storing locks and version pointers in the same key-value store as the application data, Percolator achieves durability of transaction state for free — the same replication that protects application data also protects the in-progress transaction's lock cells. There is no separate "transaction log" with separate durability concerns.

## Going deeper

1. **Original paper**: Peng & Dabek, "Large-scale Incremental Processing Using Distributed Transactions and Notifications," OSDI 2010. Available at [research.google/pubs](https://research.google/pubs/large-scale-incremental-processing-using-distributed-transactions-and-notifications/). Section 2 ("Design") contains the full pseudocode for the transaction protocol.

2. **TiKV's Percolator implementation**: TiKV is a production distributed key-value store that implements the Percolator protocol over RocksDB tablets. The [TiKV deep-dive](https://tikv.org/deep-dive/distributed-transaction/percolator/) walks through the same protocol with annotated pseudocode and explains how TiKV handles clock skew, lock resolution, and the Timestamp Oracle (called "PD" — Placement Driver — in their system).

3. **Percolator vs. Spanner** (YugaByte blog): Sid Choudhury's post ["Implementing Distributed Transactions the Google Way"](https://www.yugabyte.com/blog/implementing-distributed-transactions-the-google-way-percolator-vs-spanner/) contrasts Percolator's Timestamp Oracle with Spanner's TrueTime. The central comparison: Percolator avoids multi-region deployment by having a single Oracle; Spanner eliminates the Oracle by replacing "one true timestamp" with "timestamp + uncertainty interval" and paying a commit-wait latency to let uncertainty collapse.
