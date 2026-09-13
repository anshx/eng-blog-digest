---
title: "Fast and Expressive LLM Inference with RadixAttention and SGLang"
source: https://www.lmsys.org/blog/2024-01-17-sglang/
author: Lianmin Zheng, Liangsheng Yin, Zhiqiang Xie, et al.
company: LMSYS (UC Berkeley RISELab)
date_posted: 2024-01-17
date_digested: 2026-09-13
---

# Fast and Expressive LLM Inference with RadixAttention and SGLang

## What's new to learn

- **Radix tree as a KV cache index** — a compressed trie maps token sequences → pre-computed KV cache block lists; longest-prefix matching against this tree is how you find the maximum reusable cache on each new request.
- **Reference-counted tree eviction** — nodes in active use carry a ref count that prevents eviction; only unreferenced leaf nodes are eligible, so evicting a leaf never invalidates the shared prefix its parent encodes.
- **Cache-aware request scheduling** — the runtime reorders the incoming batch to maximize prefix tree hits, treating the radix tree as an explicit locality signal rather than executing requests in arrival order.

## Prerequisites

- **KV cache in autoregressive decoding**: each transformer layer needs the key and value tensors from all previously generated tokens; those are cached to avoid recomputing them on every new token.
- **Prefill vs. decode**: the *prefill* step processes all input tokens in parallel and is compute-bound; the *decode* step generates one token at a time and is memory-bandwidth-bound. Every input token must be prefilled before the first output token appears, so long shared prefixes paid on every request are a latency tax.
- **Paged KV cache (PagedAttention/vLLM)**: KV cache is stored in fixed-size pages, not contiguous per-request blocks, to eliminate memory fragmentation. This is the prior work RadixAttention builds on — assume the reader knows it (covered in this archive's 2026-05-30 post).
- **Radix tree / compressed trie**: a space-efficient prefix tree where edges are labeled with multi-character (here: multi-token) strings rather than single characters, so "ABCD" is a single edge rather than four.

## The core idea

In practice, a large fraction of production LLM requests share a common prefix: every request to a customer-service agent has the same 2,000-token system prompt; every few-shot reasoning request has the same 500-token example block; every multi-turn conversation replay has the same prior turns. With naive serving, every request pays the full prefill cost for that prefix — again and again.

RadixAttention eliminates this waste by maintaining a **shared radix tree** whose keys are sequences of token IDs and whose values are the corresponding pre-computed KV cache pages. When a new request arrives, the runtime walks the tree from the root, greedily extending the match as long as the request's token IDs match an existing edge. The matched prefix's KV cache is reused verbatim; only the unmatched suffix needs fresh prefill. The resulting KV cache for the new suffix is then inserted back into the tree for future reuse.

The key insight that makes this tractable — and that distinguishes a radix tree from a hash table — is *longest-prefix semantics*. A hash table gives exact-match lookups: if the request's first token differs from any stored prefix, you get nothing. A radix tree gives you maximum reuse: if 2,000 tokens match and only the final system-prompt version badge changed, you still reuse those 2,000 tokens' worth of KV pages.

## Mechanics

**Tree structure.** Each node holds:
- A list of token IDs labeling the edge from its parent (the "edge label")
- A list of GPU memory page indices for the KV cache of those tokens
- A reference count (`lock_ref`) — incremented when an active request traverses this node, decremented when the request finishes
- A last-access timestamp for LRU ordering

The root has `lock_ref = 1` (permanent), making it ineligible for eviction.

**Request handling.**
1. The scheduler calls `match_prefix(token_ids)` on the incoming request. The function walks the tree, matching as many tokens as possible. It returns the longest matching prefix length `p` and the list of matched KV page indices.
2. The runtime processes tokens `[0, p)` as a cache *hit* — their KV pages are already resident on GPU. Tokens `[p, n)` are prefilled normally.
3. After generating the first token, `extend(new_token_ids, new_kv_pages)` inserts the new suffix into the tree. If the insertion point requires splitting an existing edge (the new request shares a prefix of length `k < edge_len` with a stored node), the edge is split: a new internal node is created at position `k`, and both the old subtree and the new suffix hang from it.

**Eviction.** When GPU memory pressure requires freeing pages, the eviction policy scans leaf nodes (those with no children and `lock_ref == 0`) and removes the one with the lowest LRU priority. The pages it held are returned to the free pool; the node is unlinked. Crucially, leaf eviction never touches parent nodes — shared prefixes survive even when every branch hanging off them has been evicted. SGLang exposes seven pluggable eviction strategies (LRU is the default; others include LFU and timestamp-weighted variants).

**Concurrent access.** Concurrent requests increment the ref count on every node they traverse before prefill starts, and decrement it in a finally block. An eviction pass only considers `lock_ref == 0` nodes. This is the same reference-counting discipline as OS page frames pinned by active I/O — nothing that is being actively used can disappear beneath you.

**Batch scheduling.** The scheduler sorts the incoming batch so that requests sharing a long common prefix are dispatched together. This maximizes warm-cache utilization: the first request in the group populates the tree; subsequent requests in the same batch round hit it immediately.

**Performance numbers.** On workloads with moderate prefix sharing (agent pipelines reusing a system prompt plus tool definitions), cache hit rates reach 75–95% and end-to-end throughput is 5–6× higher than a baseline that recomputes the full prefix every request. On benchmarks with very high sharing — like tree-of-thought where hundreds of search branches share an identical prompt prefix — hit rates approach 99%.

## Where it breaks

**Zero sharing → zero gain.** If every request has a unique prefix (raw document summarization with a different document per call), the tree fills up and LRU evicts everything immediately. RadixAttention imposes non-trivial bookkeeping overhead in this case with no benefit.

**Token-level granularity bites.** Cache reuse is all-or-nothing at the token level. If a system prompt changes by even one token — a version badge, a dynamic date — the match stops at the changed token and nothing after it is reused. Implementors often work around this by pinning a canonical system prompt with a static identifier and stripping dynamic fields out.

**Privacy / timing side-channels.** Sharing KV cache between requests from different users means that a user's prefill time leaks information about what other users have recently prefixed — if another user's prefix warms the cache, your TTFT drops noticeably. Selective KV cache sharing policies (share within the same user/session, not across users) are the standard mitigation, at the cost of reduced hit rates.

**Tree lock contention.** The radix tree is a shared mutable data structure guarded by a lock. Under very high request concurrency (thousands of requests/sec on a large cluster), `match_prefix` and `extend` serialization can become a bottleneck. Sharded trees by prefix hash are the standard scaling fix, but the reference implementation uses a single lock.

**Quantization incompatibility.** A KV cache block computed at FP16 cannot be reused by a request that expects INT8 KV cache. Mixed-precision deployments must partition the tree by precision level.

## Why it works

The deepest principle is: **choose your data structure to match the algebraic structure of your keys**.

The keys here are token sequences, and token sequences have a *prefix-order* (one sequence can be a prefix of another). A hash table discards this structure entirely — it hashes the whole key and gives O(1) exact lookup. A sorted array preserves it but needs O(log n) comparison. A radix tree **exploits** it: it gives O(prefix\_length) lookup, and the bonus is that the lookup also finds the maximum reusable prefix even when there is no exact match. That "maximum reuse on partial match" semantics is the *only* reason to use a trie rather than a hash table, and it's precisely what LLM serving needs.

The same reasoning explains why this data structure appears wherever keys are sequences and partial matches have value:
- **IP routing tables** — CIDR prefixes use longest-prefix matching in a trie; a /24 route is reused by all /32 packets in that range.
- **URL routers** in web frameworks — `GET /users/:id/posts` shares prefix matching cost with `GET /users/:id/avatar`.
- **DNS caching** — zone records for `example.com` are inherited by `sub.example.com`; the zone hierarchy is a trie.
- **Compiler incremental builds** — a build system can share compiled output for `src/lib/a.o` across all targets that depend on it by treating file path prefixes as cache keys.
- **LLM agent frameworks** — every tool-use sequence starting with the same system prompt + tool definitions warms the same cache entries; the agent's conversation tree maps directly to the radix tree.

The ref-counted leaf eviction is the OS page-replacement algorithm applied to a tree instead of a flat page array. The invariant — "shared pages are never evicted unilaterally" — is the same one that keeps a shared library's `.text` segment resident as long as any process maps it, regardless of individual process lifetimes.

What makes RadixAttention the "right" solution rather than a hack is that it makes the memoization data structure isomorphic to the workload's call tree. LLM serving with system prompts *is* a tree of partial computations that share a root; the radix tree represents exactly that tree.

## Going deeper

1. **SGLang paper (NeurIPS 2024)** — the full academic treatment of both the SGLang language and the RadixAttention runtime, with formal complexity analysis and ablation studies: https://proceedings.neurips.cc/paper_files/paper/2024/file/724be4472168f31ba1c9ac630f15dec8-Paper-Conference.pdf

2. **Marconi: Prefix Caching for Hybrid LLMs (2024)** — extends the radix tree approach to models that mix full attention with sliding-window attention or SSM layers, where "prefix" has a more nuanced meaning because recurrent state can't simply be concatenated: https://arxiv.org/pdf/2411.19379

3. **Selective KV-Cache Sharing to Mitigate Timing Side-Channels (2025)** — analyzes the privacy implications of cross-user cache sharing and proposes per-user subtree partitioning: https://arxiv.org/pdf/2508.08438
