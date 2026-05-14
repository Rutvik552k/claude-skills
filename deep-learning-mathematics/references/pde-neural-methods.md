# Neural Network Methods for Partial Differential Equations

## Overview

This reference covers deep learning approaches for solving partial differential equations (PDEs):
Physics-Informed Neural Networks (PINNs), the Deep Galerkin Method (DGM), and the Deep Kolmogorov
Method. Based on Jentzen et al. (2025), Chapters on Deep Learning for PDEs.

---

## 1. Problem Setting

### General PDE Form

Consider a PDE on domain Omega subset R^d with boundary dOmega:

    L[u](x) = f(x)         for x in Omega        (PDE)
    B[u](x) = g(x)         for x in dOmega       (boundary condition)

where:
- L is a differential operator (e.g., Laplacian, advection-diffusion)
- B is a boundary operator (Dirichlet, Neumann, Robin)
- u: Omega -> R is the unknown solution

### Time-Dependent PDEs

    du/dt + L[u](t, x) = f(t, x)     for (t, x) in [0, T] x Omega
    u(0, x) = u_0(x)                   (initial condition)
    B[u](t, x) = g(t, x)              for x in dOmega

### Why Neural Networks for PDEs?

**Curse of dimensionality**: Mesh-based methods (FEM, FDM, FVM) require O(N^d) grid points
in d dimensions. For d > 3, this is computationally intractable.

Neural networks offer:
- Mesh-free formulation (no grid required)
- Potential to scale to high dimensions (d = 100+)
- Automatic differentiation for computing derivatives
- Flexible architecture choices

---

## 2. Physics-Informed Neural Networks (PINNs)

### Architecture

The solution u is approximated by a neural network u_theta: R^d -> R:

    u_theta(x) = W_L * sigma(W_{L-1} * sigma(... sigma(W_1 * x + b_1)...) + b_{L-1}) + b_L

Typical choices:
- Activation: tanh (smooth, enabling higher-order derivatives), sin (Fourier features), or swish
- Depth: 4-8 hidden layers
- Width: 20-100 neurons per layer
- Input: spatial coordinates (and time for evolution equations)

### Loss Formulation

The PINN loss combines three terms:

**PDE Residual Loss**:

    L_PDE(theta) = (1/N_r) * sum_{i=1}^{N_r} |L[u_theta](x_r^i) - f(x_r^i)|^2

where {x_r^i}_{i=1}^{N_r} are collocation points sampled in the interior of Omega.

**Boundary Condition Loss**:

    L_BC(theta) = (1/N_b) * sum_{i=1}^{N_b} |B[u_theta](x_b^i) - g(x_b^i)|^2

where {x_b^i}_{i=1}^{N_b} are points on the boundary dOmega.

**Data Loss** (if observational data available):

    L_data(theta) = (1/N_d) * sum_{i=1}^{N_d} |u_theta(x_d^i) - u_obs^i|^2

**Total Loss**:

    L(theta) = lambda_PDE * L_PDE + lambda_BC * L_BC + lambda_data * L_data

### Computing PDE Residuals

Key advantage: automatic differentiation (AD) computes exact derivatives of u_theta.

For the Laplacian Delta u = sum_{i=1}^d d^2u/dx_i^2:
1. Forward pass: compute u_theta(x)
2. First derivatives: du/dx_i via AD (backward mode)
3. Second derivatives: d^2u/dx_i^2 via AD applied to du/dx_i

Cost: O(d) times the cost of a forward pass for second derivatives.

### Collocation Point Sampling

**Uniform sampling**: x_r ~ Uniform(Omega). Simple but may miss important regions.

**Latin Hypercube Sampling (LHS)**: Stratified sampling ensuring uniform marginal distributions.
Better coverage than pure random sampling.

**Residual-based adaptive sampling**: Sample more points where |L[u_theta](x) - f(x)| is large.
1. Train initially with uniform points
2. Evaluate residual on a dense grid
3. Add points where residual is largest
4. Retrain with augmented point set

