---
title: "An Introduction to Lock-Free Programming"
source: https://preshing.com/20120612/an-introduction-to-lock-free-programming/
author: Jeff Preshing
company: Independent (formerly Ubisoft)
date_posted: 2012-06-12
date_digested: 2026-11-24
---

# An Introduction to Lock-Free Programming

## What's new to learn

1. **Progress conditions hierarchy** — A formal taxonomy (wait-free > lock-free > obstruction-free > blocking) that precisely classifies every concurrent algorithm by *what it guarantees* about thread forward progress, independent of scheduling.
2. **The ABA problem** — A class of CAS correctness bug where a memory location is changed from A→B→A between a read and a compare-and-swap, causing the CAS to succeed spuriously; the foundational problem in lock-free memory reclamation.
3. **The CAS-loop as the universal lock-free primitive** — Every lock-free algorithm reduces to the same three-step pattern: (1) snapshot shared state atomically, (2) compute desired next state locally, (3) CAS to commit; if a concurrent writer raced us, the CAS fails and we retry. This is the entire design space.

## Prerequisites

- What a *compare-and-swap* (CAS) instruction does: `CAS(&x, expected, desired)` atomically writes `desired` to `x` and returns success only if `x == expected` at the time of the write.
- Why ordinary mutexes create priority inversion and convoy problems under preemptive scheduling.
- x86 memory ordering basics — specifically that x86 gives *total store order* (TSO), so stores become visible in program order but loads can bypass stores; this matters when reasoning about CAS loop correctness beyond sequential consistency.

## The core idea

A mutex is a promise: "one thread at a time." That promise is enforced by blocking — the OS suspends waiters. The problem is that blocking couples progress of unrelated threads: if the lock-holder is preempted at the worst moment, every other thread stalls even though none of them is doing anything wrong.

Lock-free programming breaks that coupling. The formal definition from the literature (Herlihy 1991): **an algorithm is lock-free if at least one thread makes progress in every finite number of steps, regardless of how other threads are scheduled.** No single thread's preemption can stall the entire system. This is strictly stronger than "no mutexes": a spin loop that burns CPU is lock-free only if *some* thread's CAS eventually succeeds.

The progression:

| Level | Guarantee |
|---|---|
| **Wait-free** | *Every* thread completes any operation in a bounded number of *its own* steps, regardless of other threads. |
| **Lock-free** | *At least one* thread completes any operation in a bounded number of steps system-wide; individual threads may retry indefinitely under adversarial scheduling. |
| **Obstruction-free** | A thread completes if it runs in isolation (no other thread executes). Weakest useful non-blocking guarantee. |
| **Blocking** | A thread may stall indefinitely waiting for another thread to release a lock. |

