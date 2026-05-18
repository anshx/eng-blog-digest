---
title: "Snapshot Isolation vs Serializability"
source: https://brooker.co.za/blog/2024/12/17/occ-and-isolation.html
author: Marc Brooker
company: AWS
date_posted: 2024-12-17
date_digested: 2026-05-18
---

# Snapshot Isolation vs Serializability

## What's new to learn

1. **Snapshot Isolation (SI)** — A concurrency level where every transaction reads a consistent snapshot of the database as of its start time, eliminating dirty reads and non-repeatable reads while still allowing one subtle class of anomaly: *write skew*.

2. **Write skew** — The anomaly that SI permits but full serializability prevents: two concurrent transactions each read overlapping data and make writes based on what they read, together driving the database into a globally invalid state — even though neither transaction individually violated any constraint it could see.

3. **The OCC commit-rule asymmetry** — Under optimistic concurrency control, the entire difference between SI and serializable execution is a single word in the commit predicate: SI aborts on *write-write* conflicts; serializable aborts on *read-write* conflicts. That one-word change determines whether the system must track reads, which dominates coordination cost in high-throughput distributed settings.

## Prerequisites

- What a database transaction is and what ACID means at a high level
- MVCC basics: that a write creates a new *version* rather than overwriting in place (the 2026-05-13 DSQL deep dive in this archive covers this in detail)
- What a "commit" is and why concurrent commits can conflict
- Not required: deep knowledge of locking protocols or any particular database

## The core idea

Most engineers think of isolation levels as a ladder — Read Uncommitted → Read Committed → Repeatable Read → Serializable — and assume each rung costs more but is strictly more correct. What Brooker's post reveals is that **Snapshot Isolation sits in a qualitatively different position** from the others: it is almost serializable, defeated only by one very specific interaction that standard isolation analysis glosses over.

Under Optimistic Concurrency Control (OCC), a transaction executes against its snapshot freely with no coordination. At commit time, the system checks whether to accept the transaction. The commit rule for SI is:

> Commit T if and only if T's **write set** does not intersect the **write sets** of any transaction that committed between T's start and T's commit.

The commit rule for full serializability is:

> Commit T if and only if T's **read set** does not intersect the **write sets** of any transaction that committed between T's start and T's commit.

That is the entire difference: `write_set(T)` vs `read_set(T)`. Everything else — the MVCC snapshot, the commit timestamp, the abort-and-retry mechanism — is identical.

Because reads vastly outnumber writes in almost every workload, and because tracking reads requires instrumenting every data access rather than just every data mutation, that one-word change makes serializability dramatically more expensive to enforce, especially in multi-region distributed systems where coordination latency is measured in tens of milliseconds.

## Mechanics

### Why OCC state is cheap

In backward-validation OCC, all decisions are made at commit time using only data from *already committed* transactions. During execution nothing needs to be tracked centrally — no lock table, no read-set registry. If the system crashes mid-transaction, there is nothing to clean up. Write-set records are appended to a durable log at commit time; read sets (under serializable) are ephemeral and only matter during the narrow commit window. This is why OCC scales cleanly in distributed settings: each shard can independently validate its slice of a commit using only its local log of prior commits.

### The write skew pattern

Write skew requires three things simultaneously:

1. Transaction T1 reads some data D.
2. Transaction T2 reads the same (or overlapping) data D.
3. T1 and T2 write to *different* keys, both based on what they read from D.

Because T1 and T2 write to different keys, there is no write-write conflict — SI lets both commit. But if you look at the combined result, the final database state violates an invariant that neither transaction could have violated given what it individually saw.

**Canonical example — doctors on call:**

A hospital requires at least one doctor to be on call at all times. Currently both Alice and Bob are on call.

```
T_alice: read [alice=on_call, bob=on_call]   -- total on call = 2, ok to go off
         write alice = off_call

T_bob:   read [alice=on_call, bob=on_call]   -- total on call = 2, ok to go off
         write bob = off_call
```

Under SI:
- T_alice's write set = `{alice}`, T_bob's write set = `{bob}` → **no write-write conflict → both commit**
- Final state: alice=off_call, bob=off_call → **0 doctors on call, constraint violated**

Under serializable:
- T_bob's read set = `{alice, bob}` intersects T_alice's write set `{alice}` → **abort one**

The constraint violation only becomes visible by combining what two transactions read, which SI does not track.

### Why SI is still usually the right choice

Brooker's argument is that write skew is rare relative to the cost of preventing it. To avoid write skew at SI, a transaction would need to abort whenever any data it *read* was later modified by a concurrent commit — which in a read-heavy workload fires constantly even when no application constraint is at risk. Applications that need exact serializability for specific patterns can opt in selectively with `SELECT FOR UPDATE` (which converts a read into a write for conflict-detection purposes), avoiding the system-wide overhead of full serializable tracking.