**Sobol sequences**: Low-discrepancy quasi-random sequences for better space-filling in high d.

### Training Algorithm

1. Initialize network parameters theta
2. For each training iteration:
   a. Sample (or resample) collocation points {x_r}, boundary points {x_b}
   b. Compute u_theta(x) and its derivatives via AD
   c. Evaluate L_PDE, L_BC, L_data
   d. Compute total loss L(theta) = lambda_PDE * L_PDE + lambda_BC * L_BC + lambda_data * L_data
   e. Update theta via Adam or L-BFGS
3. Repeat until convergence

### Loss Balancing Strategies

**Problem**: Different loss terms may have vastly different magnitudes, causing the optimizer
to focus on one at the expense of others.

**Fixed weights**: Set lambda_PDE, lambda_BC, lambda_data manually. Requires problem-specific tuning.

**Gradient-based balancing (Wang et al., 2021)**:
    lambda_i = max_j ||nabla_theta L_j|| / ||nabla_theta L_i||

Ensures all loss components have comparable gradient magnitudes.

**Self-adaptive weights (McClenny & Brainerd, 2020)**:
    lambda_i are learnable parameters updated alongside theta.

**NTK-based weighting**: Use the Neural Tangent Kernel to balance the convergence rate of each loss term.

---

## 3. PINN Variants and Extensions

### Hard Constraint Enforcement

Instead of penalizing boundary conditions in the loss, enforce them exactly:

    u_theta(x) = g(x) + d(x) * v_theta(x)

where:
- g(x) satisfies the boundary condition: B[g] = g on dOmega
- d(x) is a distance function: d(x) = 0 on dOmega, d(x) > 0 in Omega
- v_theta(x) is the neural network output (unconstrained)

**Advantage**: L_BC = 0 exactly, reducing the optimization problem to minimizing L_PDE only.

**Distance function examples**:
- For rectangular domains: d(x) = prod_i x_i * (1 - x_i)
- For circular domains: d(x) = R^2 - ||x||^2
- For arbitrary domains: approximate using a separate neural network or level set

### Fourier Feature Networks

Replace the input x with Fourier features:

    gamma(x) = [cos(2 pi B x), sin(2 pi B x)]

where B in R^{m x d} is a frequency matrix (random Gaussian or learned).

**Benefit**: Overcomes the spectral bias of standard neural networks (tendency to learn
low-frequency components first). Enables learning high-frequency solutions.

### Multi-Scale PINNs

For problems with features at multiple scales:
1. Use multiple networks, each responsible for a different scale
2. Sum their outputs: u_theta(x) = sum_k u_theta_k(x)
3. Each network uses Fourier features at a different frequency band

### Temporal Decomposition

For time-dependent PDEs on long time intervals:
1. Divide [0, T] into sub-intervals [t_{k-1}, t_k]
2. Train a PINN on each sub-interval sequentially
3. Use the terminal value from interval k as the initial condition for interval k+1

This avoids the difficulty of training a single network over a long time horizon.

---

## 4. Deep Galerkin Method (DGM)

### Formulation

The DGM (Sirignano & Spiliopoulos, 2018) approximates the PDE solution by minimizing:

    J(theta) = integral_Omega |L[u_theta](x) - f(x)|^2 dx + lambda * integral_{dOmega} |B[u_theta](x) - g(x)|^2 dS(x)

### Monte Carlo Approximation

Replace integrals with Monte Carlo estimates:

    J_N(theta) = (1/N_r) sum_{i=1}^{N_r} |L[u_theta](x_r^i) - f(x_r^i)|^2 * |Omega|
               + (lambda/N_b) sum_{i=1}^{N_b} |B[u_theta](x_b^i) - g(x_b^i)|^2 * |dOmega|

where x_r^i ~ Uniform(Omega) and x_b^i ~ Uniform(dOmega).

### DGM Architecture

