---
title: "SFQ: Simple, Stateless, Stochastic Fairness"
source: https://brooker.co.za/blog/2026/02/25/sfq.html
author: Marc Brooker
company: AWS
date_posted: 2026-02-25
date_digested: 2026-05-16
---

# SFQ: Simple, Stateless, Stochastic Fairness

## What's new to learn

- **Stochastic Fair Queuing (SFQ)**: Achieving noisy-neighbor isolation with a *fixed* number of queues (O(1) state) by hashing tenants into buckets and serving buckets round-robin — instead of maintaining one queue per tenant.
- **Epoch-based hash perturbation**: Periodically rotating the hash seed so that tenants who collide in one epoch are statistically unlikely to collide in the next, converting a deterministic worst-case (unlucky permanent co-location) into a bounded-probability transient.
- **Stateless SFQ + best-of-k**: Combining the hash-bucketing idea with Power of Two Choices — hash each request to *k* candidate buckets and pick the shortest — to achieve near-optimal tail latency without any coordination or shared state between servers.

## Prerequisites

- Basic queuing concepts: FIFO queue, round-robin scheduling, what "queue depth" means.
- Hash functions: deterministic, deterministic-given-seed (keyed), and why output appears uniform.
- The "noisy neighbor" problem: in multi-tenant systems one heavy tenant can degrade latency for everyone sharing a resource.
- Power of Two Choices: the classical result that sampling 2 random bins and picking the least-loaded one reduces maximum bin height from O(log n / log log n) to O(log log n) — a doubly-exponential improvement from a single extra sample.

## The core idea

The obvious fix for noisy neighbors is per-tenant isolation: one queue per customer, round-robin between them. That works beautifully at small scale. At AWS Lambda scale (millions of customers per fleet), it fails: you need O(customers) memory to track which queue belongs to whom, and the O(customers) round-robin sweep becomes expensive.

SFQ makes the observation that **you do not need perfect, per-tenant fairness — you need statistical fairness**. Instead of one queue per customer, maintain N queues where N is a small constant (say 1024). Assign each incoming request to a queue using a hash of the customer identifier. Serve queues strictly round-robin. That's the whole algorithm.

The properties are remarkable: O(1) queues, O(1) enqueue, O(1) dequeue. A noisy tenant can only monopolize the queues it hashes into — and if it happens to hash into only 1 of 1024 queues, it can at most slow down the other tenants who happen to share that queue, while leaving the remaining 1023 queues completely unaffected.

The catch is hash collisions: with a static hash function, a tenant unlucky enough to permanently share a queue with a heavy neighbor is stuck there forever. Brooker's post addresses this with **epoch rotation** — periodically changing the hash seed, so colliding tenants are reshuffled into different queues at each epoch boundary. Collision in any single epoch is unavoidable; persistent collision across many epochs is exponentially unlikely.

The final embellishment is **best-of-k**: instead of hashing to exactly one queue, hash to *k* candidate queues (e.g., k=2 or 3) and enqueue into the shortest. This imports the Power of Two Choices result into the SFQ framework, tightening the tail without adding shared state. Brooker combines all three: SFQ + shuffle sharding + best-of-k, each layer multiplying the isolation benefit.

## Mechanics

**Basic SFQ**

Maintain an array of N queues. On each enqueue, compute `q = hash(tenant_id) mod N` and append to `queues[q]`. A background dequeue loop scans `queues[0..N-1]` in round-robin order, pulling one item from each non-empty queue per pass.

Because the dequeue loop touches each queue at most once per pass, no single tenant can take more than `1/N` of the total throughput when N customers are each mapped to their own bucket. When two tenants share a bucket they each get `1/(2N)` of throughput from that bucket — half of what they'd get in isolation, but still bounded.

**Epoch rotation**

Add an epoch counter that increments every T seconds (T might be 30–300 seconds depending on the workload). Include the epoch in the hash input:

```
q = hash(tenant_id, epoch) mod N
```

Any two tenants that collide in epoch E are independently re-hashed in epoch E+1. The probability they collide again is 1/N (assuming a good hash function). Over K epochs, the probability they collide in *all* K epochs is (1/N)^K — exponentially small. Practically, even with N=1024 and K=3 epochs, that's a 1-in-a-billion chance.

There is a cost: at each epoch boundary, in-flight items may be in "stale" queues. The implementation needs to either drain old queues gracefully or tolerate a brief ordering anomaly. For placement systems (rather than ordered queues) this is usually a non-issue.

**Shuffle sharding variant**

Instead of mapping each tenant to exactly one queue, map each tenant to a set of k queues drawn pseudorandomly from the N. This is shuffle sharding applied to queuing: the tenant "owns" a random k-subset of buckets. When a request arrives, pick the least-loaded bucket from that set. Two tenants interfere only if their randomly-assigned subsets overlap, and the overlap probability for two random k-subsets out of N is `C(N,k)^-1` — combinatorially tiny for even modest N and k.

**Stateless distributed application: AWS Lambda**

