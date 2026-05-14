# Sampling Methods for Diffusion Models

## Overview

Efficient sampling is critical for practical deployment of diffusion models. While
DDPMs originally required 1000 steps, modern solvers achieve comparable quality in
10-50 steps. This reference covers the major sampling methods, their mathematical
foundations, and practical configuration guidance.

---

## 1. DDIM (Denoising Diffusion Implicit Models)

### 1.1 Core Idea

Song et al. (2021) showed that the DDPM training objective corresponds to a family
of non-Markovian forward processes, enabling deterministic sampling.

### 1.2 Generalized Forward Process

Define a non-Markovian forward process parameterized by sigma:
```
q_sigma(x_{t-1} | x_t, x_0) = N(x_{t-1}; mu, sigma_t^2 I)
```

where:
```
mu = sqrt(alpha_bar_{t-1}) * x_0 + sqrt(1 - alpha_bar_{t-1} - sigma_t^2) * (x_t - sqrt(alpha_bar_t) * x_0) / sqrt(1 - alpha_bar_t)
```

When sigma_t = sqrt((1-alpha_bar_{t-1})/(1-alpha_bar_t)) * sqrt(beta_t), we recover DDPM.
When sigma_t = 0, we get DDIM (fully deterministic).

### 1.3 DDIM Update Rule

```
x_{t-1} = sqrt(alpha_bar_{t-1}) * predicted_x_0 + sqrt(1 - alpha_bar_{t-1}) * direction_pointing_to_x_t
```

where:
```
predicted_x_0 = (x_t - sqrt(1 - alpha_bar_t) * epsilon_theta(x_t, t)) / sqrt(alpha_bar_t)
direction = (x_t - sqrt(alpha_bar_t) * predicted_x_0) / sqrt(1 - alpha_bar_t)
         = epsilon_theta(x_t, t)   [the predicted noise]
```

Full update:
```
x_{t-1} = sqrt(alpha_bar_{t-1}) * ((x_t - sqrt(1-alpha_bar_t) * epsilon_theta) / sqrt(alpha_bar_t))
        + sqrt(1 - alpha_bar_{t-1}) * epsilon_theta
```

### 1.4 Stochastic DDIM (eta parameter)

Interpolate between deterministic (eta=0) and stochastic (eta=1 = DDPM):
```
sigma_t = eta * sqrt((1-alpha_bar_{t-1})/(1-alpha_bar_t)) * sqrt(1 - alpha_bar_t/alpha_bar_{t-1})

x_{t-1} = sqrt(alpha_bar_{t-1}) * predicted_x_0
         + sqrt(1 - alpha_bar_{t-1} - sigma_t^2) * epsilon_theta(x_t, t)
         + sigma_t * z,    z ~ N(0, I)
```

### 1.5 Subsequence Sampling

DDIM supports using a subsequence of timesteps tau = [tau_1, tau_2, ..., tau_S] where S << T:
```
Example: T=1000, S=50
tau = [1, 21, 41, 61, ..., 981]  (evenly spaced)
```

Apply DDIM updates only at these timesteps, skipping intermediate ones.

### 1.6 Properties

- **Deterministic**: Given x_T, output x_0 is deterministic (eta=0)
- **Consistent**: Same model weights, just different sampler
- **Interpolation**: Meaningful interpolation in x_T space
- **Inversion**: Can encode x_0 -> x_T (useful for editing)
- **Steps**: 50-100 steps for good quality, 10-20 acceptable

---

## 2. DPM-Solver (Lu et al., 2022)

### 2.1 Motivation

View diffusion sampling as solving the probability flow ODE:
```
dx/dt = f(x,t) - (1/2) g(t)^2 * s_theta(x,t)
```

Apply high-order ODE solvers for fewer steps with lower discretization error.

### 2.2 Change of Variables

Define lambda_t = log(alpha_t / sigma_t) (log-SNR, monotonically decreasing):
```
x_t = alpha_t/alpha_s * x_s - alpha_t * integral_{lambda_s}^{lambda_t} exp(-lambda) * epsilon_theta(x_lambda, lambda) d_lambda
```

This exact solution separates the linear (scaling) and nonlinear (noise prediction) parts.

### 2.3 DPM-Solver-1 (First Order)

Approximate epsilon_theta as constant over the interval:
```
x_{t_{i-1}} = (alpha_{t_{i-1}} / alpha_{t_i}) * x_{t_i}
             - sigma_{t_{i-1}} * (exp(h_i) - 1) * epsilon_theta(x_{t_i}, t_i)
```

where h_i = lambda_{t_{i-1}} - lambda_{t_i}. Equivalent to DDIM.

