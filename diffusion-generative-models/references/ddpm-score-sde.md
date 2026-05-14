# DDPM, Score Matching, and Score SDE Foundations

## Overview

This reference covers the mathematical foundations of diffusion-based generative models,
from the discrete-time DDPM formulation through continuous-time score SDEs. The three
formulations are unified under the score SDE framework (Song et al., 2021).

---

## 1. Denoising Diffusion Probabilistic Models (DDPMs)

### 1.1 Forward Process (Diffusion)

The forward process defines a Markov chain that gradually adds Gaussian noise to data
x_0 ~ q(x_0) over T timesteps.

**Single-step transition:**
```
q(x_t | x_{t-1}) = N(x_t; sqrt(1 - beta_t) * x_{t-1}, beta_t * I)
```

where beta_t is the noise schedule with 0 < beta_1 < beta_2 < ... < beta_T < 1.

**Cumulative notation:**
Define alpha_t = 1 - beta_t and alpha_bar_t = prod_{s=1}^{t} alpha_s.

**Closed-form sampling at arbitrary timestep:**
```
q(x_t | x_0) = N(x_t; sqrt(alpha_bar_t) * x_0, (1 - alpha_bar_t) * I)
```

Equivalently via reparameterization:
```
x_t = sqrt(alpha_bar_t) * x_0 + sqrt(1 - alpha_bar_t) * epsilon,  epsilon ~ N(0, I)
```

This means we can sample x_t directly from x_0 without iterating through all steps,
which is critical for efficient training.

**Signal-to-noise ratio (SNR):**
```
SNR(t) = alpha_bar_t / (1 - alpha_bar_t)
```

SNR decreases monotonically from infinity (t=0, pure signal) to near-zero (t=T, pure noise).

### 1.2 Noise Schedules

**Linear schedule (Ho et al., 2020):**
```
beta_t = beta_1 + (t-1)/(T-1) * (beta_T - beta_1)
beta_1 = 1e-4, beta_T = 0.02, T = 1000
```

**Cosine schedule (Nichol & Dhariwal, 2021):**
```
alpha_bar_t = f(t) / f(0)
f(t) = cos((t/T + s) / (1 + s) * pi/2)^2
s = 0.008 (small offset to prevent beta_t from being too small near t=0)
```

The cosine schedule provides a more gradual transition in SNR, particularly
beneficial for low-resolution images where the linear schedule destroys too much
information too quickly in early steps.

**Learned schedule (Kingma et al., 2021):**
Parameterize the noise schedule as a monotonic neural network and optimize jointly
with the diffusion model. Enables task-specific noise scheduling.

### 1.3 Reverse Process (Denoising)

The reverse process is a learned Markov chain starting from p(x_T) = N(0, I):
```
p_theta(x_{0:T}) = p(x_T) * prod_{t=1}^{T} p_theta(x_{t-1} | x_t)
p_theta(x_{t-1} | x_t) = N(x_{t-1}; mu_theta(x_t, t), Sigma_theta(x_t, t))
```

### 1.4 Forward Process Posterior

When conditioned on x_0, the forward process posterior is tractable:
```
q(x_{t-1} | x_t, x_0) = N(x_{t-1}; mu_tilde_t(x_t, x_0), beta_tilde_t * I)
```

where:
```
mu_tilde_t = (sqrt(alpha_bar_{t-1}) * beta_t) / (1 - alpha_bar_t) * x_0
           + (sqrt(alpha_t) * (1 - alpha_bar_{t-1})) / (1 - alpha_bar_t) * x_t

beta_tilde_t = (1 - alpha_bar_{t-1}) / (1 - alpha_bar_t) * beta_t
```

### 1.5 Variational Lower Bound (VLB)

The negative log-likelihood is bounded by the VLB:
```
-log p_theta(x_0) <= L_VLB = L_0 + L_1 + ... + L_{T-1} + L_T
```

