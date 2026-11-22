---
title: "Beautiful Concurrency: Software Transactional Memory in Haskell"
source: https://www.microsoft.com/en-us/research/publication/beautiful-concurrency/
author: Simon Peyton Jones
company: Microsoft Research
date_posted: 2007
date_digested: 2026-11-22
---

# Beautiful Concurrency: Software Transactional Memory in Haskell

## What's new to learn

1. **Lock non-composability**: Two correct lock-based functions cannot be safely combined into a larger atomic operation without exposing their internal lock ordering—a hidden invariant that breaks modularity.
2. **STM composability via transaction logs**: `atomically` runs a block speculatively on a private write log and commits only if the read set was not modified by another thread—the same OCC pattern as databases, but applied to in-process shared memory.
3. **`retry` and `orElse` as first-class synchronization**: `retry` blocks until any TVar read in the current transaction changes (the runtime registers interest automatically), and `orElse A B` tries A, falls back to B if A blocks, and blocks until either can make progress.

## Prerequisites

- Mutexes, semaphores, and condition variables in any language
- Basic optimistic concurrency control (OCC): execute speculatively, validate at commit, abort on conflict (covered in the Snapshot Isolation post)
- Haskell syntax helps but is not required—the ideas translate directly to any language

## The core idea

The fundamental problem with locks is **composition**. Consider a bank with two accounts, each protected by its own lock:

```haskell
withdraw :: Account -> Int -> IO ()
withdraw acc amount = withLock (lock acc) $ do
  bal <- readIORef (balance acc)
  writeIORef (balance acc) (bal - amount)

deposit :: Account -> Int -> IO ()
deposit acc amount = withLock (lock acc) $ do
  bal <- readIORef (balance acc)
  writeIORef (balance acc) (bal + amount)
```

Both functions are correct in isolation. But combining them into an atomic transfer is *not* simply `withdraw a n >> deposit b n`—that leaves a window between the two operations where another thread sees a world with the money subtracted but not yet added. The fix requires holding both locks simultaneously, but now the caller must know the internal lock of each account to get the ordering right (taking `lock a` before `lock b` everywhere), or risk deadlock.

**The fundamental rule: locks cannot be composed without breaking abstraction.** A caller of `withdraw` and `deposit` must know their internal implementation details just to safely sequence them.

STM fixes this. The same code in Haskell STM:

```haskell
withdraw :: Account -> Int -> STM ()
withdraw acc amount = do
  bal <- readTVar (balance acc)
  writeTVar (balance acc) (bal - amount)

deposit :: Account -> Int -> STM ()
deposit acc amount = do
  bal <- readTVar (balance acc)
  writeTVar (balance acc) (bal + amount)

transfer :: Account -> Account -> Int -> IO ()
transfer from to amount = atomically $ do
  withdraw from amount
  deposit to amount
```

`withdraw` and `deposit` live in the `STM` monad, not `IO`. That type distinction means they cannot perform arbitrary effects—they can only read and write `TVar`s. The caller wraps them in `atomically`, which executes the entire block as one atomic transaction. No lock ordering is needed. No deadlock is possible. The abstraction boundary is preserved.

## Mechanics

### The transaction log

When `atomically action` starts, the runtime allocates a fresh **transaction log** (a hashtable from `TVar` address to `(expected_value, written_value)`).

- `readTVar v`: check if `v` is in the log; if yes, return the written value; if no, read from memory, record `(current_value, current_value)` in the log, and return the value.
- `writeTVar v x`: write `(existing_expected, x)` into the log entry for `v`. Memory is not touched.

At the end of the `STM` block, the runtime attempts to **commit**:
1. Lock the heap briefly (or use hardware CAS in modern implementations).
2. For every `(v, expected)` in the read set: compare `expected` with the actual current value in memory. If any mismatch, **abort**: discard the log, release the lock, and restart the entire block from scratch.
3. If all match, **flush**: copy every written value from the log to memory atomically. Release the lock.

This is exactly OCC: read a snapshot, compute locally, validate the snapshot at commit. The STM monad type enforces that *only* TVar reads and writes are permitted inside a transaction—no I/O, no allocation of non-transactional state—so aborting and retrying is always safe.

### `retry`: blocking without polling

The `retry` primitive aborts the current transaction and **blocks until progress is possible**:

```haskell
limitedWithdraw :: Account -> Int -> STM ()
limitedWithdraw acc amount = do
  bal <- readTVar (balance acc)
  if bal < amount
    then retry       -- not enough funds; block
    else writeTVar (balance acc) (bal - amount)
```

When `retry` is called, the runtime records which TVars were read in this transaction (the read set). It registers the current thread as a waiter on all of them. The thread blocks. When any other thread commits a transaction that *writes* a TVar in this read set, the blocked thread is woken and retries the entire transaction from scratch.

No condition variable. No manual registration. The runtime infers "what I need to wait for" directly from "what I read." This is non-trivial: the predicate may depend on multiple TVars in complex ways, and the runtime only needs to know "re-run me when anything in the read set changes."

### `orElse`: composing blocking operations

`orElse A B` tries A in a **nested transaction log**. If A succeeds, its writes are merged into the parent log. If A calls `retry`, the nested log is discarded and B is tried in a fresh nested log. If B also retries, the outer transaction retries, blocking on the union of A's and B's read sets.

