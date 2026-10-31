---
title: "Exponential Backoff And Jitter"
source: https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
author: Marc Brooker
company: AWS
date_posted: 2015-03-04
date_digested: 2026-10-31
---

# Exponential Backoff And Jitter

## What's new to learn

1. **Phase alignment in retries**: Plain exponential backoff synchronizes clients — because every client uses the same formula, they all fail at the same moment, wait the same interval, and collide again. The backoff buys time but doesn't break the synchronization.

2. **Full vs. decorrelated jitter**: Randomizing a retry sleep has two distinct flavors with different tradeoffs: "full jitter" minimizes the total number of requests the server sees (kinder to the server, slower to finish); "decorrelated jitter" minimizes total time to complete (faster, but noisier — each sleep is a random walk that can dip near-zero).

3. **Jitter as phase randomization**: The deeper insight the post encodes — though doesn't name explicitly — is that adding randomness converts a *synchronized batch arrival* into a *Poisson-like arrival process*, changing the server's load from a spiky synchronized burst into smooth independent arrivals.

## Prerequisites

- Know what a client retry is and why you need backoff (server overload, transient network errors).
- Understand exponential growth: doubling intervals (1s, 2s, 4s, 8s…) with a cap.
- A rough mental model of what "thundering herd" means: many clients hitting a newly recovered server simultaneously and re-crashing it.
- Comfortable reading pseudocode (the algorithms are trivial to implement).

## The core idea

Suppose 100 clients all fail at the same moment — a shared backend crashed. They each apply `sleep = min(cap, base * 2^attempt)` and retry. Because the formula is deterministic and they started at the same time, they *all* wake up at the same moment. The server sees 100 requests at once, gets overloaded again, fails, and the cycle repeats. Exponential backoff prevents the server from being constantly hammered, but it does nothing to prevent the synchronized burst *when the backoff expires*.

The fix is jitter: replace the deterministic sleep with a random draw from `[0, min(cap, base * 2^attempt)]`. Now clients wake up scattered across that interval. The server sees a gradual trickle instead of a synchronized wave.

The key formula to internalize:

```
# Full jitter — uniformly random over the whole window
sleep = random_between(0, min(cap, base * pow(2, attempt)))
```

This is the simplest version, and it works. The window grows exponentially, so early retries are tightly clustered near zero and later retries are spread across a wider range — which is the right shape for a retry distribution.

## Mechanics

Brooker simulates 100 concurrent clients, each making requests against a server with limited capacity, and measures two quantities: **total calls** (how much work the server has to absorb) and **total time** (how long until all clients succeed). He compares four strategies:

### 1. No jitter (plain exponential backoff)

```
sleep = min(cap, base * 2^attempt)
```

Every client waits identically. The arrival pattern is perfectly synchronized — the server sees spiky waves. High total calls, high total time.

### 2. Full jitter

```
sleep = random_between(0, min(cap, base * 2^attempt))
```

Each client independently draws a uniform random from the full backoff window. The arrival distribution over time approaches uniform across the window. **Fewest total calls** (lowest server load), but slightly longer total time because some clients draw near-cap values.

### 3. Equal jitter

```
temp = min(cap, base * 2^attempt)
sleep = temp/2 + random_between(0, temp/2)
```

Guarantees a minimum wait of `temp/2` while still randomizing the upper half. Slightly more total calls than full jitter but a guaranteed floor on the wait, which some systems need for safety margins (e.g., circuit breaker cooldown). Performance is similar to full jitter.

### 4. Decorrelated jitter

```
sleep = min(cap, random_between(base, prev_sleep * 3))
```

Each attempt's sleep is drawn relative to the *previous* sleep, not the attempt number. This creates a random walk — each client drifts independently through time. **Fastest total time** of all strategies, because lucky draws can get close to `base`. Slightly more total calls than full jitter because some clients retry very quickly.

The simulation verdict: **Full jitter** if you want to minimize server load. **Decorrelated jitter** if you want minimum completion time and the server can handle more total requests. Both are strictly better than no-jitter on every metric.