where:
```
L_0 = -log p_theta(x_0 | x_1)                              [reconstruction term]
L_{t-1} = KL(q(x_{t-1}|x_t, x_0) || p_theta(x_{t-1}|x_t))  [denoising matching]
L_T = KL(q(x_T|x_0) || p(x_T))                              [prior matching, no params]
```

Each KL term L_{t-1} compares two Gaussians:
```
L_{t-1} = (1 / (2*sigma_t^2)) * ||mu_tilde_t(x_t, x_0) - mu_theta(x_t, t)||^2 + C
```

### 1.6 Loss Derivation: From VLB to Simplified Loss

**Step 1**: Parameterize mu_theta in terms of noise prediction:
```
mu_theta(x_t, t) = (1/sqrt(alpha_t)) * (x_t - (beta_t / sqrt(1 - alpha_bar_t)) * epsilon_theta(x_t, t))
```

**Step 2**: Substitute into L_{t-1}:
```
L_{t-1} = (beta_t^2) / (2 * sigma_t^2 * alpha_t * (1 - alpha_bar_t)) * ||epsilon - epsilon_theta(x_t, t)||^2
```

**Step 3**: Simplified loss (Ho et al., 2020) drops the weighting:
```
L_simple = E_{t, x_0, epsilon} [||epsilon - epsilon_theta(x_t, t)||^2]
```

where t ~ Uniform({1, ..., T}), x_0 ~ q(x_0), epsilon ~ N(0, I), and
x_t = sqrt(alpha_bar_t)*x_0 + sqrt(1-alpha_bar_t)*epsilon.

This simplified loss down-weights high-noise timesteps relative to the VLB weighting,
which empirically improves sample quality at the expense of slightly worse likelihood.

### 1.7 Training Algorithm (DDPM)

```
repeat:
    x_0 ~ q(x_0)                               # sample data
    t ~ Uniform({1, ..., T})                    # sample timestep
    epsilon ~ N(0, I)                           # sample noise
    x_t = sqrt(alpha_bar_t)*x_0 + sqrt(1-alpha_bar_t)*epsilon
    take gradient step on ||epsilon - epsilon_theta(x_t, t)||^2
until converged
```

### 1.8 Sampling Algorithm (DDPM)

```
x_T ~ N(0, I)
for t = T, T-1, ..., 1:
    z ~ N(0, I) if t > 1, else z = 0
    x_{t-1} = (1/sqrt(alpha_t)) * (x_t - (beta_t/sqrt(1-alpha_bar_t)) * epsilon_theta(x_t, t)) + sigma_t * z
return x_0
```

where sigma_t = sqrt(beta_t) or sigma_t = sqrt(beta_tilde_t).

---

## 2. Score-Based Generative Models (NCSN / SMLD)

### 2.1 Score Function

The score function is the gradient of the log-density:
```
s(x) = nabla_x log p(x)
```

The score function points toward regions of higher probability density and encodes
the shape of the distribution without requiring the normalizing constant.

### 2.2 Score Matching Objectives

**Explicit score matching (Hyvarinen, 2005):**
```
J(theta) = (1/2) * E_p(x) [||s_theta(x) - nabla_x log p(x)||^2]
         = E_p(x) [(1/2)||s_theta(x)||^2 + tr(nabla_x s_theta(x))] + C
```

The trace of the Jacobian is expensive; alternatives exist.

**Denoising score matching (Vincent, 2011):**
```
J_DSM(theta) = (1/2) * E_{q_sigma(x_tilde|x) p(x)} [||s_theta(x_tilde) - nabla_{x_tilde} log q_sigma(x_tilde|x)||^2]
```

For Gaussian noise q_sigma(x_tilde|x) = N(x_tilde; x, sigma^2 I):
```
nabla_{x_tilde} log q_sigma(x_tilde|x) = -(x_tilde - x) / sigma^2
J_DSM(theta) = E [||s_theta(x_tilde) + (x_tilde - x)/sigma^2||^2]
```

