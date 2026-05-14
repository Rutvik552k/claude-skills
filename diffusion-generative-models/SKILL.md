---
name: diffusion-generative-models
displayName: "Diffusion & Generative Models"
description: "Diffusion model theory, text-to-image/video generation architectures, efficient sampling, and conditioning/guidance techniques"
tags:
  - diffusion-models
  - ddpm
  - score-sde
  - text-to-image
  - text-to-video
  - dall-e
  - imagen
  - stable-diffusion
  - make-a-video
  - generative-ai
  - sampling
---

# Diffusion & Generative Models

## When to Activate

Use this skill when:
- Understanding diffusion model theory (DDPMs, score-based models, score SDEs)
- Comparing generative model architectures (VAE vs GAN vs diffusion vs flow-based)
- Working with text-to-image generation (DALL-E 2, Imagen, Stable Diffusion, CogView2)
- Working with text-to-video generation (Make-A-Video, Imagen Video, Phenaki, CogVideo)
- Implementing or optimizing efficient sampling strategies (DDIM, DPM-Solver, consistency models)
- Training diffusion models (loss functions, noise schedules, variance learning)
- Applying conditioning and guidance techniques (classifier guidance, classifier-free guidance, CLIP/T5 conditioning)
- Adapting diffusion models for scientific or domain-specific applications

## Topics Index

### 1. Diffusion Model Foundations

**DDPMs (Denoising Diffusion Probabilistic Models)**
- Forward process: q(x_t | x_{t-1}) = N(x_t; sqrt(1 - beta_t) * x_{t-1}, beta_t * I)
- Reverse process: p_theta(x_{t-1} | x_t) = N(x_{t-1}; mu_theta(x_t, t), Sigma_theta(x_t, t))
- Noise schedule: linear beta schedule from beta_1 = 1e-4 to beta_T = 0.02, T = 1000
- Variational bound: L_VLB = sum of KL divergences between forward and reverse posteriors
- Simplified loss: L_simple = E[||epsilon - epsilon_theta(x_t, t)||^2]
- See: `references/ddpm-score-sde.md`

**Score-Based Generative Models**
- Score function: s_theta(x) ~ nabla_x log p(x)
- Denoising score matching: train with perturbed data at multiple noise levels
- Langevin dynamics sampling: x_{t+1} = x_t + (epsilon/2) * nabla_x log p(x_t) + sqrt(epsilon) * z
- Annealed Langevin dynamics: decreasing noise levels for progressive refinement
- See: `references/ddpm-score-sde.md`

**Score SDEs (Stochastic Differential Equations)**
- Forward SDE: dx = f(x, t)dt + g(t)dw (continuous-time diffusion)
- Reverse-time SDE: dx = [f(x,t) - g(t)^2 * nabla_x log p_t(x)]dt + g(t)dw_bar
- Probability flow ODE: dx = [f(x,t) - (1/2)g(t)^2 * nabla_x log p_t(x)]dt
- Unified framework: DDPMs, score-based models, and SDEs as special cases
- VP-SDE (Variance Preserving): corresponds to DDPM
- VE-SDE (Variance Exploding): corresponds to NCSN/SMLD
- Sub-VP SDE: lower variance bounds
- See: `references/ddpm-score-sde.md`

**Training Objectives**
- Simplified loss (Ho et al.): predict noise epsilon from noisy input
- Noise prediction vs score prediction: epsilon_theta ~ -sigma_t * s_theta
- v-prediction (Salimans & Ho): predict v = alpha_t * epsilon - sigma_t * x
- Weighting strategies: uniform, SNR-based, min-SNR-gamma
- See: `references/ddpm-score-sde.md`

### 2. Efficient Sampling

**SDE Solvers**
- Euler-Maruyama: first-order SDE discretization
- Predictor-corrector: combine SDE solver with Langevin corrector steps
- Ancestral sampling: stochastic reverse process with scheduled noise

**ODE Solvers**
- DDIM (Denoising Diffusion Implicit Models): deterministic sampling via non-Markovian process
- DPM-Solver: high-order ODE solver for diffusion, 10-20 steps sufficient
- DPM-Solver++: improved version with thresholding for guided sampling
- Exponential integrators: exact solution of linear part
- UniPC: unified predictor-corrector framework

**Knowledge Distillation**
- Progressive distillation: halve steps iteratively (1024 -> 512 -> ... -> 4)
- Consistency models (Song et al.): single-step generation via self-consistency
- Consistency training vs consistency distillation
- TRACT: progressive knowledge transfer

**Adaptive Step Sizes**
- Error-based step size control
- Karras et al. noise schedule for sampling
- See: `references/sampling-methods.md`

### 3. Improved Likelihood & Training

