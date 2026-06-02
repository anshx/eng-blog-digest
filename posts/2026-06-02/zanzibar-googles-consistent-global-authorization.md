---
title: "Zanzibar: Google's Consistent, Global Authorization System"
source: https://www.usenix.org/conference/atc19/presentation/pang
author: Ruoming Pang et al.
company: Google
date_posted: 2019-07-10
date_digested: 2026-06-02
---

# Zanzibar: Google's Consistent, Global Authorization System

## What's new to learn

- **Relation tuple** — a three-field record `object#relation@subject` that is the universal unit of access control in Zanzibar. Every ACL model — role-based, attribute-based, relationship-based — reduces to this one data type composed with a small algebra of union, intersection, and exclusion.
- **Zookie** — an opaque causality token that encodes a Spanner snapshot timestamp. Clients store it alongside their content after an ACL write; subsequent permission checks pass it back so Zanzibar reads an ACL snapshot no older than that write, solving the "new enemy problem" without requiring full linearizability.
- **Leopard index** — a three-layer system (offline snapshot builder + online delta layer + skip-list serving layer) that materializes the transitive closure of group membership so a recursive "is this user in this group?" query that would normally require O(depth) Spanner reads becomes a near-O(1) precomputed set lookup.

## Prerequisites

- Role-based access control basics (roles, permissions, subjects — you don't need ABAC or ReBAC)
- Graph traversal (BFS/DFS to follow edges in a directed graph)
- Snapshot isolation / MVCC: the idea that a database can read at a past timestamp rather than always the latest version
- Distributed consistency vocabulary: what "external consistency" means (roughly: transactions appear to happen in an order consistent with real time), and why it's expensive

## The core idea

Every permission check is a graph reachability question.

Zanzibar stores access-control state as a collection of relation tuples. Each tuple is an edge in a directed graph: `object#relation@subject` says that `subject` has the relationship named `relation` with `object`. Subjects can be users, or they can be other `object#relation` pairs — meaning "everyone who has relation R on object O."

Checking "does `user:alice` have `viewer` on `doc:42`?" means: starting from the node `doc:42#viewer`, can we reach the node `user:alice` by following edges in this graph?

That's it. The entire Zanzibar data model and check algorithm flows from this framing. Namespace configs let services express their own ACL semantics (owner → editor → viewer inheritance, folder → child-document propagation) as rewrite rules that say how to expand one node into other nodes during traversal. The check engine performs a memoized BFS over the expanded graph and returns true if it finds the target user.

The genius of this design is that one graph-reachability engine subsumes RBAC, ABAC, and ReBAC uniformly. There is no special-case code for "group membership" vs "role assignment" vs "hierarchical inheritance" — they are all just different shapes of edge in the same graph, traversed by the same algorithm.

## Mechanics

### The data model

A relation tuple takes one of two forms:

```
doc:42#viewer@user:alice            ← user is a leaf node
doc:42#viewer@group:eng#member      ← subject is itself a userset
```

The second form is a "userset reference." It means: "every user who has the `member` relation on `group:eng` also has the `viewer` relation on `doc:42`." This single syntactic move is what makes the model recursive and expressive.

### Namespace configs and userset rewrite rules

Each namespace (type of object) has a config that defines its relations and how they compose. A document namespace might look like:

```protobuf
name: "doc"

relation { name: "owner" }

relation {
  name: "editor"
  userset_rewrite {
    union {
      child { _this {} }                       // direct editor tuples
      child { computed_userset {
        relation: "owner"                      // owners are also editors
      }}
    }
  }
}

relation {
  name: "viewer"
  userset_rewrite {
    union {
      child { _this {} }
      child { computed_userset { relation: "editor" }}   // editors ⊇ viewers
      child { tuple_to_userset {
        tupleset   { relation: "parent" }      // look up doc:42#parent@folder:5
        computed_userset { relation: "viewer"} // then check folder:5#viewer
      }}
    }
  }
}
```

Three leaf node types:
- `_this` — the base set of tuples stored directly for this relation
- `computed_userset` — follow a *different* relation on the *same* object (e.g., `editor` → `viewer` inheritance)
- `tuple_to_userset` — a two-hop move: first look up all tuples matching a given relation (`parent`) to find a set of objects (`folder:5`), then evaluate a userset on each of those objects (`folder:5#viewer`). This is how permissions propagate up folder hierarchies.

These three leaf types can be freely composed with union, intersection, and exclusion to express nearly any authorization policy.

### The check algorithm

```
Check(object, relation, user, evaluate_at_timestamp):
  expand the userset for (object, relation):
    for each direct tuple (object#relation@S):
      if S is user:
        return ALLOWED
      if S is a userset ref (O'#R'):
        recurse: Check(O', R', user, evaluate_at_timestamp)  ← memoized
  apply userset rewrite rules:
    for each rule child (computed_userset, tuple_to_userset):
      evaluate the child rule, recurse
  return DENIED
```

All Spanner reads use the same `evaluate_at_timestamp` snapshot, ensuring the check is internally consistent — it won't see half of a concurrent ACL write. Results are memoized by `(object#relation, evaluate_at_timestamp)` key across the recursion tree, so shared ancestors are computed once.

### The new enemy problem

Authorization systems that span two data stores — one for content, one for ACLs — face a subtle consistency hazard. Consider:

1. Alice grants Bob editor access to `doc:42`  (write to ACL store)
2. Alice revokes Bob's access (write to ACL store)
3. Alice creates `doc:43`, copying `doc:42`'s ACL (read ACL store, write content store)

If step 3 reads the ACL store at a snapshot taken *before* step 2's revocation has committed, `doc:43` is created with an ACL that still lists Bob — even though Bob was already removed. Bob now has access to a document that was created *after* his removal. He was supposed to be an enemy; the stale snapshot made him a new friend.

This is a read-your-writes violation across two systems: the content creation in step 3 failed to observe a write that causally preceded it.

### Zookies: causal tokens for cross-system consistency

When an application writes or modifies an ACL (e.g., removing Bob's access), it requests a **zookie** from Zanzibar. Zanzibar returns an opaque token encoding the Spanner commit timestamp of that ACL write. The application stores this zookie atomically with the content in its own database.

```
[Application DB row]
  doc_id: 43
  content: "..."
  acl_zookie: <opaque bytes encoding timestamp T_revocation>
```

When any user tries to access `doc:43`, the application retrieves the stored zookie and passes it to Zanzibar's Check API:

```
Check(
  object: "doc:43",
  relation: "viewer",
  user: "user:bob",
  consistency: at_least_as_fresh(zookie)  ← enforce minimum snapshot
)
```

Zanzibar decodes the zookie, extracts `T_revocation`, and evaluates the check at a Spanner snapshot with timestamp ≥ `T_revocation`. The revocation is guaranteed visible. New enemy problem solved.

### The three consistency modes

| Mode | Guarantees | Latency | Use case |
|------|-----------|---------|---------|
| `at_least_as_fresh(zookie)` | Snapshot ≥ zookie's timestamp | Low (caching works) | Default — content access checks |
| `at_exact_snapshot(zookie)` | Snapshot = exactly zookie's timestamp | Low (caching works) | Paginating through a consistent list |
| `fully_consistent` | Latest snapshot (real-time) | High (bypasses all caches) | Admin UIs, security-critical paths |

The caching opportunity in `at_least_as_fresh`: because "at least as fresh as T" accepts any snapshot ≥ T, Zanzibar rounds T up to a coarse boundary (1–10 second intervals). Many clients with zookies in the range (T, T+boundary] all evaluate at the same rounded snapshot. Their check results share cache entries, dramatically improving hit rates without compromising the safety guarantee.

### Leopard: precomputed transitive group membership

A naive check for "is `user:alice` a member of `group:all-googlers`?" requires recursive BFS:
- `group:all-googlers` → member of `group:eng` and `group:pm` and ...
- `group:eng` → member of `group:backend` and `group:infra` and ...
- ... → eventually reach `user:alice` (or not)

At Google scale, this is O(depth × fan-out) Spanner reads per check — untenable.

Leopard precomputes the full transitive closure and makes it queryable as a set lookup. It has three layers:

**Layer 1 — Offline snapshot builder** (runs every few minutes):  
Reads all relation tuples, walks the group-membership subgraph to full transitive closure, and writes materialized index shards. Shards are replicated globally.

**Layer 2 — Online delta layer** (consumes the Watch changelog in real time):  
Captures every tuple mutation since the last snapshot. At query time, recent mutations not yet in the offline snapshot are merged on top.

**Layer 3 — Serving system** (handles low-latency queries):  
Stores two kinds of sets: users → all groups they belong to (directly or transitively), and groups → all descendant groups. Sets are stored as ordered integer lists in a skip-list structure, enabling efficient set union and intersection:

- Set union: merge two sorted lists, O(|A| + |B|)
- Set intersection: skip-list seeks, O(min(|A|, |B|) × log|B|)

Instead of BFS through 10 levels of group hierarchy, a Leopard lookup is: "fetch the precomputed set of all groups alice belongs to, intersect with {required_group}." One Leopard RPC replaces hundreds of Spanner reads.

### System architecture

```
┌─────────────────────────────────────────────────────┐
│  Client services (Drive, YouTube, Calendar, ...)    │
└──────────────────────┬──────────────────────────────┘
                       │ Read / Write / Check / Watch
              ┌────────▼────────┐
              │   Aclserver     │  ← stateless; fan-out to Spanner + Leopard
              │   (cluster)     │
              └────────┬────────┘
            ┌──────────┴──────────┐
            ▼                     ▼
    ┌───────────────┐    ┌────────────────┐
    │    Spanner    │    │    Leopard     │
    │  (relation    │    │  (group index) │
    │   tuples +    │    │               │
    │   ns configs) │    └────────────────┘
    └───────────────┘           ▲
            │                   │ changelog
            └──────►  Watchserver ─────┘
```

The Watchserver streams an ordered changelog of all tuple mutations to Leopard (for index maintenance) and to external clients (for real-time ACL change propagation).

### Scale

- **2+ trillion** relation tuples stored across Google's Spanner instances
- **10M+ authorization checks per second** sustained
- **p95 check latency < 10ms** globally
- **99.999% availability** over 3 years of production operation
- Client services: Google Calendar, Cloud IAM, Drive, Maps, Photos, YouTube

## Where it breaks

**Namespace config migrations are treacherous.** Changing a userset rewrite rule changes what a Check returns for existing tuples. There is no safe schema migration primitive — a two-phase rollout requires deploying code that understands both old and new configs simultaneously.

**Exclusion breaks monotonicity.** `NOT blocked_users` means adding a tuple can *revoke* access (adding a block). Non-monotone policies defeat some important caching invariants (you can no longer cache "DENIED" results safely across tuple mutations) and make reasoning about correctness harder.

**Leopard has bounded freshness.** The offline snapshot is rebuilt every few minutes. For very recently changed group memberships, you depend on the delta layer; if the delta layer hasn't caught up, you may fall back to on-the-fly BFS. Group membership changes are not instantaneously consistent from Leopard's perspective.

**`fully_consistent` is a latency cliff.** Bypassing all caches to get real-time accuracy can push p95 latency from single-digit to 50ms+. Applications that require real-time ACL views (e.g., an admin dashboard that just removed access) cannot easily put this on the hot path.

**Deep `tuple_to_userset` hierarchies amplify Spanner reads.** A `viewer` check on a doc inside a folder inside a shared drive inside an organization can fan out across four namespaces, each requiring Spanner reads. Amplification is bounded but proportional to hierarchy depth.

**No built-in DENY tuples.** Zanzibar's algebra is additive; permissions are granted but not explicitly denied (beyond exclusion sets). Modeling "blacklists override whitelists" requires careful namespace config design and is easy to get wrong.

## Why it works

**The deeper principle: authorization is just consistency-aware graph reachability.**

The Zanzibar paper is, at its core, a claim that all the complexity of real-world authorization systems — roles, groups, hierarchies, delegation, ownership — is just graph structure, and the only hard problem is making that graph queryable at a causally consistent snapshot efficiently.

This collapses a typically sprawling engineering problem (each product team inventing its own ACL system) into two subproblems:
1. Store the graph as relation tuples and a config DSL (solved by Spanner + namespace configs)
2. Make point queries on the graph fast enough at the required consistency (solved by Leopard + zookies)

**The zookie pattern is a general design primitive.**

The "new enemy" problem is not authorization-specific — it's a read-your-writes violation across two systems that don't share a transaction coordinator. The zookie is Google's answer: a causality token that travels with the data, carrying just enough information (a Spanner timestamp) to let the downstream system pick a safe read snapshot.

The same pattern appears everywhere distributed systems need to synchronize two data stores:
- **HTTP ETags / `If-None-Match`**: client carries a version token; server uses it to decide cache freshness
- **Lamport clocks / vector clocks**: each message carries a causal timestamp so the receiver knows which events happened-before
- **MVCC "safe snapshot" tokens** in CockroachDB and Spanner: a client reads at a known safe timestamp to get read-your-writes on a secondary replica
- **Kafka consumer group offsets**: a consumer carries its last processed offset; downstream processors use it to avoid reprocessing

The pattern is always: *you can't afford a globally serializable protocol, so encode the causal dependency in a token that travels with the data, and let the reader use it to select a valid snapshot.*

**Leopard is incremental view maintenance in disguise.**

Transitive group membership is a recursive query. Recursive queries are slow to compute on-demand. The answer — materialize the result offline, maintain it with a rolling delta — is the same as PostgreSQL materialized views, git pack objects (full pack + thin packs), search index incremental updates, and data warehouse change data capture. The "offline base + online delta" split is a standard pattern for maintaining any derived, expensive-to-compute data at low latency.

**The namespace config DSL is a small step away from a type system.**

Userset rewrite rules describe how permissions compose. `editor ⊇ viewer` is a subtype relationship. `tuple_to_userset` is a foreign-key join in the permission graph. The fact that Zanzibar can express RBAC, ABAC, and ReBAC in one language means it's essentially a small logic language for permission-graph reachability, with unions and intersections as its connectives. That framing makes it much easier to reason about what an authorization policy actually guarantees.

## Going deeper

1. **SpiceDB** (open-source Zanzibar implementation, AuthZed) — https://github.com/authzed/spicedb. Reading the `graph/` and `datastore/` packages alongside the paper concretizes how the namespace config language maps to an actual check engine. The zookie implementation (`core/proto/core/v1/core.proto`) shows exactly what's encoded in the token.

2. **MIT 6.5840 Zanzibar FAQ** — http://nil.csail.mit.edu/6.5840/2023/papers/zanzibar-faq.txt. A dense set of Q&As from the MIT distributed systems course covering the trickiest consistency edge cases: when `at_least_as_fresh` is sufficient, how the zookie interacts with Spanner's external consistency, and why `fully_consistent` is still not strictly linearizable.

3. **OpenFGA** (Microsoft's open-source Zanzibar) — https://openfga.dev/docs/modeling/getting-started. OpenFGA's modeling language (`authorization_model.json`) is a cleaner notation than the original paper's protobuf, making it easier to experiment with the userset rewrite algebra on real authorization scenarios. The project also has thorough documentation of the "new enemy" problem and how client-side token handling prevents it.
