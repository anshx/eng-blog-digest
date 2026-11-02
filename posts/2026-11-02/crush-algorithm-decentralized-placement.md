---
title: "CRUSH: Controlled, Scalable, Decentralized Placement of Replicated Data"
source: https://www.ssrc.ucsc.edu/Papers/weil-sc06.pdf
author: Sage A. Weil, Scott A. Brandt, Ethan L. Miller, Carlos Maltzahn
company: UC Santa Cruz / Ceph
date_posted: November 2006
date_digested: 2026-11-02
---

# CRUSH: Controlled, Scalable, Decentralized Placement of Replicated Data

## What's new to learn

1. **Computed placement vs. stored placement** — instead of a lookup table mapping objects to devices, CRUSH derives placement via a pure function on a small cluster topology description; every client independently computes the same answer without coordination.
2. **Failure-domain-aware placement as a tree traversal** — physical topology (device → host → rack → row → datacenter) is encoded directly in the cluster hierarchy, so replica separation across failure domains is a structural consequence of the algorithm, not a post-hoc constraint.
3. **Exponential race / competitive hash selection (straw2)** — assigning each candidate a draw proportional to a weighted hash and picking the maximum is mathematically equivalent to sampling from Exponential distributions parameterized by weight; this makes data movement on cluster changes provably minimal (only affected items remapped).

## Prerequisites

- What consistent hashing is and why it exists (virtual nodes on a ring, O(1/n) per-node fraction)
- What replicated object storage means at a high level (objects placed on N OSDs for fault tolerance)
- Basic probability: expected value, independence; optional but helpful: exponential distribution minimum-of-k
- Understanding of why a central metadata server is a bottleneck (see also: GFS master, Zookeeper)

## The core idea

Most distributed storage systems solve the question "which devices hold object X?" by maintaining a table somewhere — a hash ring stored by a coordinator, a placement database, or a DHT. That table must be updated when the cluster changes, and every client must obtain the current state before it can issue a request.

CRUSH replaces the table with a function. Given three inputs — an object identifier `x`, a **cluster map** (a weighted tree describing your physical topology), and a **placement rule** (a policy like "3 replicas in 3 separate racks") — CRUSH deterministically outputs the set of devices that should hold `x`. The cluster map is small (kilobytes, not petabytes), easy to replicate cluster-wide, and versioned. Any client holding the current map can compute placement locally in microseconds with no network round-trips.

The algorithm works by recursively descending the topology tree. Starting at a root bucket, CRUSH applies placement rules that navigate from root toward leaves, at each level using a **bucket algorithm** to pseudo-randomly select children weighted by capacity. The "controlled" in the name means: the randomness is seeded by `(x, r, bucket_id)` so results are deterministic and reproducible by any node; "scalable" means no central coordinator; "decentralized" means no single point of failure.

## Mechanics

### The cluster map

The cluster map is a weighted directed acyclic tree. Leaf nodes are **devices** (OSDs) with weights proportional to capacity — a 10 TB drive typically weighs 10, a 2 TB drive weighs 2. Interior nodes are **buckets** named for their physical role: `host`, `rack`, `row`, `datacenter`, `root`. A bucket's weight is the sum of all children beneath it (so a rack's weight reflects its total usable storage). A 64-node cluster might look like:

```
root (weight 640)
├── datacenter-east (weight 320)
│   ├── rack-a (weight 80)
│   │   ├── host-1 (weight 20)
│   │   │   ├── osd.0 (weight 10)
│   │   │   └── osd.1 (weight 10)
│   │   └── host-2 ...
│   └── rack-b ...
└── datacenter-west ...
```

### Placement rules

A rule is a short program operating on the current selection set:

```
rule replicated_3way_rack {
  step take root
  step chooseleaf firstn 3 type rack
  step emit
}
```

