---
title: "AWQ: Activation-Aware Weight Quantization for LLM Compression and Acceleration"
source: https://arxiv.org/abs/2306.00978
author: Ji Lin, Jiaming Tang, Haotian Liu, Shang Yang, Xingyu Dang, Chuang Gan, Song Han
company: MIT / MIT-IBM Watson AI Lab
date_posted: 2023-06-01
date_digested: 2026-08-21
---

# AWQ: Activation-Aware Weight Quantization for LLM Compression and Acceleration

## What's new to learn

1. **Weight-only quantization as a memory-bandwidth play**: In autoregressive LLM decoding, the bottleneck is not FLOPS but bytes-per-second — you must load all model weights from HBM for every generated token regardless of batch size. INT4 weights transfer 4× faster than FP16, which is why quantization is primarily an inference throughput technique, not just a compression one.

2. **Salient channel identification via activation magnitude**: Only ~1% of weight channels cause the bulk of INT4 quantization error, and these channels are identifiable not by their own magnitude (weight magnitude ≠ impact) but by the magnitude of the activations they are multiplied by — high-activation channels propagate their quantization error loudly into the output.

3. **Error-equivalence scaling**: Instead of keeping salient channels in FP16 (mixed-precision, which requires non-uniform memory layout), you can multiply the weight channel by a scalar `s` and divide the corresponding activation by `s`. The math is identical, but you quantize the scaled weight instead of the original — and with appropriate group-wise quantization, this reduces the relative error on salient channels while staying hardware-friendly.

## Prerequisites

- **Transformer forward pass**: tokens → embeddings → attention + FFN layers → logits; each layer multiplies a weight matrix by an activation vector.
- **IEEE 754 quantization basics**: mapping floats to N-bit integers with a scale+zero-point; step size Δ ≈ max_range / 2^N; rounding error bounded by ±Δ/2.
- **Group quantization**: dividing a weight matrix row into groups of g (e.g., 128) columns, each with its own scale. Standard in LLM quantization because it amortizes scale overhead while giving per-group dynamic range.
- **Memory-bandwidth bound inference**: LLM decode with batch size 1 computes exactly one FLOP per weight byte loaded; any GPU's arithmetic throughput far exceeds its HBM bandwidth at this batch size.

## The core idea

Naïve round-to-nearest (RTN) quantization degrades LLM quality sharply below W8 (8-bit weights) because it treats every weight as equally important. It is not. The output error from quantizing weight matrix W is:

```
ε = (W - Quant(W)) @ x
```

For group quantization with step size Δ_g, the contribution of input channel `i` is:

```
|ε_i| ≤ (Δ_g/2) · |x_i|
```

The step size is shared within a group, but the activation magnitudes `|x_i|` vary by orders of magnitude across channels. A channel with `|x_i| = 100` amplifies its weight quantization error 100× more than one with `|x_i| = 1`. Yet RTN assigns the same step size and rounds both the same way, wasting bits on unimportant channels and mangling important ones.

AWQ's fix has two parts:

**Identify**: run a few hundred calibration samples through the model; record average `|x_i|` per input channel. The top ~1% of channels by this score are "salient" — protect them.

**Transform, don't isolate**: For each linear layer, define per-channel scales `s_i`. Apply the transformation:

```
W_new[:, i]  = W[:, i] * s_i     (scale up the weight column)
x_new[i]     = x[i] / s_i        (scale down the activation)
```

This is algebraically identical: `W_new @ x_new = W @ x`. But now, because salient channels have large `s_i`, their weight column has larger absolute values. Within a quantization group that contains a salient channel, the group's scale Δ_g is set by the max weight in the group — which is now dominated by the scaled salient channel. This means:

- The salient channel gets quantized with a step size matched to its rescaled magnitude — precise representation.
- Non-salient channels get a slightly larger step size (fewer effective bits), but since their `|x_i|` is small, their error contribution to the output is negligible.