**Noise Schedule Optimization**
- Linear schedule (Ho et al., 2020): simple but suboptimal at endpoints
- Cosine schedule (Nichol & Dhariwal, 2021): smoother SNR transition, better for low resolution
- Learned schedules: parameterize noise schedule and optimize jointly
- Shifted schedules for high resolution (Hoogeboom et al.)

**Variance Learning**
- Fixed variance: sigma_t^2 = beta_t or sigma_t^2 = beta_tilde_t
- Learned reverse variance (Nichol & Dhariwal): interpolate in log-space between bounds
- Improved ELBO: hybrid objective combining L_simple and L_VLB
- Variational diffusion models: continuous-time formulation with learned schedule

**Classifier-Free Guidance**
- Conditional model: epsilon_theta(x_t, t, c)
- Unconditional model: epsilon_theta(x_t, t, empty)
- Guided prediction: epsilon_hat = epsilon_uncond + w * (epsilon_cond - epsilon_uncond)
- Guidance scale w: typically 3-15, higher = more adherence to conditioning, less diversity
- Training: randomly drop conditioning with probability p_uncond (typically 10-20%)
- See: `references/guidance-conditioning.md`

### 4. Special Architectures

**Discrete Diffusion**
- D3PM (Structured Denoising Diffusion Models): categorical distributions, transition matrices
- Multinomial diffusion: uniform noise for categorical data
- Absorbing state diffusion: mask tokens as absorbing state
- Applications: text generation, discrete molecular graphs

**Equivariant/Invariant Diffusion**
- Molecular generation: SE(3)-equivariant diffusion
- EDM (Equivariant Diffusion Model): atom coordinates + types
- GeoDiff: torsion-aware molecular conformation
- DiffDock: protein-ligand docking

**Manifold Diffusion**
- Riemannian score matching: diffusion on curved spaces
- Riemannian Flow Matching: CNFs on manifolds
- Applications: protein structure prediction, SO(3) rotations, hyperbolic embeddings

### 5. Connections to Other Generative Models

**VAEs (Variational Autoencoders)**
- Diffusion as hierarchical VAE with T latent layers
- Shared ELBO structure: reconstruction + KL terms
- Diffusion fixes posterior collapse by constraining each step

**GANs (Generative Adversarial Networks)**
- Hybrid approaches: diffusion + adversarial loss (denoising diffusion GANs)
- GANs for few-step generation after diffusion pretraining
- Discriminator guidance as alternative to classifier guidance

**Normalizing Flows**
- Flow matching (Lipman et al.): regress vector fields, simpler than score matching
- Rectified flows: straight-line OT paths
- Continuous normalizing flows (CNFs) as probability flow ODE equivalent
- Stochastic interpolants: unified framework for flows and diffusions

**Autoregressive Models**
- Order-agnostic training: diffusion over sequence of tokens
- Discrete diffusion for text: masked language model connection
- Hybrid AR + diffusion: coarse-to-fine generation

**Energy-Based Models**
- Score matching connection: s_theta(x) = nabla_x log p_theta(x)
- EBMs trained via denoising score matching
- Shared Langevin dynamics sampling

**LLMs (Large Language Models)**
- Diffusion for discrete tokens: MDLM, SEDD, CDCD
- Continuous diffusion in embedding space
- Plaid: fast discrete diffusion language model
- Non-autoregressive text generation via diffusion

### 6. Text-to-Image Generation

**DALL-E 2 (Ramesh et al., OpenAI, 2022)**
- CLIP text encoder: maps text to CLIP embedding space
- Prior model: diffusion or autoregressive prior mapping text embedding to image embedding
- unCLIP decoder: diffusion model conditioned on CLIP image embedding
- Two-stage: prior (text -> image embedding) + decoder (image embedding -> image)
- Resolution: 1024x1024 via cascaded upsampling
- See: `references/text-to-image-architectures.md`

**Imagen (Saharia et al., Google Brain, 2022)**
- T5-XXL text encoder (4.6B params): frozen, pretrained language model
- Key insight: scaling text encoder more effective than scaling diffusion model
- Cascaded diffusion: 64x64 -> 256x256 -> 1024x1024
- Dynamic thresholding: clip predicted x_0 to adaptive percentile for high guidance scales
- DrawBench benchmark for evaluation
- See: `references/text-to-image-architectures.md`

**CogView2 (Ding et al., 2022)**
- Hierarchical transformers: cross-modal + local attention
- 6B parameter model
- Token-based approach via VQ-VAE

**Stable Diffusion (Rombach et al., Stability AI / CompVis, 2022)**
- Latent diffusion: train diffusion in VAE latent space (4x lower spatial dims)
- VAE encoder/decoder: compress images to latent representations
- U-Net with cross-attention: condition on CLIP text embeddings
- Open-source: weights and code publicly available
- Efficient: runs on consumer GPUs (4-8 GB VRAM)
- SDXL: improved base + refiner architecture
- See: `references/text-to-image-architectures.md`

