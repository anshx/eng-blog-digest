---
title: "Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism"
source: https://arxiv.org/abs/1909.08053
author: Mohammad Shoeybi, Mostofa Patwary, Raul Puri, Patrick LeGresley, Jared Casper, Bryan Catanzaro
company: NVIDIA Research
date_posted: 2019-09-18
date_digested: 2026-06-24
---

# Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism

## What's new to learn

- **Tensor parallelism (intra-layer model parallelism)**: splitting a single weight matrix across multiple GPUs — column-wise for the first layer, row-wise for the second — as opposed to data parallelism (all GPUs replicate the full model, different mini-batches) and pipeline parallelism (different layers on different GPUs).
- **Column→row chaining eliminates mid-block synchronization**: placing a column-parallel layer immediately before a row-parallel layer lets partial activations flow directly between the two matrix multiplications with no all-reduce between them — only one all-reduce per complete transformer block.
- **Head-parallel attention**: attention heads are naturally independent, so assigning contiguous head groups to GPU subsets gives the same column→row→all-reduce structure for free, without any change to the attention algorithm.

## Prerequisites

- Transformer MLP block structure: two linear projections with a GeLU activation in between (`H = GeLU(X·W1)`, `Y = H·W2`).
- Transformer self-attention: Q/K/V projections, scaled dot-product attention (per head), concatenation and output projection.
- What an **all-reduce** does: sums a tensor across all N GPUs in a group and distributes the result back to all of them. With Ring AllReduce, the bandwidth cost per GPU is 2(N−1)/N × message_size ≈ 2× message_size (covered in the Ring AllReduce digest).
- Data parallelism basics: all GPUs hold a full model copy; gradient all-reduce happens once per step after the backward pass (covered in the ZeRO and Ring AllReduce digests).

## The core idea

In a transformer MLP block, the forward pass is:

```
Y = Dropout(GeLU(X · W1) · W2)
```

where X is `(batch × seq × d)`, W1 is `(d × 4d)`, W2 is `(4d × d)`.

With k GPUs, Megatron **does not split the batch** (that's data parallelism) and **does not put W1 on one GPU and W2 on another** (that's pipeline parallelism). Instead, it splits both matrices simultaneously along the "hidden" inner dimension:

- **GPU i holds columns of W1**: `W1_i = W1[:, i·(4d/k) : (i+1)·(4d/k)]` — shape `d × (4d/k)`
- **GPU i holds rows of W2**: `W2_i = W2[i·(4d/k) : (i+1)·(4d/k), :]` — shape `(4d/k) × d`

Each GPU runs its local shard of the computation independently:

```
H_i = GeLU(X · W1_i)   # shape: batch × seq × (4d/k)
Y_i = H_i · W2_i       # shape: batch × seq × d
```

Then a single all-reduce sums the partial results: `Y = Y_1 + Y_2 + ... + Y_k`.

**Why this is correct**: expanding the full computation,

```
GeLU(X · W1) · W2
= GeLU(X · [W1_1 | W1_2 | ... | W1_k]) · [W2_1; W2_2; ...; W2_k]
= [GeLU(X·W1_1) | GeLU(X·W1_2) | ... | GeLU(X·W1_k)] · [W2_1; W2_2; ...; W2_k]
= Σ_i  GeLU(X·W1_i) · W2_i
= Σ_i  Y_i
```

The GeLU distributes over the column split because it's element-wise: each GPU owns a contiguous column block of XW1, applies GeLU element-wise to its block, then contracts with the corresponding row block of W2. The partial outputs Y_i are additive contributions to the same `(batch × seq × d)` output space.

The critical structural fact: **the inner (contraction) dimension is split, not the outer dimensions**. Each GPU's contribution is a rank-`(4d/k)` update to the same output matrix, and those contributions sum to the full-rank result.

## Mechanics

### MLP block

