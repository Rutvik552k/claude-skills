---
name: deep-learning-mathematics
displayName: Deep Learning Mathematics
description: Mathematical foundations of deep learning and linear algebra for ML — neural network theory, optimization, approximation, generalization, SVD/PCA, spectral methods, and PDE-based deep learning.
version: 1.0.0
tags:
  - deep-learning
  - mathematics
  - linear-algebra
  - optimization
  - neural-networks
  - svd
  - pca
  - backpropagation
  - gradient-descent
  - approximation-theory
  - generalization
author: rutvi
sources:
  - "Mathematical Introduction to Deep Learning: Methods, Implementations, and Theory (Jentzen, Kuckuck, von Wurstemberger, 2025)"
  - "Linear Algebra, Geometry, and Computation: for Computer Vision, Robotics, and Machine Learning (Gallier & Quaintance, 2025)"
---

# Deep Learning Mathematics

## When to Activate

Use this skill when:

- **Understanding neural network math** — formal definitions of architectures, weight matrices, activation functions, layer compositions
- **Designing architectures from theory** — choosing depth vs. width based on approximation results, understanding skip connections as ODE discretization
- **Optimization analysis** — proving or analyzing convergence of GD/SGD, selecting learning rate schedules, understanding loss landscapes
- **Proving convergence** — Kurdyka-Lojasiewicz inequalities, gradient flow analysis, convergence rates for convex and non-convex objectives
- **PCA/SVD applications** — dimensionality reduction, low-rank approximation, spectral methods for data analysis
- **Physics-informed neural networks** — solving PDEs with neural networks (PINNs, Deep Galerkin, Deep Kolmogorov), scientific computing applications
- **Generalization analysis** — bias-variance tradeoffs, PAC learning, VC dimension, double descent phenomena
- **Linear algebra for ML pipelines** — matrix decompositions, pseudo-inverses, spectral graph theory, quaternion rotations

---

## Topics Index

### 1. Neural Network Architectures (Formal)

**Fully Connected (FC) Networks**
- A fully connected network with L layers is a function f: R^{d_0} -> R^{d_L} defined by composition:
  f(x) = W_L * sigma(W_{L-1} * sigma(... sigma(W_1 * x + b_1) ...) + b_{L-1}) + b_L
- Weight matrices W_l in R^{d_l x d_{l-1}}, bias vectors b_l in R^{d_l}
- Activation function sigma applied component-wise
- Parameter count: sum_{l=1}^{L} (d_l * d_{l-1} + d_l)
- Realization function: maps parameter vector theta to the network function f_theta

**Convolutional Neural Networks (CNNs)**
- Convolution operator: (K * X)[i,j] = sum_{m,n} K[m,n] * X[i+m, j+n]
- Stride s: sample every s-th output position, reducing spatial dimensions by factor s
- Padding p: zero-pad input borders to control output size; "same" padding preserves dimensions
- Feature maps as tensor outputs; pooling operations for spatial downsampling
- Parameter sharing: same kernel applied across all spatial positions

**Recurrent Neural Networks (RNNs)**
- Hidden state recurrence: h_t = sigma(W_h * h_{t-1} + W_x * x_t + b)
- Output: y_t = W_y * h_t + b_y
- Backpropagation through time (BPTT): unrolling the recurrence for gradient computation
- Vanishing/exploding gradient problem: product of Jacobians across time steps
- LSTM and GRU as gated variants controlling information flow

**Residual Networks (ResNets)**
- Skip connection: y = F(x) + x, where F is the residual mapping
- Interpretation as ODE discretization: x_{l+1} = x_l + f(x_l, theta_l) approximates dx/dt = f(x,t)
- Enables training of very deep networks by addressing gradient degradation
- Connection to neural ODEs: continuous-depth limit of ResNets

### 2. Multivariable Calculus for Deep Learning

**Partial Derivatives and Gradients**
- Gradient of loss L with respect to parameters theta: nabla_theta L = (dL/dtheta_1, ..., dL/dtheta_p)
- Directional derivative: D_v f(x) = nabla f(x) . v

**Jacobian Matrices**
- For f: R^n -> R^m, the Jacobian J_f(x) in R^{m x n} with entries [J_f]_{ij} = df_i/dx_j
- Chain rule for compositions: J_{g o f}(x) = J_g(f(x)) * J_f(x)
- Critical for understanding how gradients flow through layers

