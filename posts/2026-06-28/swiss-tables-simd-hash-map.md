---
title: "Faster Go maps with Swiss Tables"
source: https://go.dev/blog/swisstable
author: Michael Pratt
company: Go team / Google
date_posted: 2025-02-26
date_digested: 2026-06-28
---

# Faster Go maps with Swiss Tables

## What's new to learn

1. **Metadata-data separation**: Storing a 7-bit hash fingerprint per slot in a compact control array — separate from the actual key-value data — lets SIMD scan 8 slots' metadata in a single instruction without touching any key bytes in the common case.

2. **Two-level hashing (H1/H2)**: Splitting one hash into a *group selector* (H1, upper bits) and a *slot fingerprint* (H2, lower 7 bits) separates "where to start looking" from "does this slot even have a chance of matching?" The H2 acts as a per-slot Bloom filter that eliminates ~127/128 comparisons before dereferencing any key.

3. **Three-state slots (EMPTY / DELETED / FULL)**: Distinguishing a never-used slot from a deleted tombstone lets probe sequences stop early at EMPTY while correctly skipping past DELETED, eliminating the need to rehash-on-delete that plagues simpler open-addressing schemes.

## Prerequisites

- How open-addressing hash tables work: hash a key, compute a slot index, scan forward if the slot is wrong (linear or quadratic probing)
- What a CPU cache line is: a 64-byte block fetched atomically from RAM to L1 cache on a miss
- Basic intuition about SIMD: a CPU instruction that performs the same operation on multiple data values simultaneously (e.g., comparing 8 bytes to a target in one clock)
- The tombstone problem: why you cannot mark a deleted slot as EMPTY without breaking probe chains for keys inserted after it

## The core idea

Classical open-addressing hash tables pack keys and values in a single flat array. Finding a key means hashing, jumping to a slot, and advancing if the slot doesn't match — but every slot test loads the full key-value entry (potentially 32–64 bytes) just to check equality. Most of those bytes are wasted on misses.

Swiss Tables decouple *metadata* from *data*. The table is divided into fixed-size **groups** of 8 slots. Each group has:

- An **8-byte control word** (one byte per slot), stored contiguously before the slot data.
- The actual **key-value pairs** for those 8 slots, stored after.

Each control byte holds one of:
- A 7-bit fingerprint (H2) if the slot is occupied — bit 7 is clear.
- A sentinel value with bit 7 set if the slot is EMPTY or DELETED.

On lookup, you load the 8-byte control word and SIMD-compare all 8 bytes against H2 in one instruction. The result is a bitmask of *candidate* slots. Only for candidates do you load the actual key and compare. In the common case (no match in this group) you touch exactly 8 bytes of memory per group, not 8× (key size) bytes.

This lets you operate at a higher load factor (87.5% vs. ~81%) with *fewer* cache misses per probe, not more.

## Mechanics

**Hash splitting.** For any lookup key `k` with hash `h`:

```
H1 = h >> 7          // selects the starting group
H2 = h & 0x7F        // 7-bit fingerprint (0x00–0x7F)
```

**Control byte encoding:**

| Value          | Meaning                                   |
|---------------|-------------------------------------------|
| `0x00–0x7F`   | FULL — slot occupied; value = H2 of stored key |
| `0x80`        | EMPTY — slot never used; stop probing here     |
| `0xFE`        | DELETED — tombstone from a prior deletion      |

The invariant: bit 7 set ↔ slot is not FULL. This makes the SIMD "find all FULL slots matching H2" and "find any EMPTY slot" separable by a single bit mask.

**Lookup:**

```
group = H1 mod num_groups
loop:
    ctrl = load 8 control bytes of group
    match_mask = ctrl_match(ctrl, H2)    # SIMD: bitmask of slots where byte == H2
    for each set bit i in match_mask:
        if key[group][i] == lookup_key: return value[group][i]
    if any byte in ctrl == EMPTY: return NOT_FOUND
    group = next_group(group)            # quadratic probe advance
```

**Insertion:**

1. Run the lookup loop. If key found, update in place.
2. During the probe, track the first EMPTY or DELETED slot seen.
3. After confirming the key is absent, write the key-value pair to the tracked slot; write H2 to its control byte.

**Deletion:**

1. Find the slot (lookup).
2. Write the DELETED sentinel to the control byte. **Do not write EMPTY** — that would silently hide keys further down the probe chain that were placed there because this slot was occupied at insertion time.

**Rehashing on growth:**

1. Allocate a new backing store (double the groups).
2. Re-insert every FULL slot. DELETED slots are simply not copied — they become EMPTY in the new table. This "launders" accumulated tombstones.

