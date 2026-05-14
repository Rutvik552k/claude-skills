# Text-to-Image Generation Architectures

## Overview

This reference details the architectures and design decisions of the major text-to-image
diffusion models: DALL-E 2, Imagen, Stable Diffusion, and CogView2. Each represents
a distinct approach to conditioning diffusion models on text for high-fidelity image
synthesis.

---

## 1. DALL-E 2 (Ramesh et al., OpenAI, 2022)

### 1.1 Architecture Overview

DALL-E 2 (also called unCLIP) uses a two-stage pipeline:
1. **Prior**: text embedding -> CLIP image embedding
2. **Decoder**: CLIP image embedding -> image pixels

### 1.2 CLIP Text/Image Encoders

**CLIP (Contrastive Language-Image Pre-training):**
- ViT-L/14 image encoder: 428M parameters
- Text encoder: 63M parameter transformer
- Trained on 400M image-text pairs with contrastive loss
- Shared embedding space: text and image embeddings are comparable via cosine similarity

**Role in DALL-E 2:**
- Text encoder maps text prompt to CLIP text embedding z_t
- CLIP image encoder provides training targets z_i for the prior
- The prior learns to bridge the gap: z_t -> z_i

### 1.3 Prior Model

The prior maps CLIP text embeddings to CLIP image embeddings.

**Diffusion prior (preferred):**
```
Input: CLIP text embedding z_t, timestep t, noisy CLIP image embedding z_i^t
Output: predicted clean CLIP image embedding z_i^0

Architecture: Transformer decoder
- Sequence: [text_embedding, timestep_embedding, noisy_image_embedding, CLIP_text_tokens]
- Causal attention mask
- Output: predicted z_i from the image embedding position
```

**Autoregressive prior (alternative):**
- Quantize CLIP image embedding into discrete tokens
- Autoregressive transformer generates tokens conditioned on text
- Diffusion prior outperforms autoregressive prior empirically

### 1.4 Decoder (unCLIP)

Modified GLIDE architecture conditioned on CLIP image embedding:
```
Base model: 64x64 diffusion U-Net
- Condition on CLIP image embedding z_i via:
  - Add to timestep embedding (projected)
  - Cross-attention with CLIP image embedding tokens (projected to 4 tokens)
- Also condition on text tokens for additional text alignment

Upsampler 1: 64x64 -> 256x256
- Conditioned on low-res image (concatenation) + CLIP embedding
- Separate U-Net with cross-attention

Upsampler 2: 256x256 -> 1024x1024
- Conditioned on low-res image
- Simpler U-Net, less conditioning needed at high resolution
```

### 1.5 Training Details

```
Prior training:
- Diffusion prior: 1.2B parameters
- Trained on (text, image) pairs from CLIP training data
- L_simple objective on CLIP image embeddings

Decoder training:
- Base: 3.5B parameters, 2.5M training steps
- SR 64->256: 1.5B parameters
- SR 256->1024: fine-tuned from base
- Classifier-free guidance at inference (scale 2.0 for prior, 1.25-4.0 for decoder)
```

### 1.6 Key Innovations

- **Inversion via CLIP space**: Manipulate images through CLIP embedding arithmetic
- **Variations**: Encode image -> CLIP embedding -> decode with different noise -> variation
- **Interpolation**: Spherical interpolation in CLIP embedding space
- **Text diffs**: z_new = z_i + (z_text_target - z_text_source) for editing

### 1.7 Limitations

- CLIP embedding loses fine spatial detail (binding problem)
- Struggles with attribute-object composition ("a red cube and a blue sphere")
- Two-stage pipeline introduces compounding errors
- CLIP text encoder limited to 77 tokens

---

## 2. Imagen (Saharia et al., Google Brain, 2022)

### 2.1 Architecture Overview

Imagen uses a frozen T5-XXL text encoder with cascaded pixel-space diffusion:
1. **Text encoder**: T5-XXL (frozen, pretrained)
2. **Base model**: 64x64 text-conditional diffusion
3. **Super-resolution 1**: 64x64 -> 256x256
4. **Super-resolution 2**: 256x256 -> 1024x1024

