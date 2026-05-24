---
title: "The Tail at Scale"
source: https://cacm.acm.org/research/the-tail-at-scale/
author: Jeffrey Dean and Luiz André Barroso
company: Google
date_posted: 2014-02-01
date_digested: 2026-05-24
tags: [distributed-systems, latency, performance, probabilistic]
---

# The Tail at Scale

*Jeffrey Dean and Luiz André Barroso, CACM vol. 57 no. 2, February 2014. Selected for the SIGOPS Hall of Fame Award, April 2025.*

## What's new to learn

- **Fan-out amplification**: In a parallel fan-out to N servers, P(at least one server is slow) grows quickly toward 1 — so a rare per-server tail event becomes near-certain at the system level.
- **Hedged requests**: Send a duplicate request to a second server after waiting for the 95th-percentile latency, take the first response, cancel the other — spending ~5% extra load to cut P999 by 25×.
- **Tied requests**: A tighter variant of hedging where the duplicates are aware of each other and cancel out of each other's queues once one starts processing, exploiting the insight that *queueing delay*, not *service time*, is the main source of tail variance.

## Prerequisites

- Basic probability: independence, the CDF of the max of N i.i.d. random variables.
- Familiarity with distributed RPC fan-out patterns (scatter/gather, map-reduce stages).
- Queuing theory intuition: knowing that wait time in a queue and service time are separable sources of latency.

## The core idea

The paper starts with a deceptively simple observation. Suppose every server in your fleet has a 99th-percentile (P99) latency of 1 second against a median of 10 ms. One request hitting one server: 1-in-100 chance of a one-second response — annoying but tolerable. Now suppose a typical user request fans out to 100 servers and waits for *all* of them:

```
P(at least one slow) = 1 - P(all fast) = 1 - (0.99)^100 ≈ 63%
```

At 1,000 servers this is >99.9%. A fleet that looks healthy per-machine — 1% slow requests — will make nearly every user request slow if the fan-out is wide enough. The P1 per-machine becomes the P99 at the system level.

The paper frames this as *tail tolerance* — analogous to fault tolerance. In fault-tolerant design you add redundancy so that machine *failure* doesn't stop the system. Here you add redundancy so that machine *slowness* doesn't stop the system. The same mental model, different failure mode.

## Mechanics

The paper organizes techniques into two time horizons:

### Within-request adaptation (milliseconds)

**Hedged requests**

1. Issue the request to the primary server, start a timer.
2. When the timer reaches the 95th-percentile expected latency for that request class, issue an identical request to a different server.
3. Use whichever response arrives first; send a cancel to the loser.

Why the delay? If you send two requests immediately, you double load. By waiting until P95, you only trigger the hedge for the slowest 5% of cases, adding approximately 5% extra load in total (the slow 5% pays 2× but the fast 95% pays 1×). In a Google BigTable benchmark reading 1,000 keys fanned across 100 servers, a 10 ms hedge delay reduced P999 latency from 1,800 ms to 74 ms while adding only 2% more requests.

**Tied requests**

Hedging still requires waiting for the delay timer to fire. A faster alternative: send the request to two servers *simultaneously*, but include in each request the address of the other server (they are "tied"). As soon as one server dequeues the request and starts processing, it sends a cancellation to the other. The other server drops the request from its queue.

This short-circuits the queue entirely. The key insight is that once a request *starts* executing it finishes quickly; the variance is almost entirely in how long it waits in the queue. Tied requests showed approximately 37% reduction in tail latency in Google experiments, and in shared-queue environments the latency profile nearly matched an idle cluster.

**Canary requests**

For very large fan-out (thousands of servers), a single malformed request can crash or severely delay every machine simultaneously — an accidental denial-of-service against your own fleet. The fix: send the request to one or two "canary" servers first. Only after they respond successfully does the root server fan out to the rest. Google's web search uses canary requests for all large fan-out operations.

### Cross-request adaptation (seconds to minutes)

These techniques don't help a single request in flight, but improve the statistical landscape for future requests.

**Micro-partitions**

Instead of assigning one large partition per machine, create roughly 20 partitions per machine. Each machine owns a handful; the total number of partitions is much greater than the number of machines.

Benefits:
- *Faster rebalancing*: Moving one micro-partition shifts ~5% of a machine's load, not 100%.
- *Finer-grained shedding*: When a machine gets slow, you can peel off one micro-partition (5% of load) rather than evacuating the whole machine.
- *Speed*: Recovery from a slow machine takes 1/20th the time compared to a single-partition-per-machine design.

**Selective replication**

Some micro-partitions are "hot" — a small set of keys or documents accounts for a disproportionate share of requests (Zipf distribution). For these, create additional replicas on other machines and spread their load across multiple machines. Google Web Search uses this for popular documents: copies appear in multiple micro-partitions, and queries are routed to whichever replica is least loaded.

