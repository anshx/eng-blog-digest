---
title: "SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol"
source: https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf
author: Abhinandan Das, Indranil Gupta, Ashish Motivala
company: Cornell University
date_posted: 2002
date_digested: 2026-07-11
---

# SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol

## What's new to learn

- **Epidemic failure detection**: Each node probes exactly one random peer per period and asks k others to cross-check ambiguous results, giving O(1) per-node message overhead that stays constant as the cluster grows — unlike heartbeating, which is O(N²).
- **Piggybacking dissemination**: Membership updates (join, leave, fail) ride along on the probe/ack traffic that failure detection already generates, so new information reaches every node in O(log N) rounds at zero added bandwidth.
- **Suspicion subprotocol**: A node is not immediately declared dead; it is first marked "suspected," allowing it to refute with a higher incarnation number. This single mechanism converts transient network glitches into false-positive suspects that cancel themselves rather than permanent false failures.

## Prerequisites

- Why "is this node alive?" is hard in a distributed system (no shared clock, no global memory, messages can be lost or delayed indefinitely).
- What a heartbeat-based failure detector does: each node sends a "I'm alive" message every T seconds to all others; silence for 2T means failure.
- Why O(N²) bandwidth is a problem at scale: 1,000 nodes × 1,000 heartbeats/period = 1,000,000 messages/period; at 10,000 nodes that is 100× worse.
- Basic probability: the idea that k independent Bernoulli trials reduce combined failure probability to p^k.

## The core idea

Traditional failure detectors ask every node to monitor every other node. SWIM instead asks a harder but more useful question: "Can *someone* reach node T right now?" — and answers it by randomly outsourcing the check.

Every T_p seconds, each node M picks one random target T from its membership list and sends a direct `ping`. If T answers, M is done. If T does not answer in T_ping seconds, M does not declare failure. Instead, it picks k additional nodes at random and sends them a `ping-req(T)`: "please probe T on my behalf and forward me the result." If none of those k indirect probes produce an acknowledgement, T moves to "suspected" status. A suspected node can clear its name by broadcasting `alive(T, incarnation+1)`. Only if T never refutes within a configurable timeout does M broadcast `failed(T)` and remove T from the membership list.

This separation — direct probe for the common case, indirect probes for disambiguation, suspicion as a grace period — is the full algorithm. Everything else is the implementation.

## Mechanics

### Protocol period

Each protocol period T_p is roughly 3–5 RTTs in practice (a few hundred milliseconds in a LAN cluster). Within one period, M performs exactly:
- 1 direct `ping` to a random target.
- Up to k `ping-req` messages if the direct ping fails.

Per-node message cost: O(1) per period regardless of N.

### Failure detector component (SWIM-FD)

```
Every T_p seconds, node M does:
  T ← random member from membership_list
  send ping(M→T)
  wait T_ping for ack(T→M)
  if no ack:
    S ← k random members (S ≠ T, S ≠ M)
    for each s in S: send ping-req(M→s, target=T)
    for each s in S: s sends ping(s→T) and forwards ack to M if received
    wait T_indirect for forwarded ack
    if still no ack:
      mark T as suspected
      multicast suspect(T, incarnation_T)
```

When T receives `ping-req` it does the probing itself — M never speaks to T again during this period. The forwarded `ack(T→s→M)` reaches M through two hops so M knows T is reachable by *at least one* independent path.

**Suspicion and refutation:**

When M hears `suspect(T, inc=i)`, it sets T's state to Suspected/i in its membership table. If T is alive and receives this (via gossip), it multicasts `alive(T, inc=i+1)`. The higher incarnation number wins: any node hearing both `suspect(T, 3)` and `alive(T, 4)` keeps the alive state. T is declared failed only if:
1. M has not heard `alive(T, j)` for j > i within T_suspect seconds, **and**
2. At least one other node also suspects T (the paper uses a count threshold to reduce coincidental false suspicion).

### Dissemination component (infection-style)

Every `ping`, `ack`, and `ping-req` message carries a small "gossip footer" — a list of the most recent membership events M has seen, sorted by how many times M has already forwarded them. Specifically, M attaches an event until it has been sent λ·ceil(log₂ N) times, then drops it from the footer.

This is infection-style spreading: the first node that learns an event is "infected." It infects one new node per message it sends (by including the event in the footer). Those nodes infect others. Like an epidemic with R₀ ≈ k·(fanout), the update reaches all N nodes in O(log N) protocol periods in expectation — matching epidemic spread theory exactly.

**Convergence time with numbers:** In the SWIM paper's simulation with N=100, the median dissemination latency was about 5 protocol periods. With N=1,000 it was about 7 periods (log₂ 1,000 ≈ 10). This scales beautifully.

**Priority ordering:** When a node's gossip footer is full, it drops the events that have been forwarded most, keeping the freshest or highest-priority ones (e.g., failures before joins). This ensures recent failures propagate even during heavy churn.

### Message complexity summary

| Mechanism              | Messages/period/node | Total for N nodes  |
|------------------------|----------------------|--------------------|
| Traditional heartbeat  | N                    | O(N²)              |
| SWIM-FD (direct only)  | 1                    | O(N)               |
| SWIM-FD (indirect k=3) | 1 + 3 = 4            | O(N)               |
| Dissemination (piggyback) | 0 extra           | 0 added overhead   |

