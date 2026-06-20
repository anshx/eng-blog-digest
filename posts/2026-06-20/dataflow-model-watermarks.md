---
title: "Streaming 101: The World Beyond Batch"
source: https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/
author: Tyler Akidau
company: Google
date_posted: 2015-08-05
date_digested: 2026-06-20
---

# Streaming 101: The World Beyond Batch

## What's new to learn

Three concepts most batch-trained engineers don't know until they hit production streaming:

1. **Event time vs. processing time** — Every event carries two clocks: when it actually occurred (event time) and when the processing system ingested it (processing time). The gap — called skew — is seconds to hours, and conflating them is the root cause of most streaming correctness bugs.

2. **Watermarks** — A monotonically non-decreasing timestamp W that the system asserts: "no future event will have event time < W." Watermarks are the mechanism that converts the undecidable question "have I seen all the events for this window?" into a tractable engineering trade-off between latency and correctness.

3. **Triggers and accumulation modes** — The orthogonal decomposition of *when* to emit a window result (trigger strategy) from *what* each successive emission contains relative to prior ones (discard, accumulate, or retract). This lets a single pipeline provide low-latency early estimates, a correct on-time answer, and late-data corrections without restructuring the computation.

## Prerequisites

- Understand what a MapReduce or Spark batch job does: read all input, compute per partition, reduce to output.
- Know that a system like Kafka delivers messages from producers to consumers with some delay.
- No prior stream processing experience required.

## The core idea

A batch processor operates on a finite, bounded dataset and can wait for all input before producing output. A streaming system processes an infinite, continuously arriving dataset. It cannot wait forever.

The naïve approach — timestamp each event by the wall clock at arrival and window over arrival time — breaks whenever events are delayed or out-of-order. An ad click that happened at 14:00 but arrived via a batching mobile SDK at 14:45 lands in the wrong window, corrupting every downstream metric that depends on it.

The Dataflow Model's solution: define windows in **event time** (when the event happened), not processing time (when it arrived). Then use a **watermark** — a system-maintained lower bound on future event times — to decide when a window is complete enough to emit without waiting forever.

When the watermark advances past the end of a window, that window fires. The watermark is the key that makes finite computation possible on infinite data.

## Mechanics

### Event Time vs. Processing Time

```
User clicks an ad at:       14:00:03  ← event time
Mobile SDK batches, flushes: 14:45:12 ← processing time (arrival)

Skew = 45 minutes 09 seconds
```

Skew is always ≥ 0 (a message cannot arrive before it was sent) but is variable and workload-dependent: real-time IoT sensors may have sub-second skew; offline-capable apps can accumulate hours. Windowing by processing time assigns this click to the 14:45 window; windowing by event time correctly places it in the 14:00 window.

### Windows

Windows group the infinite stream into finite, aggregatable subsets. All window types are defined over event time:

**Fixed (tumbling) windows** — Non-overlapping intervals: [14:00, 14:05), [14:05, 14:10), etc. Each event belongs to exactly one window. Used for per-hour revenue totals, per-5-minute error counts.

**Sliding windows** — Overlapping intervals: 10-minute windows stepping every 2 minutes. A single event appears in multiple windows. Used for moving averages and rate-of-change metrics.

**Session windows** — Variable-length, gap-delimited. A session starts on the first event and extends as long as events keep arriving within some inactivity gap (e.g., 30 minutes). Boundaries are data-driven and unknown until the session closes — which makes them the hardest to implement efficiently, since new events can merge two previously separate sessions retroactively.

### Watermarks

A watermark W at processing time p is a lower bound on the event time of all future events: any event arriving after processing time p will have event_time ≥ W(p). The watermark is produced by **sources** based on what they know:

- A Kafka connector can emit watermark = min(latest-committed-offset timestamp across all partitions). If every partition has delivered events up to time T, no future delivery can have event_time < T.
- A bounded file reader emits watermark = +∞ after its last record.

In a multi-stage pipeline, the **effective watermark** at any operator is the minimum of all upstream watermarks. A single stalled partition in one source holds back the watermark for the entire pipeline.

**When the watermark passes a window's end boundary**, the window fires: the system emits the accumulated result and (by default) discards the window's state.

Because watermarks are **heuristic** in practice — you cannot know with certainty that no delayed event will ever arrive — they are configured with a **allowed lateness** budget. Events that arrive after the watermark has passed their window are called **late data**.

### Triggers

The watermark fires the default ("on-time") emission. Triggers let you add more firing conditions:

- **Early triggers** — fire a speculative result periodically before the watermark arrives. Users see low-latency approximations while the window is still open.
- **Late triggers** — fire again when late data arrives after the watermark has passed. The window is reopened, the new event is folded in, and an updated result is emitted.

A composed trigger:

```
AfterWatermark()
  .withEarlyFirings(
      AfterProcessingTime.pastFirstElementInPane(Duration.standardSeconds(30))
  )
  .withLateFirings(
      AfterPane.elementCountAtLeast(1)
  )
```

This single configuration yields: a result every 30 seconds during the window (early, approximate), a result when the watermark fires (on-time, best estimate), and a result whenever a straggler arrives after the window nominally closed (late, corrective).

### Accumulation Modes

When a window fires more than once, each emission is a **pane**. The accumulation mode defines what each pane contains relative to previous panes:

**Discard** — each pane contains only events that arrived since the last firing. Downstream sums the deltas.

