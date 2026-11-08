---
title: "Futexes Are Tricky"
source: https://www.ic.unicamp.br/~islene/2s2013-mo806/futex/futex-are-tricky.pdf
author: Ulrich Drepper
company: Red Hat
date_posted: 2008-01-31
date_digested: 2026-11-08
---

# Futexes Are Tricky

## What's new to learn

1. **Fast path / slow path syscall elimination** — a mutex can protect its common case (uncontended acquire) with a single CAS in user space and zero syscalls; only threads that actually need to block touch the kernel.

2. **The three-state mutex encoding** — using 0 (unlocked), 1 (locked, no waiters), and 2 (locked, waiters exist) gives the unlock path a cheap way to decide whether a wake syscall is necessary, avoiding a kernel round trip in the common case.

3. **Atomic condition-check in FUTEX_WAIT** — the kernel verifies the futex word's value atomically before putting the thread to sleep; this closes the inherent race between checking the word in user space and deciding to call into the kernel, the precise source of the "tricky" in the title.

## Prerequisites

- What a system call is and roughly what it costs (~100 ns vs ~1 ns for a CAS)
- Atomic operations: compare-and-swap (CAS), exchange, fetch-and-decrement
- The basic locking problem: at most one thread holds a mutex at a time
- Memory ordering: why compiler and CPU reordering makes concurrent code hard

## The core idea

An OS mutex implemented purely in the kernel is correct, but every lock and unlock pays a syscall — even when no other thread is competing. For a mutex that guards a one-microsecond critical section and is acquired a million times per second, that overhead dominates.

The futex (fast userspace mutex) splits the mutex into two layers:

- **User-space layer**: a 32-bit integer in shared memory encoding the lock state. Threads manipulate it with atomic operations. No syscalls required when nothing is contended.
- **Kernel layer**: a wait queue keyed on the futex's physical address. The kernel provides exactly two primitives: `FUTEX_WAIT(addr, expected)` (sleep if `*addr == expected`) and `FUTEX_WAKE(addr, n)` (wake up to n threads waiting on addr).

The contract between the layers: user space handles the protocol; the kernel handles blocking. The result is that uncontended lock/unlock is a handful of atomic instructions — a few nanoseconds — while only the rare contended case pays the kernel overhead.

## Mechanics

### The wrong first attempt

A naive lock/unlock pair:

```c
// val: 0 = unlocked, 1 = locked
void lock(atomic_int *m) {
    while (cmpxchg(m, 0, 1) != 0)
        futex_wait(m, 1);  // wrong: race here
}
void unlock(atomic_int *m) {
    *m = 0;
    futex_wake(m, 1);  // wrong: unnecessary wake when no waiters
}
```

This is broken in two ways. First, `unlock` always calls `futex_wake` even when no thread is sleeping — wasting a syscall. Second, the check `*m == 1` in `futex_wait` and the actual sleep are not atomic from the user's perspective: a wakeup arriving between that check and the sleep is silently dropped.

The kernel's atomic check in `FUTEX_WAIT` fixes the second bug: if `*m != 1` by the time the kernel checks, it returns `EAGAIN` immediately. So the race is handled, but the first problem (wasteful always-wake) remains.

### The three-state solution

Add a third state: "locked with at least one thread waiting in the kernel."

```
0 = unlocked
1 = locked, no threads sleeping in kernel
2 = locked, one or more threads sleeping in kernel
```

