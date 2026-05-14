# Spectral Graph Theory for Machine Learning

## Overview

This reference covers spectral graph theory and its applications to machine learning: graph
Laplacians, eigenvalue properties, spectral clustering, the Cheeger inequality, and connections
to graph neural networks (GNNs). Based on Gallier & Quaintance (2025), Chapters on Spectral
Graph Theory and Graph-Based Methods.

---

## 1. Graph Fundamentals

### Definitions

An undirected weighted graph G = (V, E, w) consists of:
- Vertex set V = {v_1, ..., v_n}
- Edge set E subset V x V (symmetric: (v_i, v_j) in E iff (v_j, v_i) in E)
- Weight function w: E -> R_+ (positive edge weights)

### Matrix Representations

**Adjacency matrix** A in R^{n x n}:
    A_{ij} = w(v_i, v_j) if (v_i, v_j) in E, and 0 otherwise

**Degree matrix** D in R^{n x n} (diagonal):
    D_{ii} = d(v_i) = sum_{j: (v_i, v_j) in E} w(v_i, v_j)

For unweighted graphs: D_{ii} = number of neighbors of v_i.

---

## 2. Graph Laplacian

### Unnormalized (Combinatorial) Laplacian

    L = D - A

**Properties**:
1. L is symmetric positive semi-definite (PSD)
2. L 1 = 0 (the all-ones vector is in the null space)
3. Smallest eigenvalue lambda_1 = 0 with eigenvector v_1 = (1/sqrt(n)) * 1
4. L_{ii} = d(v_i), L_{ij} = -w(v_i, v_j) for i != j
5. For any vector f in R^n:
   f^T L f = (1/2) sum_{(i,j) in E} w_{ij} (f_i - f_j)^2 >= 0

**Proof of Property 5**:
    f^T L f = f^T D f - f^T A f
            = sum_i d_i f_i^2 - sum_{(i,j) in E} w_{ij} f_i f_j
            = (1/2) sum_{(i,j) in E} w_{ij} (f_i^2 + f_j^2 - 2 f_i f_j)
            = (1/2) sum_{(i,j) in E} w_{ij} (f_i - f_j)^2

**Interpretation**: f^T L f measures the "smoothness" of the signal f on the graph.
Small value means f varies slowly across edges (similar values at connected vertices).

### Normalized Laplacians

**Symmetric normalized Laplacian**:

    L_sym = D^{-1/2} L D^{-1/2} = I - D^{-1/2} A D^{-1/2}

**Properties**:
1. Eigenvalues in [0, 2]
2. lambda = 0 with eigenvector D^{1/2} 1
3. lambda = 2 iff graph is bipartite
4. L_sym is similar to the random walk Laplacian (same eigenvalues)

**Random walk normalized Laplacian**:

    L_rw = D^{-1} L = I - D^{-1} A

**Properties**:
1. Same eigenvalues as L_sym
2. lambda = 0 with eigenvector 1
3. D^{-1} A is the transition matrix of a random walk on G
4. Eigenvectors of L_rw are D^{1/2} times eigenvectors of L_sym

### Which Laplacian to Use?

| Laplacian | Best for                          | Eigenvalue range |
|-----------|-----------------------------------|------------------|
| L         | Unweighted graphs, theoretical analysis | [0, n * d_max] |
| L_sym     | Spectral clustering (balanced)     | [0, 2]          |
| L_rw      | Spectral clustering (unbalanced), random walks | [0, 2] |

**General recommendation**: Use L_sym or L_rw for practical applications. L is useful for
theoretical analysis and when the degree distribution is approximately uniform.

---

## 3. Spectral Properties

### Eigenvalue Ordering

Let 0 = lambda_1 <= lambda_2 <= ... <= lambda_n be the eigenvalues of L.

**Theorem**: The multiplicity of the eigenvalue 0 equals the number of connected components of G.

**Proof**: If G has k connected components G_1, ..., G_k, then:
- L has a block diagonal structure (after reordering vertices)
- Each block L_i is the Laplacian of G_i
- Each connected G_i contributes exactly one zero eigenvalue
- Total: k zero eigenvalues

### Algebraic Connectivity (Fiedler Value)

**Definition**: lambda_2 is called the algebraic connectivity or Fiedler value of G.

**Properties**:
1. lambda_2 > 0 iff G is connected
2. Larger lambda_2 implies better connected graph (harder to disconnect)
3. Fiedler vector: the eigenvector corresponding to lambda_2

