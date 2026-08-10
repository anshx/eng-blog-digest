---
title: "Toy Models of Superposition"
source: https://transformer-circuits.pub/2022/toy_model/index.html
author: Nelson Elhage, Tristan Hume, Catherine Olsson, Nicholas Schiefer, Tom Henighan, Shauna Kravec, Zac Hatfield-Dodds, Robert Lasenby, Dawn Drain, Carol Chen, Roger Grosse, Sam McCandlish, Jared Kaplan, Dario Amodei, Martin Wattenberg, Christopher Olah
company: Anthropic
date_posted: 2022-09-14
date_digested: 2026-08-10
---

# Toy Models of Superposition

## What's new to learn

1. **Superposition**: A neural network with *m* neurons can represent *n > m* features simultaneously by encoding each feature as a nearly-orthogonal direction in activation space — trading a little interference noise for the ability to store far more information than the neuron count suggests.

2. **Phase transition**: Whether a model uses superposition is not a gradual choice — there is a sharp, discontinuous threshold at which a feature flips between "assigned its own neuron" and "packed in as a direction among many." This has the structure of a first-order phase transition in physics.

3. **Polysemanticity as a structural consequence**: When features live in superposition, no single neuron aligns with any individual feature; every neuron is therefore a linear mixture of many features. Polysemanticity (neurons responding to seemingly unrelated concepts) is not a design flaw or a coincidence — it is the direct geometric consequence of superposition.

## Prerequisites

- What a neural network is: forward pass, ReLU, gradient descent. You don't need to know transformers.
- Basic linear algebra: dot product, norm, orthogonality. A projection is just `(a · b) / |b|`.
- Optional but enriching: the compressed-sensing / Johnson-Lindenstrauss intuition — that high-dimensional spaces can accommodate many nearly-orthogonal vectors.

## The core idea

The **linear representation hypothesis** holds, with substantial empirical support, that neural networks encode concepts as directions in activation space rather than as individual neurons. "Dog" might be the vector `(0.3, -0.7, 0.5, …)`, not neuron #42.

If that is true, and if the number of meaningful concepts exceeds the number of neurons, the model faces a packing problem: can it represent *n* feature-directions in *m* dimensions when *n > m*? Naively this is impossible — orthogonality requires *n ≤ m* for fully non-interfering directions.

The paper's central insight is that it **is** possible if features are **sparse**: if most features are zero most of the time, then two randomly chosen active features rarely co-occur. The interference — the dot-product cross-talk between directions — matters only when both features are simultaneously non-zero. Make the features sparse enough, and you can tolerate a little pairwise interference in exchange for representing far more features.

The model thus learns to pack features into superposition: representing each as a unit vector in R^m, accepting that these vectors are not fully orthogonal, and relying on sparsity to keep the expected interference manageable.

## Mechanics

**The toy setup.** A one-hidden-layer ReLU autoencoder with *m* neurons and synthetic inputs of *n* features:

```
x  ∈ R^n          (input; n sparse binary features, each 0 or 1)
h  = W^T x         (hidden layer; h ∈ R^m, m << n)
x̂ = ReLU(W h + b) (reconstruction attempt; b ∈ R^n)
```

*W* has shape *(m × n)*. Column *i* of *W* is the direction used to represent feature *i*. The bias *b_i* acts as a detection threshold.

**The loss.** Features have importances *S_i* (higher = more important to reconstruct accurately):

```
L = Σ_i  S_i · E[(x_i - x̂_i)²]
```

**What the model learns.** With *m = 1* neuron and *n = 2* features of equal importance and moderate sparsity, gradient descent places the two feature directions at opposite ends of the 1D axis — antipodal points on a line, the optimal packing. With *m = 2* neurons and *n = 5* features, the model arranges them as the vertices of a regular pentagon in the 2D plane. With *m = 3* and *n = 6* features, they form the vertices of a regular octahedron. The geometry that emerges is exactly the solution to the Tammes problem: how to place *n* points on the surface of a sphere in *m* dimensions to maximize their minimum pairwise separation.

**The interference formula.** The reconstruction error for feature *i* due to feature *j* being simultaneously active scales as *(W_i · W_j)²*. For a regular polygon with *n* vertices, this is *cos²(2π/n)*. For a pentagon (n=5), each pair interferes at roughly 10% level — small enough that the feature is worth representing despite the noise.

