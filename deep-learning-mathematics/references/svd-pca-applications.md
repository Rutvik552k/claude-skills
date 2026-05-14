# SVD, PCA, and Applications

## Overview

This reference covers the Singular Value Decomposition (SVD), Principal Component Analysis (PCA),
their mathematical foundations, geometric interpretations, computational methods, and applications
in machine learning and data science. Based on Gallier & Quaintance (2025), Chapters on SVD,
Spectral Theory, and Dimensionality Reduction.

---

## 1. Singular Value Decomposition (SVD)

### Definition

**Theorem (SVD Existence)**: For any matrix A in R^{m x n} with rank r, there exist:
- U in R^{m x m} orthogonal (U^T U = I)
- V in R^{n x n} orthogonal (V^T V = I)
- Sigma in R^{m x n} with diagonal entries sigma_1 >= sigma_2 >= ... >= sigma_r > 0 and zeros elsewhere

such that A = U Sigma V^T.

The sigma_i are the **singular values** of A. The columns of U are **left singular vectors**.
The columns of V are **right singular vectors**.

### Compact (Thin) SVD

    A = U_r Sigma_r V_r^T

where U_r in R^{m x r}, Sigma_r in R^{r x r}, V_r in R^{n x r} contain only the r nonzero
singular value components. This is the computationally efficient form.

### Proof of Existence

**Proof**:
1. A^T A is symmetric positive semi-definite, so it has non-negative eigenvalues
   lambda_1 >= ... >= lambda_n >= 0 with orthonormal eigenvectors v_1, ..., v_n.

2. Define sigma_i = sqrt(lambda_i) for i = 1, ..., r (the nonzero eigenvalues).

3. Define u_i = (1/sigma_i) A v_i for i = 1, ..., r. These are orthonormal:
   u_i^T u_j = (1/(sigma_i sigma_j)) v_i^T A^T A v_j = (lambda_j/(sigma_i sigma_j)) v_i^T v_j = delta_{ij}

4. Extend {u_1, ..., u_r} to an orthonormal basis {u_1, ..., u_m} of R^m.

5. Then A v_i = sigma_i u_i for i <= r and A v_i = 0 for i > r, which gives A = U Sigma V^T.

### Relation to Eigendecomposition

- Singular values of A = square roots of eigenvalues of A^T A (or A A^T)
- sigma_i(A) = sqrt(lambda_i(A^T A))
- Left singular vectors = eigenvectors of A A^T
- Right singular vectors = eigenvectors of A^T A
- If A is symmetric PSD: singular values = eigenvalues, and U = V (up to signs)

---

## 2. Geometric Interpretation of SVD

### Linear Map as Rotation-Scaling-Rotation

The SVD A = U Sigma V^T decomposes the linear map x -> Ax into three steps:

1. **V^T**: Rotate/reflect x in the domain space (R^n) to align with the principal axes
2. **Sigma**: Scale along each axis by the corresponding singular value
3. **U**: Rotate/reflect the result into the codomain space (R^m)

### Image of the Unit Sphere

The image of the unit sphere S^{n-1} under A is an ellipsoid in R^m:

    A(S^{n-1}) = { Ax : ||x|| = 1 }

The semi-axes of this ellipsoid:
- Directions: u_1, ..., u_r (left singular vectors)
- Lengths: sigma_1, ..., sigma_r (singular values)

### Volume Distortion

The volume distortion factor of A on any r-dimensional subspace is:

    |det(A restricted to subspace)| = product of relevant singular values

For the full map: vol(A(S)) = sigma_1 * sigma_2 * ... * sigma_r * vol(S) (for r-dim sets).

### Condition Number

    kappa(A) = sigma_1 / sigma_r = ||A|| * ||A^{-1}|| (for invertible A)

The condition number measures:
- How much A distorts relative lengths (large kappa = ill-conditioned)
- Sensitivity of Ax = b to perturbations in A or b
- Numerical stability of solving linear systems with A

---

## 3. Eckart-Young Theorem (Best Low-Rank Approximation)

### Statement

**Theorem (Eckart-Young-Mirsky)**: Let A = U Sigma V^T be the SVD of A. For any k <= r = rank(A),
the best rank-k approximation to A is:

    A_k = sum_{i=1}^k sigma_i u_i v_i^T = U_k Sigma_k V_k^T

In the Frobenius norm:

    ||A - A_k||_F = min_{rank(B) <= k} ||A - B||_F = sqrt(sigma_{k+1}^2 + ... + sigma_r^2)