### 2.2 Text Encoder: T5-XXL

**Key design choice**: Use a large pretrained language model instead of CLIP.
```
T5-XXL: 4.6B parameters (encoder only used)
- Pre-trained on C4 corpus (text-only)
- Frozen during diffusion training (no gradient updates)
- Output: sequence of token embeddings (not a single vector)
- Rich semantic understanding from language pretraining
```

**Critical finding**: Scaling the text encoder is more effective for text-image
alignment than scaling the diffusion model:
- T5-Small (60M) -> T5-Base (220M) -> T5-Large (770M) -> T5-XL (3B) -> T5-XXL (4.6B)
- Each size increase substantially improved FID and CLIP score
- Diminishing returns from scaling diffusion model size

### 2.3 Base Model (64x64)

```
Architecture: U-Net with Efficient Attention
- 2B parameters
- Text conditioning via cross-attention:
  Q = W_Q * h       (U-Net features, [B, H*W, C])
  K = W_K * c_text   (T5 embeddings, [B, L, D_text])
  V = W_V * c_text
  Attention = softmax(Q K^T / sqrt(d)) V

- Layer norm on text embeddings before cross-attention
- Timestep conditioning via FiLM (Feature-wise Linear Modulation):
  scale, shift = MLP(t_emb)
  h = scale * GroupNorm(h) + shift
```

### 2.4 Cascaded Super-Resolution

**SR 64->256 (Efficient U-Net):**
```
Input: 64x64 (bilinear upsampled to 256x256) concatenated with noise x_t
Conditioning: T5 text embeddings via cross-attention + low-res image via concatenation
Noise augmentation: Add noise to low-res conditioning (aug_level ~ Uniform(0, sigma_max))
- Critical for robustness: prevents SR model from learning to copy artifacts
```

**SR 256->1024 (Efficient U-Net):**
```
Input: 256x256 (upsampled) concatenated with noise x_t
Text conditioning: yes (cross-attention to T5 embeddings)
Noise augmentation: same as above
```

### 2.5 Dynamic Thresholding

Standard static thresholding clips predicted x_0 to [-1, 1]:
```
x_0_pred = clip((x_t - sqrt(1-alpha_bar) * epsilon_theta) / sqrt(alpha_bar), -1, 1)
```

Dynamic thresholding adapts the clipping threshold:
```
s = percentile(|x_0_pred|, p)    # p=99.5 typically
if s > 1:
    x_0_pred = clip(x_0_pred, -s, s) / s   # scale to [-1, 1]
```

**Why it matters**: At high classifier-free guidance scales (w=10-15+),
predicted x_0 values drift far from [-1, 1]. Static clipping causes saturation
artifacts. Dynamic thresholding rescales gracefully, enabling higher guidance
scales for better text alignment without artifact.

### 2.6 Efficient U-Net Improvements

Key architectural improvements over standard U-Net:
```
1. Memory efficiency: shift model parameters from high-res to low-res blocks
2. Skip connections: scale by 1/sqrt(2) for training stability
3. Reverse channel order: more parameters at lower resolutions
4. Attention: only at 32x32 and 16x16 resolutions (not higher)
5. Progressive training: train at lower resolution first
```

### 2.7 DrawBench

Imagen introduced DrawBench: 200 text prompts for systematic T2I evaluation.
Categories: colors, counting, spatial, text rendering, compositionality,
complex scenes, rare words, misspelled words.

### 2.8 Key Results

```
FID (COCO 256x256): 7.27 (zero-shot)
CLIP score: 0.29
DrawBench human eval: preferred over DALL-E 2 in 43.6% vs 39.2% (fidelity)
                      preferred over DALL-E 2 in 75.3% vs 24.7% (text alignment)
```

---

## 3. Stable Diffusion / Latent Diffusion (Rombach et al., 2022)

### 3.1 Architecture Overview

Latent Diffusion Models (LDMs) perform diffusion in a compressed latent space:
1. **VAE Encoder**: image x -> latent z (spatial compression)
2. **Diffusion Model**: denoise z_t in latent space
3. **VAE Decoder**: latent z -> reconstructed image x