The payoff becomes clearest in the Lambda worker fleet. The problem: a frontend router must dispatch a Lambda invocation to an idle worker. Workers are not statically partitioned by tenant — any worker can run any customer's function. The router needs to find a worker quickly without global state.

Using SFQ + best-of-k: the router hashes `(customer_id, epoch)` to k worker indices, probes those k workers, and sends to the least loaded. Every router independently computes the same hash, so they all examine the same k candidates for a given customer in a given epoch — complete consistency without a central directory. The result: tight noisy-neighbor bounds with **zero shared state between routers**.

Compare to alternatives:
- Random placement: no isolation, heavy tenants degrade everyone.
- Consistent hashing to fixed workers: isolation, but hot workers become permanent bottlenecks and adding workers requires remapping.
- Central load balancer with per-tenant state: isolation, but the LB is a coordination bottleneck and a single point of failure.

SFQ+best-of-k hits the intersection: isolation, no coordination, no static partitioning.

## Where it breaks

**Heavy-hitter saturation**: If a single tenant generates load that fills multiple buckets, the epoch rotation only helps other tenants; the heavy hitter itself may still land in contested buckets for full epochs. SFQ is not a substitute for rate limiting — it prevents one tenant from hurting others, not from hurting itself.

**Small N**: With too few queues, collision probability rises and the fairness guarantee weakens. N must be chosen large enough that a "normal" tenant is alone in its bucket most of the time. This requires knowing the expected number of concurrent active tenants.

**Epoch transition cost**: Changing the hash seed invalidates all current queue assignments. For placement-style systems (choosing which worker to use), this is fine — the next request picks a new worker. For session-affinity workloads (where the same server needs to handle all requests from one client), epoch rotation would require re-establishing affinity, adding overhead.

**Statistical, not hard, guarantees**: SFQ gives you P(bad isolation) < ε, not guaranteed isolation. A pathologically unlucky tenant can still experience poor service for a full epoch. If your SLA requires deterministic isolation, SFQ is not enough — you need explicit partitioning.

**Does not account for heterogeneous item cost**: Round-robin treats all queue items as equal. If different tenants have wildly different per-request costs (e.g., one submits 1ms tasks, another submits 10s tasks), round-robin gives the heavy-cost tenant far more resource consumption per dequeue pass. Weighted fair queuing or token-bucket rate limiting is needed in that case.

## Why it works

The post's deepest insight is that SFQ is an instance of a much more general principle: **replace exact tracking with probabilistic partitioning via universal hashing**.

This substitution appears everywhere in systems:

| Exact approach | Probabilistic replacement | Same tradeoff |
|---|---|---|
| Per-tenant queue (O(tenants) state) | SFQ hash buckets (O(1) state) | This post |
| Exact set membership | Bloom filter | False-positive ε, O(1) state |
| Exact frequency count | Count-Min Sketch | Over-count ε, O(1) state |
| Global load-balance state | Power of Two Choices | Near-optimal, 2 samples |
| Static consistent hashing table | Virtual node rendezvous | Same distribution, O(1) remapping |

In every case: exact tracking requires O(n) state and O(n) coordination; hashing requires O(1) state, zero coordination, and gives a probabilistic guarantee that is *almost as good* for practical purposes.

The epoch rotation is the key refinement that lifts SFQ above simple hash-mod: without it, a bad hash assignment is permanent. With it, the worst case is *temporary* and the probability of persistent bad luck decays geometrically with each epoch. This is structurally identical to why randomized algorithms use fresh random bits per invocation — it prevents adversarial inputs from finding fixed bad cases.

The best-of-k layer then imports the Power of Two Choices insight: even just *looking* at two options and picking the better one produces exponentially better balance than random. Combined, the three techniques (hash bucketing, epoch rotation, best-of-k) give a system that is simultaneously O(1) in state, zero-coordination across nodes, and near-optimal in load distribution — a combination that is hard to achieve any other way.

Brooker's framing is elegant: this is not a niche networking trick. It is a **scheduling primitive** applicable anywhere a system needs multi-tenant fairness without per-tenant tracking. Application-level queues, thread pool schedulers, database connection pools, CDN cache partitions — any shared resource serving multiple tenants is a candidate.

## Going deeper

1. **McKenney, P. (1990). "Stochastic Fairness Queuing"** — the original paper introducing SFQ in a networking context. Available via ResearchGate. The algorithm is simpler than it sounds; reading the original makes clear how directly the idea translates from packet queuing to any queuing problem.

2. **Mitzenmacher, M. (2001). "The Power of Two Choices in Randomized Load Balancing"** — the definitive analysis of best-of-k. The proof that k=2 gives O(log log n) maximum load vs. O(log n / log log n) for random is the key result; the intuition (a second sample breaks symmetry) is more broadly useful.

3. **Marc Brooker, "Finding Needles in a Haystack with Best-of-K" (2024)** — `brooker.co.za/blog/2024/03/25/needles.html` — Brooker's earlier post that introduces the best-of-k framing for fleet placement problems; the SFQ post builds directly on it.
