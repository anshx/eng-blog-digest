---
title: "TAO: Facebook's Distributed Data Store for the Social Graph"
source: https://www.usenix.org/conference/atc13/technical-sessions/presentation/bronson
author: Nathan Bronson, Zach Amsden, George Cabrera, Prasad Chakka, et al.
company: Facebook (Meta)
date_posted: 2013-06-25
date_digested: 2026-08-14
---

# TAO: Facebook's Distributed Data Store for the Social Graph

## What's new to learn

1. **Query-distribution-driven API design**: measure your actual production query mix first, then design a data model and API that perfectly fits the top 99%—and deliberately exclude the rest. Generality is the enemy of cache efficiency.

2. **The leader/follower cache hierarchy as a graph-aware CDN**: two tiers of caches (leaders act as origin servers; followers act as edge nodes) give you write serialization, cache miss coalescing, and horizontally scalable reads—without touching the underlying database.

3. **Embedded shard routing in IDs**: encoding the shard number directly in the high bits of every 64-bit object ID means routing decisions are O(1) bit-masking—no lookup table, no coordinator, no indirection.

## Prerequisites

- **Memcached / lookaside caching**: the "cache in front of MySQL" pattern TAO replaces.
- **Consistent hashing**: why naive sharding works, and why TAO deliberately avoids it in favor of lifetime-bound shard assignment.
- **Eventual consistency**: what it means in practice—reads may return stale data for a bounded window after a write.
- **Database replication**: master/slave MySQL, async replication lag, and why cross-region writes are expensive.

## The core idea

By 2009, Facebook's PHP application layer had grown into an enormous tangle of Memcache lookups and MySQL queries to serve the social graph. The problem wasn't raw throughput—Memcache handled that. The problem was that **Memcache knows nothing about the data model**. Invalidating a cached association list after a "like" required the application to know which cache keys to bust, and with hundreds of services reading and writing the same social data, that logic proliferated everywhere and broke constantly.

TAO's answer: **move the social-graph data model into the cache layer itself**. Build a cache that understands objects and association lists natively, so that when a write happens, the cache—not the application—figures out what to invalidate. The result is a system that simultaneously:

- Serves ~500× more reads than writes from in-memory cache,
- Coalesces concurrent cache misses so the database sees one query regardless of how many clients miss simultaneously,
- Propagates invalidations to multiple cache tiers without application involvement, and
- Scales reads horizontally by adding follower tiers while keeping writes funneled through a small leader tier.

The mental model: **TAO is a CDN applied to a graph database**. The leader tier is the origin; follower tiers are edge nodes; MySQL is the object store behind the origin; and invalidation messages are cache-purge signals. The twist is that the CDN understands your object types and edge types, so it invalidates with surgical precision rather than blunt URL-pattern matching.

## Mechanics

### The data model

TAO exposes two primitives:

**Objects** are typed nodes. Each has:
- A globally unique 64-bit ID (the shard number is embedded in the high bits)
- A 32-bit type (mapped to a string name like `User`, `Page`, `Comment`)
- An arbitrary key→value blob (up to ~1 MB; stored as a MySQL row)

**Associations** are typed directed edges. Each has the schema:

```
(id1: u64, atype: u32, id2: u64) → { time: u32, data: bytes }
```

Edges within one `(id1, atype)` pair are stored sorted by descending `time`, forming a *association list* (like "all the people who liked post P, sorted by recency").

The query API is intentionally tiny:

| Operation | Description |
|-----------|-------------|
| `obj_get(id)` | fetch object by ID |
| `assoc_get(id1, atype, id2_set)` | fetch specific edges |
| `assoc_count(id1, atype)` | edge count (cached separately) |
| `assoc_range(id1, atype, pos, limit)` | get slice of list by position |
| `assoc_time_range(id1, atype, t_low, t_high, limit)` | get slice by time window |
| `assoc_add / assoc_delete / assoc_change_type` | write operations |

That's it. No joins. No arbitrary predicates. No multi-hop traversal. If your query doesn't fit this API, you use offline batch systems (Hive, MapReduce).

