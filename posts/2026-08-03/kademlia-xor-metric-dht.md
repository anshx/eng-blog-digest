---
title: "Kademlia: A Peer-to-peer Information System Based on the XOR Metric"
source: https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf
author: Petar Maymounkov, David Mazières
company: MIT / New York University
date_posted: 2002-03-07
date_digested: 2026-08-03
---

# Kademlia: A Peer-to-peer Information System Based on the XOR Metric

## What's new to learn

1. **XOR as a routing metric**: Defining distance as `d(x, y) = x XOR y` turns a flat 160-bit ID space into a binary trie — and routing in that trie is exactly binary search on the network. The key property isn't that XOR is fast to compute; it's that XOR is *unidirectional*: for any node `x` and distance `d`, there is exactly one `y` such that `d(x, y) = d`. This means all lookups for the same key follow the same tree path, enabling on-path caching.
2. **k-bucket routing tables**: Instead of a fixed, manually-constructed finger table (like Chord), each Kademlia node maintains 160 lists of contacts, one per XOR-distance "prefix bucket." The table self-populates from ordinary traffic and automatically biases toward known-stable nodes via a "prefer old contacts" eviction policy.
3. **Iterative parallel lookup**: The `FIND_NODE` algorithm runs α (typically 3) queries concurrently, converging to the k closest nodes in O(log n) rounds. The combination of parallelism and guaranteed XOR-distance decrease per round makes this robust to mid-lookup node failures.

## Prerequisites

- **Distributed hash tables**: what they are, why P2P networks need tracker-free key-value lookup.
- **Consistent hashing**: keys are hashed to IDs and stored at the nearest node — familiar from Dynamo. Kademlia uses the same idea with a different definition of "nearest."
- **Binary tries / Patricia trees**: tree where branching is done by key bits. k-buckets are a distributed realization of a binary trie.
- **Chord (helpful, not required)**: the predecessor DHT using a ring topology and finger tables. Kademlia's design is best understood as a response to Chord's weaknesses.

Non-obvious prerequisite: the difference between a *metric* (satisfies identity, symmetry, triangle inequality) and a *routing metric* (additionally allows a node to make locally-optimal forwarding decisions). XOR satisfies both; ring distance does not (see "Why it works").

## The core idea

Every node and every key in Kademlia is a 160-bit integer — typically a SHA-1 hash. The "distance" between two IDs is their bitwise XOR, read as an unsigned integer. This is not geographic distance, physical latency, or hop count. It is a mathematical distance over the ID space.

Why XOR? Because the 160-bit binary trie gives you a natural organizing structure. Two IDs that share their top k bits are in the same subtree at depth k. XOR of those two IDs has its top k bits equal to zero, so `d(x, y) < 2^(160−k)` exactly when x and y share a k-bit prefix. Closeness in XOR = depth of shared ancestry in the trie.

Now imagine placing the n nodes of the network as leaves of this (virtual) trie. Each node knows, for each of the 160 levels, a few contacts whose IDs live in the sibling subtree at that level. This is the k-bucket table. Routing is: pick the contact whose ID most closely matches the target's bit prefix, ask them who they know closer to the target, repeat. You descend one level per round — O(log n) rounds total.

The result: a fully decentralized key-value store with O(log n) lookup, no central coordinator, and automatic self-healing when nodes join or leave.

## Mechanics

### XOR distance and the binary trie

Given 160-bit IDs, define:

```
d(x, y) = x XOR y    (interpreted as an unsigned integer)
```

This is a valid metric (identity, symmetry, triangle inequality all hold). Its critical extra property:

> **Unidirectionality**: For any `x` and integer `d > 0`, there is exactly one `y` such that `d(x, y) = d`. (Because `y = x XOR d` is unique.)

In Chord's ring topology, there are *many* nodes at distance d from node x (any node within the arc). In XOR-space, there's exactly one ID at each distance. This means every lookup for key k converges to the same sequence of subtrees in the same order, regardless of which node initiates it — enabling deterministic on-path caching.

The trie interpretation: bit k of `XOR(x, y)` is 1 if and only if x and y diverge at level k of the binary trie. The most significant differing bit determines the trie depth at which x and y "first branch apart." When routing toward target t, each hop descends toward the subtree containing t.

