---
title: "Calvin: Fast Distributed Transactions for Partitioned Database Systems"
source: http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf
author: Alexander Thomson, Thaddeus Diamond, Shu-Chun Weng, Kun Ren, Philip Shao, Daniel J. Abadi
company: Yale University
date_posted: 2012-05-20
date_digested: 2026-07-29
---

# Calvin: Fast Distributed Transactions for Partitioned Database Systems

## What's new to learn

- **Deterministic transaction execution** — when every node agrees on the *input order* of transactions before any execution begins, the outcome is predetermined; no inter-node coordination is needed during execution itself.
- **Sequencer as consensus substrate** — a Paxos/Raft-backed epoch-batching log can serve as the sole coordination point for a distributed database, replacing the role that 2PC plays in traditional systems.
- **Read-write set pre-declaration** — transactions must expose every key they will read or write before execution starts, enabling the scheduler to assign locks in a deterministic order using only local state.

## Prerequisites

- What two-phase commit (2PC) does and why it is expensive under contention: the coordinator must wait for lock-holding participants to vote before the transaction can release its locks.
- What makes a transaction history serializable: conflict-equivalence to some serial order.
- The Raft/Paxos insight that a replicated, ordered log plus deterministic state-machine application yields consistent replicas — covered in the [Raft deep dive](../2026-06-12/raft-consensus-understandable-algorithm.md).
- The "log is truth" framing from Jay Kreps' [The Log](../2026-05-25/the-log-what-every-engineer-should-know.md).

## The core idea

Every distributed transaction protocol in the archive solves the same problem: nodes that hold locks on different shards must somehow agree whether to commit or abort, *while those locks are held*. Two-phase commit takes two round trips under lock. Optimistic concurrency control (Percolator, Aurora DSQL) avoids locks but pays for read-set validation at commit time. In both cases, the agreement happens *during* execution.

Calvin's realization: if you move the agreement *before* execution, the execution itself needs zero coordination.

The mechanism is a **sequencer layer** that collects incoming transactions into 10 ms epochs and replicates each epoch's batch across all nodes via Paxos or Raft. By the time the first execution thread touches any data, every node in the cluster has already agreed, in writing, on the complete ordered list of transactions it will execute. Because execution is a deterministic function of that list, every node reaches the same result independently. There is no need to ask anyone "are you ready to commit?"

The mental model: Calvin is Raft's state-machine replication applied one layer higher. Raft says "agree on the log of commands, apply them deterministically, replicas converge." Calvin says "agree on the log of *transactions*, execute them deterministically, shards converge."

## Mechanics

### Three layers

**Sequencing layer.** Each node runs a sequencer process. Within each 10 ms epoch, the sequencer on each shard accepts client transactions destined for its shard and buffers them locally. At the epoch boundary, all sequencers exchange their buffers (a small broadcast — each shard only sees transactions that touch it, not the global set). The result is a *globally ordered, per-shard replicated* batch of transactions. Replication of this batch uses a consensus protocol (Paxos or Raft); the batch is committed to the log before any execution begins.

**Scheduling layer.** The scheduler reads the ordered transaction batch and acquires locks using a **deterministic locking algorithm**:

1. For each transaction T in epoch order, the scheduler requests locks on all keys in T's read-write set.
2. If two transactions T₁ < T₂ both want a write lock on key K, T₁'s lock request is always processed first (because T₁ comes first in the global order). Locks are granted in that predetermined sequence — no deadlock is possible, and no inter-node negotiation is needed.
3. Once a transaction acquires all its locks, an execution thread picks it up and runs it.

**Storage layer.** Regular single-node key-value or relational storage. The scheduler hands off a transaction to storage exactly once, under its locks, to completion.

### The remote-read protocol: the one coordination step that remains

Suppose transaction T needs to read key K from shard B but execute its write logic on shard A. Shard A cannot proceed until it has K's value. Calvin handles this with a single pre-execution round trip, not a 2PC:

1. At the start of the epoch, each shard identifies which transactions in its batch need remote reads from other shards.
2. A shard sends read requests for those keys to the owning shard.
3. Owning shards reply with current values (they can safely do so because the global order is already fixed — no future transaction in this epoch can change those keys before this one commits).
4. Execution then proceeds locally, with all needed data in hand.

This is one message round trip (request + reply), not two (prepare/vote + commit), and it happens *without holding any write locks*.

### Epoch batching as a throughput amplifier

Because Calvin groups transactions into batches before lock acquisition, many non-conflicting transactions in the same epoch can execute in parallel. The scheduler uses as many execution threads as the hardware has cores; threads pick up transactions as their locks become available. Under low contention this parallelism is nearly unlimited. Under high contention, conflicts serialize only the conflicting subset, not the whole batch.

### Replication modes

Calvin exposes three replication options because the sequencer layer is the only place coordination happens:

- **Asynchronous** — sequencer does not wait for replicas; fast but not synchronously durable.
- **Paxos-based synchronous** — sequencer waits for a quorum of replicas to acknowledge the epoch batch; strong durability at the cost of one quorum round trip per epoch (not per transaction).
- **Geo-replicated** — run the Paxos quorum across datacenters; add geo-latency only to the epoch, not to individual transactions.

All three modes provide full ACID semantics during normal operation.

## Where it breaks

**Read-write set pre-declaration is the central constraint.** Transactions must enumerate every key they will read or write *before execution begins*. This is fine for templated transactions ("transfer $N from account A to account B" — keys are A and B, known from the inputs). It breaks for conditional logic: "if balance > threshold, update accounts X and Y; otherwise update Z." Calvin offers a workaround — declare all *possible* keys pessimistically — but this inflates the contention footprint and wastes lock capacity on keys the transaction might not touch.

A later paper by Lu et al. (Aria, VLDB 2020) removes this requirement by using an optimistic retry mechanism, at the cost of some aborts.

**Epoch latency is a floor on commit latency.** A transaction submitted at the start of a 10 ms epoch commits no earlier than the end of that epoch plus execution time. For workloads requiring sub-millisecond round trips, Calvin's batching adds unavoidable latency. Systems like VoltDB that also use deterministic execution tune this epoch duration carefully.

**Single active master per shard.** All writes to a shard go through that shard's sequencer. There is no multi-master write path. This is intentional — two sequencers on the same shard would need to agree on order, reintroducing coordination — but it means write throughput is bounded by a single sequencer per shard.

**Sequencer is a single point of failure (mitigated by Paxos).** If the sequencer fails mid-epoch, a new epoch must begin and in-flight transactions are retried. Paxos/Raft election adds a gap; Calvin's designers accept this by tuning epochs to be short enough that retries are rare.

**Hot keys serialize globally.** If every transaction in an epoch touches the same key, Calvin's deterministic lock ordering is equivalent to a global serial execution of those transactions. The protocol does not re-shard hot keys or employ any form of key splitting — that is left to the application or an upper layer.

## Why it works

### The deep principle: move agreement before execution

Two-phase commit is expensive for one reason: the "prepare" message is sent *while locks are held*. Waiting for all participants to vote means holding those locks for the duration of at least one network round trip. Under contention, this compounds — if the prepare message is slow (network jitter, GC pause, a disk write), every other transaction that wants those keys waits.

Calvin's key insight is that 2PC is solving the wrong subproblem. The real question is: "what order will all nodes execute this set of transactions in?" 2PC answers this during execution by negotiating commit. Calvin answers it *before* execution by establishing a replicated log. Once the answer is in the log, no negotiation is needed — the outcome is already determined.

This is the same structural insight as:

- **Raft / Paxos state machine replication**: agree on the command log, apply commands deterministically, all replicas converge. Calvin is state machine replication where the "state machine" is a partitioned relational database and the "commands" are multi-shard transactions.
- **Jay Kreps' "The Log"**: making the log the source of truth for ordering collapses the N×M coordination problem into N+M subscriptions. Calvin makes the transaction input stream that log.
- **Event sourcing**: the event stream defines state. The sequencer's output is Calvin's event stream; each shard's storage is the materialized view.
- **Batch processing**: MapReduce and Spark achieve coordination-free parallelism by agreeing on the input set first, then executing a deterministic function. Calvin does the same at the transaction level.

What all of these share: the agreement happens once, on an immutable ordered record, before any mutable state is touched. Execution then becomes a pure function of that record.

### Why 2PC is not just slow but structurally wrong for replication

A subtler point: 2PC is not just a performance problem, it is an architectural mismatch with replication. If you run 2PC across replicas, you need 2PC to agree *between replicas* as well — this is why cross-datacenter 2PC is so painful. Calvin sidesteps this entirely: replication happens at the sequencer level (Paxos/Raft), and because replicas replay the same ordered log, their state converges automatically. Two-phase commit is never needed — not between shards, not between replicas, not across datacenters.

## Going deeper

1. **The original Calvin paper** (Thomson et al., SIGMOD 2012) — http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf — all the gory details on the deterministic locking algorithm, the remote read protocol, and the epoch replication mechanism.

2. **"An Evaluation of the Advantages and Disadvantages of Deterministic Database Systems"** (Thomson & Abadi, VLDB 2014) — a follow-up that honestly assesses when Calvin wins and when it loses compared to optimistic and pessimistic systems; essential for understanding the pre-declaration tradeoff in practice.

3. **Aria: A Fast and Practical Deterministic OLTP Database** (Lu et al., VLDB 2020) — https://vldb.org/pvldb/vol13/p2047-lu.pdf — removes Calvin's pre-declaration requirement by running a speculative first pass and retrying only conflicting transactions; shows how far the deterministic idea can be pushed.
