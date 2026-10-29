---
title: Demystifying Determinism in Durable Execution
source: https://jack-vanlightly.com/blog/2025/11/24/demystifying-determinism-in-durable-execution
author: Jack Vanlightly
company: Independent
date_posted: 2025-11-24
date_digested: 2026-10-29
---

# Demystifying Determinism in Durable Execution

## What's new to learn

1. **Durable execution**: a programming model where function execution is made fault-tolerant by recording every non-deterministic choice into a persistent event journal, then replaying that journal to reconstruct execution state after failure — so a multi-step workflow survives process crashes transparently to application code.

2. **The event journal**: the append-only log of every command issued (activity call, timer, signal) and result received that a durable function has produced; it is the function's write-ahead log at the application layer.

3. **The determinism contract**: *control flow* must be deterministic given the same event history, but *side effects* (external API calls, DB writes) need only be idempotent — the framework records their outcomes and replays them back on recovery without re-executing the external call.

## Prerequisites

- Why idempotency matters: an operation that can safely be applied more than once without changing the result beyond the first application.
- Write-ahead logging: the ARIES principle of appending intent to durable storage before doing the work (covered in this archive, 2026-06-29).
- Replicated state machines: apply the same ordered log of deterministic commands to the same initial state → always arrive at the same final state.

## The core idea

A long-running business process — reserve inventory, charge the customer, send a confirmation email — is just a function that calls external services sequentially. The problem: if the process dies between steps, you've lost track of where you were, and restarting from scratch can mean double-charging.

The naive fix is to checkpoint intermediate state by hand into a database. Durable execution automates this by treating every meaningful event (an activity result, a timer firing, a signal received) as a journal entry that gets persisted **before** the function sees the result. When a process dies, a new worker:

1. Loads the journal from durable storage.
2. Re-executes the function from the top.
3. Whenever it reaches a recorded step, it fast-forwards — returns the stored result without re-invoking the external service.
4. At the first *unrecorded* step, it resumes live execution.

From the function's point of view, it never noticed the crash.

The key insight: **only control flow must be deterministic — the branching logic must follow the same path given the same event history**. The actual outcomes of side effects are replayed from the journal; the framework never re-calls the external service during replay. This is what separates "control flow determinism" from "side-effect determinism."

## Mechanics

**Journal structure:**

Every step creates two sequential entries:

```
CommandEntry  { type: "ActivityInvoked", name: "ChargeCard", args: {...} }
ResultEntry   { type: "ActivityCompleted", name: "ChargeCard", result: {...} }
```

The journal is append-only and immutable. It lives in a database (Temporal's persistence layer, Restate's RocksDB-backed log, or a Postgres table in DBOS).

**Normal execution — three-step workflow:**

```
1. CommandEntry{ReserveInventory} → execute → ResultEntry{ReserveInventory, {ok, sku=A}}
2. CommandEntry{ChargeCard}       → execute → ResultEntry{ChargeCard, {ok, txn_id=42}}
3. CommandEntry{SendEmail}        → ... CRASH HERE
```

**Recovery path:**

A new worker loads the journal and re-runs the function:

- Sees `CommandEntry{ReserveInventory}` + `ResultEntry{ReserveInventory, {ok, sku=A}}` → skip invocation, return `{ok, sku=A}` immediately.
- Sees `CommandEntry{ChargeCard}` + `ResultEntry{ChargeCard, {ok, txn_id=42}}` → skip invocation, return `{ok, txn_id=42}`.
- Sees `CommandEntry{SendEmail}` with no matching result → **resume here**, actually invoke `SendEmail`.

The function's code executes the same `if/else` branches — but completed steps are answered by the journal, not by re-doing the work.

**Where determinism breaks things:**

Anything that influences a branching decision must produce the same value during replay.

```python
# BUG: datetime.now() is different during replay
if datetime.now() < promo_deadline:
    price = promo_price()
else:
    price = full_price()
```

If a crash happens and replay runs at a different wall-clock time, the branch outcome changes. The customer may be charged at full price for an order they placed during a promotion — or, worse, a charge step taken during replay that wasn't taken originally leads to a double-charge.

The fix is to record non-deterministic values as journal entries:

```python
# CORRECT: the framework records "current time" into the journal on first call;
# replay returns the stored value instead of calling the clock again.
now = workflow.now()
if now < promo_deadline:
    price = promo_price()
else:
    price = full_price()
```

Values to always wrap: wall-clock time, random numbers, UUIDs used in branching, and any external read whose value influences which path to take.

**What does NOT need to be deterministic:**

UUIDs or random values used only *inside* an activity (not in control flow) are fine — the activity's result is recorded in the journal and replayed, so the UUID is only generated once even if the activity is fast-forwarded ten times during subsequent replays.

