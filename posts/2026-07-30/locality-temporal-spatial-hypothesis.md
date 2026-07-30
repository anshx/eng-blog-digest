---
title: "Locality, and Temporal-Spatial Hypothesis"
source: https://brooker.co.za/blog/2025/10/05/locality.html
author: Marc Brooker
company: AWS
date_posted: 2025-10-05
date_digested: 2026-07-30
---

# Locality, and Temporal-Spatial Hypothesis

## What's new to learn

1. **The Temporal-Spatial Hypothesis (TSH)**: an unstated assumption woven into most sequential storage designs — "data written at approximately the same time will be read at approximately the same time, and should therefore be stored near each other."
2. **Key design as cache policy**: choosing a primary key scheme is a bet on whether TSH holds for your workload; the bet determines your buffer pool hit rate, B-tree page-split rate, and hot-spot exposure.
3. **DynamoDB's deliberate TSH rejection**: DynamoDB's hash-based partitioning is not a limitation — it is a deliberate design for workloads where TSH is false, trading read locality for write uniformity.

## Prerequisites

- How B-trees work: sorted keys stored in fixed-size pages; a page split occurs when a page overflows
- What a database buffer pool is: a fixed-size in-memory cache of disk pages, typically managed with LRU or CLOCK eviction
- What a hot spot is: a single shard, page, or node receiving disproportionate write traffic relative to the rest of the cluster
- Basic understanding of how DynamoDB partition keys work (hash → shard routing)

## The core idea

Every database stores rows somewhere on disk. The question is: *where* relative to each other? If you are using a B-tree index (InnoDB, PostgreSQL, SQLite), the "where" is determined by the sort order of the primary key. If you are inserting rows with an `AUTO_INCREMENT` integer or a time-prefixed UUID (v7), each new row lands at the right edge of the index — the page that currently holds the largest key values. The buffer pool keeps that edge page hot. Your next insert hits warm cache. The pages holding rows from six months ago slowly fall out of cache — but those rows are read rarely.

This only works if the workload respects the Temporal-Spatial Hypothesis: the rows you insert today are the same rows you read today. When a news feed shows you "latest articles," when a payment dashboard shows "recent transactions," or when a monitoring tool shows "last 5 minutes of metrics," TSH holds. The most-recently-written data is the most-frequently-read.

Now switch the primary key to a random UUID (v4). Every insert lands in a *random* position in the B-tree. To write row #10,000,001, the database has to bring in the one page out of ~10,000 pages that holds the correct range — a page not seen since the last time something hashed near that key. Your buffer pool has become nearly useless because the working set is the entire index, not the recent edge. Cache hit rate collapses. Insert throughput drops 30–50%. At scale you're effectively writing to a random-read I/O device, even if physically the disk is sequential.

DynamoDB takes the same "spread everything randomly" approach but *on purpose*: its hash-based partitioning distributes writes evenly across all storage nodes. Why is this correct for DynamoDB? Because DynamoDB workloads typically do not satisfy TSH: clients hit arbitrary items across the entire keyspace, throughput per partition is provisioned uniformly, and the worst failure mode is a hot partition (a single shard absorbing all traffic). For DynamoDB, rejecting TSH is correct. For a time-ordered B-tree table of user events, rejecting it is catastrophic.

## Mechanics

**B-tree right-edge inserts (TSH satisfied)**

With a monotonically increasing key, every insert modifies exactly one leaf page — the rightmost one — plus a bounded number of internal pages (amortized O(1) when the tree is balanced). That leaf page stays in the buffer pool between inserts. A table with 100 million rows might need to keep only a handful of warm pages in memory to sustain thousands of inserts per second.

When the leaf page fills, a page split occurs: the database allocates a new page, moves half the entries there, and updates the parent. With sequential keys, splits happen at the right edge predictably; they do not fragment existing pages.

**B-tree random inserts (TSH violated — UUIDv4)**

With UUIDv4, inserts are distributed uniformly across the 2^122 key space. A 1 GB index with 8 KB pages has ~131,000 pages; each insert is a random page fault. The buffer pool (typically 25–40% of RAM) can cache maybe 50,000 pages, so ~60% of inserts require a disk read before the write can proceed. Every page that receives an insert has empty slots because sequential range never filled it fully; the result is *index bloat* — a 1.3–1.8× inflation factor on index size compared to a sequential key. The extra size further degrades cache coverage.

Benchmark numbers across PostgreSQL studies:
- **Inserts**: UUIDv4 is 30–50% slower than BIGINT SERIAL at scale (millions of rows, 8 GB+ table)
- **UUIDv7 vs BIGINT SERIAL**: ~identical insert throughput; UUIDv7 acts like a time-ordered integer from the B-tree's perspective
- **Index bloat**: UUIDv4 tables see 40–80% larger indexes vs. sequential-key equivalents at the same row count

**Streaming systems (TSH trivially true)**

In Kafka, a producer appends messages in time order; a consumer reads them in the same time order. Both producer and consumer move together along the log. Every byte of the log file is accessed exactly once, sequentially, by each consumer. This is TSH in its purest form: the access pattern is a deterministic function of write order. Sequential I/O on SSDs and HDDs is orders of magnitude faster than random I/O, so streaming systems built on TSH can sustain millions of messages per second on commodity hardware with no caching tricks.

**DynamoDB's explicit TSH rejection**