**Sliced score matching (Song et al., 2020):**
Random projection reduces Jacobian computation:
```
J_SSM(theta) = E_{p(v)} E_{p(x)} [v^T nabla_x s_theta(x) v + (1/2)(v^T s_theta(x))^2]
```

### 2.3 Noise Conditional Score Network (NCSN)

For a sequence of noise levels sigma_1 > sigma_2 > ... > sigma_L:
```
Perturbed distribution: q_sigma(x) = integral p(y) N(x; y, sigma^2 I) dy
Score network: s_theta(x, sigma_i) ~ nabla_x log q_{sigma_i}(x)
```

**Multi-scale denoising score matching loss:**
```
L = (1/L) * sum_{i=1}^{L} lambda(sigma_i) * E_{q_{sigma_i}(x_tilde|x) p(x)} [||s_theta(x_tilde, sigma_i) + (x_tilde - x)/sigma_i^2||^2]
```

where lambda(sigma_i) = sigma_i^2 for uniform weighting.

### 2.4 Annealed Langevin Dynamics

**Sampling procedure:**
```
for i = 1, 2, ..., L:          # iterate over noise levels (decreasing)
    alpha_i = epsilon * sigma_i^2 / sigma_L^2    # step size
    for k = 1, 2, ..., K:      # Langevin steps per noise level
        z ~ N(0, I)
        x <- x + (alpha_i/2) * s_theta(x, sigma_i) + sqrt(alpha_i) * z
return x
```

The key insight: start sampling at high noise levels (where the score is well-defined
and the landscape is smooth) and progressively decrease noise for refinement.

### 2.5 Connection Between DDPM and NCSN

The DDPM noise prediction and the score function are related:
```
s_theta(x_t, t) = -epsilon_theta(x_t, t) / sqrt(1 - alpha_bar_t)
```

Both frameworks learn to denoise, but:
- DDPM predicts the noise epsilon added to clean data
- NCSN predicts the score (gradient of log-density) of the noisy distribution

---

## 3. Score SDE Framework (Song et al., 2021)

### 3.1 Continuous-Time Forward Process

The forward process is described by an Ito SDE:
```
dx = f(x, t) dt + g(t) dw
```

where:
- f(x, t): drift coefficient
- g(t): diffusion coefficient (scalar)
- w: standard Wiener process (Brownian motion)
- t in [0, T], x(0) ~ p_0 (data distribution), x(T) ~ p_T (noise prior)

### 3.2 Specific SDE Instantiations

**VP-SDE (Variance Preserving) -- corresponds to DDPM:**
```
dx = -(1/2) * beta(t) * x * dt + sqrt(beta(t)) * dw
f(x, t) = -(1/2) * beta(t) * x
g(t) = sqrt(beta(t))
```

Continuous beta(t) from beta_min = 0.1 to beta_max = 20 over t in [0, 1].
Transition kernel: p_{0t}(x_t|x_0) = N(x_t; x_0 * exp(-1/2 * integral_0^t beta(s)ds), I * (1 - exp(-integral_0^t beta(s)ds)))

**VE-SDE (Variance Exploding) -- corresponds to NCSN/SMLD:**
```
dx = sqrt(d/dt [sigma(t)^2]) * dw
f(x, t) = 0
g(t) = sqrt(d/dt [sigma(t)^2]) = sigma(t) * sqrt(2 * log(sigma_max/sigma_min))
```

where sigma(t) = sigma_min * (sigma_max/sigma_min)^t, geometric noise schedule.
Transition kernel: p_{0t}(x_t|x_0) = N(x_t; x_0, [sigma(t)^2 - sigma_min^2] * I)

**Sub-VP SDE:**
```
dx = -(1/2) * beta(t) * x * dt + sqrt(beta(t) * (1 - exp(-2*integral_0^t beta(s)ds))) * dw
```

Provides tighter ELBO than VP-SDE due to lower variance.

### 3.3 Reverse-Time SDE

