---
title: "How does a database scale any query to 9,000+ cores?"
source: https://clickhouse.com/blog/clickhouse-group-by-parallel-replicas-8900-cores
author: Tom Schreiber
company: ClickHouse
date_posted: 2025-09-12
date_digested: 2026-06-03
---

# How does a database scale any query to 9,000+ cores?

## What's new to learn

- **Mergeable partial aggregation state** — An intermediate representation for GROUP BY results that supports a `merge(a, b)` operation, enabling distribution of aggregation work to any number of workers with results recombined later without loss of correctness.
- **Granule-based demand-driven scheduling** — Breaking a table into ~8,192-row units and having workers *pull* tasks on demand (rather than statically pre-assigning data upfront), achieving near-perfect load balancing even on skewed data.
- **Consistent-hash-aware work stealing** — Using consistent hashing to prefer assigning the same granule ranges to the same replicas across queries (for OS page-cache reuse), while still allowing faster nodes to steal remaining work from slower ones.

## Prerequisites

- SQL GROUP BY semantics: what it means to compute COUNT, SUM, AVG, MAX per key group
- ClickHouse MergeTree basics: data is stored in sorted column files; a sparse primary index has one entry per *granule* (8,192 consecutive rows by default), making a granule the smallest independently-scannable chunk
- How an in-memory hash table powers GROUP BY inside a single thread
- The concept of horizontal scaling: more nodes = more parallel work

## The core idea

Any GROUP BY aggregation can be split into two algebraically separable phases:

1. **Partial phase** — Each worker scans its assigned rows and builds a local hash table: `key → partial_state`. For `count()`, the partial state is a running count; for `avg(price)`, it's a `(sum, count)` pair.
2. **Merge phase** — A coordinator receives all partial hash tables and merges them key-by-key: `merge(partial_a, partial_b)` → combined partial state → finalize to the answer.

The critical insight is that `merge` is **associative and commutative** for every standard SQL aggregate. This means you can assign *any* subset of rows to *any* worker — rows with key=42 split across three workers produce three partial states that merge to the same answer as if one worker had seen them all. No pre-sorting or key-routing (shuffle) is needed.

ClickHouse's parallel replicas feature applies this directly: data lives in a single logical storage shard (no manual resharding), a coordinator partitions the table into ~8,192-row granules and dispatches them to all replica nodes on demand. Each replica executes up to `WithMergeableState` — filtering, scanning, and partially aggregating its assigned granules — then streams the partial hash table back. The coordinator merges them all and returns the final result. 100 billion rows aggregated in 414 ms across 9,000 cores, with linear wall-time scaling because the scan is the bottleneck and it parallelizes perfectly.

## Mechanics

### MergeTree granules

ClickHouse stores data in *parts* — sets of column files for an ordered range of rows. The sparse primary index has one entry per `index_granularity` rows (default: 8,192). Each index entry is a *mark*: a byte offset into the compressed column file. Scanning begins by looking up the mark range containing the target keys, then reading whole granules at a time. A granule is the atomic unit — you can't read fewer rows from a column file.

Granules map directly to units of parallel work: the coordinator doesn't need to understand the query semantics to split up the work. It just partitions the list of `(part_name, mark_start, mark_end)` tuples and assigns them to replicas.

### Query processing stages

ClickHouse execution uses `QueryProcessingStage` to say how far each node should go before stopping and returning results to the caller:

- **`Complete`** — execute the full query, return final rows. Used for single-node queries.
- **`WithMergeableState`** — execute filtering, column reads, and partial aggregation, but do *not* finalize aggregate functions and do *not* apply final LIMIT/HAVING. Return the partial hash table as a block stream.

For parallel replicas, the coordinator sends the query to each replica with stage `WithMergeableState`. Each replica:
1. Reads its assigned granules from shared object storage
2. Decompresses columns needed for the query
3. Evaluates WHERE clause (pushes down into the column scan)
4. Builds a local hash table keyed by the GROUP BY expression
5. Does NOT finalize (e.g., does NOT divide sum by count for `avg`)
6. Streams the partial hash table to the coordinator over TCP

The coordinator then runs a *merge-aggregate* pipeline: it reads partial block streams from all replicas, merges matching keys, and at the end calls `finalize()` on each accumulated state.

### Granule assignment and consistent hashing

