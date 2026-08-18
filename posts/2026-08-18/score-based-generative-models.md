---
title: "Generative Modeling by Estimating Gradients of the Data Distribution"
source: https://yang-song.net/blog/2021/score/
author: Yang Song
company: Stanford / Google Brain
date_posted: 2021-05-05
date_digested: 2026-08-18
---

# Generative Modeling by Estimating Gradients of the Data Distribution

## What's new to learn

**Score function as the sampling primitive.** Instead of learning p(x) — which requires an intractable normalizing constant — you can learn its gradient ∇_x log p(x), called the *score function*. The score points in the direction of steepest ascent in log-probability and is all you need to run Langevin dynamics and draw samples.

**Score matching as a tractable training objective.** The naïve objective — minimize the L2 distance to the true score — involves the unknowable true score. A nineteenth-century identity (Fisher / Vincent, via Hyvärinen 2005) converts it into an equivalent objective that only requires data samples, no knowledge of the density. Denoising at a given noise level is, provably, score matching at that noise level.

**The SDE unification of DDPM and NCSN.** Both Noise Conditional Score Networks (NCSN) and Denoising Diffusion Probabilistic Models (DDPM) are discretizations of a single continuous-time stochastic differential equation (SDE). The forward SDE adds noise; the reverse SDE subtracts it, and the drift correction term is exactly the score function. This SDE framework makes both approaches special cases of the same principle.

## Prerequisites

Assume you know:
- Probability distributions, Gaussian noise, basic calculus (chain rule, integration by parts)
- How neural networks are trained with gradient descent
- Basics of Monte Carlo sampling and why it's hard in high dimensions

Non-obvious prerequisites worth reviewing:
- **Langevin dynamics**: iterative sampling algorithm dx = (ε/2)∇_x log p(x)dt + √ε dW. You can converge to exact samples from p(x) with decreasing step size ε, without ever computing Z = ∫exp(-E(x))dx.
- **Itô calculus basics**: just enough to read an SDE like dx = f(x,t)dt + g(t)dW — the f·dt part is deterministic drift, the g·dW part is Gaussian noise with variance g²dt.
- **Fokker-Planck equation**: describes how the marginal distribution p_t(x) evolves over time when x follows a SDE. Not required to use the results, but essential context for *why* SDEs model diffusion correctly.

## The core idea

Generative modeling has an embarrassing fundamental problem: we want to sample from a distribution p(x) defined implicitly by a training set, but computing p(x) requires evaluating the normalizing constant Z = ∫exp(-E(x))dx, which is intractable in high dimensions.

The score function sidesteps this completely. Define:

```
s(x) = ∇_x log p(x)
```

Notice: log p(x) = -E(x) - log Z, so ∇_x log p(x) = -∇_x E(x). The intractable log Z disappears because it's a constant. The score is the gradient of the negative energy — it points toward the modes of p(x) — and you never need Z to compute it.

Now, if you have a score estimator s_θ(x) ≈ ∇_x log p(x), you can sample from p(x) via **Langevin dynamics**:

```
x_{t+1} = x_t + (ε/2) · s_θ(x_t) + √ε · z_t,   z_t ~ N(0, I)
```

Start from random noise and iteratively step in the direction the score tells you. As ε → 0, this converges to exact samples from p(x) (under mild regularity conditions). The score function is the only thing you need.

**Training the score network.** You want to minimize the *score matching objective*:

```
L_SM = E_x [ ||s_θ(x) - ∇_x log p(x)||² ]
```

But ∇_x log p(x) is unknown (it's what we're trying to learn!). Hyvärinen (2005) showed via integration by parts that this equals, up to a constant:

```
L_SM = E_x [ tr(∇_x s_θ(x)) + (1/2)||s_θ(x)||² ]
```

This is computable from data alone. No true scores needed.

**Denoising score matching (the practical shortcut).** The trace term above is expensive to compute in high dimensions. Vincent (2011) showed that if you corrupt data with noise x̃ = x + σ·ε, ε ~ N(0,I), then the score of the *noisy* distribution p_σ(x̃) = ∫p(x)N(x̃;x,σ²I)dx satisfies:

```
∇_{x̃} log p_σ(x̃) = (x - x̃) / σ² = -ε/σ
```

So training a network to predict **which noise was added** is exactly training it to estimate the score of the noisy distribution. This is the core equivalence: **denoising = score estimation.**