```
Input X replicated on all k GPUs
│
├─ GPU 0: H_0 = GeLU(X·W1_0)  ──► Y_0 = H_0·W2_0 ─┐
├─ GPU 1: H_1 = GeLU(X·W1_1)  ──► Y_1 = H_1·W2_1 ─┤─► All-reduce ──► Y = ΣY_i
└─ GPU k: H_k = GeLU(X·W1_k)  ──► Y_k = H_k·W2_k ─┘
```

- **Forward pass**: 1 all-reduce per MLP block (transmits `batch×seq×d` floats).
- **Backward pass**: 1 all-reduce per MLP block (transmits gradients w.r.t. X, shape `batch×seq×d`).
- **Total per block**: 2 all-reduces, regardless of the size of W1 and W2.

With d=12288 (GPT-3 scale), bfloat16, batch=2, seq=2048: each all-reduce transmits 2×2048×12288×2 ≈ 96 MB.

### Self-attention block

Multi-head attention (n_heads heads, each of dimension d_head = d/n_heads) is naturally column-parallel because heads are independent:

1. **Q/K/V projections**: W_Q, W_K, W_V each `d × d`. Column-split by head groups: GPU i owns `n_heads/k` complete heads, computing Q_i, K_i, V_i.
2. **Attention**: GPU i computes full attention for its head group — `Softmax(Q_i·K_i^T / √d_head)·V_i`. No communication; heads don't interact.
3. **Output projection W_O** (`d × d`): row-split to match the head-group partitioning. GPU i computes `O_i = AttnOutput_i · W_O_i`.
4. **All-reduce**: `O = Σ_i O_i`.

Same 2 all-reduces (forward + backward) per attention block.

### Embedding and output

The vocabulary embedding matrix `(vocab_size × d)` is split row-wise across GPUs: each GPU holds `vocab_size/k` token embeddings. The cross-entropy loss is computed in a vocabulary-parallel fashion — each GPU computes logits for its vocab shard and contributes to a distributed softmax, avoiding materializing the full `batch × seq × vocab_size` logit tensor.

### Full transformer layer cost

Per transformer layer: 4 all-reduce operations (attention forward, attention backward, MLP forward, MLP backward), each of size `batch × seq × d`.

With 96 transformer layers (GPT-3 scale) and k=8-way tensor parallelism: 384 all-reduces per gradient step. At NVLink bandwidth of 600 GB/s bidirectional, a 96 MB all-reduce completes in ~0.3 ms, totaling ~115 ms communication overhead per step — manageable alongside ~800 ms of compute on 8 × A100 GPUs.

### Performance (paper results)

| Model size | GPUs | Parallelism | Throughput | Scaling efficiency |
|-----------|------|-------------|------------|--------------------|
| 1.2B      | 32   | 8-way TP    | 39 TFLOPs/GPU | baseline |
| 8.3B      | 512  | 8-way TP × 64-way DP | 15.1 PFLOPs total | 76% |

The paper implemented this in native PyTorch — no custom CUDA kernels, no compiler changes. The entire change was inserting all-reduce calls and replacing `nn.Linear` layers with `ColumnParallelLinear` and `RowParallelLinear` wrappers.

## Where it breaks

**NVLink required for meaningful scaling.** All-reduce bandwidth is the bottleneck. NVLink 3.0 provides 600 GB/s within a server node (8 GPUs); PCIe 4.0 provides only 32 GB/s. Cross-node tensor parallelism via InfiniBand (200 Gb/s ≈ 25 GB/s) is 24× slower — the all-reduce overhead would dominate compute. In practice, tensor parallelism is applied *within* a node (up to 8 GPUs), while pipeline parallelism handles the cross-node dimension.

**Memory asymmetry.** Tensor parallelism reduces weight memory by 1/k. But input activations (X) are replicated on all GPUs, and intermediate activations (H_i) are each 1/k size but exist on all GPUs simultaneously. Activation recomputation (gradient checkpointing) is still needed for large batch sizes. The 2023 "Reducing Activation Recomputation" follow-on paper addresses this via sequence parallelism.