**The phase transition.** Consider increasing the sparsity of features (fewer are active on any given input). At first the model refuses to superpose — it represents only *m* features cleanly. Then, at a critical sparsity threshold, a phase transition occurs: the model abruptly switches to representing all *n* features via superposition. Plotting feature "weight" versus sparsity shows a discontinuous jump, with hysteresis — the behavior of a first-order physical phase transition, not a smooth crossover.

You can track each feature's "dimensionality" — how much of its own neuron it occupies versus how much it is mixed with others. In the no-superposition phase this is close to 1.0; in the superposition phase it drops below *1/m* for many features simultaneously.

**Polysemanticity.** In superposition, each neuron's activation is a linear projection of multiple feature directions. If you ask "what input pattern maximally activates neuron *j*?", the answer is a mix of several features — the neuron is polysemantic by construction. This is not a corruption of the model's "intended" behavior; it is the model's encoding strategy for fitting more information into fewer neurons.

## Where it breaks

- **It is a toy.** The model is a single-layer linear autoencoder with ReLU; real transformers have attention, depth, and nonlinear composition. The mapping from toy insights to production models is speculative, though later Anthropic work (Scaling Monosemanticity, Biology of an LLM) provides supporting evidence.

- **Continuous features.** Real feature values are not binary; they follow complicated distributions. The sparsity argument generalizes qualitatively but the exact phase-transition threshold shifts.

- **Superposition need not be uniform.** Important features may avoid superposition (each getting a dedicated neuron) while less important ones are packed in. The paper verifies this but the exact assignment policy is complex and task-dependent.

- **Causality is unclear.** Observing that a network uses superposition doesn't tell you whether removing it would hurt performance, or which specific superposed features actually matter for a given behavior.

- **Interference is not always small.** At very long contexts or very high feature co-occurrence rates, the expected cross-talk can grow large enough to degrade predictions in subtle ways, and there is no known bound on when this becomes a practical problem.

## Why it works

The core mathematics is the same as **compressed sensing** (Donoho 2006, Candès et al. 2006). Compressed sensing says: if a signal has at most *k* non-zero components in some basis, you can reconstruct it exactly from only *O(k log n)* measurements taken in a random basis, as long as the measurement matrix satisfies the Restricted Isometry Property (RIP). A random matrix's columns are approximately orthogonal to each other with high probability — this is the Johnson-Lindenstrauss lemma.

Superposition is the **learned** version of compressed sensing. The weight matrix *W* is not random; it is chosen by gradient descent to minimize reconstruction loss. But gradient descent discovers the same solution that random matrices approximate: pack many sparse feature directions into few dimensions, accepting interference, because sparsity makes interference rare.

The phase transition maps onto the compressed sensing threshold. Below the critical sparsity (signals too dense), reconstruction from a lossy measurement scheme is too noisy — the model falls back to representing only *m* features cleanly. Above it (signals sparse enough), the compression gain outweighs the interference cost, and the model switches abruptly to superposition.

The geometric insight — that optimal packings are Platonic-solid-like structures — comes from sphere-packing theory. The angular distance between two feature directions determines interference; maximizing the minimum pairwise angular distance (the Tammes problem) minimizes worst-case interference. The regular polygon (2D) and Platonic solids (3D) are the exact solutions to Tammes for small *n*, which is precisely what the toy models converge to.

Broader CS analogy: just as the B-tree packs more keys into fewer disk pages by accepting variable-length records, and as LZ compression packs more text into fewer bytes by building a shared dictionary, superposition packs more features into fewer neurons by exploiting the sparsity of natural data. The unifying principle is: **exploit the statistical structure of the source to compress beyond the naïve limit.**

## Going deeper

1. **"Scaling Monosemanticity: Extracting Interpretable Features from Claude 3 Sonnet"** (Templeton et al., Anthropic, 2024) — trains a sparse autoencoder (SAE) on a residual stream of a production LLM and recovers 34 million interpretable features including multimodal, multilingual, and safety-relevant ones. Confirms that superposition is not just a toy phenomenon. https://transformer-circuits.pub/2024/scaling-monosemanticity/index.html

2. **"An Introduction to Circuits"** (Olah et al., Anthropic, 2020) — the prior work that established the framework: features as directions, circuits as computational sub-graphs, universality of features across networks. The conceptual foundation for Toy Models. https://transformer-circuits.pub/2020/circuits/index.html

3. **"Compressed Sensing: From Theory to Applications"** (Eldar & Kutyniok, 2012) — the textbook connecting the sparse-recovery math to the neural-network intuition. Chapter 1 covers the RIP and Johnson-Lindenstrauss concisely. Worth reading alongside the toy model paper to make the analogy precise.
