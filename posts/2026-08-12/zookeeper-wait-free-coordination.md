---
title: "ZooKeeper: Wait-free Coordination for Internet-Scale Systems"
source: https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf
author: Patrick Hunt, Mahadev Konar, Flavio P. Junqueira, Benjamin Reed
company: Yahoo!
date_posted: 2010-06-23
date_digested: 2026-08-12
---

# ZooKeeper: Wait-free Coordination for Internet-Scale Systems

## What's new to learn

- **Wait-free reads**: In ZooKeeper, reads are served from the local replica without any coordination — no consensus round, no leader involvement. This single decision causes read throughput to scale linearly with cluster size, decoupling read capacity from write capacity entirely.
- **Coordination kernel**: Rather than shipping built-in primitives (locks, queues, leader election), ZooKeeper exposes three minimal composable operations — ordered writes, local reads, watches — from which any coordination primitive can be built. This is the "coordination Unix": provide a small, general-purpose kernel, let users build abstractions on top.
- **Ephemeral sequential znodes as fence tokens**: Combining an ephemeral node (dies when the client session ends) with a sequential suffix (monotonically increasing number) creates a primitive that is simultaneously a fencing token, a distributed lease, and an ordered claim ticket — the single building block underneath ZooKeeper's lock, queue, barrier, and leader election recipes.

## Prerequisites

- **Consensus basics**: Know the rough shape of Paxos or Raft — proposals, quorums, log ordering. ZooKeeper's write path (ZAB protocol) is structurally similar; you don't need to know ZAB's specifics beforehand, but the vocabulary helps.
- **Sessions and leases**: Understand that a session is a client-server contract with a time-to-live: if the client doesn't heartbeat within the TTL, the server considers it dead. This is how ZooKeeper auto-expires ephemeral nodes.
- **Linearizability vs. sequential consistency**: Know that linearizability (operations appear as if instantaneous) is strictly stronger than sequential consistency (operations appear in a consistent order, but timing is not constrained). ZooKeeper sits between the two, intentionally.

## The core idea

ZooKeeper is a replicated service that maintains a hierarchical namespace of small data items called znodes — much like a filesystem tree, but purpose-built for coordination state, not file storage.

The central design decision, and the one that makes everything else work, is a **deliberate asymmetry in how reads and writes are handled**:

- **Writes are linearizable**: every write goes through a single leader, is broadcast to followers via ZAB (ZooKeeper Atomic Broadcast), and is committed only after a quorum of nodes acknowledge it. All servers observe writes in the same total order.
- **Reads are wait-free**: any server can serve a read immediately from its local in-memory snapshot, without contacting the leader or waiting for any quorum. A read returns the most recent committed state that *that server* knows about — which may lag behind the global latest by one ZAB round.

This asymmetry is not an accident or a limitation; it is the performance architecture. In practice, coordination workloads are read-dominated: services poll leader existence, watch for config changes, check membership. Writes (leader changes, config updates) are rare. Serving all reads from local state makes read throughput scale linearly with the number of servers, while write throughput is bounded by the leader (which is fine, because writes are rare).

The paper's tag line is apt: ZooKeeper is a **coordination kernel**, not a coordination suite. It provides exactly the primitives needed to compose any higher-level distributed coordination primitive, but ships none of those primitives itself.

## Mechanics

### Data model: znodes

The namespace is a tree of znodes, each identified by a POSIX-style path:

```
/
├── /config
│   └── /config/db-host
├── /election
│   └── /election/candidate-0000000042
└── /workers
    ├── /workers/worker-01
    └── /workers/worker-02
```

Each znode stores up to ~1 MB of data (coordination metadata — IP addresses, config values — not actual application data) plus version and timestamp metadata.

Znodes come in four types, by combining two boolean dimensions:

| | Persistent | Ephemeral |
|---|---|---|
| **Normal** | Survives until explicitly deleted | Dies when the client session expires or disconnects |
| **Sequential** | Gets a unique monotonically increasing 10-digit suffix appended by the server | Same, plus ephemeral semantics |