Anderson (1982) showed that the reverse of an SDE is also an SDE:
```
dx = [f(x, t) - g(t)^2 * nabla_x log p_t(x)] dt + g(t) dw_bar
```

where:
- dt is now negative (time runs backward from T to 0)
- dw_bar is a reverse-time Wiener process
- nabla_x log p_t(x) is the score function of the marginal distribution at time t

The score function nabla_x log p_t(x) is the only unknown -- we train a neural
network s_theta(x, t) to approximate it.

### 3.4 Probability Flow ODE

By removing the stochastic term, we get a deterministic ODE with identical marginals:
```
dx = [f(x, t) - (1/2) * g(t)^2 * nabla_x log p_t(x)] dt
```

Properties of the probability flow ODE:
- Same marginal distributions p_t(x) at every time t as the SDE
- Deterministic: given x_T, the trajectory is uniquely determined
- Enables exact log-likelihood computation via the instantaneous change of variables formula
- Enables interpolation in latent space
- Can use adaptive ODE solvers for efficient sampling

**Log-likelihood via instantaneous change of variables:**
```
log p_0(x_0) = log p_T(x_T) + integral_0^T nabla . [f(x,t) - (1/2) g(t)^2 s_theta(x,t)] dt
```

### 3.5 Score Matching Training Objective

The continuous-time score matching objective:
```
L = E_t { lambda(t) * E_{x_0} E_{x_t|x_0} [||s_theta(x_t, t) - nabla_{x_t} log p_{0t}(x_t|x_0)||^2] }
```

where t ~ Uniform(0, T) and lambda(t) is a positive weighting function.

Since p_{0t}(x_t|x_0) is Gaussian (known for VP/VE/sub-VP SDEs):
```
nabla_{x_t} log p_{0t}(x_t|x_0) = -(x_t - mean(t)) / variance(t)
```

For VP-SDE:
```
= -(x_t - sqrt(alpha_bar_t) * x_0) / (1 - alpha_bar_t)
= -epsilon / sqrt(1 - alpha_bar_t)
```

### 3.6 Unified Framework Summary

| Framework | Discrete/Continuous | Noise | Key Object | Sampling |
|-----------|-------------------|-------|------------|----------|
| DDPM | Discrete (T steps) | Fixed schedule | epsilon_theta | Ancestral |
| NCSN | Discrete (L levels) | Geometric | s_theta | Annealed Langevin |
| VP-SDE | Continuous | Linear beta(t) | s_theta(x,t) | Reverse SDE/ODE |
| VE-SDE | Continuous | Geometric sigma(t) | s_theta(x,t) | Reverse SDE/ODE |

All are instances of: learn the score, then simulate the reverse process.

---

## 4. Training Objectives: Prediction Targets

### 4.1 Noise Prediction (epsilon-prediction)

Standard DDPM objective:
```
L_epsilon = E [||epsilon - epsilon_theta(x_t, t)||^2]
```
Model predicts the noise that was added. Most common parameterization.

### 4.2 Score Prediction

```
L_score = E [||nabla_x log p_t(x_t) - s_theta(x_t, t)||^2]
```
Equivalent to noise prediction with rescaling: s = -epsilon / sqrt(1 - alpha_bar_t).

### 4.3 x_0 Prediction

```
L_x0 = E [||x_0 - x_theta(x_t, t)||^2]
```
Direct clean data prediction. Useful when you need the predicted x_0 (e.g., for thresholding).
Can be converted: epsilon = (x_t - sqrt(alpha_bar_t) * x_theta) / sqrt(1 - alpha_bar_t).

### 4.4 v-Prediction (Salimans & Ho, 2022)

Define v = alpha_t * epsilon - sigma_t * x_0 (where sigma_t = sqrt(1 - alpha_bar_t)):
```
L_v = E [||v - v_theta(x_t, t)||^2]
```

Advantages:
- Better for progressive distillation (more stable targets)
- Naturally handles the transition from signal-dominated to noise-dominated regimes
- Used in Imagen Video, stable diffusion v2