The coordinator holds a task queue of all `(part, mark_range)` tuples for the table. Replicas connect via TCP and request work batches. The coordinator assigns ranges using consistent hashing: `hash(part_name, mark_range_start)` determines the preferred replica. On repeated queries, the same granule ranges land on the same replica, so the OS page cache (or ClickHouse's own mark cache) is warm.

The `max_parallel_replicas` setting caps how many replicas participate. The coordinator also enforces `parallel_replicas_min_number_of_rows_per_replica` (default: 500,000 rows) — below this threshold it falls back to single-node execution to avoid coordination overhead dominating.

### Work stealing

When a replica finishes all its initially-assigned granule ranges, it requests more. The coordinator assigns remaining "orphan" ranges from slow replicas. Consistent hashing still applies during stealing: among all replicas that finished early, the one whose consistent-hash is nearest to the orphan range's hash gets priority. This preserves cache locality during stealing while still preventing tail latency from one straggler holding up the query.

PR #91374 (`allow_all_replicas_steal_orphaned_ranges`) allows all idle replicas to steal in parallel rather than one-at-a-time, further improving tail latency on very unbalanced work distributions.

### Partial state structure per aggregate

| Function | Partial state | Merge | Finalize |
|---|---|---|---|
| `count()` | `uint64` | `a + b` | identity |
| `sum(x)` | `T` (same numeric type) | `a + b` | identity |
| `avg(x)` | `(sum: T, count: uint64)` | `(a.sum+b.sum, a.count+b.count)` | `sum/count` |
| `max(x)` / `min(x)` | `T` | `max(a, b)` / `min(a, b)` | identity |
| `uniq(x)` | HyperLogLog sketch | bitwise union | estimate from HLL |
| `quantile(p, x)` | t-digest or reservoir | merge digests | read p-th quantile |

## Where it breaks

**Coordinator merge bottleneck.** The scan work is distributed, but the merge is serial on one node. At ~80–100 replicas each sending a full partial hash table, the coordinator's merge cost and inbound network bandwidth dominate. ClickHouse Cloud addresses this with *multi-stage distributed execution* (announced separately): workers exchange intermediate results and re-partition by GROUP BY keys so subsets of keys merge on different nodes, then a final merge coordinator sees only a fraction of the state.

**High-cardinality GROUP BY.** If GROUP BY keys are nearly unique (e.g., `GROUP BY session_id`), each partial hash table is almost as large as the raw data. Network transfer from 100 replicas is 100× the raw data size; the coordinator merge is O(n_keys × n_replicas). For such queries, preaggregating at write time (AggregatingMergeTree) or using sampling may be better.

**Non-mergeable exact aggregates.** `median()`, exact `percentile()`, and `mode()` require seeing all values — there is no mergeable partial state that preserves them exactly. ClickHouse offers `quantileTDigest` (bounded error) and `quantileExact` (shuffles all values to coordinator, which kills the scaling benefit). If you need exact quantiles, parallel replicas don't help.

**Cold caches.** Consistent hashing provides cache locality on *warm* caches. After a cluster resize, a node restart, or a new shard added, the consistent hash re-assigns some ranges to different nodes. Those nodes' page caches are cold for the newly assigned ranges, causing I/O spikes. For object-storage–backed clouds this is especially visible since reads from S3/GCS are much slower than local NVMe.

**JOIN support was initially absent.** The feature launched without support for JOIN queries; it only applied to pure GROUP BY aggregations. JOIN support was later contributed (by Tinybird) as a separate extension.

## Why it works

The mathematical foundation is that every standard SQL aggregate function is a **monoid homomorphism**: a function from data to intermediate state where the states form a monoid (associative merge + identity element). This means:

```
finalize(merge(partial(A), partial(B))) = finalize(partial(A ∪ B))
```

for any partition of the dataset into A and B. Because merge is associative, you can chain merges across any number of workers in any order.

This is algebraically identical to the **MapReduce** pattern (Dean & Ghemawat, OSDI 2004) — but with one critical engineering difference: **no shuffle**. MapReduce's reduce step requires that all values for a given key arrive at the same reducer, so it does a sort-by-key shuffle between map and reduce. ClickHouse skips the shuffle because its partial states are keyed hash tables that support *key-intersection merge* rather than key-exclusive reduce: `merge(table_a, table_b)` iterates over the union of keys and combines states for matching keys. No routing of rows to specific workers is necessary.

The same "partial state + merge" structure is the unifying insight behind:

- **HyperLogLog** — `union(HLL_A, HLL_B)` estimates `|A ∪ B|` (already in archive via SFQ post's sketch analogy)
- **Bloom filters** — `OR(B_A, B_B)` = membership filter for `A ∪ B`
- **Count-Min Sketch** — element-wise `max()` merges frequency tables
- **Federated machine learning (FedAvg)** — averaged model gradients from different nodes = partial state over different data shards
- **Spark's `reduceByKey` vs `groupByKey`** — the former carries partial states through the shuffle; the latter sends raw values. The ClickHouse approach is `reduceByKey` without the shuffle step.
- **Git objects** — content-addressed blobs are an immutable monoid under object union; `git merge` works because the object store is append-only and associative

The granule-based work pulling is the **work-stealing scheduler** pattern (Go goroutine scheduler, Java ForkJoinPool) applied at query execution level. The "tasks" are physical storage granules — a natural unit because the index already knows where each granule starts, so no runtime splitting is needed. The consistent-hash preference for task assignment adds a locality bias that work-stealing alone doesn't have.

## Going deeper

1. **"Introducing multi-stage distributed query execution in ClickHouse Cloud"** (ClickHouse Blog, 2025) — how ClickHouse addresses the coordinator merge bottleneck with multi-hop aggregation trees, removing the single-coordinator ceiling.
2. **"MapReduce: Simplified Data Processing on Large Clusters"** (Dean & Ghemawat, OSDI 2004) — the original paper formalizing the partial-state / merge pattern for large-scale data processing; reading it after this post reveals that ClickHouse parallel replicas is MapReduce without shuffle.
3. **"How ClickHouse executes a query in parallel"** (ClickHouse Docs, `clickhouse.com/docs/optimize/query-parallelism`) — details the full intra-node parallelism stack (thread-per-core pipelines, vectorized execution, SIMD), complementing the inter-node parallel replicas story.
