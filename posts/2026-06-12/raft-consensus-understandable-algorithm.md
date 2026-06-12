---
title: "In Search of an Understandable Consensus Algorithm (Raft)"
source: https://raft.github.io/raft.pdf
author: Diego Ongaro, John Ousterhout
company: Stanford University
date_posted: 2014-06-18
date_digested: 2026-06-12
---

# In Search of an Understandable Consensus Algorithm (Raft)

## What's new to learn

1. **Replicated state machine.** Any deterministic state machine can be made fault-tolerant by running identical copies on multiple servers that each apply the same sequence of commands from a shared ordered log. "Consensus" is exactly the problem of keeping that log consistent.

2. **Decomposing consensus into three independent subproblems.** Raft's design insight is that leader election, log replication, and the safety invariant can be specified and reasoned about separately. No other part of the algorithm needs to know how the others work.

3. **Log Matching Invariant.** If two log entries at the same index carry the same term number, then both the command stored there and every preceding entry in both logs are identical. This invariant, maintained cheaply during normal operation, is what makes leader elections safe without additional coordination.

## Prerequisites

- **RPC-style distributed systems.** Raft's two message types (RequestVote, AppendEntries) are synchronous request/response RPCs between peers.
- **Majority quorums.** Any two majorities of an N-server cluster share at least one member in common. This intersection property is the only number theory Raft needs.
- **Append-only replicated logs.** If you've read Jay Kreps's "The Log" post (in this archive) you already know why a shared ordered log is the right abstraction; Raft is the protocol that builds one reliably.
- **Epoch fencing.** The idea that a monotonically increasing counter can prove one message is strictly newer than another. Covered in the Kafka transactions post in this archive.

## The core idea

Consensus is just agreement on the ordering of commands written to a log. If every server applies commands in the same order, every copy of the state machine stays identical — regardless of how many messages were dropped or delayed.

The hard part is what happens when the server currently managing the log (the leader) crashes. You need to elect a new leader without losing any command that had already been confirmed to a client. Raft solves this with a simple rule: **only a server with the most up-to-date log can win an election.** "Most up-to-date" is a well-defined partial order on logs, and the election mechanism enforces it without needing to know which entries are committed.

Once elected, the new leader has — provably — every command that was ever confirmed to a client. Normal operation then resumes: the leader appends entries, replicates them to a majority, and commits them. Followers learn about commits from subsequent messages and apply them to their own state machines.

## Mechanics

### Terms — the fencing token

Every Raft server tracks a **current term**, a monotonically increasing integer. Terms play the role of Lamport logical timestamps:

- Every RPC carries the sender's current term.
- If a server sees a higher term in an incoming message, it immediately updates its own term and reverts to follower.
- If a server sees a lower term in an incoming message, it rejects the message.

This means no two servers can ever believe themselves to be the leader in the same term, and messages from stale leaders are silently dropped.

### Leader election

Servers cycle through three roles: follower, candidate, leader.

1. A follower that receives no heartbeat from the leader within an **election timeout** (randomized: 150–300 ms in the original paper) promotes itself to candidate.
2. It increments its current term and broadcasts `RequestVote(term, lastLogIndex, lastLogTerm)` to all peers.
3. A peer grants its vote if and only if:
   - It hasn't already voted in this term, **and**
   - The candidate's log is *at least as up-to-date* as the peer's log: the candidate's last log term is higher, or if tied on term, the candidate's log is at least as long.
4. A candidate that collects votes from a **majority** becomes leader and immediately sends heartbeats to suppress new elections.

Randomized timeouts make it very unlikely that multiple servers time out simultaneously, preventing perpetual split votes. When a split does occur, everyone waits for a new random timeout before trying again.

### Log replication

Once elected, the leader:

1. Appends each incoming client command as a new log entry tagged with the current term.
2. Sends `AppendEntries(prevLogIndex, prevLogTerm, entries[], leaderCommit)` to all followers in parallel.
   - The `(prevLogIndex, prevLogTerm)` pair is a consistency check: the follower rejects the RPC if its own log doesn't match at that position.
   - On rejection, the leader backs up `nextIndex[follower]` and retries with an earlier prefix. (Optimizations exist to speed up this back-off.)
3. Once a majority of servers have acknowledged an entry, the leader advances its `commitIndex`, applies the entry to its own state machine, and responds to the client.
4. The updated `commitIndex` is piggybacked in subsequent AppendEntries; followers apply newly committed entries to their own state machines.

The consistency check in step 2 is what enforces the **Log Matching Invariant**: if the follower's log matches at `prevLogIndex`, every preceding entry is already identical, so appending the new entries keeps the logs consistent.

### Safety — Leader Completeness Property

**Claim:** if a log entry is committed in term T, it appears in the log of every leader elected in any term > T.

**Why this holds:** A committed entry was acknowledged by a majority M₁. For any new leader to win, it must receive votes from a majority M₂. Since any two majorities share at least one member s, server s both acknowledged the committed entry and voted for the new leader. Server s would only vote for a candidate whose log is at least as up-to-date as its own. Since s's log contains the committed entry, the new leader's log must also contain it.

