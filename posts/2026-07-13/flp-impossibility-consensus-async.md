---
title: "A Brief Tour of FLP Impossibility"
source: https://www.the-paper-trail.org/post/2008-08-13-a-brief-tour-of-flp-impossibility/
author: Henry Robinson
company: The Paper Trail (Cloudera)
date_posted: 2008-08-13
date_digested: 2026-07-13
---

# A Brief Tour of FLP Impossibility

## What's new to learn

1. **Bivalency**: A distributed system configuration is *bivalent* if both decision values (0 and 1) are still reachable from it — the defining invariant the FLP proof exploits.

2. **The Scheduler Adversary**: In an asynchronous system, message-delay and process-crash are observationally identical to all other processes; an adversary that can delay messages indefinitely can simulate any crash scenario without actually crashing anything.

3. **Impossibility via bivalency preservation**: Any deterministic consensus protocol must enter a bivalent configuration, and the adversary can always find a single event to keep it bivalent — so the protocol is forced to run forever without deciding.

## Prerequisites

- The consensus problem: N processes with binary inputs that must agree on a single output (Agreement), terminate (Termination), and choose one of the inputs (Validity).
- Message-passing distributed systems: processes communicate only by sending and receiving messages; there is no shared memory.
- Crash failures vs. Byzantine failures: here a faulty process simply stops; it never sends incorrect messages.
- The distinction between synchronous (bounded delay) and asynchronous (unbounded delay) network models — the FLP result applies only to the latter.

## The core idea

FLP proves that **no deterministic algorithm can solve consensus in a fully asynchronous distributed system when even a single process can crash**.

The argument has two moves:

**Move 1 — Any protocol must start uncertain.** Show that every correct consensus protocol has at least one *bivalent initial configuration*: a starting state of the entire system from which both outcomes (decide 0, decide 1) are still possible. This is proved by a connectivity argument: imagine laying out all initial configurations on a line where adjacent configs differ only in one process's input bit. Any protocol's output must flip somewhere along this line, and at that boundary at least one config must be bivalent (both outcomes reachable).

**Move 2 — From uncertainty, uncertainty is always reachable.** Show that from any bivalent configuration, the adversary can find a single event (one message delivery) that leaves the system in another bivalent configuration. This means the adversary can feed the protocol one event at a time, perpetually keeping it in a bivalent state, and the protocol never reaches a decision.

Combine the two moves: the protocol must start in a bivalent state (Move 1), and the adversary can always extend a bivalent state with another bivalent state (Move 2). The protocol runs forever without deciding. Because termination is required for correctness, the protocol is incorrect — contradiction. Therefore no such protocol exists.

## Mechanics

### Formal model

A **configuration** `C` consists of:
- The state of each process (its local variables, including whether it has crashed)
- The set of messages in transit (the *message buffer*)

An **event** `e = (p, m)` is process `p` receiving and processing message `m`. Applying an event to a configuration produces a new configuration: `e(C)`.

A **schedule** `σ` is a sequence of events. `σ(C)` is the configuration after applying all events in order.

A configuration is **deciding** if some process has already output a value. A deciding configuration is **0-valent** or **1-valent** depending on which value it has decided (or will inevitably decide regardless of future scheduling). A configuration is **bivalent** if both outcomes are still reachable via some schedule.

### Lemma 1: Bivalent initial configurations exist

Consider two initial configurations `C₀` and `C₁` that differ only in one process's input (say process `p` holds 0 in `C₀` and 1 in `C₁`). If `C₀` is 0-valent and `C₁` is 1-valent, you can walk the sequence:

```
all-0 inputs  → ... → C₀ → C₁ → ... → all-1 inputs
```

The output must flip somewhere along this line (since the all-0 protocol must decide 0 by validity, all-1 must decide 1). At the flip point, adjacent configurations `C` and `C'` differ only in `q`'s input. Suppose `C` is 0-valent and `C'` is 1-valent. Now simulate `q` crashing at the start. Every other process sees exactly the same messages whether `q`'s input was 0 or 1 — `q` never speaks. So the system from `C` and from `C'` look identical to the rest of the processes. If the protocol decides 0 from `C`, it must also decide 0 from `C'`. But `C'` is supposed to be 1-valent — contradiction. Therefore, *not all initial configurations can be univalent*. At least one is bivalent.

### Lemma 2: Bivalent configurations beget bivalent configurations

Let `C` be a bivalent configuration and `e = (p, m)` any event applicable to it. We want to show there exists a sequence of events from `C` that applies `e` last and reaches a bivalent configuration.

Let `D` be the set of all configurations reachable from `C` *without* applying `e`. Consider `e(D) = {e(D') : D' ∈ D}` — what you get by applying `e` to everything in `D`.

Because `C` is bivalent, some schedule from `C` reaches a 0-valent deciding configuration, and some reaches a 1-valent one. The adversary can choose to suppress event `e` until just before one of these decisions, or let it through before the other. This means `e(D)` must contain both 0-valent and 1-valent configurations.

By a "neighborhood" argument: consider a configuration `E ∈ D` adjacent to `e(E)` such that `e(E)` is univalent — say 0-valent. Because `D` also contains paths to 1-valent configurations (from the above), there must be a config `F ∈ D` adjacent (via one event `f ≠ e`) such that `e(F)` is 1-valent. The two configs `e(E)` (0-valent) and `e(F) = f(e(E))` (1-valent) differ by one event `f`. But if `f` involves a process *different* from `p`, the events commute: `f(e(E)) = e(f(E))`. So `e(E)` (0-valent) and `e(f(E))` (1-valent) are both reachable by applying `e` to neighbors in `D` — meaning `e(E)` can actually reach both outcomes, so it must be *bivalent*, not 0-valent. Contradiction.

