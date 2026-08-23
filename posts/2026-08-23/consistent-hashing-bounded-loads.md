---
title: Consistent Hashing with Bounded Loads
source: https://research.google/blog/consistent-hashing-with-bounded-loads/
author: Vahab Mirrokni, Mikkel Thorup, Morteza Zadimoghaddam
company: Google
date_posted: 2017-02-07
date_digested: 2026-08-23
---

# Consistent Hashing with Bounded Loads

## What's new to learn

1. **Bounded-load consistent hashing**: a one-parameter extension (ε > 0) that hard-caps the maximum load on any server to (1+ε) × the average — a guarantee vanilla consistent hashing cannot make.
2. **Ring-probing is linear probing**: walking the ring clockwise past over-capacity servers is structurally identical to linear probing in an open-address hash table — the same mechanism, the same expected probe lengths, the same theoretical guarantees.
3. **"Cap the bin, overflow to the neighbor"**: a universal load-balancing primitive that appears in hash tables, work-stealing schedulers, cache associativity, and connection pools — bounded-load hashing is just this pattern applied to a consistent-hash ring.

## Prerequisites

- Consistent hashing basics: items are mapped to a ring, servers own arcs of the ring, each item routes to the nearest clockwise server. The Dynamo post in this archive covers this.
- Balls-into-bins probability: placing m balls uniformly at random into n bins gives a maximum load of Θ(log n / log log n) × average — even with many virtual nodes.
- Open-address hash tables: what "linear probing" means and why expected probe length is O(1/(1−α)) when load factor α < 1.

## The core idea

Standard consistent hashing routes each item to the nearest clockwise server on the ring. Virtual nodes smooth the expected distribution, but the underlying process is still balls into bins: with high probability, the heaviest-loaded server has Θ(log n / log log n) times the average load, a factor that grows with cluster size.

Bounded-load consistent hashing adds one rule and one constant:

> **Each server has a capacity of ⌈(1+ε) × m/n⌉ for any ε > 0, where m is the total number of items and n is the number of servers. When routing item i, walk clockwise from hash(i) and assign item i to the first server whose current load is below capacity.**

The result: no server is ever loaded above (1+ε) times the average — not probabilistically, but as an invariant. Set ε = 0.25 and the worst server is at most 25% more loaded than average, no matter how bad the hash collisions.

## Mechanics

**Capacity**: each server s maintains a counter load(s) (number of items assigned). The threshold is c = ⌈(1+ε) × m/n⌉. Total system capacity n × c ≥ (1+ε) × m, so there is always at least one non-full server.

**Routing (new item i)**:
1. Compute position p = hash(i) on the ring.
2. Find the nearest clockwise server s₀ at p.
3. If load(s₀) < c, assign i to s₀. Done.
4. Else try s₁ (next clockwise), s₂, ... until finding server sₖ with load(sₖ) < c.
5. Assign i to sₖ, increment load(sₖ).

**Expected walk length**: on average, each server visited has some probability of being non-full. Since total used capacity is m and total capacity is (1+ε)m, a fraction at least ε/(1+ε) of servers have remaining room in expectation. Each server in the walk is non-full with that probability (modulo correlations), so the expected number of hops is at most (1+ε)/ε = O(1/ε). For ε = 0.25, expected hops ≤ 5. The walk is geometrically fast.

**Load bound — why it holds**: suppose some server s has load(s) = c (full). The algorithm only assigns to s when load(s) < c, so s can only reach exactly c items. For s to be over capacity we would need total assigned items to exceed n × c ≥ (1+ε)m — a contradiction since there are only m items.

**Migration on server join/leave**: when a new server t joins, only items that were walking past t's position (because their natural home was at capacity) will move to t. The expected number of items that move is O(m/n), matching the O(m/n) migration guarantee of vanilla consistent hashing. Formally, the paper proves expected total migrations over a sequence of joins/leaves is O(1/ε²) per operation, amortized.

**Dynamic load (request routing)**: the same algorithm applies when "items" are in-flight requests rather than stored objects. Here m grows and shrinks in real time. In practice, track load with an approximate counter (gossip, local sampling) to avoid per-route synchronous RPCs. Slightly stale load readings cause occasional over-fills that quickly correct themselves as items complete.

**Implementation in a load balancer**: the router maintains a ring data structure (O(log n) lookup for the nearest server) and a load counter per server. Routing = ring lookup + O(1/ε) additional ring walks on average, all in memory. This is used in Google Cloud Pub/Sub for subscription-to-server assignment, and Vimeo implemented it in HAProxy to route cache requests — reducing cache bandwidth by 8× by eliminating hot-cache servers.

## Where it breaks

**Requires visible load state**: routing requires knowing each candidate server's current load, or a close approximation. Exact synchronous load queries add a round-trip to every placement decision. Gossip-based approximations introduce lag, which causes temporary over-capacity and necessitates a larger ε as headroom. Pure stateless routing (hash only, no load state) cannot implement this algorithm.

