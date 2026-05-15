---
title: "Good Performance for Bad Days"
source: https://brooker.co.za/blog/2025/05/20/icpe.html
author: Marc Brooker
company: Amazon Web Services
date_posted: 2025-05-20
date_digested: 2026-05-15
---

# Good Performance for Bad Days

## What's new to learn

1. **Metastability as a distinct failure class.** A metastable failure is not a crash — it is a self-sustaining congestive collapse: the system degrades in response to a transient stressor, then *stays* degraded long after that stressor disappears. The sustaining mechanism (usually a positive feedback loop through retries) is more important to fix than the trigger, because many different triggers lead to the same bad state.

2. **The benchmark gap: happy-path measurement misses the failure regime.** Almost all systems-paper benchmarks show throughput and latency at loads well below saturation. This is the wrong regime for availability engineering. The behavior that determines uptime is what happens *at* and *beyond* the saturation point, not what happens on a quiet Tuesday afternoon.

3. **Ironically, performance optimizations worsen metastability.** Caches, fast paths, and bulk retries make the normal operating state efficient. They also make the *degraded* state disproportionately expensive (cold cache is slower than warm, retry traffic amplifies over baseline, etc.), deepening the gap between the stable-good and stable-bad attractors.

## Prerequisites

- **Little's Law** (L = λW): average queue length equals arrival rate × average wait time. Understand that this is a theorem, not an approximation — it governs every queuing system.
- **Basic queueing theory**: know what "saturation point" means (arrival rate approaches service rate), and why latency grows nonlinearly near it (Erlang C formula).
- **Retry semantics**: understand what exponential backoff does and does not do — it spaces out individual retries but does not prevent synchronized retry bursts when many clients backed off simultaneously.

## The core idea

Imagine a service that handles 10,000 RPS without issue. A brief network hiccup causes a 5-second spike of elevated latency. Clients with aggressive timeouts start seeing failures and retry. The service, now handling *both* the baseline load and the retry load, gets slower. More clients time out. More retries pile in. The hiccup clears — but the service never recovers, because the retry storm alone is enough to keep it overloaded. An hour later, an on-call engineer manually sheds load and restarts components. The system comes back up instantly and handles the load easily.

This is a **metastable failure**: a transiently triggered, self-sustaining collapse. Three properties define it:

1. **Trigger**: any transient stressor — a hardware hiccup, a slow GC pause, a cold cache after a restart, a thundering herd after a brief outage.
2. **Sustaining effect**: a positive feedback loop that keeps the system degraded. Retries are the textbook example but not the only one: garbage collection pauses, hot-spot lock contention, and cache eviction cascades all work the same way.
3. **Persistence**: the trigger is gone; the bad state persists. Load the system would have handled easily in its normal state now exceeds what it can process, because the sustaining effect itself is generating load.

Marc Brooker's broader argument in the ICPE keynote is that this failure mode is largely *invisible* to the performance evaluation community, because benchmarks almost universally measure behavior in the linear regime (loads well below saturation). You publish a beautiful throughput-vs-latency curve with an elbow at 8,000 RPS. What you don't show is what happens at 8,500, or what happens if you push to 10,000 for 30 seconds and then drop back to 6,000. If the system gets stuck in a metastable state at 6,000 RPS — a load it "should" handle — that failure won't appear in your paper.

## Mechanics

### The positive feedback loop, concretely

Walk through the state machine with Little's Law.

Suppose a server processes requests at mean rate μ. Under normal load λ < μ, the queue stays short, latency stays low, and clients see successes. Now suppose a transient stressor pushes λ_eff momentarily above μ — queue grows, latency rises.

Clients have timeouts. When a request exceeds its timeout, the client **retries**. Each retry is a new request arriving at the server. Define the **retry multiplier** r(ρ) as the fraction of failed requests that generate a retry, where ρ = λ_eff / μ is the utilization. As ρ → 1, latency diverges (M/M/1 queue), timeouts spike, and r climbs toward 1.

The effective arrival rate becomes:

```
λ_eff = λ_original + λ_original × r(ρ)
```

If r(ρ) is large enough, this is a **fixed point above μ** — the retry traffic alone exceeds what the server can drain even if the original trigger is gone. The system is stuck.

This is the same structure as bistability in a dynamical system: two stable equilibria (healthy and collapsed), separated by an unstable tipping point. Once past the tipping point, the system falls toward the bad attractor. Removing the trigger doesn't help because the bad state is self-sustaining.

### Retries aren't the only sustaining effect

The HotOS 2021 paper that first formalized metastability (Bronson et al.) catalogued several sustaining effects seen in real outages:

- **Retry storms**: described above.
- **Cache eviction cascades**: a brief spike evacuates a large cache; subsequent requests all miss; cold-path latency drives more timeouts and retries; the cache never refills because traffic volume drops throughput below the threshold needed for cache warming.
- **GC pauses**: a full-heap GC pause causes all in-flight requests to see elevated latency, triggering timeouts, retries, and retry-induced load increases that prolong subsequent GC pressure.
- **Hot-spot lock contention**: a single locked resource becomes a bottleneck; clients queue behind it; timeouts generate retries that add to the queue; the lock is held for longer because the holder is also overloaded.

What these share: the *sustaining mechanism* is the thing to fix. "Metastable Failures in the Wild" (OSDI 2022) studied 22 real production outages and found that while triggers varied widely (hardware faults, config pushes, traffic spikes), every outage could be explained by the same two-component structure: trigger + sustaining effect. Fix the sustaining effect and the system recovers from the trigger on its own. Fix only the trigger and a slightly different one will find the same sustaining effect later.

### Why optimizations make it worse

This is the counterintuitive part. Suppose you add aggressive caching to reduce p99 latency. Normal-state performance improves. But now:

- When the cache is cold (after a restart, after an eviction storm), latency is *much* higher than it was before caching existed, because the code paths that go around the cache have atrophied.
- The ratio of degraded-state cost to normal-state cost has increased, making the gap between attractors wider.

Same with batching: batch writes reduce per-request storage cost in the normal state. But a full write queue in the degraded state means larger backlogs, slower draining, and higher amplification.

From a control-theory perspective: you've added a deeper potential well around the good attractor (optimization) at the cost of deepening the bad attractor too (degradation). The saddle point between them may not move much, but if you do fall in, it's harder to climb out.

## Where it breaks

**Not all failures are metastable.** A cascading failure where a dependency simply goes down and stays down is not metastable — it's a straightforward dependency failure. Metastability specifically requires a self-sustaining mechanism. The distinction matters for diagnosis: if removing load doesn't help, look for a sustaining effect.

**Mitigation adds complexity.** Circuit breakers, adaptive retry budgets, load shedding — all require careful tuning. Overly aggressive circuit breaking can cause "phantom failures" where a system sheds load it could have handled. The interaction between multiple mitigation layers (backoff + circuit breaker + load shedding) is hard to reason about.

**Metastability can hide behind normal postmortems.** The triggering event is easy to find in logs; the sustaining effect often isn't. A team that fixes "the network blip that caused the outage" without understanding the retry storm may do the same postmortem six months later for a different trigger.

**Benchmark-driven development doesn't expose this.** The paper's core claim is also its limitation: the industry lacks standard methodology and tooling for measuring overload behavior. The HotOS 2025 follow-up (Isaacs & Alvaro) proposes a multi-level modeling pipeline — queueing-theory CTMCs → discrete-event simulation → service emulation → real stress tests — but none of these are mainstream practice yet.

## Why it works

The deeper insight is that **distributed systems are nonlinear dynamical systems with multiple attractors**.

In a linear system, doubling the input doubles the output. In a distributed system near saturation, a 10% traffic increase can cause a 10× latency increase (the nonlinearity of queuing near ρ = 1). Retries turn this nonlinearity into a positive feedback loop. And a system with a positive feedback loop is, by definition, capable of having more than one stable equilibrium.

Control theory has understood bistability for decades. The engineering analog is a flip-flop: apply a brief pulse and the system snaps to a new stable state, then stays there after the pulse ends. A metastable distributed system is a flip-flop where the bad state is "all clients timing out and retrying" and the good state is "all clients succeeding."

The connection to **Erlang's work on telephony** (1910s) is exact: Erlang observed that telephone exchanges under overload didn't just slow down — they collapsed completely and refused to recover until operators manually cleared calls. Retries in packet-switched networks are the modern equivalent of a caller repeatedly redialing a busy number.

Jitter works precisely because it *de-correlates* the feedback loop: if clients retry at random offsets, the retry bursts don't sum coherently, and the effective retry arrival rate is smoothed out. Jitter converts a coherent (and potentially sustaining) retry wave into an incoherent one that the server can drain.

## Going deeper

1. **"Metastable Failures in Distributed Systems"** (Bronson et al., HotOS 2021) — the paper that first formally defined the failure class, with concrete examples from production. Short and essential: https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf

2. **"Metastable Failures in the Wild"** (Huang et al., OSDI 2022) — empirical analysis of 22 real outages with the trigger/sustaining-effect taxonomy applied. The table of real incidents is worth reading slowly: https://www.usenix.org/system/files/osdi22-huang-lexiang.pdf

3. **"Analyzing Metastable Failures"** (Isaacs & Alvaro, HotOS 2025) — the follow-on that proposes a practical multi-level modeling pipeline (CTMC → DES → emulation) to detect metastability before production: https://dl.acm.org/doi/10.1145/3713082.3730380