The case where `f` involves the *same* process `p` as `e` uses the fact that `p` can only process messages one at a time in FIFO order, which can also be shown to yield a bivalency contradiction.

In short: you can never reach a state where applying event `e` to *every* bivalent configuration leads to a univalent result. There's always an escape hatch back to bivalency.

### Putting it together

The adversary runs the following strategy:
1. Start in the guaranteed bivalent initial configuration (Lemma 1).
2. At every step, pick *any* pending event; by Lemma 2, there exists an ordering of events such that the system stays bivalent.
3. To keep processes "live" (so the protocol doesn't give up because some process never gets scheduled), deliver messages in a round-robin fashion — just delay the "critical" one each step.
4. The protocol never decides, yet all processes keep running. It fails termination.

## Where it breaks

**Only applies to purely asynchronous models.** The moment you add any synchrony — a timeout, a global clock, a message-delivery bound — the FLP result no longer applies. Practical systems escape it by imposing *partial synchrony*: they assume "eventually messages will be delivered within δ" and use heartbeats/timeouts as a proxy for that assumption.

**Only applies to deterministic algorithms.** Ben-Or (1983) showed that randomized consensus *can* terminate in expected finite time in an async system. The adversary can't always find the "right" event to delay because the process's next action is unpredictable. Practical randomized protocols (Rabia, PBFT with randomization) use this escape hatch.

**Worst-case, not typical-case.** The adversary's strategy requires perfect knowledge of the protocol and the ability to control scheduling precisely. Real networks have bounded-enough delay that protocols like Raft converge in milliseconds. FLP says bad executions *exist*, not that they're common.

**Says nothing about failure detectors.** Chandra and Toueg (1996) showed that even the *weakest eventually-reliable* failure detector (one that eventually stops suspecting correct processes) is enough to solve consensus. Modern systems add timeout-based failure detection and effectively "lift" themselves out of the purely async model.

**Crash-stop only.** FLP assumes processes either run correctly or silently stop. Byzantine failures (processes that send lies) require separate analysis (BFT protocols, e.g., PBFT, Tendermint).

## Why it works

The deep principle: **uncertainty cannot be eliminated without assumptions**. FLP is a formal statement of the fact that in a system with unbounded asynchrony, you cannot tell a dead process from a slow one. That observational equivalence lets the adversary simulate crashes without crashing anything, which means any decision made "in the presence of a possibly-crashed process" can be wrong.

This is the same argument structure as:

- **Halting problem** (Turing 1936): you cannot determine if an arbitrary program terminates because any "it won't halt" oracle can be subverted by a program that queries the oracle and does the opposite.
- **Sorting lower bound**: you cannot sort n elements in fewer than Ω(n log n) comparisons because each comparison resolves only one bit of uncertainty and you have log₂(n!) bits of total uncertainty.
- **CAP theorem** (Brewer 2000, Gilbert & Lynch 2002): in the presence of a network partition, you cannot have both consistency and availability, because the two sides of the partition cannot communicate to agree.

In all three cases, the proof constructs an *adversary* (a pathological input, a pathological comparison sequence, a pathological network partition) that defeats any candidate algorithm. FLP's adversary is a scheduler that controls message timing.

The practical takeaway for distributed systems engineers:

- **Every consensus protocol either adds synchrony assumptions or weakens the guarantees.** Raft uses election timeouts (partial synchrony). Cassandra uses quorums with eventual consistency (weakened agreement). Paxos uses timeouts and leader leases (partial synchrony). They *cannot* avoid this tradeoff — FLP proves it.
- **Heartbeats are not an optimization; they are a requirement.** Without a way to distinguish slow from crashed, the protocol cannot move forward. Heartbeat timeouts inject just enough synchrony to escape the FLP bound.
- **FLP explains why "always consistent, always available, always partition-tolerant" is impossible.** The three goals are in fundamental tension because they all require distinguishing crashed from slow processes — which async systems cannot do.

The meta-lesson: when a systems paper claims impossibility, look for the *adversary construction*. The proof's power comes from showing that *for any algorithm*, there exists a *pathological execution* that breaks it. Understanding what the adversary controls tells you exactly what assumption you need to escape the impossibility.

## Going deeper

1. **The original paper**: Fischer, Lynch, Paterson. "Impossibility of Distributed Consensus with One Faulty Process." *Journal of the ACM*, 32(2), April 1985. The proof is dense but only 10 pages; worth reading once the intuition is clear.

2. **"Consensus in the Presence of Partial Synchrony"** — Dwork, Lynch, Stockmeyer (1988). Introduces the partial synchrony model (the formal foundation of Raft and Paxos), showing consensus *is* solvable once you bound eventual message delay. This is the direct theoretical answer to FLP.

3. **"Unreliable Failure Detectors for Reliable Distributed Systems"** — Chandra and Toueg (1996). Proves that augmenting an async system with even the weakest failure detector (⋄W — "eventually weakly accurate") suffices to solve consensus. Maps directly to how real systems use heartbeats and timeouts to "implement" a failure detector in practice.