### 2.4 DPM-Solver-2 (Second Order)

Use a first-order Taylor expansion of epsilon_theta:
```
s_i = t_{i-1} + r * (t_i - t_{i-1}),  r = 0.5 (midpoint)

# Compute intermediate point
u = (alpha_{s_i} / alpha_{t_i}) * x_{t_i} - sigma_{s_i} * (exp(r*h_i) - 1) * epsilon_theta(x_{t_i}, t_i)

# Compute final point using the gradient estimate
x_{t_{i-1}} = (alpha_{t_{i-1}} / alpha_{t_i}) * x_{t_i}
             - sigma_{t_{i-1}} * (exp(h_i) - 1) * epsilon_theta(x_{t_i}, t_i)
             - sigma_{t_{i-1}} * (exp(h_i) - 1) / (2*r*h_i)
               * (epsilon_theta(u, s_i) - epsilon_theta(x_{t_i}, t_i))
```

Each step requires 2 function evaluations (NFE=2 per step).

### 2.5 DPM-Solver-3 (Third Order)

Uses two intermediate evaluations per step for cubic accuracy:
- NFE = 3 per step
- Typically unnecessary; DPM-Solver-2 is the sweet spot

### 2.6 DPM-Solver++ (Lu et al., 2022)

Modifications for guided sampling (high guidance scales):

**Data prediction formulation**: Reparameterize in terms of x_0 prediction:
```
x_theta(x_t, t) = (x_t - sigma_t * epsilon_theta(x_t, t)) / alpha_t
```

**Dynamic thresholding** (from Imagen):
```
s = percentile(|x_theta|, p),  p = 99.5 typically
x_theta_clipped = clip(x_theta, -s, s) / s
```

Prevents the predicted x_0 from going far out of range at high guidance scales.

**Multistep variant**: Reuse previous function evaluations across steps:
- DPM-Solver++(2M): multistep second-order, 1 NFE per step
- Most efficient practical choice for guided diffusion

---

## 3. Exponential Integrators

### 3.1 Concept

The probability flow ODE has a linear component (from the drift f(x,t)):
```
dx/dt = A(t)*x + B(t)*epsilon_theta(x,t)
```

Solve the linear part exactly, only approximate the nonlinear part:
```
x_{t-1} = exp(integral A) * x_t + integral exp(integral A) * B * epsilon_theta dt
```

### 3.2 Advantages

- Exact treatment of linear dynamics reduces error
- Particularly effective for VP-SDE where the drift is linear in x
- Foundation for DPM-Solver family

---

## 4. UniPC (Zhao et al., 2023)

### 4.1 Unified Predictor-Corrector Framework

Combines ideas from DPM-Solver and PNDM into a single framework:

**Predictor step**: any ODE solver (Euler, DPM-Solver-2, etc.)
**Corrector step**: use the new function evaluation to refine without extra cost

### 4.2 Variant B (Recommended)

```
Predictor: DPM-Solver-2 style prediction
Corrector: use model output at predicted point to correct via multistep formula
```

- Achieves DPM-Solver-2 quality with fewer NFE
- 10 steps sufficient for good quality
- Compatible with any diffusion model

---

## 5. Consistency Models (Song et al., 2023)

### 5.1 Core Idea

Learn a function f_theta that maps any point on the ODE trajectory to the trajectory's
origin (x_0):
```
f_theta(x_t, t) = x_0    for all t on the same trajectory
```

**Self-consistency property**: f_theta(x_t, t) = f_theta(x_s, s) for all s, t on same trajectory.

### 5.2 Boundary Condition

At t = epsilon (near zero): f_theta(x_epsilon, epsilon) = x_epsilon (identity).

Enforced via skip connection:
```
f_theta(x, t) = c_skip(t) * x + c_out(t) * F_theta(x, t)
c_skip(epsilon) = 1, c_out(epsilon) = 0
```

### 5.3 Consistency Distillation (CD)

Distill from a pretrained diffusion model using the ODE solver:
```
L_CD = E [d(f_theta(x_{t_{n+1}}, t_{n+1}), f_{theta^-}(x_hat_{t_n}, t_n))]
```

where x_hat_{t_n} is obtained by one ODE step from x_{t_{n+1}} using the teacher model,
theta^- is the EMA of theta, and d is a distance metric (LPIPS recommended).

### 5.4 Consistency Training (CT)

Train without a pretrained teacher, using the forward process directly:
```
L_CT = E [d(f_theta(x + sigma_{t_{n+1}} * z, t_{n+1}), f_{theta^-}(x + sigma_{t_n} * z, t_n))]
```