The original DGM paper proposes a specialized architecture with highway-like connections:

    S_1 = sigma(W_1 x + b_1)
    Z_l = sigma(W_z^l x + U_z^l S_l + b_z^l)
    G_l = sigma(W_g^l x + U_g^l S_l + b_g^l)
    R_l = sigma(W_r^l x + U_r^l S_l + b_r^l)
    S_{l+1} = (1 - G_l) . R_l + G_l . Z_l
    f(x) = W_out S_L + b_out

This architecture helps with gradient flow for deep networks solving PDEs.

### Comparison with PINNs

| Aspect              | PINNs                         | DGM                            |
|---------------------|-------------------------------|--------------------------------|
| Loss formulation    | Point-wise residual (sum)     | Integral residual              |
| Sampling            | Fixed or adaptive points      | Fresh random samples each step |
| Architecture        | Standard MLP                  | Highway/LSTM-like              |
| Optimization        | Adam + L-BFGS                 | SGD/Adam                       |
| Boundary handling   | Penalty or hard constraint    | Penalty (integral form)        |

---

## 5. Deep Kolmogorov Method

### Feynman-Kac Connection

For the parabolic PDE:

    du/dt + (1/2) trace(sigma sigma^T H_u) + mu^T nabla u = 0
    u(T, x) = g(x)

The solution has a stochastic representation (Feynman-Kac formula):

    u(t, x) = E[g(X_T) | X_t = x]

where X satisfies the SDE:

    dX_s = mu(s, X_s) ds + sigma(s, X_s) dW_s,    X_t = x

### Algorithm

1. Simulate M paths of the SDE from time t to T:
   X_s^{(m)} for m = 1, ..., M, using Euler-Maruyama discretization:
   X_{s+h}^{(m)} = X_s^{(m)} + mu(s, X_s^{(m)}) h + sigma(s, X_s^{(m)}) sqrt(h) Z^{(m)}
   where Z^{(m)} ~ N(0, I_d)

2. The loss function:
   L(theta) = (1/M) sum_{m=1}^M |u_theta(t, X_t^{(m)}) - g(X_T^{(m)})|^2

3. Minimize over theta using SGD/Adam.

### Deep BSDE Method (Han, Jentzen, E, 2018)

Solve the backward SDE (BSDE) associated with the PDE:

    Y_t = g(X_T) - integral_t^T f(s, X_s, Y_s, Z_s) ds - integral_t^T Z_s dW_s

Discretize and parameterize Z_t at each time step with neural networks:

    Y_{t_{n+1}} = Y_{t_n} - f(t_n, X_{t_n}, Y_{t_n}, Z_{t_n}) * Delta t + Z_{t_n}^T * Delta W_n

where Z_{t_n} = phi_{theta_n}(X_{t_n}) is a neural network at time step n.

**Loss**: L(theta) = E[|Y_T - g(X_T)|^2] (terminal condition mismatch)

**Advantage**: Scales to very high dimensions (d = 100+) because:
- No spatial grid is needed
- Monte Carlo sampling is dimension-independent
- Each time step uses a relatively small network

### Convergence Theory

**Theorem (Han, Long, 2020)**: Under regularity assumptions on the PDE coefficients:

    E[|u_theta(0, X_0) - u(0, X_0)|^2] <= C * (1/M + h + epsilon_approx)

where M is the number of Monte Carlo paths, h is the time step, and epsilon_approx
is the neural network approximation error.

---

## 6. Applications

### Option Pricing (Black-Scholes)

**Problem**: Price a European basket option on d assets.

PDE (Black-Scholes in d dimensions):

    du/dt + (1/2) sum_{i,j} rho_{ij} sigma_i sigma_j S_i S_j d^2u/(dS_i dS_j)
          + sum_i r S_i du/dS_i - r u = 0

Terminal condition: u(T, S) = max(max_i S_i - K, 0)  [rainbow option]

**Neural network approach**:
- Deep Kolmogorov: simulate stock price paths, train network on terminal payoff
- PINN: formulate loss with PDE residual on the domain [0,T] x R_+^d
- Tested up to d = 100 assets (Han et al., 2018)

