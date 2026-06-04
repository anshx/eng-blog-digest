---
title: "LMAX Disruptor: High Performance Alternative to Bounded Queues for Exchanging Data Between Concurrent Threads"
source: https://lmax-exchange.github.io/disruptor/disruptor.html
author: Martin Thompson, Dave Farley, Michael Barker, Patricia Gee, Andrew Stewart
company: LMAX
date_posted: 2011
date_digested: 2026-06-04
---

# LMAX Disruptor: High Performance Alternative to Bounded Queues

## What's new to learn

1. **False sharing**: When two threads write to *different* variables that happen to live in the same 64-byte CPU cache line, the MESI cache coherence protocol forces both cores to repeatedly invalidate each other's copy of that line — creating contention that looks exactly like lock contention but is invisible to the programmer.

2. **Ring buffer with sequence-number coordination**: A pre-allocated circular array where producer and consumer threads coordinate through a single monotonically-increasing integer per thread (rather than a mutex) eliminates both OS-level scheduling overhead and GC allocation pressure, while the ring index is just `sequence & (size - 1)`.

3. **Mechanical sympathy**: The principle that high-performance software must be designed around the hardware model — CPU cache lines, NUMA topology, branch predictors — rather than the OS or language concurrency model. The hardware model is the real bottleneck, and the OS model is often an inaccurate proxy for it.

---

## Prerequisites

- Basic Java or C++ threading: what threads are, what `volatile` and `synchronized` do
- CPU cache hierarchy: L1/L2/L3 caches and why cache misses are expensive (~200 ns vs ~1 ns for L1)
- The distinction between lock-based and lock-free (CAS-based) concurrency — but **not** how either performs in practice, because that's the surprise
- What a bounded queue / BlockingQueue is

---

## The core idea

LMAX Exchange builds a retail financial trading platform. Their latency requirement is ~1 ms per trade and their throughput goal is millions of trades per second — on a single machine. They benchmarked Java's `ArrayBlockingQueue` (lock-based), then an "optimistic" lock-free queue using CAS. Both were far too slow: the lock-free queue barely improved over the locked one.

Profiling showed the bottleneck was not OS scheduling, not GC, and not even CAS retries. It was **cache line invalidation traffic**. The CAS-free inner loop was fine; the memory access pattern was not.

Here is the problem in its simplest form. A producer writes to `head` (the next-empty slot index). A consumer reads from `tail` (the next-unread slot index). In a typical queue implementation these two fields live near each other — perhaps in the same object. If they share a 64-byte cache line, the producer and consumer CPUs fight over ownership of that line on every operation, even though they never touch the same *logical* variable. The CPU has no concept of "logical independence"; it only knows that two cores are writing to bytes in the same 64-byte block, and the MESI protocol demands that one of them give up its cached copy before the other can write.

This is **false sharing**. The fix is padding: waste 56 bytes to ensure each hot variable exclusively occupies one cache line. That is the entire secret.

The Disruptor formalizes this insight into a complete data structure:

- **Pre-allocate all slots at startup** so that object layout is fixed, cache-warm, and GC-free.
- **Pad every mutable sequence counter to its own cache line** so no two threads' hot fields share a line.
- **Replace queue head/tail pointers with monotonic sequence counters** — one per thread — so coordination is an integer comparison, not a mutex.

The result, on 2011 hardware, was 25 million events/second through a three-stage processing pipeline, with 52-nanosecond median per-event latency — three orders of magnitude better than a `BlockingQueue` pipeline.

---

## Mechanics

### Ring buffer

```
RingBuffer<Event>  ─── size = 2^N (power of two, chosen at startup)
  slot[0], slot[1], ..., slot[size-1]   ← pre-allocated Event objects
```

Indexing uses a bitmask: `slot = sequence & (size - 1)`. This is faster than modulo and avoids branching.

### Sequence class

```java
// Pseudocode — real impl pads to 128 bytes (64 before + 64 after)
class Sequence {
    long p1, p2, p3, p4, p5, p6, p7;    // 56 bytes of pre-padding
    volatile long value;                  // 8 bytes of actual data
    long p9, p10, p11, p12, p13, p14;   // 56 bytes of post-padding
}
```

Each producer and each consumer has one `Sequence`. Its `value` is the last sequence number that thread has "claimed" (producer) or "processed" (consumer). Padding ensures no two `Sequence` objects share a cache line, so different threads can advance their own sequences independently — no false sharing.

### Publishing (single producer)

1. **Claim**: `nextSeq = producerSequence.incrementAndGet()`
2. **Write**: `ringBuffer[nextSeq & mask].data = myEvent`
3. **Publish**: `publishedCursor.set(nextSeq)` (signals consumers this slot is ready)

Consumers watch `publishedCursor`. If `publishedCursor < waitingFor`, they spin (or yield, or block, depending on the configured `WaitStrategy`).

### Consuming

1. **Wait**: loop until `publishedCursor >= myExpectedSeq`
2. **Batch read**: consume *all* available slots up to `publishedCursor` — this is a key optimization: a consumer can drain a burst of events in one tight loop without checking the cursor for each one
3. **Advance**: `consumerSequence.set(highestConsumed)`

Producers watch the *slowest* consumer's `Sequence`. If the ring is full (producer has lapped the consumer), the producer stalls.

### Consumer dependency graphs

Consumers can declare that they must finish processing event N before a downstream consumer sees it. This forms a directed graph of `Sequence` objects. A downstream consumer waits until `min(allUpstreamSequences) >= myExpectedSeq`. The topology is expressed at construction time — no runtime overhead.