This means the leader never needs to ask "which entries were committed before my election?" The election mechanism guarantees the answer is: all of them.

### Log compaction

As the log grows unbounded, each server independently takes a **snapshot** of its committed state and discards all log entries up to the snapshot point. The snapshot includes the index and term of the last included entry (the "snapshot metadata") so AppendEntries consistency checks can still work at the boundary.

When a follower is so far behind that the leader has discarded the entries it needs, the leader sends an `InstallSnapshot` RPC directly. The follower replaces its state with the snapshot and resumes normal replication from there.

### Membership changes

Adding or removing servers is dangerous: a naive approach can create two non-overlapping majorities simultaneously, leading to two leaders in the same term.

The paper's original approach uses **joint consensus** — a transitional configuration C_old,new that requires a majority of both the old and new cluster for any decision. A simpler approach (now preferred in practice) is **single-server changes**: add or remove exactly one server at a time. With single-server changes, the old and new majorities always overlap and the joint consensus machinery is unnecessary.

## Where it breaks

**Single leader is a throughput ceiling.** All writes serialize through the leader. For write-heavy workloads, the leader's network bandwidth and CPU become the bottleneck. Multi-Raft (one Raft group per shard) is the standard fix, but adds routing and shard-split complexity.

**Disruptive servers.** A server that was partitioned and then reconnects may have an outdated term. If it immediately increments its term and sends RequestVote, it forces all servers to become followers, causing a brief outage. The **PreVote extension** mitigates this by having a candidate first check whether it *could* win a real election before actually incrementing its term.

**Linearizable reads need care.** A leader might serve stale reads if it doesn't know whether a newer leader exists. Options: (1) read-index (leader confirms it's still the leader via a heartbeat round before serving the read), (2) lease-based reads (use a clock-bounded lease, but requires bounded clock drift).

**Snapshot integration is messy.** The paper under-specifies snapshot timing, what to do with in-flight AppendEntries during snapshot installation, and concurrency between the application and the snapshotting thread. Production implementations (etcd, TiKV) spend significant complexity here.

**Log replay on startup.** After a crash, a server must replay all log entries since the last snapshot before it can participate again. For large state machines with old snapshots, this can take minutes.

## Why it works

The deeper principle in Raft is **invariant enforcement at election time, not commit time.** Most distributed protocols pay a coordination cost on every operation. Raft front-loads the cost: the "at least as up-to-date" check in RequestVote is the only place where log completeness is verified, and it runs only during elections (rare events). During the common case — steady-state log replication — the leader just broadcasts entries and waits for a majority acknowledgment, with no extra round trips for safety checks.

This is the same principle as **optimistic concurrency control** (covered in the Aurora DSQL post in this archive). OCC validates at commit time rather than acquiring locks on every read. Raft validates at election time rather than verifying log completeness on every write. Both trade frequent cheap operations for rare but expensive checks, and both rely on an invariant (read-set unchanged / log completeness) that the common case preserves without coordination.

The **term number** is a monotonically increasing epoch that implements the fencing token pattern: a server with a higher term number wins any dispute. This is identical to:
- Kafka's producer epoch (this archive, 2026-06-11) — fences zombie producers after restarts.
- MVCC transaction IDs (this archive, 2026-05-18) — determines write visibility order.
- Spanner's Paxos ballot numbers (this archive, 2026-06-06) — prevents stale leaders from writing.

The **majority quorum intersection** is the number-theoretic bedrock of all quorum-based protocols. Any write acknowledged by a majority, and any future majority that elects a leader, must share at least one server. That one server is the "witness" who ensures the past is not forgotten. The same intersection property makes 2-of-3 replication in Dynamo correct (this archive, 2026-06-09) and read/write quorums in Cassandra safe.

Stepping back: Raft is a constructive proof that **the distributed consensus problem and the distributed log problem are the same problem.** To implement a fault-tolerant log you need consensus; to implement consensus you build a fault-tolerant log. The Raft paper makes this circular dependency tractable by giving a concrete, checkable algorithm for each piece, and by proving that the pieces compose safely.

## Going deeper

1. **"Paxos Made Simple" by Leslie Lamport (2001)** — the two-page distillation of Paxos, which achieves the same result through a different decomposition (prepare/accept phases rather than leader election + AppendEntries). Comparing Paxos Made Simple with Raft reveals exactly which design decisions are essential and which are choices.  
   URL: https://lamport.azurewebsites.net/pubs/paxos-simple.pdf

2. **"Consensus: Bridging Theory and Practice" — Diego Ongaro's PhD thesis (2014)** — the extended treatment of Raft, including a complete TLA+ specification, a formal proof of the safety invariants, and detailed discussion of the membership change and log compaction edge cases the paper glosses over.  
   URL: https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf

3. **TiKV's "Deep Dive into Raft" blog series** — an account of the production issues PingCAP encountered implementing Raft in TiKV (TiDB's storage engine): pre-vote, check-quorum, leader lease, joint consensus for membership changes, and learner replicas. The gap between the paper and a shipping system.  
   URL: https://tikv.org/deep-dive/consensus-algorithm/introduction/