### 3.2 VAE (Variational Autoencoder)

**Encoder:**
```
Input: RGB image x, shape [B, 3, H, W]
Architecture: ResNet blocks + downsampling
Output: latent z, shape [B, C, H/f, W/f]
f = 8 for SD v1/v2 (512x512 -> 64x64 latent)
f = 8 for SDXL (1024x1024 -> 128x128 latent)
C = 4 latent channels

Training: reconstruction loss + KL regularization + perceptual loss (LPIPS)
```

**Decoder:**
```
Input: latent z, shape [B, 4, H/8, W/8]
Architecture: ResNet blocks + upsampling
Output: reconstructed image x, shape [B, 3, H, W]
```

**Key benefit**: Diffusion in 4x64x64 latent space instead of 3x512x512 pixel space:
- 48x fewer dimensions
- Much faster training and inference
- Perceptual compression preserves visual quality

### 3.3 U-Net Architecture

```
Input: noisy latent z_t, shape [B, 4, 64, 64]
Timestep: sinusoidal embedding -> MLP -> injected via GroupNorm scale/shift

Encoder blocks (downsampling):
  64x64: 2 ResNet blocks, (optional attention)
  32x32: 2 ResNet blocks + self-attention + cross-attention
  16x16: 2 ResNet blocks + self-attention + cross-attention
  8x8:   2 ResNet blocks + self-attention + cross-attention

Middle block:
  8x8: ResNet + self-attention + ResNet

Decoder blocks (upsampling with skip connections):
  8x8 -> 16x16 -> 32x32 -> 64x64
  Mirror of encoder with skip connections from encoder

Cross-attention for text conditioning:
  Q = W_Q * h_unet           # from U-Net spatial features
  K = W_K * text_embeddings   # from CLIP text encoder
  V = W_V * text_embeddings
  Attention(Q, K, V) = softmax(QK^T / sqrt(d)) * V
```

**SD v1.5 U-Net**: ~860M parameters
**SDXL U-Net**: ~2.6B parameters (larger attention dimensions, more transformer blocks)

### 3.4 Text Encoder

**SD v1.x**: CLIP ViT-L/14 text encoder (123M params)
- Max 77 tokens, output shape [B, 77, 768]
- Penultimate layer features (not final layer)

**SD v2.x**: OpenCLIP ViT-H/14 text encoder (354M params)
- Output shape [B, 77, 1024]

**SDXL**: Dual text encoders
- CLIP ViT-L/14 (768-dim) + OpenCLIP ViT-bigG (1280-dim)
- Concatenated: [B, 77, 2048]
- Pooled text embedding (from ViT-bigG) added to timestep embedding

### 3.5 Training

```
Dataset: LAION-5B (5 billion image-text pairs), filtered
Resolution: 256x256 pretraining, 512x512 fine-tuning (SD v1)
            256x256 -> 512x512 -> 1024x1024 progressive (SDXL)
Loss: L_simple in latent space
      E [||epsilon - epsilon_theta(z_t, t, c_text)||^2]
Batch size: 2048
Training compute: ~150K A100 GPU hours (SD v1.5)
Optimizer: AdamW, lr=1e-4
EMA decay: 0.9999
CFG dropout: 10% unconditional
```

### 3.6 SDXL Improvements

```
1. Larger U-Net (2.6B vs 860M parameters)
2. Dual text encoders (richer text features)
3. Size and crop conditioning:
   - Encode original image size (h_orig, w_orig) as Fourier features
   - Encode crop coordinates (top, left) as Fourier features
   - Concatenate with timestep embedding
   - Eliminates artifacts from multi-resolution training
4. Refiner model: separate diffusion model for final denoising steps
   - Base generates at high noise levels, refiner handles low noise
   - Improves fine detail quality
5. Micro-conditioning: aspect ratio, aesthetic score conditioning
```

### 3.7 Inference Pipeline

