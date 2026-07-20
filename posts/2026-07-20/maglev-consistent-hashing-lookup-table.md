---
title: "Maglev: A Fast and Reliable Software Network Load Balancer"
source: https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/
author: Daniel E. Eisenbud, Cheng Yi, Carlo Contavalli, Cody Smith, Roman Kononov, Eric Mann-Hielscher, Ardas Cilingiroglu, Bin Cheyney, Wentao Shang, Jinnah Dylan Hosein
company: Google
date_posted: 2016-03-21
date_digested: 2026-07-20
---

# Maglev: A Fast and Reliable Software Network Load Balancer

## What's new to learn

1. **Maglev hashing**: a lookup-table-based consistent hashing scheme where O(1) lookup time and near-perfect load balance are achieved simultaneously—unlike ring hashing, which requires trading one for the other.
2. **Permutation-filling algorithm**: each backend generates a pseudo-random permutation of table slots using modular arithmetic over a prime-sized table; a greedy round-robin fill produces a balanced, stable assignment without any coordination.
3. **Stateless connection stickiness via ECMP**: by hashing on the 5-tuple (protocol, src IP, src port, dst IP, dst port) into a shared lookup table, all Maglev machines independently map the same packet to the same backend—enabling any router to forward any packet to any Maglev without session affinity at the routing layer.

## Prerequisites

- Basic load balancing concepts: virtual IPs, health checks, backends
- Hash functions and modular arithmetic (nothing beyond `%`)
- Consistent hashing intuition: why plain mod-N assignment breaks when N changes
- TCP's 5-tuple and why a connection's packets must reach the same backend
- ECMP (Equal-Cost Multi-Path): routers can forward to multiple next hops at equal cost; packet dispatch is typically per-flow hash

## The core idea

Classical consistent hashing (Karger et al., 1997) arranges both backends and keys on a circular ring. A key's responsible backend is the first backend clockwise from the key's hash position. To improve load balance, each backend occupies V "virtual nodes"—random positions on the ring—so the average slot share is M/N but variance shrinks as V grows.

The catch: looking up a key requires a binary search over all V×N sorted positions, giving O(log(V×N)) lookup time. More virtual nodes means better balance but slower dispatch—exactly the wrong trade-off for a network fast path processing millions of packets per second.

Maglev's insight: **turn the ring into an array**. Pre-compute which backend owns each slot into a lookup table of size M (a large prime). Packet dispatch becomes:

```
backend = table[hash(5-tuple) % M]
```

That's one hash and one array index: O(1), no search. The challenge is filling that table so that:
1. Load is balanced (each backend gets ≈M/N slots)
2. Adding or removing a backend perturbs as few slots as possible
3. The fill is deterministic (every Maglev machine computes the same table from the same config)

The permutation-filling algorithm solves all three simultaneously.

## Mechanics

### System architecture

```
Client → VIP (Virtual IP)
           │
           ▼ ECMP (per-flow hash)
     ┌─────┴─────┐
  Maglev-1     Maglev-2  ...  Maglev-K
     │                           │
     └─────────┬─────────────────┘
               │ all share the same Maglev lookup table
               ▼
        Backend pool (B1, B2, ..., BN)
```

Routers use ECMP to spread incoming packets across K Maglev machines. Each Maglev machine maintains an identical lookup table (synchronized by the control plane). Because any Maglev machine maps a given 5-tuple to the same backend, ECMP can arbitrarily distribute packets without breaking per-connection stickiness. Backends respond directly to clients (Direct Server Return), bypassing the Maglev machines on the return path.

### Generating backend permutations

The table has M entries where M is a prime number chosen such that M > 100 × N (M = 65537 is used in the paper for small deployments; larger tables use primes like 655373).

For each backend b identified by a stable name (e.g., its IP address):

```
offset[b] = h₁(name(b)) mod M
skip[b]   = h₂(name(b)) mod (M − 1) + 1    # +1 ensures skip ≠ 0

permutation[b][j] = (offset[b] + j × skip[b]) mod M,  j = 0, 1, 2, …
```

Because M is prime and `1 ≤ skip < M`, every linear congruential sequence `(offset + j×skip) mod M` has period exactly M—it visits all M positions before repeating. Each backend therefore has a distinct, complete permutation of [0, M).

### The filling algorithm

```python
next = {b: 0 for b in backends}          # each backend's current index into its permutation
table = [EMPTY] * M

filled = 0
while filled < M:
    for b in backends:                    # round-robin over all backends
        c = permutation[b][next[b]]
        while table[c] != EMPTY:         # walk forward until an unclaimed slot
            next[b] += 1
            c = permutation[b][next[b]]
        table[c] = b
        next[b] += 1
        filled += 1
        if filled == M:
            break
```

The outer loop iterates until all M slots are claimed. Each backend claims its most-preferred unclaimed slot in each round. Since the permutations are pseudo-random and independent, the resulting assignment is nearly perfectly balanced: each backend gets either ⌊M/N⌋ or ⌈M/N⌉ slots, differing by at most one.

### Handling backend changes

When a backend is added or removed, the algorithm reruns with the new set. Two properties follow:

- **Stability**: Slots assigned to surviving backends don't move. Only the removed backend's M/N slots are redistributed (or a new backend claims M/N slots from others). This is mathematically equivalent to the minimal-disruption guarantee of ring consistent hashing.
- **Convergence**: All K Maglev machines arrive at the same new table independently, because the only inputs are the ordered list of backends and their names. No leader election, no gossip—just re-run the same deterministic algorithm.

### Connection tracking