The DDPM objective (predict the noise ε that was added at each timestep) is therefore not a trick or engineering simplification — it is exact score matching at each noise level.

## Mechanics

### NCSN: annealing noise scales

The raw data manifold is low-dimensional inside a high-dimensional ambient space. Off the manifold, the score ∇_x log p(x) is undefined or meaningless, causing Langevin dynamics to produce garbage.

Fix: train a single *noise-conditional* score network s_θ(x, σ) on data perturbed at L discrete noise levels σ_1 < σ_2 < ... < σ_L. The training objective is:

```
L_NCSN = (1/L) Σ_i λ(σ_i) · E_x E_{ε~N(0,I)} [ ||s_θ(x + σ_i·ε, σ_i) + ε/σ_i||² ]
```

with λ(σ) = σ² to balance gradients across noise levels.

Sampling uses **annealed Langevin dynamics**: start from pure Gaussian noise at σ_L, run many Langevin steps, drop to σ_{L-1}, run more steps, and so on down to σ_1. The high-noise steps give the chain global coverage of the data space; low-noise steps refine fine details.

### DDPM: the score network in disguise

DDPM (Ho et al. 2020) defines a Markov chain forward process:

```
q(x_t | x_{t-1}) = N(x_t; √(1-β_t) x_{t-1}, β_t I)
```

This gradually converts data to Gaussian noise over T steps (typically T = 1000). The marginal is:

```
q(x_t | x_0) = N(x_t; √ᾱ_t x_0, (1-ᾱ_t) I)
```

where ᾱ_t = Π_{s≤t} (1-β_s). The reverse process is learned as:

```
p_θ(x_{t-1} | x_t) = N(x_{t-1}; μ_θ(x_t, t), σ_t² I)
```

The optimal μ_θ can be reparameterized as predicting the noise ε that was added. The loss becomes:

```
L_DDPM = E_t E_x0 E_ε [ ||ε - ε_θ(√ᾱ_t x_0 + √(1-ᾱ_t)ε, t)||² ]
```

**The equivalence:** since x_t = √ᾱ_t x_0 + √(1-ᾱ_t)ε, the true noise that was added is ε = (x_t - √ᾱ_t x_0) / √(1-ᾱ_t). Meanwhile, the score of q(x_t | x_0) with respect to x_t is -ε/√(1-ᾱ_t). So ε_θ(x_t, t) = -√(1-ᾱ_t) · s_θ(x_t, t). Predicting the noise is the score, scaled.

### The SDE unification

Song et al. (ICLR 2021) generalized both frameworks by describing the forward process as a continuous-time SDE:

```
dx = f(x, t) dt + g(t) dW
```

where W is Brownian motion. Two special cases:
- **VP-SDE** (Variance-Preserving, = DDPM limit): f(x,t) = -β(t)/2 · x, g(t) = √β(t)
- **VE-SDE** (Variance-Exploding, ≈ NCSN): f(x,t) = 0, g(t) = √(dσ²/dt)

Anderson (1982) proved that any forward SDE has a time-reversal that is also an SDE:

```
dx = [f(x,t) - g(t)² · ∇_x log p_t(x)] dt + g(t) dW̄
```

where W̄ is reverse-time Brownian motion. This reverse SDE starts at pure noise p_T ≈ N(0,I) and evolves backward in time, ending at the data distribution p_0. The only unknown quantity in the reverse SDE is the score ∇_x log p_t(x). Train a network s_θ(x, t) ≈ ∇_x log p_t(x), plug it in, discretize, and you get a generative model.

This also yields an **ODE formulation** (called the Probability Flow ODE), obtained by setting the stochastic term to zero:

```
dx/dt = f(x,t) - (1/2) g(t)² · ∇_x log p_t(x)
```

This ODE has the same marginals as the SDE. It enables deterministic sampling (no stochastic noise) and exact likelihood computation via the change-of-variables formula — features that stochastic samplers cannot provide.

### Conditioned sampling and guidance

The score can incorporate conditioning via Bayes' rule:

```
∇_x log p(x | y) = ∇_x log p(x) + ∇_x log p(y | x)
```

**Classifier guidance**: train a classifier p_φ(y | x, t) on noisy images, then add γ · ∇_x log p_φ(y | x_t, t) to the score during sampling. The weight γ trades off quality vs. diversity.

**Classifier-free guidance**: train a single model that predicts scores for both conditioned and unconditioned distributions. At sampling time, interpolate:

```
s̃_θ(x_t, t, y) = (1 + γ) · s_θ(x_t, t, y) - γ · s_θ(x_t, t, ∅)
```

This avoids training a separate classifier while achieving better image quality at high γ. All modern text-to-image systems (Stable Diffusion, DALL-E 3, Imagen) use classifier-free guidance.

## Where it breaks

**Slow sampling.** The reverse SDE or ODE requires many function evaluations (500–1000 in original DDPM). Each evaluation is a neural network forward pass over the entire image, meaning image generation takes seconds to minutes at high resolution. DDIM reduced this to ~50 steps through deterministic sampling from the ODE. Modern distillation methods (Consistency Models, SDXL-Turbo) compress to 1–4 steps but with some quality tradeoff.

**The manifold problem is never fully solved.** The trick of using multiple noise scales mitigates but does not eliminate the issue. At very low noise levels, Langevin dynamics can still get stuck in local modes and fail to mix globally. Annealing heuristics help but introduce hyperparameters.

**Mode coverage vs. mode quality.** High classifier-free guidance weight γ sharpens quality (high FID score) but collapses diversity, making the model produce a narrow range of images. This is the diffusion-model equivalent of the GAN quality-diversity tradeoff.

**Distribution shift between training and inference.** The score network s_θ(x_t, t) is trained on samples where x_t comes from the forward noising process applied to training data. At inference, x_t is generated by the reverse process, creating compounding errors. Exposure bias — the same problem that plagues autoregressive text generation.

**Score networks need architecture design work.** The UNet architecture used in most diffusion models was designed for image segmentation, not score estimation. Recent work (DiT: Diffusion Transformers) replaces UNet with transformers for better scalability.

## Why it works

**The unnormalized-density trick generalizes everywhere.** The core insight — work with ∇ log p instead of p to eliminate Z — is not unique to generative modeling. It appears in:
- Contrastive Divergence for energy-based models (Hinton 2002)
- Natural gradient descent, which uses the Fisher information (the covariance of the score)
- Stein variational gradient descent
- Kernel Stein discrepancies for goodness-of-fit testing

The score function is the *sufficient statistic* for Langevin sampling, which is why it's all you need.

**Anderson's time-reversal of SDEs is the deep principle.** The fact that any diffusion process has a time-reversal with a computable drift correction (involving only the score) was known since 1982. Song et al. recognized that this is precisely what a generative model needs: a forward process that reaches a simple distribution (noise), plus a reverse process that generates data. The score is the *only* object connecting the two.

**Denoising and density estimation are the same problem.** Tweedie's formula (Efron 2011, rediscovered by many): if x_noisy = x_clean + N(0,σ²), then:

```
E[x_clean | x_noisy] = x_noisy + σ² · ∇_{x_noisy} log p_σ(x_noisy)
```

The posterior mean of the clean signal equals the noisy signal plus σ² times the score. So the score function is the mathematical equivalent of the optimal Bayesian denoiser. Learning to denoise is literally the same computation as learning the score. DDPM did not stumble onto an algorithm that happens to work — it is a direct application of Bayesian estimation theory.

**This is Brownian motion, reversed.** The forward process is standard diffusion. The reverse is what happens to a diffusion process when you run time backwards. The score correction term in the reverse SDE is the well-known Doob h-transform from the theory of Markov processes. All of modern diffusion-based generation is the h-transform applied to Gaussian diffusion, learned from data.

## Going deeper

1. **DDIM** (Song et al., 2020, arxiv.org/abs/2010.02502) — Shows that the generative process implied by DDPM training is not uniquely the stochastic reverse Markov chain; a family of non-Markovian reverse processes with the same marginals exists, including a deterministic ODE. Deterministic sampling dramatically speeds inference and enables meaningful latent space interpolation.

2. **Consistency Models** (Song et al., 2023, arxiv.org/abs/2303.01469) — Distills the entire ODE trajectory into a single model that maps any point on the ODE trajectory directly to x_0, reducing sampling to 1–3 steps at near-original quality. Represents the next compression after DDIM, replacing iterative refinement with a learned "jump-to-start" map.

3. **Flow Matching** (Lipman et al., 2022, arxiv.org/abs/2210.02747) — Replaces the SDE framework with ordinary probability flows (ODEs with linear interpolation paths from noise to data). Faster training, simpler theory, and better scaling than DDPM-based approaches; forms the backbone of Meta's Voicebox and the Stable Diffusion 3 / FLUX family.