- `take root` — initialises the working set to `{root}`
- `chooseleaf firstn 3 type rack` — select 3 **distinct** subtrees at the `rack` level, then from each subtree select one OSD leaf; "distinct" is the failure-domain guarantee: the three replicas land on three different racks
- `emit` — output the working set as the final replica list

The rule vocabulary handles heterogeneous policies: you can mix device classes (`ssd` for primary, `hdd` for secondary), use different replication factors per pool, or impose data-sovereignty constraints ("all replicas in `region-eu`").

### The CRUSH selection step in detail

The `select(n, type)` step recursively calls `c(x, r, bucket)` to pick children. For each attempt `r` from 1 to `n` (and retries for collisions or unavailable devices):

1. Navigate from the current bucket toward a leaf of the desired type
2. At each intermediate bucket, call the **bucket algorithm** to choose one child
3. If the chosen child is **out** (failed/removed) or would violate the failure domain, try the next `r`
4. Collect `n` successful outputs

The retry logic is bounded by a configurable `choose_total_tries` (default: 50), so pathological cases (very constrained topologies) don't spin forever.

### The four bucket algorithms

The same selection interface `c(x, r, bucket)` is implemented by four algorithms with different speed/movement tradeoffs:

**1. Uniform** — all children equal weight:
```
c(x, r, b) = (hash(x, r) + b.prime) mod n
```
O(1) lookup. The Achilles' heel: adding or removing *any* item reshuffles *all* mappings. Same failure as naive modular hashing.

**2. List** — items maintained in a list, newest prepended:
```
for item i from newest to oldest:
  if hash(x, r, i) < w_i / w_remaining:
    return i
  else: w_remaining -= w_i
```
When a new item is added to the front, exactly `w_new / w_total` of the existing mappings move to it — optimal for appends. But removing an item from the middle requires a full walk. O(n) lookup.

**3. Tree** — binary tree with per-node subtree weight sums:
At each binary split, go left with probability `w_left / (w_left + w_right)`. O(log n) both for lookup and for propagating weight changes up the tree.

**4. Straw / Straw2** — the recommended default:

```python
# Straw2 pseudocode
def choose(x, r, bucket):
    best_draw = -infinity
    best_item = None
    for item i in bucket.items:
        h = crush_hash(x, r, i.id)          # uniform in [0, 1)
        draw = log(h) / item.weight          # weight-scaled log-uniform
        if draw > best_draw:
            best_draw = draw
            best_item = i
    return best_item
```

Every item independently computes a "draw" from its own hash and weight. The item with the maximum draw wins.

**Why this is mathematically correct**: `log(U(0,1))` is distributed as `-Exp(1)`. Dividing by `w_i` gives `-Exp(w_i)`. The argument that maximises `-Exp(w_i)` is equivalently the argument that minimises `Exp(w_i)`, which has probability `w_i / Σw_j` — exactly the desired weighted probability. This is the **exponential race** or **random prioritization** technique from probability theory.

**Why this gives minimal remapping**: Each item's draw depends only on `(x, r, i.id)` and `w_i`. When item `k` is added or its weight changes, only the items whose draw would have beaten (or lost to) the new/changed `k` need to remapp. Items whose relative ordering among themselves hasn't changed keep their assignments. Straw2's update behaviour is provably optimal: *no more* data moves than the minimum required by the weight change.

### The full picture: an example

Object `0xCAFEBABE`, replicated 3× across racks, with cluster map above:

```
r=0: c(0xCAFEBABE, 0, root)
       → straw2 over {datacenter-east, datacenter-west}
       → picks datacenter-east  (w=320 vs 320, coin flip)
     c(0xCAFEBABE, 0, datacenter-east)
       → straw2 over {rack-a, rack-b}
       → picks rack-a
     c(0xCAFEBABE, 0, rack-a)
       → straw2 over {host-1, host-2}; picks host-1
       → straw2 over {osd.0, osd.1}; picks osd.1
     output: osd.1

r=1: repeat, constrained to a different rack → picks osd.7 in rack-b
r=2: picks osd.23 in datacenter-west
```

