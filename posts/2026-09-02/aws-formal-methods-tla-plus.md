---
title: "How Amazon Web Services Uses Formal Methods"
source: https://cacm.acm.org/research/how-amazon-web-services-uses-formal-methods/
author: Chris Newcombe, Tim Rath, Fan Zhang, Bogdan Bogdanov, Michael Rickard, Jonathan Allen, Nate Hochstein, Mark Santarosa
company: Amazon Web Services
date_posted: 2015-04-01
date_digested: 2026-09-02
---

# How Amazon Web Services Uses Formal Methods

## What's new to learn

1. **Temporal logic specifications**: A distributed system's design can be expressed as a state machine where correctness invariants are written in temporal logic — `□P` (always P holds, safety) and `◇P` (eventually P holds, liveness) — and a model checker exhaustively verifies those invariants over *every reachable state*, not just the test cases an engineer thought of.

2. **State space enumeration as proof**: TLC (the TLA+ model checker) performs BFS on the directed graph of all possible system states, which subsumes any hand-written test suite by covering every possible interleaving of every process and network event at the design level. Where testing samples paths, model checking enumerates them.

3. **Spec-before-code engineering**: Writing a formal spec before implementation forces engineers to make implicit assumptions explicit and to enumerate all possible event sequences rather than the "happy path." At AWS, this caught bugs that escaped both code review and informal proofs, including a 35-step failure sequence in DynamoDB that would have required a very specific network partition pattern to trigger in production.

## Prerequisites

- Finite state machines: what states and transitions are
- The difference between safety ("nothing bad happens") and liveness ("something good eventually happens")
- What it means for a distributed system to have a "state" — variable assignments, process roles, in-flight message sets

## The core idea

Informal reasoning about distributed systems is fragile because humans reason about *one execution at a time*. When a design says "the leader commits a write after a quorum acknowledges it," a human verifying this will trace through a few representative scenarios. But the real question is: does the invariant hold for *all* possible interleavings of all processes, crashes, network delays, and retries?

TLA+ (Temporal Logic of Actions, designed by Leslie Lamport) makes this question answerable. You write a *specification* — a mathematical description of what the system does, not how it does it — and then the model checker TLC answers whether your stated invariants hold across every reachable state.

A TLA+ spec has three parts:

1. **State variables**: the things that can change (`leader`, `data`, `replica_states`, `messages_in_flight`)
2. **Actions**: predicates over current-state and next-state variables describing valid transitions (`Commit`, `FailLeader`, `NetworkDrop`)
3. **Temporal properties**: what must always be true (`□NoSplitBrain`), what must eventually be true (`◇AllCommitsVisible`), or more complex combinations

Given this spec, TLC constructs the complete reachability graph by:
1. Starting from all states satisfying `Init`
2. At each state, computing all states reachable by applying any enabled action (BFS)
3. At each state, evaluating each safety invariant; at the end of all paths, evaluating liveness
4. On a violation, reporting the shortest counterexample trace — the exact sequence of actions that leads to the bad state

If TLC terminates without a violation, the invariants hold for all possible executions at the abstraction level of the model.

## Mechanics

### Writing a spec

The snippet below is a simplified illustrative fragment, not AWS's actual code. The style shows the key TLA+ constructs:

```tla
VARIABLES leader, committed, replica_acked

Init == leader \in Servers 
     /\ committed = {}
     /\ replica_acked = [n \in Servers |-> {}]

Write(v) ==                              (* leader writes value v *)
    /\ leader \in Servers                (* precondition: we have a leader *)
    /\ v \notin committed                (* only write new values *)
    /\ committed' = committed \union {v} (* state transition *)
    /\ UNCHANGED <<leader, replica_acked>>

Replicate(n, v) ==                       (* server n acknowledges v *)
    /\ v \in committed
    /\ replica_acked' = [replica_acked EXCEPT ![n] = @ \union {v}]
    /\ UNCHANGED <<leader, committed>>

LeaderFail ==
    /\ leader' = NULL
    /\ UNCHANGED <<committed, replica_acked>>

QuorumAcked(v) ==
    Cardinality({n \in Servers : v \in replica_acked[n]}) >= QuorumSize

Safety ==                                (* invariant to check *)
    leader = NULL =>                     (* after any leader failure *)
        \A v \in committed : QuorumAcked(v)  (* all committed values are durable *)

Liveness ==                              (* liveness to check *)
    \A v \in committed : <>(QuorumAcked(v)) (* every write eventually replicates *)
```

TLC explores all orderings: `Write(a)` then `LeaderFail` then `Replicate(b, a)`, or `Write(a)` then `Replicate(a)` then `LeaderFail`, etc. — every combination. `Safety` is evaluated at every state; `Liveness` is evaluated over every path to termination.

### How AWS scales model checking

The state space explodes with larger models, so AWS used "model values" — symbolic constants with no internal structure — to keep counts small. Instead of checking a 100-server DynamoDB fleet, the spec runs with 3 servers and 2 possible keys. Bugs in the logic still appear; the abstraction just can't model load-dependent race conditions.

