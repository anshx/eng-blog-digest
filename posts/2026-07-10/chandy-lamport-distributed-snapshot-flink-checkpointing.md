---
title: "Lightweight Asynchronous Snapshots for Distributed Dataflows"
source: https://arxiv.org/abs/1506.08603
author: Paris Carbone, Gyula Fóra, Stephan Ewen, Seif Haridi, Kostas Tzoumas
company: Apache Flink / KTH Royal Institute of Technology
date_posted: 2015-06-26
date_digested: 2026-07-10
---

# Lightweight Asynchronous Snapshots for Distributed Dataflows

## What's new to learn

1. **Consistent Cut**: a partition of a distributed execution's history into "past" and "future" that respects causality — if event B is in the past and A happened-before B, then A is also in the past. This is the formal definition of "what a consistent snapshot of a distributed system looks like."

2. **Marker / Barrier Protocol**: by injecting a special token that travels through the same communication channels as ordinary data, you can create a consistent cut — and therefore a valid snapshot — without ever pausing the system. The token is the cut.

3. **ABS vs. Chandy-Lamport**: when the dataflow topology is a DAG (no cycles) with FIFO channels, in-flight messages between the snapshot cut and the barrier arrival are provably empty at every edge, so you only need to record operator state — not channel state — reducing checkpoint size to the minimum possible.

## Prerequisites

- Lamport's *happens-before* relation: event A happens-before event B (A→B) if A is the send of a message that B receives, or there is a chain of such send-receive pairs. Without a global clock, happens-before is the only notion of causality available.
- FIFO channels: messages sent on the same logical channel (src, dst pair) arrive in the order they were sent. Most streaming frameworks and TCP connections satisfy this.
- Basic distributed systems: processes communicate only by sending messages; there is no shared memory, no global clock.

## The core idea

To take a photograph of a room full of people in motion, you can either tell everyone to freeze simultaneously (requires a global signal) or have each person take their own photo the moment a light wave reaches them from a strobe you just fired (the light propagates at finite speed, but the set of photos still captures a consistent moment because the light respects causality).

The Chandy-Lamport distributed snapshot algorithm (1985) uses the second approach. A special *marker* message plays the role of the strobe. A process records its own state the moment it first sees the marker, then forwards the marker on every outgoing channel. Because the marker travels through the same FIFO paths as data, it arrives at each downstream process *after* all messages that were in transit before the snapshot started — giving you a causally consistent global state without any global pause.

Apache Flink's **Asynchronous Barrier Snapshot (ABS)** paper (2015) refines this for acyclic streaming dataflow graphs. The markers are renamed *barriers*, and the key observation is: in a DAG with FIFO channels, there are *zero* messages in transit between the cut point and the barrier arrival at any edge, so the channel state is always empty. The checkpoint therefore stores only operator state (the interesting part), not channel queues. This makes checkpoints proportional to application state rather than load.

## Mechanics

### Chandy-Lamport (the foundation)

**Algorithm — two rules, one for initiation and one for receiving a marker:**

**Initiating process p_i:**
1. Record p_i's own local state S_i.
2. For every outgoing channel c from p_i, send a `<MARKER, checkpoint_id>` message. This must happen *before* sending any further application messages on c.

**Any process p_j receiving a marker on channel c_ij:**

*Case A — first marker p_j has ever seen (for this checkpoint):*
1. Record p_j's own state S_j immediately.
2. Record the state of channel c_ij as **empty** (the marker arrived before any application messages from the post-snapshot period).
3. For every outgoing channel from p_j, send a `<MARKER>`.
4. For every *other* incoming channel c_kj, begin buffering arriving application messages — they constitute that channel's in-transit state.

*Case B — p_j already recorded its state, and now receives a marker on another channel c_kj:*
1. The state of c_kj is the sequence of messages that arrived on c_kj since p_j recorded its state, up to (but not including) this marker.
2. Stop buffering c_kj.

**Termination:** when every process has recorded its state and every channel's state has been recorded.

**What you get:** a global state (S_1, S_2, …, S_n, msg(c_12), msg(c_21), …) that is *consistent*: no recorded message appears both as "received" in one process's state and "in transit" in a channel. You can replay execution from this state and reach any future state the original run would have reached.

### Flink ABS (the production refinement)

Flink's dataflow graphs are DAGs: data flows from sources through operators to sinks, with no cycles. Flink exploits two properties:

1. **FIFO channels**: every Kafka partition, every inter-operator network connection — messages arrive in send order.
2. **Acyclic topology**: no feedback loops, so barriers cannot chase their own tails.

**Consequence**: when a barrier for checkpoint N arrives at operator O on channel c, *all* messages from before the checkpoint have already been delivered on c (they were sent before the source injected the barrier). The channel state is provably empty. No buffering needed.

**Flink ABS algorithm:**

