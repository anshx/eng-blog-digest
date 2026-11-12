---
title: "In-Datacenter Performance Analysis of a Tensor Processing Unit"
source: https://arxiv.org/abs/1704.04760
author: Norman P. Jouppi, Cliff Young, Nishant Patil, David Patterson, et al.
company: Google
date_posted: 2017-06-26
date_digested: 2026-11-12
---

# In-Datacenter Performance Analysis of a Tensor Processing Unit

## What's new to learn

1. **Systolic array**: a 2D grid of processing elements where data pulses through in waves — each cell multiplies its stored weight by a passing activation and accumulates into a partial sum flowing to the next cell. The array computes a 256×256 matrix multiply with zero intermediate memory traffic.

2. **Weight-stationary dataflow**: a strategy for GEMM (general matrix-matrix multiply) in which weights are pre-loaded into the array once, and activations stream through repeatedly — amortizing the cost of a weight fetch across all activations. The dual alternatives (output-stationary, input-stationary) make different traffic trade-offs; weight-stationary is optimal when the weight matrix is large and reused.

3. **CISC-at-the-chip-level**: giving a hardware accelerator a small set of wide, coarse-grained instructions (256-byte reads, 65,536-MAC multiply) so the host CPU's control overhead is negligible relative to the compute. The opposite design axis from the RISC philosophy — and appropriate when computation is regular and the bottleneck is data movement, not branching.

## Prerequisites

- Matrix multiplication mechanics (C[i][j] = Σ_k A[i][k] × B[k][j])
- Roofline model: a kernel is either compute-bound (hits peak FLOPS) or memory-bandwidth-bound (hits peak GB/s), and "arithmetic intensity" (FLOPs / bytes) determines which
- The idea of a PCIe-attached coprocessor (like a GPU or NIC)
- DMA and FIFO buffering between host and device

## The core idea

The TPU is a PCIe card with one job: matrix multiply as fast as physically possible in 40 W. Its heart is a 256×256 systolic array: 65,536 8-bit multiply-accumulate (MAC) units arranged in a grid. Data flows into this array the way blood is pumped through the heart (hence "systolic") — in coordinated, rhythmic waves — and the design ensures that data moves the minimum possible distance for the maximum amount of work.

The key design bet: **in neural network inference, data movement is more expensive than arithmetic**. A matrix unit running at 700 MHz would need 65,536 × 700M = 45.9 TB/s of data supply to stay busy. No SRAM or DRAM can deliver that. The systolic array solves this by keeping the data *inside the chip* for as long as possible, maximizing reuse per byte loaded from memory.

## Mechanics

### The matrix multiply unit (MXU)

The MXU is a 256×256 grid of PE (processing element) cells. Each PE holds:
- A single 8-bit weight (preloaded from the weight FIFO)
- A multiply-accumulate unit

The computation flows in two streams simultaneously:

1. **Weights**: loaded once from the weight FIFO into the leftmost column, then passed rightward one cell per cycle until the entire row of the array holds the corresponding row of the weight matrix.
2. **Activations**: fed in from the top, one column per cycle. Each activation enters the topmost cell, is multiplied by the weight stored there, and the partial sum flows *downward*, accumulating in each successive cell as the activation descends.

After 256 cycles, one full output column of the result matrix has accumulated in the bottommost row. Meanwhile, the next column of activations has already been fed in at the top. The pipeline is perfectly full.

This is **weight-stationary dataflow**: a 256×256 weight matrix loaded once from SRAM is applied to hundreds or thousands of activation vectors without ever reading weights from SRAM again. For typical MLP layers (e.g., 1024×4096 weight matrix), the weight is reused 1024 times per load.

### Memory hierarchy

| Component | Size | Bandwidth |
|---|---|---|
| Weight FIFO (off-chip weight DRAM) | 2 GB off-chip | 30 GB/s |
| Unified Buffer (activation SRAM) | 24 MiB on-chip | ~1.3 TB/s |
| Accumulators (partial sum SRAM) | 4 MiB (32-bit) | very high |

Notice: **there is no general-purpose cache.** The Unified Buffer is explicitly software-managed. The compiler (not hardware) decides when to DMA activations in and out. This is the same trade-off as GPU shared memory vs. L1 caches — explicit management is harder to program but eliminates cache conflict misses and speculative prefetches.

### Instruction set (deliberately coarse)

The TPU exposes five CISC-style macro-instructions to the host CPU:

1. `Read_Host_Memory` — DMA read from host DRAM into Unified Buffer
2. `Read_Weights` — DMA read of weights into the weight FIFO
3. `MatrixMultiply/Convolve` — run the MXU for the specified tile dimensions
4. `Activate` — apply elementwise activation function (ReLU, sigmoid, tanh) on the accumulators
5. `Write_Host_Memory` — DMA write Unified Buffer back to host

Each `MatrixMultiply` instruction covers 256×256 multiplications (65,536 MACs) in one shot. A typical neural network layer is hundreds or thousands of such instructions. Compare: a CPU SIMD instruction covers 8–16 multiplications; it would take ~8,000 CPU instructions to represent the same amount of work. This amortization of instruction-fetch and decode overhead is the CISC principle applied to the chip boundary.

### Die and power

- Process node: 28 nm (TSMC)
- Die area: 331 mm²
- TDP: 40 W
- Clock: 700 MHz
- Peak INT8 throughput: 65,536 × 700M × 2 = **92 TOPS**

For comparison, the K80 GPU of the same era peaked at ~8 TOPS INT8. The TPU outperforms it on inference by deploying all transistors toward one structured computation instead of a general-purpose pipeline.

