---
title: "Cuckoo Filter: Practically Better Than Bloom"
source: https://www.eecs.harvard.edu/~michaelm/postscripts/cuckoo-conext2014.pdf
author: Bin Fan, Dave Andersen, Michael Kaminsky, Michael D. Mitzenmacher
company: CMU / Intel Labs / Harvard
date_posted: 2014-10-01
date_digested: 2026-09-14
---

# Cuckoo Filter: Practically Better Than Bloom

## What's new to learn

1. **XOR-relocatable fingerprints**: Storing only a short fingerprint per item, combined with XOR-based alternate-bucket computation, makes fingerprints *self-movable* — you can relocate a fingerprint to its other candidate bucket without ever knowing the original key.
2. **Bucket-based cuckoo hashing raises load factor**: Grouping b slots per bucket instead of 1 lifts the maximum achievable occupancy from ~50% (single-slot cuckoo) to ~95.5% (b=4), which is what makes the space math work out.
3. **Probabilistic sets can support deletion**: Bloom filters cannot delete because a set-bit is shared by any item that hashes there. Cuckoo filters store each fingerprint in a specific, independent slot — so deleting an item is as straightforward as removing one slot entry.

## Prerequisites

- **Bloom filters** (the archive's 2026-08-13 post): k hash functions, bit array, false positive rate, why deletion is impossible.
- **Cuckoo hashing basics**: each key has two candidate buckets; on a collision you evict the existing occupant and relocate it to its alternate bucket. The name comes from the cuckoo bird's habit of displacing other eggs.
- **Binary indexed tables / power-of-2 arithmetic**: understanding why the XOR trick requires table sizes to be powers of 2.

## The core idea

A cuckoo filter is a hash table where each slot stores only a *fingerprint* — a short hash of the inserted key — rather than the key itself. For any key x:

- Fingerprint: `f = h'(x)` (a b-bit truncation of a hash of x).
- Two candidate buckets: `i₁ = h(x) % m` and `i₂ = (i₁ ⊕ hash(f)) % m`.

Lookup checks whether `f` appears in either bucket. If yes, x is *probably* in the set. If no, x is *definitely* not.

The part that makes the whole thing work: **the XOR formula is its own inverse**. Given only the fingerprint `f` and bucket position `i`, the other position is `j = (i ⊕ hash(f)) % m`. And from `j`, the original position is `(j ⊕ hash(f)) % m = i`. Fingerprints carry their own "how to find my other home" address, encoded in their bits.

This self-inverse property is what Bloom filters lack. A Bloom filter writes k separate bits for each item, all of which are shared across items that happen to hash to the same positions. There is no "which item does this bit belong to" answer. In a cuckoo filter, each fingerprint occupies a specific slot, so deletion is just removing that slot's value.

## Mechanics

### Data structure

The filter is an array of `m` buckets, each holding `b` fingerprint slots. Typical parameters: `b = 4` slots, `f = 8`–`16` bits per fingerprint. Table size `m` must be a power of 2 (required for the XOR trick to work correctly — see below).

### Lookup(x)

```
f  = fingerprint(x)          # b-bit hash of x
i1 = hash(x) & (m - 1)       # first candidate bucket
i2 = (i1 ^ hash(f)) & (m-1)  # second candidate bucket (XOR trick)
return (f in bucket[i1]) or (f in bucket[i2])
```

Always exactly two bucket reads — at most two cache-line fetches regardless of false positive rate.

### Insert(x)

```
f  = fingerprint(x)
i1 = hash(x) & (m - 1)
i2 = (i1 ^ hash(f)) & (m-1)

if bucket[i1] or bucket[i2] has empty slot:
    store f there; return OK

# Both buckets full — start eviction chain
i = randomly choose i1 or i2
for n in range(MAX_KICKS):   # typically 500
    j = random slot in bucket[i]
    swap f with bucket[i][j]  # evict existing fingerprint into f
    i = (i ^ hash(f)) & (m-1) # compute evicted fingerprint's alt bucket
    if bucket[i] has empty slot:
        store f there; return OK

return FAILURE  # table too full; needs resize
```

The key step is `i = (i ^ hash(f)) & (m-1)` after the swap: you now hold the evicted fingerprint `f` and its current bucket `i`; its alternate bucket is computed using only those two things.

### Delete(x)

```
f  = fingerprint(x)
i1 = hash(x) & (m - 1)
i2 = (i1 ^ hash(f)) & (m-1)
if f in bucket[i1]: remove one copy from bucket[i1]; return OK
if f in bucket[i2]: remove one copy from bucket[i2]; return OK
return NOT_FOUND
```

**Critical caveat**: you must only delete items you know were inserted. Two different keys x and y can have identical fingerprints and overlapping candidate buckets. If x was never inserted, deleting x might remove y's fingerprint slot, creating a false negative for y.

### Why power-of-2 is required

With `m = 2^k` (mask `M = m - 1`), bitwise AND commutes with XOR:

```
(a ⊕ b) & M = (a & M) ⊕ (b & M)
```

So the round-trip works: `((i ⊕ h) & M) ⊕ h) & M = i` when `i < m`.

With arbitrary `m`, the modulo after XOR breaks the self-inverse property. Most real implementations therefore require power-of-2 table sizes.

### Space and false positive rate

With `b` slots per bucket and `f`-bit fingerprints:

- False positive rate: `ε ≈ 2b / 2^f`  (at most `2b` candidate slot checks, each matches with probability `1/2^f`)
- To achieve target `ε`: `f = log₂(2b / ε)` bits
- Bits per stored item: `f / load_factor` where load_factor ≈ 0.955 for `b=4`

| Target ε | Bloom (bits/item) | Cuckoo b=4, f-bits | Cuckoo (bits/item) |
|----------|-------------------|--------------------|--------------------|
| 1%       | 9.6               | 10 bits            | 10.5               |
| 0.1%     | 14.4              | 13 bits            | 13.6               |
| 0.01%    | 19.2              | 16 bits            | 16.8               |

For high false positive tolerances (ε ≥ 3%), Bloom filters use slightly less space. For low false positive rates and especially for applications that need deletions, cuckoo filters win — and they always win on *lookup speed* because Bloom requires k ≈ log₂(1/ε) cache-line reads versus cuckoo's fixed 2.

At ε = 1%, Bloom uses k = 7 hash functions → up to 7 random memory reads per lookup. Cuckoo always uses 2. Measured throughput favors cuckoo by 2–4× at these rates.

## Where it breaks

**Insertion failure at high load.** When the table is ≥95% full, the eviction chain can form a cycle. Once MAX_KICKS relocations fail, insertion returns an error. The table must be rebuilt with a larger `m`. There is no graceful degradation — unlike a Bloom filter, which accepts all insertions and just increases its false positive rate.

**Spurious deletion corrupts the filter.** If you delete an item that was never inserted, you may remove a legitimate fingerprint, causing future lookups to return false negatives. This is rarely a concern in well-structured use cases but is a footgun when deletes are user-controlled.

**Fingerprint collisions between distinct keys.** Two keys that have the same `f`-bit fingerprint *and* overlap in their candidate buckets are indistinguishable in the filter. This is what creates false positives; it also means you cannot safely delete by key — you can only delete a fingerprint, and if two keys share a fingerprint, you might inadvertently delete the wrong one.

**Power-of-2 table sizes waste space.** If you need to store exactly 3 million items and target 90% load, you need `m = 4M / (4 × 0.9) ≈ 1.1M` buckets — rounded up to `2^21 = 2M`, wasting 90% of the capacity. Real deployments must over-provision or accept lower occupancy.

**Semi-sorting optimization is complex.** The paper describes a "semi-sorted bucket" variant that reduces fingerprint size by 1 bit by sorting the 4 fingerprints within each bucket and storing only the differences. This recovers some space but complicates the implementation.

## Why it works

The deeper principle: **XOR is the simplest self-inverse bijection**.

XOR has the property `a ⊕ b ⊕ b = a` for any bits. This makes it attractive whenever you need a mapping `f: (position, tag) → alternate_position` where `f(f(pos, tag), tag) = pos`. There is no simpler function with this round-trip property.

The same self-inverse trick appears in:
- **XOR-linked lists**: each node stores `prev_addr XOR next_addr`. Given one neighbor's address, you can compute the other — a doubly-linked list using single-pointer storage.
- **RAID-5 parity**: the parity block is the XOR of all data blocks. Losing any one block recovers it as the XOR of all others (same self-inverse structure).
- **Diffie-Hellman key exchange**: not XOR, but the same *algebraic self-inverse in a group* idea — you can compute the shared secret from either direction without a private key.

At a higher level, the cuckoo filter is an example of **structural separation between identity and location**. A Bloom filter conflates the two: a bit position *is* part of an item's identity. A cuckoo filter separates them: a fingerprint encodes identity, bucket position encodes location, and the XOR formula relates one to the other. Because location is not baked into the identity representation, identity can be physically moved — which is exactly what makes deletion and eviction possible.

This separation principle recurs broadly:
- inode tables: file identity (inode number) is separate from file location on disk → files can be moved without changing their identity
- Consistent hashing: a node's identity (its hash ring position) is separate from the data it currently holds → rebalancing moves data without breaking the identity scheme
- The Bw-Tree (2026-09-01): introducing a logical page ID (LPID) separates page identity from physical memory address → structural modifications reduce to a single CAS on the mapping table

## Going deeper

1. **Cuckoo Hashing (Pagh & Rodler, ESA 2001)**: the parent technique. Explains the two-choice random graph analysis that shows cuckoo hashing achieves O(1) expected worst-case lookup — an improvement over open addressing that's not obvious until you model it as random 2-coloring.

2. **Vacuum Filters (Wang et al., VLDB 2020)**: a 2020 improvement that eliminates insertion failures while maintaining space efficiency. Uses a more complex "fingerprint migration" scheme to handle full tables gracefully — a good read if you need a production-safe alternative.

3. **Power of Two Choices (Azar et al., 1999)**: the underlying theoretical result explaining why cuckoo hashing's two candidate buckets reduces maximum load from Θ(log n / log log n) to Θ(log log n). The 2026-07-27 post in this archive covers this paper; the cuckoo filter's high achievable load factor (95%) is a direct consequence.
