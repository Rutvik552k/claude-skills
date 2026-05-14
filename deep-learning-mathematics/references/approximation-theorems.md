# Approximation Theorems for Neural Networks

## Overview

This reference covers the mathematical theory of function approximation by neural networks:
universal approximation theorems, depth-width tradeoffs, ReLU expressivity results, and
the curse of dimensionality. Based on Jentzen et al. (2025), Chapters on Approximation Theory.

---

## 1. Notation and Setup

### Neural Network Function Class

Define the class of neural networks with L hidden layers, widths (d_1, ..., d_L), and activation sigma:

    F(L, d, sigma) = { f: R^{d_0} -> R^{d_{L+1}} : f(x) = W_{L+1} sigma(...sigma(W_1 x + b_1)...) + b_{L+1} }

where W_l in R^{d_l x d_{l-1}} and b_l in R^{d_l}.

### Approximation Quality

For a target function f*: [0,1]^d -> R, the approximation error is:

    E_approx(F) = inf_{f in F} ||f - f*||

where ||.|| is typically the L^inf (supremum) norm, L^2 norm, or L^p norm.

### Function Spaces

- C([0,1]^d): continuous functions on the unit cube
- C^s([0,1]^d): s-times continuously differentiable functions
- W^{s,p}([0,1]^d): Sobolev space (s derivatives in L^p)
- Lip(K, [0,1]^d): K-Lipschitz functions

---

## 2. Universal Approximation Theorem (One Hidden Layer)

### Cybenko's Theorem (1989)

**Theorem**: Let sigma: R -> R be continuous and sigmoidal (i.e., sigma(t) -> 1 as t -> +inf
and sigma(t) -> 0 as t -> -inf). Then for any f* in C([0,1]^d) and any epsilon > 0,
there exist N in N, alpha_j in R, w_j in R^d, b_j in R such that:

    sup_{x in [0,1]^d} |f*(x) - sum_{j=1}^N alpha_j * sigma(w_j^T x + b_j)| < epsilon

**Proof Strategy**:
1. The set S = span{sigma(w^T x + b) : w in R^d, b in R} is a subspace of C([0,1]^d).
2. By the Hahn-Banach theorem, if S is not dense, there exists a nonzero bounded linear
   functional mu (a signed measure) that vanishes on S.
3. Show that mu vanishing on all sigma(w^T x + b) implies mu = 0 (contradiction).
4. Key step: the sigmoidal property of sigma allows approximation of indicator functions
   of half-spaces, and these generate all Borel sets.

### Hornik's Theorem (1991)

**Theorem**: The universal approximation property holds for any non-constant, bounded, continuous
activation function sigma. The sigmoidal condition is not necessary.

More precisely: if sigma is non-polynomial, then single-hidden-layer networks with sigma
are dense in C(K) for any compact K subset R^d.

### Leshno et al. (1993)

**Theorem**: A single-hidden-layer network with activation sigma is a universal approximator
if and only if sigma is not a polynomial.

This gives a complete characterization: polynomial activations fail, everything else works.

---

## 3. Quantitative Approximation Rates

### Classical Approximation (Non-Neural)

For f* in C^s([0,1]^d) (s-smooth) and an approximation method using N parameters:

    Best approximation error >= Omega(N^{-s/d})

This is the **curse of dimensionality**: the rate degrades as dimension d increases.

### Single Hidden Layer Networks

**Theorem (Barron, 1993)**: If f* has bounded first moment of the Fourier transform:

    C_f = integral_{R^d} ||omega|| |f_hat(omega)| d omega < inf

then there exists a single-hidden-layer network with N neurons such that:

    ||f_N - f*||_{L^2} <= O(C_f / sqrt(N))

Key: **this rate is independent of dimension d** (no curse of dimensionality).

**Proof sketch**:
1. Write f*(x) = integral sigma(w^T x + b) dmu(w, b) using the Fourier representation.
2. Approximate the integral by a finite sum using N random samples (Monte Carlo).
3. The variance of the Monte Carlo estimate gives the O(1/sqrt(N)) rate.

**Lower bound (Barron, 1993)**: For functions not in the Barron class, single-layer networks
require N = Omega(epsilon^{-d/s}) neurons for epsilon approximation of C^s functions.

### Deep ReLU Networks

**Theorem (Yarotsky, 2017)**: For f* in C^s([0,1]^d) (s-smooth), there exists a ReLU network
with O(log(1/epsilon)) layers and O(epsilon^{-d/s} * log(1/epsilon)) parameters that achieves:

    ||f_theta - f*||_{L^inf} <= epsilon