The cap is important: without a maximum, the backoff interval grows without bound. With a cap, the window eventually tops out and the jitter draws become uniform over the maximum wait, which is fine.

## Where it breaks

**Jitter doesn't help idempotency**: Jitter only spreads the *timing* of retries. If your retried operation isn't idempotent (write-once semantics, non-repeatable side effects), jitter doesn't solve the duplicate-execution problem. You still need idempotency keys or conditional writes.

**Cap sensitivity**: If your cap is too low relative to your recovery time, all clients pile up around the cap value. Jitter helps but doesn't solve what is now a capacity planning problem.

**The fast-movers problem with decorrelated jitter**: Because sleep is bounded below by `base` (not zero), decorrelated jitter doesn't allow zero-wait retries. But since it's a random walk relative to the previous sleep, a client can take a short sleep chain down near `base` repeatedly — mimicking fast retry behavior that isn't actually helpful if the server needs time to recover. Full jitter's per-attempt bound prevents this.

**Cold start**: For the very first retry (attempt=0), all three jitter strategies have windows on the order of `base`, so the first retry still tends to be somewhat synchronized. Only after several doublings does the window spread clients apart meaningfully. The metastable failure case (archive: 2026-05-15) — where a thundering herd is triggered by a very brief spike — can still cascade before jitter kicks in.

**Jitter ≠ circuit breakers**: Jitter reduces server load under transient overload, but if the server is persistently overloaded, jittered retries still consume capacity. The complement to jitter is a circuit breaker that *stops retrying entirely* when the failure rate crosses a threshold.

## Why it works

The deeper principle is **phase decorrelation**: synchronized agents competing for a shared resource are worse than the same number of independent agents, because synchronization means each wave is as bad as all agents combined.

This is the same insight underlying three other well-known mechanisms:

**CSMA/CD (Ethernet, 1970s)**: When two nodes transmit at the same time and detect a collision, each waits a random multiple of the slot time before retrying. Without randomization, both nodes would retry simultaneously and keep colliding forever. The binary exponential backoff with randomization is *exactly* the algorithm Brooker is describing, applied to 1980s Ethernet hardware.

**Raft's election timeout (covered in archive)**: Each node uses a *random* timeout in [T, 2T] before calling an election. If all nodes used the same timeout, they'd all hold elections simultaneously, splitting votes and never making progress. The randomization is the mechanism that ensures one node wins with high probability. This is jitter applied to leader election.

**SWIM gossip failure detection (covered in archive)**: Each node probes a *random* peer per period, rather than going around a fixed ring. Fixed-order probing would cause synchronized network bursts; randomization distributes probe traffic evenly.

The unifying mental model: **randomization converts a globally synchronized process into locally independent ones**. Independent Poisson-like arrivals are how queueing theory's clean results (Little's Law, M/M/1 queue analysis) work — and they give much more predictable service times than synchronized bursts. Jitter is the engineering trick that makes your retry behavior *look like* the nice statistical model rather than the adversarial synchronized one.

There's also a connection to the **Power of Two Choices** (covered in archive): the insight there is that one comparative bit of information — picking the lighter of two random servers — collapses worst-case load dramatically. Jitter is analogous: one bit of randomness (when to retry) collapses the worst-case synchronized collision.

## Going deeper

1. **Marc Brooker's follow-up post, "Jitter"** (https://brooker.co.za/blog/2015/03/21/backoff.html) — the personal blog version of the AWS post with slightly more commentary on the intuition behind the simulation results.

2. **"Making Retries Safe with Idempotency Keys"** — any treatment of this topic (Stripe's API design docs are a canonical reference) pairs well here: jitter handles *when* to retry; idempotency keys handle *whether* the retry is safe.

3. **"Reliability, constant work, and a good cup of coffee"** by Marc Brooker (https://brooker.co.za/blog/2020/08/06/erlang.html) — explores the deeper principle: systems that do *constant work regardless of load* (like acks sent on a schedule rather than triggered by events) are more stable than systems that react to load. Jitter is a step toward this, but constant-work systems take it further.