The sequential suffix is server-assigned and collision-free across concurrent creates — this is what makes it useful as a fencing token and ordered claim ticket.

### API

The API is deliberately small:

```
create(path, data, flags)          → path       # flags: EPHEMERAL, SEQUENTIAL
delete(path, version)
exists(path, watch?)               → stat | false
getData(path, watch?)              → data, stat
setData(path, data, version)       → stat
getChildren(path, watch?)          → [child_names]
sync(path)                         # forces client to see leader's latest state
```

The `watch` parameter on read operations registers a **one-shot watch**: the server will notify the client exactly once when the znode is created, deleted, or modified. After firing, the watch is gone; the client must re-register if it wants further notifications.

The `version` parameter on mutating operations is an optimistic concurrency check: `delete(path, version=-1)` deletes unconditionally; `delete(path, version=3)` atomically succeeds only if the current version is 3. This is MVCC at the znode level.

### Write path: ZAB

Every write goes through the leader via ZAB (ZooKeeper Atomic Broadcast), which is structurally similar to Raft:

1. **Leader election**: Servers negotiate who is leader based on who has the highest ZXID (ZooKeeper Transaction ID — an epoch + offset pair). The server with the most up-to-date log wins.
2. **Broadcast phase**: The leader assigns each write a ZXID, broadcasts the proposal to all followers, and commits once it hears from a quorum. Followers apply in ZXID order.
3. **FIFO client ordering**: Within a single client session, ZAB guarantees that operations are processed in submission order. If client A does `create("/lock")` followed by `setData("/config", ...)`, both are applied in that order across all replicas.

ZXIDs encode the leader epoch in the high 32 bits and the transaction count in the low 32 bits: `ZXID = (epoch << 32) | counter`. This means ZXIDs from different epochs are globally totally ordered — the same insight Raft uses with term + log index — and you can detect stale leaders by checking if their ZXID epoch is behind.

### Read path: local, wait-free

A read (`exists`, `getData`, `getChildren`) is processed entirely by the server receiving the request. No ZAB round, no leader involvement. The server reads its local in-memory data structures and responds immediately.

Consequence: a client might see slightly stale data. If writer W commits `/config/db-host = "10.0.0.5"` and reader R immediately reads from a follower that hasn't yet received the ZAB broadcast, R sees the old value.

ZooKeeper provides two guarantees that bound the staleness:

