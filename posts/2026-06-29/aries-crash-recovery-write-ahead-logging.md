---
title: "ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging"
source: https://blog.acolyer.org/2016/01/08/aries/
author: C. Mohan, Don Haderle, Bruce Lindsay, Hamid Pirahesh, Peter Schwarz
company: IBM Research
date_posted: 1992-01-01
date_digested: 2026-06-29
---

# ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging

## What's new to learn

1. **The STEAL/NO-FORCE buffer policy** — a 2×2 design space describing *when* dirty pages may (or must not) reach disk relative to commit; ARIES sits at the corner that maximizes both buffer-pool freedom and commit throughput, at the cost of needing a full undo/redo log.

2. **Log Sequence Numbers (LSNs) as causal timestamps** — every log record and every on-disk page share a monotonic counter; comparing a page's `pageLSN` to a log record's LSN tells you whether that update is already durable, making recovery idempotent.

3. **"Repeat history regardless"** — ARIES's non-obvious redo phase replays *all* log records (including uncommitted transactions) to reconstruct the exact pre-crash state before any rollback happens; this single principle eliminates entire classes of correctness edge cases.

## Prerequisites

- **Database buffer pool**: an in-memory cache of disk pages; the buffer manager decides when to evict (flush) pages to disk.
- **ACID transactions**: atomic, consistent, isolated, durable; here durability is the property we're solving for.
- **What a disk write costs**: sequential log writes are ~100× cheaper than random page writes; that ratio drives every design decision in ARIES.

## The core idea

A crashed database has a problem: at the moment of failure, some committed transactions may not have reached disk (lost if we just restart), and some uncommitted transactions may have partially reached disk (corrupting the database if we don't undo them). Naively, you could avoid this by forcing every dirty page to disk on commit and never letting dirty pages escape before commit — but that makes commits expensive (force all pages on commit) and holds buffer frames hostage for the entire transaction.

ARIES makes a different bargain: **relax all restrictions on when pages hit disk, and instead use a log to record enough information to fix anything that went wrong.** The Write-Ahead Log (WAL) protocol is the one rule that must hold: *before a dirty page is written to disk, all log records that describe changes to that page must be in the log on disk first*. This single invariant is what lets you guarantee recoverability.

At crash time, the log is a complete, ordered record of every intended change. Three phases (Analysis, Redo, Undo) use that record to restore the database to a consistent state.

## Mechanics

### The STEAL/NO-FORCE design space

There are two orthogonal decisions a buffer manager must make:

| | STEAL | NO-STEAL |
|---|---|---|
| **FORCE** | Write dirty pages whenever; force all at commit | Never evict before commit; force all at commit |
| **NO-FORCE** | Write dirty pages whenever; don't force at commit | Never evict before commit; don't force at commit |

- **STEAL**: The buffer manager *may* evict a dirty page to disk before the modifying transaction commits, freeing that frame for another transaction. Requires undo (the evicted data belongs to an uncommitted txn and might need to be rolled back).
- **NO-STEAL**: The buffer manager holds dirty frames until commit. No undo needed, but frames stay locked for the transaction's lifetime — catastrophic for long transactions.
- **FORCE**: On commit, flush every dirty page belonging to that transaction to disk. No redo needed, but commit now triggers many random disk writes.
- **NO-FORCE**: On commit, only flush the log (sequential writes). Dirty pages reach disk lazily. Redo needed on crash.

ARIES uses **STEAL + NO-FORCE**: the worst case for the log (both undo and redo records needed), but the best case for runtime performance. Commit is cheap (just flush log records, which are sequential), and the buffer pool is always free to evict any frame it needs.

### Log records and LSNs

Every update produces a log record written to the tail of the log. Each record contains:

- **LSN** (Log Sequence Number): a monotonically increasing identifier, often the byte offset of the record in the log file.
- **prevLSN**: the LSN of the previous log record from the *same* transaction — creating a singly-linked list backward through each transaction's history.
- **transID**: which transaction produced this record.
- **pageID**: which page was modified.
- **undo info**: enough data to reverse the update (e.g., the old value).
- **redo info**: enough data to re-apply the update (e.g., the new value).

Every page on disk carries a **pageLSN** field in its header: the LSN of the last log record that modified that page. This is the key invariant: `page.pageLSN` tells you exactly how far into the log that page has been applied.

### The WAL protocol in two rules

1. **Before writing a dirty page to disk**: flush the log up through that page's `pageLSN`. (So the undo info for that page's changes is already durable.)
2. **Before committing a transaction**: flush all log records for that transaction. (A COMMIT log record must hit disk before the commit acknowledgment returns to the caller.)

