---
title: "Scalable Go Scheduler Design"
source: https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit
author: Dmitry Vyukov
company: Google
date_posted: 2012-05-02
date_digested: 2026-09-23
---

# Scalable Go Scheduler Design

## What's new to learn

1. **The G/M/P trifecta — separating runnable tasks, parallelism slots, and OS threads**: The introduction of a third scheduler entity P ("Processor") decouples the per-CPU state (local run queue, allocator cache) from the OS thread, so that blocking one OS thread in a syscall does not strand other runnable goroutines.

2. **Syscall handoff**: When a goroutine blocks in a kernel call, its logical "parallelism slot" (P) is transferred to a different OS thread so other goroutines keep running — solving the classical M:N threading hole where blocking syscalls freeze the whole scheduler.

3. **Spinning threads as a latency knob**: Keeping a small number of idle OS threads actively polling for work rather than sleeping buys sub-microsecond goroutine wakeup latency at the cost of wasted cycles on idle cores — a deliberate, bounded trade-off between throughput and tail latency.

## Prerequisites

- OS-level threads and what `GOMAXPROCS` controls in Go
- What goroutines are conceptually (user-space coroutines with their own stacks)
- Why a global mutex is a scalability bottleneck at high contention
- What a blocking syscall is and why it suspends the calling thread in kernel

## The core idea

Before Go 1.1 the scheduler had only two kinds of objects: **G** (goroutine) and **M** (OS thread). Every scheduling action — enqueue a goroutine, dequeue it, steal it — required acquiring a single global lock. That lock held the entire scheduler state, so at millions of goroutines the lock became the bottleneck. Worse, when a goroutine made a blocking syscall, its M parked in the kernel, and the goroutines on that M's run queue could not run until the syscall returned — no other M stepped in.

Vyukov's design, shipped in Go 1.1, adds **P** (Processor): a logical CPU slot that owns all the per-core state. There are exactly `GOMAXPROCS` P's, each holding a local run queue (a lock-free circular buffer), the memory allocator's per-thread cache, and — when a goroutine is running — a bound M. OS threads now pick up work by acquiring an idle P, not by acquiring a global lock. When an M blocks in a syscall, it releases its P so another M can take over and keep running the remaining goroutines in that P's queue.

The result is a three-level multiplexing:
```
Goroutines (G, millions)  →  Processors (P, GOMAXPROCS)  →  OS threads (M, ~GOMAXPROCS + blocked)
```

## Mechanics

### The Three Entities in Detail

**G (goroutine)**: A user-space task. Starts with a 2 KB stack that grows and shrinks dynamically (up to 1 GB by default). Contains its own program counter, stack pointer, and a state field (`running`, `runnable`, `waiting`, `dead`). Creating a goroutine costs ~1 µs and ~3 KB; creating an OS thread costs ~10 µs and ~8 MB of virtual address space.

**M (machine)**: A real OS thread created and managed by the Go runtime. M's run Go code only when bound to a P. When an M makes a blocking syscall it releases its P and sits in the kernel; it still exists but consumes no Go parallelism. The runtime pools idle M's to avoid creating/destroying OS threads on each syscall.

**P (processor)**: A logical CPU slot. Each P holds:
- A *local run queue*: a lock-free single-producer/multi-consumer ring buffer, capacity 256 goroutines
- A pointer to the currently-executing G
- An `mcache`: per-P slab allocator that handles small allocations (< 32 KB) without any lock
- Timers and deferred goroutines

`GOMAXPROCS` is the number of P's and the ceiling on true Go code parallelism.

### The M Scheduling Loop

Every M runs this loop whenever its current G yields:

```
schedule():
1. Every 61st call: try global run queue (prevents starvation)
2. Try local P run queue
3. If local queue empty:
   a. Try global run queue (batch steal)
   b. Try netpoller (goroutines woken by I/O)
   c. Try stealing half of a random P's local queue (4 attempts)
   d. If still nothing: park M (sleep), release P to idle list
```