```
Source operator (barrier injection):
  - Checkpoint coordinator sends TRIGGER to all sources.
  - Source records its state (e.g., Kafka offset).
  - Source emits <BARRIER, checkpoint_N> downstream on every output partition.

Intermediate operator (barrier alignment):
  - Has k input channels. Receives barriers at different times (fast upstream vs. slow).
  - When barrier N arrives on channel i: block channel i (buffer further input from i).
  - Continue processing from un-blocked channels.
  - When ALL k barriers have arrived: record operator state asynchronously, unblock all channels, emit <BARRIER, checkpoint_N> on every output partition.

Sink operator:
  - Same alignment as intermediate. On all-barriers received: acknowledge to coordinator.

Coordinator:
  - When all sinks acknowledge checkpoint N: declare checkpoint N complete. Persist the global state to durable storage (HDFS, S3, RocksDB remote).
```

**Recovery:**
On failure, roll back all operators to their last completed checkpoint state, and reset all input sources to the recorded offsets. Because the snapshot is causally consistent, replaying from this point produces exactly the results that would have been produced had the failure not occurred — giving exactly-once semantics.

**Unaligned checkpoints (Flink 1.11+):** barrier alignment blocks fast channels waiting for slow ones, creating head-of-line blocking under backpressure. The fix: snapshot in-flight messages on unblocked channels *as part of the checkpoint state* (restoring the full Chandy-Lamport channel recording). This trades larger checkpoints for lower checkpoint latency under load.

## Where it breaks

**FIFO assumption**: if channels reorder messages (e.g., UDP, unordered parallel transport), the channel state is no longer trivially empty even in a DAG, and you must implement the full Chandy-Lamport channel recording. Many real systems (out-of-order networks, multiple parallel TCP connections) violate FIFO.

**Barrier alignment = HOL blocking**: the blocking step waits for the slowest channel, which under backpressure can delay checkpoints for seconds. The unaligned variant fixes this but creates checkpoints that are harder to manage (state includes in-flight messages, which can be large).

**Cyclic dataflows**: iterative algorithms (machine learning, graph processing) create cycles. Barriers in cycles need special handling — Flink requires that iterative regions define explicit "iteration barriers" at loop-back edges, which are managed separately. The standard ABS does not generalize to cycles.

**State size growth**: checkpoint latency scales with operator state size. A stateful join operator holding a 100 GB hash table will block network I/O during state serialization unless the operator implements truly asynchronous copy-on-write snapshotting (which Flink partially supports via RocksDB incremental checkpoints).

**Coordinator is a single point of coordination**: the checkpoint coordinator must track barrier acknowledgments from all operators. At very high parallelism (thousands of tasks), this coordinator can become a bottleneck.

## Why it works

The deepest insight: a *consistent cut* is exactly a **downward-closed set in the happens-before partial order**. If event e is in the "past" of the cut and event e' happened-before e, then e' must also be in the past — you cannot include a receive without including the corresponding send.

The marker protocol *implements* this downward-closure automatically: the marker travels through the same channels as application messages. If process p receives a message m from q *before* the marker from q, then p includes m in the past. If p receives the marker from q *before* m, then m is in the future. Since channels are FIFO, "before the marker" is exactly "happened-before the marker-send event." The marker is the cut, propagated as data.

This is the same structural insight that appears across distributed systems under different names:

| Domain | "Marker" | What it demarcates |
|---|---|---|
| Chandy-Lamport | MARKER message | Global state snapshot |
| Apache Flink ABS | Checkpoint BARRIER | Consistent checkpoint epoch |
| Kafka consumer groups | Committed offset record | Exactly-once processing boundary |
| MVCC databases | Transaction begin/commit timestamps | Consistent snapshot read |
| CPU memory model | MFENCE / acquire-release | Visibility boundary between stores and loads |
| Two-phase commit | PREPARE message | Point of no return for commit |
| Git | Commit SHA | Immutable point in history DAG |

In every case, a *token that flows through the same causal paths as the data it observes* creates a boundary that is consistent with respect to causality. You cannot observe a "send without the receive" or a "receive without the send." The token makes the invisible ordering relationship explicit and tangible.

The reason ABS is cheaper than the full Chandy-Lamport protocol is the same reason a *read barrier* in a CPU is cheaper than a full memory fence: when you know the topology (DAG = no cycles, FIFO = ordered delivery), you can prove the channel state is always empty at the cut, and skip recording it. **Topology knowledge converts theoretical overhead into zero overhead.**

## Going deeper

1. **Original Chandy-Lamport paper (1985)**: "Distributed Snapshots: Determining Global States of Distributed Systems" — the 7-page original, still the clearest statement of the consistent-cut formalism. Lamport's personal page: https://lamport.azurewebsites.net/pubs/chandy.pdf

2. **Flink's unaligned checkpoints deep dive**: "From Aligned to Unaligned Checkpoints — Part 1: Checkpoints, Alignment, and Backpressure" on the Apache Flink blog (2020) — explains exactly why barrier alignment causes backpressure and how in-flight message recording solves it, at the cost of larger state.

3. **"Lightweight Asynchronous Snapshots for Distributed Dataflows" (2015)**: the full paper — https://arxiv.org/abs/1506.08603 — includes formal correctness proofs, performance measurements showing sub-100ms checkpoint latency at high throughput, and the extension to cyclic topologies.
