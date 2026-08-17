---
title: "Reverse Mode Automatic Differentiation"
source: https://eli.thegreenplace.net/2025/reverse-mode-automatic-differentiation/
author: Eli Bendersky
company: personal blog (former Google engineer)
date_posted: 2025
date_digested: 2026-08-17
---

# Reverse Mode Automatic Differentiation

## What's new to learn

1. **Reverse-mode automatic differentiation (reverse-mode AD)**: A method to compute the gradient of a scalar-valued function with respect to all its inputs in a single backward traversal of the computation graph. This is the mechanism underlying every ML framework's `.backward()` / `grad()` call — not symbolic math, not finite differences, but a precise mechanical application of the chain rule in reverse.

2. **The VJP (Vector-Jacobian Product) as the composable primitive**: At every node in a computation graph, instead of materializing the full local Jacobian, you store a closure that accepts an incoming sensitivity ("adjoint") and returns the adjoint distributed to that node's parents. This lets the full gradient accumulate in one backward sweep.

3. **The forward/reverse mode asymmetry**: For a function ℝⁿ → ℝ¹ (n parameters, scalar loss), forward mode needs n passes while reverse mode needs exactly one. That asymmetry is structural: reverse mode propagates a single scalar back to all inputs; forward mode would propagate n separate perturbations forward.

## Prerequisites

- Multivariable calculus: partial derivatives, the chain rule, and a working sense of what a Jacobian matrix J[i,j] = ∂f_i/∂x_j represents
- Directed acyclic graphs and topological sort (why order matters when accumulating a sum over paths)
- Basic ML context: loss function, gradient descent, the difference between parameters and activations

## The core idea

Training a neural network with gradient descent requires ∂L/∂θ_i for all parameters θ_i simultaneously. A network with 175 billion parameters cannot compute gradients by finite differences (175 billion extra forward passes) or by carrying symbolic expressions through the graph (expressions explode exponentially from reuse).

The insight is this: **run the forward computation once, recording every operation in a directed graph. Then traverse that graph backward exactly once, propagating "how much the loss changes per unit change here" from the output node back to every input.** The result is the full gradient in at most a small constant times the forward pass cost.

This is reverse-mode automatic differentiation. It is not an approximation. It computes the exact gradient. It is what `loss.backward()` does in PyTorch, what `jax.grad(f)(x)` does in JAX, and what `tape.gradient()` does in TensorFlow.

## Mechanics

### Step 1: Forward pass — build the computation graph

Run the network normally. At every primitive operation, record:
- The operation type and its inputs
- Any intermediate value the backward VJP will need

For a toy example — `L = (x + y) * z`:

```
v1 = x + y     [op: add; parents: x, y]
L  = v1 * z    [op: mul; parents: v1, z; cached: (v1=x+y, z)]
```

### Step 2: Initialize the adjoint of the output

Define the adjoint of any node v as:

```
v̄  =  ∂L / ∂v
```

Start with L̄ = 1 (by definition, ∂L/∂L = 1).

### Step 3: Backward pass — accumulate adjoints in reverse topological order

For each node v in reverse order, distribute v̄ to each parent u using the **VJP rule for that operation**:

| Operation     | Forward       | VJP: how ū accumulates                     |
|---------------|---------------|---------------------------------------------|
| v = a + b     | v̄ given       | ā += v̄ &nbsp; &nbsp; b̄ += v̄              |
| v = a * b     | v̄ given       | ā += v̄ · b &nbsp; &nbsp; b̄ += v̄ · a     |
| v = ReLU(a)   | v̄ given       | ā += v̄ · 𝟙[a > 0]                          |
| v = W x       | v̄ given       | W̄ += v̄ xᵀ &nbsp; &nbsp; x̄ += Wᵀ v̄      |
| v = softmax(a)| v̄ given       | ā += (diag(v) − vvᵀ) v̄                    |
| v = log(a)    | v̄ given       | ā += v̄ / a                                 |

For the `(x + y) * z` example:

```
L̄ = 1

Process L = v1 * z:
  v̄1 += L̄ · z    (= z)
  z̄  += L̄ · v1   (= x + y)

Process v1 = x + y:
  x̄  += v̄1        (= z)
  ȳ  += v̄1        (= z)
```

Result: ∂L/∂x = z, ∂L/∂y = z, ∂L/∂z = x + y. All three in one backward sweep.

### Why one pass suffices

For f: ℝⁿ → ℝ¹, the Jacobian is a **1 × n row vector** (the gradient). Forward mode builds this column by column — n Jacobian-vector products. Reverse mode builds it in one shot by propagating the single output sensitivity backward.

More precisely: the chain rule says the Jacobian of f = fₖ ∘ … ∘ f₁ is Jf = Jₖ ⋅ … ⋅ J₁. Multiplying right-to-left, each factor shrinks the 1-dimensional output adjoint by one Jacobian transpose — a sequence of matrix-vector multiplies, never a matrix-matrix multiply.