That's it. These two rules are sufficient to guarantee recoverability.

### Checkpoints

Periodically ARIES writes a **checkpoint** to the log, recording:
- The **dirty page table (DPT)**: every page currently dirty in the buffer pool and the LSN of the first log record that made it dirty (`recLSN`).
- The **active transaction table (ATT)**: every in-progress transaction and the LSN of its most recent log record.

ARIES uses "fuzzy checkpoints" — the DPT and ATT snapshots are written to the log, but dirty pages are not necessarily flushed (no stall on checkpoint). The checkpoint provides a starting point for recovery without requiring a full log scan from the beginning of time.

### Three-phase recovery

**Phase 1 — Analysis**: Scan forward from the last checkpoint's log record to the end of the log.
- Reconstruct the DPT and ATT as they were at the time of crash.
- Any page in the DPT has a `recLSN` = the LSN that first dirtied it. The minimum `recLSN` across the entire DPT is the **redo start LSN** — the earliest log record that might not be reflected on disk.
- Any transaction in the ATT at the end of this scan was active when the crash happened and must be undone.

**Phase 2 — Redo**: Scan forward from the redo start LSN to the end of the log, applying *every* update log record unconditionally — including those from uncommitted transactions.

For each log record:
1. If the page is not in the DPT, skip (already on disk and no changes needed).
2. If `recLSN > log_record.LSN`, skip (the first dirty LSN for this page is later — this record was already on disk before the crash).
3. Fetch the page from disk. If `page.pageLSN >= log_record.LSN`, skip (already applied).
4. Otherwise, apply the redo. Update `page.pageLSN`.

The mantra is "**repeat history regardless**": ARIES doesn't try to be clever about which records to replay. By restoring the database to its exact pre-crash state (including partial work from uncommitted transactions), it simplifies every subsequent step.

**Phase 3 — Undo**: Process the active transaction table in reverse LSN order (largest LSN first), rolling back each uncommitted transaction one log record at a time.

For each undo:
1. Apply the undo operation to the page (restore old value).
2. Write a **Compensation Log Record (CLR)** to the log. A CLR says "I have undone log record X." Its `nextUndoLSN` pointer skips directly to the record before X in that transaction's history, bypassing already-undone records.
3. Move to the previous record in that transaction's chain (`prevLSN`).

CLRs are the safety net: if the database crashes *again during undo*, the CLRs already written ensure we don't redo-and-then-undo the same operations twice. CLRs are themselves redo-only records; they're never undone.

### End-to-end example

Say transaction T1 modifies page P at LSN 100, and crashes before committing. P gets evicted to disk (STEAL) with `pageLSN=100`. Crash. On recovery:

- Analysis finds T1 in the ATT (uncommitted), P in the DPT with `recLSN=100`.
- Redo fetches P from disk; `page.pageLSN=100 >= 100`, so this record is skipped (already on disk). ✓
- Undo finds T1 is uncommitted. Applies the undo of LSN 100, writes CLR(100) to log. T1 is marked complete.
- P now contains the pre-T1 value, durably.

Now say T2 also modified P (LSN 200) and committed, but P didn't make it to disk before the crash:

- Redo sees P in DPT with `recLSN=200`. Fetches P; `page.pageLSN < 200`. Applies LSN 200. P now reflects T2's committed change. ✓
- Undo ignores T2 (it was committed, not in ATT).