**Cache affinity breaks under load**: in a CDN-style cache, the same URL should land on the same server (cache hit). But if server s₀ is full when URL u arrives, u goes to s₁. Later, when load on s₀ drops, the next request for u still hashes to s₀ and may return a cache miss if u isn't there yet. Bounded-load hashing is optimized for load balance, not cache reuse — these two objectives pull in opposite directions.

**Item weight heterogeneity**: the paper assumes each item has unit weight. In a cache storing objects from 1 KB to 1 GB, "count of items" is a poor load metric; "bytes stored" matters more. Implementing bounded-load with variable item weights requires tracking total bytes per server, and the capacity formula becomes (1+ε) × total_bytes / n. The walk terminates once a server has remaining byte capacity, but predicting future item sizes requires metadata that may not be available at routing time.

**Burst cascades**: if a request spike arrives and fills server s₀ instantly, the entire arc's traffic gets pushed clockwise to s₁, s₂, …, potentially overloading them in sequence before the system catches up. This "wave" behavior is bounded by O(1/ε) hops but can cause elevated latency spikes for ε < 0.1.

**ε is sensitive**: too small → walks are longer, more migrations per server change. Too large → some servers are substantially overloaded before the algorithm kicks in. The right ε depends on cluster size, expected load variance, and acceptable migration rate — it is not auto-tuning.

**No built-in hot-key handling**: a single extremely popular item always lands on its assigned server (it has no concept of "move the hot item away"). Bounded-load balances the *count* of items per server, not the request rate per item. A viral cache hit will still hammer the one server holding that item; you need separate popularity-based replication (like consistent hashing + replica placement) for this.

## Why it works

The deepest insight: **consistent hashing with bounded loads is linear probing on a ring**.

In an open-address hash table with linear probing:
- Hash key k to slot h(k).
- If h(k) is occupied, try h(k)+1, h(k)+2, … until an empty slot.
- At load factor α, expected probe length is O(1/(1−α)).

In bounded-load consistent hashing:
- Hash item i to ring position h(i).
- If the nearest server is at capacity, try the next server clockwise.
- At "ring load factor" 1/(1+ε), expected probe length is O((1+ε)/ε) = O(1/ε).

The mapping is exact. The "ring position" corresponds to the hash slot; the "server" corresponds to a hash bucket; "at capacity" corresponds to "occupied". The geometric distribution governing probe length in linear probing is the same distribution governing hop count in bounded-load consistent hashing.

This equivalence explains the migration bound too. In linear probing, inserting a key moves it only within a cluster of consecutive occupied slots — the cluster doesn't drift unboundedly. Analogously, a new server on the ring can only receive items from the nearby arc, bounding disruption.

The broader pattern is **capacity-capped probing with overflow to neighbors**:

| System | "Full bin" | "Walk to neighbor" | Guarantee |
|---|---|---|---|
| Open-address hash table | slot occupied | try next slot | O(1) probe length at α < 1 |
| Work-stealing scheduler | local deque full | steal from random peer | O(1) expected idle time |
| CPU set-associative cache | cache set full | evict LRU in set | O(associativity) conflict misses |
| DB connection pool | pool at max | queue and wait | bounded wait time |
| Consistent hashing (this) | server at (1+ε)×avg | walk ring clockwise | max load ≤ (1+ε)×avg |

Each row is the same algorithm: probe the natural location, cap it, overflow deterministically to the next candidate. The cap converts a probabilistic guarantee ("with high probability, max load is Θ(log n)") into a deterministic invariant ("always, max load ≤ (1+ε)×avg"). The cost is O(1/ε) extra work per placement — an ε-controlled tradeoff between load balance and placement cost.

The critical property that makes this work on a ring rather than an array is **consistent displacement**: items near the boundary of an overloaded arc are all displaced in the same clockwise direction. This means a new server at position p can "absorb" displaced items from [p−arc, p] without disturbing items beyond that arc — preserving the locality property of consistent hashing while bounding load.

## Going deeper

1. **The arxiv paper** (full proofs and formal bounds): Mirrokni, Thorup, Zadimoghaddam (2016) — https://arxiv.org/abs/1608.01350
2. **Original consistent hashing** (the ring construction this extends): Karger et al., "Consistent Hashing and Random Trees," ACM STOC 1997 — https://dl.acm.org/doi/10.1145/258533.258660
3. **"The Power of Two Choices in Randomized Load Balancing"** (already in this archive, 2026-07-27) — the complementary result: sampling two random servers and picking the lighter gives O(log log n) max load vs. O(log n). Bounded-loads consistent hashing is a capacity-enforcement approach; power-of-two-choices is a sample-and-compare approach — both exponentially improve over one random choice.
