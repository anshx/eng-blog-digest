---
title: "Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases"
source: https://link.springer.com/chapter/10.1007/978-3-319-14472-6_2
author: Sandeep S. Kulkarni, Murat Demirbas, Deepak Madeppa, Bharadwaj Avva, Marcelo Leone
company: Michigan State University / University at Buffalo
date_posted: 2014-12-01
date_digested: 2026-08-26
---

# Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases

## What's new to learn

1. **Hybrid Logical Clock (HLC):** A 64-bit timestamp `(l, c)` where `l` is a physical-time high watermark and `c` is a logical counter; `l` advances with physical time (stays within NTP skew ε) while `c` tracks causality within the same millisecond — combining Lamport clock ordering with bounded drift from wall time.

2. **Bounded drift invariant:** Unlike a Lamport clock (which can grow to an astronomically large value disconnected from real time), an HLC `l` is always ≤ max physical clock in the system + ε — making HLC timestamps usable as MVCC version numbers that also encode approximate real time.

3. **Consistent snapshots without atomic clocks:** Because HLC timestamps are close to physical time and causality-consistent, you can take a snapshot at "time T" by waiting for a node's HLC to exceed T + ε, without GPS hardware or the 7 ms Spanner commit-wait.

## Prerequisites

- **Lamport clocks and happens-before** (see the August 24 post in this archive — HLC is a direct extension).
- **NTP clock synchronization:** Network Time Protocol keeps clocks within a bounded skew ε ≈ 100–500 ms across data center nodes.
- **MVCC:** Multi-Version Concurrency Control uses timestamps as version numbers; a read at snapshot time T sees all writes committed before T.
- **Partial synchrony:** The distributed systems assumption that messages arrive within bounded (but unknown) delay; contrasted with asynchrony (no delay bound) and synchrony (known delay).

## The core idea

Lamport's logical clock, covered in the previous post, gives you causality: if event `e` happened-before `f`, then `lc(e) < lc(f)`. But the clock values are purely ordinal — a Lamport timestamp of 1,000,000 tells you nothing about when the event happened in real time. To read a "consistent snapshot at noon UTC" you'd need to somehow map noon to a Lamport value, which requires coordination.

TrueTime (Google Spanner) solves this the expensive way: GPS antennas and atomic clocks keep physical clocks within ε ≈ 7 ms. A write commits with a wall-clock timestamp, and readers wait 7 ms for that timestamp to be safely in the past (commit-wait). Atomically accurate hardware is not available in most cloud environments.

HLC threads the needle. It keeps a timestamp that:
- Tracks `happened-before` exactly like a Lamport clock, so `e → f ⟹ hlc(e) < hlc(f)`.
- Stays within ε of the maximum physical clock seen anywhere in the system, so you can relate HLC timestamps to real time without special hardware.
- Fits in a standard 64-bit integer.

The trick is two components: `(l, c)`. The `l` component ("logical-physical") plays the role of physical time: it is always at least as large as the local physical clock, but advances with received messages just like a Lamport clock. The `c` component ("counter") is a tie-breaker that counts causally ordered events that share the same `l` value. Whenever physical time advances past `l`, `c` resets to zero — because a new millisecond has begun and there's no backlog of causally ordered events to sequence.

## Mechanics

### The state

Every process `j` maintains two integers:
```
l.j   — physical-time high watermark (initialized to 0)
c.j   — logical tie-breaker counter (initialized to 0)
```

Comparison of HLC timestamps is lexicographic: `(l₁, c₁) < (l₂, c₂)` iff `l₁ < l₂`, or (`l₁ = l₂` and `c₁ < c₂`).

### Local event or send

```python
def send_or_local(j):
    l_old = l.j
    l.j   = max(l_old, pt.j)      # never go below physical time
    if l.j == l_old:
        c.j = c.j + 1             # physical time didn't advance; use counter
    else:
        c.j = 0                   # physical time jumped forward; reset counter
    return (l.j, c.j)             # stamp the event / attach to outgoing message
```

### Receive

```python
def receive(j, msg_timestamp):
    l_m, c_m = msg_timestamp
    l_old = l.j
    l.j   = max(l_old, l_m, pt.j)         # absorb the sender's high watermark
    if l.j == l_old == l_m:
        c.j = max(c.j, c_m) + 1           # all three tied; take the larger counter
    elif l.j == l_old:
        c.j = c.j + 1                     # local was highest; bump local counter
    elif l.j == l_m:
        c.j = c_m + 1                     # message was highest; inherit sender's counter
    else:
        c.j = 0                           # physical time beat both; fresh millisecond
    return (l.j, c.j)
```

### Packing into 64 bits

In practice: use the upper 48 bits for `l` (milliseconds since epoch — enough for ~8,000 years) and the lower 16 bits for `c`. CockroachDB uses nanoseconds and a smaller counter field. The key insight is that `c` only grows when physical time is *not* advancing — i.e., when multiple causally ordered events happen within the same clock granularity — so it rarely exceeds a few hundred.

### The bounded-drift proof sketch

By induction:
- Initially `l.j = 0 ≤ pt.j` for all `j`.
- On a local event: `l.j = max(l_old, pt.j) ≥ pt.j`, so `l.j ≥ pt.j`. The only way `l.j > pt.j` is if `l_old > pt.j`, which by the inductive hypothesis means `l_old` was received from some message whose sender's `l` was at most `pt.sender + ε`. Since all physical clocks are within ε of each other (NTP bound), `l.j ≤ pt.j + ε`.
- On receive: `l.j = max(l_old, l_m, pt.j)`. By induction, both `l_old ≤ pt.j + ε` and `l_m ≤ pt.sender + ε ≤ pt.j + 2ε`. More precisely, the paper proves `l.e ≤ max_physical_time + ε` where `max_physical_time` is the maximum physical clock reading across all processes at any time.