```
[Source] → [Journal] → [Replication]
       ↘               ↗
         [BusinessLogic]
```

The `BusinessLogic` consumer waits for `Journal` to finish (durability first). `Replication` waits for both.

### Wait strategies

| Strategy       | Latency   | CPU use | Mechanism             |
|----------------|-----------|---------|----------------------|
| `BusySpin`     | ~20 ns    | 100%    | Tight integer compare |
| `Yielding`     | ~50 ns    | High    | `Thread.yield()`      |
| `Sleeping`     | ~100 ns   | Medium  | `LockSupport.parkNanos` |
| `Blocking`     | ~1000 ns  | Low     | Condition variable    |

LMAX used `BusySpin` with CPU affinity (pinning the disruptor thread to a dedicated core) for production.

### Multiple producers

With multiple producers, step 1 (claim) uses CAS: `producerSequence.compareAndSet(current, current+1)`. This reintroduces some contention, but since each producer CASes *only once* per event and writes to a different slot, contention is localized to the single `producerSequence` integer — not spread across a shared queue node.

---

## Where it breaks

**Fixed capacity.** If producers outpace consumers, producers block when the ring is full. There is no overflow queue. You must size the ring for your peak burst, not your average rate.

**Multiple producers pay a CAS tax.** The single-producer mode has no CAS at all. Multiple producers must CAS-compete for sequence numbers. Under very high producer parallelism, this becomes a bottleneck.

**Object reference indirection (Java).** Ring slots store *references* to pre-allocated `Event` objects, not the event data itself. Reading a slot still requires a pointer dereference into another heap location. In C++ you can embed objects directly in the ring, eliminating this indirection.

**NUMA.** If the producer thread runs on NUMA node 0 and the consumer on NUMA node 1, the ring buffer's physical memory is "remote" to one of them — each access pays a ~100 ns NUMA penalty. For true mechanical sympathy, producer and consumer should be pinned to the same NUMA node, or the buffer should be allocated node-locally.

**Does not help with algorithmic work.** The Disruptor is fast at *transfer*. If your event handler does expensive processing, the ring buffer itself is not the bottleneck. It only removes the inter-thread transfer overhead.

---

## Why it works

The deeper principle: **lock-free is not the same as cache-friendly, and cache-friendly is what makes things fast**.

The standard mental model of concurrency performance is: locks are slow because they cause OS context switches. The fix is lock-free data structures using CAS. This model is correct but incomplete. Even a CAS-based queue is slow if its hot fields share cache lines across threads, because the CPU's coherence protocol — not the OS scheduler — is doing the coordination, and it operates at ~200 ns per round trip between cores rather than the ~20 ns of an L1 hit.

False sharing is the hardware-layer equivalent of a lock: an invisible serialization point imposed by the CPU's cache coherence protocol. The Disruptor defeats it with a single technique — padding — and then builds the rest of the design around that insight: pre-allocate (fixed layout), use counters not pointers (single hot variable per thread), batch reads (fewer coherence round trips).

This same principle explains a cluster of other high-performance designs:

| System | Pattern | Why |
|--------|---------|-----|
| **DPDK** | Per-core packet ring buffers | Each NIC queue is owned by exactly one CPU core; no cross-core coherence traffic |
| **io_uring** | Shared ring between kernel and user space | Avoids syscall overhead; kernel and user space coordinate through shared atomic sequence numbers (exactly the Disruptor pattern) |
| **Linux kernel per-CPU variables** | `DEFINE_PER_CPU(type, var)` | One copy per physical CPU; reads and writes have zero coherence cost |
| **Column-oriented storage** | Pack same-field values together | CPU prefetcher reads them as a unit; sequential SIMD scans stay L1/L2-hot |
| **Arena allocators** | Bump-pointer allocations into a slab | All allocations in one thread are co-located; no false sharing with other threads' allocations |

The unifying theme is Martin Thompson's **mechanical sympathy**: "You don't have to be an engineer to be a racing driver, but you do have to have mechanical sympathy." The analogy to software: you don't have to know how CPUs work to write programs, but the highest-performance programs are written by people who do — and who design their data layout, access patterns, and communication protocols around the CPU's actual behavior rather than an abstraction layer that papers over it.

The "X is just Y" insight: **the Disruptor is what you get when you design a concurrent queue for the CPU's memory model instead of for the Java threading model**. The Java model says: protect shared state with locks or CAS. The CPU model says: keep each thread's hot mutable state in its own cache line, make reads and writes sequential, and avoid any state that two cores need to write to concurrently. The Disruptor satisfies the CPU model; most queues satisfy only the Java model.

---

## Going deeper

1. **LMAX Disruptor white paper**: https://lmax-exchange.github.io/disruptor/disruptor.html — the original technical document by Martin Thompson et al., covering the full design rationale, performance benchmarks, and the dependency graph model.

2. **Martin Thompson's "Mechanical Sympathy" blog** (specifically the False Sharing post): https://mechanical-sympathy.blogspot.com/2011/07/false-sharing.html — the foundational write-up of why false sharing exists, how to measure it with `perf`, and how to fix it with padding.

3. **Trisha Gee's "Dissecting the Disruptor" series**: https://trishagee.com/2011/07/22/dissecting_the_disruptor_why_its_so_fast_part_two__magic_cache_line_padding/ — a multi-part walkthrough of the Disruptor source code that shows exactly where padding is applied and why each choice was made.