```
1. Encode text prompt with CLIP text encoder -> c_text
2. Sample z_T ~ N(0, I) in latent space
3. For t = T, T-1, ..., 1 (using DDIM/DPM-Solver):
   a. Compute conditional:  eps_c = epsilon_theta(z_t, t, c_text)
   b. Compute unconditional: eps_u = epsilon_theta(z_t, t, empty)
   c. Classifier-free guidance: eps = eps_u + w * (eps_c - eps_u)
   d. DDIM/solver step: z_{t-1} = solver_step(z_t, eps, t)
4. Decode latent: x = Decoder(z_0)
```

### 3.8 Fine-Tuning Methods

**LoRA (Low-Rank Adaptation):**
```
W_new = W_original + alpha * B * A
A: [r, d_in],  B: [d_out, r],  r << min(d_in, d_out)
Typical rank: 4-128
Applied to: cross-attention Q, K, V, O projections
```

**Textual Inversion:**
```
Learn new token embedding v* for a concept
Optimize: argmin_v* E [||eps - eps_theta(z_t, t, c_text(v*))||^2]
Frozen model, only optimize the embedding vector
```

**DreamBooth:**
```
Fine-tune entire model on 3-5 images of a subject
Prior preservation: mix subject images with class-prior images
Loss: L_subject + lambda * L_prior
```

**ControlNet (Zhang et al., 2023):**
```
Copy U-Net encoder blocks as trainable branch
Input: conditioning signal (edges, depth, pose, etc.)
Output: added to U-Net decoder via zero-initialized convolutions
Locked original weights + trainable copy
```

### 3.9 Key Advantages

- Open-source weights and code (community-driven ecosystem)
- Runs on consumer GPUs (4-8 GB VRAM with optimizations)
- Rich ecosystem of fine-tuning methods and extensions
- Fast inference (~2-5 seconds for 512x512 on RTX 3090)

---

## 4. CogView2 (Ding et al., 2022)

### 4.1 Architecture Overview

Token-based approach using hierarchical transformers:
```
Text tokens -> Cross-modal transformer -> Image tokens -> Local attention refiner
```

### 4.2 Key Components

**CogLM backbone**: 6B parameter pretrained language model
**Tokenizer**: VQ-VAE with 8192 codebook (32x32 token grid for 480x480)

**Hierarchical generation:**
```
Stage 1: Cross-modal attention transformer
- Input: text tokens + partial image tokens
- Generate 32x32 image token grid
- Global cross-modal attention for text-image alignment

Stage 2: Local parallel autoregressive (CogLM-refine)
- Refine image tokens using local attention windows
- 4x4 local attention for parallel refinement
- Improves high-frequency detail
```

### 4.3 Training Strategy

```
Pre-training: 30M text-image pairs
Fine-tuning: higher quality subsets with CLIP filtering
Local attention: 4x4 windows with overlapping boundaries
Inference: hierarchical token generation + refinement
```

### 4.4 Comparison to Diffusion Approaches

- Faster inference (autoregressive, no iterative denoising)
- Lower image quality compared to Imagen/DALL-E 2
- Higher memory requirements (6B transformer)
- Less flexible conditioning compared to diffusion

---

## 5. Architecture Comparison Table

### 5.1 High-Level Comparison

| Feature | DALL-E 2 | Imagen | Stable Diffusion v1.5 | SDXL | CogView2 |
|---------|----------|--------|----------------------|------|----------|
| Text encoder | CLIP ViT-L/14 | T5-XXL (4.6B) | CLIP ViT-L/14 | CLIP-L + OpenCLIP-bigG | CogLM (6B) |
| Text encoder params | 63M | 4,600M | 123M | ~1,400M (combined) | 6,000M |
| Diffusion space | Pixel | Pixel | Latent (4x64x64) | Latent (4x128x128) | Token |
| Base resolution | 64x64 | 64x64 | 64x64 latent (512px) | 128x128 latent (1024px) | 32x32 tokens (480px) |
| Final resolution | 1024x1024 | 1024x1024 | 512x512 | 1024x1024 | 480x480 |
| Upsampling stages | 2 (SR) | 2 (SR) | 1 (VAE decode) | 1 (VAE decode) | 1 (refine) |
| Diffusion params | ~3.5B (base) | ~2B (base) | ~860M | ~2.6B | N/A (AR) |
| Total params | ~5.5B | ~7B+ | ~1B | ~6.6B | ~6B |
| Open source | No | No | Yes | Yes | Partial |
| Consumer GPU | No | No | Yes (4GB+) | Yes (8GB+) | No |

