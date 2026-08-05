---
title: Dapper, a Large-Scale Distributed Systems Tracing Infrastructure
source: https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/
author: Benjamin H. Sigelman, Luiz André Barroso, Mike Burrows, Pat Stephenson, Manoj Plakal, Donald Beaver, Saul Jaspan, Chandan Shanbhag
company: Google
date_posted: 2010-04
date_digested: 2026-08-05
---

# Dapper, a Large-Scale Distributed Systems Tracing Infrastructure

## What's new to learn

Three non-obvious ideas that will change how you think about distributed systems observability:

**1. The span model is a distributed log join problem.** Every span carries `(trace_id, span_id, parent_span_id, start_timestamp, end_timestamp, annotations)`. There is no central coordinator collecting this during execution. Instead, each process writes its span to a local file independently, and the trace is reconstructed *offline* by joining all spans on `trace_id`, then arranging them into a tree using the parent pointers. The insight is that you don't need coordination during execution — you need only enough shared state (the IDs) to reconstruct causality after the fact.

**2. Sampling must be decided at the root and propagated — not per-span.** If each component independently samples at probability *p*, you get partial traces: the root records a span, a mid-tier service doesn't, its downstream child does. The result is orphaned spans with no parent — useless for understanding end-to-end behavior. Dapper's fix is simple but consequential: the *first* component to receive an external request flips the coin and records the decision in the trace context. Every downstream component reads this flag and either *always* records (sampled) or *never* records (unsampled). Complete traces or nothing. No orphans.

**3. Trace collection must be out-of-band, not in-band.** It would be natural to piggyback trace data on RPC response headers. Dapper explicitly rejects this. Reasons: it inflates response size on hot paths, it doesn't work for fire-and-forget requests or async callbacks where there's no response to piggyback on, and it couples tracing latency to request latency. Instead, spans are written to local disk and a separate background daemon ships them to Bigtable asynchronously. The request path is decoupled entirely.

## Prerequisites

- **RPC mechanics**: what it means for service A to call service B synchronously; the request/response cycle
- **Thread-local storage**: a per-thread key-value map that survives through a call stack without being passed explicitly; how context propagates across function boundaries in a single process
- **Basic probability**: what a sampling rate of 1/1024 means; independent vs. correlated sampling
- **Happens-before**: Lamport's definition — event A *happens-before* event B if A causally precedes B. Useful but not required.

No distributed systems depth needed. The Dapper paper is deliberately self-contained.

## The core idea

When a user clicks "Search" on Google, the resulting HTTP request fans out across dozens of services: frontend, index servers, spell checker, ads, personalization, and more. When that request is slow, which service is responsible? When it fails, where did the failure originate?

Without tracing, you have isolated logs from each service — timestamps don't align to microsecond precision across machines, and there's no shared identifier tying a log line from the frontend to a log line from the ads service that it called.

Dapper's answer: give every request a globally unique `trace_id` at the very first point of entry. When that service calls another service, include `trace_id` in the RPC metadata. When the downstream service calls *its* dependencies, propagate `trace_id` again. Every service also generates a local `span_id` for its own unit of work and records the `span_id` of whoever called it as `parent_span_id`. 

The result: every span written anywhere in the system carries enough information to reconstruct the full causal tree. Offline, you collect all spans, group by `trace_id`, and arrange by parent pointer. You now have a complete picture of which services were called, in what order, for how long, with what annotations.

The key abstraction: **a trace is a tree of spans**. Each span is a named, timed operation. Edges in the tree represent "called by." The root span has no parent.

## Mechanics

### Instrumentation layers

Dapper instruments three layers, not application code directly:

1. **A tracing library** (< 1000 lines of C++, < 800 lines of Java) that manages trace context and span lifecycle
2. **RPC framework hooks** that automatically propagate trace context in request headers and start/end spans around every RPC call
3. **Thread-local storage** for passing trace context within a single process across function boundaries without polluting function signatures

Application developers interact with Dapper only to add *annotations* — key-value pairs attached to spans for domain-specific context (e.g., `"user_id" → 12345`, `"cache_hit" → false`). The structural plumbing is invisible.

### Span lifecycle

```
Request arrives at service A
  → RPC framework reads trace_id + parent_span_id from headers
  → Dapper library creates span: {trace_id, span_id=fresh_id(), parent_span_id, start=now()}
  → span stored in thread-local storage
  → application code runs, optionally adding annotations
  → service A calls service B
      → RPC framework reads current span from thread-local
      → sends {trace_id, span_id=current_span_id} in B's request headers
      → B creates its own child span
  → service A receives B's response
  → span.end = now()
  → span written to local log file
```

Cost numbers from Google's production measurements:
- **204 ns** to create a root span (first span in a trace)
- **176 ns** to create a non-root span (trace context already exists)
- **9 ns** to perform the unsampled check (the common case when not recording)
- **~40 ns** per annotation

### Sampling: two tiers

**Runtime sampling (tier 1)**: The root component samples at rate `1/1024` by default for high-volume services. The sampling decision is a single bit propagated in every downstream RPC header. Downstream components check this bit; if not set, they skip span creation entirely and pay only the 9 ns lookup cost.

**Collection-time sampling (tier 2)**: Even at 1/1024, Google-scale produces enormous data. After spans land in Bigtable, a secondary filter samples again using a hash of `trace_id` to keep a configurable fraction of complete traces. This is safe because the sampling unit is an entire trace, not individual spans — the tier-1 guarantee of "complete trace or nothing" is preserved.

**Adaptive sampling**: Dapper also supports parameterizing by *desired rate* rather than *probability*. A service that handles 10 RPS at 1/1024 records ~0.01 traces/sec — too sparse. A service handling 10,000 RPS records ~9.8 traces/sec — likely more than needed. Adaptive sampling adjusts probability dynamically per endpoint to hit a target rate.

