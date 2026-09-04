---
title: "How we saved 100 terabytes of memory by optimizing 1.1.1.1's DNS cache"
source: https://blog.cloudflare.com/dns-cache-memory-optimization-1111/
author: Sebastiaan Neuteboom
company: Cloudflare
date_posted: 2026-08-27
date_digested: 2026-09-04
---

# How we saved 100 terabytes of memory by optimizing 1.1.1.1's DNS cache

## What's new to learn

1. **Type–invariant mismatch is a hidden tax**: Every Rust collection type (Vec, String, HashMap) encodes an assumption about future behavior — it will grow. When that assumption is false, you pay the assumption's cost in space with zero benefit. Replacing Vec<T> with Box<[T]> is not a micro-optimization; at 250 billion entries, removing one 8-byte field saves 2 TB.

2. **Enum sizing is the largest variant rule**: A Rust `enum` is always sized to hold its largest variant. One bulky rare type (NAPTR, SVCB) inflates the memory footprint of every instance of every common type (A, AAAA). Boxing the outlier is the idiomatic fix and can collapse per-entry size by 50%+.

3. **Wire format as lazy encoding**: Storing raw, length-prefixed wire bytes instead of parsed Rust structs defers parse cost to read time, eliminates heap allocations per field, and often shrinks the in-memory footprint below the struct's lower bound — the same trade-off as columnar storage deferring decoding to projection time.

## Prerequisites

- **Rust fat pointers**: `Box<[T]>` and `&[T]` are *fat pointers* — two words (pointer + length). `Vec<T>` is *three* words (pointer + length + capacity). Knowing this at the bit level is essential.
- **DNS record structure**: A DNS response has four sections — Question, Answer, Authority, Additional — each containing Resource Records (RR). Each RR has an owner name (the domain it describes), a type (A, AAAA, MX, TXT, …), class, TTL, and RDATA payload.
- **DNS wire format**: On the wire, domain names are encoded as length-prefixed labels (`\x03www\x07example\x03com\x00`) with pointer compression. An A record's RDATA is literally 4 bytes. An AAAA is 16 bytes.
- **Rust enum layout**: Rust enums are tagged unions: size = size of the largest payload + discriminant tag (+ alignment padding).

## The core idea

Big Pineapple, Cloudflare's DNS caching daemon behind 1.1.1.1, held roughly 250 billion entries at any given moment in 2026. At that scale, **one wasted byte per entry costs 250 GB of RAM**. The post describes five successive structural changes to a Rust `CacheEntry` type — no algorithm changes, no data structure changes at the macro level — that cut per-entry size from **953 bytes to 420 bytes (56% reduction)** and freed **~100 terabytes of RAM** fleet-wide, equivalent to 130 Gen 13 servers.

The central lesson: **data layout is the primary lever before algorithms**. You cannot cache-optimize or SIMD your way out of 533 bytes per entry × 250B entries ≈ 133 TB. But you can redesign the type.

## Mechanics

### Starting point: what 953 bytes looked like

A DNS cache entry must store the full response so it can be replayed without a resolver round-trip. The naïve approach maps the DNS response structure directly to Rust types:

```rust
// Conceptual before (simplified)
struct CacheEntry {
    owner:    String,           // 24 bytes (ptr + len + cap)
    records:  Vec<ResourceRecord>,  // 24 bytes header + heap
}

struct ResourceRecord {
    owner_name: String,         // 24 bytes — often identical to query!
    rtype:      RecordType,     // enum sized to largest variant
    ttl:        u32,
    rdata:      RData,          // enum sized to largest variant
}

enum RData {
    A(Ipv4Addr),                // 4 bytes
    Aaaa(Ipv6Addr),             // 16 bytes
    Mx(u16, String),            // 2 + 24 bytes
    Naptr(u16, u16, String, String, String, String), // ~150 bytes
    // ...
}
```

Every `Vec` and `String` wastes 8 bytes for a capacity field that will never change after the entry is inserted. Every `enum RData` is padded to the size of its largest variant (NAPTR at ~150 bytes), so an A record consumes 150 bytes for 4 bytes of actual data. Three separate `Vec<ResourceRecord>` lists (answer, authority, additional sections) each have their own 24-byte header.

