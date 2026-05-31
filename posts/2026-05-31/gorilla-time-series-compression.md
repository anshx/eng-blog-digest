---
title: "Gorilla: A Fast, Scalable, In-Memory Time Series Database"
source: https://www.vldb.org/pvldb/vol8/p1816-teller.pdf
author: Tuomas Pelkonen, Scott Franklin, Justin Teller, Paul Cavallaro, Qi Huang, Justin Meza, Kaushik Veeraraghavan
company: Facebook (Meta)
date_posted: 2015-08-01
date_digested: 2026-05-31
---

# Gorilla: A Fast, Scalable, In-Memory Time Series Database

## What's new to learn

- **Delta-of-delta timestamp encoding**: Representing when a sample arrived by its *second* derivative rather than its raw value — so metrics that arrive on schedule cost exactly 1 bit each instead of 64.
- **XOR-based float compression**: Encoding only the bits that *changed* between consecutive 64-bit floats by XORing, then exploiting the block structure of IEEE 754 to store just the non-zero "meaningful" middle bits.
- **Predictive coding applied to monitoring data**: The meta-principle that knowing your data's regularity lets you encode prediction errors instead of values, beating general-purpose compressors by an order of magnitude.

## Prerequisites

- IEEE 754 double-precision floating-point layout (sign + exponent + mantissa, and why two "similar" floats share a prefix of identical bits).
- Basic bit manipulation: XOR, counting leading/trailing zeros.
- What a time-series monitoring system looks like: a metric is a stream of `(timestamp, float_value)` pairs produced at a fixed scrape interval.

## The core idea

A monitoring metric is a stream of `(timestamp, float_value)` pairs. The naive representation is 8 bytes for each field, 16 bytes total per data point. At Facebook's 2015 scale — 2 billion unique time series, roughly 12 million data points per second — that is almost 200 MB/s raw, or more than 16 TB/day, just for the in-memory working set needed to answer queries in milliseconds.

The previous system stored data in HBase on disk. P99 read latency was around 15 seconds, which is long enough to make dashboards unusable during incidents — exactly when you need them most. Facebook built Gorilla to keep the last 26 hours entirely in RAM, with a target P99 query latency of tens of milliseconds.

Gorilla's key insight is that monitoring data is *boring* in the information-theoretic sense:

1. **Timestamps arrive on a schedule.** If a service is scraped every 60 seconds, every inter-sample delta is 60. The delta of deltas is zero. Zero is cheap.
2. **Values change slowly.** CPU utilization does not teleport from 30% to 95% between adjacent samples. Request latency drifts, not jumps. Consecutive floats for slowly-changing metrics differ in only a few mantissa bits.

Gorilla exploits both regularities with two separate encoding schemes — one for timestamps, one for values — then concatenates the bit streams. The result: **average 1.37 bytes per (timestamp, value) pair**, an 11.7× reduction from 16 bytes uncompressed.

## Mechanics

### Time series blocks

Data is organized into non-overlapping 2-hour blocks. The current (head) block lives entirely in memory. Finished blocks are serialized to disk as compact byte arrays and freed from RAM. Within a block, each time series is a pair of independently bit-packed streams: the timestamp stream and the value stream.

### Timestamp encoding (delta-of-delta)

Within a block, Gorilla records:

1. **Block header timestamp** `t₀`: stored in full (64 bits).
2. **First delta** Δ₁ = `t₁ − t₀`: stored as a 14-bit unsigned integer (covers up to 4.5 hours at one-second resolution, enough for a 2-hour block with drift).
3. **Every subsequent timestamp**: compute the delta-of-delta `D = (tₙ − tₙ₋₁) − (tₙ₋₁ − tₙ₋₂)` and write it with a variable-length prefix code:

| `D` value | Bit encoding | Total bits |
|---|---|---|
| `D = 0` | `0` | **1** |
| `−63 ≤ D ≤ 64` | `10` + 7-bit signed value | 9 |
| `−255 ≤ D ≤ 256` | `110` + 9-bit signed value | 12 |
| `−2047 ≤ D ≤ 2048` | `1110` + 12-bit signed value | 16 |
| anything else | `1111` + 32-bit value | 36 |

For metrics scraped on a fixed schedule, `D = 0` for roughly **96% of data points**. Those timestamps cost 1 bit each. A 2-hour block at 60-second resolution has 120 samples; after the 14-bit first delta, 118 samples cost 1 bit — about 15 bytes total for all 120 timestamps.

### Value encoding (XOR-based float compression)

IEEE 754 double-precision layout:

```
bit 63  : sign     (1 bit)
bits 62-52: exponent (11 bits)
bits 51-0 : mantissa (52 bits)
```

Gorilla encodes values as follows:

1. **First value**: stored in full (64 bits).
2. For each subsequent value, compute `X = v_current ⊕ v_previous` (bitwise XOR).
   - If `X = 0` (same value): write `0`. Done. (1 bit)
   - If `X ≠ 0`: write `1`, then inspect the "meaningful block" — the contiguous span of non-zero bits between `lz` leading zeros and `tz` trailing zeros:
     - **Case A** (block fits inside previous block): new `lz ≥ old lz` and new `tz ≥ old tz`. Write `10` followed by only the meaningful bits (reusing the previously stored block boundaries). No overhead for block metadata.
     - **Case B** (new block needed): write `11`, then `5 bits` for `lz`, then `6 bits` for the meaningful-block length, then the meaningful bits.

**Why XOR for floats?** Suppose CPU utilization drifts from 71.3% to 71.5%. Both values have the same sign (0) and the same biased exponent (since both lie in [0.5, 1.0), exponent is the same). Their XOR has:
- 12 matching leading zero bits (sign + exponent)
- Roughly 45 matching trailing zero bits (lower mantissa unchanged)
- Only ~7 "meaningful" bits in the middle

