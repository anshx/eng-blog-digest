---
title: "Orca: A Distributed Serving System for Transformer-Based Generative Models"
source: https://www.usenix.org/conference/osdi22/presentation/yu
author: Gyeong-In Yu, Joo Seong Jeong, Geon-Woo Kim, Soojeong Kim, Byung-Gon Chun
company: Seoul National University
date_posted: July 2022
date_digested: 2026-08-11
---

# Orca: A Distributed Serving System for Transformer-Based Generative Models

## What's new to learn

1. **Iteration-level (continuous) batching**: Scheduling LLM inference at the granularity of a single forward pass — one token generated per sequence — so that finished sequences immediately free their GPU slot for new arrivals rather than waiting for the entire batch to complete.

2. **Selective batching**: The insight that attention operations are sequence-history-dependent (cannot be naively batched across requests with different KV cache lengths) while FFN and linear layers are token-local (can be batched across *all* tokens in flight, regardless of which request they belong to).

3. **Work-conserving GPU scheduling**: The general principle that a GPU serving variable-length generation workloads should never be idle while requests are queued, and that achieving this requires preemption at token-generation granularity rather than at request completion.

## Prerequisites

- How autoregressive decoding works: each new token requires one full forward pass through the model, conditioned on all previous tokens via a KV cache
- What a KV cache is: the key and value projections for all prior tokens in a sequence are stored to avoid recomputing them on every step
- Transformer architecture basics: the attention sublayer (Q/K/V projections, scaled dot-product, softmax) and the feedforward sublayer (two linear layers with a GeLU nonlinearity)
- Intuition for GPU batching: why larger batch sizes improve arithmetic intensity (FLOPs / bytes moved) and thus GPU utilization

## The core idea

Naive LLM serving batches at **request level**: group N requests, run the model until *every single request* generates an EOS token or hits the maximum length, return all N results, start a new batch. The problem is that real workloads have wildly variable output lengths. If one request needs 5 tokens and another needs 2,000, the 5-token request finishes in 5 iterations — then its GPU slot sits empty for 1,995 iterations while the long request runs out.

Orca breaks this by scheduling at the **iteration level**. An iteration is one forward pass through all model layers — it generates exactly one new token per in-flight sequence. After each iteration, the scheduler:

1. Identifies sequences that just generated EOS (or hit max length) and evicts them immediately
2. Admits new requests from the waiting queue to fill the freed slots
3. Runs the next iteration on the updated, "refilled" batch

The GPU is never voluntarily idle while work is queued. Requests continuously drain out (on completion) and drip in (from the queue) — hence the colloquial name **continuous batching**.

The 36.9× throughput improvement over NVIDIA FasterTransformer on GPT-3 175B at the same latency target comes almost entirely from this: naive batching wastes GPU compute on padding and idle slots; continuous batching reclaims it all.

## Mechanics

**The iteration loop (simplified):**

```python
while True:
    # Evict finished sequences; return results to callers
    done = [s for s in running if s.last_token == EOS or len(s) >= max_len]
    for s in done:
        running.remove(s)
        results[s.id] = s.tokens

    # Fill freed slots from the waiting queue
    while len(running) < max_slots and waiting:
        running.append(waiting.pop(0))

    if not running:
        break

    # One forward pass: one new token per sequence
    new_tokens = model.step(running)
    for s, tok in zip(running, new_tokens):
        s.tokens.append(tok)
```

**Why attention cannot be batched naively:**

Standard batching stacks N sequences into a 3D tensor `[N, seq_len, d_model]`. This requires all sequences to have the same length (or padding). In continuous batching, sequences have heterogeneous KV cache lengths — sequence A might be on its 4th token, sequence B on its 831st. There is no uniform `seq_len` to form a 3D tensor.

**Selective batching — the key implementation insight:**

Orca observes that Transformer operations fall into two categories:

*Sequence-aware* operations — **attention layers**: The output for token *t* in sequence *i* depends on the KV cache of all prior tokens in *sequence i*. Different sequences have different-length caches. You cannot batch these across sequences with different histories; you must process each sequence individually.

*Token-local* operations — **FFN, linear projections, layer norm, GeLU**: The output for any given token is a pure function of *that token's embedding alone* — no dependence on sequence position, sequence length, or which request the token belongs to. All tokens in the batch are therefore interchangeable for these operations.

Orca's solution: flatten all tokens across all in-flight sequences into a single 2D token batch for non-attention layers, then split back per-sequence for attention.

```python
# Attention phase: per-sequence, variable-length KV lookup
attn_outputs = []
for seq in batch:
    q = W_q @ seq.current_embedding
    k = W_k @ seq.current_embedding
    v = W_v @ seq.current_embedding
    kv_cache[seq.id].append((k, v))             # grow sequence's own cache
    K, V = stack(kv_cache[seq.id])              # [L_i, d_k] and [L_i, d_v]
    score = softmax(q @ K.T / sqrt(d_k))        # [L_i] attention distribution
    attn_outputs.append(score @ V)              # [d_v] output token embedding

# FFN phase: ALL tokens batched together across ALL sequences
all_tokens = stack(attn_outputs)                # [total_tokens, d_model]
ffn_out = W2 @ gelu(W1 @ all_tokens.T)         # one large matrix multiply
```

The FFN phase sees `total_tokens` tokens — the sum of all sequences' current positions — rather than just `batch_size` tokens. This dramatically improves the arithmetic intensity of the biggest compute kernels.

