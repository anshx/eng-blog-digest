---
title: "Dynamo: Amazon's Highly Available Key-value Store"
source: https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html
author: Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, Gunavardhan Kakulapati, Avinash Lakshman, Alex Pilchin, Swaminathan Sivasubramanian, Peter Vosshall, Werner Vogels
company: Amazon
date_posted: 2007-10-01
date_digested: 2026-06-09
---

# Dynamo: Amazon's Highly Available Key-value Store

## What's new to learn

- **Consistent hashing with virtual nodes**: a technique for distributing keys across a ring where adding or removing one node only remaps its immediate neighbors, and multiplying each node's ring presence makes load distribution tunable and heterogeneity-friendly.
- **Merkle tree anti-entropy**: two replicas detect which key ranges diverge in O(log N) round trips by comparing a tree of hashes rather than comparing data directly—binary search applied to replica synchronization.
- **Sloppy quorum with hinted handoff**: instead of requiring writes to reach a fixed set of nodes, Dynamo writes to the first N *available* nodes on the ring and attaches a "hint" so the data can be forwarded to the intended node once it recovers—decoupling durability from availability.

## Prerequisites

- Basic replication concepts: primary/replica, write quorum (W), read quorum (R), N replicas
- The CAP theorem at a hand-wavy level (consistency, availability, partition tolerance—pick two)
- Merkle trees at a surface level (a tree where each parent is the hash of its children) is helpful but not required—the post derives the need from first principles

## The core idea

Dynamo's thesis is that certain Amazon storage workloads—shopping cart, session state, product catalogue—need to *always accept writes*, even during node failures or network partitions. The paper calls this "always-writable." Getting there means explicitly trading away strong consistency: different replicas may hold different versions of the same key simultaneously, and conflicts are reconciled lazily on read, sometimes by the application itself.

Every design choice in Dynamo flows from this commitment:
- Partition data across nodes with consistent hashing so no single coordinator bottlenecks writes.
- Replicate with sloppy quorums so a write succeeds even when some preferred replicas are down.
- Track version history with vector clocks so the system knows *which* versions are concurrent.
- Detect and repair replica divergence with Merkle trees so background anti-entropy is efficient.

The result is a system where P99 write latency is decoupled from whether a subset of nodes is unavailable—at the cost of occasional multi-version reconciliation on read.

## Mechanics

### Partitioning: consistent hashing ring

Dynamo maps both keys and nodes to positions on a 128-bit circular hash space (the "ring"). A key is owned by the first node encountered when walking clockwise from the key's hash. The crucial property: when a node joins or leaves, only its *immediate successor* gains or loses keys—all other key→node assignments remain unchanged. This limits rebalancing work to O(K/N) keys instead of O(K) for a naive modulo scheme.

**The virtual node trick.** A physical node with twice the RAM should own twice as many key ranges. Instead of placing each machine at one ring position, Dynamo gives each physical node dozens of *virtual nodes* (tokens), each at a distinct ring position. Adjust the token count per machine to reflect its capacity. This also improves failure recovery: when one physical node fails, its tokens were spread around the ring, so the load shifts to many different neighbors rather than hammering one successor.

### Replication: preference lists

For each key position, Dynamo takes the next N distinct physical nodes clockwise as the *preference list* for that key (skipping virtual tokens that map to already-included physical nodes). All N nodes receive every write; reads return on R responses; writes complete on W responses. The tunable pair (W, R) with W + R > N gives configurable consistency vs. availability—but Dynamo typically runs at (W=1, R=1) or (W=2, R=2) with N=3 for production latency targets.

### Sloppy quorum and hinted handoff

Classic quorum: "write must reach nodes 1, 2, 3." Dynamo's sloppy quorum: "write must reach the first N *available* nodes clockwise from the key hash." If node 2 is down, node 4 (the next available) steps in temporarily. Node 4 stores the write with a *hint*—metadata saying "this belongs to node 2." When node 2 recovers, node 4 forwards the hinted write and deletes its local copy.

This is the key insight: **durability and availability are decoupled from topology**. A write achieves durability (W copies on disk) even though the preferred topology is broken, and correctness is restored asynchronously.

### Versioning: vector clocks

Each object carries a vector clock—a list of (node, counter) pairs. When node A writes, it increments `A`'s counter. When node B writes after reading from A, it increments `B`'s counter and keeps `A`'s counter. Two versions are comparable (one causally precedes the other) iff one vector clock dominates the other component-wise. Two versions are *concurrent* iff neither dominates—both must be returned to the reader.

The server tries to merge concurrent versions using semantic knowledge (e.g., shopping cart contents can be unioned). If it can't, it returns all concurrent versions to the client, which must reconcile them (e.g., by displaying a "merged cart" UI). This is the "punting conflicts to the application" tradeoff.

**Clock truncation.** In practice, vector clocks grow as coordinators change. Dynamo truncates entries beyond a threshold, keeping only the most recent N (coordinator, counter, timestamp) triples. This sacrifices some causal accuracy—truncated entries may be treated as unrelated when they're actually causally related—to bound clock size.

### Failure detection: gossip-based membership

Every T seconds, each node randomly picks a peer and exchanges membership lists. Within O(log N) gossip rounds, every node learns of any change. This gives eventual convergence for failure detection without a centralized directory. Each node independently tracks which peers it considers "alive" based on recent gossip.

### Anti-entropy: Merkle trees

This is the most transferable mechanism in the paper.

**The problem**: Node A and Node B should hold the same 10 million keys for a given key range. After a network partition, they may have diverged. How do you find *which* keys differ without comparing all 10 million records?