Consistent hashing routes new flows deterministically, but an existing TCP connection could be disrupted if a backend disappears mid-flight. Maglev therefore maintains a separate **connection tracking table**: a hash table keyed on 5-tuple that caches (flow → backend) for active connections.

Lookup order:
1. Check the connection table. If hit, use the cached backend (even if it's no longer in the Maglev table).
2. If miss, use the Maglev lookup table; write the result into the connection table.
3. Entries expire on idle timeout.

This two-table design separates concerns: the Maglev table handles new flows with O(1) dispatch; the connection table handles the edge case of backend changes for existing flows. Because ECMP spreads packets across Maglev machines, a packet for an existing flow could land on a different Maglev than its predecessors. The connection table is therefore maintained per-machine with best-effort consistency rather than synchronization.

### Performance

The paper reports ~80 Gbps aggregate throughput per Maglev cluster with a 10 Gbps NIC per machine, processing packets in the Linux kernel (no DPDK required). The bottleneck is the NIC, not the hashing computation.

## Where it breaks

**Backend identity is name-bound**: the permutation is derived from the backend's stable name (usually its IP). If an IP changes (e.g., a server is reprovisioned), its permutation changes and M/N slots shift. This is rarely a problem in practice but means "renaming" a backend is not cheap.

**Table size is fixed at construction**: M must be chosen before knowing N. The invariant M > 100N means you need to over-provision M to handle future backend scaling. For very large backend pools (thousands of backends), even M = 655373 may become limiting.

**Hot elephant flows**: a single high-bandwidth TCP connection can't be split across backends—Maglev assigns the entire flow to one backend. For video streaming or large file transfers this means one backend can be saturated while others are idle.

**ECMP symmetry is assumed**: if the network topology changes and ECMP weights become unequal, Maglev's load distribution inherits that imbalance. Maglev only balances within the set of packets it receives.

**GRE overhead**: Maglev forwards packets to backends using GRE tunneling (or equivalent encapsulation), adding ~20 bytes per packet. For small packets (e.g., DNS), this represents significant overhead relative to payload.

## Why it works

### The deeper principle: offline precomputation collapses online search

Maglev hashing is a special case of a general pattern: **replace an O(log N) or O(N) online search with an O(1) lookup into a precomputed table, as long as the key space is bounded and the mapping is stable enough to pay the precomputation cost.**

The same pattern appears throughout computer science:
- **Perfect hash tables**: precompute a collision-free mapping for a known static key set, enabling O(1) lookup with no collisions.
- **Jump consistent hashing** (Lamping & Veach, 2014): collapses consistent hashing to 5 lines of code by exploiting the mathematical structure of the assignment problem, achieving O(ln N) time with zero memory—but only for sequentially numbered buckets.
- **Software TLBs / page table walks**: the OS precomputes virtual→physical mappings into the TLB so memory accesses are O(1) instead of O(page table depth).
- **Routing tables / FIBs**: BGP computes reachability offline and stores the result in a Forwarding Information Base so packet forwarding is a single table lookup.

In every case, the insight is: *you cannot afford to compute the answer at dispatch time, so compute it once and store it in a form that answers queries in one step.*

### The filling algorithm is a balanced assignment via greedy scheduling

The permutation-filling algorithm is equivalent to an online **bipartite matching** procedure: left vertices are M table slots, right vertices are N backends. Each round of the outer loop finds a maximum matching where each backend's degree is exactly 1. After M rounds, every slot is matched. The greedy criterion (claim most-preferred unclaimed slot) produces a balanced matching because the permutations are pseudo-randomly distributed: each backend's first ⌈M/N⌉ preferred slots are spread uniformly across [0, M), so no backend's preferred slots cluster in a way that starves others.

This connects directly to **parallel scheduling theory**: the algorithm is equivalent to the "List Scheduling" approximation for makespan minimization on identical machines. The permutations are the "job processing times," and the round-robin is the scheduler. List Scheduling on identical machines with unit jobs is optimal—which explains why the result is exactly ⌊M/N⌋ or ⌈M/N⌉ per backend.

### The meta-insight

> Maglev hashing is the "consistent hash ring" idea materialized into a fixed lookup table by solving a balanced scheduling problem. It separates **what to compute** (a balanced, stable assignment of slots to backends) from **when to compute it** (offline, at config change time), so the hot path pays only an array index.

This separation of offline precomputation from online dispatch is the same principle underlying Dremel's columnar layout (precompute repetition/definition levels offline; scan runs at query time), FlashAttention's tiling (precompute which tiles fit in SRAM; read them once at attention time), and ARIES's WAL (precompute commit decisions at log time; apply them lazily at checkpoint time).

## Going deeper

1. **"Maglev: A Fast and Reliable Software Network Load Balancer"** (Eisenbud et al., OSDI 2016) — the primary source: covers the full packet processing pipeline, the control plane design, VIP configuration, and production deployment lessons at Google scale. Available as a PDF from Google Research.

2. **"A Fast, Minimal Memory, Consistent Hash Algorithm"** (Lamping & Veach, 2014, arXiv:1406.2294) — Google's Jump Consistent Hash: 5 lines of C that achieve the same minimal-disruption property as ring hashing with zero memory, by exploiting the recursive structure of the assignment problem. A fascinating contrast to Maglev's table approach—when you can number your buckets sequentially, you don't need a table at all.

3. **"Consistent Hashing: Algorithmic Tradeoffs"** (Damian Gryski, Medium 2018) — a compact survey comparing ring hashing, rendezvous (HRW) hashing, jump consistent hashing, and Maglev hashing on load variance, memory use, disruption, and lookup time—the best single-page reference for choosing among the four.