This means the `l` component of any HLC timestamp is within ε of the physical time at which the event occurred. Contrast with a Lamport clock, where the value grows monotonically with every event and is bounded only by the number of events, not by real time.

### How CockroachDB uses it

CockroachDB assigns each MVCC write the node's current HLC value as its version timestamp. When a transaction commits, it advances the local HLC. Reads at snapshot time T scan for versions ≤ T.

The subtlety: if node A commits at HLC=(1000ms, 0) and then sends a causally-after message to node B, node B's HLC will absorb l=1000ms and advance to at least (1000ms, 1). Any subsequent write on B gets a timestamp > (1000ms, 0). So causality is preserved without global coordination.

For reading a consistent snapshot at T, CockroachDB waits until the "closed timestamp" signal propagates — nodes periodically broadcast "I will not accept new writes at HLC < T_closed." A reader that sees T_closed ≥ T from all relevant nodes can safely read at T without further waiting.

## Where it breaks

**Counter overflow.** If a node receives thousands of causally ordered messages within a single millisecond, `c` can overflow its 16-bit field. CockroachDB handles this by widening the message's HLC or by rate-limiting (the paper acknowledges this as a degenerate case).

**Clock going backwards.** NTP can step the physical clock backward (not just slew it). The HLC algorithm does not handle backward steps: if `pt.j` suddenly decreases, the `max(l_old, pt.j)` still picks `l_old`, so the invariant `l.j ≥ pt.j` holds — but if the clock skew suddenly reverses by more than ε, the bounded-drift guarantee is violated until clocks re-converge. In practice, production deployments use `chrony` or similar tools configured to never step backward by more than a safe threshold (CockroachDB refuses to start if NTP offset exceeds a configured maximum).

**Snapshot reads with NTP skew.** To safely read at timestamp T, you need to wait ε (the maximum NTP skew) before declaring T "in the past." With NTP ε ≈ 500 ms this is painful. CockroachDB's closed-timestamp mechanism avoids this wait for most reads by letting nodes explicitly signal their "frozen frontier." YugabyteDB offers configurable max skew (default 500 ms).

**Doesn't compress causality across partitions.** HLC only observes causality through message passing. Two events on partitioned nodes that never communicate look concurrent to HLC even if one physically happened first — just like Lamport clocks. This is unavoidable without global coordination.

**Not a substitute for external consistency.** HLC gives *causal* consistency, not full *external* consistency (linearizability). Two independent transactions on different nodes that don't communicate can appear in any order. Spanner's TrueTime achieves external consistency by paying the 7 ms commit-wait; HLC cannot guarantee this without special hardware.

## Why it works

The deeper principle is **lazy physical-time embedding**: piggyback the physical time high watermark onto the existing Lamport clock propagation, and use a counter only for the residual within-granularity ordering.

Lamport's insight was: any message passing system already propagates ordering information; we just need to attach a monotone counter to that propagation. HLC's additional insight: physical time *also* propagates through messages (if I sent you a message at 2 PM, you now know "at least some node saw 2 PM"). So we can extract a physical-time high watermark for free from the same message-passing infrastructure, and bolt the logical counter on top only for the tie-breaking cases where physical time hasn't advanced.

This is an instance of the general pattern **"use the existing coordination channel as a secondary piggyback for a weaker guarantee"**:
- TCP carries sequence numbers for free alongside the payload.
- Git commit SHAs carry timestamp hints alongside the DAG structure.
- Raft log entries carry term+index alongside the command payload.
- HLC carries physical time alongside the Lamport clock propagation.

The bounded-drift property is what makes HLC practical over pure Lamport clocks. It means you can answer the question "does this event precede physical time T?" with a bounded error — and that question is exactly what MVCC snapshot reads need.

A subtler beauty: the counter `c` resets to 0 every time physical time advances. This means `c` grows only when the system is producing many causally ordered events *faster than the clock granularity* — i.e., under load. In the common case of distributed systems where inter-node messages take tens of milliseconds, `c` will be 0 or 1 almost always, and the HLC timestamp is indistinguishable from a plain physical timestamp.

## Going deeper

1. **The original paper:** Kulkarni, Demirbas et al., "Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases," OPODIS 2014 — https://link.springer.com/chapter/10.1007/978-3-319-14472-6_2. The full proof of the bounded-drift theorem and the consistent-snapshot algorithm are here.

2. **CockroachDB's transaction model:** The CockroachDB design docs explain how HLC timestamps integrate with Raft log indices and the "closed timestamp" mechanism for zero-wait snapshot reads. Start with the RFC at github.com/cockroachdb/cockroach and search for "HLC."

3. **"Spanner: Google's Globally Distributed Database" (OSDI 2012)** — the TrueTime counterpoint: how GPS + atomic clocks achieve external consistency at 7 ms commit-wait. Reading this alongside the HLC paper makes the tradeoff space concrete: ε ≈ 7 ms + hardware cost (TrueTime) vs. ε ≈ 500 ms + commodity NTP (HLC). Most systems without planetary-scale latency requirements choose HLC.