**Why such a small API?** Facebook analyzed millions of production MySQL queries and found that **99.8% were either single-object lookups or association list reads**. The remaining 0.2%—complex joins, aggregations, arbitrary traversals—went to batch pipelines anyway. The restricted API is what makes the cache layer possible.

### The cache hierarchy

Within each geographic region, TAO has two cache tiers:

```
 Clients
    │
    ▼
┌──────────────────────────────┐
│   Follower Tier (many)       │  ← serves most reads from local RAM
│  (tier 1, tier 2, tier 3…)   │
└──────────┬───────────────────┘
           │ on miss or write
           ▼
┌──────────────────────────────┐
│   Leader Tier (one per rgn)  │  ← serializes writes, queries MySQL
└──────────┬───────────────────┘
           │ on miss
           ▼
┌──────────────────────────────┐
│   MySQL Shards               │  ← durable storage
└──────────────────────────────┘
```

**Leaders**:
- One leader server "owns" each shard in a region.
- All writes go through the leader: `follower → leader → MySQL`.
- After a write commits, the leader sends async **invalidation** messages (or full **refill** messages for small objects) to all follower tiers that cached that shard.
- The leader also **serializes concurrent writes** to the same association list, preventing the race condition where two simultaneous "like" writes produce an incorrect count.

**Followers**:
- Many follower tiers, each containing a full set of TAO servers covering all shards.
- Cache hits are served locally (O(1), no network hop).
- Cache misses are forwarded to the leader, which may in turn query MySQL.
- Followers never talk directly to MySQL.

**Why multiple follower tiers, not just one?** Facebook adds follower tiers to a region to absorb read load without adding write pressure to the leader. Each tier independently caches the same data with its own LRU eviction. Clients are assigned to a tier deterministically (by client ID hash), so the same client always hits the same follower tier first—locality for caching.

### Write path

```
1. Client sends assoc_add(id1=P, atype=like, id2=U) to local follower
2. Follower forwards to leader (P's shard owner)
3. Leader writes to MySQL (UPDATE assoc_count + INSERT into assoc table)
4. Leader sends INVAL(P, like) to all follower tiers asynchronously
5. Leader ACKs to follower → follower ACKs to client
```

Steps 4 and 5 can proceed in parallel. The client gets the ack before all followers have invalidated—this is the source of potential stale reads.

### Read path (cache hit)

```
1. Client → follower (assoc_count(P, like))
2. Follower → client (from in-memory cache) ← 99%+ of reads
```

### Read path (cache miss)

```
1. Client → follower (miss)
2. Follower → leader (miss forwarded)
3. Leader → MySQL (SELECT COUNT(*) WHERE id1=P AND atype=like)
4. Leader stores result in leader cache; sends REFILL to originating follower
5. Leader → follower → client
```

**Miss coalescing**: if 500 followers all miss on `assoc_count(P, like)` simultaneously (e.g., a viral post), the leader issues exactly **one** MySQL query and fans out the result. This prevents thundering-herd cache miss storms.

### Shard assignment and ID encoding

Every object's ID has this layout:

```
 63          48  47    0
 ┌────────────┬─────────┐
 │  shard_id  │ seq_num │
 └────────────┴─────────┘
```

The shard ID is embedded at object-creation time and **never changes**. This means:
- Routing to the right MySQL shard or cache server is a single bit-mask operation—no lookup table.
- An object and all of its associations (which are stored on the same shard) always live together.
- There is no object migration, ever.

### Geographic distribution

TAO runs in multiple regions worldwide:

- One **master region** accepts all writes (mutations are applied to the master MySQL cluster).
- **Slave regions** replicate from master MySQL asynchronously. They serve reads locally and forward writes to the master region's leader tier.

Cross-region write path:
```
Client (EU) → EU follower → EU leader → US master leader → US MySQL
                               ↑
                        async replication
                               ↓
                        EU MySQL ← US MySQL
                               ↓
                        EU leader sends INVALs to EU followers
```

This means a user in Europe who likes a post will briefly see their own action reflected before the replication arrives, because TAO does a **local optimistic update** in the follower cache immediately on the write path. If replication lag is 200 ms, most users never notice.

### Hot shard mitigation: shard cloning