The depth provides a logarithmic factor improvement over shallow networks.

**Theorem (Lu et al., 2017)**: For width-(d+4) ReLU networks (fixed width), the depth required
for epsilon-approximation of Lipschitz functions on [0,1]^d is:

    L = O(epsilon^{-d})

Width above d+4 does not help asymptotically; depth is the critical resource.

---

## 4. ReLU Network Expressivity

### Piecewise Linear Functions

**Fact**: Any ReLU network computes a piecewise linear (PWL) function, and conversely,
any continuous PWL function can be represented by a ReLU network.

**Definition**: A PWL function f: R^d -> R has a finite set of linear regions {R_1, ..., R_K}
(polytopes covering R^d) such that f is affine on each R_i.

### Linear Region Counting

**Theorem (Montufar et al., 2014)**: A ReLU network with L layers and width n_l at layer l
can compute a function with at most:

    prod_{l=1}^{L-1} floor(n_l / d_0)^{d_0} * sum_{j=0}^{n_L} C(n_L, j)

linear regions, where d_0 is the input dimension.

**Simplified upper bound**: O(prod_{l=1}^L (n_l / d_0)^{d_0}) which is exponential in depth.

**Lower bound construction**: There exist networks achieving:

    Omega((n/d)^{(L-1)d} * 2^n)

linear regions, confirming that depth gives exponential expressivity.

### Depth vs. Width Tradeoff

**Theorem (Telgarsky, 2016)**: For any L >= 2, there exists a function computable by a
depth-L^2 network with O(1) width that cannot be approximated within constant error
by any depth-L network with fewer than 2^{L/2} neurons.

**Interpretation**: Depth provides exponential efficiency over width for certain function classes.

### Triangle Wave Construction

**Lemma**: The triangle (sawtooth) function g(x) = 2|x - floor(x + 1/2)| on [0,1]
can be computed by a ReLU network with 2 neurons and 1 hidden layer:

    g(x) = 2 * ReLU(x) - 4 * ReLU(x - 1/2) + 2 * ReLU(x - 1)

**Theorem**: The k-fold composition g^k = g o g o ... o g (k times) is a piecewise linear
function with 2^k linear pieces on [0,1]. This function:
- Can be computed by a ReLU network with O(k) layers and O(1) width
- Requires Omega(2^k) neurons if restricted to O(1) layers

**Proof**:
1. Each application of g doubles the number of linear pieces.
2. After k applications: 2^k linear pieces, each of length 2^{-k}.
3. A deep network: compose k copies of the 1-layer implementation of g -> depth O(k), width O(1).
4. A shallow network: must represent 2^k distinct affine pieces -> requires Omega(2^k) neurons.

---

## 5. Specific Approximation Results

### Approximation of Smooth Functions by ReLU Networks

**Theorem (Yarotsky, 2017)**: For f* in W^{s,inf}([0,1]^d) with ||f*||_{W^{s,inf}} <= 1:

    inf_{f in F(L,W,ReLU)} ||f - f*||_{L^inf} <= C * (W * L)^{-2s/d}

where W is the total number of weights and L is the depth.

**Matching lower bound**: For any ReLU network with W weights:

    sup_{f* in W^{s,inf}([0,1]^d)} inf_{f in F} ||f - f*||_{L^inf} >= c * W^{-2s/d}

**Interpretation**: The optimal rate for ReLU networks on Sobolev functions is Theta(W^{-2s/d}),
which matches classical (non-adaptive) approximation rates. The advantage of depth appears
in the constants and in the ability to exploit additional structure.

### Approximation of Polynomials

**Lemma**: The product function (x, y) -> x * y on [-1, 1]^2 can be approximated to accuracy
epsilon by a ReLU network with O(log(1/epsilon)) layers and O(log(1/epsilon)) neurons.

**Proof**: Use the identity x * y = ((x+y)^2 - (x-y)^2) / 4 and approximate the squaring
function x^2 by composing sawtooth functions:
- Define s_k(x) = x - g(x)/4^k iteratively. After k steps, ||s_k(x) - x^2|| <= 4^{-k}.
- Each step uses O(1) neurons. Total: O(log(1/epsilon)) depth and neurons.

**Consequence**: Any polynomial of degree p in d variables can be approximated to accuracy
epsilon by a ReLU network with O(p * d * log(1/epsilon)) layers.

### Approximation of Lipschitz Functions

**Theorem**: For K-Lipschitz f*: [0,1]^d -> R and a ReLU network with W weights:

    inf_{f in F} ||f - f*||_{L^inf} <= C * K * W^{-2/d}

