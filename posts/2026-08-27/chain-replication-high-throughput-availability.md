---
title: "Chain Replication for Supporting High Throughput and Availability"
source: https://www.usenix.org/conference/osdi-04/chain-replication-supporting-high-throughput-and-availability
author: Robbert van Renesse, Fred B. Schneider
company: Cornell University
date_posted: 2004-10-06
date_digested: 2026-08-27
---

# Chain Replication for Supporting High Throughput and Availability

## What's new to learn

1. **Read-at-tail linearizability**: Routing all reads to the chain's last node gives you strong consistency "for free" — no quorums, no lease checks, no log scans — because the tail only ever holds state that every earlier node has already committed.
2. **Pipeline parallelism applied to replication**: Each node in the chain processes a different in-flight write simultaneously, exactly like fetch/decode/execute/writeback stages in a CPU pipeline — giving throughput that scales with chain depth rather than bottlenecking at a single leader.
3. **Failure-mode determinism via append-only pending lists**: Because writes flow in one direction and each node tracks "sent but not yet committed" entries, failure recovery is pointer arithmetic on a list rather than leader re-election and log reconciliation.

## Prerequisites

- Linearizability (strong consistency): every operation appears to take effect atomically at a single point between its invocation and response; reads always see the latest committed write.
- The standard primary-backup replication model: a primary accepts writes and mirrors them to backups; a write is committed when all backups have acknowledged it.
- Fail-stop failure model: a failed node simply stops and never sends another message (no Byzantine behavior, no silent data corruption).
- External failure detector / coordinator (Chubby, ZooKeeper): a separate highly-available service detects node failures and triggers reconfiguration.

## The core idea

Take a cluster of storage servers and arrange them in a linear chain: one **head**, one **tail**, and zero or more **middle nodes** in between. Then impose two rules:

1. **All writes enter at the head.** The head applies the update locally and passes it downstream. Each node in turn applies and passes. The tail applies the update and sends the acknowledgment to the client.
2. **All reads go to the tail.** The tail always reflects the committed state — every write the tail has seen has already been processed by every node ahead of it.

Why does rule 2 give you linearizability for free? Because a write reaches the tail only after it has propagated through every earlier node. The moment the tail sends an ACK to the client, the write is durably stored everywhere. Any subsequent read to the tail necessarily sees that write's effect. There is no gap where a read could return a stale value after a write has been acknowledged — the write's commit point *is* the tail's application of it.

You can teach this to a colleague at a whiteboard with two sentences: "The tail is the truth. If the tail has it, everyone has it."

## Mechanics

### Normal operation

Each node maintains a **pending list**: the sequence of updates it has received but that the tail has not yet acknowledged (i.e., not yet committed). An update moves from "pending" to "acknowledged" when the tail processes it and sends an ACK.

**Write path (object `o`, new value `v`):**

```
client → HEAD → N₁ → N₂ → ... → TAIL
                                  ↓
                               client ← ACK
```

1. Client sends `write(o, v)` to HEAD.
2. HEAD adds `(o, v)` to its pending list, applies the update to local state, and forwards `write(o, v)` to N₁.
3. N₁ adds to its pending list, applies, and forwards to N₂.
4. ... and so on through the chain.
5. TAIL adds to its pending list, applies, and sends ACK to client. The ACK also flows backwards through the chain so each node can remove the entry from its pending list (optional optimization; the pending list can also be purged via sequence numbers).

**Read path (object `o`):**

```
client → TAIL → client (value of o)
```

One round-trip, no coordination.

### Failure handling