If a celebrity's post goes viral, its shard becomes a hotspot on the leader. TAO handles this by **shard cloning**: replicating the leader state for that shard to multiple leader machines. Followers then round-robin (or randomly pick) which leader replica to send cache misses to. Writes still go to all leader replicas (with simple two-phase-like broadcast), but read-miss load is spread across them.

## Where it breaks

**Eventual consistency across tiers**: A follower cache may serve stale data for the invalidation propagation window (typically milliseconds, occasionally seconds under high load). Facebook's acceptance criterion is "social tolerance"—seeing a like from 5 seconds ago is fine. TAO does not provide read-your-own-writes guarantees across different follower tiers.

**Association list truncation**: TAO only caches the first N associations per (id1, atype) pair (top by recency). If a user has 500 000 followers, TAO may only cache the most recent 5 000. Deep pagination falls through to MySQL and is expensive—by design, Facebook's UX is engineered to avoid it.

**Cross-shard transactions**: TAO has no distributed transactions. A write that touches two objects in different shards (e.g., adding an association and updating a counter on a separate object) is issued as two independent writes. Application code must handle partial failures.

**Write hotspots at the leader**: A single leader server serializes all writes to its shards. If a shard's write rate exceeds one server's capacity, the only option is shard splitting (which requires ID migration, so Facebook avoids it). Shard cloning helps for reads but not for writes.

**The restricted API is the restriction**: if your access pattern doesn't fit `obj_get`, `assoc_range`, or `assoc_count`, you don't use TAO—you wait for a batch pipeline. Real-time complex graph traversal (e.g., "friends of friends who also like X") is not achievable with this API.

## Why it works

The deep principle is: **match your abstraction to the invariants of your workload, not to the generality of a data model**.

Facebook discovered that social-graph reads have a structural property: they are overwhelmingly *local* in the graph. Users look at one profile (object lookup), one post's likes (association list), one comment thread (list of associations from a single node). Multi-hop traversal—"who are my friends' friends?"—is a tiny fraction of production reads, and those go to batch systems.

This locality property is what makes the leader/follower hierarchy work. Because 99.8% of reads can be answered by looking at one `(id1, atype)` pair or one object ID, **each cache entry is self-contained**. You can cache it, invalidate it, and refill it without knowing about any other cache entry. This is exactly the property CDNs exploit: cacheable URLs that can be independently invalidated.

If the access pattern were random graph traversal, no cache hierarchy would help—each traversal would touch many nodes in unpredictable order, and cache hit rates would collapse. TAO works because **social UX patterns impose locality on graph access patterns**.

The embedded-shard-ID trick is the same insight as consistent hashing, taken one step further: instead of a hash ring that can migrate objects as nodes join/leave, TAO bakes the shard assignment into the ID at creation time and never changes it. This sacrifices rebalancing flexibility for zero-cost routing and simpler operational semantics.

The corollary principle, which recurs throughout systems design:

> Restricting the query API to what you actually need is not a limitation—it's an enabler. Every excluded query type is one fewer thing the cache must handle correctly.

DNS does this (name → address, no reverse lookup in the core protocol). Redis does this (specific data structures with O(1) ops, no arbitrary predicates). TAO does this (objects + association lists, no joins). The restriction is the design.

## Going deeper

1. **"Scaling Memcache at Facebook" (NSDI 2013)** — the predecessor to TAO, describing how Facebook scaled Memcache to millions of QPS *before* building a graph-aware layer. Reading it alongside TAO shows exactly which problems TAO was built to solve: https://www.usenix.org/conference/nsdi13/technical-sessions/paper/nishtala

2. **Zanzibar: Google's Consistent, Global Authorization System (USENIX ATC 2019)** — uses a similar (object, relation, user) edge model for access control, but adds a causality token (zookie) for read-your-own-writes guarantees. Shows what you must pay in complexity when "social tolerance" is not acceptable. (Covered in this archive: 2026-06-02.)

3. **"Unicorn: A System for Searching the Social Graph" (VLDB 2013)** — Facebook's real-time graph search engine built on top of TAO's data. Shows the next layer up: inverted indexes over association lists for "people named John who live in Seattle and work at Facebook."
