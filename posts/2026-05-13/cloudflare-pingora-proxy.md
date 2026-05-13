---
title: "How we built Pingora, the proxy that connects Cloudflare to the Internet"
source: https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/
author: Yuchen Wu, Andrew Hauck
company: Cloudflare
date_posted: 2022-09-14
date_digested: 2026-05-13
---

# How we built Pingora, the proxy that connects Cloudflare to the Internet

## What's new to learn

1. **Per-process connection pool fragmentation**: When an HTTP proxy scales by adding worker *processes*, each process gets its own isolated TCP/TLS connection pool. Adding more workers doesn't just add capacity — it fragments the pool, so each worker's pool is colder and issues more handshakes. Scaling can actively hurt connection reuse.

2. **Multithreading as a connection-pool strategy**: Switching from multiprocessing (the NGINX model) to multithreading lets every thread share one unified pool. Any thread can reuse any connection regardless of which thread opened it, because they live in the same address space.

3. **Rust's compile-time safety as a substitute for process isolation**: Process isolation's main appeal is protection from shared-state bugs. Rust's ownership model and `Send`/`Sync` traits enforce at compile time that mutable state is accessed safely across threads — giving the safety of process isolation with the sharing benefits of threads, and without a garbage collector.

## Prerequisites

- How a reverse proxy works: client terminates at the proxy, proxy opens a separate upstream connection to the origin
- NGINX's pre-fork model: N worker processes, each with an event loop and a local connection cache
- Why TLS handshakes are expensive: certificate chain verification, ECDH key exchange, and cipher negotiation add ~1–10 ms of CPU-bound latency and can't be parallelised with the actual data transfer
- Basic Rust: `Arc<T>`, `Mutex<T>`, and `async`/`await`

## The core idea

NGINX pre-forks N worker processes. The OS distributes incoming connections across them, usually round-robin on the listening socket. Each worker is fully isolated: separate address space, separate file descriptors, separate upstream connection pool.

This isolation is the source of a non-obvious scalability trap. When worker 1 opens a TLS connection to `origin.example.com` and then sits idle, worker 2 cannot borrow it. The connection stays stranded in worker 1's pool. Now scale from 8 to 16 workers: you haven't doubled your connection reuse — you've split the same pool across twice as many silos, halving the average fill of each. More workers → colder pools → more handshakes → more CPU → higher latency, in a vicious cycle.

At Cloudflare's scale (hundreds of millions of requests per second across thousands of edge machines), this fragmentation burned enormous CPU in TLS re-negotiations.

Pingora's core fix is a single architectural change: replace processes with threads. All threads share one process and therefore one connection pool. The pool is a hash map from `(host, port, TLS SNI, protocol)` to a list of idle connections. Any thread that needs an upstream connection consults this map, gets an existing connection if one is available (no matter which thread created it), or opens a new one. The scope of reuse expands from *one worker* to *every thread in the process*.

## Mechanics

**NGINX's connection lifecycle (simplified):**

1. OS sends an accept event to whichever worker is waiting on the listen socket.
2. That worker proxies the request and caches the upstream connection in its local `ngx_http_upstream_keepalive_module` pool.
3. The next request for the same origin must land on *the same worker* to hit that cached connection. Any other worker goes to the origin cold.
4. With N workers uniformly distributing load, connection-reuse probability for a given origin is bounded by `1/N` for a perfectly uniform distribution — so a 64-worker NGINX has at most ~1.5% theoretical reuse even if the pool is full.

In practice Cloudflare measured much better than that (because large origins accumulate real connections), but the fragmentation was still costing them orders of magnitude more handshakes than necessary.

**Pingora's unified pool:**

```
┌──────────────────────────────────────────────────────┐
│  Pingora process (one OS process, many OS threads)   │
│                                                      │
│  Thread 1 ──┐                                        │
│  Thread 2 ──┤── Shared connection pool ──► origin A  │
│  Thread 3 ──┤   (Arc<Mutex<HashMap<Key, Vec<Conn>>>> │
│  Thread 4 ──┘    sharded by key hash)    ──► origin B│
└──────────────────────────────────────────────────────┘
```

- The pool is behind an `Arc` so all threads can hold a reference without copying.
- The inner `Mutex` is sharded by key (host+port hash) to reduce contention — each bucket has its own lock, so two threads opening connections to different origins don't block each other.
- The Tokio work-stealing scheduler means a thread with no pending I/O can pick up work from a busy thread's queue, keeping all cores saturated without a dispatcher process.
- TLS session tickets and session IDs are cached per origin in the pool entry and reused on new connections, so even a freshly-opened connection avoids a full handshake.

**Custom HTTP library:**

Cloudflare couldn't use an off-the-shelf Rust HTTP library because real-world origin servers routinely send technically-invalid HTTP: status codes 599–999, malformed `Content-Length`, truncated headers. NGINX's decades of workarounds for these are not documented; they're embedded in C code and Lua patches. Pingora re-implements HTTP/1.1 and HTTP/2 parsing in Rust to replicate those tolerances. This is a maintenance cost but also meant they could use Rust's type system to model the full HTTP state machine, including extensions needed for Cloudflare-specific features (e.g., routing a request to a different origin mid-stream with different headers — something NGINX's filter pipeline doesn't support).

