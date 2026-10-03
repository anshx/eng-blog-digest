---
title: "Monarch: Google's Planet-Scale In-Memory Time Series Database"
source: https://www.vldb.org/pvldb/vol13/p3181-adams.pdf
author: Colin Adams, Luis Alonso, Ben Atkin, John Banning, et al.
company: Google
date_posted: 2020-08-31
date_digested: 2026-10-03
---

# Monarch: Google's Planet-Scale In-Memory Time Series Database

## What's new to learn

1. **Target-based routing** — route all metrics for an entity (a VM, job, or container) to the same storage node rather than routing by metric name; this single decision determines which monitoring queries are cheap and which are globally expensive.

2. **Algebraic aggregate requirement** — a distributed query tree can merge partial results efficiently only when the aggregate function is *decomposable* (i.e., a semigroup/monoid: merge(partial_A, partial_B) = aggregate(A ∪ B)); Monarch's query language is deliberately restricted to enforce this, which forces histograms over raw percentile values.

3. **Zone-first reliability** — partition a planet-scale system into self-contained geographic zones that can serve queries in isolation during global outages, with a global layer as an optional overlay rather than a mandatory dependency.

## Prerequisites

- What a time series is (metric name + labels + sequence of (timestamp, value) pairs)
- Basics of distributed sharding: consistent hashing, range-based partitioning
- What an aggregate function is and why sum/count/min/max differ from median/percentile
- Rough familiarity with Bigtable or any wide-column store (for the comparison to disk-based alternatives)

## The core idea

Every monitoring system at planet scale faces a routing dilemma: when a time series arrives, to *which node* do you send it?

The two natural choices are **metric-based routing** (all CPU metrics go to the CPU shard, all RPC metrics go to the RPC shard) and **target-based routing** (all metrics for `frontend-vm-23` go to the same shard, regardless of what metric they are).

Monarch chose target-based routing. That choice drives almost every subsequent decision in the system.

With target-based routing, querying "what is wrong with *this specific server right now?*" is cheap: all its data lives on one leaf. Querying "what is the 95th percentile CPU usage *across the entire fleet*?" is expensive: it must scatter to every leaf. This is exactly the right trade-off for monitoring — the urgent, on-call query is the per-entity diagnostic, not the fleet-wide aggregation (which can tolerate a few extra seconds).

Once you commit to target-based routing, the query path naturally becomes a recursive scatter-gather tree: to answer any fleet-wide aggregate, a root mixer fans the query out to zone mixers, each of which fans out to leaves in that zone, and results bubble back up via merge operations. For this tree-merge to be *correct*, every aggregate function in the query must be algebraically decomposable — you can compute a sum-of-sums, but you cannot compute a median-of-medians.

The full system is the mechanical consequence of those two decisions: target routing + algebraic aggregates.

## Mechanics

### Data model

A Monarch time series is identified by a `(target, metric, labels)` triple:

- **Target**: the monitored entity — a structured key like `{job="/frontend", zone="us-east1", task="42"}`. Exactly one field in the target schema is annotated as `location`; its value determines the Monarch zone where the time series is stored.
- **Metric**: the measurement kind, e.g. `/rpc/server/latency`. The metric schema specifies the value type (int64, double, distribution) and allowed label keys.
- **Labels**: additional key-value dimensions on the measurement (e.g., `method`, `status_code`).

Time series within a zone are sharded lexicographically by target string. Every leaf in a zone holds a contiguous range of target-space.

### Storage: leaves