Reconfiguration is managed by an external master (the paper assumes a single master; in practice you'd use ZooKeeper/etcd for the master itself). The master monitors the chain via heartbeats and rebuilds the chain atomically when a failure is detected.

**Case 1 — HEAD fails.**  
The master designates N₁ as the new HEAD. N₁'s pending list already contains everything the old HEAD forwarded before dying. N₁ re-sends any entries it has not yet seen propagate past it (i.e., not yet ACKed). No data is lost.

**Case 2 — TAIL fails.**  
The master designates N_(k-1) (second-to-last node) as the new TAIL. N_(k-1)'s local state already includes every committed write (all writes passed through it before reaching the old tail). Pending entries in N_(k-1) are simply promoted to committed, and N_(k-1) begins serving reads. No data is lost, and no recovery replay is needed.

**Case 3 — middle node Nᵢ fails.**  
N_(i-1) must bypass Nᵢ and connect directly to N_(i+1). The problem: N_(i-1) may have forwarded writes to Nᵢ that Nᵢ passed to N_(i+1) *before* failing. N_(i-1) must not re-send those writes. The fix: the master asks N_(i+1) to report the last update it received from Nᵢ. N_(i-1) then compares this against its own pending list and sends only the writes N_(i+1) hasn't seen. A one-time synchronized catch-up, then normal operation resumes.

In all cases, recovery is O(|pending|) — a small bounded set of recent in-flight writes — never O(log) or O(n²) like a full consensus round.

### CRAQ: Distributing Read Load

The basic protocol concentrates all reads on the tail, which becomes a bottleneck for read-heavy workloads. **CRAQ (Chain Replication with Apportioned Queries)** lifts this restriction while preserving linearizability.

Each node tracks the **commit state** of every object it holds:

- **Clean**: the latest version this node holds has been acknowledged by the tail.
- **Dirty**: a newer version has propagated to this node but not yet been acknowledged by the tail.

Read protocol at any node Nᵢ:

```
if object is Clean at Nᵢ:
    return local value  # it's the committed value
else:
    ask TAIL: "what is the latest committed version number?"
    return the local version matching that number
```

When the write commit rate is low (most objects are Clean at any given moment), reads are served locally from every node — linear read scalability. When writes are heavy (more Dirty objects), some reads fall through to the tail, which gracefully degrades to basic chain replication behavior.

CRAQ achieves near-horizontal read scaling while maintaining linearizability. The paper reports 2–3× throughput improvement over basic chain replication under read-dominant workloads.

### Performance characteristics

| Metric | Basic Chain (n nodes) | CRAQ (n nodes) |
|---|---|---|
| Write latency | n RTTs (sequential pipeline) | n RTTs |
| Write throughput | High (pipelined, each node ≈ independent) | Same |
| Read latency | 1 RTT (tail) | 1 RTT (local) or 2 RTTs (tail query) |
| Read throughput | Bounded by single tail | ~n× tail throughput |

Write latency grows with chain length, which is the fundamental cost of requiring every replica to apply the write before commit. This is the same tradeoff as synchronous replication everywhere vs majority quorums in Paxos/Raft: you trade lower latency for stronger guarantees.

## Where it breaks

**Write latency is O(n)** in chain length. For a five-node chain with 1 ms per hop, write latency is at minimum 5 ms — worse than a Paxos/Raft majority quorum of 3/5 nodes at 3 ms. Chain replication optimizes throughput, not latency.

**Tail is a read hot-spot** (in basic chain replication). CRAQ mitigates this but adds the complexity of tracking clean/dirty state per object per node.

**External master is a single point of failure.** In the paper, the master itself is assumed highly available (Chubby, ZooKeeper). If the master is unavailable, the chain cannot be reconfigured after a failure — the system stalls until the master recovers. This doesn't violate consistency (stalled is fine) but it hurts availability.

**No inherent cross-chain transactions.** Each chain manages one object (or one shard). Transactions spanning multiple chains require higher-level coordination (e.g., 2PC) — the same limitation as any sharded storage system.

**Fail-stop assumption.** The protocol assumes failed nodes stop completely. A node that crashes and recovers with stale state (split-brain during reconfiguration) could serve stale reads if it mistakenly believes it is still the tail. The master's epoch / generation counter prevents this in practice, but it's not self-healing — it requires correct master behavior.

## Why it works

The deep principle: **chain replication converts "commit" from a coordination event into a propagation event**.

In Paxos and Raft, committing a write requires a *voting round* — the leader must hear from a quorum of nodes that they received the write. This is fundamentally a broadcast-and-gather pattern: the leader sends to N nodes and waits for N/2+1 responses before declaring the write committed.

Chain replication replaces broadcast-and-gather with a **pipeline**: the write flows through nodes in sequence, and the tail's application of it *is* the commit. There is no gather phase. The head never waits for acknowledgments from multiple nodes — it simply passes the baton and moves on. The pipeline processes multiple writes concurrently, one per node depth.

This is exactly the same insight as CPU instruction pipelining: rather than completing one instruction fully before starting the next (sequential), different pipeline stages work on different instructions simultaneously. The "commit" of an instruction (its writeback) is when it passes through the last stage — the same as the write's commit point being the tail.

The "reads go to tail" rule is the invariant that makes this safe: the tail is the pipeline's "retirement unit" — a write only exits the pipeline (is visible to reads) after it has fully traversed all stages. This is analogous to the **store buffer drain** rule in hardware memory models: a store is visible to other cores only after it drains through the store buffer to the cache coherence fabric.

A second connecting principle: chain replication's failure model is **append-only log management at each node**. Because writes flow in one direction and each node's pending list records what it has forwarded but not yet seen committed, failure recovery is never "figure out who has what" — it's "compare your list to your neighbor's list and fill the gap." This is exactly how Raft's log reconciliation works during leader election (`matchIndex` / `nextIndex`), but in chain replication the flow direction guarantee makes the reconciliation trivially local: only the predecessor-successor boundary matters, not the global log state.

Chain replication thus generalizes the insight that **a total ordering imposed on writes at ingestion time eliminates coordination at read time** — the same principle behind Kafka's partition ordering, the append-only GFS lease model (covered in this archive), and Spanner's commit-wait.

## Going deeper

1. **CRAQ paper (USENIX ATC 2009):** Terrace and Freedman, "Object Storage on CRAQ: High-Throughput Chain Replication for Read-Mostly Workloads." Extends chain replication with the clean/dirty tracking described above and provides a thorough evaluation. Available at https://www.usenix.org/conference/usenix-09/object-storage-craq-high-throughput-chain-replication-read-mostly-workloads

2. **Azure Storage (SOSP 2011):** Calder et al., "Windows Azure Storage: A Highly Available Cloud Storage Service with Strong Consistency." Azure's production implementation uses a chain-replication-based Append Stream Layer for its blob storage. Seeing how the protocol was adapted at hyperscale gives a clear picture of its real-world tradeoffs. Available at https://dl.acm.org/doi/10.1145/2043556.2043571

3. **Vertical Paxos / Reconfigurable Paxos (PODC 2009):** Lamport, Malkhi, and Zhou, "Vertical Paxos and Primary-Backup Replication." Shows how chain replication and Paxos can be unified under a common framework of "primary-order" protocols — useful for understanding why both approaches satisfy the same linearizability requirements via structurally different invariants. Available at https://dl.acm.org/doi/10.1145/1582716.1582783