This is the rate s = 1 (one derivative of smoothness) in the general Sobolev result.

---

## 6. Beyond Standard Approximation: Composition Structure

### Benefits of Compositional Structure

**Definition**: f* has compositional structure if it can be written as:

    f*(x) = h_L o h_{L-1} o ... o h_1(x)

where each h_l: R^{d_l} -> R^{d_{l+1}} is a "simple" function (e.g., depends on few variables).

**Theorem (Poggio et al., 2017)**: If f* has compositional structure with L levels, where
each component function is in C^s, then a deep network matching the compositional structure
achieves approximation rate:

    epsilon = O(N^{-2s/d_max})

where d_max = max_l d_l is the maximum intrinsic dimension at any level.

**Contrast**: A non-compositional approach achieves epsilon = O(N^{-2s/d}) with d being the
full input dimension. When d_max << d, compositionality breaks the curse of dimensionality.

### Connection to Feature Learning

- Shallow networks: learn a flat representation; limited by input dimension.
- Deep networks: learn hierarchical features; each layer captures structure at a different scale.
- Approximation theory supports: if the target function has hierarchical structure, deep networks are exponentially more efficient.

---

## 7. Approximation of Specific Function Classes

### Indicator Functions

**Theorem**: The indicator function 1_{[a,b]}(x) can be approximated to accuracy epsilon in L^1
by a ReLU network with O(1/epsilon) neurons and O(1) layers.

For d-dimensional axis-aligned rectangles: O((1/epsilon)^d) neurons needed.

### Radial Functions

For f*(x) = phi(||x||) where phi: R+ -> R is smooth:

    Approximation is essentially one-dimensional regardless of d.

A deep network can approximate the norm function and compose with a 1D approximation of phi.

### Oscillatory Functions

For f*(x) = sin(omega^T x) with ||omega|| = N:

A shallow network requires O(N) neurons, but a deep network can exploit the structure
of the sinusoidal through composition and approximate with O(log N) neurons.

---

## 8. Negative Results and Limitations

### No Free Lunch

**Theorem**: For any approximation method using N parameters, there exist continuous functions
on [0,1]^d for which the approximation error does not converge to zero as N -> inf
(without smoothness assumptions).

### Curse of Dimensionality is Real

For C^s functions in d dimensions, no method (including deep networks) can avoid the rate
O(N^{-s/d}) in the worst case over all C^s functions.

Deep networks can only break the curse by exploiting additional structure:
- Compositional structure
- Low intrinsic dimensionality
- Symmetries (invariances, equivariances)
- Smoothness beyond C^s (analyticity)

### Approximation vs. Optimization Gap

Universal approximation guarantees existence of good parameters but says nothing about:
1. Whether gradient-based optimization can find them
2. Whether the number of training samples is sufficient
3. Whether the solution generalizes

The approximation-optimization-generalization tradeoff (see SKILL.md) is central.

---

## 9. Summary of Key Rates

| Function Class          | Network Type     | Parameters N   | Rate               |
|------------------------|-----------------|----------------|---------------------|
| Continuous (Cybenko)    | 1 layer, sigmoid | N neurons      | Exists but no rate  |
| Barron class           | 1 layer, any     | N neurons      | O(1/sqrt(N))        |
| C^s([0,1]^d)           | 1 layer, ReLU    | N neurons      | O(N^{-2s/d})        |
| C^s([0,1]^d)           | L layers, ReLU   | W weights      | O((WL)^{-2s/d})     |
| Compositional C^s      | Matching depth   | N params       | O(N^{-2s/d_max})    |
| Lipschitz              | ReLU, any depth  | W weights      | O(W^{-2/d})         |
| PWL, K pieces          | ReLU             | Optimal depth  | Exact with O(K log K)|

---

## 10. Practical Implications

1. **Depth matters**: For structured (compositional) targets, deeper networks are exponentially
   more efficient. This justifies architectures like ResNets with 50-150+ layers.

2. **Width for smooth targets**: For uniformly smooth targets without special structure,
   wider shallow networks may be competitive. Width provides a "brute force" approach.

3. **ReLU is sufficient**: Despite being piecewise linear, ReLU networks can approximate
   smooth functions to arbitrary accuracy. The approximation is via increasingly fine
   piecewise linear interpolation.

4. **Curse of dimensionality is the enemy**: Always look for structure (compositional,
   low-dimensional manifold, symmetries) that can reduce the effective dimension.

5. **Approximation is necessary but not sufficient**: Good approximation rates only guarantee
   that the model class is expressive enough. Optimization and generalization are separate challenges.
