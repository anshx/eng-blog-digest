---
title: "MetaRoCE: A New RDMA Transport Built for AI-Scale Ethernet"
source: https://engineering.fb.com/2026/08/24/networking-traffic/metaroce-rdma-transport-ai-ethernet/
author: Meta Engineering
company: Meta
date_posted: 2026-08-24
date_digested: 2026-09-24
---

# MetaRoCE: A New RDMA Transport Built for AI-Scale Ethernet

## What's new to learn

1. **PFC deadlock — lossless fabric's hidden failure mode at scale**: Standard RDMA over Ethernet (RoCEv2) requires Priority Flow Control (PFC) to pause upstream switches when buffers fill. At scale, switches form cyclic buffer dependencies — each waiting for its downstream neighbor — creating permanent deadlocks that paralyze entire fabric segments.

2. **Loss-tolerant RDMA**: MetaRoCE abandons the lossless-fabric requirement entirely and instead handles packet loss in NIC hardware using per-path sequence tracking plus a 256-bit Selective Acknowledgment (SACK) map — applying the End-to-End Argument to datacenter networking.

3. **Receiver-driven fair-share rate hints**: Instead of DCQCN/AIMD's probe-and-backoff distributed search, each receiver NIC computes the max-min fair bandwidth allocation across all active senders and returns that exact rate in every ACK — replacing iterative distributed convergence with a single receiver-side computation per RTT.

## Prerequisites

- What RDMA is: the NIC moves data directly between host memory regions without CPU involvement, using an explicit target address that the sender specifies in the RDMA write/send operation.
- Basic packet-switched networking: switches forward packets; ECN (Explicit Congestion Notification) marks packets when buffers approach capacity.
- TCP's AIMD: senders probe by increasing their window each RTT until loss/ECN, then halve on a congestion signal.
- Why AI training needs low-latency RDMA: collective operations (AllReduce, AllGather) synchronize gradients across thousands of GPUs every training step; a single stalled connection stalls the entire collective.

## The core idea

RDMA over Ethernet was designed around one guarantee: the fabric will never drop a packet. Achieving this requires Priority Flow Control (PFC) — a pause mechanism where a switch sends a "stop sending" frame to its upstream neighbor when its buffers fill. Pause frames cascade backward through the network, creating backpressure that reaches the sending NIC.

At small scales this works. At the scale of a 100,000+ GPU training cluster, it catastrophically fails.

The problem is **cyclic buffer dependency**. Imagine three switches arranged so that traffic from A passes through B, from B passes through C, and from C passes through A (which can happen under ECMP routing when elephant flows from different sources interleave). Each switch pauses its upstream neighbor because the downstream neighbor is also paused. The result is a deadlock: every switch is holding buffer space, waiting for a buffer that will never drain. The network freezes. Meta and others have confirmed these deadlocks occur in practice in large clusters.

MetaRoCE's answer is radical: **stop asking the network to be lossless**. Treat Ethernet exactly as it is — a best-effort packet network — and move reliability into the NIC endpoints. A dropped packet is detected by a gap in per-path sequence numbers, triggering a targeted selective retransmit. PFC is never needed, deadlocks cannot form, and the fabric can freely drop packets under congestion rather than stalling.

The second problem MetaRoCE solves is congestion control convergence. The standard approach (DCQCN, used by RoCEv2) is AIMD driven by ECN marks: senders gradually increase their rate until they see congestion marks, then cut by half. Convergence to max-min fair sharing takes many RTTs of probe-and-backoff. During that search, some flows get starved while others overbuy bandwidth.

MetaRoCE replaces this search with a **receiver-driven fair-share hint**: the receiving NIC sees every packet arriving from every active sender. It can compute, in real time, each sender's fair-share allocation using max-min fairness. That computed rate is piggybacked onto every ACK. The sender reads "your fair share is 42 Gbps" and immediately sets its pacing rate — no searching needed.

## Mechanics

### Connection splitting and multipath

When two NICs establish a MetaRoCE connection, they negotiate a set of logical **paths** — one for each distinct route through the ECMP fabric. Each path is identified independently and carries its own state:

- Per-path **sequence number** — a monotonically increasing counter assigned by the sender to every packet on that path.
- Per-path **RTT estimate** — updated from ACK timestamps.
- Per-path **ECN state** — how many recent packets on this path carried an ECN mark.
- Per-path **utilization** — inferred from timing between ACKs.

This per-path bookkeeping lets the NIC steer traffic toward less-congested paths and detect problems on a per-path basis without the entire connection being affected.

### Out-of-order delivery without a reorder buffer

Because different paths have different latencies, packets from the same RDMA write can arrive out of order. Conventional wisdom says you need a reorder buffer to reconstruct the original sequence before delivering to the application.

MetaRoCE eliminates that buffer by exploiting a property unique to RDMA semantics: **every packet already carries its final destination memory address**. An RDMA write specifies the remote virtual address and offset that each byte should land at. When a packet arrives, the NIC knows exactly where in the receiver's memory region to DMA it — regardless of whether earlier or later packets in the same write have arrived.

The result: MetaRoCE writes each arriving packet **directly to its final memory location** immediately upon arrival. There is no staging area. The 256-bit SACK vector acts purely as a **presence map** — a bitmap of which sequence positions within the current window have been received. When the sender gets an ACK with a SACK vector showing gaps, it retransmits only the missing packets.