```haskell
-- Santa waits for either all 9 reindeer OR any 3 elves
waitForReindeer :: STM Group
waitForReindeer = do
  n <- readTVar numReindeer
  if n < 9 then retry else return Reindeer

waitForElves :: STM Group
waitForElves = do
  n <- readTVar numElves
  if n < 3 then retry else return Elves

waitForHelpers :: STM Group
waitForHelpers = waitForReindeer `orElse` waitForElves
```

This is the **Santa Claus problem**: neither `waitForReindeer` nor `waitForElves` knows about the other, yet `orElse` combines them into a correct non-deterministic choice. The runtime will wake the thread when *either* `numReindeer` or `numElves` changes, and the first branch that doesn't retry wins. No condition variable combination. No manual wakeup fanout. Pure composition.

The key insight: `orElse` turns blocking into a *value* you can pass around, combine, and abstract over—the same relationship as `>>=` turning I/O operations into composable values.

### The commit cost

The brief heap lock taken during commit serializes all concurrent commits across the entire heap. GHC's implementation avoids this for the common uncontested case by using CAS operations on individual TVars, but under high contention multiple committing threads still serialize. This is the core scalability limit: a single hot TVar is a global serialization point.

## Where it breaks

**I/O inside transactions is impossible by construction.** The STM monad type enforces this: you cannot call `readFile`, `sendPacket`, or any other IO action from within `atomically`. This is correct—if a transaction aborts, any real-world effects already performed cannot be rolled back. The type system catches this at compile time, not at runtime. In languages without this enforcement (like Clojure), accidentally performing I/O inside a transaction causes subtle bugs.

**Livelock under high contention.** If many threads contend on the same TVar, all will commit, see conflicts, abort, and retry in an overlapping pattern. Unlike database systems, Haskell STM does not implement backoff or priority queuing. In practice, fine-grained TVars avoid this—but coarse-grained use (one TVar protecting a large data structure) can produce pathological behavior.

**Transactions are in-process only.** STM commits atomically within one OS process and survives neither crashes nor cross-machine coordination. For durability you still need a database; STM is not a substitute.

**No ordering or fairness guarantees.** `retry` wakes a thread when a TVar changes but provides no guarantee about which thread wins if many are waiting on the same TVar. High-priority threads may starve under heavy load.

**Long transactions increase abort rates.** The more TVars a transaction reads and the longer it runs, the higher the probability that some other thread commits a conflicting write before validation. This is the same problem as long-running database transactions under SI: the longer you hold your snapshot, the more likely it goes stale.

## Why it works

The deeper principle is **optimistic concurrency is compositional; pessimistic locking is not**.

Locks are *pessimistic*: they acquire control of a resource before any work begins. To combine two lock-using operations, you must acquire all their locks before starting either one. This requires knowing their internal lock identities and imposing a global ordering—both break abstraction.

Transactions are *optimistic*: they execute first and validate later. To combine two transactional operations, you simply sequence them inside `atomically`. The combined transaction reads and writes a merged set of TVars, validates the entire set at commit, and either commits or aborts atomically. No pre-declared lock set. No ordering. No deadlock.

This is the same asymmetry as SQL: `SELECT … FOR UPDATE` (pessimistic) must declare which rows to lock before reading them, while `SERIALIZABLE` transactions (optimistic) read freely and validate at commit. STM brings the SQL `SERIALIZABLE` isolation model into shared memory.

The `retry` / `orElse` pattern reveals a second principle: **let the runtime track dependencies rather than programming them manually**. With condition variables, you must register on each condition before sleeping. With `retry`, the STM runtime records your read set automatically and uses it to determine when to wake you. This is exactly how ZooKeeper watches work: register once, get notified when the data changes, re-read and check the condition. STM generalizes this to arbitrary Haskell predicates over any set of TVars.

The "X is just an instance of Y" insight: **STM is OCC applied to RAM**. The transaction log is the in-memory equivalent of a database's undo log. The read-set validation is snapshot isolation's conflict check. `retry`'s wake-on-write is ZooKeeper's watch mechanism. And `orElse`'s composability is Unix pipes applied to blocking operations: each stage is independently correct, and composition preserves correctness.

## Going deeper

1. **"Composable Memory Transactions"** — Harris, Marlow, Peyton Jones, Herlihy (CACM 2008). The formal treatment: a denotational semantics for STM, proof that `orElse` is commutative in a specific sense, and the original GHC implementation details. https://cacm.acm.org/research/composable-memory-transactions/

2. **Clojure's STM** (`dosync`, `ref`, `alter`, `ensure`, `commute`) — a production implementation in a dynamically typed language, with the `commute` optimization for commutative operations (like incrementing a counter) that permits out-of-order commits without conflict. https://clojure.org/reference/refs

3. **Intel TSX (Transactional Synchronization Extensions)** — hardware transactional memory in x86: the CPU's L1 cache tracks the read/write set, and cache coherency protocol conflicts trigger hardware aborts. The same OCC pattern, but validated in hardware with zero software overhead for uncontested transactions. The 2021 Spectre mitigation disabled TSX on most Intel CPUs, but the design is instructive. https://en.wikipedia.org/wiki/Transactional_Synchronization_Extensions