```
160-bit ID space as binary trie (abbreviated to 4 bits for illustration):

Level 0:          [root]
               /         \
Level 1:    [0xxx]       [1xxx]
           /     \       /     \
Level 2: [00xx] [01xx] [10xx] [11xx]
           ...     ...    ...    ...

Node with ID 1010 maintains contacts in:
  bucket 0: nodes with prefix 0xxx   (top bit differs)
  bucket 1: nodes with prefix 11xx   (shares bit 0, differs on bit 1)
  bucket 2: nodes with prefix 100x   (shares bits 0-1, differs on bit 2)
  bucket 3: nodes with prefix 1011   (shares bits 0-2, differs on bit 3)
```

### k-bucket routing table

Node n maintains 160 k-buckets. k-bucket j holds contacts `c` such that:
```
2^j  ≤  d(n.id, c.id)  <  2^(j+1)
```
That is, contacts whose XOR with n starts with 0s in all positions above j and has a 1 in position j.

Each bucket holds at most `k` contacts (k = 20 in production BitTorrent/IPFS). When bucket j is full and a new contact `c` arrives:
1. **Ping** the least-recently-seen contact in the bucket.
2. If it **responds**: keep it, discard `c`. (Old contacts are preferred.)
3. If it **doesn't respond**: evict it, insert `c`.

This "least-recently-seen, evict only if unresponsive" policy is empirically motivated: P2P uptime studies show that the longer a node has been online, the longer it is likely to stay online. The policy biases routing tables toward reliable contacts, making the network stable under churn.

A node's routing table self-populates: every incoming message carries the sender's `(nodeID, IP, port)`, which is inserted into the appropriate bucket. No explicit join-time "finger table construction" is needed.

### The four RPCs

```
PING(node)                    → alive/dead
STORE(key, value)             → ACK
FIND_NODE(target_id)          → list of k closest known nodes to target_id
FIND_VALUE(key)               → value (if stored) OR list of k closest nodes
```

All four RPCs serve double duty: they perform their task *and* update the sender's k-bucket for the recipient.

### Lookup: FIND_NODE

To find the k nodes closest to target `t`:

```
1. shortlist ← k closest nodes from own routing table to t
   queried   ← {}

2. loop:
   candidates ← (shortlist \ queried), sorted by d(id, t)
   if top-k of candidates == top-k of shortlist at last iteration:
       break   # converged — no closer nodes found
   
   pick α closest candidates not yet queried
   for each c in parallel:
       result ← FIND_NODE(t) sent to c
       add result contacts to shortlist (deduplicated)
       queried ← queried ∪ {c}

3. return k closest in shortlist that have been queried
```

Parameters: k = 20 (replication), α = 3 (parallelism).

**Convergence argument**: At each round, every queried node `c` returns its k closest contacts to `t`. Because `c` was selected as the closest un-queried node, its k closest to `t` cannot be farther from `t` than `c`. Each round therefore either discovers a strictly closer node (advancing toward `t` in the trie) or fails to do so — at which point the loop terminates. With n nodes, depth of the trie is log₂(n), so convergence takes O(log n) rounds.

### Value storage: STORE and FIND_VALUE

To store key `k` with value `v`:
1. FIND_NODE(k) to get the k closest nodes to the key hash.
2. STORE(k, v) to all k of them.

To retrieve:
1. Issue FIND_VALUE(k) instead of FIND_NODE.
2. Any node holding the value returns it directly.
3. Nodes storing cached values along the return path reduce future lookup latency.

Values are **republished every 24 hours** by the storing node and **republished by recipients on join**. If the k storing nodes all leave within 24h, the value becomes unavailable.

### Node join

Joining node `j`:
1. Knows at least one bootstrap contact `b`.
2. Inserts `b` into appropriate k-bucket.
3. Runs FIND_NODE(j.id) — a self-lookup.
4. This populates j's routing table (responses carry contacts).
5. Refreshes all k-buckets that haven't been touched in the last hour (by looking up a random ID in each bucket's range).

## Where it breaks

**Sybil attacks**: An attacker who controls enough nodes can generate node IDs clustered around a target key, becoming the "k closest" nodes to that key and thus controlling what value is returned. The k-bucket eviction policy partially mitigates this (favoring long-lived nodes), but a patient Sybil operator with long-lived nodes wins. Production DHTs (S/Kademlia, IPFS) add cryptographic node-ID generation constraints to raise the cost of targeted Sybil placement.

**Eclipse attacks**: A well-positioned attacker fills a victim's k-buckets with attacker nodes, cutting the victim off from the honest network. This requires controlling enough routing-table entries before the victim's buckets fill with legitimate contacts.