## Where it breaks

**Recovery time scales with oldest active transaction.** The undo phase must replay from the oldest uncommitted transaction's first log record. A long-running transaction (hours of work) forces a long recovery window. PostgreSQL and InnoDB both have background "vacuum/purge" threads specifically to prevent this. Many modern databases cap transaction runtime or force periodic partial commits.

**Redo scans can be enormous.** If the redo start LSN is far in the past (low-water mark of the DPT), the redo phase re-applies many records. Aggressive checkpointing reduces this at the cost of increased checkpoint I/O. This is a continuous tuning parameter.

**Fine-grained locking assumptions.** ARIES was designed to coexist with record-level locking (not page locking), which is why it needs CLRs and careful partial-rollback support. Simpler recovery schemes (like SQLite's WAL or many key-value stores) avoid this by using page-level or segment-level locks, but pay in concurrency.

**Log volume.** Writing before-images (undo) *and* after-images (redo) doubles the log I/O compared to redo-only schemes. Systems like Postgres use "logical logging" for undo to reduce this, but at the cost of more complex recovery logic.

**Not directly portable to distributed systems.** ARIES assumes a single-node transaction manager with a single log. Distributing it (D-ARIES) requires cross-node log coordination, which is exactly what Raft/Paxos provide — but the two systems must be carefully integrated to avoid double-committing or double-aborting.

## Why it works

The deepest insight in ARIES is that **the log is a first-class data structure**, not an afterthought. Once you commit to writing undo *and* redo information before any page touches disk, you've bought yourself a time machine: you can replay forward to any committed state, or reverse to any prior consistent state.

**LSNs are just Lamport timestamps applied to a single-node log.** The same monotonic-counter-as-causal-marker pattern appears in:
- Kafka offsets (consumer can replay from any point)
- Raft log indices (commit index = last applied LSN)
- MVCC transaction IDs in PostgreSQL (visibility determined by comparing snapshot txid to a row's xmin/xmax)
- The `pageLSN` on each page is exactly the same as the "version" in optimistic concurrency control

**STEAL/NO-FORCE is "pay with CPU, not with I/O at the critical path."** The commit path does one sequential write (the log flush). Random page writes happen lazily in the background, amortized over many transactions, at full sequential-write speed (pages are written in bulk during checkpoint flushes). This is the same trade-off as LSM trees (convert random writes to sequential) and B-tree WAL (defer random writes).

**"Repeat history regardless" (redo all, then undo some) is correctness by construction.** The alternative — selectively replaying only committed transactions — requires knowing which transactions were committed *at crash time*, which is unknowable without the Analysis phase. By replaying everything first, ARIES avoids reasoning about partial states. This is exactly why event sourcing systems replay *all* events to rebuild state before applying business rules: partial state is harder to reason about than full state.

**CLRs make undo idempotent.** This is the same design principle as idempotent message processing (seen in Kafka "exactly-once," Zanzibar's zookie tokens, and distributed sagas): once you've done something, record a durable marker so that a re-run skips it. A CLR is just an undo receipt.

## Going deeper

1. **The original ARIES paper** — C. Mohan et al., "ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging," *ACM TODS* 1992. The paper is dense but the first 20 pages cover the core ideas; skip directly to Section 5 (the three phases) if short on time.

2. **CMU 15-445 "Database Crash Recovery" lecture notes** — Andy Pavlo's annotated slides at cs.cmu.edu/15-445 cover ARIES with worked examples and diagrams that make the phase interactions concrete. The Fall 2023 lecture is particularly good.

3. **"PostgreSQL WAL Internals"** (PostgreSQL documentation, Chapter 29) — see how ARIES maps onto a real codebase: `pg_lsn`, `xl_prev` (prevLSN), `ReadRecord`/`ApplyWAL` (redo), and `heap_xlog_*` functions. Cross-referencing the ARIES paper with this code makes both click.