For DynamoDB's rebalancing algorithm, TLC ran on a cluster checking tens of billions of states over a few hours, finding 7 design bugs. The counterexample traces — sometimes 35+ steps long — were directly usable as documentation of the bug and its fix.

### What bugs AWS found

- **DynamoDB rebalancing (7 bugs)**: The most severe required a 35-step sequence: a specific pattern of leadership transfers, partial writes, and crashes during table rebalancing that could permanently lose committed data. No test caught it because reproducing it required threading a very specific needle across process lifecycles. The TLC counterexample trace was the first time the team understood exactly what sequence of events caused it.

- **S3 authentication race**: A concurrent read and delete, interleaved with an authorization check in a specific order, could allow a request to succeed that should have been rejected. Two-process interleaving that testing didn't cover.

- **EBS Paxos variant (2 bugs)**: AWS used an optimized Paxos variant. TLC found two holes in the optimization that left open a path to split-brain: two nodes could simultaneously believe they held the lease, allowing divergent writes. The spec made the implicit "only one leader can hold the lease at a time" assumption explicit as an invariant, and TLC immediately falsified it in their optimized variant.

### What the engineers reported

Seven Amazon teams adopted TLA+. The consistent feedback: the return on investment was high because specs were written before code, not after, so bugs found by TLC were design bugs — cheap to fix in specification vs. expensive to fix in production. Several engineers said informal proofs they had written by hand were wrong; TLC found the error and showed the counterexample.

## Where it breaks

**State space explosion.** The number of reachable states grows exponentially in the number of processes, values, and network events modeled. A 5-server cluster with 10 message types can reach trillions of states. AWS mitigated this by staying at high abstraction levels and using symbolic constants.

**The spec-code gap.** TLA+ verifies the *design*, not the *implementation*. A correct spec does not guarantee a correct implementation. The code can diverge from the spec over time, and bugs introduced in translation (a real issue as specs are usually in TLA+ and code in Java or C++) are invisible to TLC. AWS treated specs as living documentation, updated them when code changed, but this requires discipline.

**Liveness is hard.** Safety violations (bad states) are easy to check. Liveness properties ("eventually the system makes progress") require explicit *fairness* conditions — assumptions about which actions the scheduler is willing to keep enabling. Getting fairness conditions right is difficult, and weak fairness assumptions can hide real liveness bugs. AWS focused primarily on safety properties.

**High barrier to entry.** TLA+ has unusual syntax (based on set theory and Zermelo-Fraenkel notation), and temporal logic is not intuitive for most software engineers. AWS reported this as the main adoption friction and developed internal training to address it.

## Why it works

The deeper principle is that TLA+ model checking is **BFS on the happens-before DAG of all possible system executions**, with your invariants as the acceptance condition.

Most distributed system bugs are not bugs in any individual component — they are specific orderings of valid events that the designer didn't consider. A Paxos implementation might be correct for every interleaving the developer tested but wrong for a specific ordering they never tried. Model checking makes "every ordering" explicit.

This situates model checking on a spectrum of verification approaches:

| Technique | Coverage of trace space | Reproducible? |
|---|---|---|
| Formal proof | All traces (symbolic) | Yes |
| Model checking (TLA+) | All traces (enumerated) | Yes |
| Deterministic simulation (FoundationDB) | Many traces (seeded random) | Yes |
| Property-based testing (QuickCheck) | Many traces (random) | Partially |
| Unit tests | Few traces (hand-written) | Yes |
| Manual testing | Very few traces | No |

FoundationDB's simulation testing (see the archive entry from 2026-05-19) is one step down from model checking: it uses a seeded PRNG to reproducibly sample from the space of possible interleavings, and can replay any bug. TLA+ model checking covers *all* interleavings but requires working at the design level, not the implementation level. They're complementary: TLA+ verifies the design before you write code; simulation testing verifies the code after you write it.

The insight generalizes beyond distributed systems: any time you have a system with concurrent actors and shared state, the "did I verify all possible interleavings?" question is the right question to ask, and the answer determines how much confidence your verification gives you.

## Going deeper

1. **"Specifying Systems"** by Leslie Lamport (free PDF from Lamport's website): the canonical TLA+ book, with a step-by-step development of the TLA+ language and TLC model checker through distributed algorithm examples including Lamport clocks and Paxos.

2. **"Practical TLA+"** by Hillel Wayne (Apress, 2018): a more accessible introduction focused on real-world specification tasks, with examples in distributed system design and state machine modeling.

3. **"A Formal Specification of the RAFT Consensus Protocol"** (Diego Ongaro's TLA+ spec, on GitHub): a complete TLA+ spec of Raft, checkable with TLC, showing exactly how the spec maps to the algorithm described in the Raft paper — and what invariants Raft guarantees that the spec makes checkable.