The "every 61st call" check is subtle: without it, a busy-loop of goroutines could prevent any goroutine enqueued globally (e.g., one that just returned from a syscall) from being noticed by the P that's always busy with its own queue. The prime 61 interleaves global checks uniformly across P's so no single P is always the one to pick up global work.

### Syscall Handoff in Full Detail

Suppose goroutine G1 calls `os.File.Read()`:

1. **`entersyscall()`**: Before entering the kernel, the runtime records that M1 is entering a syscall and atomically marks P1 as `Psyscall` (not `Prunning`). M1 is now "on loan" to the kernel — it may need its P retaken.

2. **M1 blocks in kernel**: M1 is suspended; it holds no P. P1 is in `Psyscall` state, still pointing at M1 but not running anything.

3. **sysmon retakes**: A background goroutine (`sysmon`) runs every 20 µs. It scans P's in `Psyscall` state and if any have been there for > 20 µs, it *retakes* them: transitions P1 to `Pidle` and hands it to an idle M (or wakes/creates a new M) to keep running P1's queued goroutines.

4. **`exitsyscall()`**: M1 returns from the kernel. It tries to reacquire any idle P. If one is available, M1 resumes with G1. If not, G1 is moved to the global run queue and M1 goes to the idle M pool.

The key invariant: *a goroutine that is runnable always has a path to an M to run on*, bounded only by `GOMAXPROCS` active P's.

For **network I/O** there is no blocking syscall at all. When a goroutine calls `net.Conn.Read()` and the socket has no data:

1. The goroutine parks itself in state `Gwaiting`
2. The file descriptor is registered with the *netpoller* (epoll on Linux, kqueue on BSD/macOS)
3. When the fd becomes readable, the netpoller transitions the goroutine back to `Grunnable` and places it in the global queue
4. No M was ever blocked; the goroutine just waited off-thread

### Work Stealing

When P empties its local queue after the global-queue and netpoller checks:

1. Picks a victim P at random (up to 4 attempts)
2. Steals **half** of the victim's local queue in one batch