Relies on adjacent noise levels having similar score functions.

### 5.5 Generation

**Single-step**: x_0 = f_theta(x_T, T), where x_T ~ N(0, sigma_max^2 I)

**Multi-step (improved quality)**: Alternate between denoising and re-noising:
```
x_0_hat = f_theta(x_T, T)
x_{t'} = x_0_hat + sigma_{t'} * z    (re-noise to intermediate level)
x_0_hat = f_theta(x_{t'}, t')         (denoise again)
... repeat for desired quality
```

### 5.6 Properties

- Single-step generation possible (vs 10-1000 for other methods)
- Quality improves with more steps (1-step < 2-step < 4-step)
- Competitive with DDPM at 2-4 steps on CIFAR-10
- Training can be unstable; requires careful hyperparameter tuning

---

## 6. Progressive Distillation (Salimans & Ho, 2022)

### 6.1 Method

Iteratively halve the number of sampling steps:

```
Round 1: Teacher uses N steps -> Student learns to match in N/2 steps
Round 2: Student becomes teacher -> New student learns N/4 steps
...
Continue until reaching target (e.g., 4 steps)
```

### 6.2 Training

Student takes one step to match teacher's two steps:
```
Teacher: x_t -> x_{t-1} -> x_{t-2}   (two DDIM steps)
Student: x_t -> x_{t-2}              (one step, trained to match)

L = ||student_output - teacher_two_step_output||^2
```

### 6.3 v-Prediction

Progressive distillation works best with v-prediction parameterization:
```
v = alpha_t * epsilon - sigma_t * x_0
```

This provides more stable training targets during distillation compared
to epsilon-prediction.

### 6.4 Results

- 4-step sampling achieves FID close to 1000-step DDPM on ImageNet
- Each distillation round roughly doubles the gap from the teacher
- Combined with classifier-free guidance for conditional generation

---

## 7. TRACT (Berthelot et al., 2023)

### 7.1 Truncated Consistency Training

Similar to progressive distillation but uses consistency-like objective:
- Split the diffusion process into segments
- Train each segment independently with consistency loss
- Progressive: start with many segments, merge as training progresses

### 7.2 Advantages

- More stable than full consistency training
- Does not require a pretrained teacher (unlike consistency distillation)
- Competitive at 1-4 sampling steps

---

## 8. Karras et al. (2022) Noise Schedule for Sampling

### 8.1 Optimal Time Discretization

Karras et al. propose a specific noise level schedule for sampling:
```
sigma_i = (sigma_max^(1/rho) + i/(N-1) * (sigma_min^(1/rho) - sigma_max^(1/rho)))^rho
```

where rho = 7 (recommended), placing more steps at lower noise levels where
detail is generated.

### 8.2 Preconditioning

Standardize the network input and output:
```
D_theta(x; sigma) = c_skip(sigma) * x + c_out(sigma) * F_theta(c_in(sigma) * x; c_noise(sigma))
```

where c_skip, c_out, c_in, c_noise are chosen to keep inputs/outputs at unit variance.

### 8.3 Second-Order Heun's Method

```
d_i = (x_i - D_theta(x_i; sigma_i)) / sigma_i      # tangent at current point
x_{i+1} = x_i + (sigma_{i+1} - sigma_i) * d_i       # Euler step

If sigma_{i+1} != 0:
    d_i' = (x_{i+1} - D_theta(x_{i+1}; sigma_{i+1})) / sigma_{i+1}  # tangent at next point
    x_{i+1} = x_i + (sigma_{i+1} - sigma_i) * (d_i + d_i') / 2       # Heun correction
```

2 NFE per step, second-order accuracy.

---

## 9. Step-Quality Tradeoff

### 9.1 Comparison of Methods at Various Step Counts

| Method | 5 Steps | 10 Steps | 20 Steps | 50 Steps | 1000 Steps |
|--------|---------|----------|----------|----------|------------|
| DDPM (ancestral) | Very poor | Poor | Mediocre | Fair | Good (FID ~3-5) |
| DDIM (eta=0) | Poor | Fair | Good | Very good | Good |
| DPM-Solver-2 | Fair | Good | Very good | Excellent | Excellent |
| DPM-Solver++(2M) | Good | Very good | Excellent | Excellent | Excellent |
| UniPC | Good | Very good | Excellent | Excellent | Excellent |
| Consistency (1-step) | Fair | N/A | N/A | N/A | N/A |
| Consistency (2-step) | N/A | Good | N/A | N/A | N/A |
| Prog. Distill. (4-step) | N/A | Good | N/A | N/A | N/A |

### 9.2 General Guidance