In the operator (spectral) norm:

    ||A - A_k||_2 = min_{rank(B) <= k} ||A - B||_2 = sigma_{k+1}

### Proof (Frobenius Norm)

1. ||A - B||_F^2 = sum_{i=1}^r sigma_i^2 - 2 * trace(B^T A) + ||B||_F^2 for any B.

2. For rank(B) <= k, write B = sum_{j=1}^k s_j p_j q_j^T (SVD of B).

3. By the von Neumann trace inequality:
   trace(B^T A) <= sum_{j=1}^k sigma_j(A) * sigma_j(B)

4. The minimum of ||A - B||_F^2 over rank-k B is achieved when B matches the top-k
   singular components of A.

### Approximation Error

The relative error of rank-k approximation:

    ||A - A_k||_F / ||A||_F = sqrt(sum_{i=k+1}^r sigma_i^2) / sqrt(sum_{i=1}^r sigma_i^2)

The fraction of "energy" (variance) captured by A_k:

    ||A_k||_F^2 / ||A||_F^2 = sum_{i=1}^k sigma_i^2 / sum_{i=1}^r sigma_i^2

---

## 4. Truncated SVD

### Algorithm

Given A in R^{m x n} and target rank k:

1. Compute SVD: A = U Sigma V^T
2. Keep only the top k components: A_k = U_k Sigma_k V_k^T
3. Store U_k (m x k), Sigma_k (k x k diagonal), V_k (n x k): total (m + n + 1) * k entries

### Computational Complexity

- Full SVD: O(min(m^2 n, m n^2))
- Truncated SVD (randomized): O(m n k) for rank-k approximation
- Iterative methods (Lanczos/Arnoldi): efficient when k << min(m, n)

### Randomized SVD Algorithm

1. Generate random Gaussian matrix Omega in R^{n x (k + p)} where p is oversampling (e.g., p = 10)
2. Form Y = A * Omega (m x (k+p) matrix)
3. Compute QR: Y = Q R (Q is m x (k+p) orthonormal)
4. Form B = Q^T A (small (k+p) x n matrix)
5. Compute SVD of B: B = U_B Sigma_B V_B^T
6. Return U = Q U_B, Sigma = Sigma_B, V = V_B

**Guarantee**: With high probability, ||A - U Sigma V^T||_F <= (1 + epsilon) * ||A - A_k||_F.

### Applications of Truncated SVD

- **Image compression**: Store image as A_k instead of A; compression ratio = (m + n) * k / (m * n)
- **Latent Semantic Analysis (LSA)**: Term-document matrix -> top-k SVD captures semantic relationships
- **Recommender systems**: User-item rating matrix -> low-rank factors for collaborative filtering
- **Noise reduction**: Singular values below a threshold correspond to noise; truncate to denoise

---

## 5. Principal Component Analysis (PCA)

### Problem Setup

Given data matrix X in R^{n x d} (n samples, d features), with rows x_1, ..., x_n:

1. Center the data: X_c = X - (1/n) 1 1^T X (subtract column means)
2. Covariance matrix: C = (1/(n-1)) X_c^T X_c in R^{d x d}

### PCA as Variance Maximization

**Problem**: Find direction w in R^d (||w|| = 1) that maximizes the variance of projected data:

    max_{||w||=1} Var(X_c w) = max_{||w||=1} w^T C w

**Solution**: w_1 = eigenvector of C corresponding to the largest eigenvalue lambda_1.

The k-th principal component maximizes variance subject to orthogonality to the first k-1:

    w_k = argmax_{||w||=1, w perp w_1,...,w_{k-1}} w^T C w

Solution: w_k = eigenvector of C corresponding to the k-th largest eigenvalue lambda_k.

### PCA via SVD

**Connection**: If X_c = U Sigma V^T is the SVD of the centered data matrix, then:

- Principal component directions: columns of V
- Principal component scores: X_c V = U Sigma
- Eigenvalues of C: lambda_i = sigma_i^2 / (n - 1)

**Proof**: C = (1/(n-1)) X_c^T X_c = (1/(n-1)) V Sigma^T U^T U Sigma V^T = V (Sigma^2/(n-1)) V^T.

### Explained Variance

- Variance along k-th PC: lambda_k
- Total variance: sum_{i=1}^d lambda_i = trace(C)
- Explained variance ratio for k-th PC: lambda_k / trace(C)
- Cumulative explained variance for top-k PCs: sum_{i=1}^k lambda_i / trace(C)

**Rule of thumb**: Choose k such that cumulative explained variance >= 0.95 (or use the "elbow" in the scree plot).