### 5.2 Text Conditioning Comparison

| Method | Text Representation | Conditioning Mechanism | Strength |
|--------|-------------------|----------------------|----------|
| DALL-E 2 | CLIP embedding (single vector + tokens) | Prior + cross-attention | Compositionality via CLIP |
| Imagen | T5 token embeddings (sequence) | Cross-attention + FiLM | Deep language understanding |
| Stable Diffusion | CLIP token embeddings (sequence) | Cross-attention | Efficient, flexible |
| SDXL | Dual CLIP embeddings (concatenated) | Cross-attention + pooled emb | Richer text features |
| CogView2 | CogLM token embeddings | Cross-modal attention | Large LM backbone |

### 5.3 Quality vs Efficiency Tradeoff

```
Quality ranking (approximate, based on human evaluation):
1. Imagen (highest text fidelity)
2. DALL-E 2 / SDXL (close)
3. Stable Diffusion v1.5
4. CogView2

Efficiency ranking (inference speed):
1. Stable Diffusion v1.5 (latent space, smallest model)
2. SDXL (latent space, larger but still efficient)
3. CogView2 (autoregressive, single pass)
4. DALL-E 2 (two-stage pipeline)
5. Imagen (large models, pixel space)

Accessibility ranking:
1. Stable Diffusion v1.5 (open source, consumer GPUs)
2. SDXL (open source, consumer GPUs)
3. CogView2 (partially open)
4. DALL-E 2 (API only)
5. Imagen (not publicly released)
```

---

## 6. Evaluation Metrics

### 6.1 FID (Frechet Inception Distance)

```
FID = ||mu_real - mu_gen||^2 + Tr(Sigma_real + Sigma_gen - 2*(Sigma_real * Sigma_gen)^{1/2})
```
Lower is better. Measures distributional similarity. Standard on COCO 30K.

### 6.2 CLIP Score

```
CLIP_score = E [cos(CLIP_image(x), CLIP_text(c))]
```
Higher is better. Measures text-image alignment.

### 6.3 Inception Score (IS)

```
IS = exp(E [KL(p(y|x) || p(y))])
```
Higher is better. Measures quality and diversity.

### 6.4 Human Evaluation

- Pairwise preference: which image is better for a given prompt?
- Separate ratings for fidelity (image quality) and alignment (text match)
- DrawBench: standardized prompt set for systematic comparison
- PartiPrompts: 1600+ prompts for compositionality evaluation

---

## Source Citations

- Ramesh, A., Dhariwal, P., Nichol, A., Chu, C., & Chen, M. (2022). "Hierarchical Text-Conditional Image Generation with CLIP Latents." arXiv:2204.06125.
- Saharia, C., Chan, W., Saxena, S., et al. (2022). "Photorealistic Text-to-Image Diffusion Models with Deep Language Understanding." NeurIPS 2022.
- Rombach, R., Blattmann, A., Lorenz, D., Esser, P., & Ommer, B. (2022). "High-Resolution Image Synthesis with Latent Diffusion Models." CVPR 2022.
- Podell, D., English, Z., Lacey, K., et al. (2023). "SDXL: Improving Latent Diffusion Models for High-Resolution Image Synthesis." arXiv:2307.01952.
- Ding, M., Zheng, W., Hong, W., & Tang, J. (2022). "CogView2: Faster and Better Text-to-Image Generation via Hierarchical Transformers." NeurIPS 2022.
- Singh, A. (2023). "A Survey of AI Text-to-Image and AI Text-to-Video Generation."
- Yang, L. et al. (2023). "Diffusion Models: A Comprehensive Survey." ACM Computing Surveys.