**Bounds on lambda_2**:
- lambda_2 <= n/(n-1) * min_i d_i (vertex connectivity bound)
- lambda_2 >= 2 * (1 - cos(pi/n)) for connected graphs (approaches 0 as n -> inf for path graphs)
- lambda_2 <= 2 * e(G) / (n-1) where e(G) is the edge connectivity

### Spectral Gap

The spectral gap lambda_2 - lambda_1 = lambda_2 (since lambda_1 = 0) measures:
- How fast a random walk on G mixes (mixing time ~ 1/lambda_2)
- How well-connected G is
- The quality of spectral clustering (larger gap -> cleaner separation)

### Eigenvalues of Special Graphs

**Complete graph K_n**: lambda_1 = 0, lambda_2 = ... = lambda_n = n

**Path graph P_n**: lambda_k = 2 - 2*cos(pi*(k-1)/n) for k = 1, ..., n

**Cycle graph C_n**: lambda_k = 2 - 2*cos(2*pi*(k-1)/n) for k = 1, ..., n

**Star graph S_n**: lambda_1 = 0, lambda_2 = ... = lambda_{n-1} = 1, lambda_n = n

**Complete bipartite K_{p,q}**: Nonzero eigenvalues are p, q, and p+q (with multiplicities)

---

## 4. Cheeger Inequality

### Cheeger Constant (Isoperimetric Number)

For a subset S subset V, define:

    cut(S, S^c) = sum_{i in S, j in S^c} w_{ij}
    vol(S) = sum_{i in S} d_i

**Cheeger constant**:

    h(G) = min_{S: 0 < vol(S) <= vol(V)/2} cut(S, S^c) / vol(S)

**Interpretation**: h(G) measures the "bottleneck" of the graph — how hard it is to cut
the graph into two roughly equal parts.

### Statement

**Theorem (Cheeger Inequality)**: For the normalized Laplacian:

    h(G)^2 / 2 <= lambda_2 <= 2 * h(G)

Equivalently:

    lambda_2 / 2 <= h(G) <= sqrt(2 * lambda_2)

### Proof Sketch (Upper Bound: lambda_2 <= 2 * h(G))

1. Let S be the set achieving the Cheeger constant: h(G) = cut(S, S^c) / vol(S).
2. Define the indicator vector f in R^n: f_i = 1 if i in S, f_i = 0 otherwise.
3. By the Rayleigh quotient characterization:
   lambda_2 <= (f^T L f) / (f^T D f) (after projecting out the constant eigenvector)
4. Compute: f^T L f = cut(S, S^c) and f^T D f = vol(S) * vol(S^c) / vol(V)
5. After manipulation: lambda_2 <= 2 * cut(S, S^c) / vol(S) = 2 * h(G)

### Proof Sketch (Lower Bound: h(G)^2 / 2 <= lambda_2)

1. Take f as the Fiedler vector (eigenvector of lambda_2)
2. Define a "sweep cut": sort vertices by f_i values, and for each threshold t,
   consider S_t = {i : f_i <= t}
3. Show that the best sweep cut satisfies: h(S_t) <= sqrt(2 * lambda_2)
4. Since h(G) <= h(S_t), we get h(G) <= sqrt(2 * lambda_2), i.e., h(G)^2 / 2 <= lambda_2

### Significance

- **Algorithmic**: The Fiedler vector gives a near-optimal graph partition
  (within a quadratic factor of the Cheeger constant).
- **Theoretical**: Connects the spectral gap (algebraic) to the combinatorial
  expansion property (geometric) of the graph.
- **Multi-way**: Higher-order Cheeger inequalities relate lambda_k to k-way partitions.

---

## 5. Spectral Clustering

### Algorithm (Unnormalized Spectral Clustering)

Input: Similarity matrix W in R^{n x n}, number of clusters k.

1. Compute the graph Laplacian L = D - W
2. Compute the first k eigenvectors v_1, ..., v_k of L (smallest eigenvalues)
3. Form matrix V in R^{n x k} with columns v_1, ..., v_k
4. Row i of V gives a k-dimensional representation of vertex i
5. Apply k-means to the rows of V to obtain clusters C_1, ..., C_k

### Algorithm (Normalized Spectral Clustering — Ng, Jordan, Weiss)

1. Compute L_sym = I - D^{-1/2} W D^{-1/2}
2. Compute first k eigenvectors u_1, ..., u_k of L_sym
3. Form U in R^{n x k} with columns u_1, ..., u_k
4. Normalize rows: T_{ij} = U_{ij} / sqrt(sum_j U_{ij}^2)
5. Apply k-means to rows of T

