---
title: "Scaling Memcache at Facebook"
source: https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala
author: Rajesh Nishtala, Hans Fugal, Steven Grimm, Marc Kwiatkowski, Herman Lee, Harry Li, Ryan McElroy, Michael Paleczny, Daniel Peek, Paul Saab, David Stafford, Tony Tung, Venkat Venkataramani
company: Facebook
date_posted: 2013-04-02
date_digested: 2026-08-15
---

# Scaling Memcache at Facebook

## What's new to learn

**Lease tokens** — A 64-bit token issued on cache miss that functions as an optimistic lock for cache fill: only the lease-holder may write back the value, and the lease is invalidated if the key is deleted before the write arrives.

**Invalidation via replication stream (McSqueal)** — Instead of having application servers issue cache deletes on writes, a separate daemon subscribes to the MySQL binlog and derives cache invalidations from the replication event stream, guaranteeing completeness independent of application-layer correctness.

**Gutter pools** — A small dedicated pool of memcached servers that acts as a failure absorber: when a primary server is unreachable, clients route misses to the gutter rather than the primary's neighbors, preventing cascading overload from a single failure.

## Prerequisites

- How memcached works at a basic level: clients hash keys to servers, GET returns a value or a cache miss, SET stores a value with a TTL, DELETE removes it.
- MySQL replication: the primary writes to a binary log (binlog) and replicas tail that log to apply changes.
- The "cache-aside" or "lazy population" pattern: on a cache miss, the application fetches from the database and then stores the result in cache.
- The concept of a "thundering herd": many clients simultaneously missing the same cache key and simultaneously stampeding the database.

## The core idea

By 2013 Facebook served over a billion users from a social graph that no relational database could serve directly. The architecture layered memcached in front of MySQL everywhere, and the paper describes how they made that layer correct, efficient, and failure-tolerant at global scale.

The central insight is that **cache fill is a concurrency problem**, and the right tool for concurrency problems is **optimistic locking** — not pessimistic locks, not invalidation races, but a lightweight token mechanism that detects conflicts and resolves them cheaply.

When a key expires or is deleted, dozens of clients may simultaneously miss and try to fill it. If all of them query the database and try to SET the result, you get:
- A thundering herd (17,000 database queries per second, measured in the paper)
- A stale-set race (a client reads a stale value, the key is invalidated, then the client writes the stale value back — making the cache wrong even after the invalidation)

The lease token mechanism eliminates both problems with one primitive. On a miss, the memcached server issues one token for the key (rate-limited to one per 10 seconds). The token-holder is responsible for filling the cache. Every other concurrent miss is told to wait briefly and retry. When the token-holder calls SET, it must present the token; if the key was invalidated between the GET and SET, the token is revoked and the SET is rejected. The stale value never enters the cache.

This reduced peak database query rates from 17,000/s to 1,300/s on hot keys — a 13× reduction — from a change to a single parameter in the caching layer.

## Mechanics

### Within a cluster: clients and servers

A Facebook "cluster" is a group of memcached servers and the web servers that talk to them. Client libraries (not memcached itself) handle routing, replication, and failure handling — memcached is deliberately kept simple.

- **GET requests** go over UDP (lower overhead, fire-and-forget for most requests). UDP misses are treated as cache misses, not errors.
- **SET and DELETE** use TCP for reliability.
- Keys are distributed by consistent hashing across servers. Clients have a local map of the topology.
- When a web server needs 100 keys, it fans them out to the appropriate memcached servers in parallel — but uses a **sliding window** to avoid "incast": the client limits itself to roughly 1,000 simultaneous outgoing requests. If 100 web servers each fan out 1,000 requests to the same 100 memcached servers, the switch ports see a synchronized burst. The window breaks that synchronization.

### Lease tokens: stale sets and thundering herds

The lease mechanism lives inside the memcached server:

1. Client calls `get(key)` → miss → server creates a `(key, lease_token)` entry and returns the miss with the 64-bit token.
2. Server rate-limits: if another client `get`s the same key within 10 seconds of the last lease grant, the server returns "wait" (not a token). The client sleeps briefly and retries.
3. Token-holder fetches fresh data from the database.
4. Token-holder calls `set(key, value, lease_token)`. Server checks: is the token still valid?
   - **Token valid**: store the value, all waiting clients get the fresh result on retry.
   - **Token revoked** (because a DELETE arrived for the key in the interim): reject the SET. The token-holder re-GETs, gets a new token, and tries again.