**Idempotency requirement:**

Side effects run at-least-once: if a crash happens after an activity completes but before its `ResultEntry` is persisted, the activity runs again. Therefore **activities must tolerate being called twice with the same inputs** — use idempotency keys at the external service (a `txn_id` passed in the charge request, a conditional `INSERT IF NOT EXISTS`, an S3 `if-none-match` header).

**Workflow code versioning:**

Because replay re-executes historical code, editing a workflow while old instances are in-flight is dangerous. Removing a step that old instances have already recorded leaves orphaned journal entries the new code doesn't know what to do with. Solutions:

- **Version markers** (Temporal's `workflow.get_version()`): insert a branching point that records which code version first executed. Old instances take the old branch; new instances take the new one.
- **New workflow type names**: retire old workflow definitions and launch new instances with new names.
- **Continue-as-new**: truncate the journal at a safe checkpoint, emit a summary state, and restart the workflow with a fresh journal — essential for workflows that run for months and would otherwise accumulate millions of journal entries.

## Where it breaks

**Journal replay latency**: Reconstructing state for a workflow with millions of recorded events takes seconds. Temporal's "continue-as-new" limits journal length; Restate uses a different model that avoids full replay by persisting state snapshots after each step.

**Replay-only bugs are invisible in tests**: A double-charge from a non-deterministic branch only triggers on recovery, not in happy-path testing. These bugs are notoriously hard to reproduce in unit tests.

**Not for high-throughput workloads**: Journal writes add single-digit milliseconds of overhead per step. At sub-millisecond granularity (stream processing, real-time inference), this is unacceptable. Durable execution is suited for "business process" timescales — minutes to months — not microsecond loops.

**Versioning is operationally expensive**: Every breaking change to a running workflow type requires maintaining multiple code versions simultaneously. Teams routinely underestimate the cost of long-lived workflow instances.

**Tight coupling to the orchestrator**: All coordination must be routed through the framework's journal, not through direct service calls or shared queues. Third-party libraries that spawn their own threads or use `asyncio` directly can violate the determinism contract silently.

## Why it works

The deeper principle: **durable execution is ARIES WAL lifted from the storage engine to application code**.

ARIES (the database crash-recovery algorithm) works by:

1. Appending a redo record to the WAL before modifying any page.
2. On crash, replaying the WAL from the last checkpoint — applying redo records to reconstruct committed state.
3. Uncommitted transactions: roll back using undo records.

Durable execution works by:

1. Appending a `CommandEntry` and `ResultEntry` to the journal before the function sees the result.
2. On crash, replaying the journal from the top — returning stored results to reconstruct the function's in-memory state.
3. Unrecorded steps: resume live execution.

The parallel is exact. **The journal IS the redo log.** **Replay IS crash recovery.** The determinism constraint is what makes redo replay produce the same final application state, exactly as ARIES requires that redo operations be idempotent to be replayed safely.

Going further: durable execution is also a single-node replicated state machine. If you replicate the journal across multiple nodes via Raft and route workflow commands through that replicated log, you get a workflow engine that survives arbitrary node failures — which is exactly how Temporal production clusters and AWS Step Functions are implemented. "Durable execution" and "RSM" are the same idea at different abstraction levels: both are the observation that *any deterministic function of an event log can be made fault-tolerant by persisting that log*.

The broader rule: any system that needs to survive partial failures while executing multi-step stateful logic will rediscover this pattern. See it in:

- **Event sourcing (CQRS)**: entity state = fold over event log. Replay gives you any past state.
- **Database WAL (ARIES)**: page state = apply redo log. Replay gives you durable commits.
- **Consensus (Raft)**: cluster state = apply command log. Replication gives you fault tolerance.
- **Durable execution (Temporal/Restate)**: function state = replay event journal. Recovery gives you crash-proof workflows.

All four are the same append-only-log pattern at different layers of the stack.

## Going deeper

1. **Jack Vanlightly's follow-up series** — "The Durable Function Tree" (Parts 1 & 2, December 2025) and "The Three Durable Function Forms" (December 2025) build a formal taxonomy of what durable functions can express and how nesting and fan-out/fan-in behave.

2. **Temporal's architecture docs** — [docs.temporal.io/references/architecture](https://docs.temporal.io/references/architecture) explains how the history, matching, and frontend services distribute and persist event histories at production scale.

3. **ARIES: A Transaction Recovery Method** (Mohan et al., 1992) — the database WAL algorithm whose logic durable execution mirrors at the application layer. The steal/no-force buffer policy and LSN-based idempotent redo are the direct antecedents of the journal's command/result design. Covered in this archive (2026-06-29).