### Optimization 1: Box<[T]> / Box<str> instead of Vec<T> / String

A `Vec<T>` is `(ptr: *mut T, len: usize, cap: usize)` — 24 bytes on 64-bit systems. A `Box<[T]>` is a fat pointer `(ptr: *mut T, len: usize)` — 16 bytes. Capacity is gone because boxed slices cannot grow.

```rust
// After opt 1
owner:   Box<str>,          // 16 bytes (was 24)
records: Box<[ResourceRecord]>, // 16 bytes (was 24)
```

DNS records are immutable once cached. Dropping the capacity field is semantically correct and free. **This single change saved >15 TB** across the fleet (millions of Vec/String fields each carrying a dead 8-byte word).

### Optimization 2: Merged record lists with offset indexing

Before: three separate `Box<[ResourceRecord]>` — answer, authority, additional — each a fat pointer (16 bytes) plus heap allocation header.

After: one flat `Box<[ResourceRecord]>` buffer for all sections, plus two `u16` offsets marking where authority starts and where additional starts.

```
[answer records | authority records | additional records]
  0                 ^offset_auth       ^offset_add
```

Two `u16`s (4 bytes total) replace two redundant 16-byte fat pointers. Net saving: ~28 bytes per entry, plus reduced allocator overhead from consolidating three heap blocks into one.

### Optimization 3: Owner name deduplication

Every Resource Record has an `owner_name` field — the domain name the record describes. In the vast majority of cached responses, all records in the answer section have the same owner as the query name (you asked for `example.com A` and you get `example.com A` records back).

The fix: store the owner name only when it differs from the cache key (the query name). Otherwise, reconstruct it from the key at read time.

This eliminates a `Box<str>` (16 bytes + heap allocation) per record in the common case. For a response with 3 answer records, that's 48 bytes + 3 heap allocations gone.

### Optimization 4: Boxing large enum variants

Rust enum layout rule: `size_of::<RData>() == size_of::<LargestVariant>() + discriminant`. If `NAPTR` takes 150 bytes, every `RData::A` also occupies ~152 bytes despite containing only 4 bytes of payload.

The fix: `Box<NaptrData>` inside the enum. Now:

```rust
enum RData {
    A(Ipv4Addr),           // 4 bytes payload
    Aaaa(Ipv6Addr),        // 16 bytes payload
    Mx(u16, Box<str>),     // 2 + 16 bytes
    Naptr(Box<NaptrData>), // 8 bytes (just a pointer)
    // ...
}
```

The enum shrinks to the size of the largest *common* type (AAAA at 16 bytes) instead of the largest *rare* type. A and AAAA records — the vast majority of cached entries — drop from ~152 bytes to ~20 bytes. NAPTR pays one extra indirection on the rare path it is actually used.

### Optimization 5: Raw wire format storage

Fully parsed Rust structs have per-field allocations. A `MX` record has `String` for the exchange domain; a `TXT` record has `Vec<Box<[u8]>>` for its character strings. Each introduces heap pointers, alignment padding, and allocator overhead.

The insight: **cached entries are write-once, read-almost-never** (most cache hits never fully inspect the RDATA; they just replay it to the client as bytes). So instead of storing rich parsed structs, store the raw DNS wire format bytes with a length prefix:

```rust
struct ResourceRecord {
    // owner stored separately (opt 3)
    rtype:    RecordType,   // u16
    ttl:      u32,
    rdata:    Box<[u8]>,    // raw wire bytes — always compact
}
```

For an A record, `rdata` is 4 bytes. For AAAA it is 16 bytes. For NAPTR it is exactly its wire encoding, no richer. Parse only at the moment when the application logic needs the structured fields — which is rare compared to cache read-through.

### Result

| Metric                   | Before      | After       | Change  |
|--------------------------|-------------|-------------|---------|
| Per-entry size           | 953 bytes   | 420 bytes   | −56%    |
| Insert throughput        | 625K/sec    | 893K/sec    | +43%    |
| Lookup latency           | 828 ns      | 670 ns      | −19%    |
| p99 memory per instance  | 9.3 GB      | 5.3 GB      | −43%    |
| Fleet memory freed       | —           | ~100 TB     | —       |

