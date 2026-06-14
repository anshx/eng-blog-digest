---
title: "mimalloc: Free List Sharding in Action"
source: https://www.microsoft.com/en-us/research/blog/mimalloc-a-high-performance-scalable-memory-allocator-for-the-modern-era/
author: Daan Leijen, Ben Zorn, Leonardo de Moura
company: Microsoft Research
date_posted: 2019-06-01
date_digested: 2026-06-14
---

# mimalloc: Free List Sharding in Action

## What's new to learn

1. **Free list sharding** — instead of one global free list per size class (which serializes all malloc/free under a lock), maintain thousands of tiny per-page free lists so threads rarely contend; the probability of two threads hitting the same page simultaneously is low by the same argument that makes hash bucketing work.

2. **Three-tier deallocation routing** — each 64 KiB page carries three distinct free lists (`free`, `local_free`, `thread_free`) with strictly decreasing synchronization cost; every deallocation is routed to the cheapest tier it's eligible for, making the common case (same-thread free) branch-free and lock-free.

3. **Segment abandonment / page stealing** — when a thread's segment becomes mostly empty (or the thread exits), the segment is published to a global "abandoned" list and any thread that next needs a page can adopt it, transferring ownership with a single CAS — work-stealing applied to memory rather than to tasks.

## Prerequisites

- How `malloc` works at a basic level: size-class bucketing (small objects go in fixed-size bins), the OS gives the allocator large chunks via `mmap`/`sbrk`, the allocator sub-divides them.
- What a **free list** is: a singly-linked list threaded through the payload bytes of freed blocks.
- What a **CAS** (compare-and-swap) is: an atomic "if mem == expected { mem = new; return true }" instruction — the primitive for lock-free programming.
- What **false sharing** is: two threads writing to independent variables that happen to share a CPU cache line, causing cache-coherence traffic even though the variables are logically unrelated.
- **Thread-local storage (TLS)**: a per-thread global that the runtime initializes without programmer effort.

## The core idea

The bottleneck in a multithreaded allocator is **contention on shared free lists**. Classic glibc malloc uses one lock-protected arena per group of threads; jemalloc and tcmalloc give each thread a thread-local cache but still route most cross-thread frees through a shared structure.

mimalloc's answer: push ownership all the way down to the **page level**. Each 64 KiB page belongs to exactly one thread's heap, and that thread is the only one that *allocates* from it. The key observation is that **allocation and deallocation are asymmetric workloads**:

- Allocation always happens on the owning thread → no synchronization needed.
- Same-thread deallocation (freeing something you just allocated) → also no synchronization needed.
- Cross-thread deallocation (another thread frees an object that lives in your page) → one CAS, and even this is rare in most workloads.

Rather than pessimistically protecting every operation, mimalloc models these three cases with three separate data structures, then routes each operation to the right one at the call site.

## Mechanics

### Memory hierarchy

```
heap (per thread, in TLS)
  └─ segments (4 MiB regions acquired from OS via mmap)
       └─ pages (64 KiB each; each page holds blocks of exactly one size class)
            └─ blocks (fixed-size; size determined by the page's size class)
```

A thread's **heap** is a thin TLS struct that holds pointers to active pages for each size class. **Segments** are the OS-level allocation unit (4 MiB). Each segment is sliced into **pages**, and every page is committed to one size class (e.g., "all 32-byte blocks"). This means there is zero internal fragmentation *within* a page; the only waste is at the page boundary.

Size classes are spaced to give at most ~12% internal fragmentation: 8, 10, 12, 16, 20, 24, 32, 40, 48, 64 … bytes, up to 1 KiB for "small" objects. Objects >1 KiB get their own page; objects >64 KiB bypass the page system and go directly to `mmap`.

### The three free lists

Each page carries exactly three free lists:

| List | Updated by | Sync needed? | When used |
|---|---|---|---|
| `free` | owning thread only | None | Main allocation source; popped on every `malloc` |
| `local_free` | owning thread only | None | Accumulates same-thread frees; merged into `free` when `free` is empty |
| `thread_free` | any thread | One CAS | Cross-thread frees (other thread freed an object in this page) |

**Fast-path malloc** is three machine instructions:

```c
block = page->free;           // load head of free list
page->free = block->next;     // pop
return block;
```

No lock, no atomic, no branch — just a pointer load and store. Because `free` is only ever written by the owning thread, even the store is safe without fencing.

**Same-thread free** pushes to `local_free` (also just a pointer store):

```c
block->next = page->local_free;
page->local_free = block;
```

**Cross-thread free** pushes to `thread_free` with a CAS loop:

```c
do {
    block->next = page->thread_free;
} while (!CAS(&page->thread_free, block->next, block));
```

This is the only synchronization point in the entire allocator for typical workloads.

**Slow-path malloc** (when `free` is empty):

1. Adopt `local_free` → set `free = local_free`, `local_free = NULL`. Cost: two pointer stores.
2. Collect `thread_free` → atomically swap the head of `thread_free` to NULL, prepend the collected chain to `free`. Cost: one atomic swap.
3. If both were empty, move to the next page in the size class, or allocate a fresh page from the segment.

The insight is that steps 1 and 2 are paid *once per page drain*, amortized across however many blocks that page holds (e.g., a 64 KiB page of 32-byte objects holds 2048 blocks — you pay the merge cost once every ~2048 allocations).

### Page stealing and segment abandonment