**Architecture coupling.** The column→row fusion requires that the operation between W1 and W2 is element-wise (GeLU, Dropout, bias). A non-separable operation (like a global normalization such as BatchNorm) would require an intermediate all-reduce, breaking the fusion.

**Communication frequency.** 4 all-reduces per layer × 96 layers = 384 synchronization points per step. Even if each is fast, the sheer number limits the ability to overlap communication with compute (overlapping requires a compute-to-communication ratio >> 1 per operation, which is tight for small batch sizes).

**Pipeline bubbles (when combined with pipeline parallelism).** The "Megatron v2" paper (SC 2021) combines tensor parallelism within a node with pipeline parallelism across nodes, but pipeline stages are idle during startup (forward fill) and drain (backward drain). This "bubble fraction" is `(p-1)/(m+p-1)` where p is pipeline stages and m is micro-batches per batch, requiring large micro-batch counts to amortize.

## Why it works

The column→row split is an instance of **outer product decomposition for distributed matrix multiplication**:

```
C = A · B = Σ_k  A[:,k] ⊗ B[k,:]
```

Any rank-1 decomposition of the contraction dimension produces independent contributions that sum to C. Megatron splits along `4d/k`-wide slices instead of single columns, but the principle is the same.

This is the same structure as:
- **BLAS register blocking**: tile the inner loop `k` dimension to keep a `(d_r × k_c)` tile of A and a `(k_c × d_c)` tile of B in registers, accumulating into a `(d_r × d_c)` output tile. The outer product of each tile pair contributes to the output — summed across tiles.
- **Systolic arrays (TPU)**: each cell in the matrix accumulates `A[i,k]·B[k,j]` for its assigned (i,j); the k-index streams through in time. The spatial dimension of the array corresponds to the row/column split; the streaming dimension corresponds to the contraction; the accumulation corresponds to the all-reduce.
- **Ring AllReduce scatter-reduce phase**: each worker computes `grad[i*G/N : (i+1)*G/N]` for its ring position, which is the sum of all workers' contributions to that gradient slice — the same "split along contraction, accumulate partial sums" logic.

Megatron achieves **one all-reduce per transformer block** (not one per matrix multiply) because the GeLU element-wise activation is "transparent" to the partitioning — it doesn't mix columns from different GPUs. This makes the column→row fusion possible: the two matrix multiplications share the same inner-dimension partition, so the intermediate activation H_i is self-contained on each GPU and never needs to be communicated. The boundary cost is paid exactly once at the layer output.

This is **stream fusion** applied to distributed computation: merge two separate communication points (one after W1, one after W2) into a single boundary communication (after the full block) by keeping data resident on each GPU through both operations.

## Going deeper

1. **"Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM"** (Narayanan et al., SC 2021 — https://arxiv.org/abs/2104.04473) — the "Megatron v2" paper that combines tensor parallelism, pipeline parallelism, and data parallelism into a 3D strategy, with interleaved pipeline schedules to reduce pipeline bubbles, demonstrated at 3,072-GPU scale.

2. **"Reducing Activation Recomputation in Large Transformer Models"** (Korthikanti et al., MLSys 2023 — https://arxiv.org/abs/2205.05198) — shows that the regions *between* tensor-parallel blocks (LayerNorm, Dropout on the full activation) can be sequence-parallelized (splitting the sequence dimension), replacing a replicated activation with a sharded one and cutting activation memory by the tensor-parallel degree.

3. **"Flash Communication: Reducing Tensor Parallelization Bottleneck for Fast LLM Inference"** (arXiv:2412.04964, 2024) — demonstrates that FlashAttention's tiling can be adapted to overlap tensor-parallel all-reduce communication with compute, reducing the effective communication cost by ~2×.