Throughput and latency both improved as a bonus: smaller entries mean more entries fit in CPU cache, improving locality for both inserts and lookups.

## Where it breaks

**Read-time parse cost**: Wire format storage shifts parse cost from write (insert) to read (lookup). If a workload queries RDATA structured fields on every cache hit, this is worse than the parsed approach. Big Pineapple's workload is overwhelmingly "replay to client as bytes" — the right shape. A different DNS server doing DNSSEC validation on every hit would need structured storage.

**Boxing introduces indirection**: `Box<NaptrData>` costs one extra pointer dereference. For NAPTR-heavy workloads (rare in practice), this could hurt cache locality. The 99% case wins; the 1% pays.

**Owner reconstruction cost**: Deduplicating owner names requires reconstructing the domain from the cache key on lookup. This is cheap (a memcpy) but is a correctness dependency: if the cache key representation ever changes, owner reconstruction must change too. Hidden coupling.

**Optimization surface area shrinks**: After these five changes, the entry is near the information-theoretic minimum for what must be stored. Future wins require changing what data is cached or changing the cache architecture (e.g., partial record storage, sharding by record type).

## Why it works

All five optimizations share one root cause: **type invariants were mismatched to usage invariants**.

- `Vec` encodes "I may need to grow." DNS cache entries never grow after insert. → Remove the capacity field.
- `enum` encodes "every variant should be O(1) to access uniformly." NAPTR is accessed in <1% of lookups. → Pay a pointer indirection for the rare case, not a size penalty for the common case.
- Parsed structs encode "fields will be accessed individually." DNS responses are mostly relayed as opaque bytes. → Pay parse cost only when fields are actually accessed.

The unifying principle is **Succinct Data Structures** applied at the application level: store exactly the information your actual query patterns require, encoded in the most compact form compatible with those queries. Succinct structures from theoretical CS do this formally (a succinct binary tree takes `2n + o(n)` bits vs. the pointer-based tree's `O(n log n)` bits). The Cloudflare engineers did it pragmatically: profile what operations actually happen, then design a type that is exactly right-sized for those operations and no more.

This is also Knuth's "Premature optimization" corollary seen in reverse: **delayed structural optimization is the real inefficiency**. The team did not optimize an algorithm; they corrected a structure that had been built with wrong assumptions.

At 250 billion entries, the multiplication effect makes every byte an engineering decision. The same principle applies to:
- **Database row formats**: Postgres's `HeapTuple` stores system columns per-row that are often redundant (same principle as owner name deduplication)
- **Columnar compression**: RLE and dictionary encoding exploit the assumption "many values repeat" — valid for analytics, wrong for random writes
- **Protocol Buffers encoding**: field presence in proto3 uses varint + field tags rather than fixed offsets, trading random-access speed for compactness — the same schema-on-read trade-off as wire format storage
- **memcached slab allocator**: pre-sizes allocation classes to avoid internal fragmentation — same "match allocation size to actual object size" reasoning

## Going deeper

1. **Ulrich Drepper, "What Every Programmer Should Know About Memory" (2007)** — sections 2 and 3 cover CPU cache hierarchy and DRAM timing; section 6 covers how data layout determines cache efficiency. Still the reference for why struct size × entry count = cache pressure.

2. **"Rust Performance Pitfalls" (Nnethercote, 2024)** and the `heaptrack` / `dhat` profiler documentation — tools for measuring per-type heap allocation counts and sizes in Rust, which is the empirical starting point for any layout optimization campaign.

3. **"Succinct Data Structures" (Jacobson 1989; Navarro & Mäkinen 2007 survey)** — the formal theory of representing n-bit structures using n + o(n) bits while supporting O(1) or O(log n) operations. Rank/select on bitvectors, Elias-Fano encoding, and the wavelet tree are the canonical results. The Cloudflare work is an informal instance of these ideas at the application layer.
