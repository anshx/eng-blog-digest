---
title: "How Discord Automates ScyllaDB Clusters at Scale"
source: https://discord.com/blog/how-discord-automates-scylladb-clusters-at-scale
author: Discord Persistence Infrastructure Team
company: Discord
date_posted: 2026-05-08
date_digested: 2026-05-11
---

# How Discord Automates ScyllaDB Clusters at Scale

## What's new to learn

- **Conditions as first-class workflow citizens** — safety pre-conditions expressed as blocking workflow elements rather than ad-hoc procedural checks, making safety structurally unskippable.
- **Zone-aware concurrency primitives** — composable parameters (`concurrency_unit`, `concurrency_limit`) that let you express complex multi-zone scheduling constraints declaratively, without custom orchestration code.
- **Resumable coordinator state via local SQLite** — using a single-writer local database to track distributed job progress, decoupling durability from distribution.

## Prerequisites

- **Quorum in distributed databases**: the requirement that a majority (or weighted subset) of replicas acknowledge a read/write before it's considered successful. Losing quorum means the cluster can't serve consistent responses.
- **Node drain**: the operation of gracefully migrating a database node's token ranges (data responsibilities) to peers before taking it offline for maintenance.
- **ScyllaDB**: a C++ rewrite of Apache Cassandra, using a shard-per-core architecture. Data is distributed across a ring via consistent hashing; each node owns a set of token ranges.
- **Availability zones (AZs)**: isolated fault domains within a cloud region. ScyllaDB clusters are typically spread across 3 AZs so that an AZ failure doesn't lose quorum.
- **Prometheus**: time-series metrics system. ScyllaDB exports operational metrics (repair progress, compaction backlog, node state) as Prometheus-scrapable endpoints.

## The core idea

Discord manages dozens of ScyllaDB clusters totaling hundreds of nodes. Operations like rolling OS upgrades, cluster expansion, and shadow cluster standup used to require careful manual sequencing — an engineer had to know the right sequence, check quorum at each step, wait for nodes to stabilize, and avoid running steps in parallel across zones.

They built **SCP (Scylla Control Plane)**: a workflow orchestration framework where safety invariants aren't documented in a runbook — they're typed, mandatory elements in the workflow definition itself. A workflow is a YAML-described graph of tasks and conditions. A condition is a task variant that blocks forward progress until some observable cluster state satisfies a criterion. Before any node is drained, SCP checks quorum. It cannot be skipped because the check isn't a reminder; it's the gate.

The result is that complex, multi-hour cluster operations become trusted, resumable background jobs. The team ships features rather than supervising maintenance windows.

## Mechanics

### The four-layer model

SCP is structured as four composable concepts:

1. **Task** — the atomic unit of work. Examples: `drain-node`, `check-repair-status`, `run-cleanup`. Tasks come in two flavors:
   - *Node tasks*: operate on a single node (e.g., drain this node)
   - *Cluster tasks*: coordinate across the entire cluster (e.g., trigger a full repair)

2. **Condition** — a special task type that doesn't do work, but *blocks* until a criterion is satisfied. It polls ScyllaDB's internal API or Prometheus metrics on a configurable interval, then either passes, retries, or times out with an error surfaced to the operator.

3. **Workflow** — a declarative YAML definition that sequences tasks and conditions, with concurrency controls applied:
   - `concurrency_unit`: the grouping key for parallelism — `zonal` groups nodes by AZ, `global` treats all nodes as one pool
   - `concurrency_limit`: maximum nodes executing a given task simultaneously, across all groups

   Together, these primitives can express constraints like:
   > "Restart nodes one zone at a time; within a zone, at most two nodes may restart concurrently."
   Without writing any custom orchestration code.

4. **Job** — a running or completed instance of a workflow, bound to a specific cluster. SCP persists job state to a local SQLite database, recording which tasks have completed, which are in-flight, and which have failed on each node.

### The quorum-safety check

Before any drain operation executes, SCP automatically verifies two things:

- **Quorum safety**: given the current set of live nodes, does draining this node still leave enough replicas alive for consistent reads and writes?
- **Cluster health**: is the cluster in a stable, non-degraded state (no ongoing repairs that would leave data underreplicated if a node leaves)?

These checks live inside the task definition, not in a calling script. There is no API surface to drain a node without them running.

### The `is-up-normal` condition

After a node rejoins a cluster (e.g., after an OS upgrade), SCP waits for it to be fully operational before proceeding to the next node. The condition polls ScyllaDB's status API. The key detail: it doesn't pass on the first successful poll. It requires **60 consecutive seconds of healthy responses** — a stabilization window. A node that's intermittently healthy (e.g., during GC pressure after rejoining) won't prematurely unblock the workflow. The condition will wait up to 24 hours before surfacing an error, so long-running operations like streaming data off a replacement node are accounted for.

### Webhook notifications and human-in-the-loop moments