### Collection pipeline

```
Application process
  → writes spans to local log file (mmap'd, low-latency)
    ↓ (asynchronous, ~O(15 seconds) typical delay)
Dapper collector daemon (one per machine)
  → reads local log files
  → ships to Dapper Bigtable cluster
    ↓
Bigtable
  → row key: trace_id
  → columns: one per span
  → retention: several weeks
    ↓
Dapper query API / Depot
  → used by Dapper UI and programmatic clients
```

Scale at Google (2010): **1 TB of trace data per day**, across thousands of services. Daemon overhead: **< 0.3% of one CPU core** per machine. Network overhead: **< 0.01% of total network traffic**. Storage: **~426 bytes per span** after compression.

### Thread-local and callback propagation

For synchronous, thread-per-request models, thread-local storage naturally carries trace context through a call stack. The RPC framework reads it when making outbound calls.

For callbacks and async models, trace context must be explicitly captured at the point the callback is registered and restored when it fires. Dapper instructs teams to capture the current trace context in the closure — a discipline similar to how structured concurrency tools like Go's context package work.

## Where it breaks

**Coalescing effects**: Suppose a slow query to a shared cache is the root cause of slow user-facing requests. Dapper will show high latency in every trace that touches the cache, but won't directly show *why* the cache is slow — that requires looking at the cache's own traces and correlating. Dapper shows where time was spent, not why a dependency is slow.

**Batch and async workloads**: The span model assumes request-scoped trees. MapReduce jobs process millions of records in waves of map and reduce tasks — there's no single "root request." Dapper's authors note this limitation explicitly: batch workloads need a different granularity (e.g., trace the job, not individual record operations).

**Fire-and-forget work**: If service A sends a message to a queue and the queue consumer runs minutes later, the parent-child link is severed unless the message explicitly carries `trace_id`. Message queue integrations require explicit instrumentation to thread the context through.

**Kernel events**: Dapper lives entirely in userspace. Events like kernel scheduling delays, network stack behavior, or disk I/O are invisible unless the OS provides hooks. Profiling flame graphs and Dapper traces show complementary but non-overlapping views.

**Explains where, not why**: Dapper tells you which service consumed the most latency. It cannot tell you *why* — whether the slowness is due to CPU contention, lock contention, bad query plans, or downstream misbehavior. That diagnosis still requires metrics, logs, and profiler data. Tracing narrows the search space; it doesn't eliminate it.

**Sampling bias**: At 1/1024, rare events (a bug triggered by one-in-a-million inputs) will almost never be sampled. Tail latency at the 99.9th percentile may not appear in traces at all if absolute request volume is modest.

## Why it works

The deeper principle: **distributed tracing is the practice of making the happens-before partial order explicit as a persistent, labeled graph.**

Lamport (1978) showed that in a distributed system, you cannot assign a total order to events — only a partial order based on causality. A happens before B if A sent a message that B received, or if A and B are in the same process and A precedes B in execution order. Dapper doesn't fight this; it *encodes* it. The `parent_span_id` field is exactly the Lamport "message sent" edge made persistent and retrievable.

This pattern recurs everywhere in systems:

- **Git commit DAG**: each commit records parent hash(es); the DAG encodes the happens-before order of changes. You reconstruct history by following edges.
- **MVCC version chains**: each tuple version records the previous version pointer; you reconstruct the history of a database row by following the chain.
- **Chandy-Lamport snapshots**: process states are captured along with in-flight messages; you reconstruct a consistent global state by joining them.
- **Vector clocks**: each event records the full vector of logical clocks, encoding all known causal dependencies.

In all cases: record causal IDs locally, collect asynchronously, reconstruct the partial order offline by joining on IDs. Dapper applies this pattern to request latency.

The sampling insight reinforces this: you need the *entire* happens-before chain for a trace to be useful. A partial chain (orphan spans) is worse than no chain, because it misleads. Root-based sampling is the mechanism that preserves the invariant: either the full causal chain is recorded, or none of it is.

The instrumentation-layer insight also follows: the happens-before relationship is encoded in the *communication infrastructure* (RPC calls), not in application logic. Instrumenting the RPC framework captures causality automatically for all services, without requiring application developers to reason about it.

## Going deeper

**The paper itself**: [Dapper, a Large-Scale Distributed Systems Tracing Infrastructure](https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/) (Sigelman et al., 2010). Short and readable — 14 pages including figures and references.

**OpenTelemetry**: The current industry standard for distributed tracing, metrics, and logs. Directly descended from Dapper's model. The [OTel trace specification](https://opentelemetry.io/docs/concepts/signals/traces/) maps exactly to Dapper's concepts: Span → Span, TraceId → trace_id, ParentSpanId → parent_span_id. Understanding Dapper makes the OTel spec immediately legible.

**Zipkin** (Twitter, 2012) and **Jaeger** (Uber, 2017): Open-source implementations of Dapper's architecture. Zipkin is simpler; Jaeger is more production-hardened. Both are now CNCF projects.

**"Dapper's Regrets"** (Ben Sigelman, 2020): The original designer's retrospective on what was wrong with Dapper's design and what OpenTelemetry fixes. Covers why per-process sampling (rather than per-request) would have been better, and why the annotation model was too unstructured.

**W3C TraceContext**: The HTTP header standard (`traceparent`, `tracestate`) that codifies how `trace_id` and `span_id` propagate across services from different vendors. Solving the multi-vendor propagation problem Dapper didn't have to solve internally.

**Monarch** (Google, 2020): Google's time-series monitoring system, which complements Dapper. Metrics tell you *that* something is wrong; traces tell you *where*. Together they give you why. Worth reading after Dapper to understand how Google's observability stack fits together.