**Data loss under churn**: If all k storing nodes for a key go offline between republication windows (24h), the data is lost. The replication factor k = 20 makes this unlikely but not impossible during large concurrent outages or coordinated attacks.

**Latency blindness**: XOR distance has no correlation with geographic or network latency. A node that is XOR-close could be on the other side of the planet. Production systems (BitTorrent uTP, IPFS) add latency-aware contact selection heuristics on top of the basic Kademlia routing, but this is outside the original design.

**Bootstrap dependence**: A new node needs at least one working bootstrap contact. Centralized bootstrap lists are common in practice (BitTorrent's DHT uses a few well-known bootstrap servers), reintroducing a mild centralization point.

## Why it works

The deep principle: **Kademlia is binary search over a distributed binary trie**.

In a regular binary trie, searching for key `t` means: at each internal node, inspect bit k of `t` to decide left or right, descend to the child. In Kademlia, "descend to the child" means "query the closest known node in the sibling subtree at this level." The routing table (k-buckets) is the trie's node cache — you maintain k contacts per trie level so that no level is empty.

The **unidirectionality** of XOR is what makes this possible without backtracking. Contrast with Chord's ring metric: many nodes lie on the "clockwise arc" to a target, so routing requires a finger-table lookup to skip over them. In Kademlia, there is exactly one XOR distance between any two IDs, so picking the "closest" contact is an unambiguous, monotone operation — each step moves strictly to a deeper subtree.

The **k-bucket eviction policy** is an instance of a general principle: "prefer contacts whose reliability is empirically demonstrated." In contrast to Chord's static finger table (which may contain dead nodes), Kademlia's table is continuously refreshed from live traffic and pruned by evicting non-responsive nodes. The long-tail uptime distribution means nodes that survive the churn are genuinely reliable.

**The "X is an instance of Y" insight**: Kademlia is consistent hashing (map keys to node IDs, store at nearest nodes) where "nearest" is measured in a trie metric rather than a ring metric. The trie metric's XOR structure makes routing binary search on bit prefixes — the same pattern as a radix tree, a Patricia trie, or a range-partitioned B-tree, just distributed across nodes.

**Connection to other systems in this archive**:
- **Dynamo** uses consistent hashing on a ring with Merkle tree anti-entropy; Kademlia replaces the ring with a trie and bakes anti-entropy into ordinary lookups.
- **Maglev** precomputes a lookup table for O(1) consistent hashing; Kademlia does the opposite — lazy, on-demand trie traversal for max distribution.
- **Adaptive Radix Trees (ART)** and **Roaring Bitmaps** use the same "different node types for different densities" as Kademlia's 160-bucket table: fine-grained knowledge of the nearby subtree, coarse knowledge of far subtrees.
- **Skip lists** share the "multiple levels of shortcuts indexed by bit depth" theme — Kademlia k-buckets are skip list lanes but distributed across machines.

**Real-world deployments**:
- **BitTorrent Mainline DHT** (2005–present): ~100–200M active nodes at peak, used for tracker-less torrent lookup.
- **IPFS / libp2p**: uses `kad-dht`, a close variant, for content routing across the distributed web.
- **Ethereum devp2p** (discv4/discv5): modified Kademlia for peer discovery across the Ethereum node network.
- **Kad network** (eMule): one of the first large-scale Kademlia deployments, ~5M nodes.

## Going deeper

1. **Chord: A Scalable Peer-to-Peer Lookup Service for Internet Applications** — Stoica et al., SIGCOMM 2001 (https://dl.acm.org/doi/10.1145/383059.383071). The predecessor DHT using an O(log n) finger table on a ring. Reading Chord alongside Kademlia shows concretely how symmetric XOR distance simplifies routing and why ring-distance asymmetry complicates finger-table maintenance.

2. **S/Kademlia: A Practicable Approach Towards Secure Key-Based Routing** — Baumgart & Mies, PDCS 2007. Adds two Sybil-resistance mechanisms: cryptographic node-ID generation (SHA(publicKey) as node ID) and parallel disjoint lookup paths to prevent Eclipse attacks. Directly addresses the security limitations of the original design.

3. **libp2p kad-dht specification** (https://github.com/libp2p/specs/tree/master/kad-dht). The production spec that IPFS runs. Shows how real systems deviate from the 2002 paper: provider records, disjoint query paths, latency-aware peer selection, and record freshness validation.