For join operations — where a new node begins streaming data, temporarily stressing the cluster — SCP fires a webhook to the on-call engineer before each join begins. The automation proceeds without waiting for a response (it's a notification, not an approval gate), but the team gets visibility at the moments with the highest blast radius.

### Resumability via SQLite

SCP's coordinator tracks all job state in a local SQLite database:
- Per-node task completion status
- In-progress tasks at time of interruption
- Failure state and error messages

If SCP restarts mid-job — due to a deploy, a crash, or intentional pause — it reads the SQLite state and resumes from the exact step where execution stopped. Completed tasks are never re-run. Tasks that were mid-execution when interrupted are retried from the start (they're designed to be idempotent at the task level).

### What gets automated

The framework currently drives:
- **Rolling OS upgrades**: drain → upgrade → rejoin, one node per zone at a time, across hundreds of nodes
- **Cluster expansion**: adding nodes and waiting for streaming and repair to complete before adding the next
- **Shadow cluster standup**: spinning up a full replica cluster for testing, in a defined sequence
- **Smarter expansion strategies** (in development): adaptive scheduling based on cluster load metrics

## Where it breaks

**Condition timeouts require tuning**: A 24-hour wait for `is-up-normal` is generous, but wrong for clusters where a 30-minute stall means "something's broken." Teams need to profile their expected operation times and configure appropriate timeouts per workflow — there's no universal default.

**SQLite is a single point of failure for the coordinator**: If the machine or container running SCP's coordinator dies permanently (not just restarts), the SQLite state is lost. The team accepts this tradeoff: re-running a job from scratch is rarely catastrophic, and the alternative — distributed coordinator state — introduces far more complexity.

**Prometheus polling granularity**: Conditions that poll Prometheus inherit Prometheus's scrape interval (typically 15–60 seconds). A condition checking a metric that changes in 5-second windows may either miss a brief failure or delay progress by an extra scrape interval. This is a fundamental coupling between the automation's temporal resolution and the metrics system's.

**YAML workflow definitions don't type-check at author time**: A misconfigured `concurrency_unit` or a typo in a task name won't be caught until the job runs and fails. There's no static validation layer described.

**Webhook notifications aren't approval gates**: For the highest-risk operations, a notification-only model means the job proceeds even if the on-call engineer is unreachable or sees the alert and wants to abort. True human-in-the-loop for joins would require making the workflow pausable on webhook receipt.

## Why it works

The deep principle here is **lifting implicit knowledge into explicit structure**.

In a manual runbook, "check quorum before draining" is a step that lives in a document and in an engineer's head. It can be skipped under pressure, misunderstood by a newcomer, or simply forgotten. When you express quorum safety as a typed condition in the workflow graph, it becomes structurally impossible to skip — not because of policy enforcement, but because the workflow's execution model requires conditions to pass before downstream tasks begin. This is the same insight behind type systems, contracts in Eiffel, and assertions in formal verification: moving correctness requirements from documentation into the execution artifact.

The `concurrency_unit` + `concurrency_limit` design connects to **blast radius control**, which appears throughout distributed systems in forms like:

- **Canary deployments**: roll out to 1% of traffic before 10%
- **Cell-based architectures**: shard a service into independent cells so a failure can't cascade globally
- **Circuit breakers**: stop sending traffic to a struggling dependency

All of these are ways of saying: limit how much of the system can be in a transitional/risky state simultaneously. SCP's zone-aware concurrency is the same idea applied to database cluster management: never let two AZs be in maintenance at once, because that's the quorum failure threshold.

The SQLite coordinator state is an instance of a broader pattern: **local state for coordination is often more reliable than distributed state for coordination**. This is why ZooKeeper and etcd are hard to operate at scale — any distributed consensus system has failure modes of its own. A SQLite file on the coordinator's local disk has a simple, well-understood failure model: the disk works, or it doesn't. For a workload where "redo from scratch" is acceptable, local SQLite eliminates an entire class of distributed consistency bugs.

The 60-second stabilization window in `is-up-normal` is an instance of **hysteresis** — a technique from control theory where a system requires a state to be sustained for a duration before acting on it. Thermostats use hysteresis to prevent toggling on and off every few seconds when temperature is near the threshold. SCP uses it to prevent "healed, then immediately stressed" nodes from unblocking the workflow prematurely.

## Going deeper

1. **ScyllaDB's operator documentation on cluster operations**: The ScyllaDB docs describe what node drain, repair, and streaming actually do at the protocol level — essential for understanding what SCP's conditions are waiting for. [https://operator.docs.scylladb.com/](https://operator.docs.scylladb.com/)

2. **Temporal.io's blog on durable workflow execution**: Temporal solves a closely related problem — workflows that can survive process crashes — at a higher abstraction level and with a distributed backend. Comparing SCP's SQLite approach with Temporal's approach illuminates the tradeoffs between simplicity and generality. [https://temporal.io/blog](https://temporal.io/blog)

3. **"So You've Lost Quorum: Lessons From Accidental Downtime" (ScyllaDB)**: A companion piece from ScyllaDB describing the failure modes that quorum-safety checks like SCP's are designed to prevent — useful for understanding the stakes. [https://www.scylladb.com/tech-talk/so-youve-lost-quorum-lessons-from-accidental-downtime/](https://www.scylladb.com/tech-talk/so-youve-lost-quorum-lessons-from-accidental-downtime/)
