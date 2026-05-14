# Optimization and Convergence Theory for Deep Learning

## Overview

This reference covers the mathematical theory of optimization algorithms used in deep learning:
gradient descent (GD), stochastic gradient descent (SGD), momentum methods, and their convergence
guarantees. Based on Jentzen et al. (2025), Chapters on Gradient Descent and Stochastic Optimization.

---

## 1. Problem Formulation

### Empirical Risk Minimization

Given training data {(x_i, y_i)}_{i=1}^n, the empirical risk is:

    L(theta) = (1/n) * sum_{i=1}^n l(f_theta(x_i), y_i)

where:
- f_theta: R^d -> R^k is the neural network parameterized by theta in R^p
- l: R^k x R^k -> R is the loss function (e.g., squared error, cross-entropy)
- Goal: find theta* = argmin_{theta} L(theta)

### Population Risk

The true objective is the population risk:

    R(theta) = E_{(x,y) ~ P} [l(f_theta(x), y)]

We only have access to L(theta) as an approximation of R(theta).

---

## 2. Gradient Descent (GD)

### Algorithm

    theta_{k+1} = theta_k - eta_k * nabla L(theta_k)

where eta_k > 0 is the learning rate (step size) at iteration k.

### Smoothness Assumption

**Definition**: L is L-smooth if nabla L is L-Lipschitz continuous:

    ||nabla L(theta) - nabla L(phi)|| <= L * ||theta - phi||    for all theta, phi

Equivalent condition for twice-differentiable functions:

    ||H_L(theta)||_op <= L    for all theta

where H_L is the Hessian and ||.||_op is the operator norm.

### Descent Lemma

**Lemma**: If L is L-smooth, then for any theta, phi:

    L(phi) <= L(theta) + nabla L(theta)^T (phi - theta) + (L/2) ||phi - theta||^2

**Proof sketch**: Apply the fundamental theorem of calculus and the Lipschitz condition on the gradient.

### Convergence for Convex Functions

**Theorem (Convex case)**: If L is convex and L-smooth, and eta = 1/L, then:

    L(theta_k) - L(theta*) <= (L * ||theta_0 - theta*||^2) / (2k)

This gives O(1/k) convergence rate.

**Proof**:
1. By the descent lemma with phi = theta_{k+1} = theta_k - (1/L) nabla L(theta_k):
   L(theta_{k+1}) <= L(theta_k) - (1/(2L)) ||nabla L(theta_k)||^2

2. By convexity: L(theta_k) - L(theta*) <= nabla L(theta_k)^T (theta_k - theta*)
   <= ||nabla L(theta_k)|| * ||theta_k - theta*||

3. Combining and telescoping yields the result.

### Convergence for Strongly Convex Functions

**Definition**: L is mu-strongly convex if:

    L(phi) >= L(theta) + nabla L(theta)^T (phi - theta) + (mu/2) ||phi - theta||^2

**Theorem**: If L is mu-strongly convex and L-smooth with condition number kappa = L/mu, and eta = 1/L:

    ||theta_k - theta*||^2 <= (1 - 1/kappa)^k * ||theta_0 - theta*||^2

This gives linear (exponential) convergence rate O(exp(-k/kappa)).

**Proof**:
1. From the update rule and strong convexity:
   ||theta_{k+1} - theta*||^2 = ||theta_k - theta* - (1/L) nabla L(theta_k)||^2
   = ||theta_k - theta*||^2 - (2/L) nabla L(theta_k)^T (theta_k - theta*) + (1/L^2) ||nabla L(theta_k)||^2

2. Using co-coercivity (consequence of L-smoothness + strong convexity):
   nabla L(theta_k)^T (theta_k - theta*) >= (mu*L)/(mu+L) ||theta_k - theta*||^2 + 1/(mu+L) ||nabla L(theta_k)||^2

3. Substituting and simplifying gives the contraction.

### Convergence for Non-Convex Functions

**Theorem**: If L is L-smooth (not necessarily convex), bounded below by L_inf, and eta = 1/L:

    min_{k=0,...,K-1} ||nabla L(theta_k)||^2 <= (2L * (L(theta_0) - L_inf)) / K