- **Read your own writes**: if a client writes and then reads within the same session, the read is served from the same server (or the server waits until it has caught up to at least the write's ZXID before responding). A client never sees its own writes disappear.
- **Monotonic reads**: within a session, reads observe a non-decreasing snapshot of the server's state. You will never see a znode's version go *backward* across reads in the same session.

If an application genuinely needs linearizable reads (sees the absolute latest globally committed state), it calls `sync(path)` before reading. `sync()` forces the server to catch up to the leader's latest committed ZXID before processing the next read. It is an escape hatch for applications that truly need strong read consistency, paid for with a network round-trip.

### Watch semantics: one-shot, push, level-triggered

Watches are the notification layer. When you call `exists("/election/leader", watch=true)` and the node later changes, your client receives a WatchEvent notification.

Three design choices are worth noting:

1. **One-shot**: Watches fire exactly once. This prevents a thundering herd: if 1,000 clients are watching a node and it changes, they each get exactly one notification. They don't continuously fire — clients must consciously re-register. This also prevents event storms from a single flapping node.

2. **Level-triggered, not edge-triggered**: A WatchEvent says *what changed* (NodeCreated, NodeDeleted, NodeDataChanged, NodeChildrenChanged) but not *what it changed to*. Clients must re-read the znode to learn the new state. This is intentional: it prevents clients from acting on stale state embedded in the notification itself.

3. **No missed events within a session**: ZooKeeper guarantees that if the event happens after the watch is registered and before it fires, the client will be notified. However, there is a window between when a watch fires and when the client re-registers it during which a second change could occur — applications must be written to handle this correctly (re-read and check state, not just act on the notification).

### Building coordination primitives

The power of ZooKeeper's kernel is demonstrated by these standard recipes:

**Leader election (thundering-herd-safe):**
1. Each candidate creates an ephemeral sequential node: `/election/n_` → `/election/n_0000000042`.
2. Each candidate calls `getChildren("/election")`, sorts them, and checks if its own node is the smallest. If yes: I am leader.
3. If not: watch the predecessor node (not the current leader). When the predecessor dies, re-check. Only one client is notified per leader change — O(n) total notifications, not O(n²).

```
# Naive (wrong): all clients watch /election/leader → O(n²) notifications on change
# Correct: each candidate watches only its predecessor → O(n) total
```

**Distributed lock:**
1. `create("/lock/lock-", EPHEMERAL|SEQUENTIAL)` → get back `/lock/lock-0000000007`.
2. `getChildren("/lock")`, sort; if my node is first, I hold the lock.
3. If not, watch the predecessor. When it disappears, re-check.
4. Releasing the lock: `delete("/lock/lock-0000000007")`. Because it's ephemeral, it also auto-releases if the client crashes.

**Barrier:**
1. Coordinator creates `/barrier`.
2. Each worker creates `/barrier/worker-N` (persistent).
3. All workers watch `/barrier` for NodeDeleted.
4. When all N workers have joined (`getChildren("/barrier").length == N`), coordinator deletes `/barrier`. Workers wake up and proceed.

**Distributed queue (FIFO):**
1. Each producer creates `/queue/item-` (sequential) → `/queue/item-0000000001`, etc.
2. Consumer calls `getChildren("/queue")`, takes the smallest sequence number, processes it, and deletes it.

All four primitives emerge from exactly three properties: ordered writes (ZAB), local reads (wait-free), and watches (one-shot notification).

### Architecture

The cluster runs in a **star topology**: one leader, multiple followers. Typically 3, 5, or 7 nodes (odd number for quorum). All writes go to the leader; any server can serve reads.

Each server keeps the entire namespace in **memory** (znodes + data). This is what enables O(1) reads — no disk I/O on the read path. The on-disk component is a write-ahead log (for ZAB durability) and periodic snapshots.

At Yahoo production scale (time of the paper): ~10,000 concurrent connections, ~2,000 operations/second across the cluster. Read throughput scales linearly with ensemble size; write throughput is bound by the leader (roughly 10,000–15,000 writes/second in practice). A 7-server ensemble can serve ~500,000 reads/second at around 1–2 ms median latency.

## Where it breaks

**Stale reads without sync()**: Applications that read immediately after a write on a *different* client may see old data. Most ZooKeeper recipes are designed to handle this (re-read after watch fires), but engineers who assume ZooKeeper is fully linearizable will write incorrect code.

**Write throughput doesn't scale**: All writes funnel through one leader. A 7-node cluster doesn't give you 7× the write throughput of a 1-node cluster; it gives you slightly *less* (replication overhead). If you need high write throughput, ZooKeeper is the wrong tool. This is explicitly by design: coordination workloads are read-heavy, and ZooKeeper is optimized for that profile.

**Data size limit**: Znodes hold at most ~1 MB. ZooKeeper is for *metadata* — configuration, membership, lock state — not for storing actual application data. A common mistake is putting data that belongs in a proper database into ZooKeeper because "it's convenient."

**Watch races**: Between a watch firing and the client re-registering a new watch, another event may occur and go unnoticed. Well-written ZooKeeper clients always re-read state after registering a watch (not just after a watch fires), to handle this window. This is easy to get wrong.

**Thundering herd from naive recipes**: Watching the current leader's node (rather than your predecessor) causes all N-1 clients to be notified and make requests when the leader changes. For large ensembles, this creates a spike in server load. The predecessor-watch pattern described above is the standard fix, but it's non-obvious.

**Latency tail**: Because writes go through a full ZAB round (leader + quorum acknowledgment), write tail latency is sensitive to follower lag and network jitter. With a 3-node ensemble spanning multiple availability zones, a slow follower can push write p99 into tens of milliseconds.

**Split-brain on network partition (mostly safe but nuanced)**: ZooKeeper correctly refuses writes in a minority partition (the leader can't get a quorum), but reads in the minority partition continue to be served — from potentially stale state. An application that uses reads to check lock ownership may incorrectly conclude it holds a lock if it's in a minority partition that still serves old data.

## Why it works

The deep reason ZooKeeper's asymmetric consistency model is correct for coordination workloads comes from what coordination *actually is*. Coordination is fundamentally about **ordering claims on a shared resource** — who is the leader, who holds the lock, which configuration version is current. That ordering only requires that writes are totally ordered across all observers. Reads don't need to be at the same instantaneous cut; they need to be monotonic and to reflect your own writes.

This is the same reasoning behind:

- **MVCC in databases**: readers take a snapshot, don't block writers. The snapshot may lag slightly; that's acceptable because a query's self-consistency (reading from the same snapshot throughout) matters more than seeing the absolute latest row.
- **RCU in the Linux kernel** (covered in the July 14 post): readers access data without any lock; updaters defer reclamation until all readers in the current epoch have drained. Readers are fast because they don't coordinate; the invariant that matters is that no reader sees a freed pointer, not that it sees the absolute latest value.
- **The CALM theorem** (covered in the July 28 post): a monotone computation (one where new information only adds, never reverts) never needs coordination. ZooKeeper's read model exploits this: if you're checking "does the leader node exist?", a slightly stale "yes" is wrong but a stale "no" is safe (you'll re-check after the watch fires). Coordination recipes are designed to be monotone-safe.

The "X is just Y" insight here: **ZooKeeper is MVCC applied to distributed metadata**. Each znode has a version number (the `czxid`, `mzxid`, `pzxid` fields). Reads take a local snapshot at a consistent version; writes go through the leader and bump the version. "Wait-free reads" is just "MVCC snapshot reads" at the granularity of a distributed service rather than a single-node database. The consistency level ZooKeeper provides is exactly what databases call *Read Committed* — you always see committed writes, but not necessarily the globally latest committed write.

The coordination kernel idea connects to a broader systems principle: **the best abstractions are those that compose well with each other, not those that ship with complete solutions**. Unix pipes compose better than GUI applications. HTTP verbs + resources compose better than custom RPC schemas. Ephemeral + sequential znodes + watches compose into any coordination primitive. The more you build into the kernel, the more you constrain how users can assemble it into what they actually need.

A final systems insight: **ephemeral nodes are distributed leases made structural**. A lease is a contract: "I may act as leader until time T; if I fail to renew before T, my lease expires and I am no longer valid." In ZooKeeper, the lease period is the session timeout (typically 5–30 seconds), the renewal is the client heartbeat, and the "lease expired" event is the server deleting all ephemeral nodes for that session. Crash recovery is automatic: whatever state the crashed process owned — its lock, its leader claim, its worker slot — disappears without any external process or timeout manager. The session IS the lease.

## Going deeper

1. **"The Chubby Lock Service for Loosely-Coupled Distributed Systems"** (Mike Burrows, Google, OSDI 2006) — the predecessor at Google that ZooKeeper was designed to improve upon. Chubby takes a different approach (linearizable reads through Paxos, coarse-grained locks as a first-class primitive). The comparison between the two clarifies exactly which design choices ZooKeeper made and why: https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/

2. **etcd documentation — why etcd replaced ZooKeeper for Kubernetes** — etcd uses Raft instead of ZAB, exposes a key-value API instead of a filesystem namespace, and offers optional linearizable reads (via `ReadIndex` from Raft). Understanding the differences teaches which parts of ZooKeeper's design were essential and which were accidental complexity: https://etcd.io/docs/v3.5/learning/design-learner/

3. **ZooKeeper Recipes and Solutions (Apache docs)** — the official collection of coordination recipes, including the thundering-herd-safe leader election and the double-barrier. Reading the code-level description of how each recipe handles the watch-race window teaches the subtlety of building on a wait-free read model: https://zookeeper.apache.org/doc/current/recipes.html