DynamoDB's partition key is run through a hash function before mapping to a storage node. The design goal is to ensure that a burst of writes — even if they share a temporal pattern — does not all land on one node. This is the right choice when:
- Reads are point lookups (no range scans that benefit from locality)
- Traffic can spike unpredictably on any item (celebrity accounts, viral content)
- Partition-level throughput limits are the binding constraint

DynamoDB's anti-pattern is the "hot key": choosing a partition key like `DATE` or a low-cardinality status field so that many rows share the same key. All traffic for that date/status floods one partition. This is what happens when engineers accidentally impose TSH — clustering recent data together — on a system designed to reject it.

**UUIDv7: TSH at millisecond granularity**

UUIDv7 encodes a 48-bit Unix timestamp in milliseconds in the high-order bits, followed by a random suffix. Within one millisecond, UUIDs are random (spreads concurrent inserts across that millisecond's range); across milliseconds, they are ordered. This provides near-sequential B-tree insertion with a small amount of randomness that distributes concurrent inserts from multiple threads.

Brooker's proposed improvement goes further: replace the raw `unix_ms` with `H(unix_ms, unix_ms >> 13 | salt)`, where the salt is node-specific. This adds enough entropy to avoid hot spots across distributed systems while preserving coarse-grained temporal ordering. The tradeoff is that the key is no longer directly decodable to a timestamp, but the B-tree locality benefits are retained.

## Where it breaks

**Multi-writer concurrency**: Sequential keys create B-tree latch contention at the right edge. Under high-concurrency workloads (thousands of inserts/second on a single table), all writers compete for the latch on the rightmost leaf page. Postgres handles this with "fastpath locking" and split optimizations, but it remains a real bottleneck at extreme write rates. Random keys distribute this latch pressure across all leaves.

**Query patterns that span time**: If your application frequently queries historical data — "show me all orders from 2023" — those pages are cold. TSH helps only when reading the most recently written data; it doesn't help (and slightly hurts, via larger sequential scan distances) for arbitrary historical range queries.

**Multi-dimensional locality**: B-tree order provides locality in exactly one dimension (the primary key). If you also need locality by `customer_id` or `category`, you need a separate index, and that index can't simultaneously respect TSH unless customer behavior is also temporally correlated (often a reasonable assumption for sessions, less so for aggregate analytics).

**Distributed sharding**: In a sharded database, a sequential key causes all new inserts to hit the shard that owns the current "right edge" — effectively the same hot-spot problem DynamoDB was designed to avoid. Distributed SQL systems (Spanner, CockroachDB) often recommend hash-prefixed keys or range-split policies for this reason. The correct answer depends on whether your cluster's binding constraint is write throughput uniformity or read locality.

## Why it works

The Temporal-Spatial Hypothesis is just **locality of reference** applied to the temporal dimension of a database.

CPU caches exploit two well-known principles:
- **Temporal locality**: a memory address accessed recently will likely be accessed again soon → LRU eviction
- **Spatial locality**: memory near a recently accessed address will likely be accessed soon → cache lines (64 bytes)

Database buffer pools exploit the exact same principles, but the "address" is a page on disk and the "neighborhood" is defined by the sort order of the primary key. When you choose a sequential primary key, you are implicitly asserting that *pages sorted by key insertion order exhibit temporal locality* — the most recently modified pages are the most likely to be accessed again. This is true precisely when TSH holds.

The UUID v4 anti-pattern is an accidental violation of temporal locality: you're evicting hot pages from the buffer pool to make room for the random page needed by the next insert, when that next random page will itself be evicted before it's read again. The buffer pool becomes a random-access translator instead of a cache.

DynamoDB's correctness comes from recognizing that its access pattern — uniform random reads across the full keyspace — means *no page is more likely to be accessed than any other*. Under that assumption, locality is worthless, and the only cache that matters is a per-item key-value cache (DAX). Optimizing for write uniformity instead of read locality is the right engineering choice for that access model.

Streaming systems' efficiency comes from TSH being a tautology: the consumer always reads what the producer just wrote, so temporal ordering of writes *is* the access pattern. Sequential I/O is simply the hardware expression of TSH.

The underlying principle: **every storage system is implicitly implementing a cache replacement policy. The key scheme you choose determines the hit rate of that cache.** Sequential keys → LRU works perfectly. Random keys → LRU is useless, effective cache size approaches zero.

## Going deeper

1. **Brooker's follow-up: "Fixing UUIDv7 (for database use-cases)"** — https://brooker.co.za/blog/2025/10/22/uuidv7.html — proposes `H(unix_ms, unix_ms >> 13 | salt)` to balance B-tree locality with write distribution in multi-node setups.

2. **Hilbert curve space-filling as multi-dimensional TSH** — Hilbert curves map 2D or higher-dimensional coordinates to a 1D order that preserves spatial locality across all dimensions. Used in Parquet row-group ordering, geospatial indexing, and (as seen in the COST paper covered earlier in this archive) graph edge ordering to convert random DRAM access into sequential scans. The Hilbert curve is TSH extended to non-temporal dimensions.

3. **"The Tail at Scale" (covered in this archive, 2026-05-24) + buffer pool sizing** — tail latency from cold-page I/O is exponentially amplified when a request fans out to N shards: even one cold page read per shard is N concurrent I/O events. Sequential keys that keep the buffer pool hot are a tail-latency tool as much as a throughput tool.