### Algorithm (Normalized Spectral Clustering — Shi & Malik)

1. Compute L_rw = I - D^{-1} W
2. Compute first k eigenvectors of L_rw (equivalently, generalized eigenvectors of L v = lambda D v)
3. Apply k-means to rows

### Why Spectral Clustering Works

**Ideal case**: If the graph has k connected components, then:
- lambda_1 = ... = lambda_k = 0
- Eigenvectors v_1, ..., v_k are (up to rotation) indicator vectors of the components
- k-means on the eigenvector matrix perfectly recovers the components

**Near-ideal case**: If the graph has k "almost disconnected" components (small inter-cluster edges):
- lambda_1 = ... = lambda_k are near zero (but not exactly zero)
- Eigenvectors are perturbations of the ideal indicator vectors
- k-means approximately recovers the clusters

**Perturbation theory (Davis-Kahan)**: If the spectral gap lambda_{k+1} - lambda_k is large,
the eigenvector subspace is robust to perturbations. This justifies spectral clustering
when clusters are well-separated.

### Similarity Graph Construction

Given data points x_1, ..., x_n, construct a similarity graph:

**Epsilon-neighborhood graph**: Connect x_i and x_j if ||x_i - x_j|| < epsilon.

**k-nearest neighbor (kNN) graph**: Connect x_i to its k nearest neighbors (and symmetrize).
- Mutual kNN: edge iff both are in each other's kNN
- Symmetric kNN: edge iff either is in the other's kNN

**Fully connected graph**: w_{ij} = exp(-||x_i - x_j||^2 / (2 sigma^2)) for all i, j.
The bandwidth sigma controls the scale of similarity.

**Recommendation**: kNN graph with k ~ log(n) or Gaussian kernel with sigma chosen by
the "self-tuning" method (Zelnik-Manor & Perona, 2005).

---

## 6. Spectral Graph Theory for GNNs

### Graph Convolution via Laplacian

**Spectral definition of graph convolution**:

    g *_G f = U (g_hat . f_hat) = U diag(g_hat) U^T f

where:
- L = U Lambda U^T is the eigendecomposition of L
- f_hat = U^T f is the graph Fourier transform of signal f
- g_hat in R^n is the filter in the spectral domain
- . denotes element-wise multiplication

### ChebNet (Defferrard et al., 2016)

Approximate the spectral filter with Chebyshev polynomials:

    g_theta *_G f approx sum_{k=0}^{K-1} theta_k T_k(L_tilde) f

where:
- L_tilde = 2L/lambda_max - I (scaled to [-1, 1])
- T_k is the k-th Chebyshev polynomial: T_0(x) = 1, T_1(x) = x, T_{k+1}(x) = 2x T_k(x) - T_{k-1}(x)
- theta_0, ..., theta_{K-1} are learnable parameters

**Advantages**:
- K-localized: each node aggregates information from its K-hop neighborhood
- O(K |E|) computation (no eigendecomposition needed)
- K parameters per filter (not n)

### GCN (Kipf & Welling, 2017)

Simplify ChebNet to K = 1 (first-order approximation):

    H^{(l+1)} = sigma(D_tilde^{-1/2} A_tilde D_tilde^{-1/2} H^{(l)} W^{(l)})

where:
- A_tilde = A + I (adjacency with self-loops)
- D_tilde_{ii} = sum_j A_tilde_{ij}
- H^{(l)} in R^{n x d_l} is the feature matrix at layer l
- W^{(l)} in R^{d_l x d_{l+1}} is the learnable weight matrix

**Spectral interpretation**: GCN applies a low-pass filter on the graph, smoothing node features
according to the graph structure. The renormalization trick (A_tilde = A + I) prevents the
eigenvalues from being exactly 0 or 2.

### Spectral Analysis of GNN Expressivity

**Over-smoothing**: As the number of GCN layers increases, all node features converge to the
same value (proportional to the leading eigenvector of the normalized adjacency).

    H^{(L)} -> v_1 v_1^T H^{(0)} as L -> inf

This is because D_tilde^{-1/2} A_tilde D_tilde^{-1/2} is a low-pass filter, and repeated
application attenuates all frequencies except the lowest.

**Mitigation strategies**:
- Skip connections (residual GNNs)
- Jumping knowledge networks (aggregate features from all layers)
- PairNorm or DiffPool for normalization
- Higher-order polynomial filters (ChebNet, BernNet)

### Graph Attention and Spectral Theory