Recovery: x_0 = alpha_t * x_t - sigma_t * v_theta, epsilon = sigma_t * x_t + alpha_t * v_theta.

### 4.5 Loss Weighting Strategies

**Uniform weighting**: L = E_t [||target - prediction||^2], t ~ Uniform.

**SNR weighting**: weight each timestep proportional to SNR or 1/SNR.

**Min-SNR-gamma (Hang et al., 2023):**
```
weight(t) = min(SNR(t), gamma) / SNR(t)
gamma = 5 (recommended)
```
Clips the weight for high-SNR (low-noise) timesteps to prevent them from dominating.
Speeds up convergence significantly.

**P2 weighting (Choi et al., 2022):**
```
weight(t) = 1 / (k + SNR(t))^gamma
```
Down-weights high-SNR steps to focus on challenging timesteps.

---

## 5. Network Architecture

### 5.1 U-Net (Standard for Diffusion)

The standard architecture used in DDPM and most diffusion models:

```
Input: x_t (noisy image), t (timestep)

Timestep embedding:
- Sinusoidal positional encoding of t
- Two linear layers with SiLU activation
- Injected via scale-shift in GroupNorm layers

Encoder path (downsampling):
- ResNet blocks with GroupNorm + SiLU
- Self-attention at lower resolutions (16x16, 8x8)
- Downsampling via strided convolutions

Bottleneck:
- ResNet block + self-attention + ResNet block

Decoder path (upsampling):
- Skip connections from encoder (concatenation)
- ResNet blocks with GroupNorm + SiLU
- Self-attention at corresponding resolutions
- Upsampling via transposed convolutions or nearest-neighbor + conv

Output: epsilon_theta (predicted noise), same shape as input
```

### 5.2 Conditioning via Cross-Attention

For text-conditioned models, add cross-attention layers:
```
Q = W_Q * h       (from diffusion U-Net features)
K = W_K * c       (from text encoder output)
V = W_V * c       (from text encoder output)
Attention(Q, K, V) = softmax(Q K^T / sqrt(d_k)) V
```

### 5.3 Diffusion Transformer (DiT)

Replace U-Net with vision transformer (Peebles & Xie, 2023):
- Patchify input image into tokens
- Condition on timestep and class via adaptive layer norm (adaLN-Zero)
- Standard transformer blocks with self-attention
- Better scaling properties than U-Net

---

## 6. Key Implementation Details

### 6.1 EMA (Exponential Moving Average)

Maintain an EMA of model weights for evaluation:
```
theta_EMA <- decay * theta_EMA + (1 - decay) * theta
decay = 0.9999 (typical)
```

EMA weights produce significantly better samples than raw training weights.

### 6.2 Variance Reduction in Training

- Use the closed-form q(x_t|x_0) rather than iterative forward process
- Antithetic sampling: pair each t with T-t
- Importance sampling of timesteps based on loss magnitude

### 6.3 Mixed Precision Training

- FP16/BF16 for most computations
- FP32 for variance-sensitive operations (normalization, loss)
- Gradient scaling to prevent underflow

---

## Source Citations

- Ho, J., Jain, A., & Abbeel, P. (2020). "Denoising Diffusion Probabilistic Models." NeurIPS 2020.
- Song, Y. & Ermon, S. (2019). "Generative Modeling by Estimating Gradients of the Data Distribution." NeurIPS 2019.
- Song, Y., Sohl-Dickstein, J., Kingma, D.P., Kumar, A., Ermon, S., & Poole, B. (2021). "Score-Based Generative Modeling through Stochastic Differential Equations." ICLR 2021.
- Nichol, A. & Dhariwal, P. (2021). "Improved Denoising Diffusion Probabilistic Models." ICML 2021.
- Salimans, T. & Ho, J. (2022). "Progressive Distillation for Fast Sampling of Diffusion Models." ICLR 2022.
- Yang, L. et al. (2023). "Diffusion Models: A Comprehensive Survey." ACM Computing Surveys.