### Heat Equation

    du/dt = alpha * Delta u    on [0,T] x Omega

Initial condition: u(0, x) = u_0(x)
Boundary: u(t, x) = 0 on dOmega

**PINN formulation**:
- L_PDE = ||du_theta/dt - alpha * sum_i d^2u_theta/dx_i^2||^2
- L_IC = ||u_theta(0, x) - u_0(x)||^2
- L_BC = ||u_theta(t, x)||^2 on dOmega

Analytical solutions available for validation (e.g., Gaussian initial data).

### Allen-Cahn Equation

    du/dt = epsilon^2 * Delta u + u - u^3

A semilinear PDE with sharp interfaces. Challenging for traditional methods due to:
- Small epsilon requires very fine meshes (interface width ~ epsilon)
- Neural networks can resolve thin interfaces without mesh refinement

### Hamilton-Jacobi-Bellman (HJB) Equations

    du/dt + min_a {L_a[u] + f_a} = 0

Arising in stochastic optimal control. High-dimensional (d = state dimension).

Deep learning approach:
- Parameterize both the value function u and the optimal control a as neural networks
- Train jointly using the PDE residual and boundary/terminal conditions

### Navier-Stokes Equations

    du/dt + (u . nabla) u = -nabla p + nu Delta u
    div u = 0

PINN approach:
- Network outputs: (u_1, u_2, u_3, p) as functions of (t, x, y, z)
- PDE residual: momentum equations + incompressibility constraint
- Data: sparse velocity measurements from experiments
- Advantage: can assimilate noisy, sparse experimental data

---

## 7. Theoretical Foundations

### Approximation of PDE Solutions

**Theorem (Jentzen et al.)**: For a class of semilinear parabolic PDEs in d dimensions,
there exist neural networks with O(d * epsilon^{-c}) parameters that approximate the
solution to accuracy epsilon, where c is a constant independent of d.

**Significance**: This overcomes the curse of dimensionality — the number of parameters
grows polynomially (not exponentially) in d.

### Error Analysis for PINNs

**Theorem (Shin et al., 2020)**: For elliptic PDEs, the PINN error satisfies:

    ||u_theta - u||_{H^1(Omega)} <= C * (L_PDE^{1/2} + L_BC^{1/2} + epsilon_approx)

where epsilon_approx is the best approximation error in the network class.

**Interpretation**: If the PINN loss is driven to zero, the network converges to the true solution
(assuming the network class is rich enough).

### Convergence of the Deep BSDE Method

**Theorem (Hutzenthaler et al., 2020)**: For semilinear parabolic PDEs with globally
Lipschitz nonlinearities, the Deep BSDE method converges:

    E[|Y_0 - u(0, x)|^2] -> 0

as M -> inf, h -> 0, and network approximation error -> 0.

The convergence rate is:

    O(h^{1/2} + M^{-1/2} + epsilon_net)

---

## 8. Architecture Patterns for PDE Solving

### Modified MLP

Standard MLP with:
- Smooth activations (tanh, sin, swish) for PDEs requiring higher-order derivatives
- Skip connections for gradient flow
- Input normalization to [0, 1] or [-1, 1]

### Multi-Output Networks

For systems of PDEs (e.g., Navier-Stokes: u, v, w, p):
- Shared hidden layers (feature extraction)
- Separate output heads for each variable
- Joint training with combined loss

### Fourier Neural Operator (FNO)

Learn the solution operator (map from PDE coefficients/source to solution):

    Layer: v_{l+1}(x) = sigma(W_l v_l(x) + (K_l v_l)(x))

where (K_l v)(x) = F^{-1}[R_l . F[v]](x) is a convolution in Fourier space.

**Advantage**: Once trained, maps any input (initial condition, source term) to the solution
without retraining. Generalizes across PDE instances.

### DeepONet (Deep Operator Network)

    G(u)(y) = sum_{k=1}^p b_k(u) * t_k(y)

