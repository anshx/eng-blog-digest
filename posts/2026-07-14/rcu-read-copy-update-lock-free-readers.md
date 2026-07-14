---
title: "What is RCU, Fundamentally?"
source: https://lwn.net/Articles/262464/
author: Paul E. McKenney
company: IBM (Linux kernel community)
date_posted: 2007-12-17
date_digested: 2026-07-14
---

# What is RCU, Fundamentally?

## What's new to learn

1. **Grace period** — The bounded interval after a write during which the old version must be preserved. It ends when every CPU has passed through at least one *quiescent state* since the write, proving no pre-write reader can still hold a reference.
2. **Quiescent state** — A moment when a CPU provably holds no RCU-protected reference. In the classic kernel this is simply a context switch: because RCU read-side critical sections disable preemption, a scheduled task proves the old critical section is over.
3. **Publish-subscribe pointer protocol** — `rcu_assign_pointer` / `rcu_dereference`: the writer atomically publishes a new version with a store barrier; the reader loads it with a data-dependency fence. Readers see either old or new, never a torn half-written state.

## Prerequisites

- What a mutex and reader-writer lock are, and why they serialize on cache lines
- Basic memory ordering: that CPU stores can be reordered and that fences constrain that reordering
- Why reference counting (e.g., `std::shared_ptr`) is slow: every increment/decrement is an atomic operation that bounces the cache line between cores
- *Helpful but not required*: MVCC (Multi-Version Concurrency Control) from databases — the parallel is very direct

## The core idea

Reader-writer locks have a dirty secret: even though readers don't write the *data*, they write the *lock counter*. That counter is shared state; on a 128-core machine, `pthread_rwlock_rdlock()` becomes a cache-line ping-pong tournament. Throughput *falls* as you add readers.

RCU's solution is to eliminate the shared reader-side counter entirely.

The observation: if you atomically publish a new version of a data structure and wait until you can prove that no reader holds the *old* version, you can safely free the old version. The wait is the "grace period." Readers require zero shared state to participate — they just disable preemption (already free in the hot path) and dereference.

This splits the write into two asynchronous phases:
1. **Publish**: atomically install the new version. New readers see new; old readers finish at their own pace.
2. **Reclaim**: after the grace period, free the old version.

The price paid by the writer (waiting for a grace period) is amortized over many reads. For read-heavy data structures — routing tables, process lists, module lists — this tradeoff is enormous.

## Mechanics

### The read side

```c
rcu_read_lock();                    // disables preemption (just a counter in the task struct)
struct foo *p = rcu_dereference(gp); // LOAD with data-dependency barrier
if (p)
    do_something(p->field);
rcu_read_unlock();                  // re-enables preemption
```

`rcu_read_lock()` does not touch any shared memory. On CONFIG_PREEMPT_NONE kernels it compiles to nothing. The "barrier" in `rcu_dereference` on most architectures is just a compiler barrier, not a CPU fence — the Alpha exception aside, hardware data-dependency ordering is sufficient.

Cost: essentially zero. No cache-line writes. No atomic operations.

### The write side

```c
// 1. Allocate and initialize new version
struct foo *new_fp = kmalloc(sizeof(*new_fp), GFP_KERNEL);
*new_fp = *gp;          // copy existing
new_fp->field = 42;     // apply mutation

// 2. Atomically publish — readers now see new_fp or old gp
rcu_assign_pointer(gp, new_fp);    // STORE with memory barrier

// 3. Wait for grace period — block until all pre-publication readers are done
synchronize_rcu();

// 4. Now safe to reclaim
kfree(old_fp);
```

`rcu_assign_pointer` inserts a write memory barrier before the pointer store, ensuring the new data is visible before the pointer itself. This pairs with the load in `rcu_dereference`.

### The grace period mechanism

`synchronize_rcu()` needs to answer: "have all CPUs that could have seen the old pointer finished their read-side critical sections?"

The classic implementation (Tree RCU) runs a "quiescent state" detector:

1. Record which CPUs are online at the start of the grace period.
2. Wait until every CPU reports at least one quiescent state — a context switch, idle entry, or CPU-hotplug event.
3. Once every CPU has quiesced at least once, any read-side critical section that was in-flight at step 1 must have ended. **Why?** Because RCU read-side critical sections disable preemption. If a CPU ran the scheduler (quiescent state), no preemption-disabled critical section was active at that moment.

Tree RCU organizes CPUs into a tree of "rcunodes" to reduce lock contention when reporting quiescent states on large NUMA systems. Grace periods on a 128-core machine complete in microseconds to low milliseconds.

For callers that don't want to block, `call_rcu(old_fp, callback)` registers a callback that fires after the next grace period — the reclaim work is deferred without blocking the writer.

### The list-update idiom

The most common RCU pattern is linked-list modification:

```c
// Insert (writer):
new_entry->next = entry->next;    // set up pointer first
rcu_assign_pointer(entry->next, new_entry);  // publish atomically

// Delete (writer):
rcu_assign_pointer(prev->next, entry->next); // unlink atomically
synchronize_rcu();                            // wait
kfree(entry);                                 // safe now

// Traverse (reader):
rcu_read_lock();
list_for_each_entry_rcu(p, &head, list) {
    // p is stable for this iteration
}
rcu_read_unlock();
```