Graph Attention Networks (GAT) use learned attention weights:

    alpha_{ij} = softmax_j(LeakyReLU(a^T [W h_i || W h_j]))

**Spectral interpretation**: GAT applies an adaptive, data-dependent filter. The attention
weights modify the graph Laplacian: L_attn = D_attn - A_attn where A_attn_{ij} = alpha_{ij}.

This allows the network to learn which frequency components to emphasize at each layer.

---

## 7. Advanced Spectral Properties

### Courant-Fischer Min-Max Theorem

**Theorem**: The k-th eigenvalue of L (or any symmetric matrix) satisfies:

    lambda_k = min_{dim(S)=k} max_{f in S, f != 0} (f^T L f) / (f^T f)
             = max_{dim(S)=n-k+1} min_{f in S, f != 0} (f^T L f) / (f^T f)

**Consequence**: lambda_2 = min_{f perp 1} (f^T L f) / (f^T f)
This gives a variational characterization of the Fiedler value.

### Interlacing Theorems

**Cauchy Interlacing**: If H is a principal submatrix of L (obtained by deleting rows and columns
corresponding to some vertices), then the eigenvalues of H interlace those of L:

    lambda_k(L) <= lambda_k(H) <= lambda_{k + n - m}(L)

where L is n x n and H is m x m.

**Application**: Deleting a vertex from G does not change the eigenvalues too much.
This is used in divide-and-conquer algorithms for spectral decomposition.

### Eigenvalues and Graph Properties

| Property                  | Spectral Characterization              |
|---------------------------|----------------------------------------|
| Connected                 | lambda_2 > 0                           |
| Bipartite                 | lambda_n = 2 (for L_sym)               |
| k connected components    | lambda_1 = ... = lambda_k = 0          |
| Expander graph            | lambda_2 >= c for some constant c > 0  |
| Diameter D                | D <= ceil(cosh^{-1}(n-1) / cosh^{-1}((lambda_n + lambda_2)/(lambda_n - lambda_2))) |
| Chromatic number chi      | chi >= 1 + lambda_n / |lambda_1| (for adjacency matrix eigenvalues) |

### Ramanujan Graphs

A d-regular graph is **Ramanujan** if all eigenvalues lambda of the adjacency matrix satisfy:

    |lambda| <= 2 * sqrt(d - 1)    (for lambda != +/-d)

Ramanujan graphs are optimal expanders: they have the largest possible spectral gap
among d-regular graphs (by the Alon-Boppana bound).

**Significance for ML**: Ramanujan graphs provide the best "information propagation" structure
for message-passing neural networks on regular graphs.

---

## 8. Graph Signal Processing

### Graph Fourier Transform

For a signal f in R^n on a graph with Laplacian L = U Lambda U^T:

    f_hat = U^T f    (analysis: transform to spectral domain)
    f = U f_hat       (synthesis: transform back to spatial domain)

**Analogy with classical Fourier analysis**:
- Eigenvectors of L play the role of sinusoids
- Eigenvalues play the role of frequencies (squared)
- Low eigenvalue eigenvectors = "smooth" signals on the graph
- High eigenvalue eigenvectors = "oscillatory" signals

### Graph Filtering

A graph filter H is defined spectrally:

    H f = U h(Lambda) U^T f

where h(Lambda) = diag(h(lambda_1), ..., h(lambda_n)) is the frequency response.

**Low-pass filter**: h(lambda) large for small lambda, small for large lambda.
Effect: smooths the signal on the graph.

**High-pass filter**: h(lambda) small for small lambda, large for large lambda.
Effect: extracts local variations (edge detection on graphs).

**Band-pass filter**: h(lambda) large only in a specific frequency range.

### Total Variation on Graphs

    TV(f) = sum_{(i,j) in E} w_{ij} |f_i - f_j|

This is the L1 analog of the Laplacian quadratic form (which uses squared differences).

**Graph total variation minimization** is used for:
- Denoising signals on graphs
- Semi-supervised learning (label propagation with sparse labels)
- Image segmentation (where the graph represents pixel adjacency)

---

## 9. Applications

### Community Detection

**Modularity matrix**: B = A - (d d^T) / (2m) where m = sum_ij w_ij / 2.

**Spectral modularity**: Find the partition maximizing Q = (1/(4m)) s^T B s where s in {-1, +1}^n.

Relaxation: s = leading eigenvector of B, then threshold to obtain partition.

### Semi-Supervised Learning

Given labels for a subset S subset V, propagate to unlabeled vertices:

    min_f sum_{(i,j) in E} w_{ij} (f_i - f_j)^2 + mu sum_{i in S} (f_i - y_i)^2