**Comparison Table**

| Model | Text Encoder | Diffusion Space | Resolution | Key Innovation |
|-------|-------------|-----------------|------------|---------------|
| DALL-E 2 | CLIP ViT-L/14 | Pixel (64x64 base) | 1024x1024 | CLIP prior + unCLIP |
| Imagen | T5-XXL (4.6B) | Pixel (64x64 base) | 1024x1024 | Large frozen LM encoder |
| Stable Diffusion | CLIP ViT-L/14 | Latent (64x64 = 512px) | 512x512+ | Latent space efficiency |
| CogView2 | Custom tokenizer | Token space | 480x480 | Hierarchical transformers |

### 7. Text-to-Video Generation

**Make-A-Video (Singer et al., Meta AI, 2022)**
- Core insight: leverage T2I models + unsupervised video data (no text-video pairs needed)
- Spatiotemporal decomposition: factorize 3D ops into spatial + temporal components
- Pseudo-3D convolutions: 2D spatial conv followed by 1D temporal conv
- Pseudo-3D attention: 2D spatial attention followed by 1D temporal attention
- Frame interpolation network: increase temporal resolution
- Spatial + temporal super-resolution: 64x64 -> 256x256 -> 768x768 at 16fps
- Pipeline: T2I base -> spatiotemporal layers -> frame interpolation -> SR
- See: `references/text-to-video-architectures.md`

**Imagen Video (Ho et al., Google Brain, 2022)**
- Cascaded video diffusion: 7 sub-models in cascade
- Base: 16x40x24 (frames x H x W) -> final: 128x1280x768
- v-prediction parameterization: predict v = alpha_t * epsilon - sigma_t * x
- Progressive distillation for fast sampling
- Joint image-video training for quality
- See: `references/text-to-video-architectures.md`

**Phenaki (Villegas et al., Google Research, 2023)**
- C-ViViT encoder: compress video to discrete tokens
- Variable-length video generation from sequential text prompts
- Bidirectional masked transformer for token generation
- Time-conditioned generation for scene transitions

**CogVideo (Hong et al., 2022)**
- Multi-frame-rate hierarchical training
- Based on CogView2 architecture, extended to video
- 9B parameter model
- Dual-channel attention for frame coherence

**GODIVA (Wu et al., Microsoft, 2021)**
- 3D sparse attention: factored spatial + temporal attention
- VQ-VAE tokenization of video frames
- Autoregressive generation in token space

**NUWA (Wu et al., Microsoft, 2022)**
- 3D Nearby Attention (3DNA): local attention with 3D receptive fields
- Unified model: text-to-image + text-to-video + video prediction
- Covers 8 visual generation tasks

### 8. Applications

**Computer Vision**
- Super-resolution: SR3, SRDiff (iterative refinement from LR input)
- Inpainting: RePaint (resample known regions each step)
- Semantic segmentation: label diffusion, conditional generation
- Image editing: SDEdit (add noise then denoise with guidance), InstructPix2Pix
- 3D generation: DreamFusion (SDS loss), Magic3D, Point-E, Shap-E
- Video editing: Tune-A-Video, FateZero, Pix2Video

**Natural Language Processing**
- Non-autoregressive text generation: Diffusion-LM, DiffuSeq
- Controllable text: plug-and-play with classifiers
- Machine translation: MDLM-based approaches
- Text infilling: masked diffusion models

**Multi-Modal Generation**
- Text-to-image: DALL-E 2, Imagen, Stable Diffusion
- Text-to-3D: DreamFusion, Magic3D, ProlificDreamer
- Text-to-motion: MDM (Motion Diffusion Model), MotionDiffuse
- Text-to-audio: AudioLDM, DiffSound, Make-An-Audio
- Text-to-video: Make-A-Video, Imagen Video, Phenaki

**Scientific Applications**
- Drug design: DiffSBDD (structure-based), DiffLinker (molecular linkers)
- Materials science: CDVAE (Crystal Diffusion VAE)
- Medical imaging: denoising, reconstruction, anomaly detection
- Protein structure: RFdiffusion, FrameDiff, Chroma

**Temporal Data**
- Time series forecasting: CSDI, TimeGrad, TSDiff
- Time series imputation: conditional diffusion on observed values
- Spatial-temporal: traffic, weather prediction

## Key Concepts

### Forward Process (Diffusion)
The forward process gradually adds Gaussian noise to data over T steps:
```
q(x_t | x_{t-1}) = N(x_t; sqrt(1 - beta_t) * x_{t-1}, beta_t * I)
```
Using the reparameterization with alpha_bar_t = prod(1 - beta_s):
```
q(x_t | x_0) = N(x_t; sqrt(alpha_bar_t) * x_0, (1 - alpha_bar_t) * I)
x_t = sqrt(alpha_bar_t) * x_0 + sqrt(1 - alpha_bar_t) * epsilon
```