The key invariant: a traversal that starts after `rcu_assign_pointer` sees the new list; one that started before finishes on the old list and is guaranteed to complete before `kfree`.

## Where it breaks

**Long-lived readers starve reclamation.** If a CPU holds `rcu_read_lock()` for a second — say, a tight computation loop — the grace period cannot end on that CPU. In a system with gigabytes of pending-free RCU memory, this causes OOM. The solution is to never do blocking work (I/O, sleeping) inside an RCU read-side critical section.

**Not a consistent multi-pointer snapshot.** Each `rcu_dereference()` is independent. If a data structure spans two pointers (a "double-pointer" structure), a reader may see pointer A from the old version and pointer B from the new version. This is usually fine for independent objects, but wrong for structures where the two pointers must be consistent. Fix: embed both pointers in a single parent struct and atomically swap the parent pointer.

**Write-heavy workloads.** Writers pay one grace period per write. If writes arrive faster than grace periods complete, `call_rcu` callbacks queue up. Under heavy write pressure, the callback queue can grow without bound. RCU is fundamentally a read-optimized primitive.

**PREEMPT_RT kernels.** Real-time kernels allow tasks to sleep with preemption held. This breaks the "quiescent state = context switch" invariant. PREEMPT_RT's solution (Tasks RCU) tracks which tasks were observed to be in a read-side critical section and waits for each to exit explicitly — heavier for readers, but still correct.

**No write ordering between writers.** RCU does not serialize writers. Multiple concurrent writers need an external lock (a mutex is fine — only readers are wait-free). The prototypical pattern is "one spinlock for writers, RCU for readers."

## Why it works

The underlying principle: **any ordering event that is already present in the system can serve as an epoch boundary for deferred reclamation.**

RCU exploits the kernel scheduler. The scheduler already runs context switches; those context switches already disable-then-reenable preemption; therefore they are already perfect epoch boundaries. RCU piggybacks on this for free.

This exact same pattern appears everywhere:

| System | Write event | Epoch signal | Reclamation trigger |
|--------|------------|--------------|---------------------|
| **RCU** | Pointer swap | CPU context switch | All CPUs quiesced once |
| **MVCC (PostgreSQL/InnoDB)** | Row write | Transaction commit timestamp | Oldest active transaction ID advances past old version |
| **Epoch-based reclamation (lock-free)** | Node unlink | Thread epoch counter increment | All threads in epoch ≥ reclaim epoch |
| **Hazard pointers** | Node unlink | Thread announces/clears hazard pointer | No thread's hazard pointer matches the node |
| **Generational GC** | Object allocation | Minor GC promotion | Object survives enough minor GCs → old-gen |
| **Chandy-Lamport snapshot** | Channel state record | Barrier token traverses channel | All channels have delivered their barrier |

The key difference between these schemes is **what serves as the epoch signal**:

- RCU uses a CPU event (schedule()). Zero-cost for readers. Granularity = one scheduling quantum.
- MVCC uses transaction timestamps. Non-zero cost (timestamp allocation), but enables arbitrary-duration snapshots.
- Epoch-based reclamation uses an explicit counter. Readers pay one atomic read per access. More flexible than RCU (works in user-space without kernel cooperation).
- Hazard pointers announce specific pointers. Per-pointer precision, but readers pay a write per pointer touched.

RCU is at one extreme: minimum reader cost, maximum restriction (can't sleep, can't be preempted). Hazard pointers are at the other extreme: any pointer can be protected, but readers must announce each one.

The abstract structure is: "readers signal when they're done via some side-channel; writers wait for the signal; reclaim." The signal channel determines the cost and generality.

A final insight: in a uniprocessor or cooperative multitasking system, RCU degenerates to a trivial optimization — any point where the single CPU could "switch" is a quiescent state. This is why Node.js can safely garbage-collect objects after the event loop tick with no reference counting at all: the single-threaded event loop IS the epoch boundary.

## Going deeper

1. **McKenney et al., "RCU Usage In the Linux Kernel: One Decade Later" (2013)** — A survey of all the places in the Linux kernel that use RCU, what patterns emerge, and what went wrong in early usage. Hosted at https://pdos.csail.mit.edu/6.828/2025/readings/rcu-decade-later.pdf and used in MIT's OS course. Excellent for understanding the breadth of RCU's applicability.

2. **"A Tour Through RCU's Requirements" — Linux kernel documentation** (https://www.kernel.org/doc/html/latest/RCU/Design/Requirements/Requirements.html) — McKenney's formal specification of what RCU must guarantee, written as a series of scenarios that each must pass. The most rigorous treatment of the correctness conditions.

3. **Desnoyers et al., "User-Level Implementations of Read-Copy Update" (2012), *IEEE Transactions on Parallel and Distributed Systems*** — Describes liburcu, which brings RCU to userspace without kernel cooperation. Shows that the core principle is portable: quiescent states can be detected via POSIX signals, thread-local epoch counters, or memory barriers. The paper https://liburcu.org/ and source show how userspace async I/O frameworks (and modern Rust's ArcSwap) exploit the same idea.