- Branch network b_k: processes the input function u (evaluated at sensor locations)
- Trunk network t_k: processes the output location y
- Universal approximation theorem for operators (Chen & Chen, 1995)

---

## 9. Practical Considerations

### Activation Function Choice

| Activation | Pros                            | Cons                        |
|------------|----------------------------------|-----------------------------|
| tanh       | Smooth, bounded, standard        | Vanishing gradient for deep |
| sin        | Periodic, good for oscillatory   | May create spurious patterns|
| swish      | Smooth, unbounded, self-gated    | Slightly more expensive     |
| ReLU       | Fast, simple                     | Zero second derivative!     |
| softplus   | Smooth ReLU approximation        | Slower than ReLU            |

**Warning**: ReLU has zero second derivative almost everywhere. It is unsuitable for PDEs
requiring second-order derivatives (e.g., Laplacian). Use smooth activations instead.

### Optimizer Selection

**L-BFGS**: Often preferred for PINNs due to:
- Better convergence for smooth optimization landscapes
- Second-order information helps with the ill-conditioning of PDE losses
- Typically used after initial Adam training

**Two-phase training**:
1. Phase 1: Adam (1000-10000 steps) for initial exploration
2. Phase 2: L-BFGS for fine convergence

### Scaling to High Dimensions

- Monte Carlo sampling becomes essential (deterministic grids impossible)
- Use the Deep BSDE or Deep Kolmogorov methods (exploit stochastic representations)
- Variance reduction: antithetic sampling, control variates, importance sampling
- Parallelize over Monte Carlo paths (embarrassingly parallel)

### Verification and Validation

1. **Manufactured solutions**: choose a solution u, compute f = L[u], solve with PINN, compare
2. **Convergence study**: increase N_r, N_b, network size; error should decrease
3. **Conservation laws**: check if the PINN solution satisfies known conservation properties
4. **Comparison with classical solvers**: for low-d problems, compare with FEM/FDM solutions

---

## 10. Common Challenges and Mitigations

### Challenge: Failure to Satisfy PDE

**Symptom**: L_PDE remains large while L_BC decreases.
**Cause**: Network overfits to boundary conditions, ignoring interior PDE.
**Fix**: Increase lambda_PDE, use more interior collocation points, try adaptive weighting.

### Challenge: Spectral Bias

**Symptom**: Network learns low-frequency components but misses high-frequency details.
**Cause**: Neural networks have inherent low-frequency bias (related to NTK eigenspectrum).
**Fix**: Fourier feature encoding, multi-scale architecture, curriculum learning (low to high frequency).

### Challenge: Gradient Pathology

**Symptom**: Gradients of different loss terms have very different magnitudes.
**Cause**: PDE operator creates highly non-uniform gradient flow.
**Fix**: Gradient normalization, learning rate scheduling per loss term, NTK-based balancing.

### Challenge: Causality Violation (Time-Dependent)

**Symptom**: Network does not respect the causal structure of evolution equations.
**Cause**: All time points are trained simultaneously; no temporal ordering enforced.
**Fix**: Causal training (weight early times more), temporal decomposition, time-marching schemes.

### Challenge: Sharp Gradients/Shocks

**Symptom**: Network smooths out discontinuities or sharp fronts.
**Cause**: Neural networks are smooth functions; representing discontinuities requires many parameters.
**Fix**: Adaptive refinement near discontinuities, domain decomposition, enriched basis functions.

---

## 11. Summary: Method Selection Guide

| Problem Type               | Recommended Method     | Key Advantage              |
|----------------------------|------------------------|----------------------------|
| Elliptic PDE, low-d        | PINN                   | Simple, well-studied       |
| Parabolic PDE, high-d      | Deep Kolmogorov/BSDE   | Scales with dimension      |
| Nonlinear PDE, moderate-d  | DGM                    | Flexible, integral form    |
| Operator learning           | FNO / DeepONet         | Generalizes across inputs  |
| Data assimilation           | PINN with data term    | Combines physics + data    |
| Stochastic control (HJB)   | Deep BSDE              | Natural BSDE formulation   |