Stealing *half* rather than *one* is a bandwidth/latency trade-off. One stolen goroutine immediately gives the thief work, but it will need to steal again quickly. Stealing half amortizes the cost of the steal (lock contention on the victim's queue) over multiple goroutines. It also balances load: if the victim was holding 100 goroutines and the thief takes 50, both will eventually finish around the same time.

### Spinning Threads

When a goroutine becomes runnable and there are idle P's, someone needs to wake up. Waking a sleeping OS thread from a kernel park (`futex_wait`) takes ~5 µs — enough to matter for latency-sensitive servers. Go keeps up to `GOMAXPROCS/2` M's in a *spinning* state: they are awake, hold a P, but have no G to run and are actively looking for work. When a new runnable G appears:

- A spinning M immediately grabs it — zero wakeup latency
- No spinning M available → wake a sleeping M from the idle pool

The cap of `GOMAXPROCS/2` spinning threads prevents more than half the CPUs from burning cycles on futile spinning; beyond that threshold new idle M's sleep immediately.

### Preemption Evolution

**Go ≤ 1.13 (cooperative preemption)**: A goroutine is preempted only when the compiler-inserted stack-growth prologue fires at function entry. A tight loop with no function calls (`for {}`) could hold an M indefinitely, preventing GC stop-the-world from completing.

**Go 1.14+ (asynchronous preemption)**: The `sysmon` goroutine sends `SIGURG` to any M whose G has been running for more than 10 ms. `SIGURG` was chosen because it is unused by most Unix programs. The signal handler saves the goroutine's full register state, transitions it to `Grunnable`, and re-queues it — allowing the M to pick up the next G. GC stop-the-world can now complete in bounded time regardless of goroutine behavior.

### The `sysmon` Background Thread

A special M that runs *without* a P. Its jobs:
- **Retake**: preempt G's running > 10 ms (via SIGURG), retake P's stuck in syscalls > 20 µs
- **Poll**: check the network poller and wake goroutines whose I/O is ready
- **Scavenge**: return physical memory pages to the OS when heap footprint shrinks
- **Timers**: force-fire timers whose deadlines have passed if their P is busy

`sysmon` runs at 10 µs frequency when active, backing off exponentially to 10 ms when the system is idle.

## Where it breaks

**Blocking syscall overhead**: Every blocking syscall requires at least two atomic operations (entersyscall/exitsyscall) and potentially a P-retake cycle. Programs dominated by very frequent short blocking syscalls (e.g., tight loops reading small chunks from a local file) will see higher overhead than an equivalent thread-per-goroutine model.

**GOMAXPROCS is a hard parallelism cap**: CPU-bound goroutines cannot exceed `GOMAXPROCS` in parallel. This is by design but surprises users who set `GOMAXPROCS=4` on a 32-core machine expecting linear scaling.

**GC stack scanning**: Every goroutine stack must be scanned during GC stop-the-world. One million goroutines × 2 KB initial stacks × (pointer scanning time) adds up. Programs that hold millions of long-lived, sleeping goroutines still pay GC cost proportional to goroutine count.

**NUMA-blindness**: The work-stealing algorithm picks a victim P at random with no regard for NUMA topology. A goroutine created on NUMA node 0 might steal to node 3, polluting node 0's L3 cache with remote accesses. The scheduler has no NUMA-aware affinity (a long-standing open proposal).

**Goroutine leak invisibility**: Unlike structured concurrency nurseries, goroutines are unscoped — a goroutine blocked waiting on a channel that will never receive a value runs forever. The scheduler is designed for correctness; detecting "goroutines that will never make progress" is not its job. `runtime.NumGoroutine()` is the only diagnostic hook.

## Why it works

The deepest principle here is **level separation via indirection**: every time you want two layers to scale independently, you add a third entity that each layer can point to without owning.

The old G-M model collapsed "parallelism capacity" and "execution mechanism" into a single thing (the OS thread). When the OS thread blocked, both were gone. Introducing P as the "parallelism capacity" layer — the thing that holds the local queue and the mcache — lets the OS thread (M) come and go without disrupting the P's ongoing work.

This is the same structural move as:
- **Virtual memory**: virtual page ≠ physical frame ≠ DRAM cell. Adding the frame indirection lets the OS swap pages without changing the virtual address space.
- **Database buffer management** (ARIES, Postgres heap): logical page ≠ buffer frame ≠ disk block. Pinning a frame lets reads proceed while the underlying disk block migrates.
- **CPU register renaming**: architectural register ≠ physical register. Adding the rename table lets the pipeline execute out-of-order without exposing WAR/WAW hazards to the ISA.

In each case: "add an indirection layer between the abstract resource and the physical one, and let the physical one be reused, blocked, or replaced without disturbing the abstract one."

The work-stealing component independently applies the **Blumofe-Leiserson theorem** (1999): a randomized work-stealing scheduler with P processors achieves the theoretically optimal parallel runtime O(T₁/P + T∞) for any computation DAG — where T₁ is the serial time and T∞ is the critical-path length — using at most O(P × T∞) steals in expectation. Go's scheduler is not purely fork-join (goroutines can block on channels indefinitely) so the theorem doesn't apply exactly, but the randomized-steal design inherits the theorem's locality and efficiency properties in the common case.

## Going deeper

1. **"Scalable Go Scheduler Design Doc" by Dmitry Vyukov (2012)** — the original design document, available at the source link above. Dense but worth reading for the reasoning behind every design decision, including the alternatives considered and rejected.

2. **Blumofe & Leiserson, "Scheduling Multithreaded Computations by Work Stealing" (JACM 1999)** — the theoretical foundation. Proves the O(T₁/P + T∞) bound and bounds the expected number of steals; the analysis is accessible if you know big-O notation and basic probability.

3. **"Inside the Go Scheduler" series by Ardan Labs** (ardanlabs.com/blog, 2018) — the most accessible practical explanation, with diagrams showing G/M/P hand-offs and syscall transitions step-by-step. A good companion to the design doc.