### Roofline analysis and actual utilization

The paper's most instructive finding: **the TPU is often not compute-bound.** For MLP and LSTM inference at batch size 1, the workload is *memory-bandwidth-bound* on the weight FIFO (30 GB/s). Utilization of the matrix unit on these workloads falls to 10–25%.

Why? At batch size 1, there is exactly one activation vector per weight matrix read. The systolic array processes 256 activation columns simultaneously, so it can keep itself busy only if 256 or more activation vectors are waiting. With batch = 1, the weight fetch dominates. The fix is to increase batch size (even to 4–8), which reuses each weight fetch against more activations, increasing arithmetic intensity above the ridge point.

CNNs fare much better: convolutional weight kernels are small relative to the number of spatial positions they're applied to, so arithmetic intensity is naturally high, and the array runs near its compute ceiling.

### Benchmark results (vs. Haswell CPU + K80 GPU)

| Workload | TPU vs CPU | TPU vs GPU |
|---|---|---|
| MLP0 (inference) | ~100× | ~40× |
| MLP1 | ~30× | ~10× |
| LSTM0 | ~15× | ~5× |
| LSTM1 | ~20× | ~6× |
| CNN0 | ~40× | ~15× |
| CNN1 | ~30× | ~10× |

Performance/watt: **30–80× better than CPU, 30× better than GPU**. The design was originally motivated by inference serving latency SLOs, not throughput. The TPU's deterministic execution (no branch prediction, no speculative execution, no cache evictions) makes 99th-percentile latency much more predictable than the same workload on a GPU with preemption.

## Where it breaks

**Low arithmetic intensity workloads**: Any model that cannot batch efficiently (online serving at batch=1, attention-heavy models with tiny batch windows) leaves the matrix unit starved. Memory bandwidth becomes the binding constraint, and the chip runs at a fraction of peak TOPS.

**INT8 precision ceiling**: The TPU was designed for inference with weights already quantized to 8 bits. Training requires higher precision (FP32 or BF16); the v1 TPU cannot train. (TPU v2/v3 added BF16 and training capability.)

**Irregular computation**: Sparse weight matrices, dynamic shapes, graphs with complex branching — the systolic array's sweet spot is dense regular GEMM. Sparse attention patterns, mixture-of-experts routing, and recurrent loops with variable-length inputs do not map cleanly to its structure.

**Programming model**: There is no general GPU-like SIMT thread model. The hardware is controlled by a compiled "assembly" via a hardware abstraction layer. Mapping novel operators to the matrix unit requires explicit compiler support; arbitrary PyTorch ops cannot just run on the hardware.

**No cache hierarchy**: every memory access must be explicitly managed by the compiler's DMA schedule. This yields great performance for regular access patterns but makes irregular access patterns (dynamic sparse indexing, irregular graph traversal) extremely difficult.

## Why it works

The systolic array is **hardware loop tiling** — the same insight that makes BLAS Level 3, FlashAttention, and cache-oblivious algorithms fast, but hardwired into silicon.

The naive three-nested-loop matrix multiply:
```
for i in range(M):
  for j in range(N):
    for k in range(K):
      C[i][j] += A[i][k] * B[k][j]
```
visits each element of B[k][j] once per row of A — M times total. If B doesn't fit in registers, this causes M × (K×N) memory reads. Tiling reorganizes the loops so that a B-tile is loaded into cache and all matching A-tiles computed before evicting. The 256×256 systolic array goes one step further: it makes the tile **the register file** (the weight-holding cells), and the loop-nesting "schedule" is physically encoded in silicon wire delays. The hardware never allows the wrong access pattern.

The deeper principle is **data-movement minimization through ownership**: give compute units exclusive, pre-loaded ownership of a tile of data, and stream all dependent inputs to those compute units. The same principle underlies:

- CPU register files (compiler owns register allocation, minimizing DRAM traffic)
- SRAM scratchpads in GPUs (programmer controls shared memory, eliminates L1 thrashing)
- LMAX Disruptor ring buffer (pre-allocated, single-producer ownership)
- Chain replication's tail read (data moves in one direction, readers own the tail's state)

The second lesson is about **specialization vs. flexibility**. A CPU is a universal machine — it can branch, speculate, cache-evict, handle interrupts, run OS code. All of that machinery is transistors *not* doing multiply-accumulate. By removing every general-purpose feature not needed for inference (branches, L1/L2 cache, FP64, out-of-order execution), the TPU devotes ~95% of its die to actual computation. This is the design strategy behind cryptographic co-processors, video encoders, and every application-specific integrated circuit: **the right answer for a stable, well-defined workload is a narrow machine designed for it**.

## Going deeper

1. **"TPU v4: An Optically Reconfigurable Supercomputer for Machine Learning with Hardware Support for Embeddings"** (Jouppi et al., 2023) — traces how the architecture evolved to v4: BF16 training support, 3D torus interconnect, optical circuit switching for reconfigurable topology.

2. **H.T. Kung, C.E. Leiserson — "Systolic Arrays (for VLSI)"** (1978) — the original paper introducing systolic computing. Short and dense; the taxonomy of data flow patterns (weight-stationary, output-stationary, row-stationary) is laid out here.

3. **"Halide: A Language and Compiler for Optimizing Parallelism, Locality, and Recomputation in Image Processing Pipelines"** (Ragan-Kelley et al., 2013) — applies the algorithm/schedule separation that the TPU's compiler implements: separate what to compute from *how* to tile and pipeline it, the same conceptual split that makes the systolic array's data flow composable with the host compiler's scheduling.
