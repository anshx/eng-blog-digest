---
title: "vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention"
source: https://blog.vllm.ai/2023/06/20/vllm.html
author: Woosuk Kwon, Zhuohan Li, Siyuan Zhuang, Ying Sheng, Lianmin Zheng, Cody Hao Yu, Joseph Gonzalez, Hao Zhang, Ion Stoica
company: UC Berkeley (LMSys)
date_posted: 2023-06-20
date_digested: 2026-05-30
---

# vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention

## What's new to learn

1. **KV cache fragmentation as the LLM serving bottleneck.** Before vLLM, every serving system pre-allocated a contiguous block of GPU memory for each request's key-value cache at its maximum possible sequence length. This caused 60–80% of GPU memory to be wasted on reserved-but-unused space, fundamentally limiting how many requests could be batched simultaneously — and therefore capping throughput.

2. **PagedAttention: paging applied to GPU memory management.** Rather than allocating one big contiguous slab per request, PagedAttention divides the KV cache into small, fixed-size *blocks* (16–32 tokens each) and tracks their location via a per-request *block table* — the direct analogue of an OS page table. Physical memory blocks can live anywhere in GPU DRAM; the block table provides the indirection that makes them appear contiguous to the attention kernel.

3. **Copy-on-write enables zero-cost KV sharing.** When a request forks into multiple output sequences (beam search, parallel sampling), the prompt's KV blocks are shared read-only among all forks, with a reference count. When two forks diverge, the diverging fork copies only the block being written — exactly the COW semantics Linux uses for `fork()`. This makes beam search and speculative decoding dramatically cheaper.

## Prerequisites

- **Transformer KV cache basics**: During autoregressive generation, each forward pass appends new key and value vectors for the new token to a per-layer cache. Subsequent steps read the full cached sequence — so cache size grows as O(tokens generated).
- **GPU memory tiers**: GPU HBM (high-bandwidth memory) is tens of GB but scarce; there is no swap. Running out means dropping requests or OOM-killing the process.
- **Continuous batching (Orca)**: The technique of dynamically adding new requests to an in-flight batch when earlier requests finish, rather than waiting for a full batch to complete. This is the scheduling model vLLM builds on.
- **Virtual memory paging (OS)**: Fixed-size pages, page tables mapping virtual to physical addresses, demand paging, and copy-on-write semantics. Not strictly required, but the PagedAttention design is explained directly in these terms.

## The core idea

Every LLM inference request produces an unpredictable number of output tokens. But GPUs cannot dynamically resize allocations — they require contiguous memory blocks. The pre-vLLM systems (FasterTransformer, Orca, HuggingFace) handled this by allocating the maximum possible KV cache size (max sequence length × model width) for every request at admission time. For a model with 13B parameters and 2,048 max tokens, one request's KV cache might be 1.6 GB — reserved in full even if the request generates only 10 tokens.

The result is severe waste:
- **Internal fragmentation**: the tail of the reserved block is almost never fully used.
- **External fragmentation**: when requests complete at different times, they leave gaps in the reserved pool that are too small for new requests with different max lengths.
- Combined, 60–80% of GPU memory sits reserved-but-useless.

The OS solved an identical problem for process memory in the 1960s: **virtual memory paging**. Instead of giving each process a contiguous physical address range, the OS divides physical RAM into fixed-size *frames* and maps *virtual pages* to frames via a *page table*. Processes see a contiguous address space; the OS allocates frames on demand, picking any available frame regardless of physical location.

PagedAttention applies this idea exactly:
- **Physical KV block**: a fixed-size slot in a pre-partitioned GPU memory pool (e.g., 16 tokens × all layers × hidden dim).
- **Logical KV block**: the attention kernel's view — it treats each request's KV cache as a contiguous sequence of blocks numbered 0, 1, 2, …
- **Block table**: a small per-request array mapping each logical block index to its physical block index in the pool.

When a request generates its nth token, vLLM's memory manager allocates one free physical block from the pool (if the current last block is full) and appends its index to the request's block table. The modified attention kernel dereferences the block table during computation — a single array lookup per block accessed.

