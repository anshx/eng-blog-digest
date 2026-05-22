---
title: "BBR: Congestion-Based Congestion Control"
source: https://queue.acm.org/detail.cfm?id=3022184
author: Neal Cardwell, Yuchung Cheng, C. Stephen Gunn, Soheil Hassas Yeganeh, Van Jacobson
company: Google
date_posted: 2016-09-01
date_digested: 2026-05-22
---

# BBR: Congestion-Based Congestion Control

## What's new to learn

1. **The Kleinrock optimal operating point**: a network path has exactly two fundamental parameters — bottleneck bandwidth (BtlBw) and round-trip propagation time (RTprop). Their product (the bandwidth-delay product, BDP) is the exact in-flight byte count that maximizes throughput with zero persistent queueing. It is computable, not a guess.

2. **Loss ≠ congestion**: Every TCP algorithm before BBR (Reno, CUBIC, Tahoe) used packet loss as the primary congestion signal. BBR shows this is a category error — loss is a late-arriving, noisy proxy for congestion; measuring BtlBw and RTprop directly gives you the actual state of the pipe, with none of the structural failure modes loss-based control has.

3. **Pacing rate as the control variable**: Traditional TCP controls only the congestion window (cwnd, a byte count). BBR also controls pacing rate (bytes/second). Sending cwnd bytes uniformly across an RTT instead of in a burst eliminates the transient queue spikes responsible for most bufferbloat — even when cwnd is at the ideal level.

## Prerequisites

- What a congestion window (cwnd) is and how AIMD (Additive Increase, Multiplicative Decrease) works in Reno/CUBIC
- What the bandwidth-delay product (BDP) means: the number of bytes that can be "in flight" to fully fill a pipe without overflow
- What bufferbloat is: sustained queue buildup in network buffers that inflates latency without causing loss
- That RTT increases before packet loss occurs when buffers are full — i.e., that delay is an earlier signal than loss

## The core idea

Every network path has two fundamental physical parameters: its bottleneck bandwidth (BtlBw, the rate of the slowest link along the path) and its round-trip propagation time (RTprop, the base latency assuming no queueing). Their product — the bandwidth-delay product (BDP) — is exactly how many bytes should be in flight at any moment to fully utilize the pipe with zero queueing delay. This is Kleinrock's optimal operating point.

Traditional TCP never computes BDP. Instead, it reacts to packet loss. This creates two structurally distinct failure modes:

- **Shallow buffers**: Loss occurs before the buffer fills, so TCP backs off before ever reaching full link utilization. You leave bandwidth on the table.
- **Deep buffers (bufferbloat)**: TCP only backs off once loss finally arrives — by which point the buffer may hold hundreds of milliseconds of queued traffic. The connection delivers throughput but at terrible latency. Every web page, video call, and game suffers.

BBR estimates BtlBw and RTprop continuously, then drives the sending rate such that exactly BDP bytes are in flight. When in-flight bytes equal BDP, you get maximum throughput and zero persistent queue.

Google deployed BBR first on its internal B4 WAN in 2015, then on YouTube in 2016. Results: 2–25x higher throughput than CUBIC on the B4 network; 4% higher YouTube throughput globally, 14%+ in developing countries where last-mile conditions are worst.

## Mechanics

BBR runs as a state machine with four phases. At all times it maintains two estimates:

- **`BtlBw`**: the maximum observed *delivery rate* (bytes acknowledged per second) over the last ~10 RTTs — a sliding-window maximum.
- **`RTprop`**: the minimum observed RTT over the last 10 seconds — a sliding-window minimum.

#### Phase 1: STARTUP

Analogous to TCP slow start. BBR doubles the pacing rate every RTT until `BtlBw` saturates (no significant increase over three consecutive RTTs). This fills the pipe quickly. The cost: one full BDP worth of excess bytes ends up queued in the bottleneck buffer.

#### Phase 2: DRAIN

BBR immediately cuts pacing gain to `ln(2)/2 ≈ 0.35` to drain the startup queue. Once the estimated RTT returns toward `RTprop`, the queue is clear and BBR transitions to the steady state.

#### Phase 3: PROBE_BW (steady state, >98% of connection lifetime)

BBR runs repeating 8-RTT cycles:

| Sub-phase | Duration | Pacing gain | Purpose                                |
|-----------|----------|-------------|----------------------------------------|
| Probe up  | 1 RTT    | 1.25        | Send 25% faster to test for more BW   |
| Drain     | 1 RTT    | 0.75        | Send 25% slower to clear any new queue |
| Cruise    | 6 RTTs   | 1.0         | Send at estimated BtlBw                |

The 1.25 / 0.75 pulses keep `BtlBw` fresh: if bandwidth changes (another flow exits, or a wireless link improves), the next probe-up RTT observes the new delivery rate and updates the estimate. The drain RTT ensures no net queue is added over the cycle.

#### Phase 4: PROBE_RTT

Every 10 seconds, if `RTprop` has not been refreshed recently, BBR enters this phase: it drops cwnd to 4 packets (the minimum) for at least 200ms. This completely drains all buffers on the path, exposing the true propagation delay. After collecting a fresh `RTprop` sample, BBR returns to PROBE_BW.

