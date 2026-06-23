---
title: "Meet Alice. Alice is impatient."
source: https://brooker.co.za/blog/2026/06/19/waiting.html
author: Marc Brooker
company: AWS / Personal Blog
date_posted: 2026-06-19
date_digested: 2026-06-23
---

# Meet Alice. Alice is impatient.

## What's new to learn

1. **The inspection paradox (length-biased sampling)**: when you observe a system at a random moment in time, the interval you land in is statistically biased toward being longer than a typical interval, because longer intervals "fill up more time" and are more likely to contain any given observation.

2. **Size-biased distribution**: if X is a random variable with density f(x), the size-biased version of X has density x·f(x)/E[X]. The expected value of this biased distribution is E[X²]/E[X] = E[X] + Var[X]/E[X] — always ≥ E[X], with equality only when variance is zero.

3. **The service-vs-user metric gap**: a service operator counts events (requests, outages) and computes E[X]; a user, arriving at a random wall-clock moment, experiences the size-biased distribution. The wider the tail, the larger the gap — and "tail-heavy" is exactly what latency and outage distributions look like in production.

## Prerequisites

- Probability: probability density functions, expectation E[X], variance Var[X]
- Renewal theory: basic familiarity with point processes (or willingness to think about "events happening at random times")
- Service SLOs: knowing what P50/P99 mean and why they differ from arithmetic mean

## The core idea

Imagine you operate a service. Your dashboard shows mean latency = 100 ms. Alice, a user, complains her mean wait is 1 second. You both have logs. You're both right.

The discrepancy is not a measurement error. It arises from a fundamental difference in *perspective*:

- **You** count completed requests and compute their average duration. You're sampling the distribution of request lengths, one event per sample. This gives E[X].
- **Alice** observes the service at random wall-clock moments (whenever she triggers a request). At that moment, she is more likely to be in the middle of a long request than a short one, because long requests occupy more of the timeline.

This is the **inspection paradox** (also called the waiting-time paradox or bus paradox). Alice doesn't experience the distribution f(x); she experiences a *time-weighted* version of it.

**Bus paradox intuition**: Suppose buses arrive in two alternating patterns — sometimes 5 minutes apart, sometimes 55 minutes apart, each with probability ½. The average gap is 30 minutes. But a random arrival is in the 5-minute gap with probability 5/60, and in the 55-minute gap with probability 55/60. The weighted average gap length experienced is (5×5 + 55×55)/60 = 50.8 minutes — nearly double the arithmetic mean. The more variance in the distribution, the more the user-experienced mean diverges from the service-measured mean.

## Mechanics

### The size-biased distribution

Let X be a non-negative random variable (e.g., request latency) with pdf f(x) and mean μ = E[X]. When you observe at a random point in time, the probability that you land inside an interval of length x is proportional to x itself (longer intervals are proportionally more likely to contain a random time). So the effective density you experience is:

```
f̃(x) = x · f(x) / μ
```

This is the **size-biased distribution** of X. Its mean is:

```
E[X̃] = ∫ x · f̃(x) dx = ∫ x² · f(x) / μ dx = E[X²] / E[X]
```

Expanding via the identity E[X²] = Var[X] + (E[X])²:

```
E[X̃] = E[X] + Var[X] / E[X]  =  μ + σ²/μ
```

The gap between what users experience and what the service reports is **exactly σ²/μ** — the variance-to-mean ratio (the index of dispersion). For Poisson-style workloads where σ = μ, this doubles the experienced mean. For log-normal latency distributions (fat tails, high σ), the gap can be an order of magnitude.

### Computing the user-experienced mean in practice

Given P50 and P99, you can fit a log-normal distribution (two parameters: log-mean μ_ln and log-stddev σ_ln), then compute:

```
E[X] = exp(μ_ln + σ_ln²/2)          [service-measured mean]
E[X²] = exp(2μ_ln + 2σ_ln²)
E[X̃] = E[X²] / E[X] = exp(μ_ln + 3σ_ln²/2)   [user-experienced mean]
```

The amplification factor is exp(σ_ln²) — exponential in the log-variance. A service with P50=100 ms and P99=2 s has σ_ln ≈ 1.5, giving an amplification of exp(1.5²) ≈ 9.5×. Users experience roughly 950 ms mean even though the service records 100 ms mean.

### Residual lifetime: how long until the next event completes

From renewal theory, if you arrive at a random moment inside an in-progress interval, you care about how much time remains — the **residual lifetime** R(t):

```
E[R] = E[X²] / (2 · E[X])   as t → ∞
```