The maximum waste per request is `block_size - 1` tokens in the last block, bounded regardless of maximum sequence length. Average measured waste: **< 4%**, versus 60–80% before.

## Mechanics

### Physical memory partitioning

At startup, vLLM computes how much GPU memory is available after loading model weights, reserves a configurable fraction (default: 90%), and divides it into a fixed pool of equal-size physical blocks. For a 13B LLaMA model on an A100 (80 GB), typical settings yield ~950 physical blocks of 16-token capacity each.

The memory manager maintains a global free-list of block indices. Allocating a block is O(1) (pop from free-list); freeing is O(1) (push back).

### Block table layout

Each active sequence maintains:
```
block_table[i] = physical_block_index   # for logical block i
```

For a sequence at position `t` (t tokens generated so far), it holds `ceil(t / block_size)` logical blocks. The block table array is small (at most `max_seq_len / block_size` entries — typically a few dozen integers) and fits trivially in CPU or GPU memory.

During an attention computation for token position `p`, the kernel computes:
```
logical_block   = p // block_size
offset_in_block = p %  block_size
physical_block  = block_table[logical_block]
kv_location     = kv_cache_pool[physical_block][offset_in_block]
```

This is one integer division, one modulo, and one table lookup — negligible overhead against the attention FLOP budget.

### Scheduler and preemption

vLLM's scheduler maintains a queue of waiting requests and a set of running sequences. On each step it:
1. Tries to advance all running sequences (generate one token each).
2. If GPU memory is exhausted, *preempts* the lowest-priority running sequence by swapping its blocks to CPU RAM or simply evicting them (accepting re-computation from the prompt on re-admission).
3. Admits new requests from the waiting queue as memory allows.

Because blocks are fixed-size and independent, preemption never requires compacting other sequences' memory. The freed blocks go directly back to the free-list.

### Copy-on-write for forked sequences

When a request uses beam search (k beams) or parallel sampling (n outputs), the scheduler forks the sequence at generation time:
- All beams start with the same prompt KV blocks. Rather than copying them, vLLM increments each block's reference count.
- A block with `refcount > 1` is read-only.
- When beam i needs to append to a shared block (because the block is full and all beams shared it), vLLM copies the block to a new physical block, decrements the shared block's refcount, and writes to the new private copy.
- When a beam finishes or is pruned, its blocks' refcounts are decremented; blocks that reach zero return to the free-list.

For a beam-search request with k=4 beams and a 512-token prompt, 32 blocks (512 / 16) are shared among all 4 beams for free. Without COW, each beam would independently hold a full 512-token copy: 4× the memory. With COW, shared prompt memory is O(1), divergence is paid incrementally.

### Prefix caching

A later extension (also in vLLM) hashes the content of each physical block and inserts fully-filled, immutable blocks into a cache indexed by hash. When a new request arrives whose prefix matches cached blocks, those blocks are reused without any computation or copy. System prompts shared across all requests (e.g., a long instruction set) are effectively computed once and shared across every simultaneous request.

### Performance numbers (from the SOSP 2023 paper)

| System | Throughput vs. HuggingFace (13B LLaMA) |
|--------|----------------------------------------|
| HuggingFace Transformers | 1× (baseline) |
| FasterTransformer | ~2–3× |
| Orca | ~4–6× |
| **vLLM (PagedAttention)** | **up to 24×** |

Under throughput-vs-latency tradeoff curves, vLLM achieves 2–4× the throughput of FasterTransformer and Orca at the same P99 latency target. The gain is purely from packing more requests into the same GPU memory budget.

## Where it breaks

**Block size is a tunable tradeoff, not a free variable.** Smaller blocks reduce internal fragmentation (less waste in last block) but increase block table overhead and reduce the granularity of prefetch and memory coalescing in the attention kernel. Larger blocks waste more memory per request tail but enable more efficient GPU memory access. The sweet spot (16–64 tokens) is workload-dependent.

**CPU swap has poor cost characteristics.** When preemption swaps blocks to CPU RAM, a future re-admission must copy them back over PCIe (12–32 GB/s) — orders of magnitude slower than NVLink or even GPU HBM. For latency-sensitive workloads, swapping is typically disabled and preemption instead re-computes from the prompt.