- **1-4 steps**: Consistency models or heavily distilled models required
- **5-10 steps**: DPM-Solver++(2M) or UniPC recommended
- **10-25 steps**: Sweet spot for quality/speed; DPM-Solver++ or Karras Heun
- **25-50 steps**: Diminishing returns; only for maximum quality
- **50+ steps**: Rarely needed with modern solvers; legacy DDPM territory

---

## 10. Practical Sampling Configurations

### 10.1 Stable Diffusion (Latent Diffusion)

Recommended defaults:
```
Sampler: DPM-Solver++(2M) or Euler Ancestral
Steps: 20-30
CFG Scale: 7-8 (for v1.5), 5-7 (for SDXL)
Scheduler: Karras
Resolution: 512x512 (v1.5), 1024x1024 (SDXL)
```

### 10.2 Imagen / Pixel-Space Models

```
Sampler: DDPM (ancestral) or DDIM
Steps: 100-256 (base), 50-100 (super-resolution)
CFG Scale: 5-15
Dynamic thresholding: enabled with percentile 99.5
```

### 10.3 Video Diffusion Models

```
Sampler: DDPM or DDIM
Steps: 50-250 (higher for temporal coherence)
CFG Scale: 7-15
Temporal consistency: shared noise or temporal smoothing in latent space
```

### 10.4 Noise Schedule Selection for Sampling

| Schedule | Best For | Characteristics |
|----------|----------|----------------|
| Uniform (linear in t) | Legacy, DDPM default | Too many steps at high noise |
| Quadratic | Medium-step sampling | Better step allocation |
| Karras (rho=7) | General purpose | More steps at low noise |
| Log-linear | High-step counts | Even coverage in log-SNR |
| Trailing | Few-step distilled models | Skip high-noise region |

### 10.5 Stochasticity Settings

- **eta=0** (deterministic DDIM): Reproducible, good for interpolation/editing
- **eta=0.5-0.8**: Slight stochasticity improves diversity without quality loss
- **eta=1.0** (DDPM): Maximum diversity, requires more steps for quality
- **Churn** (Karras): Add controlled stochasticity at each step, improves quality

---

## 11. Advanced Techniques

### 11.1 Restart Sampling (Xu et al., 2023)

Add noise back to an intermediate level and re-denoise:
```
Denoise: x_T -> x_{t_restart}
Add noise: x_{t_restart} -> x_{t_restart + delta}
Re-denoise: x_{t_restart + delta} -> x_0
```

Mixes exploration (adding noise) with exploitation (denoising).
Improves diversity while maintaining quality.

### 11.2 Stochastic Churn (Karras et al., 2022)

At each step, optionally add a small amount of noise:
```
x_hat = x + sqrt(sigma_hat^2 - sigma^2) * z,  z ~ N(0, I)
sigma_hat = sigma + gamma * sigma
```

where gamma controls the churn amount. Improves sample quality
for certain model architectures.

### 11.3 Ancestral Sampling with Variance Correction

For stochastic samplers, correct the noise injection to match the
forward process variance exactly:
```
sigma_up = sqrt(sigma_{t-1}^2 - sigma_down^2)
sigma_down = sigma_{t-1} * sqrt(1 - (sigma_t / sigma_{t-1})^2) * ... [model-specific]
```

---

## Source Citations

- Song, J., Meng, C., & Ermon, S. (2021). "Denoising Diffusion Implicit Models." ICLR 2021.
- Lu, C., Zhou, Y., Bao, F., Chen, J., Li, C., & Zhu, J. (2022). "DPM-Solver: A Fast ODE Solver for Diffusion Probabilistic Model Sampling." NeurIPS 2022.
- Lu, C., Zhou, Y., Bao, F., Chen, J., Li, C., & Zhu, J. (2022). "DPM-Solver++: Fast Solver for Guided Sampling of Diffusion Probabilistic Models."
- Zhao, W., Bai, L., Rao, Y., Zhou, J., & Lu, J. (2023). "UniPC: A Unified Predictor-Corrector Framework for Fast Sampling of Diffusion Models." NeurIPS 2023.
- Song, Y., Dhariwal, P., Chen, M., & Sutskever, I. (2023). "Consistency Models." ICML 2023.
- Salimans, T. & Ho, J. (2022). "Progressive Distillation for Fast Sampling of Diffusion Models." ICLR 2022.
- Karras, T., Aittala, M., Aila, T., & Laine, S. (2022). "Elucidating the Design Space of Diffusion-Based Generative Models." NeurIPS 2022.
- Yang, L. et al. (2023). "Diffusion Models: A Comprehensive Survey." ACM Computing Surveys.