Final set: `{osd.1, osd.7, osd.23}`. Any Ceph client computing this gets the same answer. Zero coordination. Zero round-trips.

## Where it breaks

**Cluster-map consistency is still required.** CRUSH is a pure function of the map version. If two clients hold different map versions, they compute different replica sets for the same object. The Ceph monitor cluster uses Paxos to publish versioned maps; during a map update race, clients may read from stale locations and need to redirect. The function is stateless, but its *input* still needs to converge.

**Statistical non-uniformity at small scale.** With 3 OSDs and straw2's hash randomness, the variance in object counts can be 10–20% from ideal. The law of large numbers rescues you only when you have many objects per OSD. Ceph uses placement groups (PGs) as the unit of placement — each PG is an independent CRUSH input — to amortize this variance.

**Complex rules increase retry cost.** A rule requiring "3 replicas on 3 different racks, each on a different host, preferring NVMe devices" may require many retries before finding a valid placement if the cluster is small or partially failed. Worst case is O(depth × retries). In practice, `choose_total_tries=50` is enough for well-provisioned clusters.

**Rebalancing is minimal but not zero.** Adding a new host causes data to migrate from every existing host to the new one, in proportion to its weight. This is unavoidable for any correct replication scheme. CRUSH guarantees no *extra* movement, but "minimum necessary" is still O(1/n) of total data.

**Topology must reflect reality.** Two OSDs that CRUSH thinks are in different racks (because the operator put them in the same CRUSH bucket) but physically share a power feed will both fail together, violating the fault-domain guarantee. CRUSH cannot audit physical reality; the cluster map must be maintained accurately.

## Why it works

The deeper principle is: **encode structure in the algorithm's input shape, not in a runtime-maintained table.**

Consistent hashing solves "which node owns key X" with a stored ring of virtual nodes. The ring itself must be maintained — a distributed data structure requiring coordination. CRUSH replaces the ring with a hierarchy plus a bucket algorithm. The hierarchy is the right shape to model real physical topologies; the algorithm is simple enough that any client can run it; and the exponential race makes weights first-class without storing any per-item derived state.

This is the same insight underlying:
- **Rendezvous hashing (HRW)**: assign each candidate a score = `hash(key, candidate)`, pick the max. CRUSH is HRW applied recursively to a tree.
- **Maglev's consistent hash lookup table** (in this archive): precompute a static table offline so online dispatch is O(1). CRUSH goes the other direction — no table at all, compute on the fly.
- **Bloom filters / Count-Min Sketch**: replace per-element storage with a hash function and accept a bounded error. CRUSH accepts statistical non-uniformity for the benefit of no table.
- **Git's content-addressed objects**: given a blob's SHA, you derive its location in the object store without a lookup. CRUSH: given `object_id + cluster_map`, you derive its OSD set.

The general lesson: when your lookup problem has the property that the key + a small shared description fully determines the result, a pure function beats a table. Tables have synchronization cost; functions have compute cost. For placement — where the description (topology) changes rarely and the queries happen on every I/O — the function wins decisively.

## Going deeper

1. **Weil's PhD thesis**: "Ceph: Reliable, Scalable, and High-Performance Distributed Storage" (UC Santa Cruz, 2007) — explains how CRUSH fits into the full Ceph RADOS layer, including the monitor cluster, OSD recovery, and the RADOS object store. Available via UMI/ProQuest.
2. **Rendezvous hashing (HRW)**: Thaler & Ravishankar, "Using Name-Based Mappings to Increase Hit Rates" (1998) — the simpler flat version of the same exponential race idea; useful for understanding CRUSH at single-level before generalising to hierarchies.
3. **Ceph CRUSH map documentation** at `docs.ceph.com/en/latest/rados/operations/crush-map/` — the operational reference showing how to express real placement policies, device classes, weight sets, and the newer CRUSH tunables (`straw_calc_version`, `chooseleaf_stable`) that fix edge-case distribution bugs in the original algorithm.
