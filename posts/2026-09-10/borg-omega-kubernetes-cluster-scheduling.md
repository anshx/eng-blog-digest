---
title: "Borg, Omega, and Kubernetes: Lessons Learned from Three Container-Management Systems over a Decade"
source: https://queue.acm.org/detail.cfm?id=2898444
author: Brendan Burns, Brian Grant, David Oppenheimer, Eric Brewer, John Wilkes
company: Google
date_posted: 2016-03-23
date_digested: 2026-09-10
---

# Borg, Omega, and Kubernetes: Lessons Learned from Three Container-Management Systems over a Decade

## What's new to learn

1. **The three-tier scheduler taxonomy** — cluster schedulers fall into exactly three architectures: monolithic (one component owns all decisions), two-level (a resource broker + per-framework schedulers), and shared-state (parallel schedulers each with a full cluster view, coordinated by OCC). Each tier trades policy expressiveness against scheduling throughput.

2. **Cluster scheduling as optimistic concurrency control** — placing a workload on a node is a database transaction: read cluster state → score candidate nodes → submit placement atomically → retry on conflict. Omega's insight is that resource conflicts are rare in practice (< 1% of placement attempts), so OCC's "pay on conflict" beats 2PL's "pay on every access."

3. **Containers as the new OS kernel for datacenters** — the cluster manager becomes the abstraction layer that maps application intent ("I need 4 cores, 8 GiB, HTTP port 80") onto physical machines, exactly as an OS kernel maps processes onto hardware. The container image is the new executable binary.

## Prerequisites

- **Optimistic concurrency control (OCC)**: read-validate-commit protocol where transactions speculatively proceed without locks and retry only on actual conflict. (Covered in depth in the ARIES and Snapshot Isolation entries.)
- **Cgroups and Linux namespaces**: the kernel primitives that enforce CPU/memory limits and provide filesystem, network, and PID isolation — the mechanism that makes a "container" a container.
- **Paxos / Raft**: how the Borgmaster's state is replicated for fault tolerance.

## The core idea

Imagine the entire cluster as a single relational table: each row is a machine, each column is a resource (CPU, RAM, disk, GPU). Placing a job is a `UPDATE` statement — claim some rows (machines) by decrementing their available-resource columns. The question is: how do many concurrent schedulers coordinate these updates without serializing every scheduling decision through one bottleneck?

**Borg** answered: use one master (BorgMaster). All scheduling goes through a single Paxos-replicated component. Safe, but the scheduler is a global bottleneck.

**Omega** answered: replicate the full cluster state to every scheduler, let them all run independently, and commit with OCC. Each scheduler sees a snapshot, makes a placement decision, then submits it as a transaction. If two schedulers claimed the same machine, one wins; the other re-reads current state and retries. The cell-state store is the transaction arbitrator.