**Results in production (serving ~1 trillion requests/day):**

| Metric | Improvement |
|--------|-------------|
| CPU usage | −70% |
| Memory usage | −67% |
| Median TTFB | −5 ms |
| P95 TTFB | −80 ms |
| New TLS handshakes/s | 3× fewer |
| Connection reuse (one major customer) | 87.1% → 99.92% (160× fewer new connections) |
| Aggregate TLS time saved | ~434 years per day |

## Where it breaks

**Larger blast radius per crash.** With NGINX, a misbehaving worker crashes in isolation; the OS recycles it and the other workers keep running. In a multi-threaded process, a thread that corrupts shared memory (via an `unsafe` block or FFI call) can kill the whole process. Cloudflare mitigates this by running many Pingora processes across the fleet — a crash affects only one — but it's a real tradeoff of the shared-memory model.

**Lock contention under pathological access patterns.** The sharded pool helps, but the shards are still mutexes. A workload that hammers a single hot origin (one shard) from thousands of threads will see contention. NGINX avoids this by design: each worker is contention-free for its own pool.

**Proprietary HTTP parser maintenance.** Not using `hyper` or another community-maintained library means Cloudflare owns every HTTP quirk they encounter. When a new server misbehaves in a new way, they have to patch their own parser.

**The build-vs-buy decision was a multi-year bet.** Cloudflare explicitly evaluated Envoy (Lyft's NGINX replacement, also Rust-compatible) and decided it didn't fit their programmability needs. Choosing neither "keep NGINX" nor "adopt Envoy" and instead building Pingora from scratch required ~100 engineers over several years. The efficiency gains only justified this because Cloudflare's scale is large enough that a 70% CPU reduction translates to a significant fraction of their infrastructure bill.

## Why it works

The deeper principle: **the granularity of isolation determines the granularity of resource sharing**.

Process isolation was the right call in 2004 when NGINX was designed. C's memory model made shared-state multithreading genuinely dangerous. Processes gave safety by construction. The cost — isolated pools, IPC overhead for any shared state — was acceptable because TLS was uncommon and handshakes were cheap relative to the payload work.

By 2022, TLS is everywhere (TLS 1.3 is still meaningful CPU even on modern hardware), and workloads have inverted: the proxy spends far more CPU on TLS key exchange than on payload processing for many request types. Suddenly the cost of not reusing connections dominates.

Rust changed the calculus: it provides *compile-time* safety guarantees strong enough to replace the *runtime* guarantee that processes give you. You don't need an OS boundary to prevent data races; the borrow checker enforces that at the language level. So Pingora can have shared pools (and thus maximum reuse) *and* safety — a combination NGINX's era simply couldn't offer.

This is a specific instance of a pattern that recurs throughout systems design: **isolation has a cost, and that cost is inability to share**. The same tradeoff appears at every level of the stack:

- **CPU caches**: L1 is per-core (fully isolated, fastest); L3 is shared across cores (coordination overhead, but larger effective pool).
- **Database backends**: PostgreSQL uses one process per connection (isolation), which is why connection poolers like PgBouncer exist — they're doing exactly what Pingora does: introducing a shared-state layer between clients and isolated workers to amortise connection cost.
- **Container vs. VM**: Containers share the OS kernel (more sharing, less isolation); VMs have a hypervisor boundary (full isolation, higher overhead).
- **Go's goroutines**: Share heap memory like Pingora's threads, but rely on GC to prevent use-after-free. Rust achieves the same without GC by making ownership explicit at compile time.

The "so X is just an instance of Y" insight here: **Pingora's connection pool win is just PgBouncer applied to the HTTP proxy layer**. PgBouncer exists because PostgreSQL's per-process model fragments connection state; Pingora is the answer to NGINX's per-process model fragmenting TLS connection state. The solution is the same — introduce a shared-memory tier that pools the expensive resource across all workers.

## Going deeper

1. **[Open sourcing Pingora: our Rust framework for building programmable network services (Cloudflare, 2024)](https://blog.cloudflare.com/pingora-open-source/)** — When Cloudflare published Pingora's source, this companion post explains the public API and the "life of a request" hook model (request filter phases, upstream selection callbacks). Useful if you want to build your own proxy on top of Pingora rather than just understanding the architecture.

2. **[Cloudflare just got faster and more secure, powered by Rust (Cloudflare, Dec 2025)](https://blog.cloudflare.com/20-percent-internet-upgrade/)** — Three years after the original Pingora post, Cloudflare completed the full migration: FL2 (the new edge proxy built on Oxy, a higher-level framework above Pingora) replaced FL1 (NGINX/LuaJIT) in production, delivering another 10 ms median latency improvement and 25% overall throughput increase. The capstone of the Pingora story, and a case study in managing a multi-year critical infrastructure migration.

3. **[PgBouncer architecture documentation](https://www.pgbouncer.org/features.html)** — The PostgreSQL connection pooler solves exactly the same problem Pingora solves, just for the database protocol. Reading PgBouncer's session-pooling vs. transaction-pooling vs. statement-pooling modes makes the "isolation vs. sharing granularity" tradeoff concrete in a different domain — and helps you see why this class of shared-pool proxy appears independently wherever expensive per-process connections are fragmented.