The **age** A(t) (time already elapsed since the interval started) has the same distribution as R(t) in steady state. Together:

```
E[A + R] = E[X²] / E[X]
```

This is exactly the expected length of the interval you find yourself in — consistent with the size-biased formula above. The residual expected wait *alone* is half of that.

### Practical example with numbers

| Latency distribution       | μ      | σ/μ (CV) | User-experienced mean |
|----------------------------|--------|----------|-----------------------|
| Deterministic (σ=0)        | 100 ms | 0        | 100 ms (no gap)       |
| Exponential (σ=μ)          | 100 ms | 1        | 200 ms (2×)           |
| Log-normal P99=500 ms      | 100 ms | ~2       | 500 ms (5×)           |
| Log-normal P99=2 s         | 100 ms | ~4.5     | ~2 s (20×)            |

The rightmost column is what Alice measures. The leftmost is what your dashboard shows.

### The same logic applies to outages

Replace "request latency" with "outage duration". An on-call engineer observing outages over time will count N outages and compute a mean. But a user who experiences a random business day has a higher chance of being on a day with a long outage than a short one. Reported MTTR and user-experienced downtime diverge by exactly the same σ²/μ term. This is why availability SLAs need to be stated in terms of aggregate unavailable minutes per user, not just total outage count.

## Where it breaks

- **Non-stationarity**: the derivation assumes the process is in steady state. At startup, right after a deployment, or under a traffic spike, the residual-lifetime distribution hasn't stabilized. The gap can be larger *or* smaller than the steady-state formula predicts.
- **Correlated latency**: the math assumes independent request durations. In practice, a cache miss or GC pause can affect many consecutive requests, making the effective variance larger than per-request variance alone.
- **User arrival patterns**: the model assumes users arrive uniformly in time (Poisson arrivals). If users batch at specific times (e.g., all at the 9 AM business-day open), the inspection paradox still applies but the weight function differs.
- **Capped or truncated distributions**: if your timeout is at 10 s, you never observe X > 10 s, artificially reducing your measured σ. The user's X̃ is still biased by the true tail before timeout; you just call those requests "errors".

## Why it works

The inspection paradox is a direct consequence of **size-biased sampling**: events with larger "size" (duration, weight, count) are proportionally more likely to be sampled under a uniform random draw from the measurement axis they inhabit.

The same principle explains:

- **Friendship paradox** (Feld, 1991): on any social network, your friends have more friends than you on average. Sampling a random person's neighbors is sampling by degree — high-degree nodes appear more often in friend lists. E[friend's degree] = E[degree²]/E[degree] ≥ E[degree].
- **Why classes feel crowded**: a random student encounters more peers in a full class than an empty one. Mean class size experienced by students > mean class size reported by the registrar.
- **Survivorship bias in reliability**: a system still running when you inspect it is more likely to be a system with a long lifetime. This inflates reliability estimates unless corrected.

In all cases, the structure is identical: a uniform random sample from a *continuous* space (time, social connections, students) is automatically a *size-biased* sample from the *discrete* population of events.

The **fundamental inequality** is just Jensen's applied to the convex function x²:

```
E[X²] ≥ (E[X])²   →   E[X²]/E[X] ≥ E[X]
```

with equality iff Var[X] = 0 (no randomness). The more variance, the more the operator's and the user's views diverge.

**Design implication**: if you want to close the gap between what you measure and what users experience, you must reduce variance, not just average. A service that completes 99% of requests in 80 ms and 1% in 10 s is *not* equivalent to one that completes 100% in 100 ms — even though the arithmetic means are the same, the user-experienced mean is wildly different. This is the rigorous mathematical reason why reducing tail latency matters far more than optimizing the median.

## Going deeper

1. **Marc Brooker's earlier post on utilization** — "[Latency Sneaks Up On You](https://brooker.co.za/blog/2021/08/05/utilization.html)" explores the related phenomenon that queue depth grows super-linearly as utilization approaches 1 (the M/M/1 queue), which compounds the inspection paradox at high load.

2. **Karl Sigman's Columbia lecture notes on renewal theory** (Columbia IEOR 4106) — formalizes the residual lifetime, excess lifetime, and age distributions, and proves E[R] = E[X²]/(2E[X]) rigorously via the renewal reward theorem.

3. **Allen Downey's "Inspection Paradox" Jupyter notebook** — `allendowney.github.io/InspectionParadox/` — a hands-on simulation of the bus paradox and friendship paradox with Python, useful for building intuition before internalizing the math.
