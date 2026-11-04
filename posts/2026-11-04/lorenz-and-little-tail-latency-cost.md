---
title: "Lorenz and Little: How Much Does Your Tail Cost?"
source: https://brooker.co.za/blog/2026/07/29/lorenz-and-little.html
author: Marc Brooker
company: AWS
date_posted: 2026-07-29
date_digested: 2026-11-04
---

# Lorenz and Little: How Much Does Your Tail Cost?

## What's new to learn

1. **The latency Lorenz curve** — a way to visualize not just where latency *percentiles* fall, but how much each latency cohort contributes to the *total latency mass* of your service, borrowing directly from the Lorenz inequality measurement tool in economics.

2. **Little's Law as a capacity bridge** — the queuing-theory identity N = λW (average concurrency = arrival rate × average response time) means that the fraction of total latency attributable to slow requests is exactly the fraction of system concurrency those requests consume — and therefore their share of capacity cost.

3. **The tail-dominance effect** — in any heavy-tailed distribution (which real service latencies almost always are), a tiny fraction of slow requests can consume the majority of system concurrency, making tail latency not just a p99 user-experience problem but the primary driver of infrastructure spend.

## Prerequisites

- **Little's Law** (N = λW): a fundamental queuing identity relating average number of in-flight requests to arrival rate and mean latency. No stability assumptions beyond steady state.
- **CDF / latency percentiles**: what p50, p99, p99.9 mean in a latency histogram.
- **System concurrency**: the number of requests simultaneously in flight in a service at any instant.

The post does not require economics background; Brooker explains the Lorenz curve from scratch.

## The core idea

Your p50 latency and your p99 latency are both familiar. What you almost never see is: *what fraction of your total concurrency is the p99 tail consuming right now?*

Little's Law gives you this for free. Because average concurrency N = λ × E[T], the average number of in-flight requests at any moment is proportional to the **sum** of all request latencies, not just their average. That means:

> **The fraction of total latency mass contributed by a subset of requests equals the fraction of total concurrency consumed by that subset.**

Concurrency ∝ capacity cost (more in-flight requests → more threads, connections, memory, CPU). So "fraction of latency mass" and "fraction of capacity cost" are the same number.

The Lorenz curve is the tool that makes "fraction of latency mass" something you can see and measure. In economics it plots "cumulative fraction of population (sorted by income) → cumulative fraction of total income earned." Applied to latency it becomes:

> **Sort all requests by their latency (ascending). For each percentile p, plot the cumulative fraction of total latency contributed by the slowest fraction.**

The resulting curve runs from (0, 0) to (1, 1). The key number is the **area above the curve and to the right** — that's the latency mass owned by the tail.

## Mechanics

**Step 1 — Build the latency Lorenz curve.**

Take a sample of N request latencies {t₁, t₂, …, t_N}, sorted ascending.

```
Cumulative latency fraction at rank k:
  L(k/N) = (t₁ + t₂ + … + t_k) / (t₁ + t₂ + … + t_N)
```

This is the Lorenz curve. Plot (k/N, L(k/N)) for k = 1…N.

**Step 2 — Read the tail burden.**

The y-value at x = 0.99 tells you the fraction of total latency contributed by the *fastest* 99% of requests. The complement (1 − L(0.99)) is the fraction consumed by the *slowest* 1% (your p99 tail).

**Step 3 — Apply Little's Law.**

Because N = λ · E[T] and E[T] = (Σ tᵢ) / N:

```
Concurrency consumed by p99+ tail  =  (1 − L(0.99)) × total_concurrency
```

**Worked example — why the tail can dominate.**

Suppose your service has:
- Median latency: 2 ms
- p99 latency: 200 ms
- 1 request in 100 hits the p99 tail

For every 100 requests:
- 99 × 2 ms = 198 ms of latency mass
- 1 × 200 ms = 200 ms of latency mass
- Total = 398 ms, of which 50.3% comes from the 1% tail.

By Little's Law, that 1% of requests holds **50.3%** of your server concurrency at any moment — half your infrastructure, for 1% of your traffic. With a p99 at 100× the median, the tail "costs" the same as the other 99 requests combined.