Leaves are the only components that hold raw time series data. All data is kept **in memory** — typically a few hours to a few days of retention depending on the metric. The rationale is query latency: disk I/O makes sub-second dashboard refresh painful at any reasonable scale. Longer retention is served by exporting sampled or aggregated data to Bigtable (not covered in the paper's focus).

Each leaf owns a target-space range assigned by a **range assigner**, which rebalances the assignment as new metrics arrive or leaves join/leave. Leaves serve both ingestion writes and query reads.

### Ingestion path

```
Producer (monitored service)
  → Ingestion router       (routes to the correct zone based on location field)
     → Leaf router          (routes to the correct leaf based on target string)
        → Leaf               (stores data in memory)
```

The key invariant: **ingestion is stateless at every layer above the leaf**. Routers hold no data; they only look up routing tables maintained by the range assigner. A zone boundary failure stops new writes to that zone but leaves the read path intact.

### Query path: the mixer tree

Monarch runs queries as a three-level tree:

```
User / Borgmon (query issuer)
  → Root mixer              (global layer; fans out to all zones)
     → Zone mixer           (one per zone; fans out to leaves in zone)
        → Leaves             (compute partial aggregates, return results)
```

A leaf evaluates the query locally (filter rows, compute partial aggregate) and returns a compact partial result. The zone mixer merges partial results from all leaves in the zone into a zone-level partial result. The root mixer merges zone-level results into the final answer.

This is valid only when the aggregate function F satisfies:

```
F(A ∪ B) = merge(F(A), F(B))
```

for arbitrary disjoint data sets A and B. Examples:
- **Algebraic**: COUNT, SUM, MIN, MAX, MEAN (represented as (count, sum) pair)
- **Distributable with extra state**: percentile — impossible to merge P99(A) and P99(B); instead, store a distribution (histogram) on each leaf, merge histograms, compute P99 at the root

This forces all percentile-style SLOs to be expressed as "P99 of distribution D" — the leaf stores a histogram, which is algebraically mergeable (bin-by-bin sum).

### Zone self-sufficiency

Each zone has its own ingestion path, range assigner, leaf set, and zone mixer. A `zone query` to a zone mixer returns results for data in that zone only, without contacting any global component. During global outages (network partition between zones, root mixer failure), each zone continues to serve zone-scoped queries.

Global queries require the root mixer and are degraded when zones are unavailable. This is a deliberate availability trade-off: local on-call queries (almost always zone-scoped for a specific task) stay up; fleet-wide aggregation queries degrade gracefully.

### Configuration plane

Monarch does not expose a "write here" API to producers. Instead, producers configure **collection configs** that describe *what to scrape, from which targets, at what interval*. A separate collection subsystem executes those configs and writes results to Monarch. This decoupling means the monitoring pipeline itself can be monitored and adjusted without changing producer code.

## Where it breaks

**No P99 across arbitrary dimensions in real time**: If you need P99 latency broken out by (method, region, customer_tier) with arbitrary slice-and-dice, you need to pre-define the histogram buckets. Ad-hoc slice changes require redefining collection.

**Target-based routing hurts cross-entity metric queries**: "Which 1% of servers have the highest CPU?" must scatter globally, hit every leaf, sort all results, and return the top-k. This is unavoidably O(all leaves) work.

**In-memory only ⇒ limited retention**: Keeping petabytes of raw time series in RAM is possible at Google's scale, but it places a hard ceiling on retention. Anything older than the in-memory window requires a separate (slower) query to Bigtable.

**No joins**: There is no mechanism to join two different metric streams on the fly. Relating "latency for endpoint X" to "CPU usage for the backend serving X" is an application-level concern.

**Zone granularity is coarse**: If the location field is `zone`, all metrics for an entire zone land in one Monarch zone. A multi-region service with metrics scoped at the zone level behaves well; a single-region service that uses finer location granularity needs careful schema design.

## Why it works

### MapReduce on an aggregation tree

The mixer hierarchy is **recursive MapReduce**:
- Map: each leaf applies filter predicates and computes a partial aggregate (reducer in the Map sense)
- Reduce: each mixer merges partial aggregates from its children

The algebraic requirement is exactly the commutativity + associativity requirement for a MapReduce reducer. This is the same insight behind:
- **ClickHouse parallel replicas** (this archive): scatter partial aggregates across replicas, merge at coordinator
- **HyperLogLog** (this archive): each node's register-max is mergeable because max is a monoid
- **Ring AllReduce** (this archive): gradient sum is decomposable; you pipeline sums around a ring
- **MapReduce** (this archive): the reducer must be a commutative monoid

Monarch adds the geographic dimension: the tree of mixers mirrors the geographic hierarchy, so "query fanout" and "network locality" are co-designed.

### Zone isolation is the physical manifestation of availability zones

Cloud providers call them AZs; Monarch calls them zones. The principle is identical: treat geographic failure domains as the unit of fault isolation, and build the system so each domain is independently operable. Zone mixers serve local queries; root mixers are optional overlays. Monarch was doing this in 2010, years before "AZ-aware architecture" became cloud-industry vocabulary.

### The routing dimension is a database design decision in disguise

Sharding by target is **row-oriented distribution** (a row = all metrics for one entity). Sharding by metric would be **column-oriented distribution** (a column = one metric across all entities). The archive's RUM conjecture (Algorithms Behind Modern Storage Systems) frames this as a read/write amplification trade-off. Monarch's choice optimizes for single-entity reads at the cost of cross-entity metric reads — the right trade-off for interactive on-call debugging versus batch analytics.

### Algebraic restriction as a schema contract

By restricting queries to algebraically decomposable aggregates, Monarch converts a correctness requirement (distributed merge must be lossless) into a query language constraint (no unsupported aggregates). This is the same principle as linearizability-through-CAS in lock-free data structures: *encode the correctness invariant in the interface so violations are impossible to express*, rather than detectable only at runtime.

## Going deeper

1. **The original VLDB 2020 paper** — Colin Adams et al., "Monarch: Google's Planet-Scale In-Memory Time Series Database," PVLDB Vol 13 No 12 (2020): https://www.vldb.org/pvldb/vol13/p3181-adams.pdf — the canonical reference with detailed evaluation numbers (>PB in memory, millions of queries/sec).

2. **Gorilla: A Fast, Scalable, In-Memory Time Series Database** (this archive, 2026-05-31) — covers just the compression layer (delta-of-delta timestamps, XOR floats) that a system like Monarch would use inside each leaf; together they form the full stack.

3. **Streaming 101 / The Dataflow Model** (this archive, 2026-06-20) — the deeper theory of event-time vs processing-time and watermarks, which contextualizes *why* monitoring systems need a time-series model distinct from a batch DB.