**Lock** (Drepper's third implementation):

```c
void lock(atomic_int *m) {
    int c;
    // Fast path: try 0 → 1 (uncontended acquire, no syscall)
    if ((c = cmpxchg(m, 0, 1)) == 0)
        return;

    // Slow path: someone holds it
    // Upgrade to state 2 so the holder knows to wake us
    if (c != 2)
        c = xchg(m, 2);
    while (c != 0) {
        futex_wait(m, 2);   // kernel: sleep if *m still == 2
        c = xchg(m, 2);     // re-try, but always set state to 2
    }
}
```

**Unlock**:

```c
void unlock(atomic_int *m) {
    // If old value was 2, there are waiters; otherwise no wake needed
    if (fetch_and_dec(m) != 1) {
        // Had waiters (old == 2). Set to 0 and wake one.
        *m = 0;
        futex_wake(m, 1);
    }
    // If old == 1 (no waiters), fetch_and_dec already set to 0. Done.
}
```

**Why state 2 propagates through contended acquires**: every contending thread calls `xchg(m, 2)` before sleeping. This guarantees that so long as any thread is waiting, the value is 2 — giving the unlocker a reliable signal that a wake call is needed.

**The subtle correctness argument**: between `fetch_and_dec` setting `*m` from 2 to 1, and the subsequent `*m = 0`, another thread could call `cmpxchg(m, 0, 1)` and find neither 0 nor the right value. It then executes `xchg(m, 2)`, making `*m = 2` again. The unlocker then sets `*m = 0` and calls `futex_wake`. No lock is lost: either the new thread acquired it via `xchg` returning 1 (meaning the old holder had just released), or the wake call correctly unblocks a sleeper. The kernel's atomic check catches each case.

### FUTEX_WAIT atomicity — why it prevents lost wakeups

The critical race scenario:

```
Thread A holds lock (m = 2)
Thread B executes fetch_and_dec: m = 1 → 0, decides "no waiters", skips wake
Thread A notices m != 2 from its xchg, sees 0, acquires successfully
```

No lost wakeup here. Now the dangerous case:

```
Thread A holds lock (m = 1)
Thread B: cmpxchg(m, 0, 1) fails, m becomes 2 via xchg
Thread A: fetch_and_dec sees old value = 1 (no waiters!), sets m = 0, skips wake
Thread B: calls futex_wait(m, 2)
Kernel: checks *m == 2? No, *m = 0. Returns EAGAIN immediately.
Thread B: re-tries xchg(m, 2); since m=0, this returns 0, so it acquires!
```

The kernel's atomic check is the safety net: a thread never sleeps if the futex value no longer matches what it expected, so the "unlocker went first" case always self-heals.

### What the kernel actually does

The futex syscall is intentionally minimal. The kernel keeps a hash table (the "futex hash table") mapping physical addresses to wait queues. `FUTEX_WAIT`:

1. Locks the hash bucket for `addr`.
2. Atomically reads `*addr` and compares with `val`.
3. If they differ: unlock bucket, return `EAGAIN`.
4. Otherwise: add the calling thread to the wait queue, unlock bucket, call the scheduler.

`FUTEX_WAKE`:
1. Locks the hash bucket.
2. Removes up to `n` threads from the wait queue.
3. Marks them runnable.
4. Unlocks bucket.

The hash on physical address (not virtual) matters: two processes that mmap the same file or shared memory segment will hash to the same bucket, so a futex in shared memory works across processes.

### Condition variables

A condition variable built on futexes follows the same pattern. The CV holds its own futex word (a "sequence number"). `wait` atomically drops the mutex, reads the sequence number, and calls `FUTEX_WAIT` on it. `signal` increments the sequence number and calls `FUTEX_WAKE`. The sequence number approach means spurious wakeups are possible (a benign EAGAIN), which is why POSIX requires callers to re-check the predicate in a loop — an API decision driven by the implementation.

## Where it breaks

**Priority inversion**: nothing in the basic futex prevents a low-priority thread from holding a lock that a high-priority thread needs. Linux has `FUTEX_LOCK_PI` for priority inheritance, but it complicates everything: the kernel must track lock ownership, and you now have a fully kernel-mediated lock that defeats the original motivation in the contended case.

**Private vs. shared futexes**: the kernel hashes on physical address by default (enabling cross-process use), which requires a TLB lookup to resolve virtual → physical. Linux adds `FUTEX_PRIVATE_FLAG` to skip this and use the virtual address directly when the futex is not shared — a small but measurable speedup for single-process use.

**Kernel hash collisions**: if thousands of futexes hash to the same bucket, `FUTEX_WAKE` on one wakes all threads in the bucket only to have them check and re-sleep. This "hash thundering herd" is rare but real. Increasing `CONFIG_FUTEX_HASHSIZE` mitigates it.

**The 32-bit limit**: a futex word is exactly 32 bits. This constrains the state machine: Drepper's scheme uses the entire int to encode lock state; a reader-writer lock needs careful packing to fit readers, writers, and waiter counts into 32 bits.

**Complexity**: the three-state protocol is subtle enough that Drepper himself says he got it wrong at first. Every piece of locking library code (glibc's `pthread_mutex_t`, Rust's `std::sync::Mutex`, Go's `sync.Mutex`) re-implements a version of this protocol; bugs in those implementations are notoriously hard to find and reproduce.

## Why it works

The deeper principle: **give user space a protocol medium (the shared integer) and give the kernel only the scheduling services (block/wake) — never the protocol itself.**

This is one instance of the general "data path in user space, control plane in kernel" pattern:

| System | User-space protocol | Kernel service |
|---|---|---|
| futex | CAS on shared int | block / wake |
| io_uring | shared SQ/CQ ring buffers | I/O completion |
| DPDK | poll on NIC descriptor ring | interrupt (fallback only) |
| vDSO `clock_gettime` | read memory-mapped time page | update time page (kernel-side) |
| RDMA | post to NIC queue pair | connection setup |

What they all share: the **common case** (mutex uncontended, I/O already complete, time already in cache, NIC ring has space) costs zero mode switches. The **rare case** (must actually block, needs an interrupt, needs a connection) calls into the privileged layer. The key design challenge in every case is preventing a lost event between the check in user space and the blocking call — and in every case, the solution is an atomic visibility contract with the kernel/hardware: FUTEX_WAIT's atomic check, io_uring's ordered SQPOLL ring, RDMA's send completion queue ordering.

The futex also demonstrates that **three states beat two states** when you need to know whether anyone is sleeping. This is a broader principle in protocol design: encoding "someone is waiting" in the shared state eliminates a mandatory notification broadcast. The same idea appears in TCP's FIN_WAIT vs CLOSE_WAIT (knowing which side initiated the close), MVCC transaction state (COMMITTED vs ABORTED vs IN_PROGRESS), and SSI's SIREAD "conflict pending" flag.

## Going deeper

1. **"Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux"** (Franke, Russell, Kirkwood, OLS 2002) — the original design paper describing why futexes were added to Linux, their performance vs. alternatives, and the initial implementation. Drepper's paper builds on this.

2. **glibc `nptl/pthread_mutex_lock.c`** — reading the production implementation of `pthread_mutex_t` shows all the edge cases Drepper's paper elides: recursive mutexes, error-checking mutexes, robust mutexes (which handle holder death), and the `FUTEX_LOCK_PI` path for priority inheritance.

3. **"Locks, Mutexes, and Futexes"** by Eli Bendersky (eli.thegreenplace.net/2018/basics-of-futexes/) — a step-by-step C implementation walkthrough that takes Drepper's third algorithm and benchmarks it; a useful complement to the paper for seeing the assembly and timing numbers.