**The Gini coefficient** summarizes the whole curve in one number, exactly as it does for income inequality:

```
Gini = 1 − 2 × area_under_Lorenz_curve
```

Gini = 0 means all requests take identical time (perfect equality, the Lorenz curve is the diagonal). Gini → 1 means one request accounts for essentially all the latency mass. For typical web services Gini in the 0.4–0.7 range is common, implying substantial tail dominance.

**Comparison with the CDF / latency histogram.**

| Tool | What it shows |
|---|---|
| CDF | Where individual request latency falls |
| Latency histogram | Distribution of request latencies |
| Lorenz curve | *Burden* distribution: how much latency mass each cohort contributes |

The Lorenz curve is not just a rescaled CDF. It reweights each percentile by its latency value, making slow requests visually heavy. A 2× latency increase at the p99 tail shifts the curve more than a 2× increase at p50.

## Where it breaks

**Async / event-loop services.** Little's Law counts *in-flight* requests (requests whose timer is running), which includes idle async waits. If your p99 is high because a request is waiting in a database queue, the wall-clock latency is large but the CPU/memory cost on your service may be near zero. In async systems, Lorenz-curve burden overstates cost unless you measure *active* (CPU-consuming) time, not elapsed time.

**Bounded queues and backpressure.** If your service drops or throttles requests under load, the latency distribution is truncated by the system's own behavior. The curve then understates what costs would be under free-run conditions.

**Short measurement windows.** Lorenz curves require a representative sample of the latency distribution. A 1-minute window during a traffic spike may not capture the actual tail behavior.

**Cost ≠ concurrency in some billing models.** For serverless platforms billed by GB-seconds, concurrency is cost. For fixed-size server fleets, concurrency affects queueing depth and therefore p99 of *other* requests, not the cloud bill directly.

## Why it works

The bridge between inequality measurement and capacity planning rests on a single algebraic fact: **Little's Law is linear in latency**.

N = λW means average concurrency is a linear function of average latency. That linearity means you can decompose it:

```
N_total = N_fast + N_tail
         = λ × E[T_fast] + λ × E[T_tail]
```

Fraction of concurrency from tail = E[T_tail] / E[T_total] = tail latency mass / total latency mass.

This is *exactly* the definition of the Lorenz curve integral — which is why the tool transfers perfectly.

The deeper principle is: **any quantity whose total is a linear aggregate of per-unit values can be inequality-analyzed with a Lorenz curve**. This same analysis applies to:

- **Memory fragmentation**: what fraction of total heap is consumed by the largest 1% of allocations?
- **Query cost**: what fraction of total DB CPU is consumed by the slowest 1% of queries?
- **Batch job scheduling**: what fraction of cluster-hours goes to the longest-running 1% of jobs?
- **Network traffic**: what fraction of total bytes is sent by the top 1% of users?

In each case, Lorenz curve → Gini coefficient is the right diagnostic, and "move the burden distribution left" (reduce the Gini coefficient) is the right optimization target.

**The latency-inequality framing also reframes optimization targets.** Most services optimize for reducing *average* latency or a specific percentile. But the Lorenz/Little decomposition says: the highest-ROI optimization is the one that moves the most latency mass, not necessarily the one that moves the most percentile. A 2× improvement in p99.9 may move more latency mass than a 10% improvement in median — and therefore deliver a bigger reduction in concurrency and cost.

The Lorenz curve turns "which part of the tail to attack" from an intuition question into a measurement question.

## Going deeper

1. **Little's Law — Marc Brooker (2018)**: Brooker's earlier post walking through Little's Law proofs and common misapplications — the prerequisite to this one. https://brooker.co.za/blog/2018/06/20/littles-law.html

2. **The Tail at Scale — Jeff Dean & Luiz André Barroso (2013)**: The canonical paper on tail latency as a fan-out problem in large distributed systems; the statistical argument for why hedged requests and micro-partitions are necessary. https://dl.acm.org/doi/10.1145/2408776.2408794

3. **The Lorenz Curve (original) — Lorenz (1905)**: The original economics paper; surprisingly readable. The same 1-page derivation applies directly to latency burden. Any econometrics textbook covers the Gini coefficient derivation from the Lorenz curve integral.