**SIMD vs. portable fallback.** On amd64, the match uses SSE2 `pcmpeqb` / `pmovmskb`. On other architectures, Go 1.24 uses a portable 64-bit word trick:

```go
// broadcast H2 to all 8 byte positions
needle := uint64(h2) * 0x0101010101010101
xored  := ctrl_word ^ needle         // matching bytes become 0x00
// hasZeroByte trick: each 0x00 byte sets its 0x80 bit
mask   := (xored - 0x0101010101010101) & ^xored & 0x8080808080808080
// bits 7,15,23,... correspond to slots 0,1,2,...
```

**Load factor improvement.** Old Go maps used a bucket-based design with overflow buckets and a max load factor of 13/16 (81.25%). Swiss Tables target 7/8 (87.5%). For a map with 3.5 million entries, Datadog measured memory drop from 726 MiB to 217 MiB (70% reduction) after upgrading to Go 1.24.

**Probe sequence.** Swiss Tables use a quadratic sequence over *groups* — not individual slots — jumping to groups at offsets 0, 1, 3, 6, 10, ... from the starting group (triangular number increments modulo `num_groups`). Because `num_groups` is a power of two, all groups are visited before repeating, guaranteeing eventual termination.

## Where it breaks

- **Small maps**: For maps with fewer than ~8 entries, the group overhead is not worth it. Go 1.24 keeps an inline fast path for tiny maps.
- **H2 false positives**: 1/128 collisions on the fingerprint still require a full key comparison. The fingerprint filters, it doesn't prove.
- **Concurrent access**: Swiss Tables are not thread-safe. Concurrent writes require `sync.Map`, a mutex, or a sharded design.
- **Architecture coverage**: Go 1.24 uses SIMD only on amd64; other platforms use the portable bit-trick, which is still faster than the old design but doesn't exploit vector units.
- **Tombstone accumulation**: Under heavy delete-insert churn, DELETED slots accumulate until the next rehash, slightly increasing probe chain lengths. A table that never shrinks can carry tombstones indefinitely if growth never triggers.
- **Hash quality matters more**: Because H2 is only 7 bits, a poor hash function that clusters low bits will cause H2 collisions and degrade the false-positive filter. Swiss Tables inherit all of open addressing's sensitivity to hash quality.

## Why it works

The central idea is **"scan cheap metadata first, fetch expensive data only on a hit"** — a pattern that appears at every level of the systems stack:

- **B+-trees**: Internal nodes store only separator keys (cheap) to route lookups; values live in leaf nodes (expensive). The hot path through the tree never touches data until the final leaf.
- **Column stores**: A predicate `WHERE status = 'ACTIVE'` scans only the `status` column's compact encoded bytes before fetching any other column. Null bitmaps and run-length encoding metadata are similarly cheap first-passes.
- **CPU L1 cache tag arrays**: Cache tags — compact address bits identifying each 64-byte cache line — live in a small dedicated SRAM, separate from the data SRAM. A cache lookup reads tags first (cheap), then fetches the data line only on a tag hit (expensive).
- **Struct-of-Arrays (SoA)**: SIMD loops prefer data laid out as one array per field rather than one array of structs, precisely because the "hot" field for filtering is packed contiguously and can be loaded without interleaved waste from "cold" fields.

The SIMD comparison converts 8 sequential probes into 1 instruction — O(1) with a constant that fits in a single clock cycle on modern CPUs. And grouping control bytes into a 64-bit word means a single cache-line fetch covers the metadata for 8 slots, vs. 8 separate cache-line fetches for a naive approach.

The EMPTY/DELETED/FULL three-state design is also the same "two kinds of absence" insight underlying free lists in memory allocators (a block that's never been allocated vs. one that was returned), or the distinction between `nil` (never set) and an explicit zero value in optional-typed languages. Both are about making the *reason* for absence structurally observable.

## Going deeper

1. **Abseil Swiss Tables design notes** — https://abseil.io/about/design/swisstables — the canonical design documentation from the original Google Abseil team, with pseudocode for the SIMD match operations and a discussion of the two-level open-addressing scheme.

2. **CppCon 2017: Matt Kulukundis, "Designing a Fast, Efficient, Cache-friendly Hash Table, Step by Step"** — https://www.youtube.com/watch?v=ncHmEUmJZf4 — the original presentation walking through the design from scratch; explains *why* each decision was made, not just what it is.

3. **Datadog: "How Go 1.24's Swiss Tables saved us hundreds of gigabytes"** — https://www.datadoghq.com/blog/engineering/go-swiss-tables/ — a concrete production case study with before/after heap profiles, showing the 70% memory reduction on a 3.5M-entry routing cache and how to profile map memory in a live Go service.