#### Pacing vs. cwnd

The most subtle mechanism: BBR controls both a pacing rate and a cwnd. Traditional TCP can transmit a full cwnd in a burst at line rate; the burst fills the buffer, the queue drains, then the next burst arrives. BBR spaces packets to be sent at precisely `pacing_rate = BtlBw × pacing_gain` — roughly one packet every `packet_size / pacing_rate` seconds. This eliminates transient queue spikes even when cwnd is sized correctly.

The sending constraints are:

```
pacing_rate = BtlBw × pacing_gain
cwnd        = BtlBw × RTprop × cwnd_gain    (with a small minimum)
```

Actual bytes in flight are limited by the stricter of the two. In PROBE_BW cruise, both equal BDP (pacing_gain = cwnd_gain = 1.0), so neither constraint bites and the pipe runs exactly full.

## Where it breaks

**RTT unfairness**: Two competing BBR flows with RTTs of 100ms and 10ms both run 8-RTT probe cycles — but 8 × 10ms = 80ms while 8 × 100ms = 800ms. The short-RTT flow probes bandwidth ten times more often and acquires a disproportionately large share of a shared bottleneck. BBRv2 partially addresses this by incorporating an explicit loss-rate signal, which indirectly penalizes flows consuming more than their fair share.

**Coexistence with loss-based flows**: BBR does not back off on packet loss the way CUBIC does. On a bottleneck link shared with CUBIC flows, BBR's refusal to yield when packets drop can crowd out the loss-based flows, which are multiplying their window down. On very shallow buffers, this can starve CUBIC completely.

**ACK compression inflating BtlBw**: On cellular and WiFi paths, ACKs may be delayed and then arrive in rapid bursts. The instantaneous delivery rate of a burst of ACKs looks much higher than the actual bottleneck rate. BBR may lock onto this inflated estimate and over-send until the next probe-up cycle detects the overestimate.

**Slow decay after a competing flow exits**: `BtlBw` is a 10-RTT sliding-window max. If a large competing flow suddenly departs and the available bandwidth drops, the old (higher) estimate persists for up to 10 RTTs before decaying. During that window BBR over-sends.

**PROBE_RTT desynchronization across flows**: Multiple BBR flows on the same path each independently decide when to enter PROBE_RTT. If they don't synchronize their drain periods, the bottleneck queue never fully empties during any single flow's PROBE_RTT phase, and all flows systematically overestimate RTprop. This leads to BBR running persistently above the optimal point.

## Why it works

**BBR is model-based control applied to TCP.** Traditional congestion control is reactive: change behavior only in response to events (loss, ECN marks). BBR is closed-loop: it maintains an explicit model of the pipe (two numbers: BtlBw, RTprop), computes the desired operating point (BDP bytes in flight), and uses pacing + probing to drive the system toward that point. This is the same architecture as a PID controller, a Kalman filter, or model predictive control in any domain.

**The measurement dilemma — and how BBR solves it.** BtlBw and RTprop cannot be measured simultaneously. Filling the pipe to probe for bandwidth adds queueing that inflates the measured RTT above RTprop. Draining the queue to get a clean RTprop sample requires sending below BtlBw. BBR resolves this with temporal multiplexing: PROBE_BW refreshes BtlBw by periodically over-sending; PROBE_RTT refreshes RTprop by periodically under-sending. They alternate, and the model stays fresh. This is analogous to how quantum measurement cannot observe two conjugate variables simultaneously — you rotate the measurement basis in time.

**The deeper principle**: when the natural observable for a control loop is a proxy for what you actually want to optimize, model the underlying quantity directly. Packet loss is not "congestion" — it is the overflow symptom of a buffer that has already been congested for some time. Measuring the bottleneck's capacity and propagation delay directly is strictly better than inferring them from queue-overflow events. This principle recurs throughout engineering:

- Observability: tracking latency percentiles directly vs. inferring health from error rates
- Database query planning: measuring actual cardinality statistics vs. using stale estimates
- ML training: using losses whose gradients align with the actual objective vs. convenient proxies
- Distributed systems: measuring resource utilization directly vs. reacting to timeouts

## Going deeper

1. **BBRv3 IETF draft** (Cardwell et al., 2023 — ongoing): https://www.ietf.org/archive/id/draft-cardwell-ccwg-bbr-00.html — the latest specification, with explicit loss integration and per-ACK bandwidth samples that fix the RTT unfairness and ACK compression issues.

2. **"Should BBR be the default TCP congestion control protocol?"** (arXiv 2025): https://arxiv.org/abs/2510.22461 — independent evaluation of BBRv2/v3 on 2025-era network topologies including shallow-buffer data center links, 5G, and WiFi; gives an honest accounting of when BBR still loses to CUBIC.

3. **Linux kernel BBR commit** (0f8782ea, 2016): https://github.com/torvalds/linux/commit/0f8782ea14974ce992618b55f0c041ef43ed0b78 — the original ~800-line kernel implementation; the comments read like a compressed version of the paper and show exactly how pacing, cwnd, and the four-state machine are wired together in production.
