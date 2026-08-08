---
title: "Memory Reordering Caught in the Act"
source: https://preshing.com/20120515/memory-reordering-caught-in-the-act/
author: Jeff Preshing
company: (independent)
date_posted: 2012-05-15
date_digested: 2026-08-08
tags: [concurrency, hardware, cpu, memory-model, systems]
---

# Memory Reordering Caught in the Act

## What's new to learn

1. **Store-load reordering**: A CPU can delay making a store visible to other cores until *after* a subsequent load has completed — even on x86. This produces outcomes that are literally impossible under sequential consistency, and the effect is reproducible in a 30-line C program.

2. **The store buffer**: Each CPU core has a private write queue. A core's own stores bypass it (load-forwarding), so the core sees its own writes immediately, but other cores see nothing until the buffer drains — which is the hardware mechanism that causes reordering.

3. **Memory barriers (fences)**: Explicit CPU instructions (`MFENCE` on x86, `dmb ish` on ARM) that force the store buffer to drain before any subsequent load proceeds, restoring the ordering guarantee your concurrent algorithm needs.

## Prerequisites

- Basic shared-memory concurrency: threads, shared variables, what "race condition" means
- C/C++ familiarity at the level of global variables and function calls
- You do **not** need prior knowledge of CPU microarchitecture; the post reconstructs the hardware model from the observable behavior

## The core idea

Every concurrent programmer implicitly assumes **sequential consistency**: all threads see all memory operations in one global order, and each thread's operations appear in program order within that sequence. This assumption is false on every modern CPU, and the reason is the **store buffer**.

When a core issues a store, putting it directly into shared cache would require acquiring that cache line's exclusive ownership via the cache coherence protocol — a potentially expensive round-trip. Instead, the core queues the store in a private FIFO buffer and moves on immediately. The store reaches shared cache only when the buffer drains asynchronously. Meanwhile, the **core sees its own stores instantly** (the hardware forward-routes loads from the same core through the buffer), but no other core does.

The consequence: a `store(x)` followed by a `load(y)` can, from another thread's perspective, appear as if the load happened *before* the store. This is **store-load reordering**, and it's the only form of reordering that x86 (with its relatively strong TSO model) permits. ARM and POWER permit much more.

## Mechanics

**The experiment.** Two threads run concurrently, sharing two zero-initialized globals `x` and `y`, and two per-thread registers `r1` and `r2`:

```
Thread 1:       Thread 2:
  x = 1;          y = 1;
  r1 = y;         r2 = x;
```

Under sequential consistency, one of these four interleavings must occur:

| Scenario | Result |
|---|---|
| T1 runs fully, then T2 | r1=1, r2=1 |
| T2 runs fully, then T1 | r1=1, r2=1 |
| T1 stores x, T2 stores y, T1 loads y, T2 loads x | r1=1, r2=1 |
| T2 stores y, T1 stores x, T2 loads x, T1 loads y | r1=1, r2=1 |

In **every** sequentially consistent execution, at least one thread completes its store before the other does its load. So **r1=0, r2=0 is impossible** — someone must see the other's write.

Preshing's demo makes `r1=0, r2=0` appear reliably on x86 hardware:

1. Thread 1 issues `x = 1` → enters T1's store buffer (not yet in cache)
2. Thread 2 issues `y = 1` → enters T2's store buffer (not yet in cache)
3. Thread 1 reads `y` → cache holds 0 (T2's write is buffered, not yet in cache)
4. Thread 2 reads `x` → cache holds 0 (T1's write is buffered, not yet in cache)
5. Both store buffers drain → `x` and `y` become 1 everywhere, but too late

Both threads read 0. This is not a race condition in the conventional sense — there are no data races on `r1` or `r2`. It is a pure visibility ordering effect.

**The fix is one instruction per thread:**

```
Thread 1:           Thread 2:
  x = 1;              y = 1;
  MFENCE;             MFENCE;     ← fence drains store buffer
  r1 = y;             r2 = x;
```

`MFENCE` (Memory Fence) blocks the core until all prior stores have propagated to the coherence domain. With the fence in place, `r1=0, r2=0` becomes impossible again.

**The C++ atomic mapping.** The C++11 memory model exposes this as named ordering annotations:

| C++ annotation | x86 emission | ARM emission |
|---|---|---|
| `memory_order_seq_cst` store | `mov` + `MFENCE` (or `XCHG`) | `stlr` (store-release) |
| `memory_order_release` store | plain `mov` — TSO makes it free | `stlr` |
| `memory_order_relaxed` store | plain `mov` | plain `str` |
| `memory_order_seq_cst` load | plain `mov` — TSO makes loads see all prior stores | `ldar` (load-acquire) |
| `memory_order_acquire` load | plain `mov` | `ldar` |

The key insight: on x86, `release`/`acquire` pairs cost nothing extra because TSO already prevents load-load and store-store reordering. On ARM, every acquire load and release store emits a barrier instruction. Code that relies on the "free acquire/release" property of x86 and omits explicit barriers **will silently break on ARM** — and the bug will only appear under specific timing conditions.

**What x86 TSO (Total Store Order) guarantees:**

| Reordering | x86 TSO |
|---|---|
| Load → Load | Forbidden ✓ |
| Store → Store | Forbidden ✓ |
| Load → Store | Forbidden ✓ |
| **Store → Load** | **Permitted** ✗ |

ARM/POWER permit all four, which is why they need barrier instructions for each.

**Acquire-release as a one-way gate.** The canonical pattern for lock-free message passing:

```cpp
// Producer thread
data = compute_result();
flag.store(true, memory_order_release);  // "gate": no prior write crosses this

// Consumer thread
while (!flag.load(memory_order_acquire)) {}  // "gate": no subsequent read crosses this
assert(data == expected);                     // safe: happens-before established
```

A `release` store acts as a downward barrier: all stores *above* it in program order are flushed before this store is committed. An `acquire` load acts as an upward barrier: all loads *below* it in program order wait until this load completes. Together they establish a **happens-before edge** from producer to consumer: every write the producer made before the release is visible to the consumer after the acquire.

## Where it breaks

**`volatile` is not a memory barrier.** In C, `volatile` prevents the *compiler* from caching a value in a register, but it does not emit a CPU fence. Java's `volatile` *does* imply acquire/release (the Java Memory Model fixed this in Java 5.0/JDK 1.5), but C/C++ `volatile` does not. Code using C `volatile` for cross-thread communication is broken.

**Double-checked locking was famously wrong for a decade** (1996–2004) because of this. The pattern:

```cpp
if (instance == nullptr) {         // first check — no lock
  lock.acquire();
  if (instance == nullptr) {       // second check — locked
    instance = new Singleton();    // store to `instance` before ctor body
  }                                // ... violates assumption
  lock.release();
}
```

The compiler or CPU can reorder `instance = new Singleton()` to store the pointer *before* the constructor body executes (because the compiler may split the allocation and initialization). A thread passing the first check can receive a non-null but incompletely-initialized object. The fix (per C++11) is to make `instance` a `std::atomic<Singleton*>` with proper acquire/release.

**TSO does not mean "safe on x86."** TSO prevents all reorderings *except* store-load. But many lock-free algorithms assume stricter guarantees — specifically that when you do a CAS, all your prior stores are visible before the CAS succeeds. Under TSO this holds, because CAS requires an exclusive cache line, which implicitly drains the store buffer first (`LOCK` prefix). But it is easy to build a mental model where TSO implies more than it does.

**Portability traps.** Because x86 gives acquire/release "for free," code tested only on x86 can have real bugs that only manifest on ARM servers (Amazon Graviton, Apple M-series) or RISC-V systems. The Linux kernel has discovered multiple such bugs over the years when porting to ARM64.

**Memory ordering is not cache coherence.** These are distinct problems: *coherence* ensures that if two cores write to the same address, they eventually agree on the final value (handled by MESI/MOESI protocols, automatic). *Ordering* governs which stores are visible when, and in what relative order (requires explicit barriers). A system can have perfect coherence and still exhibit the `r1=0, r2=0` outcome above.

## Why it works

The deeper principle: **happens-before must be explicitly constructed at every level of the computing stack.**

At the distributed-systems level, Lamport's 1978 paper established that "message send" and "message receive" create happens-before edges: everything a sender did before sending is visible to the receiver after receiving. Raft log indices, Kafka offsets, and Zanzibar's zookie (a Spanner timestamp) are all mechanisms for propagating this relationship across nodes.

At the CPU level, a `release` store and an `acquire` load establish the *exact same relation* — but within a single machine. When producer does `flag.store(true, release)` and consumer does `flag.load(acquire)` and sees `true`, the consumer can be certain that all of the producer's prior stores (to `data`, etc.) are visible. This is literally the same happens-before edge, materialized by draining a store buffer instead of committing a log entry.

So: **memory barriers are the message-passing primitive of shared-memory systems.** Acquire/release semantics on a multicore CPU are structurally identical to vector clock ticks in a distributed system. The difference is network latency (nanoseconds vs milliseconds) and the mechanism for establishing the edge (fence instructions vs quorum writes), not the fundamental structure.

This unifies several things in the archive:
- **LMAX Disruptor** (cache-line padding) prevents *false sharing* — a coherence-level phenomenon. But the ring buffer also needs a `volatile` long for the sequence number, which gives Java's acquire/release ordering so consumers see all fields of an entry before seeing the updated sequence.
- **Linux RCU** relies on memory barriers (`smp_mb()`) to ensure that a pointer clear is globally visible before the grace period begins. The "epoch signal" (the scheduler tick) provides the *timing* but barriers provide the *ordering*.
- **io_uring**'s shared ring buffer uses a `smp_mb()` between writing a submission queue entry and advancing the tail pointer, so the kernel never sees a partial SQE.

In all three cases, the barrier is establishing a happens-before edge across threads — the same primitive, at the CPU-cache level, that Lamport invented for distributed clocks.

## Going deeper

1. **Jeff Preshing's full memory ordering series** — "Memory Ordering at Compile Time," "Acquire and Release Semantics," "The Happens-Before Relation," "The Synchronizes-With Relation," and "A Lock-Free… Without Locks" are each short standalone posts that build the complete model incrementally. Start at preshing.com/archives.

2. **Paul McKenney, "Is Parallel Programming Hard, And, If So, What Can You Do About It?"** — Available free at kernel.org; Chapter 15 (Memory Ordering) is the definitive treatment of Linux-kernel memory ordering, including formal proofs and the LKMM (Linux Kernel Memory Model) that is now machine-checked by tools like `herd7`.

3. **"A Tutorial Introduction to the ARM and POWER Relaxed Memory Models"** by Lau et al. (2012, available via ACM) — The ARM and POWER memory models are notoriously difficult to specify; this paper formalizes them operationally and shows exactly which reorderings each architecture allows. Essential reading if you're writing lock-free code for heterogeneous deployments.
