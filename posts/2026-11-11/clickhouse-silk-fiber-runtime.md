---
title: "Announcing Silk: a silky smooth fiber runtime for ClickHouse"
source: https://clickhouse.com/blog/silk
author: James Cunningham, Vadim Skipin
company: ClickHouse
date_posted: 2026-06-25
date_digested: 2026-11-11
---

# Announcing Silk: a silky smooth fiber runtime for ClickHouse

## What's new to learn

1. **Stackless vs. stackful fiber tradeoff** — Stackless coroutines (C++20 `co_await`) turn each async function into a heap-allocated state machine; they're viral (every function in the call chain must annotate itself), but have small per-frame cost. Stackful fibers allocate one fixed stack per task and let any function at any depth suspend transparently, at the cost of that upfront allocation.

2. **NUMA-aware work stealing** — When an idle thread steals work from a neighbor, it pays a cache-warming penalty proportional to memory distance. Silk formalizes this with a topology-derived cost table (HT sibling ≈ 1 µs, same socket ≈ 50 µs, cross-socket ≈ 500 µs) and skips steal candidates whose cost exceeds the thread's idle budget, keeping stolen fibers on cache-warm CPUs.

3. **Suspension-callback pattern** — Atomically solving the race between "check if the I/O event already arrived" and "park the fiber": the caller passes a callback to `suspend(callback, ctx)` that runs *after* the fiber is fully parked, letting it register the wakeup without a lost-wakeup window.

## Prerequisites

- How an OS thread scheduler works: ready queue, context switch, run-to-completion
- What C++20 coroutines do (`co_await`, suspension points, state machines)
- io_uring basics: submission queue (SQ), completion queue (CQ), `io_uring_enter`
- NUMA topology: why memory accesses to a remote NUMA node cost more than local ones

## The core idea

Silk is a user-space fiber scheduler built for a database server. It runs one pinned OS thread per CPU. Each thread owns its own io_uring ring, a lock-free bounded ready queue, and a sleep tree. When a fiber wants to read a file or poll a socket, it submits the I/O to its thread's io_uring ring and suspends via a callback. When the kernel signals completion, the fiber is re-enqueued on the same CPU (for cache locality). Idle CPUs steal ready fibers from topology-nearby neighbors, subject to a cost-aware budget.

The name nods to Cilk—MIT's 1994 work-stealing runtime—except Silk replaces Cilk's fork/join model with cooperative suspension driven by io_uring completions.

## Mechanics

### The scheduler loop

Each active CPU runs:

```
loop:
  drain_ready_queue()         // run all locally ready fibers
  drain_io_completions()      // process io_uring CQEs, enqueue woken fibers
  if still idle:
    try_steal_from_neighbor() // cost-limited work stealing
  if still idle:
    sleep(io_uring_enter2)    // block until wakeup eventfd fires
```

An `eventfd` registered as a persistent multishot poll wakes the sleeping thread when any other CPU enqueues work for it. This avoids spinning in the common case.

### Context switching

Silk uses `Boost.Context` (`fcontext_t`) for the actual CPU register save/restore, which is about 3.3 ns per switch (6.5 ns round-trip) on a 3.6 GHz Xeon. Each fiber gets a 64 KB mmap'd stack with guard pages at both ends. Stacks are pooled (mmap'd once at startup, never freed) and recycled to avoid repeated syscalls. ASan stack-switch annotations make address sanitizers fiber-aware.

Fiber states: `SUSPENDED → READY → RUNNING → STOPPED`. A transient `SUSPEND_REQUESTED` state handles the race where a wakeup arrives mid-suspension.

### The suspension-callback pattern

```cpp
// Generic form:
// suspend(callback, ctx)
//   - parks the current fiber
//   - then runs: callback(ctx, this_fiber)
//   - callback must call enqueueReady(this_fiber) when the event arrives
```

The callback runs *after* the fiber has fully parked, so the callback can register the fiber as a waiter without risk. If the event already arrived before the fiber parked, `schedule()` can be called inside the callback to cancel the suspension—the fiber goes straight to `READY` instead.

This is a classic "park then register" pattern to prevent lost wakeups, applied here without futexes by pushing the callback into the fiber's metadata.

### Async I/O via io_uring

Each `read`, `write`, or `poll` operation:

1. Submits an SQE to the calling CPU's ring (zero cross-CPU overhead on the common path)
2. Calls `suspend(wakeup_callback, fiber_ptr)` to park the fiber
3. When the kernel produces a CQE, the scheduler's service loop picks it up and calls `enqueueReady(fiber_ptr)`

Silk also supports *registered buffers*: `readFixed`/`writeFixed` register buffer memory once with `io_uring_register_buffers`, avoiding per-I/O page-pinning overhead. Registration is replicated to every active ring, so locked memory scales with CPU count × buffer size—a known footprint risk on wide machines.

### NUMA-aware work stealing

At startup, Silk reads `/sys/devices/system/cpu/cpu*/topology/` and builds a victim list per CPU, sorted by cache-warming cost:

| Relationship | Cost threshold |
|---|---|
| Hyperthreading sibling (shared L1/L2) | ~1 µs |
| Same physical socket (shared L3) | ~50 µs |
| Different socket (remote NUMA node) | ~500 µs |