**Latency-induced probation**

When a server's tail latency rises significantly above fleet average, temporarily remove it from the active pool. Continue sending *shadow* (non-user-affecting) requests to the probationary server to collect latency statistics. When it recovers, reintroduce it. This prevents a struggling machine from dragging down every user request that happens to land on it, while the shadow traffic maintains visibility into its health.

### What causes the per-server tail in the first place?

The paper catalogs the root causes of within-machine latency variance:

- **Global resource contention**: shared CPU, memory, or network buses between co-located services.
- **Daemon interference**: background GC pauses, log rotation, periodic health checks, scheduled jobs.
- **Power management**: CPUs throttling when they hit thermal limits.
- **Queue head-of-line blocking**: a single large request at the front of a FIFO queue delays all smaller requests behind it. Mitigation: maintain separate queues per priority tier; break large requests into smaller units.
- **Garbage collection**: JVM pause-the-world events, even with concurrent collectors, can inject multi-ms spikes.
- **Network bufferbloat**: large switch buffers filling up, inflating RTT for everything behind the large flow.

## Where it breaks

**Idempotency required.** Hedged and tied requests assume that executing the same request twice produces the same result and that a second execution can be cancelled or ignored. Write operations or side-effecting RPCs need extra care — either use idempotency keys or restrict hedging to read paths.

**Amplified load under sustained overload.** In a normally operating system, hedges fire only for the slow 5% and cancel quickly. But if the system is *already overloaded* and most requests are slow, hedging fires for nearly every request and doubles load — exactly when you can least afford it. Some implementations add a circuit breaker that disables hedging when the hedging rate exceeds a threshold.

**Cancellations are best-effort.** A cancel message may arrive after the server has already started processing. You then pay for two completions. Tied requests mitigate this (cancel happens at queue-time, before processing begins) but don't eliminate it.

**Cross-request techniques require observability.** Latency-induced probation and selective replication depend on accurate per-machine latency histograms, which themselves add overhead. If your monitoring has high latency or poor coverage, the adaptation lags behind reality.

**Fan-out must actually be parallel.** The variance-amplification math assumes the N sub-requests run in parallel. If your scatter/gather serializes any portion (e.g., sequential retries), the math doesn't apply directly.

## Why it works

The deeper principle is that **variance in a max-of-N distribution shrinks the moment you change the aggregation rule**.

For N i.i.d. variables with CDF F, `P(max ≤ t) = F(t)^N`. To keep the max below threshold `t` with high probability you need `F(t)` close to 1, which forces `t` far into the tail of the per-server distribution. Conversely, `P(min ≤ t) = 1 - (1 - F(t))^N`, which saturates toward 1 very quickly — the minimum of N samples is almost always near the lower tail.

Hedging changes the aggregation from `max` (wait for all) to `min` (take the fastest). You're not making any single server faster — you're changing *which statistic you extract from the ensemble*. The fan-out is still wide, but instead of being punished by the worst server you're rewarded by the best one. This is precisely the structure of **Power of Two Choices** (covered in this archive for load balancing) but applied to latency instead of queue depth.

The redundancy framing connects to a general principle in reliability engineering: **you can buy tolerance to rare events cheaply when the cost-per-hedge is small and events are independent**. A 5% load premium is the "insurance premium"; the 25× P999 reduction is the "payout". The math is favorable because tail events at one server are (mostly) independent of tail events at another.

This also explains *why* micro-partitions and probation work: they reduce variance by removing the high-variance machines from the distribution before the max is computed. Fine-grained partitions make rebalancing cheap so you can respond to variance faster. Both are variance-*prevention* while hedging is variance-*tolerance* — the paper offers both because no single strategy dominates.

The analogy to fault tolerance is precise: RAID trades a small space overhead for tolerance of disk *failure*; hedged requests trade a small compute overhead for tolerance of disk (or network, or GC) *slowness*. The same redundancy principle, different event type.

## Going deeper

1. **"Power of Two Choices"** — Mitzenmacher, 1996. The probabilistic foundation for why sampling even two options collapses the expected maximum dramatically. Directly underpins why hedging to just one extra server is enough.

2. **"Dynamo: Amazon's Highly Available Key-Value Store"** (SOSP 2007) — Section 4.5 on "Handling temporary failures" describes coordinator-replica request hedging (they call it "sloppy quorum" + hinted handoff) as a production instantiation of within-request tolerance ideas.

3. **"Attack of the Killer Microseconds"** (Barroso et al., CACM 2017) — The follow-up paper focusing on the nanosecond-to-microsecond sources of latency variance that tail-tolerance techniques can't eliminate: NUMA effects, last-level cache misses, power management, and PCIe tail latency. Shows where the 2014 paper's solutions hit their limits.
