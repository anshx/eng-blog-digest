---
title: "Transactions in Apache Kafka"
source: https://www.confluent.io/blog/transactions-apache-kafka/
author: Matthias J. Sax (Confluent Engineering)
company: Confluent / Apache Kafka
date_posted: 2017-11-01
date_digested: 2026-06-11
---

# Transactions in Apache Kafka

## What's new to learn

1. **Zombie fencing via monotonically increasing epochs** — A producer that crashes and restarts bumps an epoch counter associated with its stable identity; any subsequent write from the old (lower-epoch) instance is rejected at the broker, deterministically killing the stale "zombie" before it can corrupt state. This is a narrower, cheaper form of the same primitive that Raft uses to fence old leaders.

2. **Exactly-once is a composition of three orthogonal guarantees** — Idempotency (suppress duplicates within a session), Atomicity (multi-partition writes succeed or fail together), and Isolation (readers don't observe partial transactions) are solved by three separate mechanisms that can each be used independently. Conflating them into one concept is the source of most confusion about whether "exactly once" is even possible.

3. **LSO (Last Stable Offset) as a consumer-facing read cursor** — Consumers with `read_committed` isolation only see messages up to the highest offset at which all in-flight transactions are resolved. Long-running transactions hold this cursor back, trading freshness for consistency — the exact same head-of-line-blocking tradeoff as TCP.

## Prerequisites

- Basic Kafka model: topics, partitions, producers, consumer groups, broker replication.
- At-least-once vs at-most-once delivery: why retries produce duplicates, why no-retry drops messages.
- The concept of a distributed log as the source of truth (WAL analogy).
- Two-phase commit at a high level (enough to know that "prepare" → "commit" splits a decision into a durable record plus a distribution step).

## The core idea

The question "is exactly-once delivery possible in a distributed system?" has a classic answer: *in general, no* — failures during commit can leave the system in an unknown state. Kafka's answer is: yes, within a Kafka cluster, if you define "exactly once" precisely and solve three sub-problems separately.

The insight is this decomposition:

```
Exactly once = Idempotency + Atomicity + Isolation
```

**Idempotency** means that if the producer retries a send (because the broker acked but the ack was lost), the broker deduplicates it. The producer is no longer "fire and forget" — it has an identity (Producer ID + epoch) and a sequence number per partition. The broker rejects anything out of order.

**Atomicity** means that writes to multiple partitions (or a consumer offset commit + a downstream topic write) are a single all-or-nothing operation. A Transaction Coordinator broker manages a two-phase commit over the involved partition leaders.

**Isolation** means that consumers with `isolation.level=read_committed` see only fully committed transactions. The broker tracks an LSO — the first offset where an open transaction begins — and withholds everything at or beyond that boundary.

Each layer is independently useful and independently deployable. Understanding them separately is the key to reasoning about where exactly-once actually holds and where it breaks.

## Mechanics

### Layer 1 — Idempotent Producer (PID + sequence numbers)

When a producer enables `enable.idempotence=true`, the broker assigns it a **Producer ID (PID)** — a 64-bit integer unique to that session. Every batch the producer sends carries:

```
(PID, partition, sequence_number)
```

The sequence number is monotonically increasing per (PID, partition) pair, starting at 0. The broker tracks the last sequence number seen for every (PID, partition) and applies three rules:

- `seq == last + 1` → accept (normal case)
- `seq <= last` → reject as duplicate (producer retry)
- `seq > last + 1` → reject as gap (out-of-order delivery, data loss risk)

Crucially, these sequence numbers are **persisted to the replicated log**, not just kept in broker memory. If the partition leader fails and a follower takes over, the new leader inherits the full sequence history. This is unlike TCP, where sequence numbers reset on reconnection and crash recovery is left to the application.

This layer alone eliminates within-session duplicate sends. It does not survive producer restarts (new session = new PID), and it doesn't address multi-partition atomicity.

### Layer 2 — Producer Epochs and Zombie Fencing

For stream-processing workloads ("read from topic A, transform, write to topic B, commit consumer offset"), a persistent identity across producer restarts is essential. This is the `transactional.id` — a human-provided string that uniquely identifies a *logical* producer (e.g., a specific task in a stream topology).

When the producer calls `initTransactions()`, the **Transaction Coordinator** (described below) looks up the transactional.id and **bumps the epoch** — a 16-bit monotonically increasing counter stored durably in the transaction log. This epoch is then attached to all messages from this new producer instance.

The fencing rule is simple: brokers reject any write from a (transactional.id, epoch) pair where the epoch is lower than the current maximum known for that id. When the new instance bumps the epoch to `N+1`, the old instance — possibly still alive and attempting to write with epoch `N` — immediately starts receiving `ProducerFenced` errors and must halt.

```
Old instance          New instance
epoch=5               calls initTransactions()
  |                         |
  |                   Coordinator bumps epoch to 6
  |                         |
  | send(msg, epoch=5)      |
  |→ PRODUCER_FENCED        |
  | (must stop)             |
```

This is deterministic termination of the stale writer without any external process management. The epoch acts as a *fencing token* — the same concept Martin Kleppmann introduces in DDIA for distributed lock safety, and the same mechanism Raft uses with term numbers to prevent old leaders from writing after a new election.

### Layer 3 — Transaction Coordinator and Two-Phase Commit

The Transaction Coordinator is a special role any broker can play, assigned by hashing `transactional.id mod num_partitions` over the internal `__transaction_state` topic (50 partitions by default). This ensures each transactional.id has exactly one responsible coordinator.

The coordinator maintains a **state machine** per transactional.id, persisted as a WAL to its owned partition of `__transaction_state`:

```
Empty → Ongoing → PrepareCommit → CompleteCommit
                → PrepareAbort  → CompleteAbort
```

A full transaction lifecycle:

1. **`initTransactions()`**: Producer contacts coordinator; coordinator bumps epoch, writes new producer state to __transaction_state, returns (PID, epoch) to producer.
2. **`beginTransaction()`**: Local call; no coordinator contact yet.
3. **`send(record)` to partition P**: Producer sends `AddPartitionsToTxn` to coordinator (first write to P only) — coordinator records that P is part of this transaction in its WAL.
4. **`commitTransaction()`**:
   - Producer sends `EndTxn(COMMIT)` to coordinator.
   - **Phase 1**: Coordinator writes `PrepareCommit` to `__transaction_state`. This is the **point of no return**. If coordinator crashes here, it will resume Phase 2 on recovery.
   - **Phase 2**: Coordinator sends `WriteTxnMarkers(COMMIT)` RPCs to every involved partition leader. Leaders append an `EndTransactionMarker` control record to their logs.
   - Coordinator writes `CompleteCommit` to `__transaction_state`. Done.

Abort follows the same pattern with `PrepareAbort` + `WriteTxnMarkers(ABORT)`.

For the read-process-write pattern, the producer also calls `sendOffsetsToTransaction(offsets, consumerGroupId)`. This causes the coordinator to register the `__consumer_offsets` partition as a participant, so the consumer offset commit is atomically bundled with the output writes. Either both the output records and the offset commit are visible, or neither is.

### Layer 4 — Read Committed Isolation (Consumer Side)

The `EndTransactionMarker` records written in Phase 2 are the mechanism for consumer isolation. A consumer with `isolation.level=read_committed` sees messages only up to the **Last Stable Offset (LSO)**:

```
LSO = min(first_offset_of_any_open_transaction, log_end_offset)
```

Concretely: if transaction T spans offsets 100–150 and is still open, the LSO sits at 100. Consumers can read up to offset 99 regardless of whether other offsets beyond 150 are available. When T commits (or aborts), its EndTransactionMarkers are written, LSO advances past 150, and consumers catch up.

For aborted transactions, the broker also maintains an **aborted transaction index** per segment. Consumers request this index alongside their fetch response and use it to skip over aborted-transaction records without reading them.

A fetch response to a `read_committed` consumer carries:
- Records up to LSO
- The list of aborted transaction ranges within that fetch window (so consumer can skip them)

## Where it breaks

**Head-of-line blocking from the LSO.** A long-running or stuck transaction pins the LSO and prevents all `read_committed` consumers from advancing past its start offset, even for unrelated messages written later. This is the exact tradeoff TCP makes: the receive buffer won't deliver segment 5 to the application until segments 3 and 4 arrive, even if segment 6 is already there.

**Coordinator overhead on every transaction.** Each transaction requires at least 2 durable writes to `__transaction_state` (PrepareCommit + CompleteCommit) plus N `WriteTxnMarkers` RPCs, one per involved partition. High-throughput workloads that batch hundreds of messages into one transaction amortize this well; workloads that create a transaction per message do not.

**Exactly-once ends at the Kafka cluster boundary.** EOS guarantees hold only for writes and reads within a single cluster. If the stream processor writes to an external database or calls a third-party API inside the processing loop, those operations are outside Kafka's transaction and require their own idempotency mechanisms. "Exactly once to Kafka" does not mean "exactly once to your whole system."

**Epoch space is 16 bits.** The producer epoch is a 16-bit counter. After 65,535 restarts with the same `transactional.id`, the epoch wraps. In practice this is never reached, but it is a hard limit in the protocol design (KIP-360 addressed this by resetting on epoch overflow rather than rejecting the producer forever).

**Zombie fencing only works if the transactional.id is stable.** If two tasks in a stream topology accidentally share a transactional.id, they will fence each other. The ID must be globally unique per logical writer and stable across restarts; this is entirely the application's responsibility.

## Why it works

The deeper principle is that **"exactly once" is not a single property but a composition of three orthogonal concerns**, each of which has a well-understood solution:

| Concern | Classic solution | Kafka's solution |
|---------|-----------------|-----------------|
| Duplicate suppression | Sequence numbers | PID + per-partition sequence |
| Multi-party atomicity | Two-phase commit | Transaction Coordinator + WAL |
| Read isolation | MVCC / snapshot | LSO + EndTransactionMarkers |

Each mechanism is an instance of something general:

**Epoch-based fencing is Lamport logical clocks applied to a producer identity.** The epoch is a monotonically increasing number that defines a happens-before relationship: epoch N+1 strictly follows epoch N. Any write from epoch N is provably older than any write from epoch N+1, so brokers can safely fence old writers — exactly as Raft rejects messages from terms older than the current term, or MVCC rejects writes from transaction IDs older than the snapshot.

**The Transaction Coordinator is a log-backed state machine, not a lock.** The coordinator does not hold a lock on the partitions in the transaction. It only writes intent (PrepareCommit) to a durable log, then distributes the decision. Participants (partition leaders) independently apply the decision once they receive it. Durability of the intent survives coordinator crashes. This is the same reason 2PC works even when the coordinator restarts: the prepare record is the commit point.

**LSO-based isolation is epoch-based MVCC without versioned storage.** Kafka's log is append-only; you can't go back and mark old records as "invisible." Instead, the LSO is a read pointer that only advances when all in-flight transactions at or below it are resolved — exactly what MVCC's transaction snapshot does for point-in-time reads, but expressed as a per-partition watermark rather than per-row version chains.

The composite insight: Kafka's EOS is the same machinery as a distributed database's transaction system, implemented over an append-only log where the "database" *is* the log. The log-native form of MVCC is the LSO; the log-native form of fencing is the epoch; the log-native form of 2PC is writing prepare/commit records into a dedicated coordinator partition.

## Going deeper

1. **KIP-98 — Exactly Once Delivery and Transactional Messaging** (Apache Kafka Wiki): The original design document, written by Jay Kreps, Jun Rao, Guozhang Wang, and others. Contains the full state machine, protocol message formats, epoch semantics, and the consumer group integration. More complete than any blog post.

2. **Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 9 ("Consistency and Consensus")**: The section on fencing tokens (pages 302–303) and the section on exactly-once stream processing (pages 476–479) show how the same epoch-fencing and 2PC patterns appear in distributed locks, ZooKeeper, and stream processors beyond Kafka. Reading this alongside KIP-98 makes the generalization concrete.

3. **"Transactions in Apache Kafka" follow-up — "Exactly-once Semantics is Possible: Here's How Apache Kafka Does it"** (Confluent Engineering Blog, 2017, https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/): Jay Kreps' companion post that explains the motivating scenarios (stream processing zombie producers) and why prior "proofs" that exactly-once is impossible were solving a different problem (cross-system, not intra-system).