**Backpropagation Derivation**
- Recursive application of the chain rule through the computational graph
- For layer l: dL/dW_l = dL/da_l * (da_l/dz_l) * (dz_l/dW_l)
- Where z_l = W_l * a_{l-1} + b_l (pre-activation) and a_l = sigma(z_l) (post-activation)
- Backward pass computes delta_l = (W_{l+1}^T * delta_{l+1}) . sigma'(z_l) recursively
- Computational cost: O(1) times the forward pass cost

**Hessian and Second-Order Information**
- Hessian matrix H_L(theta) with entries [H]_{ij} = d^2 L / (dtheta_i dtheta_j)
- Eigenvalues of the Hessian characterize local curvature of the loss landscape
- Saddle points: Hessian has both positive and negative eigenvalues
- Connection to Newton's method: theta_{k+1} = theta_k - H^{-1} nabla L(theta_k)
- Fisher information matrix as an approximation: F = E[nabla log p(y|x,theta) * nabla log p(y|x,theta)^T]

### 3. Approximation Theory

**Universal Approximation (1D, Cybenko)**
- Cybenko's Theorem (1989): For any continuous sigma (sigmoidal), the set {sum_{j=1}^N alpha_j * sigma(w_j * x + b_j)} is dense in C([0,1]) with respect to the supremum norm.
- Requires: sigma is sigmoidal, meaning sigma(t) -> 1 as t -> +inf and sigma(t) -> 0 as t -> -inf.
- Implication: a single hidden layer network can approximate any continuous function on a compact set to arbitrary accuracy, given enough neurons.
- Non-constructive: does not specify how many neurons are needed.

**Multivariate Approximation (Curse of Dimensionality)**
- For smooth functions in d dimensions, classical approximation rates scale as O(N^{-s/d}) where s is the smoothness order.
- Curse of dimensionality: number of parameters needed grows exponentially with d for classical methods.
- Deep networks can potentially break this curse for structured function classes.

**ReLU Network Expressivity**
- ReLU networks compute piecewise linear functions.
- A ReLU network with L layers and width w can produce O(w^L) linear regions.
- Depth is exponentially more efficient than width for representing certain functions.
- Exact representation: any piecewise linear function with N breakpoints can be represented by a ReLU network with O(log N) layers.

**Depth Separation Results**
- There exist functions computable by depth-(L+1) networks of polynomial size that require exponential width for depth-L networks.
- Triangle wave example: f(x) = 2*|x - floor(x + 1/2)| iterated k times requires width O(1) and depth O(k) but width O(2^k) for depth O(1).
- Theoretical justification for deep over shallow architectures.

### 4. Optimization Theory

**Gradient Flow ODEs**
- Continuous-time gradient descent: dtheta/dt = -nabla L(theta(t))
- Loss is monotonically non-increasing along gradient flow: dL/dt = -||nabla L||^2 <= 0
- Convergence analysis via Lyapunov functions

**Gradient Descent Convergence**
- Step size condition: for L-smooth functions, step size eta < 2/L ensures descent.
- Convex case: GD converges at rate O(1/k) for convex, O(exp(-k)) for strongly convex with condition number kappa = L/mu.
- Non-convex case: GD finds epsilon-approximate stationary point in O(1/epsilon^2) iterations.

**Stochastic Gradient Descent (SGD)**
- Mini-batch gradient: g_k = (1/|B_k|) sum_{i in B_k} nabla l(theta_k; x_i, y_i)
- Unbiased estimator: E[g_k] = nabla L(theta_k)
- Variance reduction techniques: SVRG, SAGA
- Learning rate schedules: constant, step decay, cosine annealing, warmup + decay
- Convergence rate: O(1/sqrt(k)) for convex, O(1/k^{1/4}) for non-convex (under standard assumptions)

**Backpropagation Algorithm**
- Forward pass: compute and cache activations a_0, a_1, ..., a_L
- Backward pass: compute gradients layer-by-layer using cached values
- Memory cost: O(sum of layer sizes) for storing activations
- Gradient checkpointing: trade O(sqrt(L)) memory for O(L) recomputation

**Kurdyka-Lojasiewicz (KL) Inequality**
- A function f satisfies KL at x* if: phi'(|f(x) - f(x*)|) * ||nabla f(x)|| >= 1
  in a neighborhood of x*, where phi is a desingularizing function.