When a thread exits or when a segment's pages become mostly empty (>75% of blocks freed), mimalloc "abandons" the segment by publishing it to a lock-free global list:

```
abandoned_list ← CAS push(segment)
```

Any other thread needing a fresh page can "adopt" an abandoned segment — taking ownership with a CAS pop and assigning its pages to the adopting thread's heap. The cross-thread frees that previously landed in `thread_free` of those pages are now local to the new owner, so subsequent frees require no synchronization.

This is structurally identical to **work stealing in task schedulers**: each worker has a local deque; when it runs dry it steals from another worker's deque (or from a global queue). The unit of work here is a page (or segment) rather than a task.

### Benchmark numbers

On the `redis` benchmark (heavily concurrent, production workload):

| Allocator | Throughput vs baseline |
|---|---|
| glibc malloc | 1.0× |
| jemalloc | 1.07× faster than tcmalloc |
| tcmalloc | baseline for comparison |
| mimalloc | +7% over tcmalloc, +14% over jemalloc |

Memory overhead (ratio of OS-committed memory to live object bytes) on a 500 GiB production service:

| Allocator | Overhead |
|---|---|
| System allocator | 1.1× |
| Competing concurrent allocator | 4.0× |
| mimalloc | 1.3× |

The 4× overhead competitor achieves its speed by keeping enormous per-thread caches; mimalloc achieves similar speed with 3× less memory waste because abandoned-segment reclamation keeps memory tightly coupled to live objects.

## Where it breaks

**Cross-thread-heavy workloads**: producer threads allocate, consumer threads free. Every free hits `thread_free` with a CAS. The amortization benefit still holds (one CAS per object, then the adopting producer merges cheaply), but latency is higher than pure thread-local patterns. jemalloc's thread-caches can outperform mimalloc here if cache hit rates are high.

**Very large objects (>64 KiB)**: these bypass the page system entirely and go to `mmap` + a red-black tree for tracking. The three-list optimization doesn't apply.

**Object pinning across thread lifetime**: if thread A allocates an object that outlives thread A, the segment gets abandoned and eventually adopted by thread B. This is correct but causes a transfer of ownership that adds latency at adoption time.

**Huge thread counts with sparse segments**: with thousands of short-lived threads each touching a few pages, abandoned segments accumulate. The global abandoned list becomes a bottleneck if many threads are simultaneously trying to adopt.

**No NUMA awareness by default**: segments are allocated from whichever NUMA node the OS picks. On NUMA machines with dozens of sockets, cross-node memory access latency can dominate allocator choice. jemalloc and tcmalloc have explicit NUMA-arena options.

## Why it works

The deep principle is **synchronization routing by operation class**: rather than applying the same (expensive) synchronization to every operation, you classify operations by which parties are involved, then give each class a data structure that matches exactly the synchronization it needs and no more.

mimalloc's three free lists are a direct application:

| Operation class | Parties involved | Required synchronization | Data structure used |
|---|---|---|---|
| Allocate | owning thread only | none | `free` list (pointer load/store) |
| Free (same thread) | owning thread only | none | `local_free` (pointer load/store) |
| Free (cross thread) | owning + any other | one atomic | `thread_free` (CAS) |

This pattern recurs across systems:

- **Read-write locks**: readers can share a counter (many readers, zero writes), writers need exclusive access. Same classification idea — read ↔ `free`, write ↔ `thread_free`.
- **Staged commit in databases** (WAL + deferred apply): the fast path (sequential log write) is separated from the slow path (random B-tree page update); the merge is batched and amortized.
- **Double buffering in GPU rendering**: one buffer is being drawn to (local, no sync), the other is being consumed by the display. Swap is the single synchronization point.
- **Linux per-CPU variables** (`per_cpu`): kernel data structures keep a copy per CPU; cross-CPU mutation (the cross-thread-free equivalent) triggers an IPI (inter-processor interrupt), which is the rare slow path.

The amortization argument is also general: any design that batches many low-cost local operations and pays a single higher-cost merge at the end (lazy promotion in SIEVE, Gorilla's XOR delta-of-delta accumulation, LSM tree compaction) follows the same shape. The trick is finding a data structure where the "local" and "remote" writes can be physically separated without breaking correctness.

mimalloc makes this concrete for memory: the correctness invariant is that every freed block ends up in exactly one free list. The three-list design upholds this because each block is pushed to exactly one list on free, and the merge protocol on the allocation slow path drains the lists in strict order, guaranteeing no double-free or loss.

## Going deeper

1. **The original paper** — "Mimalloc: Free List Sharding in Action" (APLAS 2019), Leijen, Zorn, de Moura: https://www.microsoft.com/en-us/research/publication/mimalloc-free-list-sharding-in-action/ — The formal treatment proves bounded fragmentation and derives the amortized O(1) cost.

2. **Hoard: A Scalable Memory Allocator for Multithreaded Applications** (ASPLOS 2000), Emery Berger et al. — the paper that introduced per-thread heaps with "superblock" ownership transfer, the direct ancestor of mimalloc's segment abandonment. Freely available at: https://people.cs.umass.edu/~emery/pubs/berger-asplos2000.pdf

3. **jemalloc's "Arenas" design** — Jason Evans' original jemalloc paper and the FreeBSD allocator documentation explain how arenas (fixed-size pools of pages assigned round-robin to threads) solve the same problem with a slightly different tradeoff: more memory overhead but simpler ownership semantics: https://jemalloc.net/jemalloc.3.html