Storing those 7 bits with ~4 bits of overhead (Case A or B encoding) costs 11–20 bits instead of 64. For a metric whose values barely move, most samples are Case A and cost roughly `2 + meaningful_bits` bits.

### System architecture

Each Gorilla host manages a subset of time series (sharded by metric ID via consistent hashing). Two geographically separated regions each hold a full replica. Completed blocks are written to GlusterFS as compressed byte arrays for long-term retention; Gorilla itself is the fast cache over the most recent 26 hours.

On restart, Gorilla recovers from the most recent periodic checkpoint plus a mutation journal, then accepts reads immediately — a sub-minute recovery path versus the minutes-scale HBase cold start.

### End-to-end numbers

| Metric | Before Gorilla (HBase) | After Gorilla |
|---|---|---|
| In-memory footprint per data point | 16 bytes | 1.37 bytes avg |
| Compression ratio | 1× | ~12× |
| P99 read latency | ~15 seconds | ~11 ms |
| System | Disk-backed, query-time decompression | Fully in-memory |

## Where it breaks

**Irregular scrape intervals** eliminate the timestamp benefit. If events are generated on demand (traces, logs, user actions), delta-of-delta has no structure to exploit. The 96% single-bit rate drops to wherever the true distribution of inter-sample jitter falls.

**Fast-oscillating values** eliminate the float benefit. A metric that alternates widely between samples has XOR values with few leading or trailing zeros. Every sample is Case B, costing `2 + 5 + 6 + meaningful_bits` bits — sometimes worse than raw storage for extreme cases.

**No late writes.** Gorilla appends sequentially within the active block. A data point arriving after its 2-hour window is dropped. This is acceptable for scrape-based monitoring but disqualifies Gorilla as a general event store.

**No updates, no deletes, no transactions.** Each `(metric_id, timestamp)` is written exactly once. The design assumes the writer is correct on the first attempt.

**Durability window.** Between periodic checkpoints, unacknowledged writes that haven't been flushed to both replicas can be lost if both copies of a shard fail simultaneously. Facebook accepted this risk: losing a few minutes of monitoring data in a catastrophic failure is tolerable; having the monitoring system itself be unavailable is not.

## Why it works

### The information-theoretic framing

Shannon's source coding theorem says the optimal code for a symbol assigns it `−log₂(p)` bits, where `p` is the symbol's probability. If 96% of timestamp deltas-of-delta are zero, the optimal code for "D = 0" is `−log₂(0.96) ≈ 0.06 bits`. Gorilla uses 1 bit — not quite optimal, but close. The rest of the code table handles the long tail.

The float scheme is analogous: if a metric rarely changes value, P(XOR = 0) is high and the 1-bit case handles most samples. When values do change, XOR reveals the *change mask*, not the value itself, and change masks for slowly-drifting metrics tend to be sparse (few non-zero bits) and concentrated in the same bit positions across samples (same block, enabling Case A re-use).

### XOR vs. arithmetic subtraction

A natural alternative would be `v_current − v_previous` (arithmetic difference). For slowly-changing floats, this difference is a small number — but it is still a floating-point number, with its own sign, exponent, and mantissa fields. You can't just strip the exponent; you have to store it in case the next subtraction result has a different order of magnitude.

XOR avoids this entirely. `X = v_current ⊕ v_previous` is not a float in any meaningful sense; it is a 64-bit integer that is literally a bitmask of "which bits differ." The `lz` and `tz` of that mask tell you exactly which bits changed, with no normalization. Storing only those bits is lossless, with zero semantic overhead.

So Gorilla's float compression is **DPCM (Differential Pulse-Code Modulation) applied in XOR space** rather than arithmetic space. DPCM has been used in audio and video codecs since the 1950s. Gorilla shows that XOR is the natural DPCM operation for binary representations of slowly-changing values.

### The transferable principle

The abstraction: **when you can predict your data, encode the residual**. When predictions are accurate, residuals are small and cheap to store. The entire family of compression techniques is really instances of this:

- **Video P-frames**: predict the next frame from the previous one (motion vectors); encode only the prediction error.
- **Git delta objects**: predict a blob as a close ancestor; encode the byte-level diff.
- **FLAC audio**: fit a linear predictor to recent samples; encode 24-bit residuals at 3–6 bits each.
- **LZ4/LZ77**: predict a substring as a copy from earlier in the stream; encode offset + length instead of the bytes.
- **rsync**: predict remote file blocks as local blocks; send only blocks that differ.

Gorilla adds a domain-specific insight on top: **for IEEE 754 floats, XOR is the right residual**, because it directly produces a sparse bitmask with no exponent normalization overhead.

## Going deeper

1. **"Writing a Time Series Database from Scratch" — Fabian Reinartz (2017)**: How Prometheus adapted Gorilla's chunk format into a production TSDB with label-based indexing and multi-hour block compaction. The canonical practical follow-up: https://fabxc.org/tsdb/

2. **Beringei: The open-source Gorilla implementation — Engineering at Meta (2017)**: Operational context and the decision to open-source the engine, plus discussion of what changed between the paper's design and production reality: https://engineering.fb.com/2017/02/03/core-infra/beringei-a-high-performance-time-series-storage-engine/

3. **"Gorilla: A Fast, Scalable, In-Memory Time Series Database" — VLDB 2015 paper**: The primary source, with full derivations of the encoding tables, failure-mode analysis across a three-region deployment, and benchmark comparisons against HBase: https://www.vldb.org/pvldb/vol8/p1816-teller.pdf