Practical engineering reality: most useful lock-free data structures are lock-free, not wait-free. Wait-free algorithms exist (Herlihy's universal construction, some RCU-style reads) but are often slower in the common uncontended case than a lock-free or even a mutex-based implementation.

## Mechanics

### The CAS loop

Every lock-free algorithm uses the same skeleton:

```cpp
T expected = atomic_load(&shared);
T desired;
do {
    desired = f(expected);           // compute desired next state
} while (!CAS(&shared, &expected, desired));
// postcondition: shared == desired
```

`CAS` on modern hardware atomically does:
```
if (*addr == expected) { *addr = desired; return true; }
else { expected = *addr; return false; }
```

The retry on failure is load-bearing: it re-reads the actual current value into `expected` so the next iteration computes `desired` from fresh state. This retry makes the algorithm lock-free: if our CAS fails, it proves *some other thread's CAS succeeded*, so the system made forward progress.

A concrete example — a lock-free stack push (Treiber stack):

```cpp
void push(Node* node) {
    Node* top = head.load(acquire);
    do {
        node->next = top;
    } while (!head.compare_exchange_weak(top, node, release, relaxed));
}
```

Each iteration tries to splice `node` onto the head. If `head` changed since we read it (another thread pushed), the CAS fails, `top` is updated to the real head, and we try again with `node->next` updated. At least one of the racing pushes always wins each CAS, so the stack is lock-free.

### The ABA problem

Consider a lock-free stack with these two items: head → A → B.

A thread T1 wants to pop A. It reads `head = A` and computes `desired = B` (pop A, leaving B as head).

Before T1's CAS fires:

1. Thread T2 pops A (head → B).
2. Thread T2 pops B (head → null).
3. Thread T2 pushes A back (head → A, but A→next is now garbage or recycled).

Now T1's CAS fires: `head == A` (matches!), so the CAS succeeds and sets `head = B`. But B was already freed/reused — the stack now points into garbage. The ABA problem: the pointer looked the same but the *world had changed* beneath it.

**Standard fixes:**

1. **Tagged pointers (double-width CAS)** — Pair the pointer with a monotonically increasing counter. CAS on the pair atomically. ABA is impossible because the counter never repeats.
   ```cpp
   struct TaggedPtr { Node* ptr; uintptr_t tag; };
   // CAS on the {ptr, tag} pair; each successful CAS increments tag
   ```

2. **Hazard pointers** (Maged Michael, 2004) — Before dereferencing a pointer, publish it in a thread-local "hazard pointer" slot. The free-list scanner checks all hazard pointer slots before reclaiming memory, so a pointer in-use by any thread is never recycled. Progress guarantee: each thread's hazard slot acts as a GC root.

3. **Epoch-based reclamation** (EBR) — Maintain a global epoch counter. Threads announce their current epoch when entering a critical section. Memory freed in epoch E is deferred until all threads have advanced past E. This is RCU's core mechanism (see also "What is RCU, Fundamentally?" in this archive).

4. **Garbage collection** — If the language runtime ensures memory is never reused at the same address while any reference exists (reference counting, tracing GC), ABA cannot occur.

### Memory ordering

CAS alone does not provide full sequential consistency. On weakly-ordered hardware (ARM, Power) and even on x86 for compiler reordering, reads and writes outside the CAS can be reordered unless you annotate them:

```cpp
// ARM: without barriers, the store to node->value might be
// visible to other threads AFTER the head CAS that made node reachable!
node->value = 42;                   // non-atomic write
head.compare_exchange_strong(..., memory_order_release);  // release barrier
// guarantees: all writes before the CAS are visible to any thread
// that subsequently reads head with memory_order_acquire
```

The acquire/release pair forms a synchronizes-with edge: the release write "synchronizes-with" a subsequent acquire read of the same variable, establishing a happens-before relationship between the operations bracketed by them.

## Where it breaks

**Composability.** CAS covers exactly one machine word (or on some architectures, two adjacent words with DCAS). You cannot atomically pop from one lock-free stack and push to another in a single atomic step. This forces applications to either tolerate non-atomic multi-structure operations or use Software Transactional Memory (which re-introduces coordination overhead).

**Live-lock under pathological scheduling.** Lock-free ≠ free from livelock. Under adversarial scheduling, one thread can keep winning every CAS while another keeps retrying. This is an *adversarial* scenario — real OS schedulers make it unlikely — but it means lock-free is not the same as starvation-free. Wait-free is needed for starvation freedom.

**Higher complexity for simple cases.** A lock-free hash table is vastly more complex to implement correctly than `mutex + unordered_map`. For latency-insensitive code paths, the simplicity of a mutex often wins. Lock-free is primarily valuable for hot paths where lock contention is a measured bottleneck.

**Wait-free algorithms are often slower in practice.** Universal constructions for wait-free algorithms (Herlihy 1991) work by having each thread help complete pending operations from other threads, but this helping mechanism adds overhead in the common uncontended case. Practical wait-free algorithms (like a wait-free queue by Kogan and Petrank) are complex engineering achievements, not everyday code.

**Verifying correctness is hard.** A linearizability proof requires identifying a *linearization point* — a single atomic instruction at which each operation appears to take effect. Finding that point is subtle; Preshing notes that for complex algorithms it may not be a fixed instruction but can depend on execution history.

## Why it works

Lock-free programming is **optimistic concurrency control (OCC) at hardware granularity**.

In database OCC: read shared state, compute changes locally, attempt to commit; if someone else committed concurrently, abort and retry. The CAS loop is exactly this, with the "transaction" being a single atomic operation on a single word.

The ABA problem has a precise analogue: in OCC, it corresponds to a transaction reading version 5, the row being updated twice (5→6→5 by coincidence, perhaps with different data), and the transaction wrongly believing no conflict occurred. Database MVCC prevents this by using non-repeating transaction IDs (monotonic version counters), just as the tagged-pointer fix uses a monotonic ABA counter.

The progress guarantees map to distributed system consistency levels. Wait-free is the concurrent-data-structure analogue of **linearizability with bounded latency** (every request finishes in bounded time regardless of contention). Lock-free is the analogue of **linearizability** (every operation finishes eventually, assuming the system is non-faulty as a whole). Blocking with mutexes is the analogue of a system that may stall if any participant fails.

Memory barriers and acquire/release semantics are the concurrent-data-structure analogue of **happens-before in distributed systems**: a release write to a variable "sends a message" to any thread that subsequently acquire-reads it, establishing a causal ordering on all the operations that happened before the release.

The **ABA problem = memory reclamation under concurrent readers** is the same problem RCU, epoch-based reclamation, and hazard pointers solve in the kernel. Each of these mechanisms is a GC root protocol: delay reclaiming memory until you can prove no thread holds a reference to it. Generational GC, MVCC snapshot isolation, and RCU grace periods are all instantiations of the same abstract pattern: "defer destruction until quiescence."

## Going deeper

1. **Herlihy, M., "Wait-Free Synchronization," ACM TOPLAS, 1991** — The original paper defining the progress condition hierarchy and proving that every object has a wait-free implementation using CAS as a universal primitive. The technical result: a `consensus number` classifies primitives by which of them can implement any shared object wait-free for N threads; CAS has infinite consensus number.

2. **Herlihy, M. & Shavit, N., "The Art of Multiprocessor Programming"** — The definitive textbook. Chapters 9–15 build progressively more complex lock-free structures (stacks, queues, skiplists, hash tables) with full correctness proofs and performance analysis.

3. **Michael, M.M., "Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects," IEEE TPDS, 2004** — The seminal paper on memory reclamation for lock-free data structures, introducing the hazard pointer protocol. Directly comparable to RCU (see also "What is RCU, Fundamentally?" in this archive), which solves the same problem by epoch-based quiescence rather than per-pointer publishing.
