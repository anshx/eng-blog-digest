---
title: "Physalia: Millions of Tiny Databases"
source: https://brooker.co.za/blog/2020/02/17/physalia.html
author: Marc Brooker
company: AWS
date_posted: 2020-02-17
date_digested: 2026-06-17
---

# Physalia: Millions of Tiny Databases

## What's new to learn

1. **Blast-radius minimization over system-wide uptime**: Availability is meaningfully a per-client property; a failure that takes down one large HA cluster affects every client simultaneously, while a failure in a small cell affects only that cell's 2–7 clients. Minimizing how many clients any single failure can touch is a strictly different optimization target than maximizing mean uptime.

2. **Topology-aware quorum placement**: Co-locating a Paxos majority with its clients inside a single AZ means a network partition that isolates the AZ is the exact partition that still leaves the majority intact — the worst common failure mode becomes invisible rather than an outage.

3. **Client-key affinity as a first-class design input**: Knowing which client will access which key in advance lets you collapse a client's availability dependency down to a single tiny cell rather than a global cluster — but this only works when the access pattern is stable and foreseeable.

## Prerequisites

- **Paxos / Raft basics**: quorum writes, leader election, why you need floor(N/2)+1 votes for a decision.
- **CAP theorem**: why you must choose between consistency and availability during a network partition.
- **AWS EBS overview**: EBS volumes are virtual block devices attached to EC2 instances; a volume's "configuration record" tracks which host it's attached to and its current state.

## The core idea

Standard availability thinking asks: "How do I make the system never go down?" The answer is replicas, quorums, and multi-AZ deployments — and the resulting 99.99% uptime figure is measured system-wide.

Physalia asks a different question: "When the system does go down, how many clients are affected?"

A system that is 99.99% available but, when it fails, fails for all ten thousand clients simultaneously is worse than a system that is 99.9% available per client but whose failures are isolated to 5 clients at a time — because correlated full-system failures are disproportionately damaging. SLAs are averages; what customers experience are incidents.

This reframing leads to the Physalia architecture: instead of one large Paxos cluster, use millions of tiny ones. Each **cell** is a 7-node Paxos group serving only 2–7 specific clients. A cell's failure affects only its handful of clients. Every other cell continues serving its clients normally.

## Mechanics

### Why EBS needs this

Physalia stores configuration records for AWS EBS: which host has each volume attached, and what state the volume is in. This record is consulted on every I/O operation (routing), so it is on the hot path. It needs strong consistency — a split-brain decision about which host owns a volume would cause data corruption. It also needs very high availability — any unavailability in the config service stops I/O.

The naive solution — a single distributed key-value store — has the right consistency properties but the wrong blast-radius properties: any partial failure that drops a quorum's worth of nodes takes the config service offline for every EBS volume in the region simultaneously.

### The colony

A Physalia deployment is a **colony** of millions of **cells**.

Each cell is:
- Exactly **7 nodes** (Paxos replicas)
- Serving a **small, fixed set** of clients — typically 2–7 EBS host machines
- Owning a **small, fixed key set** — volume configurations for the volumes attached to those hosts

A given key belongs to exactly one cell. A given client reads and writes its keys through its assigned cells. These bindings are established at key-creation time and do not change unless explicitly migrated.

The **colony controller** is a separate service responsible for:
1. Creating a cell (or reusing an existing one with room) when a new volume is provisioned
2. **Topology-aware placement** of that cell's 7 nodes across the physical infrastructure
3. Monitoring cell health and initiating rebalancing when nodes fail
4. Reconstituting cells after permanent node loss

Critically, the colony controller is **not on the hot path**. Once a client knows which cell owns its keys, all subsequent reads and writes go directly from the client to the cell's nodes. The controller being unavailable does not prevent existing cells from operating.

### Topology-aware placement

This is the mechanism that converts cell decomposition from "just smaller blast radius" into "partition-survivable without cross-AZ latency."

The colony controller has knowledge of the physical topology: which nodes are on which rack, power domain, network switch, and AZ. When it creates a cell for clients in AZ-1, it places nodes as follows:

- **4 nodes in AZ-1** (the client's AZ)
- **3 nodes distributed across AZ-2 and AZ-3**

Since Paxos quorum = floor(7/2)+1 = **4**, the 4 nodes inside AZ-1 are exactly one quorum. This has two consequences:

**Low-latency normal operation.** In the common case (no failures), the client achieves quorum without leaving AZ-1. No cross-AZ round trips on the critical path.

**Partition survivability.** If a network partition isolates AZ-1 from AZ-2 and AZ-3, the 4 nodes inside AZ-1 still form a valid quorum. The cell keeps working. From the client's perspective, nothing has happened.

Compare this to the standard 3-replica cluster with one replica per AZ. During the same partition, the AZ-1 replica has only 1 vote — it cannot form a majority without at least one of the remote replicas. AZ-1 clients lose access.

Physalia makes the most common failure mode — AZ isolation — a non-event by design.

The controller also diversifies within the AZ: the 4 AZ-1 nodes are spread across different racks and power domains, so a rack-level failure that takes out 2 of them still leaves 2 local + 3 remote = 5 total nodes, which exceeds quorum.

### What a cell failure looks like

Suppose a software bug in the cell daemon crashes nodes in 1% of cells. In a traditional cluster, that 1% of nodes is spread uniformly; any 4+ failures can break the global quorum, taking down the service for all clients.

In Physalia, cells are independent. If a bug crashes nodes in some fraction of cells, only those specific cells lose quorum. Each downed cell affects its 2–7 clients. Cells that share no nodes with the buggy ones keep running. The blast radius is bounded by (fraction of cells affected) × (clients per cell), not "all clients."

The paper reports that this property was validated in production: during a large-scale network event that would have taken down a conventional distributed store, Physalia's impact was limited to the cells whose nodes spanned the partition boundary.

### The colony controller's placement algorithm

Placement is a constrained optimization problem:
- Minimize correlation between a cell's nodes (maximize topology separation — different racks, different power domains, different AZs after the primary-AZ quota is filled)
- Keep cells close to their clients (minimize latency)
- Balance load across available nodes
- Respect failure-domain budgets (don't put too many cells on the same rack, or one rack failure cascades)

The controller scores candidate placements, picks the best, and records the mapping. It does this millions of times as the fleet grows, and re-runs placement for cells that need rebalancing after node failures.

## Where it breaks

**Requires foreseeable client-key affinity.** Physalia works because EBS volumes are attached to specific hosts — the client-key binding is known at creation time. For a general-purpose cache or store where any client might access any key at any time, you cannot assign keys to cells by client identity. The technique does not generalize to arbitrary access patterns.

**Higher total node count.** Millions of 7-node cells require far more node slots than a single 7-node cluster. Each node stores only kilobytes of data, so the nodes themselves are tiny, but the operational burden of managing millions of independent Paxos groups is real. The colony controller is a significant piece of infrastructure in its own right.

**Colony controller is a soft SPOF for provisioning.** If the controller is unavailable, you can't create new cells or rebalance degraded ones. The hot path (reads and writes) is unaffected, but you can't provision new volumes and you can't respond to node failures. The controller itself needs its own high-availability story.

**Universal code bugs break everything.** A bug in the Paxos implementation or cell daemon code affects every cell simultaneously. Blast-radius containment works for infrastructure failures; it doesn't help when the failure is a universal software defect. The paper acknowledges this and points to rigorous testing (including model checking and fault injection) as the mitigation.

**Full AZ loss still kills its cells.** If an entire AZ suffers a complete node failure (not a network partition, but actual machine failures), cells that placed 4 of 7 nodes in that AZ fall below quorum. Those cells' clients lose access. Physalia bounds this to only those clients, but it doesn't eliminate the impact of full AZ failures.

## Why it works

The foundational principle is **bulkhead isolation**, borrowed from naval architecture: compartmentalize the hull so a breach floods only one section. Each Physalia cell is a bulkhead. A failure can spread only within a cell; it cannot propagate to neighboring cells.

This principle recurs everywhere in reliable systems engineering:
- **AZ and region isolation** in cloud design: deploy across AZs so an AZ failure has bounded blast radius
- **Microservice decomposition**: a service failure affects only its direct callers, not the entire monolith
- **Shard isolation in distributed databases**: a shard failure affects only the data in that shard
- **Circuit breakers**: prevent a failing dependency from dragging down the caller

What Physalia adds is taking bulkheading to the extreme granularity of "one cell serving N clients," rather than the more common granularities of "one AZ" or "one shard." And it adds **topology-aligned placement** to make the most common failure modes (AZ isolation) non-events rather than outages.

There is a second deep principle: **failure mode alignment**. The topology of cell placement is designed so the failure modes that ARE likely — AZ network partitions, rack-level power failures — are exactly the failure modes that leave the quorum intact. You're not just limiting blast radius in general; you're making the specific high-probability failures safe by construction. This is the same engineering instinct behind RAID's choice of XOR-based parity (very fast for sequential writes) over more general error-correcting codes: align your resilience to the likely failures, not the theoretical worst case.

Together: decompose state into independent cells to limit blast radius, then align cell topology with the actual failure distribution to make common failures benign.

## Going deeper

- **Marc Brooker's blog post on Physalia** — https://brooker.co.za/blog/2020/02/17/physalia.html — Brooker's own framing of the key insights in ~800 words; a good companion to the full paper.
- **The NSDI 2020 paper (full)** — https://www.usenix.org/conference/nsdi20/presentation/brooker — includes the placement algorithm, the colony controller design, production availability numbers, and the failure injection experiments.
- **"The Morning Paper" review by Adrian Colyer** — https://blog.acolyer.org/2020/03/04/millions-of-tiny-databases/ — a thorough section-by-section breakdown of the paper, useful if you want a guided reading of the dense parts.