- Implication: gradient descent converges to a critical point with finite path length.
- Applies to real-analytic functions and semi-algebraic functions (covers most neural network losses).

**Batch Normalization**
- Normalize pre-activations: z_hat = (z - mu_B) / sqrt(sigma_B^2 + epsilon)
- Learnable affine transform: y = gamma * z_hat + beta
- Smooths the loss landscape, enabling larger learning rates.
- Theoretical effect: reduces the Lipschitz constant of the loss and its gradients.

**Random Initializations**
- Xavier/Glorot: W ~ N(0, 2/(n_in + n_out)) or U(-sqrt(6/(n_in+n_out)), sqrt(6/(n_in+n_out)))
- He/Kaiming: W ~ N(0, 2/n_in), designed for ReLU activations
- Principle: maintain variance of activations and gradients across layers
- Symmetry breaking: random initialization prevents neurons from computing identical functions

### 5. Generalization Theory

**Bias-Variance Decomposition**
- Expected risk = Bias^2 + Variance + Irreducible noise
- Bias: error from approximating the true function with a restricted model class
- Variance: sensitivity of the learned model to the training data
- Deep networks: high capacity (low bias) but potentially high variance

**VC Dimension and Rademacher Complexity**
- VC dimension of a hypothesis class H: largest set that H can shatter
- For linear classifiers in R^d: VCdim = d + 1
- For neural networks: VC dimension scales with O(W * L * log(W)) where W is the number of weights
- Rademacher complexity: R_n(H) = E[sup_{h in H} (1/n) sum_{i=1}^n sigma_i * h(x_i)]
- Generalization bound: L(h) <= L_hat(h) + 2*R_n(H) + sqrt(log(1/delta)/(2n))

**PAC Learning Bounds**
- Probably Approximately Correct: with probability >= 1-delta, |L(h) - L_hat(h)| <= epsilon
- Sample complexity: n >= (1/epsilon^2) * (VCdim * log(1/epsilon) + log(1/delta))
- Uniform convergence: controls the worst-case gap over all hypotheses simultaneously

**Overparameterization and Double Descent**
- Classical regime: more parameters -> overfitting -> higher test error
- Interpolation threshold: model just fits training data perfectly
- Double descent: test error decreases again beyond the interpolation threshold
- Implicit regularization of SGD: among all interpolating solutions, SGD finds one with small norm
- Neural tangent kernel (NTK) regime: overparameterized networks behave like kernel methods

### 6. Overall Error Analysis

**Approximation + Estimation + Optimization Decomposition**
- Total error = Approximation error + Estimation (statistical) error + Optimization error
- Approximation error: inf_{f in F} ||f - f*|| (how well can the best function in F approximate f*?)
- Estimation error: ||f_hat - f_F*|| (how well does the empirical minimizer approximate the best in F?)
- Optimization error: ||f_algo - f_hat|| (how close does the algorithm get to the empirical minimizer?)

**Oracle Inequalities**
- Bound: R(f_hat) <= inf_{f in F} R(f) + C * complexity(F) / n
- Trade-off: richer F reduces approximation error but increases estimation error
- Model selection: choose F to balance these terms (structural risk minimization)

**Sample Complexity of Deep Networks**
- For ReLU networks with L layers, width w, and W total weights:
  Sample complexity scales as O(W * L * log(W) / epsilon^2) for epsilon-accurate generalization
- Depth helps: deeper networks can achieve the same approximation with fewer total parameters
- Norm-based bounds: generalization depends on weight norms, not just parameter count

### 7. Deep Learning for PDEs

**Physics-Informed Neural Networks (PINNs)**
- Residual loss: L_PDE = ||N[u_theta](x)|| where N is the differential operator
- Boundary enforcement: L_BC = ||u_theta(x) - g(x)|| on boundary
- Total loss: L = lambda_PDE * L_PDE + lambda_BC * L_BC + lambda_data * L_data
- Training: collocation points sampled in the domain; no mesh required
- Architecture: fully connected networks with smooth activations (tanh, sin)

**Deep Galerkin Method (DGM)**
- Solve high-dimensional PDEs by reformulating as optimization over neural network parameters
- Loss: L(theta) = integral ||L[u_theta](x)||^2 dx + boundary terms
- Monte Carlo sampling for integral approximation
- Scales to dimensions where mesh-based methods are infeasible (d > 3)