5. On any write to the database, the application calls `delete(key)` on memcache — it does *not* set the new value. The cache is always filled lazily, never pushed.

Why delete instead of set? Deletes are idempotent and safe to apply multiple times. SET races are subtle: if two writes race, the later SET might lose to an earlier one due to network reordering, leaving a stale value. Delete-then-lazy-fill avoids the race by making the next fill always pull from the authoritative database.

### McSqueal: invalidation via the replication stream

Application servers issue deletes when they write. But they can crash, lose network connectivity, or have bugs. To guarantee completeness, Facebook runs **McSqueal** alongside every MySQL replica:

1. McSqueal subscribes to the MySQL binlog replication stream.
2. On each row mutation, it parses the SQL or row event and derives the corresponding memcache key(s).
3. It issues `delete(key)` to all memcached servers in the local cluster.
4. Because it follows the replication stream rather than the application logic, it is causally consistent: a delete from McSqueal never arrives before the corresponding write has been committed.

This is Change Data Capture (CDC) applied to cache management. The binlog is the source of truth; the cache is a derived view; McSqueal is the pipeline that keeps them aligned.

In slave (replica) regions, McSqueal replicates the invalidations from the master's changes. But there is a timing problem: after a user writes, the binlog replicates to the slave region, but there may be seconds of lag. During that window, the user's own next request might be served stale data from the local memcache — a read-your-own-writes violation.

**Remote markers** solve this: when an application server completes a write, before routing to the master MySQL, it atomically sets a "remote marker" key in the local cluster's memcache with a short TTL (say, a few seconds). Future reads for that object check for the remote marker; if found, they bypass local memcache and read from the master region directly. Once replication catches up and the marker TTL expires, reads fall back to the local cache as normal.

Remote markers are a client-carried causal token: a cheap signal that "this client has seen a write that the local replica may not yet reflect."

### Gutter pools: failure absorption

When a memcached server dies, naively the keys it owned would miss and pound the database. Redistributing those keys to the neighboring servers via consistent hashing is also wrong: the neighbors are already near capacity and would be overwhelmed.

Facebook's answer: **gutter servers**. A small pool (~1% of cluster capacity) receives requests only when the primary server for a key is unreachable:

1. Client fails to reach the primary → routes the miss to a deterministic gutter server (key hash modulo gutter pool size).
2. If not in gutter: fetch from database, store in gutter with a short TTL (~2–3 seconds).
3. All subsequent clients for the same key and same gutter server hit the gutter cache.
4. When the primary recovers, the short TTL on gutter entries expires and traffic automatically shifts back.

The gutter pool changes the failure mode from "failed server → database overload → cascading failure" to "failed server → one database query per gutter server per ~2 seconds per affected key." This is a bounded blast radius.

### Pool decomposition by access pattern

Not all keys belong in the same pool. Facebook identifies three types by empirical access analysis:

- **Wildcard pool**: the default. Items with typical access patterns.
- **Small items, high frequency ("hot pools")**: extremely popular items (e.g., a celebrity's profile) that must not be evicted by less popular items. Isolated in dedicated servers.
- **Large items, low frequency ("cold pools")**: items that are accessed rarely but are large enough to waste space in a high-frequency pool. The eviction policy would eject them aggressively; in a separate pool with a different eviction tuning, they survive longer.

The key insight: pool membership is based on *access pattern*, not key namespace. The same logical "object type" might have some instances in the hot pool and others in the cold pool.

### Regional architecture

Facebook has multiple data centers (regions). One is the master (handles all writes). Others are replicas. Within each region:

- Reads are served by the local cluster (fast).
- Writes go to the master region's MySQL, then replicate via binlog to slave regions.
- McSqueal in each slave region subscribes to the local replica's binlog and derives local cache invalidations.

This gives geographically local reads (low latency) at the cost of eventual consistency for writes. The remote marker mechanism (described above) is the patch that prevents obvious read-your-own-write violations.

## Where it breaks

**Hot key concentration** — Lease tokens rate-limit to one token per 10 seconds per key. For a truly hot key (queried 10,000 times per second), every requester during the 10-second window must wait. The single database query per 10 seconds is the bottleneck; for extremely hot data (celebrity profile, viral post), even this is too slow.

**Invalidation fan-out** — A delete for a popular key must be broadcast to all clusters in all regions. With many clusters, the invalidation message fan-out grows linearly. For writes that touch many keys simultaneously, the broadcast overhead is significant.

**Eventual consistency is explicit** — The system is not linearizable or sequentially consistent. Remote markers are a best-effort patch, not a formal guarantee. If the application writes to master and the remote marker expires before replication completes (e.g., due to a replication lag spike), the user can still read stale data.

**McSqueal parsing fragility** — McSqueal parses the binlog to derive keys. This ties the cache layer to the database schema. Schema changes require coordinating McSqueal's parsing logic. The paper acknowledges this as an operational burden.

**Cold start** — When a new cluster comes online or a server restarts, it has an empty cache. If traffic switches immediately, the cold cache generates enormous database load. The paper describes "cluster-warming": routing a new cluster's traffic through an existing warm cluster as a miss proxy, so the new cluster fills from cache rather than database.

## Why it works

The paper's deepest insight is not any individual mechanism — it's the **protocol design principle** that recurs across all of them:

> Prefer delete (invalidate) over update (replace). Prefer optimistic checks over locks. Prefer derived computation over application-layer coordination.

Each choice follows from a single observation: **cache correctness is a distributed consensus problem, and you want to do as little coordination as possible**.

**Lease tokens = optimistic concurrency control for cache fill.** Database OCC (the same thing ZooKeeper uses for watches, and HTTP uses for ETags) works by: read with a version, do work, compare-and-swap to write — abort if version changed. Lease tokens are exactly this: read with a token, fill from DB, write with the token — abort if the token was revoked (key was invalidated). The memcache layer is just OCC where the "version" is a lease and the "transaction" is a DB read + cache write.

**McSqueal = CDC for derived views.** The pattern "treat the write-ahead log as an event stream, derive secondary state from it" is universal: Kafka Connect/Debezium does it for ETL pipelines, Differential Dataflow does it for incremental view maintenance, Flink does it for streaming aggregations. McSqueal is CDC where the derived view is the cache. The binlog is the authoritative event log; correctness derives from following it, not from application code correctly issuing deletes.

**Gutter pools = overflow reservoir with TTL-bounded recovery.** The circuit-breaker pattern (fail fast rather than cascading) is well known. Gutter pools add an overflow buffer between the circuit breaker and silence: instead of "circuit open → all traffic falls to the database", it's "circuit open → traffic falls to the gutter → database sees at most one query per (TTL × gutter_size) per key." The short TTL is the recovery mechanism: no operator action needed to drain the gutter when the primary recovers.

**Remote markers = client-carried causal token.** Zanzibar's zookies (a timestamp that proves the client has seen at least a certain point in causal history), MVCC's read-your-own-write semantics, HTTP ETags for conditional PUTs — all of these are the same idea: the client carries a lightweight proof of what causally happened before its request, so the server can decide whether to serve local state or go upstream. Remote markers are the smallest possible version of this: a boolean flag with a TTL.

## Going deeper

1. **TAO: Facebook's Distributed Data Store for the Social Graph** (USENIX ATC 2013) — Facebook's next layer, purpose-built for graph access patterns; shows what happens when you need cache invalidation and association-list reads to be atomic, which memcache cannot provide.

2. **Memcached internals: the slab allocator and LRU** — The implementation paper for memcached itself, by Brad Fitzpatrick. Understanding why the slab allocator exists (fixed-size chunks to avoid fragmentation) clarifies why "pool decomposition by item size" is the natural extension of what memcached was already doing internally.

3. **"Scaling Databases at Facebook" (2014 VLDB)** — The companion paper describing the MySQL sharding strategy (Shard Manager) that the memcache layer sits in front of, including how schema changes and replication are managed across thousands of MySQL shards.