This gives O(1/sqrt(K)) rate to an epsilon-stationary point (||nabla L|| <= epsilon) in O(1/epsilon^2) steps.

---

## 3. Stochastic Gradient Descent (SGD)

### Algorithm

    theta_{k+1} = theta_k - eta_k * g_k

where g_k is a stochastic gradient estimate, typically a mini-batch gradient:

    g_k = (1/|B_k|) * sum_{i in B_k} nabla l(f_{theta_k}(x_i), y_i)

### Properties of Stochastic Gradients

1. **Unbiasedness**: E[g_k | theta_k] = nabla L(theta_k)
2. **Bounded variance**: E[||g_k - nabla L(theta_k)||^2 | theta_k] <= sigma^2

The variance sigma^2 decreases with batch size |B|: sigma^2(B) = sigma^2 / |B|

### Convergence for Convex Functions

**Theorem**: If L is convex and L-smooth, with constant step size eta = c / sqrt(K):

    E[L(theta_bar_K)] - L(theta*) <= O(||theta_0 - theta*||^2 / sqrt(K) + sigma * ||theta_0 - theta*|| / sqrt(K))

where theta_bar_K = (1/K) sum_{k=0}^{K-1} theta_k (averaged iterate).

Rate: O(1/sqrt(K)) — slower than deterministic GD due to gradient noise.

### Convergence for Strongly Convex Functions

**Theorem**: If L is mu-strongly convex and L-smooth, with step size eta_k = 2 / (mu * (k + 1)):

    E[L(theta_k)] - L(theta*) <= O(L * sigma^2 / (mu^2 * k))

Rate: O(1/k) — compared to O(exp(-k)) for deterministic GD.

### Convergence for Non-Convex Functions

**Theorem**: If L is L-smooth, with constant step size eta = c / sqrt(K):

    min_{k=0,...,K-1} E[||nabla L(theta_k)||^2] <= O((L(theta_0) - L_inf) / sqrt(K) + sigma / K^{1/4})

Rate: O(1/K^{1/4}) to find an epsilon-stationary point.

---

## 4. Learning Rate Schedules

### Constant Learning Rate

    eta_k = eta_0

Simple but requires careful tuning. Too large -> divergence. Too small -> slow convergence.
For SGD: constant eta does not converge to the minimum; oscillates in a neighborhood of size O(eta * sigma^2).

### Step Decay

    eta_k = eta_0 * gamma^{floor(k / S)}

Reduce learning rate by factor gamma every S steps. Common choices: gamma = 0.1, S = 30 epochs.

### Cosine Annealing

    eta_k = eta_min + (eta_max - eta_min) / 2 * (1 + cos(pi * k / T))

Smoothly decays from eta_max to eta_min over T iterations. Often restarted (warm restarts).

### Linear Warmup

    eta_k = eta_target * min(1, k / k_warmup)

Linearly increase from 0 to eta_target over k_warmup steps. Stabilizes early training when
loss landscape is poorly conditioned.

### Warmup + Cosine Decay (Combined)

    eta_k = eta_target * (k / k_warmup)                          if k < k_warmup
    eta_k = eta_min + (eta_target - eta_min)/2 * (1 + cos(...))  if k >= k_warmup

Standard schedule for transformer training (Vaswani et al., 2017; Loshchilov & Hutter, 2016).

### Polynomial Decay

    eta_k = eta_0 * (1 - k/T)^p

Common choice: p = 1 (linear decay) or p = 0.5 (square root decay).

### Robbins-Monro Conditions

For SGD convergence to the true minimizer, the step sizes must satisfy:

    sum_{k=0}^{inf} eta_k = inf    and    sum_{k=0}^{inf} eta_k^2 < inf

Example: eta_k = c / (k + 1) satisfies both conditions.

---

## 5. Momentum Methods

### Classical Momentum (Polyak, 1964)

    v_{k+1} = beta * v_k - eta * nabla L(theta_k)
    theta_{k+1} = theta_k + v_{k+1}

- beta in [0, 1) is the momentum coefficient (typically 0.9)
- Accelerates convergence in directions of consistent gradient
- Dampens oscillations in directions of high curvature