**Block table lookup is a new kernel dispatch pattern.** The modified attention kernel that dereferences the block table is non-standard and must be carefully written for each hardware target. Early vLLM shipped a pure-Python block table lookup with a custom CUDA kernel; integrating with new hardware (AMD ROCm, Intel Gaudi, custom ASICs) requires porting this kernel.

**COW adds a write-path branch.** Every new block allocation must check whether the block being appended to is shared. Under heavy parallel sampling, this check becomes a per-step CPU operation. In practice, the overhead is immaterial (a dictionary lookup), but it is a new code path invisible in simpler serving systems.

**Prefix cache invalidation is coarse.** The prefix cache keyed by block hash evicts whole blocks, not individual tokens. A request that shares 95% of a prefix with a cached request but differs in the last few tokens of the last block misses the cache entirely for that block. Fine-grained token-level caching (as in SGLang's RadixAttention) addresses this but complicates the memory manager significantly.

**Static pool sizing.** The physical block pool is carved out at startup. If a workload's actual memory usage is dramatically different from what was projected (e.g., all requests happen to be long), the pool may be undersized, increasing preemption, or oversized, leaving capacity underutilized.

## Why it works

The deeper principle is that **fixed-size indirection eliminates external fragmentation**.

External fragmentation is the disease of variable-size allocation: you have 100 MB free but it's split into 1,000 fragments of 100 KB each, and a 1 MB request can't be satisfied even though there's enough total space. The cure in every domain is the same: **stop allocating variable-size regions and instead allocate fixed-size units, then use an indirection table to make them appear contiguous to the consumer**.

- OS virtual memory uses page frames (4 KB each) + page tables.
- Database buffer pools use fixed-size pages (4–16 KB) + buffer descriptors.
- Slab allocators (kmalloc, jemalloc's size classes) use fixed-size object slots.
- GPU memory managers for matrix operations use fixed-tile scratch spaces.
- PagedAttention uses KV blocks (16–64 tokens each) + block tables.

In every case, the tradeoff is identical: you accept one indirection per access (the table lookup), you accept a small upper bound on internal fragmentation (one fixed-size unit per allocation tail), and you eliminate external fragmentation entirely. The table itself is small (proportional to number of allocations, not to allocation size), so it fits in CPU cache or a fast register file.

The second insight — COW for forked sequences — generalizes just as cleanly. COW works whenever:
(a) Two consumers initially share the same data, and
(b) Copies only need to be made for the diverging unit, not the whole dataset.

Linux `fork()` shares the entire address space until the first write. vLLM beam search shares KV blocks until the first token divergence. Functional persistent data structures share array nodes until the first modification. The structure of the insight is the same.

There is also a scheduling lesson here: **paged allocation is what makes preemption cheap**. If KV memory were one big contiguous slab per request, preempting a request would require copying a potentially gigabyte-scale buffer to CPU RAM. Because memory is already partitioned into small independent blocks, preemption can selectively evict individual blocks (or all blocks of one request) without touching any other request's memory. The same property makes garbage collection efficient in systems that use similar slab structures.

## Going deeper

1. **vLLM SOSP 2023 paper**: https://arxiv.org/abs/2309.06180 — the full system description, including the scheduler formalization, the custom CUDA kernel for block-table attention, and the complete benchmark methodology. Section 3 derives the memory utilization analysis; Section 4 describes the preemption and COW protocols.

2. **SGLang RadixAttention (2024)**: https://lmsys.org/blog/2024-01-17-sglang/ — extends prefix caching beyond whole-block granularity to a radix trie of token-level KV prefixes. A request shares not just common system prompts but any common prefix, at token resolution. Demonstrates that the paging abstraction composes with more sophisticated caching policies.

3. **Mooncake (2024)**: https://arxiv.org/abs/2407.00079 — a disaggregated KV cache system that extends PagedAttention across nodes, treating KV blocks as transferable objects between prefill and decode servers. Shows how the block abstraction enables a full secondary market for GPU memory, where idle capacity on one machine can serve another's KV cache needs over RDMA.