**Deep Kolmogorov Method**
- Exploit connection between parabolic PDEs and stochastic processes via Feynman-Kac formula
- u(t,x) = E[g(X_T) | X_t = x] where X satisfies an SDE
- Approximate the conditional expectation with a neural network
- Applications: option pricing (Black-Scholes), stochastic control

**Applications**
- Option pricing: Black-Scholes PDE in high dimensions (basket options)
- Heat equation: u_t = Delta u with various boundary conditions
- Navier-Stokes: fluid dynamics simulations
- Schrodinger equation: quantum mechanics applications

### 8. Linear Algebra Foundations

**Vector Spaces and Linear Maps**
- Vector space axioms: closure, associativity, distributivity, identity elements
- Basis: linearly independent spanning set; dimension = cardinality of basis
- Rank-nullity theorem: dim(V) = rank(T) + nullity(T) for linear map T: V -> W
- Change of basis: [T]_{B'} = P^{-1} [T]_B P where P is the change-of-basis matrix

**Matrix Decompositions**
- LU decomposition: A = LU (L lower triangular, U upper triangular); used for solving Ax = b
- QR decomposition: A = QR (Q orthogonal, R upper triangular); used for least squares
- Schur decomposition: A = Q T Q* (T upper triangular, Q unitary); exists for all square matrices
- Cholesky: A = LL* for positive definite A; half the cost of LU

**Singular Value Decomposition (SVD)**
- A = U Sigma V^T where U in R^{m x m}, Sigma in R^{m x n}, V in R^{n x n}
- U, V orthogonal; Sigma diagonal with singular values sigma_1 >= sigma_2 >= ... >= 0
- Geometric interpretation: any linear map is a rotation, scaling, then rotation
- Eckart-Young theorem: best rank-k approximation is A_k = sum_{i=1}^k sigma_i u_i v_i^T
  minimizing ||A - B||_F over all rank-k matrices B
- Applications: image compression, latent semantic analysis, pseudo-inverse computation

**Principal Component Analysis (PCA)**
- Covariance matrix: C = (1/(n-1)) X^T X (centered data)
- PCA directions = eigenvectors of C, sorted by eigenvalue magnitude
- Explained variance ratio: lambda_k / sum_i lambda_i
- Connection to SVD: if X = U Sigma V^T, then principal components are columns of V
- Dimensionality reduction: project onto top-k principal components

**Spectral Theory**
- Symmetric matrices have real eigenvalues and orthogonal eigenvectors (spectral theorem)
- Rayleigh quotient: R(x) = (x^T A x) / (x^T x); extrema are eigenvalues
- Min-max characterization: lambda_k = min_{dim(S)=k} max_{x in S} R(x)
- Positive definite: all eigenvalues > 0; equivalent to x^T A x > 0 for all x != 0

**Norms**
- Vector norms: L1 (sum |x_i|), L2 (sqrt sum x_i^2), Linf (max |x_i|)
- Matrix norms: Frobenius (sqrt sum a_{ij}^2 = sqrt sum sigma_i^2), nuclear (sum sigma_i), operator (max sigma_i)
- Dual norm: ||x||_* = sup{|x^T y| : ||y|| <= 1}
- L1 norm dual is Linf; L2 is self-dual; nuclear norm dual is operator norm
- Norm equivalence: in finite dimensions, all norms are equivalent up to constants

### 9. Advanced Linear Algebra for ML

**Pseudo-Inverses (Moore-Penrose)**
- A^+ = V Sigma^+ U^T where Sigma^+ inverts nonzero singular values
- Least squares: x = A^+ b minimizes ||Ax - b||_2
- Minimum norm solution: among all minimizers, A^+ b has smallest ||x||_2
- Properties: A A^+ A = A, A^+ A A^+ = A^+, (A A^+)^T = A A^+, (A^+ A)^T = A^+ A

**Matrix Exponential**
- exp(A) = sum_{k=0}^{inf} A^k / k!
- Properties: exp(0) = I, d/dt exp(tA) = A * exp(tA)
- Connection to neural ODEs: solution of dh/dt = A*h is h(t) = exp(tA) * h(0)
- For diagonalizable A = P D P^{-1}: exp(A) = P exp(D) P^{-1}
- Lie group connection: exp maps from Lie algebra to Lie group

**Quaternions and Rotations**
- Quaternion: q = a + bi + cj + dk with i^2 = j^2 = k^2 = ijk = -1
- Unit quaternions form SU(2), double cover of SO(3)
- Rotation by angle theta about axis u: q = cos(theta/2) + sin(theta/2)(u_1 i + u_2 j + u_3 k)
- Rotation action: v' = q v q^{-1} (v embedded as pure quaternion)
- Advantages over rotation matrices: no gimbal lock, compact (4 vs 9 parameters), smooth interpolation (SLERP)

**Haar Bases and Wavelets**
- Haar basis: piecewise constant functions on dyadic intervals
- Multiresolution analysis: nested subspaces V_0 subset V_1 subset ...
- Wavelet decomposition: signal = coarse approximation + detail coefficients at each scale
- Applications: image compression (JPEG 2000), denoising, feature extraction

**Hadamard Matrices**
- H_n: n x n matrix with entries +/-1 and H_n H_n^T = n I
- Recursive construction: H_{2n} = [[H_n, H_n], [H_n, -H_n]]
- Applications: error-correcting codes, compressed sensing, fast transforms
- Walsh-Hadamard transform: O(n log n) computation

**Spectral Graph Theory**
- Graph Laplacian: L = D - A (D degree matrix, A adjacency matrix)
- Normalized Laplacian: L_norm = D^{-1/2} L D^{-1/2}
- Properties: L is positive semi-definite; smallest eigenvalue is 0 with eigenvector proportional to 1
- Cheeger inequality: h(G)/2 <= lambda_2 <= 2*h(G), where h(G) is the Cheeger constant (edge expansion)
- Spectral clustering: partition graph using eigenvectors of L corresponding to smallest eigenvalues
- Connection to GNNs: graph convolution defined via Laplacian eigenbasis

---

## Key Concepts

### The Approximation-Optimization-Generalization Tradeoff

The central triad of deep learning theory:

1. **Approximation**: Can the model class represent the target function?
   - Controlled by architecture choice (depth, width, activation)
   - Universal approximation says "yes" but does not address efficiency

2. **Optimization**: Can we find the best parameters?
   - Non-convex loss landscape with many local minima and saddle points
   - SGD + overparameterization often succeeds in practice (loss of convexity is benign)
   - KL inequality provides convergence guarantees for analytic losses

3. **Generalization**: Does training performance transfer to unseen data?
   - Classical bounds (VC dimension) are often too loose for deep networks
   - Norm-based and PAC-Bayes bounds are tighter
   - Implicit regularization of SGD plays a crucial role

**The total error is bounded by the sum of all three components.**
Reducing one often increases another; architecture and algorithm design must balance all three.

---

## Problem-Solving Patterns

### Pattern 1: Architecture Selection via Approximation Theory
- **Problem**: Choosing network depth and width for a given task.
- **Approach**: Analyze the target function class. If functions have compositional structure, prefer depth. If smoothness is the primary feature, width may suffice for shallow networks.
- **Tools**: Depth separation results, ReLU expressivity bounds, piecewise linear region counting.

### Pattern 2: Learning Rate Tuning via Convergence Analysis
- **Problem**: Training is diverging or converging too slowly.
- **Approach**: Estimate the Lipschitz constant L of the gradient. Set initial learning rate eta ~ 1/L. Use warmup if initial loss landscape is poorly conditioned.
- **Tools**: Smoothness analysis, learning rate finder (empirical), convergence rate formulas.

### Pattern 3: Diagnosing Generalization Failures
- **Problem**: Large gap between training and test performance.
- **Approach**: Check bias-variance decomposition. If high variance, consider regularization (weight decay, dropout, data augmentation). If double descent is occurring, increasing model size may help.
- **Tools**: Learning curves, weight norm monitoring, Rademacher complexity estimates.

### Pattern 4: Dimensionality Reduction Pipeline
- **Problem**: High-dimensional data causing computational or statistical issues.
- **Approach**: Apply PCA/SVD to reduce dimensions. Choose k such that cumulative explained variance exceeds a threshold (e.g., 95%). Verify that downstream task performance is preserved.
- **Tools**: Scree plot, explained variance ratio, truncated SVD algorithms.

### Pattern 5: PDE Solving with Neural Networks
- **Problem**: Solve a PDE in high dimensions where mesh-based methods fail.
- **Approach**: Formulate as a PINN or use the Deep Kolmogorov method if a stochastic representation exists. Sample collocation points. Balance PDE residual and boundary losses.
- **Tools**: PINN loss formulation, Feynman-Kac formula, adaptive collocation point sampling.

### Pattern 6: Spectral Analysis for Graph-Structured Data
- **Problem**: Clustering or partitioning graph-structured data.
- **Approach**: Compute the graph Laplacian. Use the Fiedler vector (second smallest eigenvector) for bipartition. For k clusters, use the k smallest eigenvectors and apply k-means.
- **Tools**: Cheeger inequality for quality bounds, normalized vs. unnormalized Laplacian selection.

---

## Common Pitfalls

1. **Confusing universal approximation with learnability**: A network can approximate any function does not mean gradient descent will find the right parameters. Approximation != optimization.

2. **Ignoring the curse of dimensionality**: Classical approximation rates degrade exponentially with dimension. Deep networks may help but are not immune; structure in the data is essential.

3. **Misapplying VC dimension bounds**: VC bounds for deep networks are often astronomically loose. Norm-based bounds or PAC-Bayes bounds are more informative in practice.

4. **Wrong initialization scale**: Too large causes exploding activations/gradients; too small causes vanishing. Xavier for tanh/sigmoid; He for ReLU. Always match initialization to activation function.

5. **Neglecting loss landscape geometry**: Treating the loss as a simple convex function. In practice, saddle points dominate over local minima in high dimensions, and batch normalization changes the geometry significantly.

6. **Truncating SVD/PCA too aggressively**: Keeping too few components loses essential information. Always check reconstruction error and downstream task impact.

7. **PINN training instabilities**: Unbalanced loss weights (lambda_PDE vs lambda_BC) cause the network to satisfy boundary conditions but ignore the PDE or vice versa. Use adaptive weighting strategies.

8. **Confusing spectral and spatial graph convolutions**: Spectral methods require fixed graph structure; spatial methods generalize to different graphs. Choose based on whether the graph is static or varies across samples.

---

## Source Material

### Primary References

1. **Jentzen, A., Kuckuck, B., & von Wurstemberger, P. (2025)**. *Mathematical Introduction to Deep Learning: Methods, Implementations, and Theory*. Applied Mathematics series. Comprehensive treatment of:
   - Formal neural network definitions and realization functions
   - Universal approximation theorems (1D and multivariate)
   - ReLU network expressivity and depth separation
   - Gradient descent and SGD convergence theory
   - Backpropagation derivation and computational complexity
   - Kurdyka-Lojasiewicz convergence framework
   - Bias-variance decomposition and generalization bounds
   - Overall error decomposition (approximation + estimation + optimization)
   - Deep learning methods for PDEs (PINNs, DGM, Deep Kolmogorov)

2. **Gallier, J., & Quaintance, J. (2025)**. *Linear Algebra, Geometry, and Computation: for Computer Vision, Robotics, and Machine Learning*. Springer. Comprehensive treatment of:
   - Vector spaces, linear maps, and matrix decompositions (LU, QR, Schur, SVD)
   - Spectral theory for symmetric and Hermitian matrices
   - SVD: geometric interpretation, Eckart-Young theorem, applications
   - PCA: covariance-based derivation, explained variance, dimensionality reduction
   - Moore-Penrose pseudo-inverse and least squares
   - Matrix exponential and connections to Lie groups
   - Quaternions, SU(2), SO(3), and rotation representations
   - Haar bases, wavelets, and multiresolution analysis
   - Hadamard matrices and fast transforms
   - Spectral graph theory: Laplacian, Cheeger inequality, spectral clustering
   - Norms: vector norms, matrix norms, dual norms, Frobenius, nuclear

### Reference Files

- `references/optimization-convergence.md` — GD/SGD convergence proofs, learning rate schedules, momentum methods
- `references/approximation-theorems.md` — Universal approximation statements, depth-width tradeoffs, ReLU expressivity
- `references/svd-pca-applications.md` — SVD computation, geometric interpretation, PCA derivation, applications
- `references/pde-neural-methods.md` — PINNs, Deep Galerkin, Deep Kolmogorov with loss formulations
- `references/spectral-graph-theory.md` — Graph Laplacian, spectral clustering, Cheeger inequality, GNN foundations