**Convergence**: For mu-strongly convex, L-smooth functions with optimal beta:

    Rate: O(exp(-k * sqrt(mu/L)))    vs.    O(exp(-k * mu/L)) for vanilla GD

This is a quadratic improvement in the condition number.

### Nesterov Accelerated Gradient (NAG, 1983)

    v_{k+1} = beta * v_k - eta * nabla L(theta_k + beta * v_k)
    theta_{k+1} = theta_k + v_{k+1}

Key difference: gradient is evaluated at the "look-ahead" point theta_k + beta * v_k.

**Convergence**: For convex, L-smooth functions:

    L(theta_k) - L(theta*) <= O(L * ||theta_0 - theta*||^2 / k^2)

This is O(1/k^2) compared to O(1/k) for vanilla GD — provably optimal for first-order methods.

### Adam (Kingma & Ba, 2015)

    m_{k+1} = beta_1 * m_k + (1 - beta_1) * g_k              (first moment estimate)
    v_{k+1} = beta_2 * v_k + (1 - beta_2) * g_k^2            (second moment estimate)
    m_hat = m_{k+1} / (1 - beta_1^{k+1})                      (bias correction)
    v_hat = v_{k+1} / (1 - beta_2^{k+1})                      (bias correction)
    theta_{k+1} = theta_k - eta * m_hat / (sqrt(v_hat) + epsilon)

Default hyperparameters: beta_1 = 0.9, beta_2 = 0.999, epsilon = 1e-8.

**Properties**:
- Adaptive per-parameter learning rates
- Scale-invariant to gradient magnitudes
- Combines benefits of momentum and RMSProp
- Known convergence issues for certain problems (Reddi et al., 2018)

### AdamW (Loshchilov & Hutter, 2019)

Decouples weight decay from the adaptive gradient update:

    theta_{k+1} = theta_k - eta * (m_hat / (sqrt(v_hat) + epsilon) + lambda * theta_k)

Unlike L2 regularization in Adam, which interacts with the adaptive learning rate,
AdamW applies weight decay directly. This distinction matters in practice.

---

## 6. Kurdyka-Lojasiewicz (KL) Theory

### KL Inequality

**Definition**: A differentiable function L satisfies the KL inequality at a critical point theta*
if there exists a neighborhood U of theta*, eta > 0, and a continuous concave function
phi: [0, eta) -> [0, inf) with phi(0) = 0, phi differentiable on (0, eta), and phi' > 0, such that:

    phi'(|L(theta) - L(theta*)|) * ||nabla L(theta)|| >= 1

for all theta in U with 0 < |L(theta) - L(theta*)| < eta.

### KL Exponent

If phi(s) = c * s^{1-alpha} for alpha in [0, 1), then the KL exponent is alpha.

- alpha = 0: L has a sharp minimum (strong convexity-like)
- alpha = 1/2: corresponds to quadratic growth
- alpha -> 1: degenerate case, very flat minimum

### Convergence Under KL

**Theorem**: If L is L-smooth, satisfies KL at all critical points, and theta_k is generated by GD
with eta < 1/L, then:

1. The sequence {theta_k} has finite length: sum_{k=0}^{inf} ||theta_{k+1} - theta_k|| < inf
2. theta_k converges to a critical point theta*

**Convergence rates** depend on the KL exponent alpha:
- alpha = 0: finite convergence (reaches theta* in finitely many steps)
- alpha in (0, 1/2]: linear convergence ||theta_k - theta*|| <= C * r^k for some r in (0,1)
- alpha in (1/2, 1): sublinear convergence ||theta_k - theta*|| <= C * k^{-(1-alpha)/(2*alpha-1)}

### Applicability to Neural Networks