### PCA as Minimum Reconstruction Error

**Equivalent formulation**: PCA finds the k-dimensional subspace that minimizes reconstruction error:

    min_{W in R^{d x k}, W^T W = I} sum_{i=1}^n ||x_i - W W^T x_i||^2

This is the projection onto the subspace spanned by the top-k eigenvectors of C.

**Reconstruction error** = sum_{i=k+1}^d lambda_i (the sum of discarded eigenvalues).

---

## 6. Kernel PCA

### Motivation

Standard PCA finds linear subspaces. For nonlinearly structured data, map to a feature space
phi: R^d -> R^D (D possibly infinite) and apply PCA there.

### Kernel Trick

Instead of computing phi(x) explicitly, work with the kernel matrix:

    K_{ij} = phi(x_i)^T phi(x_j) = k(x_i, x_j)

Common kernels:
- Polynomial: k(x, y) = (x^T y + c)^p
- RBF/Gaussian: k(x, y) = exp(-||x - y||^2 / (2 * sigma^2))
- Laplacian: k(x, y) = exp(-||x - y||_1 / sigma)

### Algorithm

1. Compute kernel matrix K in R^{n x n}
2. Center: K_c = K - (1/n) 1 K - (1/n) K 1 + (1/n^2) 1 K 1
3. Eigendecompose: K_c = Q Lambda Q^T
4. Principal component scores: alpha_k = (1/sqrt(lambda_k)) * q_k
5. Projection of new point x: z_k = sum_{i=1}^n alpha_k[i] * k(x_i, x)

### Complexity

- Standard PCA: O(n * d^2 + d^3) (covariance matrix + eigendecomposition)
- Kernel PCA: O(n^2 * d + n^3) (kernel matrix + eigendecomposition)
- Better when d >> n (more features than samples)

---

## 7. Probabilistic PCA

### Model

    x = W z + mu + epsilon

where:
- z ~ N(0, I_k) is the latent variable (k-dimensional)
- W in R^{d x k} is the loading matrix
- mu in R^d is the mean
- epsilon ~ N(0, sigma^2 I_d) is isotropic noise

### Maximum Likelihood Solution

    W_ML = U_k (Lambda_k - sigma^2 I)^{1/2} R

where U_k contains the top-k eigenvectors of C, Lambda_k is the corresponding eigenvalues,
and R is an arbitrary rotation matrix.

    sigma^2_ML = (1/(d-k)) sum_{i=k+1}^d lambda_i

**Interpretation**: The noise variance is the average of the discarded eigenvalues.

### Connection to Factor Analysis

Factor analysis generalizes PPCA by allowing non-isotropic noise:

    epsilon ~ N(0, Psi) where Psi = diag(psi_1, ..., psi_d)

This allows different noise levels for different features but requires iterative estimation (EM).

---

## 8. Incremental and Online PCA

### Incremental PCA

When data arrives in batches, update the SVD incrementally:

Given current A_k = U_k Sigma_k V_k^T and new data block B:

1. Project: P = U_k^T B (projection of B onto current subspace)
2. Residual: Q_B R_B = (B - U_k P) (QR of residual)
3. Build: K = [[Sigma_k, P], [0, R_B]]
4. SVD of K: K = U_K Sigma_K V_K^T
5. Update: U_{k+1} = [U_k, Q_B] U_K, Sigma_{k+1} = Sigma_K

### Streaming PCA (Oja's Rule)

For one sample at a time, update the principal direction:

    w_{t+1} = w_t + eta_t * (x_t x_t^T w_t - w_t)  [then normalize]

Simplified: w_{t+1} = w_t + eta_t * x_t (x_t^T w_t) [then normalize]

Convergence: w_t -> v_1 (first eigenvector) under Robbins-Monro conditions on eta_t.

**Generalization to k components**: Use Sanger's rule or the GHA (Generalized Hebbian Algorithm).

---

## 9. Applications in Machine Learning

### Dimensionality Reduction for Classification

1. Compute PCA on training data
2. Project train and test to top-k dimensions
3. Train classifier in reduced space
4. Benefits: fewer parameters, reduced overfitting, faster training

**Caution**: PCA is unsupervised; it maximizes variance, not class separability.
For supervised dimensionality reduction, consider LDA (Linear Discriminant Analysis).

### Data Visualization

- Project to k = 2 or k = 3 for plotting
- PCA preserves global structure (large distances) better than local structure
- For local structure preservation, consider t-SNE or UMAP