**Distributed tensor parallelism:**

For GPT-3-scale models (175B parameters), Orca adds tensor parallelism across multiple GPUs. Each attention head's Q/K/V projections are split column-wise across GPUs, and the output projection is split row-wise — one all-reduce per attention sublayer. The FFN's two linear layers follow the same column-then-row split, with one all-reduce after the second linear. The iteration-level scheduler runs on a controller node; model shards on worker GPUs wait for the scheduler's per-iteration dispatch.

**Admission control:**

Not all waiting requests can be admitted immediately — the KV cache for `k` in-flight sequences consumes `k × L × 2 × d_model × n_layers × dtype_bytes` bytes of VRAM. Orca's admission controller estimates VRAM headroom using the current total token count and blocks admission when remaining VRAM is insufficient. This estimation is loose (you don't know how many more tokens each sequence will generate), which is why later work (vLLM/PagedAttention) replaced it with fine-grained paged memory management.

## Where it breaks

**KV cache memory fragmentation**: Orca pre-allocates contiguous memory per sequence for its full maximum length. Since sequences vary dramatically in actual length, this wastes 60–80% of VRAM on over-allocation — the exact problem PagedAttention was designed to solve.

**Prefill-decode interference**: When a new request joins the running batch, its prompt tokens all need to be processed at once (prefill: compute-bound, high FLOPs/byte). This runs in the same iteration as one-token decode steps for ongoing requests (memory-bandwidth-bound, low FLOPs/byte). Batching them together creates a compute spike that inflates the decode latency for every ongoing sequence in that iteration. Chunked prefill (Sarathi, 2023) mitigates this by spreading the prompt across multiple iterations. Prefill-decode disaggregation (DistServe/Mooncake) eliminates it by routing them to separate hardware.

**Attention is still sequential per sequence**: Selective batching batches the FFN efficiently, but attention is still O(L²) per sequence and still processed one sequence at a time. For very long contexts, attention dominates and the FFN-batching advantage shrinks.

**Tail latency**: Continuously admitting new long-context requests can delay ongoing requests' decode steps by growing the effective batch size. FCFS admission can cause priority inversions where a newly arrived 10,000-token request blocks a 1-token response.

**Output-length unpredictability**: The scheduler cannot know in advance how long any sequence will run, so it cannot plan batch composition optimally. Requests that turn out to be short waste little, but requests that turn out to be very long hold their slot for hundreds of iterations, preventing admissions.

## Why it works

The fundamental insight is that **request-level batching is non-work-conserving**.

A scheduler is *work-conserving* if it is never idle when there is runnable work. Request-level batching violates this invariant: as soon as any sequence in a batch finishes, its GPU compute is wasted — but the batch cannot change until every sequence finishes. With variable output lengths (which are universal in LLM workloads), long-tail sequences cause the rest of the batch's slots to sit idle.

This is the exact same principle as **preemptive multitasking** in operating systems. Cooperative multitasking (non-preemptive) requires processes to voluntarily yield; a long-running process starves everyone else. Preemptive scheduling forces a context switch at each time quantum — and Orca's iteration boundary is the preemption quantum for GPU inference. The model is the CPU; the KV cache is the process context that gets saved and restored.

From queuing theory (Kleinrock's conservation law): for a single-server queue with variable service times, a work-conserving FCFS discipline minimizes the mean number of customers in system among all non-preemptive work-conserving policies. Non-work-conserving policies (like request-level batching) necessarily perform worse in mean waiting time. Orca recovers optimality.

The **selective batching** principle is a special case of "identify and batch the commutative, order-independent subcomputations." The FFN is a pure function of one token's embedding — it is trivially parallelizable over tokens because tokens do not interact in the FFN sublayer. Attention is a join between a query token and all prior key-value pairs in *the same sequence* — that dependency structure prevents inter-sequence batching. Recognizing this independence-vs-dependency partition is the algorithmic kernel of the paper, and the same principle applies anywhere you want to exploit parallelism: only batch operations whose outputs are independent of each other's inputs.

The deeper unifying frame: **the right granularity of scheduling reveals the right granularity of parallelism**. If you schedule at request granularity, you treat a multi-thousand-token generation as the atomic unit of work. If you schedule at iteration granularity, you discover that each token step is nearly embarrassingly parallel across sequences (except for attention), and you can exploit that structure.

## Going deeper

1. **vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention** (Kwon et al., 2023) — the immediate follow-on that fixed Orca's memory management weakness. Once continuous batching became standard, KV cache fragmentation became the binding constraint; paged KV caches solved it. Covered in this archive at 2026-05-30.

2. **Sarathi: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills** (Agrawal et al., 2023) — [arxiv.org/abs/2308.16369](https://arxiv.org/abs/2308.16369) — directly addresses the prefill-decode interference problem in Orca's continuous batching. Proposes splitting a prompt's prefill across multiple iterations as "chunks" interleaved with ongoing decodes, smoothing the compute spike.

3. **Disaggregated Inference / DistServe** (Zhong et al., 2024) — covered in this archive at 2026-05-14 — takes the prefill-decode problem to its logical endpoint: route them to separate hardware pools entirely, exploiting their orthogonal bottleneck profiles (compute-bound vs. memory-bandwidth-bound), rather than trying to coexist in the same batch.