The KL inequality holds for:
- Real-analytic functions (by Lojasiewicz's original result)
- Semi-algebraic functions (polynomial activations, finite datasets)
- Sub-analytic functions
- Neural network losses with analytic activations and finite training data

---

## 7. Batch Normalization: Optimization Perspective

### Effect on Loss Landscape

**Theorem (Santurkar et al., 2018)**: Batch normalization makes the loss landscape significantly
smoother. Specifically, it reduces:

1. The Lipschitz constant of the loss: ||nabla L(theta_1) - nabla L(theta_2)|| is smaller
2. The variation of the loss along training trajectories
3. The "effective" condition number of the optimization problem

### Consequence for Learning Rate

Since the Lipschitz constant L is reduced, the maximum stable learning rate eta_max = 2/L is larger.
This explains why batch-normalized networks can be trained with higher learning rates.

### Gradient Propagation

Batch normalization ensures that for each layer:
- Activations have zero mean and unit variance (within each mini-batch)
- Gradients maintain appropriate magnitude across layers
- Internal covariate shift is reduced (though the original explanation is debated)

---

## 8. Initialization Theory

### Xavier/Glorot Initialization

**Motivation**: Maintain variance of activations across layers for linear or tanh networks.

For layer l with n_in inputs and n_out outputs:

    W ~ N(0, 2 / (n_in + n_out))    [normal variant]
    W ~ U(-sqrt(6 / (n_in + n_out)), sqrt(6 / (n_in + n_out)))    [uniform variant]

**Derivation**: For z = Wx where x has variance sigma^2 and W has variance sigma_W^2:
- Var(z_j) = n_in * sigma_W^2 * sigma^2
- For forward and backward propagation to preserve variance: n_in * sigma_W^2 = 1 and n_out * sigma_W^2 = 1
- Compromise: sigma_W^2 = 2 / (n_in + n_out)

### He/Kaiming Initialization

**Motivation**: Account for ReLU activation which halves the variance (E[ReLU(x)^2] = Var(x)/2).

    W ~ N(0, 2 / n_in)    [fan-in variant]
    W ~ N(0, 2 / n_out)   [fan-out variant]

**Derivation**: For ReLU networks, the forward propagation variance equation becomes:
- Var(a_l) = (1/2) * n_l * sigma_W^2 * Var(a_{l-1})
- Setting this to 1: sigma_W^2 = 2 / n_l

### Orthogonal Initialization

Initialize weight matrices as random orthogonal matrices (sampled from the Haar measure on O(n)).

**Property**: Singular values are all 1, so the Jacobian preserves norms exactly.
Particularly effective for RNNs where gradient propagation through many time steps is critical.

---

## 9. Summary Table: Convergence Rates

| Setting              | Algorithm | Rate                        | Step Size          |
|----------------------|-----------|-----------------------------|--------------------|
| Convex, L-smooth     | GD        | O(1/k)                     | eta = 1/L          |
| Str. convex, L-smooth| GD        | O(exp(-k/kappa))            | eta = 1/L          |
| Convex, L-smooth     | NAG       | O(1/k^2)                   | eta = 1/L          |
| Str. convex, L-smooth| Momentum  | O(exp(-k*sqrt(mu/L)))       | eta = 1/L          |
| Non-convex, L-smooth | GD        | O(1/sqrt(k)) for ||grad||   | eta = 1/L          |
| Convex               | SGD       | O(1/sqrt(k))               | eta = c/sqrt(k)    |
| Str. convex          | SGD       | O(1/k)                     | eta = c/k          |
| Non-convex           | SGD       | O(1/k^{1/4}) for ||grad||  | eta = c/sqrt(k)    |

---

## 10. Practical Guidelines

1. **Start with Adam/AdamW** for initial experiments; switch to SGD+momentum for final training if better generalization is needed.
2. **Use warmup** for the first 1-10% of training, especially for transformers.
3. **Learning rate finder**: increase eta exponentially, plot loss vs. eta, choose the point of steepest descent.
4. **Weight decay**: 1e-4 to 1e-2 is typical; higher values regularize more aggressively.
5. **Batch size and learning rate scaling**: when increasing batch size by factor k, scale learning rate by sqrt(k) (linear scaling also used).
6. **Gradient clipping**: clip ||g|| to a maximum value (e.g., 1.0) to prevent gradient explosion, especially in RNNs and transformers.
7. **Monitor gradient norms**: sudden spikes indicate instability; sudden near-zero values indicate vanishing gradients.