This is a regularized least squares on the graph. The solution involves the graph Laplacian:

    f* = mu (L + mu I_S)^{-1} y

where I_S is the diagonal matrix with 1s at labeled positions.

### Manifold Learning

**Laplacian Eigenmaps (Belkin & Niyogi, 2003)**:
1. Build a kNN or epsilon-neighborhood graph from data
2. Compute the graph Laplacian L
3. Embed: use the eigenvectors of L corresponding to the smallest nonzero eigenvalues
4. Map x_i to (v_2(i), v_3(i), ..., v_{k+1}(i))

**Justification**: In the limit of infinite data on a manifold, the graph Laplacian
converges to the Laplace-Beltrami operator on the manifold. Its eigenfunctions provide
an optimal low-dimensional embedding.

**Connection to diffusion maps**: Use eigenvectors of D^{-1} A (random walk matrix) instead
of L. Equivalent to Laplacian eigenmaps with the random walk Laplacian.

### Graph Partitioning and Load Balancing

Spectral methods for balanced graph partitioning:
1. Compute the Fiedler vector v_2
2. Partition: S = {i : v_2(i) <= median(v_2)}, S^c = V \ S
3. Recursive spectral bisection for k-way partitioning

**Quality guarantee**: By the Cheeger inequality, the spectral cut is within a quadratic
factor of the optimal cut.

**Applications**: circuit layout, parallel computing load balancing, mesh partitioning for FEM.

---

## 10. Computational Aspects

### Computing Eigenvalues of Graph Laplacians

**Dense methods** (for small graphs, n < 10000):
- Full eigendecomposition: O(n^3)
- Libraries: numpy.linalg.eigh, scipy.linalg.eigh

**Sparse iterative methods** (for large graphs):
- Lanczos algorithm: computes the k smallest eigenvalues in O(k * |E|) time
- LOBPCG (Locally Optimal Block Preconditioned Conjugate Gradient): efficient for smallest eigenvalues
- Libraries: scipy.sparse.linalg.eigsh, ARPACK

**Approximation algorithms**:
- Power iteration: for the largest eigenvalue, O(|E|) per iteration
- Chebyshev polynomial approximation: approximate the spectral filter without eigendecomposition

### Scaling Spectral Clustering

For very large graphs (n > 10^6):
1. **Nystrom approximation**: Sample m << n landmarks, compute eigenvectors on the sampled graph,
   interpolate to full graph. Cost: O(m^3 + n * m * k)
2. **Random projection**: Project the Laplacian to lower dimension, cluster in projected space.
3. **Multi-level methods**: Coarsen the graph, cluster the coarsened graph, refine.
   METIS, Graclus algorithms.

### Numerical Stability

- Graph Laplacians are singular (lambda_1 = 0), so matrix inverses must handle the null space
- For normalized Laplacians, isolated vertices cause D^{-1/2} to have zero/infinite entries
- Add a small regularization: L_reg = L + epsilon * I to handle near-zero eigenvalues
- Use generalized eigenvalue solvers for L v = lambda D v rather than computing D^{-1/2} L D^{-1/2} explicitly

---

## 11. Connections to Other Fields

### Electrical Networks

The graph Laplacian is the admittance matrix of an electrical resistor network:
- Vertices = nodes, edges = resistors with conductance w_{ij}
- Effective resistance between nodes i, j: R_{ij} = (e_i - e_j)^T L^+ (e_i - e_j)
  where L^+ is the pseudo-inverse of L
- Connection to commute time of random walks: E[T_{ij} + T_{ji}] = 2m * R_{ij}

### Algebraic Topology

The graph Laplacian is the 0-th Hodge Laplacian of a simplicial complex:
- L_0 = B_1 B_1^T where B_1 is the incidence matrix (boundary operator)
- Higher-order Laplacians L_k = B_k^T B_k + B_{k+1} B_{k+1}^T capture k-dimensional topology
- Application: topological data analysis, higher-order GNNs (simplicial neural networks)

### Continuous Limit

In the limit of n -> inf data points sampled from a manifold M with density p:

    (4 pi t)^{d/2} / n * L_epsilon f(x) -> Delta_M f(x) + 2 * nabla log p(x) . nabla f(x)

where:
- L_epsilon is the graph Laplacian with Gaussian kernel bandwidth epsilon = sqrt(t)
- Delta_M is the Laplace-Beltrami operator on M
- d is the intrinsic dimension of M

This convergence result justifies using graph Laplacians for manifold learning.