```
Early pane 1:  clicks = 12   (events in first 30s)
Early pane 2:  clicks = 7    (events in next 30s)
On-time pane:  clicks = 3    (remaining)
Downstream:    12 + 7 + 3 = 22  ✓
```

**Accumulate** — each pane contains the full running total since the window opened. Downstream just displays the latest value, overwriting the previous.

```
Early pane 1:  clicks = 12
Early pane 2:  clicks = 19
On-time pane:  clicks = 22    ← downstream shows this
```

**Accumulate + Retract** — each pane retracts the previous value and asserts the new one. Required for downstream operators (joins, outer aggregations) that cannot tolerate receiving the same event contribution twice.

```
Early pane 1:  +12
Early pane 2:  -12, +19      ← undo 12, assert 19
On-time pane:  -19, +22      ← undo 19, assert 22
```

Retraction is the most powerful mode but the hardest to operationalize: the pipeline must remember what it last emitted and propagate "undo" messages to every downstream consumer.

### The What/Where/When/How Framework

The Dataflow Model makes four orthogonal design choices explicit:

| Axis | Question | Example answer |
|------|----------|---------------|
| **What** | What are you computing? | Sum of clicks per ad campaign |
| **Where** | In which event-time interval? | Fixed 5-minute windows |
| **When** | At what processing-time point should results fire? | Early every 30s, on-time at watermark, late on arrival |
| **How** | How do successive panes relate? | Accumulate (overwrite) |

Any streaming system makes choices on all four axes, either explicitly or by baking a default into its API. Making them explicit forces the pipeline author to articulate the latency-vs-correctness tradeoff they're accepting.

## Where it breaks

**Watermark accuracy is a fundamental tension.** An aggressive watermark (assume small skew) provides low latency but silently drops late events. A conservative watermark (assume large skew) provides correctness but high latency. The right setting is workload-specific and must be re-tuned when upstream systems change their batching behavior.

**Session windows are expensive at scale.** Merging sessions requires retroactively retracting both predecessor sessions and emitting a merged replacement. Under high cardinality (millions of distinct user sessions), the state management and retraction fan-out become bottlenecks. Systems like Flink bound this with a maximum session gap.

**Retractions require end-to-end support.** Most downstream systems — SQL databases, dashboards, feature stores — can ingest new values but lack an "undo" primitive. The accumulate+retract mode is semantically correct but operationally fragile unless every sink in the lineage supports retraction.

**Cross-stream joins compound watermark divergence.** Joining two streams produces an output watermark = min(left_watermark, right_watermark). If one stream's source falls behind (e.g., a Kafka partition with a slow producer), it stalls the join for all matching records in the other stream, regardless of how far ahead the other side's watermark is.

**Exactly-once semantics are orthogonal to the model.** The Dataflow Model defines correct *output semantics* but says nothing about fault tolerance. Achieving exactly-once delivery requires separate machinery: checkpointed operator state + idempotent output sinks (Flink's approach) or two-phase-commit to transactional sinks (Kafka Transactions + Flink).

## Why it works

The watermark is a **logical clock for event-time progress**.

In distributed systems, a Lamport timestamp solves an analogous problem: you cannot know the absolute wall-clock time at remote nodes, so you track a logical ordering instead. A Lamport timestamp says "this event causally follows all events with a lower counter." A watermark says "no event with event time < W will ever arrive."

Both are monotonically non-decreasing. Both are *commitments about ordering* rather than measurements of absolute time. Both substitute a tractable approximation for an unknowable ground truth.

The deeper unification: **batch processing is streaming with a single watermark jump**.

A MapReduce job over a bounded file is a streaming pipeline where:

1. The watermark starts at −∞ (the first record has not yet been read)
2. Records are consumed, advancing the watermark record by record
3. After the last record, the source emits watermark = +∞
4. All open windows see the watermark pass their end boundary, fire exactly once, and close

The same pipeline logic — windows, triggers, accumulation — handles both modes. This is the proof that a streaming system with correct watermark semantics needs no separate "batch accuracy layer." A system like Apache Beam makes this concrete: you author one pipeline, choose a bounded or unbounded source, and the runtime handles the rest. The lambda architecture (separate batch and speed layers) was a workaround for systems that lacked correct streaming semantics; the Dataflow Model is the argument that the workaround was never necessary.

The same pattern appears in snapshot isolation (MVCC): a transaction "sees" a consistent snapshot by tracking the high-watermark of committed transactions at the moment it opened. Both mechanisms convert a continuously advancing stream of state changes into a well-defined point-in-time view. The watermark is MVCC's read timestamp applied to event time instead of transaction time.

## Going deeper

1. **"Streaming 102: The World Beyond Batch"** by Tyler Akidau — [https://www.oreilly.com/radar/the-world-beyond-batch-streaming-102/](https://www.oreilly.com/radar/the-world-beyond-batch-streaming-102/) — the follow-up post that works through triggers and accumulation modes with detailed diagrams and the full trigger composition API; essential reading after this one.

2. **"MillWheel: Fault-Tolerant Stream Processing at Internet Scale"** (Google VLDB 2013) — the predecessor system inside Google that motivated the Dataflow Model; shows how low watermarks are computed from Kafka-like injectors, how they propagate through a DAG of computations, and how state is checkpointed for fault tolerance and exactly-once delivery.

3. **Apache Flink "Timely Stream Processing" documentation** — [https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/concepts/time/](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/concepts/time/) — the canonical open-source implementation of these ideas; shows how `WatermarkStrategy` is assigned at sources, how watermarks flow through the operator DAG, how late records are routed to side outputs, and how event-time timers fire window functions.