### Feature Extraction and Whitening

**PCA Whitening**: Transform data so that the covariance is the identity:

    x_white = Lambda^{-1/2} V^T (x - mu)

Properties:
- Zero mean, identity covariance
- Decorrelates features
- Equalizes variance across all directions

**ZCA Whitening**: x_zca = V Lambda^{-1/2} V^T (x - mu)
Preserves the original coordinate system better than PCA whitening.

### Image Compression

For an m x n grayscale image A:
1. Compute SVD: A = U Sigma V^T
2. Truncate to rank k: A_k = U_k Sigma_k V_k^T
3. Storage: k * (m + n + 1) instead of m * n
4. Compression ratio: m * n / (k * (m + n + 1))
5. Quality: PSNR = 10 * log10(255^2 / MSE) where MSE = ||A - A_k||_F^2 / (m*n)

### Recommender Systems

User-item rating matrix R (sparse, m users x n items):
1. Fill missing values (e.g., with row/column means)
2. Compute truncated SVD: R approx U_k Sigma_k V_k^T
3. Predicted rating for user i, item j: (U_k Sigma_k V_k^T)_{ij}
4. Regularized version: min ||P(R - U V^T)||_F^2 + lambda (||U||_F^2 + ||V||_F^2)
   where P is the projection onto observed entries

### Latent Semantic Analysis (LSA)

Term-document matrix M (vocabulary x documents):
1. Apply TF-IDF weighting
2. Compute rank-k SVD: M approx U_k Sigma_k V_k^T
3. Document similarity: compare columns of Sigma_k V_k^T
4. Term similarity: compare rows of U_k Sigma_k
5. Query matching: project query q into latent space: q_latent = Sigma_k^{-1} U_k^T q

---

## 10. Numerical Considerations

### Numerical Stability

- Never form A^T A explicitly for computing SVD (doubles the condition number)
- Use the Golub-Kahan bidiagonalization or divide-and-conquer algorithms
- For PCA, compute SVD of X_c directly rather than eigendecomposition of X_c^T X_c

### Centering Matters

PCA requires centered data. Forgetting to center:
- First PC will point toward the mean instead of the direction of maximum variance
- All subsequent PCs will be wrong

### Scaling

If features have different units/scales, standardize (z-score) before PCA:
    x_std = (x - mu) / sigma (per feature)

Otherwise, features with larger variance dominate the principal components regardless of importance.

### Choosing k (Number of Components)

Methods:
1. **Explained variance threshold**: choose k such that cumulative >= 95%
2. **Scree plot**: look for an "elbow" in the eigenvalue spectrum
3. **Kaiser criterion**: keep PCs with lambda_k > 1 (for standardized data)
4. **Cross-validation**: choose k that minimizes reconstruction error on held-out data
5. **Parallel analysis**: compare eigenvalues to those from random data of same size

### Sparse PCA

Standard PCA loadings are dense (all features contribute). Sparse PCA adds an L1 penalty:

    max_{||w||=1} w^T C w - lambda ||w||_1

This yields interpretable components where most loadings are exactly zero.
Algorithms: SPCA (Zou et al., 2006), iterative thresholding, ADMM.

---

## 11. Connections to Deep Learning

### PCA as a Linear Autoencoder

A linear autoencoder with k hidden units:
    encoder: z = W_e x,  decoder: x_hat = W_d z

Training with MSE loss: the optimal solution satisfies W_d W_e = V_k V_k^T (projection onto PCA subspace).

**Result**: Linear autoencoders learn the PCA subspace (not necessarily the PCA basis).

### SVD in Neural Network Weight Matrices

- Weight matrix SVD reveals the "effective rank" of a layer
- Low-rank weight matrices: W = U_k Sigma_k V_k^T reduces parameters from m*n to k*(m+n)
- LoRA (Low-Rank Adaptation): W = W_0 + B A where B in R^{m x k}, A in R^{k x n}
  Fine-tune only A and B (much fewer parameters than full W)

### Spectral Normalization

Normalize weight matrices by their largest singular value:
    W_normalized = W / sigma_1(W)

Ensures Lipschitz continuity of the network (Lipschitz constant <= 1 per layer).
Used in GANs (Miyato et al., 2018) for training stability.

### SVD for Understanding Learned Representations

- Compute SVD of hidden layer activations across a dataset
- Singular value spectrum reveals effective dimensionality
- Rapid singular value decay -> representations live on a low-dimensional manifold
- Connection to neural collapse in classification: terminal layer features converge to a simplex ETF