### Reverse Process (Denoising)
The learned reverse process recovers data from noise:
```
p_theta(x_{t-1} | x_t) = N(x_{t-1}; mu_theta(x_t, t), sigma_t^2 * I)
mu_theta(x_t, t) = (1/sqrt(alpha_t)) * (x_t - (beta_t / sqrt(1 - alpha_bar_t)) * epsilon_theta(x_t, t))
```

### Guidance
Guidance steers generation toward desired conditions:
- **Classifier guidance**: epsilon_hat = epsilon_theta - s * sigma_t * nabla_x log p_phi(y|x_t)
- **Classifier-free guidance**: epsilon_hat = (1 + w) * epsilon_theta(x_t, c) - w * epsilon_theta(x_t, empty)
- Guidance scale controls fidelity-diversity tradeoff

## Problem-Solving Patterns

1. **Choosing a diffusion framework**: Start with DDPM for simplicity. Move to score-SDE for theoretical flexibility. Use latent diffusion for efficiency on high-resolution tasks.

2. **Selecting a sampler**: DDIM for deterministic, fast sampling (50 steps). DPM-Solver++ for 10-20 step high-quality results. Consistency models for single-step generation at quality cost.

3. **Text-to-image model selection**: Stable Diffusion for open-source/custom fine-tuning. Imagen for maximum text alignment. DALL-E 2 for strong compositionality via CLIP prior.

4. **Text-to-video approach**: Make-A-Video for leveraging existing T2I models without text-video pairs. Imagen Video for highest quality with massive compute. Phenaki for variable-length, multi-scene generation.

5. **Improving sample quality**: Increase guidance scale (but watch for saturation artifacts). Use dynamic thresholding (Imagen). Apply cascaded super-resolution. Train with cosine noise schedule.

6. **Reducing inference cost**: Progressive distillation to reduce steps. Latent space diffusion (Stable Diffusion). Consistency models for single-step. Quantization and model pruning.

7. **Conditioning on new modalities**: Cross-attention for text/label conditioning. Concatenation for spatial conditioning (inpainting, depth). Adapter modules (ControlNet, T2I-Adapter) for structural conditioning.

## Common Pitfalls

1. **Guidance scale too high**: Causes oversaturation, artifacts, and mode collapse. Use dynamic thresholding or reduce scale.

2. **Insufficient training steps**: Diffusion models require long training (often 200K-1M+ iterations). Underfitting manifests as blurry or incoherent samples.

3. **Wrong noise schedule for resolution**: Linear schedules work poorly for high-res images. Use cosine or shifted schedules. Adjust for latent vs pixel space.

4. **Ignoring EMA**: Exponential moving average of weights is critical for sample quality. Typical decay: 0.9999.

5. **Sampling steps too few without proper solver**: Naive DDPM needs 1000 steps. Must use DDIM/DPM-Solver for fewer steps. Each solver has a minimum step threshold for quality.

6. **FID overfitting**: Optimizing only for FID can hurt perceptual quality and diversity. Evaluate with multiple metrics (FID, CLIP score, IS, human eval).

7. **Video temporal coherence**: Naive frame-by-frame generation produces flickering. Must use temporal attention/convolution layers and enforce temporal consistency.

8. **Text encoder selection**: Larger/better text encoders (T5-XXL vs CLIP) significantly impact text alignment. Frozen pretrained encoders often outperform jointly trained ones.

## Source Material

- **Yang, L., Zhang, Z., Song, Y., Hong, S., Xu, R., Zhao, Y., Zhang, W., Cui, B., & Yang, M.-H. (2023).** "Diffusion Models: A Comprehensive Survey of Methods and Applications." *ACM Computing Surveys.* arXiv:2209.00796. Comprehensive taxonomy of diffusion model theory, methods, and applications.

- **Singh, A. (2023).** "A Survey of AI Text-to-Image and AI Text-to-Video Generation." *Survey paper.* Coverage of T2I models (DALL-E 2, Imagen, CogView2, Stable Diffusion) and T2V models (Make-A-Video, Imagen Video, Phenaki, CogVideo, GODIVA, NUWA).

- **Singer, U., Polyak, A., Hayes, T., Yin, X., An, J., Zhang, S., Hu, Q., Yang, H., Ashual, O., Gafni, O., Parikh, D., Gupta, S., & Taigman, Y. (2022).** "Make-A-Video: Text-to-Video Generation without Text-Video Data." *Meta AI Research.* arXiv:2209.14792. Spatiotemporal decomposition, pseudo-3D layers, and frame interpolation for T2V.