The Baur-Strassen theorem (1983) formalizes this: **the gradient of any arithmetic circuit of size T can be computed in O(T) operations** — linear in the forward computation cost, independent of n.

### What frameworks actually store

Frameworks do not store full Jacobian matrices. They store one VJP closure per node:

```python
# conceptually, for v = a * b:
def mul_vjp(v_bar, a, b):
    return v_bar * b, v_bar * a   # (ā, b̄)
```

The closure captures `(a, b)` from the forward pass. During the backward pass, the engine calls each closure in reverse topological order, accumulating into leaf-node tensors (the parameters).

PyTorch's autograd engine stores this as a C++ "Node" with a `apply()` method. JAX uses a functional representation where each primitive registers a `jvp` rule and a `transpose` rule, and the AD transformation is composed symbolically.

## Where it breaks

**Memory grows with depth.** Intermediate activations must be kept alive until their VJP is called. A 96-layer transformer trained with sequence length 4096 and batch size 4 can need 60–100 GB of activation memory on top of parameter memory.
*Fix*: gradient checkpointing — store only every k-th layer, recompute the rest during the backward pass. Costs one extra forward pass; cuts memory from O(depth) to O(depth / k).

**Vanishing and exploding gradients.** Each VJP multiplies the adjoint by a local Jacobian. Chain-multiplying 96 such Jacobians with eigenvalues < 1 drives gradients to zero; eigenvalues > 1 drive them to infinity.
*Fix*: residual connections (preserve gradient magnitude via identity path), layer normalization (keep activations from saturating nonlinearities), gradient clipping.

**Dynamic control flow.** The backward pass must replay the exact same branch as the forward pass. If your model has data-dependent `if`-statements, the captured graph records which branch ran for this example.
*Fix*: eager execution (PyTorch's default) records the actual trace; JAX's `jit + lax.cond` traces through both branches symbolically.

**Second-order gradients.** Hessians are O(n²) in both space and time. Training with exact curvature information is infeasible at scale.
*Fix*: Hessian-vector products via forward-over-reverse composition (O(forward cost) per HVP); structured approximations like K-FAC; first-order methods that exploit second-order structure implicitly (Adam, Shampoo).

**Recompute vs. store tradeoff is global.** Gradient checkpointing saves memory but requires a global recomputation schedule. Getting the optimal schedule is an NP-hard problem (it is equivalent to the "pebble game" on the computation graph).

## Why it works

Reverse-mode AD is **dynamic programming on a DAG**.

The adjoint of any intermediate node v is:

```
v̄  =  Σ_{w : v→w}  (∂w/∂v) · w̄
```

This is a sum over all nodes w that v feeds into — the chain rule across all paths from v to L. The DP insight: if we process nodes in reverse topological order, every w that v feeds into is guaranteed to have its w̄ fully accumulated before we compute v̄. No recomputation, no recursion — just one pass.

More algebraically: differentiation is a linear operation on functions. The derivative of a composition is the composition of the derivatives (chain rule = functoriality of the derivative). Reverse mode evaluates this in the direction that maps one output sensitivity to all input sensitivities — the transpose (dual) of the forward direction. For the specific structure of ML (many inputs → one scalar loss), the dual direction is exponentially cheaper.

The **deeper principle**: **every differentiable computation is also a linear map from its inputs' perturbations to its output's perturbation (the Jacobian). Reverse mode transposes that linear map for free, because the computation graph is a DAG and transposing a DAG is just reversing its edges.** This is why AD works generically across all program structures: loops, recursion, conditionals — as long as the forward execution records an acyclic trace, the backward pass is just a graph traversal.

This principle unifies backpropagation with:
- **Gradient checkpointing** — trading forward-pass redundancy for memory by exploiting the DP structure
- **Flash Attention** — avoiding materializing the full attention matrix, which is the Jacobian of the attention operation
- **ZeRO** — partitioning the leaf adjoints (gradients) across workers
- **Ring AllReduce** — averaging those leaf adjoints after each worker computes its shard

## Going deeper

1. **Baydin, Pearlmutter, Radul, Siskind — "Automatic Differentiation in Machine Learning: a Survey" (2018, arXiv:1502.05767)** — the canonical survey covering forward and reverse modes, source transformation vs. operator overloading, higher-order AD, and the landscape of implementations from Autograd to Theano to TensorFlow.

2. **Linnainmaa (1970, Finn. transl. 1976) — the original reverse-mode AD paper** — where the algorithm was first described for arithmetic formulas, independently of and predating backpropagation in neural networks by 16 years. Shows the technique is a general mathematical tool, not an ML-specific trick.

3. **JAX documentation: "The Autodiff Cookbook"** (jax.readthedocs.io/en/latest/notebooks/autodiff_cookbook.html) — practical treatment of VJPs, JVPs, forward-over-reverse, and custom `jvp`/`vjp` rules. The best resource for seeing how reverse and forward modes compose to give Hessian-vector products, per-example gradients (via `vmap`), and operator-specific optimizations.