This is why MetaRoCE can sustain 86% throughput at 1% packet loss and continue operating at 10% loss rates: retransmits are targeted at specific gaps, all other packets are already in final memory, and no flow-wide stall occurs.

### Receiver-driven fair-share hints

The receiving NIC maintains a **flow table** — an entry per active sender showing recent arrival rate and allocated bandwidth. On every ACK:

1. Divide total inbound link capacity by the number of active senders to get a baseline per-sender allocation.
2. For each sender using *less* than its baseline: mark its unused bandwidth as available.
3. Redistribute available bandwidth to senders that could use more (bounded by their actual demand).
4. Repeat until no sender's allocation can be increased — this is max-min fairness, and it converges in at most N_senders iterations.
5. Write the resulting allocation for this sender into the ACK header.

The sender reads that rate in the ACK and adjusts its pacer. ECN marks from the network are still honored (the sender cuts rate on ECN) as a fast-response safety mechanism; receiver hints handle steady-state allocation.

### Hybrid congestion windows

MetaRoCE maintains a congestion window at two levels:

- **Per-path window**: controls how many bytes can be in flight on one logical path. Responds quickly to per-path ECN marks by trimming that specific path.
- **Per-connection window**: aggregate outstanding bytes across all paths. Responds to receiver-driven hints and ensures the total rate stays within the receiver's allocation.

When a path becomes congested, traffic is automatically shifted to other paths in the connection without affecting the total rate.

## Where it breaks

**NIC ASIC dependency.** All per-path state, the SACK logic, and the receiver-side fair-share computation run in NIC silicon. MetaRoCE requires new NIC hardware (Meta developed it with AMD Pensando). Existing RoCEv2 NICs cannot be firmware-upgraded; deployed infrastructure must be replaced.

**Receiver-side computation at very high fan-in.** If thousands of senders all target the same receiver (as happens with AllReduce on a large cluster), the receiver's flow table and fair-share computation grow proportionally. This is bounded but requires NIC silicon area and clock cycles that scale with max fan-in.

**Memory bandwidth for SACK tracking.** Writing out-of-order packets directly to final memory locations is elegant but creates scattered memory writes at the receiver. At 400 Gbps with small packets, this pattern of non-sequential DMA writes can stress the host memory bus.

**Single-vendor hardware today.** The spec is being contributed to OCP (Open Compute Project), and the reference implementation is DPDK-based, but broad silicon support is months to years away.

**Assumes trusted, well-behaved senders.** The receiver fair-share hint is advisory — a malicious or buggy sender can ignore it. MetaRoCE is designed for intra-cluster AI workloads, not adversarial internet traffic.

## Why it works

Two principles combine to make MetaRoCE work:

**The End-to-End Argument** (Saltzer, Reed, Clark, 1984): reliability guarantees should be provided at the communication endpoints, not inside the network. Lossless fabric is expensive, fragile, and deadlock-prone at scale. Moving loss detection and recovery to the NICs is cheaper, more reliable, and removes a whole failure mode (PFC deadlock). The network just needs to move bits; the NICs worry about which bits arrived.

**"The holder of global state should be the scheduler."** AIMD distributes the bandwidth allocation problem across all senders, who independently probe for their fair share. This works but converges slowly and oscillates. The receiving NIC has something no individual sender has: a real-time view of *all* active senders and *all* their arrival rates simultaneously. Max-min fair allocation is a trivially computable function of that information. When you have global state, use it for global decisions — don't distribute the problem back out to the parties who only have local views.

This second principle appears widely:
- **BBR congestion control**: rather than inferring capacity from loss (a symptom), BBR directly measures bottleneck bandwidth and minimum RTT — replacing distributed symptom-chasing with direct measurement.
- **Maglev consistent hashing**: rather than computing hash ring membership at query time, precompute the entire ring into a lookup table — replacing online computation with offline scheduling.
- **Database lock managers**: centralized, because the lock manager has the complete lock-graph that individual transactions cannot see.
- **Disruptor ring buffer**: a single sequence counter acts as the authoritative scheduler rather than having all producers compete via CAS.

The pattern: identify which component has complete information about the shared resource, make that component the scheduler, and have it explicitly allocate rather than having participants search.

MetaRoCE applies that pattern to the problem of bandwidth fairness across thousands of GPU-to-GPU connections, at a level of precision (per-RTT per-sender allocation) that distributed AIMD cannot achieve without orders of magnitude more convergence time.

## Going deeper

1. **"RDMA over Commodity Ethernet at Scale"** (Guo et al., SIGCOMM 2016, [dl.acm.org](https://dl.acm.org/doi/10.1145/2934872.2934908)) — the paper that documents PFC deadlock, congestion spreading, and head-of-line blocking in RoCEv2 deployments, and introduces DCQCN, the algorithm MetaRoCE replaces.

2. **"Homa: A Receiver-Driven Low-Latency Transport Protocol for Datacenter Networks"** (Montazeri et al., SIGCOMM 2018, [homa.cs.stanford.edu](https://homa.cs.stanford.edu/)) — the academic predecessor: receiver-driven scheduling for datacenter TCP replacement, including the priority-based grant mechanism that directly inspired receiver-driven rate hints in MetaRoCE.

3. **"BBR: Congestion-Based Congestion Control"** (Cardwell et al., 2016, [queue.acm.org](https://queue.acm.org/detail.cfm?id=3022184)) — the model-based congestion control that establishes the principle of measuring the bottleneck directly rather than inferring it from loss signals, the same philosophy MetaRoCE applies at the receiver.