In a multi-region deployment like Aurora DSQL, coordination latency is 50–100 ms across regions. Under SI, reads never trigger cross-region coordination (MVCC serves them locally from any replica). Under serializable, every read that is later modified by a concurrent transaction triggers a cross-region abort. At scale, the abort storm caused by read-write conflict detection would erase the throughput advantage of distributing the system in the first place.

### Fekete's anomaly: write skew with a read-only bystander

A particularly subtle variant (Fekete et al., 2004) shows that even a *read-only* transaction can participate in a non-serializable history under SI. Three transactions are involved:

```
T1: writes key 0
T2 (read-only): reads key 0 (sees old value, before T1), reads key 1 (sees T3's update)
T3: reads key 0 (sees old value, before T1), writes key 1
```

There is no serial order for T1, T2, T3 that is consistent with all three observations, yet none of the three has a write-write conflict with the others (T2 doesn't write at all; T1 and T3 write to different keys). SI commits all three; a serializable system must abort at least one.

Brooker notes that this anomaly is a *direct consequence* of the reduced coordination that makes SI scalable. The very same property that lets reads proceed without coordination — that they are not tracked for conflict detection — is what makes this anomaly possible.

## Where it breaks

**Write skew is real, not just theoretical.** Multi-row constraints (account balances summing to a minimum, inventory counts, doctor-on-call minimums) are exactly the pattern write skew exploits. Applications running under SI that assume "the database is serializable" will have latent bugs in these cases.

**`SELECT FOR UPDATE` is a workaround, not a fix.** Promoting specific reads to write-intent reads closes individual skew windows but requires the developer to identify every such constraint and annotate every relevant read. In practice this is error-prone, especially as the schema evolves.

**High write contention still hurts SI.** OCC's optimism breaks down when many transactions compete to write the same rows. Write-write conflicts cause aborts, which cause retries, which cause more conflicts. Under sustained high contention, 2PL systems that queue waiters may outperform OCC systems that burn CPU on aborts.

**Serializable SSI adds runtime overhead.** PostgreSQL's Serializable Snapshot Isolation (introduced in PG 9.1) tracks anti-dependency cycles at runtime and aborts transactions that would form a dangerous cycle. It is significantly smarter than read-set intersection checking, but it still has non-zero overhead and must conservatively abort under uncertainty.

**Long read sets hurt serializable performance.** An OLAP-style transaction that scans 10 million rows has a 10-million-entry read set that must be checked against any concurrent write. Under SI, the same transaction is free.

## Why it works

The deeper principle is that **you only need to coordinate over what can actually conflict**, and what can conflict is determined by what you track.

Write-write conflicts are visible from the write log alone. No extra instrumentation needed — the log already records what was written and when. SI achieves its safety guarantee purely from existing commit-time metadata.

Read-write conflicts require tracking reads, which is structurally more expensive:
- Reads happen continuously throughout execution, not only at commit.
- A single transaction might read thousands of rows; its write set is typically much smaller.
- In a distributed system, read tracking either centralizes through a coordinator (defeating scalability) or is tracked locally and cross-checked at commit (expensive at scale).

This is the same tradeoff that appears throughout distributed systems under the heading of **coordination proportionality**: the system must coordinate *exactly* as much as the guarantee it offers requires, no more, no less. SI offers a weaker-than-serializable guarantee and pays proportionally less coordination cost. Serializable offers a stronger guarantee and must pay proportionally more.

The "X is just Y" insight: **write skew is what you get when two transactions both act as if they are the only writer of an invariant, when the invariant actually spans data that neither one writes.** SI protects the writes; it cannot protect the reads that informed those writes without being serializable. Every other "weaker than serializable" anomaly (dirty reads, non-repeatable reads, phantoms) is in some sense just a special case of "I read something and another transaction changed it." Write skew is the version where the *other* transaction never touched your writes, so the change is invisible to write-conflict detection.

## Going deeper

1. **Cahill, Rőhm, Fekete — "Serializable Isolation for Snapshot Databases" (SIGMOD 2008)** — Introduces Serializable Snapshot Isolation (SSI): rather than tracking full read sets, SSI tracks only *anti-dependency cycles* (the minimal graph structure that signals a serialization failure). This reduces overhead dramatically and is the basis for PostgreSQL's serializable mode since version 9.1.

2. **Fekete, Liarokapis, O'Neil, O'Neil, Shasha — "A Read-Only Transaction Anomaly Under Snapshot Isolation" (VLDB 2004)** — The original paper proving that SI is not serializable even for read-only workloads. The concrete example is compact and worth reading in full; it forced the database community to rethink the relationship between SI and weak serializability guarantees.

3. **Marc Brooker — "What Fekete's Anomaly Can Teach Us About Isolation" (February 2025, brooker.co.za)** — A walk-through of the anomaly in a banking scenario, with the explicit OCC lens showing exactly why SI lets the anomaly pass and serializable catches it. A natural companion to the December 2024 post.