When a CPU has been idle for time T, it iterates the victim list and skips any candidate whose cost > T. This means a CPU that just went idle will only steal from HT siblings; it won't attempt a cross-socket steal until it has been idle for ≥ 500 µs. The total steal time is capped at T.

Victim selection within a tier is shuffled using the CPU number as seed—deterministic but varied enough to distribute load.

### Adaptive CPU width

Not all CPUs run all the time. Silk keeps an *active prefix* of CPUs awake, ordered physical cores first, then HT siblings. The prefix:

- **Grows** when backlog persists for a full measurement window, or when chained wakeups queue up
- **Shrinks** when the rightmost CPU records three consecutive idle windows
- One "standby" CPU just outside the prefix keeps timed parks so it notices aged backlog

This saves power and avoids cache pollution when queries are light. The tradeoff: partial loads near a width boundary can cause run-to-run throughput variance during warmup.

### Stackless vs. stackful: the core design decision

Silk's authors considered C++20 coroutines before choosing fibers. The comparison (from `docs/coroutines.md`):

| | Stackless (`co_await`) | Stackful (Silk fibers) |
|---|---|---|
| Context switch (measured) | — | ~3.3 ns (Boost.Context) |
| Task creation | ~98 ns | ~40–50 ns |
| Memory per task | N × frame size per coroutine chain | One 64 KB stack (lazy RSS) |
| Deep call chains | Overhead grows linearly with depth | Constant |
| Third-party library integration | Requires rewriting to async | Transparent |
| Viral annotation | Yes (`co_await` propagates up) | No |

The deciding factor for ClickHouse was **third-party library integration**. ClickHouse uses Poco (HTTP) and the AWS SDK, both of which are synchronous. Rewriting them as stackless coroutines would be prohibitive. Stackful fibers let those libraries block normally—the blocking call parks the fiber, not the OS thread.

Silk's TCP echo benchmark (64-byte messages, loopback) shows about 15× higher throughput than Boost.Asio coroutines (epoll backend) at 1 connection, narrowing to about 4× at 1,024 connections. The gap comes from Asio's mutex-protected handler dispatch vs. Silk's per-CPU lock-free ready queues.

## Where it breaks

**Fixed stack size.** At 64 KB per fiber, deep recursion or stack-intensive code hits guard pages. The post acknowledges this but notes lazy page mapping keeps actual RSS much lower than virtual size.

**Registered buffer footprint.** `readFixed`/`writeFixed` lock memory once per active ring. On a 32-CPU machine with 100 MB of registered buffers, that's 3.2 GB of locked memory—this can exceed `RLIMIT_MEMLOCK`.

**io_uring kernel requirement.** Silk needs a modern Linux kernel (5.10+ for most features). It has a fallback path for environments where `io_uring_setup` is refused (e.g., restricted Docker containers), but the fallback loses the async I/O benefit.

**Adaptive width variance.** The adaptive CPU width algorithm can shift width at the load boundary during warmup, causing run-to-run throughput variance on partial loads.

**Not portable.** The scheduler reads Linux-specific sysfs topology paths and uses Linux-specific io_uring. It explicitly targets ClickHouse's Linux deployment environment.

**Cooperative scheduling only.** There is no preemption. A fiber that burns CPU without yielding will starve others on that CPU. Silk assumes well-behaved fibers that do I/O or explicitly yield.

## Why it works

The deeper principle: **a user-space fiber scheduler is a user-space reimplementation of OS scheduler ideas**—ready queues, context switching, NUMA locality, adaptive resource scaling—tuned for a single process's workload rather than the entire system.

The OS scheduler doesn't know that a blocked thread is waiting for a Parquet read vs. a network round-trip; it just unblocks it when the fd becomes readable. Silk knows its own I/O topology, so it can:

- Skip the kernel scheduler entirely (3.3 ns context switch vs. ~5–10 µs for a Linux thread)
- Keep I/O completions pinned to the CPU that submitted them (cache locality)
- Tune steal costs to the actual memory hierarchy (not just "steal or don't steal")

This is the same insight behind LMAX's Disruptor (keep everything on one CPU, avoid cross-thread coordination), behind Netty's event loop model (one thread per NIC queue), and behind Go's M:N goroutine scheduler. What differs is *degree of control*: Silk, as a C++ library, directly controls stack layout, io_uring rings per CPU, and topology-derived steal costs in ways that managed runtimes can't.

The stackless vs. stackful decision is a special case of the general tradeoff: **transparency to the call stack costs memory; invasiveness to the call stack costs migration effort**. For a codebase with deep synchronous library dependencies (Poco, AWS SDK), stackful is a one-time cost of 64 KB per fiber; stackless is a recurring tax paid every time a new library must be async-ified.

## Going deeper

1. **Silk source + docs** — `github.com/ClickHouse/silk`, especially `docs/scheduler.md` and `docs/coroutines.md`: the scheduler design is unusually well-documented for an engineering project.
2. **"Scalable Go Scheduler Design" (Dmitry Vyukov, 2012)** — The reference design for M:N work-stealing with per-P run queues; Silk reaches similar conclusions from a C++ starting point.
3. **"io_uring and the Kernel Queue" (Jens Axboe, LWN)** — The definitive description of io_uring's submission/completion ring design and registered-buffer semantics, which underpin Silk's async I/O path.
