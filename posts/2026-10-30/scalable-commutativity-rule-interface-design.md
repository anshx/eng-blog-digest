---
title: "The Scalable Commutativity Rule: Designing Scalable Software for Multicore Processors"
source: https://dl.acm.org/doi/10.1145/2699681
author: Austin T. Clements, M. Frans Kaashoek, Nickolai Zeldovich, Robert T. Morris, Eddie Kohler
company: MIT CSAIL
date_posted: 2015-11-01
date_digested: 2026-10-30
---

# The Scalable Commutativity Rule: Designing Scalable Software for Multicore Processors

## What's new to learn

- **SIM commutativity** — a formal definition of when two interface operations commute "in some history": neither caller can distinguish which ran first, under any reachable state at the time of the call. The "State-Invocation-Model" framing extends classical algebraic commutativity to stateful, history-dependent interfaces like POSIX.
- **The Scalable Commutativity Rule (SCR)** — the theorem: whenever interface operations commute (in some history), there exists an implementation that is *conflict-free* (generates no cache-to-cache invalidation traffic) for those operations. Equivalently, non-commutative operations are structurally unscalable; no implementation can fix that without changing the interface.
- **COMMUTER** — a tool that takes a symbolic Python-like interface model, uses symbolic execution + an SMT solver to enumerate all (state, operation-pair) combinations that commute, and then generates concrete test cases that detect whether an implementation actually *exploits* that commutativity.

## Prerequisites

- **Cache coherence** — why writing to a cache line on core A forces an invalidation message to every other core that has a copy (MESI protocol). Shared writes serialize at the cache-line granularity, imposing cross-core synchronization costs even without a single lock.
- **Scalability** — a system is *scalable* for an operation if its throughput grows linearly with core count; it is *not scalable* if adding cores yields diminishing returns because cores contend on shared state.
- **System call semantics** — POSIX specifies what system calls return, not how they are implemented. The subtlety is that some POSIX specifications mandate specific *return values* (e.g., `open` returns the lowest available file descriptor) that inadvertently force ordering between concurrent calls.

## The core idea

Think of your interface specification as a ceiling on your implementation's scalability. If the specification requires two operations to be distinguishable by order — if swapping them could produce different return values or different observable side effects — then any correct implementation must somehow communicate which ran first. That communication costs synchronization, whether it is a lock, a memory fence, or a cache-line invalidation.

The Scalable Commutativity Rule turns this around:

> **Whenever operations commute in some history, there exists an implementation that is conflict-free (scalable) for those operations in that history.**

"Conflict-free" is precise: the implementation touches no memory word that another core reads or writes simultaneously. Zero cache sharing → zero coherence traffic → linear throughput scaling.

The contrapositive is equally sharp: **if your API is non-commutative, no implementation is scalable**. You cannot retrofit scalability after the fact if the specification mandates ordering.

This is a design-time insight. It gives engineers a pre-ship checklist: before you publish an API, ask "do the performance-critical operations commute?" If not — and you care about multicore scaling — you need to relax the specification, not optimize the implementation.

## Mechanics

### SIM commutativity

Classical algebra says two functions commute if `f ∘ g = g ∘ f`. For stateful interfaces this is too strong: `fork()` obviously does not commute with itself in *all* states, but it may commute in many specific states (both forks succeeding, producing distinct children).

SIM commutativity asks: is there *some* history H (some sequence of prior calls, some input state) in which operations A and B commute? If yes, the rule applies in that history, and a scalable implementation must exist for those states.

This conditional framing is crucial: the rule does not say "all operations must commute everywhere." It says "wherever they commute, you can be scalable there." An implementation can be scalable in the common case (commutative states) even if it must serialize in the edge case (non-commutative states).

### The COMMUTER pipeline

1. **Model** — the engineer writes a small (~200-line) Python model of the interface. State is symbolic (via Rosette/Z3 SMT), operations are functions over that state.
2. **Analyze** — COMMUTER runs symbolic execution to find all (state, op₁, op₂, inputs) tuples where op₁ and op₂ commute: the return values and final state are identical regardless of execution order.
3. **Generate** — for each commutative case, COMMUTER emits a concrete test: run op₁ and op₂ on two separate cores simultaneously, then compare results and measure whether any cache lines were shared between the two cores (via hardware performance counters tracking LLC misses attributable to cross-core sharing).
4. **Verify** — if the test detects cache sharing, the implementation is not exploiting the available commutativity — a missed opportunity for scaling.

For 18 POSIX file-system and virtual-memory operations, COMMUTER generated **13,664 tests** (SOSP 2013 version; expanded to 26,238 in the TOCS 2015 journal paper). Running these against two kernels:

| Kernel | Tests passed (scaled) |
|--------|----------------------|
| Linux  | 68%                  |
| sv6    | 99%                  |

### What Linux gets wrong (and why)

Three examples illustrate the gap:

**`open()` — the lowest-fd contract.**  
POSIX mandates that `open()` returns the lowest-numbered available file descriptor. If two threads simultaneously open files in the same process, they must coordinate to determine which fd is "lower." Linux's fd allocation is a bitset scan under a per-file-table lock: inherently sequential. The API specification — not the implementation — causes this. A hypothetical `open()` that returns *any* available fd (like `openat()` into a preallocated pool) would commute and be trivially scalable.

**`stat()` — the shared timestamp.**  
POSIX `stat()` returns `st_atime` (last access time), updated on every file read. Any two concurrent reads of the same file fight over writing `st_atime` into the inode, even though neither is modifying file content. Linux must write-lock the inode for every read. sv6 avoids this by making `st_atime` updates lazy and approximate (or omitting them where the application allows), aligning with a later POSIX relaxation that permits implementations to omit `st_atime` updates.

**File creation — global inode numbering.**  
Creating two files in the same directory requires assigning two distinct inode numbers. Linux draws from a *global* inode counter, serializing all file creations cluster-wide. sv6 uses per-directory inode namespaces: inode numbers are only unique within a directory, so two directories can allocate concurrently without coordination.

### sv6: the reference scalable kernel

sv6 was designed specifically to pass COMMUTER tests. Its design choices are direct consequences of the rule:

- **fd allocation**: returns any free fd from a per-thread freelist, not the lowest globally.
- **Inode numbers**: per-directory allocators; inodes are (directory, local-id) pairs.
- **mmap**: per-core virtual address space bookkeeping, avoiding a global VMA tree.
- **Page allocation**: per-core page caches eliminate the global buddy allocator lock.

On a mail-server benchmark (spawning many short-lived processes, creating files), sv6 achieves near-linear throughput growth as core count increases; Linux plateaus at a fraction of theoretical peak.

## Where it breaks

**Inherently non-commutative interfaces.** Some specifications are non-commutative by design, and the interface cannot be changed (e.g., a standard the system must comply with). POSIX's lowest-fd guarantee for `open()` is this: any compliant implementation must serialize. If you must comply, the rule says you simply cannot scale that operation — and it is telling you something true.

**Sufficient, not necessary.** The rule identifies a *sufficient* condition for scalability to exist: commutativity ⟹ scalable implementation exists. It is not necessary: some non-commutative operations can still be made nearly-scalable in practice via combining networks, hardware transactional memory, or log-structured designs. The rule sets a guaranteed ceiling, not an absolute one.

**Small operation sets.** COMMUTER exhausts over pairs and triples of operations; the combinatorial space explodes for larger groups. Real applications invoke many calls in complex interleavings, some of which COMMUTER does not model.

**Application-level sharing.** If an application deliberately shares state (two threads updating the same counter), no OS interface design can fix that; the contention is in user code, not the kernel API.

**Model accuracy.** The COMMUTER model must faithfully capture the interface contract. If the model is simpler than the actual spec (e.g., omits error paths), it may deem operations commutative when the real spec does not.

## Why it works

The rule's correctness rests on a clean information-theoretic argument.

Commutativity means no caller can distinguish execution order. If no caller can tell which operation ran first, then the two operations cannot be communicating via the interface. But synchronization *is* communication: a lock, a fence, or a cache invalidation all convey "I happened before you." If there is no observable ordering to communicate, there is no need for communication — and therefore no need for synchronization.

More formally: if operations A and B commute in history H, construct an implementation where A and B are routed to separate per-core data structures (e.g., per-core logs or per-core subtrees). A reads and writes only core₁'s structures; B reads and writes only core₂'s structures. Since the caller cannot distinguish execution order, any later "merge" operation that reconciles the two structures can produce a result consistent with either ordering — and it will be correct.

This is the same insight as the **CALM theorem** (Consistency As Logical Monotonicity, covered in this archive), but applied one level down the stack:
- CALM says: if a distributed computation is *monotone* (adding more facts never retracts prior results), it needs no coordination across network partitions.
- SCR says: if a shared-memory interface operation is *commutative* (adding more operations cannot expose their order), it needs no coordination across cache lines.

Both are instances of "algebraic order-independence ⟹ implementation freedom from synchronization." The difference is the medium: CALM targets network round-trips; SCR targets cache-coherence traffic.

The connection to **CRDTs** is also direct: a CRDT is a data structure where update operations are designed to commute (and be associative and idempotent), so replicas can apply updates in any order and converge. The SCR paper is asking the same question at the OS API layer: can we redesign system call semantics so that concurrent calls commute, enabling OS implementations that behave like CRDTs inside the kernel?

## Going deeper

1. **The SOSP 2013 version of the paper** (the conference original, slightly shorter): [clements13scalable.pdf](https://read.seas.harvard.edu/~kohler/pubs/clements13scalable.pdf) — accessible and well-paced.
2. **Austin Clements' PhD dissertation** (2014) — extends the proof, adds more POSIX operations, and covers the sv6 kernel design in depth: https://pdos.csail.mit.edu/papers/aclements-phd.pdf
3. **A Revised and Verified Proof of the SCR** (Tsai, Leners, Mu, Eras, 2018) — the original SOSP proof had a subtle gap; this Coq-verified proof closes it: https://arxiv.org/abs/1809.09550