The scales `s_i` are not stored alongside the quantized weights for runtime use — instead, they are *folded* into the preceding layer (absorbed into a LayerNorm scale or previous linear layer's output). At inference time, the network is exactly the original network except that certain weight columns have been pre-scaled before quantization.

The optimal `s_i` are found by grid search over `α ∈ [0, 1]`:

```
s_i = mean_activation_magnitude(channel_i)^α
```

`α = 0` gives no scaling (RTN); `α = 1` gives full activation-proportional scaling. AWQ sweeps α in steps of 0.1, selecting the value that minimizes the L2 reconstruction error on calibration data for each layer.

## Mechanics

**Step-by-step quantization pipeline:**

1. **Calibration data collection**: pass 128–512 random samples from a representative dataset through the model in FP16. Record per-input-channel activation magnitudes for each linear layer, averaged across tokens and samples.

2. **Scale search per layer**: for each linear layer, sweep `α` from 0 to 1. For each α:
   - Compute `s_i = |x_i|^α` for all input channels
   - Scale the weight matrix: `W' = W * diag(s)`
   - Quantize `W'` using RTN with group size g (typically 128)
   - Dequantize and compute reconstruction error against FP16 `W @ x`
   - Keep the `α` with minimum reconstruction error

3. **Scale absorption**: the scale `diag(s)` must be removed from the inference path since it was baked into the quantized weights. Common approach:
   - If the previous layer is LayerNorm: divide the LayerNorm scale parameter by `s` element-wise
   - If the previous layer is a linear layer: divide its weight rows by `s`
   - Either way, the overall computation `LayerNorm_out → linear_in → W` is unchanged

4. **Quantization storage**: weights stored as INT4 (two 4-bit values per byte) with per-group FP16 scale and zero-point. At inference, dequantize on the fly to FP16 before matrix multiply — or use fused INT4 matrix-multiply kernels that operate directly on packed 4-bit weights.

**Memory layout**: a 7B-parameter model in FP16 requires ~14 GB; in INT4 with per-group scaling overhead (~0.5%), ~3.6 GB. This lets a model that required two A100s fit on one, or a 70B model fit on a single 80GB A100 instead of needing 4.

**Kernel implementations**: NVIDIA's TensorRT-LLM ships AWQ kernels that dequantize INT4 weights to FP16 on-the-fly inside the GEMM kernel using Tensor Core instructions, hiding the dequantization latency inside the arithmetic pipeline. vLLM and llama.cpp have similar implementations.

## Where it breaks

**Activations remain FP16**: AWQ is weight-only quantization. Activations (the KV cache, attention scores, intermediate FFN activations) stay FP16. To quantize activations too, you need methods like SmoothQuant that separately handle the activation distribution. Full W4A8 (4-bit weights, 8-bit activations) is harder because activation distributions have outliers that are less predictable.

**Small models hurt more**: Quantization imposes a fixed absolute error floor. At 7B parameters, the weight matrices are large enough that per-group scaling (128-element groups) gives fine-grained range adaptation. Below 1–2B parameters, quality degradation at W4 becomes noticeable even with AWQ.

**Task sensitivity**: tasks requiring precise numerical outputs (arithmetic, code generation with exact constants, structured data extraction) are more sensitive to quantization errors than tasks like open-ended generation or summarization. AWQ's calibration on general text may not catch task-specific error concentrations.

**The grid search is a heuristic**: AWQ finds scales by minimizing layer-wise reconstruction error, not end-to-end loss. There can be interactions between layers (error in one layer affects what activations the next layer sees). The grid search also only explores power-law scales, not arbitrary per-channel scales. GPTQ's Hessian-based approach is theoretically more principled at finding per-weight optimal rounding but is slower and can be brittle to calibration data.

**Quantization groups mix salient and non-salient channels**: the scale folding works per-input-channel but the group quantization scale is per-group. If a group contains both salient and non-salient channels, the salient channel's scaling inflates Δ_g for all channels in the group, actively hurting the non-salient ones. This is a real tradeoff — mitigated but not eliminated.

## Why it works

The deeper principle is **importance-weighted error allocation**: given a fixed error budget (determined by the number of bits), concentrate precision where errors cost the most, and relax it where errors are cheap.

AWQ implements this not by varying bit-width per channel (which breaks hardware alignment), but by an algebraically equivalent re-parameterization — scale the data so that "important" channels appear large to the fixed-width quantizer, which then allocates them proportionally more precision through a larger step-size-to-range ratio. This is the same insight as:

- **Importance sampling** in Monte Carlo integration: allocate more samples to regions with high variance × probability, renormalize.
- **Non-uniform quantization** in audio codecs (µ-law, A-law): map the input through a non-linear function before uniform quantization, so small values (where the ear is more sensitive) get more levels.
- **Perceptual image compression** (JPEG): quantize in DCT space with a perceptual weighting matrix that allocates more bits to low-frequency components (where human vision is sensitive) and fewer to high-frequency noise.
- **Attention mechanisms**: softmax-weighted aggregation is just importance-weighted averaging; channels (queries) with high attention score contribute proportionally more to the output.

All of these are instances of the same principle: **if you cannot allocate resources uniformly, measure importance and pre-transform the space so the uniform allocator inadvertently becomes non-uniform**.

The specific trick of using an algebraically equivalent rescaling to avoid hardware-unfriendly mixed-precision layouts is itself generalizable: "absorb the non-uniformity into a preceding scale-and-shift that can be fused into an adjacent operation" appears in LayerNorm fusion, weight tying, and model merging.

## Going deeper

- **GPTQ** (Frantar et al., 2022, arxiv.org/abs/2210.17323): the preceding technique that AWQ improves upon. Uses optimal brain surgeon (second-order Hessian) to find the minimum-error rounding for each weight, layer by layer. More computation to apply (~hours for a 70B model) but theoretically tighter error minimization.

- **SmoothQuant** (Xiao et al., 2022, arxiv.org/abs/2211.10438): extends the scaling-trick idea to *activation* quantization by migrating quantization difficulty from activations to weights. The same per-channel scaling principle, but applied across the W/X boundary so both can be INT8 — enabling W8A8 Tensor Core acceleration, which is faster than W4A16 on some hardware.

- **AutoAWQ library** (github.com/casper-hansen/AutoAWQ): the production implementation of AWQ for Hugging Face models. Implements the calibration search, scale folding, and INT4 GEMM kernels. Understanding its `quantize()` method and how it handles each layer type (attention projections, FFN, embedding) concretizes everything above.