**Kubernetes** synthesized the two: it uses a shared API server (like Omega's cell-state store) as the canonical source of truth, but uses a single default scheduler plus optional custom schedulers — simpler than Omega's full parallelism, more flexible than Borg's monolith.

The deep insight is that resource allocation at scale is a database problem, and the right concurrency model depends on the conflict rate. In a cluster where thousands of machines each have room for dozens of jobs, contention is structurally rare.

## Mechanics

### Borg

**Cell**: a set of ~10,000 homogeneous machines. The cell is the unit of scheduling isolation — jobs are placed within a cell, not across cells.

**BorgMaster**: a Paxos-replicated five-replica master that holds all cluster state and runs the scheduler loop. One scheduler goroutine per pending task: find feasible machines → score them → assign the best one → commit to replicated state.

**Borglet**: a per-machine agent that receives task placements and manages local cgroups. It polls BorgMaster periodically, providing a pull-based heartbeat.

**Priority and preemption**: every task has a numeric priority. Higher-priority tasks can preempt lower-priority ones, freeing resources immediately. In practice: `monitoring > production > batch > best-effort`. Quotas per-priority per-cluster prevent priority inversions at scale.

**What breaks**: the single scheduler loop limits throughput. Profiling found that ~80% of BorgMaster CPU was spent inside the scoring function. Adding more replica BorgMasters only helps read availability, not write throughput.

### Omega

**Cell state**: a centralized key-value store (think: an in-memory DBMS row per machine) that holds claimed resources. The store provides optimistic transactions: a scheduler submits a set of resource claims; if any claim conflicts with another committed transaction, the whole submission is rejected.

**Parallel schedulers**: each scheduler type (batch scheduler, production scheduler, MapReduce-specific scheduler) runs as an entirely separate process. Each scheduler maintains a local copy of cell state (kept fresh by polling or push). Scheduling algorithms run against the local copy — full scoring without any locking. When done, submit the placement as an OCC transaction.

**Conflict resolution**: when two schedulers try to claim the same machine concurrently, the cell state detects the conflict and rejects one. The rejected scheduler re-reads current state and reschedules the affected tasks. Google measured < 1% conflict rate in production — most of the time, capacity is ample and schedulers operate on disjoint portions of the cluster.

**Gang scheduling problem**: OCC falls apart when a job needs N machines simultaneously-or-not-at-all. Each machine claim is a separate transaction, so you cannot atomically claim all N across multiple cell-state transactions. Omega works around this by acquiring resources incrementally and releasing partial holds if the full gang can't be assembled, at the cost of wasted work.

### Kubernetes

Kubernetes converges on a middle path:

| Concern | Borg | Omega | Kubernetes |
|---|---|---|---|
| Cluster state | Paxos replicated monolith | Shared cell-state store | API server (etcd-backed) |
| Scheduling | Single loop in master | Parallel per-type schedulers | One default scheduler + plugin system |
| Conflict model | Serialized (no OCC needed) | Full OCC on cell state | Watch-based, mostly serialized |

**Key Kubernetes abstractions**: Pods (smallest schedulable unit), ReplicaSets (desired-count controllers), Services (stable DNS names over ephemeral Pods), Deployments (rolling-update controller). These are **declarative intent** objects, not imperative commands.

**Controllers and reconciliation**: every Kubernetes component — the scheduler, the replica controller, the node controller — is an independent control loop that watches the API server for desired state, compares it to actual state, and takes corrective action. This is the OCC pattern applied at the control plane level, not just the scheduling level.

**Labels and selectors**: instead of naming specific instances (`pod-1234`), Kubernetes targeting uses label sets (`app=frontend, tier=web`). This decouples the controller policy (select by label) from the physical instance allocation, making rolling updates, A/B tests, and canary deploys natural.

## Where it breaks

**Gang scheduling**: OCC over per-machine transactions can't atomically claim N machines. Frameworks like Spark or MPI that need all workers simultaneously before starting require workarounds (gang-booking via the scheduler, or accepting incremental starts). Kubernetes addresses this partly with `PodGroup` CRDs in batch extensions, but it's still not native.

**Global policy enforcement**: because Omega's schedulers are independent processes, enforcing cluster-wide fairness (e.g., "no single team uses more than 20% of cluster") requires an external quota system. Each scheduler must check quotas before submitting, but there's no atomic cross-scheduler policy evaluation. Borg's monolithic scheduler enforces global policy trivially because every decision passes through one chokepoint.

**State explosion at very large scale**: each Omega scheduler caches a full copy of cluster state. At 1 million nodes, this becomes expensive. Kubernetes mitigates this by partitioning scheduling decisions (each node is a relatively small object) but still requires the full node list for scoring.

**Cascading preemption storms**: in Borg's priority system, a high-priority job can trigger a chain of preemptions that cascade through the cluster. The scheduler must account for the downstream effects of each preemption, which requires a bounded-depth search to avoid exponential worst cases.

**Container-to-container network**: Kubernetes assumes a flat container network (every Pod has a routable IP). Implementing this correctly across physical machines with diverse network fabrics requires CNI plugins and care around NAT, overlay networks, and MTU — not something the scheduler model addresses.

## Why it works

**OCC is optimal when conflicts are rare.** This is the same reason MVCC beats 2PL in read-heavy databases: if most transactions don't conflict, paying the retry cost on the rare exception beats paying a lock-acquire cost on every transaction. Google's data shows that in a large cluster, the vast majority of scheduling decisions target machines with ample capacity — conflicts are the exception, not the rule.

**The deeper principle**: *any system where many agents independently modify a mostly-available shared resource pool benefits from OCC over pessimistic locking.* This exact pattern appears in:

- **Kubernetes controllers**: ReplicaSet controller, Deployment controller, and HPA all watch the same API server and make OCC-style updates (write with resource version, retry on conflict). They never hold locks across observations.
- **AWS capacity scheduling**: EC2 Spot instance placement is a bidding/placement system where multiple customers simultaneously try to claim the same capacity. OCC (bid accepted or rejected) beats 2PL (serialize all bids).
- **GPU memory allocators in vLLM**: PagedAttention's block table claims blocks optimistically per-request; conflicts are handled by eviction, not by locking the pool during scoring.
- **Mesos two-level scheduling**: Mesos brokers resource *offers* (the monolith says "here are 4 CPUs on node X, take them or leave them"). Frameworks accept offers optimistically; offers can expire. This is OCC without explicit retry — the offer model shifts conflict detection to the offer cycle.

**The container-as-binary insight**: by making the container image the unit of deployment rather than bare processes, the cluster manager gains reproducibility. An image is a sealed artifact — it runs identically on any machine. This is the same insight as immutable infrastructure: *treating compute state as an immutable value rather than a mutable resource* simplifies scheduling, migration, and failure recovery dramatically. If a machine dies, the scheduler can start the same container elsewhere without any coordination with the dead machine.

**Declarative over imperative**: Kubernetes' `desired state → actual state → reconcile` model is a control theory feedback loop applied to distributed systems. You specify what you want (replicas=5), not how to get there. The system continuously converges toward the desired state, tolerating failures, preemptions, and node additions transparently. This is the CALM theorem in practice: monotone coordination-free operations (adding replicas) need no synchronization; only non-monotone operations (reducing replicas safely) need coordination.

## Going deeper

1. **"Large-scale cluster management at Google with Borg"** (OSDI 2015, Verma et al.) — the full Borg paper with detailed utilization data, the cell-compaction metric, and evaluation of priority inversion rates. The source for Borg's architectural internals.

2. **"Mesos: A Platform for Fine-Grained Resource Sharing in the Data Center"** (NSDI 2011, Hindman et al.) — the two-level scheduling alternative. Mesos uses resource *offers* rather than OCC; frameworks see the offer and decide to accept, decline, or let it expire. Compare this with Omega's transaction model to see how different "mostly non-conflicting" concurrency models trade off.

3. **Kubernetes documentation: "Controllers"** (kubernetes.io/docs/concepts/architecture/controller/) — explains the watch-reconcile loop in concrete terms, with the ReplicaSet controller as the worked example. The source code for `kube-controller-manager` shows every controller as an independent OCC client against the API server.