**Naive approach**: compare sorted lists key by key — O(N) data transferred.

**Dynamo's approach**: build a Merkle (hash) tree over the key space.

1. Divide the key space into buckets (leaf nodes).
2. Each leaf holds the hash of all key-value pairs in that bucket.
3. Each internal node holds the hash of its children's hashes.
4. The root hash summarizes the entire key range.

To sync two replicas:
1. Exchange root hashes. If equal → no divergence → done.
2. If unequal → exchange child hashes of the root. Each child covers half the key space.
3. Recurse into the subtree whose child hashes differ.
4. At the leaves, exchange actual key-value data for only the divergent bucket.

This finds the divergent keys in **O(log N) round trips** instead of O(N). The Merkle tree is essentially binary search applied to the question "where did our data diverge?" — each comparison halves the search space.

Each node maintains a separate Merkle tree per virtual node key range, recomputed incrementally as keys change.

### Read path in full

1. Client contacts a coordinator (any node, via load balancer).
2. Coordinator sends read requests to the top N nodes on the preference list (both local and remote).
3. Waits for R responses.
4. If all responses carry the same vector clock → return the single value.
5. If responses carry concurrent vector clocks → return all versions; application reconciles.
6. Coordinator writes the reconciled version back with a synthesized vector clock (read-repair).

## Where it breaks

**Vector clock explosion.** In high-churn write environments with many different coordinators, the vector clock for a hot key can grow large. Dynamo's truncation heuristic loses causal information and can cause false "concurrent" designations—meaning the application sees spurious conflicts that could have been resolved automatically.

**Client-side conflict resolution is hard.** Not all data types have a sensible merge strategy. Dynamo's shopping cart example uses set-union, which can leave deleted items "resurrecting" after a partition heals (a known Dynamo bug that Amazon documented publicly).

**Merkle tree maintenance cost.** Recomputing Merkle trees incrementally is expensive at high write rates. If a leaf bucket covers too large a key range, granularity is poor; if too small, the tree is deep and expensive to maintain. This is a tuning problem with no universal answer.

**Sloppy quorum edge cases.** If the W hinted replicas all crash before forwarding their hints, the write is lost. Durability depends on the probability of multiple correlated failures during the hint-forwarding window. This is usually acceptable but is not the same guarantee as a strict quorum.

**Single-key hotspots.** Consistent hashing distributes keys evenly *in aggregate*, but a hot key (e.g., a trending product) still hits only its preference list's N nodes. Virtual nodes help load distribution across machines but not within a single key.

## Why it works

The deeper principle behind Merkle tree anti-entropy is **hierarchical hash summarization as binary search**. Whenever you need to identify where two large datasets differ, don't compare them element by element—build a tree of hashes and compare from the root down. Every level of agreement halves the search space. You only transfer data where the hashes finally disagree.

This pattern appears throughout distributed systems:
- **Git**: every commit is a Merkle tree. Two git repos detect which objects they need to exchange by comparing tree hashes, not by listing all objects.
- **Bitcoin**: each block header contains a Merkle root of all transactions. Light clients can verify a single transaction's inclusion by checking O(log N) hashes, not downloading the whole block.
- **IPFS and content-addressed storage**: content IDs are Merkle CIDs; sync is comparison of Merkle DAGs.
- **ZFS and Btrfs**: the on-disk format is a Merkle tree over filesystem blocks. Corruption is detected and localized by comparing block hashes up the tree.
- **CockroachDB range sync**: range descriptors are compared via Merkle-like hash layers before transferring actual data.

The broader lesson from Dynamo's overall design: **eventual consistency is a deliberate tradeoff, not a failure of rigor**. By picking A (availability) and P (partition tolerance) over strict C (consistency) in the CAP theorem, Dynamo achieves sub-10ms P99 write latency at Amazon scale. The cost is: application developers must understand and handle concurrent versions. For many write-heavy, read-light workloads (shopping carts, session state), that tradeoff is worth it. For financial ledgers or inventory, it is not. Knowing *which* workloads tolerate eventual consistency is as important as knowing how to implement it.

The (N=3, W=2, R=2) knob is also a transferable pattern—it appears in Cassandra, Riak, Scylla, and DynamoDB. You can slide the consistency-availability dial by adjusting R and W:
- W=1 → lowest write latency, highest risk of dirty reads
- R=N → strong read consistency, worst read latency
- W + R > N → bounded staleness (overlapping quorums guarantee at least one up-to-date replica in every read)

## Going deeper

1. **"Cassandra: A Decentralized Structured Storage System"** (Lakshman & Malik, 2010) — Cassandra combines Dynamo's consistent hashing and vector-clock-free conflict resolution with the column-family data model from Bigtable. Seeing Dynamo's ideas in a second system clarifies which parts are essential vs. Amazon-specific.

2. **"CRDT Primer / CRDTs: The Hard Parts"** (Kleppmann, 2020) — CRDTs are a principled answer to Dynamo's shopping-cart resurrection problem: data types designed so that concurrent writes *always* merge correctly, removing the need for application-level conflict resolution. The archive already has a deep dive on CRDTs that pairs well with this post.

3. **"Merkle DAGs and the IPFS whitepaper"** (Benet, 2014) — Shows Merkle trees generalized from trees to directed acyclic graphs, enabling content-addressed storage, deduplication, and distributed sync without any central coordinator. It is the logical extension of the anti-entropy insight in Dynamo to a fully peer-to-peer setting.