### False positive rate

With k=3 independent indirect probes and per-path packet-loss probability p, the probability all k paths fail simultaneously despite the target being alive is p^k. At p=0.05 (5% loss): 0.05^3 = 0.000125 — roughly 1 in 8,000 periods, versus 5% for a direct probe alone.

### Detection time bound

The SWIM paper proves that the expected time from a node failing to it being suspected is bounded by 2·T_ping + T_indirect ≈ 1.6–2 protocol periods, independent of N. This is a much stronger guarantee than probabilistic heartbeat timeouts, which have variance proportional to N as heartbeat queues back up.

## Where it breaks

**Weak consistency is real.** Two nodes can simultaneously disagree about whether a third is alive during the dissemination window. Any application that requires strong membership agreement (e.g., primary election without a separate consensus protocol) cannot rely on SWIM alone.

**Churn degrades convergence.** If nodes are joining and leaving faster than the gossip can propagate, the membership list never converges. SWIM assumes the join/leave rate is O(1) per protocol period, not O(N).

**Network partitions produce false failures.** If the cluster splits, nodes on each side will eventually declare the other side failed. This is usually the *correct* response — you can't reach them — but it means SWIM does not distinguish partition from failure.

**Suspicion latency adds to MTTR.** The grace period for refutation (T_suspect) is typically 5–10 protocol periods. For a 500 ms period, detecting a dead node and propagating the result takes 5–8 seconds end-to-end. Raft-style leader elections on top of SWIM need to account for this delay.

**Small clusters waste the indirection.** For N < 20, the protocol overhead of k=3 indirect probes can exceed what simple all-to-all heartbeating would cost. SWIM is designed for N ≥ 50–100.

**The Lifeguard problem.** If node M is itself unhealthy (CPU saturated, GC pausing), its probes time out not because T is dead but because M is too slow to process the response. It then incorrectly suspects T. Hashicorp's Lifeguard extension (2017) addresses this by tracking how often M's own probes are "nacked" and slowing M's detection rate when M itself is under duress.

## Why it works

**The epidemiology isomorphism** is not a metaphor — it is exact. The SWIM dissemination model is the SIR (Susceptible–Infected–Recovered) epidemic model on a complete graph:

- Susceptible: nodes that haven't heard the update.
- Infected: nodes that are actively forwarding it (within their λ·log N transmission budget).
- Recovered: nodes that have hit the transmission limit and stop forwarding.

SIR on a complete graph converges in O(log N) time because each infected node "infects" one new node per contact, and the number of infected nodes doubles each period until saturation. This is textbook epidemic theory — Kermack-McKendrick (1927) — applied to packet-switched networks.

**SWIM-FD is independent probing as redundancy.** The indirect probe mechanism is structurally identical to other redundancy patterns in the archive:
- *The Tail at Scale* (hedged requests): send two requests, accept the first response. Indirect probing sends k requests to k witnesses, accepts the first valid ack.
- *Reed-Solomon erasure codes*: any k-of-N symbols suffice. Any one indirect ack suffices — M does not need all k to succeed.
- *RAID-5 parity*: one disk failure is recoverable. One dropped probe is recoverable because k-1 others continue.

The deeper unifying principle: **distributed systems tolerate failures by replacing single-point checks with independent-witness quorums.** For data: quorum reads/writes. For time: TrueTime's uncertainty interval. For membership: SWIM's indirect probes. The size of the quorum (k) controls the failure-probability budget.

**Piggybacking = amortized broadcast.** SWIM achieves O(log N) dissemination at zero extra bandwidth cost by combining it with probe traffic. This is the same principle as:
- OS page-fault handlers that prefetch adjacent pages into a TLB entry (amortizing TLB miss cost over multiple virtual pages).
- TCP's Nagle algorithm (accumulating small writes into fewer segments).
- ClickHouse's parallel replicas scattering work across existing connections.

Lazy amortization — "since we're already sending a message, attach the thing we need to propagate" — consistently eliminates extra round trips.

**Why heartbeating fails:** Traditional heartbeating has each node *pushing* its state to all others. This requires each node to *know about* all others, creating O(N²) coordination. SWIM *pulls* by asking random witnesses and expects epidemic spreading to fill in the gaps. The asymmetry between O(N²) push and O(N log N) epidemic pull is the same tradeoff between all-pairs communication and random gossip in social network information diffusion.

## Going deeper

1. **Lifeguard: Local Health Awareness for More Accurate Failure Detection** (Hashicorp, SOSP 2017 workshop, [arXiv:1707.00788](https://arxiv.org/abs/1707.00788)) — extends SWIM so that degraded nodes slow their own failure detection, preventing an overloaded node from falsely declaring healthy peers dead.

2. **Hashicorp memberlist** (Go implementation) — [github.com/hashicorp/memberlist](https://github.com/hashicorp/memberlist) — production implementation used by Consul, Nomad, and Serf; the code is readable and the README explains every parameter and its relationship to SWIM's theoretical bounds.

3. **Epidemic Algorithms for Replicated Database Maintenance** (Demers et al., 1987) — the foundational paper that first applied epidemic theory to distributed database synchronization, and the intellectual ancestor of all gossip-based protocols including SWIM.
