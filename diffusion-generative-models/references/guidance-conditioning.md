# Guidance and Conditioning in Diffusion Models

## Overview

Guidance and conditioning are the mechanisms by which diffusion models are steered
toward generating samples that match desired attributes (text prompts, class labels,
images, etc.). This reference covers classifier guidance, classifier-free guidance,
text encoder conditioning strategies, and practical guidance scale effects.

---

## 1. Unconditional Diffusion Models

### 1.1 Baseline

An unconditional diffusion model learns:
```
p_theta(x_{t-1} | x_t) = N(x_{t-1}; mu_theta(x_t, t), sigma_t^2 I)
```

The model learns the unconditional data distribution p(x). There is no mechanism
to control what is generated -- sampling produces random samples from the training
distribution.

### 1.2 Conditional Objective

A conditional diffusion model learns:
```
p_theta(x_{t-1} | x_t, c) = N(x_{t-1}; mu_theta(x_t, t, c), sigma_t^2 I)
```

where c is a conditioning signal (class label, text embedding, image, etc.).

Training loss:
```
L = E_{t, x_0, epsilon, c} [||epsilon - epsilon_theta(x_t, t, c)||^2]
```

This learns p(x|c), but may not strongly enforce the conditioning -- the model
can partially ignore c and still achieve low loss. Guidance addresses this.

---

## 2. Classifier Guidance (Dhariwal & Nichol, 2021)

### 2.1 Mathematical Derivation

Starting from Bayes' rule:
```
p(x_t | y) = p(x_t) * p(y | x_t) / p(y)

Taking gradients:
nabla_{x_t} log p(x_t | y) = nabla_{x_t} log p(x_t) + nabla_{x_t} log p(y | x_t)
```

The conditional score decomposes into:
- Unconditional score: nabla_{x_t} log p(x_t) (from the diffusion model)
- Classifier gradient: nabla_{x_t} log p(y | x_t) (from a noise-aware classifier)

### 2.2 Guided Sampling

Replace the score in the reverse process:
```
nabla_{x_t} log p_s(x_t | y) = nabla_{x_t} log p(x_t) + s * nabla_{x_t} log p(y | x_t)
```

where s is the guidance scale (s > 1 amplifies the classifier signal).

In terms of noise prediction:
```
epsilon_hat = epsilon_theta(x_t, t) - s * sigma_t * nabla_{x_t} log p_phi(y | x_t)
```

### 2.3 Noise-Aware Classifier

The classifier must operate on noisy inputs x_t, not clean images:
```
Classifier p_phi(y | x_t):
  - Trained on noisy images at all noise levels
  - Architecture: typically a U-Net encoder or ResNet
  - Input: (x_t, t) -- noisy image and timestep
  - Output: class probabilities p(y | x_t, t)

Training:
  - Sample (x_0, y) from labeled dataset
  - Sample t ~ Uniform(1, T), epsilon ~ N(0, I)
  - Compute x_t = sqrt(alpha_bar_t) * x_0 + sqrt(1 - alpha_bar_t) * epsilon
  - Minimize cross-entropy: -log p_phi(y | x_t, t)
```

### 2.4 Sampling Algorithm with Classifier Guidance

```
x_T ~ N(0, I)
for t = T, T-1, ..., 1:
    # Diffusion model prediction
    eps_pred = epsilon_theta(x_t, t)

    # Classifier gradient
    grad = nabla_{x_t} log p_phi(y | x_t, t)

    # Guided noise prediction
    eps_guided = eps_pred - s * sigma_t * grad

    # Standard DDPM/DDIM step using eps_guided
    z ~ N(0, I) if t > 1, else z = 0
    x_{t-1} = (1/sqrt(alpha_t)) * (x_t - (beta_t/sqrt(1-alpha_bar_t)) * eps_guided) + sigma_t * z
return x_0
```

### 2.5 Limitations of Classifier Guidance

1. **Requires separate classifier**: must train a noise-aware classifier on the same data
2. **Classifier quality matters**: poor classifier = poor guidance
3. **Adversarial gradients**: classifier gradients may exploit diffusion model artifacts
4. **Limited to classification**: hard to extend to free-form text conditioning
5. **Computational overhead**: requires gradient computation through classifier at each step

---

## 3. Classifier-Free Guidance (Ho & Salimans, 2022)

### 3.1 Core Idea

Eliminate the need for a separate classifier by training a single model that can
produce both conditional and unconditional predictions.

### 3.2 Training

During training, randomly drop the conditioning with probability p_uncond:
```
for each training example (x_0, c):
    if random() < p_uncond:        # typically p_uncond = 0.1 to 0.2
        c = empty_condition         # null/empty token, zero vector, etc.
    
    t ~ Uniform(1, T)
    epsilon ~ N(0, I)
    x_t = sqrt(alpha_bar_t) * x_0 + sqrt(1 - alpha_bar_t) * epsilon
    loss = ||epsilon - epsilon_theta(x_t, t, c)||^2
```

This single model learns both:
- epsilon_theta(x_t, t, c): conditional prediction (when c is present)
- epsilon_theta(x_t, t, empty): unconditional prediction (when c is dropped)

### 3.3 Guided Prediction

At inference, interpolate between conditional and unconditional predictions:
```
epsilon_guided = epsilon_theta(x_t, t, empty) + w * (epsilon_theta(x_t, t, c) - epsilon_theta(x_t, t, empty))
```

Equivalently:
```
epsilon_guided = (1 - w) * epsilon_theta(x_t, t, empty) + w * epsilon_theta(x_t, t, c)
```

Or in the common formulation (extrapolation with w > 1):
```
epsilon_guided = epsilon_uncond + w * (epsilon_cond - epsilon_uncond)
```

where w is the guidance scale.

### 3.4 Connection to Classifier Guidance

Classifier-free guidance implicitly performs classifier guidance:
```
epsilon_guided = epsilon_uncond + w * (epsilon_cond - epsilon_uncond)

In score form:
s_guided = s_uncond + w * (s_cond - s_uncond)
         = s_uncond + w * (nabla log p(x_t|c) - nabla log p(x_t))
         = (1 - w) * nabla log p(x_t) + w * nabla log p(x_t|c)

Using Bayes' rule (nabla log p(x|c) = nabla log p(x) + nabla log p(c|x)):
s_guided = nabla log p(x_t) + w * nabla log p(c|x_t)
```

This is equivalent to classifier guidance with scale w, where the implicit
classifier is:
```
log p(c|x_t) proportional to (epsilon_uncond - epsilon_cond) / sigma_t
```

### 3.5 Guidance Scale Effects

```
w = 0:   Pure unconditional generation (ignore text/conditioning entirely)
w = 1:   Standard conditional generation (no guidance amplification)
w = 1-3: Mild guidance (slightly sharper, mild quality improvement)
w = 3-7: Moderate guidance (good balance of quality and diversity)
w = 7-15: Strong guidance (high fidelity to condition, reduced diversity)
w > 15:  Very strong guidance (risk of oversaturation, artifacts, mode collapse)
```

**Fidelity-diversity tradeoff:**
- Higher w -> higher fidelity to conditioning (better CLIP score, text alignment)
- Higher w -> lower diversity (samples cluster around modes)
- Higher w -> initially better FID, then worse FID (U-shaped curve)
- Optimal w depends on task, model, and evaluation metric

### 3.6 Practical Guidelines

```
Text-to-image:
  Stable Diffusion v1.5: w = 7-8
  SDXL: w = 5-7
  Imagen: w = 7-15 (with dynamic thresholding)
  DALL-E 2 prior: w = 2.0
  DALL-E 2 decoder: w = 1.25-4.0

Text-to-video:
  Base model: w = 7-15
  Super-resolution: w = 3-7

Class-conditional:
  ImageNet: w = 1.5-4.0

p_uncond (training dropout rate):
  Common: 0.1 (10% unconditional)
  Range: 0.05-0.2
  Higher p_uncond -> model learns stronger unconditional distribution
  Lower p_uncond -> model focuses more on conditional distribution
```

### 3.7 Computational Cost

Each guided sampling step requires TWO forward passes:
```
Step 1: eps_uncond = epsilon_theta(x_t, t, empty)    # unconditional
Step 2: eps_cond   = epsilon_theta(x_t, t, c)        # conditional
Step 3: eps_guided = eps_uncond + w * (eps_cond - eps_uncond)
```

This doubles the inference cost compared to unguided sampling.

**Optimization: batched guidance**
```
Batch both predictions together:
  inputs = [x_t, x_t]           # duplicated
  conds  = [empty, c]           # unconditional + conditional
  [eps_uncond, eps_cond] = epsilon_theta(inputs, t, conds)  # single batched forward pass
```

---

## 4. Text Encoder Conditioning

### 4.1 CLIP Text Encoder

**Architecture:**
```
CLIP ViT-L/14 text encoder:
  - 12-layer transformer (63M params for base, 123M for ViT-L/14)
  - BPE tokenizer, max 77 tokens
  - Output: [CLS] token embedding (pooled) + per-token embeddings
  - Trained with contrastive image-text loss on 400M pairs
```

**Conditioning approaches:**

**Pooled embedding (DALL-E 2 prior):**
```
c = CLIP_text(prompt)[CLS]    # single vector, shape [D]
Inject via: addition to timestep embedding, FiLM, adaptive normalization
```

**Token embeddings (Stable Diffusion):**
```
c = CLIP_text(prompt)          # full sequence, shape [77, 768]
Inject via: cross-attention in U-Net
  Q = W_Q * h_spatial          # from U-Net
  K = W_K * c                  # from text
  V = W_V * c
  Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) V
```

**Strengths**: good visual-semantic alignment, trained on images
**Weaknesses**: 77 token limit, may miss nuanced language understanding,
inherits CLIP's biases and failure modes

### 4.2 T5 Text Encoder

**Architecture:**
```
T5-XXL (Raffel et al., 2020):
  - 24-layer encoder (4.6B params for XXL)
  - SentencePiece tokenizer, max 512 tokens (or more)
  - Output: per-token embeddings, shape [L, 4096]
  - Pretrained on C4 corpus (text-only, no images)
  - Frozen during diffusion training
```

**Conditioning in Imagen:**
```
c = T5_XXL_encoder(prompt)     # shape [L, 4096]

Cross-attention:
  Q = W_Q * h_spatial          # from U-Net features
  K = W_K * c                  # from T5
  V = W_V * c
  
Layer norm applied to T5 embeddings before projection:
  c_normed = LayerNorm(c)
  K = W_K * c_normed
  V = W_V * c_normed

FiLM conditioning (additional):
  Pooled text embedding (mean-pool T5 output)
  scale, shift = MLP(pool(c))
  h = scale * GroupNorm(h) + shift
```

**Strengths**: deep language understanding, handles complex prompts, longer context
**Weaknesses**: not trained on images (no visual grounding), larger model,
no contrastive image-text training

### 4.3 Dual Text Encoders (SDXL)

```
Encoder 1: CLIP ViT-L/14 -> shape [77, 768]
Encoder 2: OpenCLIP ViT-bigG/14 -> shape [77, 1280]

Token-level: concatenate along feature dimension
  c_tokens = concat([c_clip, c_openclip], dim=-1)   # shape [77, 2048]
  Used for cross-attention in U-Net

Pooled: from OpenCLIP ViT-bigG
  c_pooled = OpenCLIP_text(prompt)[CLS]              # shape [1280]
  Added to timestep embedding
```

**Advantage**: combines CLIP's visual grounding with OpenCLIP's larger model capacity.

### 4.4 Text Encoder Comparison

| Encoder | Params | Max Tokens | Training Data | Visual Grounding | Language Depth |
|---------|--------|------------|--------------|-----------------|---------------|
| CLIP ViT-L/14 | 123M | 77 | 400M image-text | Yes (contrastive) | Moderate |
| CLIP ViT-H/14 | 354M | 77 | 2B image-text | Yes (contrastive) | Good |
| OpenCLIP ViT-bigG | ~695M | 77 | LAION-2B | Yes (contrastive) | Good |
| T5-Small | 60M | 512 | C4 (text-only) | No | Low |
| T5-XXL | 4,600M | 512 | C4 (text-only) | No | Very High |
| T5-XXL + CLIP | ~4,723M | 512 | Mixed | Partial | Very High |

**Key finding from Imagen**: Scaling the text encoder provides more benefit
than scaling the diffusion model. T5-XXL > T5-XL > T5-Large by large margins.

---

## 5. Conditioning Mechanisms

### 5.1 Cross-Attention

Most common method for text conditioning:
```
Location: inserted after self-attention in U-Net transformer blocks
Applied at: typically 32x32, 16x16, 8x8 resolutions (not at full resolution)

For each cross-attention layer:
  h: U-Net spatial features, shape [B, HW, C_model]
  c: text embeddings, shape [B, L, C_text]
  
  Q = W_Q(h)    # [B, HW, d_head * n_heads]
  K = W_K(c)    # [B, L, d_head * n_heads]
  V = W_V(c)    # [B, L, d_head * n_heads]
  
  attn = softmax(Q @ K^T / sqrt(d_head))   # [B, n_heads, HW, L]
  out = attn @ V                            # [B, n_heads, HW, d_head]
  out = W_O(out)                            # [B, HW, C_model]
  
  h = h + out   # residual connection
```

The attention map attn[b, head, spatial_pos, text_token] reveals which spatial
locations attend to which text tokens -- useful for understanding and debugging.

### 5.2 Adaptive Layer Normalization (adaLN)

Used in DiT (Diffusion Transformer) and some U-Nets:
```
Standard LayerNorm: y = (x - mu) / sigma * gamma + beta

adaLN: conditioning-dependent scale and shift
  gamma, beta = MLP(c_embed)    # c_embed includes timestep + class/text
  y = (x - mu) / sigma * gamma + beta

adaLN-Zero (DiT): initialize gamma=1, beta=0, plus a scaling factor alpha:
  gamma, beta, alpha = MLP(c_embed)
  y = alpha * ((x - mu) / sigma * gamma + beta)
  Initialize alpha = 0 (output starts at zero)
```

### 5.3 FiLM (Feature-wise Linear Modulation)

```
scale, shift = MLP(conditioning)
h = scale * Norm(h) + shift

Applied per-channel to feature maps
Lightweight and effective for global conditioning signals
```

### 5.4 Concatenation

For spatial conditioning (inpainting, depth, edges):
```
Input to U-Net: concat([x_t, c_spatial], dim=channel)
  x_t: noisy image/latent, shape [B, C, H, W]
  c_spatial: condition, shape [B, C_cond, H, W]
  Combined: shape [B, C + C_cond, H, W]

First conv layer adjusted to accept C + C_cond input channels
```

Used for: inpainting masks, depth maps, edge maps, low-res conditioning in SR.

### 5.5 ControlNet (Zhang et al., 2023)

Structural conditioning without retraining the base model:
```
Architecture:
  - Copy U-Net encoder blocks as trainable "ControlNet" branch
  - Lock original U-Net weights
  - ControlNet takes conditioning input (edges, depth, pose, etc.)
  - Zero-initialized convolutions connect ControlNet outputs to U-Net decoder

Forward pass:
  1. U-Net encoder: h_enc = UNet_encoder(x_t, t, c_text)
  2. ControlNet: h_ctrl = ControlNet(x_t + c_spatial, t, c_text)
  3. Zero conv: h_ctrl_scaled = zero_conv(h_ctrl)    # initialized to zero
  4. U-Net decoder: output = UNet_decoder(h_enc + h_ctrl_scaled, t, c_text)

Training: only ControlNet + zero conv weights are trained
Advantage: preserves base model quality, adds controllability
```

### 5.6 T2I-Adapter

Lighter alternative to ControlNet:
```
Architecture:
  - Small adapter networks (77M params vs ~400M for ControlNet)
  - Encode conditioning into multi-scale features
  - Add to U-Net encoder features via addition (not concatenation)
  
F_adapter = Adapter(c_spatial)    # multi-scale feature maps
h_enc_new = h_enc + F_adapter     # add at matching resolutions
```

### 5.7 IP-Adapter (Image Prompt Adapter)

Condition on reference images instead of (or in addition to) text:
```
Image features: CLIP image encoder -> image embedding tokens
Decoupled cross-attention:
  - Text cross-attention: Attn(Q, K_text, V_text) [original]
  - Image cross-attention: Attn(Q, K_image, V_image) [added]
  - Combined: h = h + attn_text + lambda * attn_image

lambda controls the image influence strength
```

---

## 6. Advanced Guidance Techniques

### 6.1 Dynamic Thresholding (Imagen)

Prevents saturation at high guidance scales:
```
def dynamic_threshold(x_0_pred, percentile=0.995):
    s = torch.quantile(torch.abs(x_0_pred), percentile, dim=pixel_dims)
    s = torch.maximum(s, torch.ones_like(s))  # at least 1
    x_0_pred = torch.clamp(x_0_pred, -s, s) / s
    return x_0_pred
```

Without dynamic thresholding, guidance scale > 10 often produces artifacts.
With it, scales of 15-30+ become usable.

### 6.2 Self-Attention Guidance (SAG)

Use the model's own self-attention maps to guide generation:
```
1. Compute attention maps from self-attention layers
2. Identify important regions (high attention)
3. Blur/degrade non-important regions
4. Compute guided prediction using degraded vs original
5. Guidance term = difference between original and degraded predictions
```

Does not require a classifier or conditioning modification.

### 6.3 Perturbed Attention Guidance (PAG)

Replace self-attention maps with identity maps as degradation:
```
eps_normal = epsilon_theta(x_t, t, c)              # normal forward pass
eps_perturbed = epsilon_theta_PAG(x_t, t, c)        # self-attn replaced with identity
eps_guided = eps_normal + s * (eps_normal - eps_perturbed)
```

Improves sample quality without any conditioning. Works as orthogonal enhancement
to classifier-free guidance.

### 6.4 Negative Prompting

In classifier-free guidance, replace empty condition with a negative prompt:
```
Standard: eps = eps_empty + w * (eps_cond - eps_empty)
Negative: eps = eps_neg + w * (eps_cond - eps_neg)

where eps_neg = epsilon_theta(x_t, t, c_negative)
```

The model is guided AWAY from the negative prompt and TOWARD the positive prompt.
Common negative prompts: "blurry, low quality, deformed, ugly, bad anatomy"

### 6.5 Composable Guidance

Combine multiple conditions:
```
eps_guided = eps_uncond + w1 * (eps_cond1 - eps_uncond) + w2 * (eps_cond2 - eps_uncond)
```

Or in score form:
```
score_guided = score_uncond + w1 * nabla log p(c1|x_t) + w2 * nabla log p(c2|x_t)
```

Enables combining text + image + class + spatial conditions simultaneously.

### 6.6 Guidance Rescaling

Rescale the guided prediction to match the expected magnitude:
```
eps_guided = eps_uncond + w * (eps_cond - eps_uncond)
eps_rescaled = eps_guided * (||eps_cond|| / ||eps_guided||)^phi
```

where phi in [0, 1] controls rescaling strength. Prevents the norm explosion
that occurs at high guidance scales.

---

## 7. Guidance in Specialized Settings

### 7.1 Video Generation

```
Guidance is applied per-frame but with temporal consistency:
  - Same guidance scale for all frames in a clip
  - Base model: high guidance (w=7-15) for text alignment
  - SR models: lower guidance (w=3-5) for quality

Cross-frame guidance consistency:
  - Shared noise for unconditional prediction across frames
  - Temporal smoothing of guidance vectors
```

### 7.2 3D Generation (SDS Loss)

Score Distillation Sampling for text-to-3D:
```
Optimize 3D representation theta (NeRF, mesh, etc.):
  1. Render image x = render(theta, camera_pose)
  2. Add noise: x_t = sqrt(alpha_bar_t) * x + sqrt(1 - alpha_bar_t) * epsilon
  3. Predict noise: eps_pred = epsilon_model(x_t, t, text)
  4. SDS gradient: nabla_theta L_SDS = E_t,eps [w(t) * (eps_pred - eps) * d_x/d_theta]
  5. Update: theta <- theta - lr * nabla_theta L_SDS
```

SDS uses guidance scale w = 100 (very high) for strong text adherence in 3D.

### 7.3 Image Editing

```
SDEdit (Meng et al., 2022):
  1. Start with source image x_source
  2. Add noise: x_t = sqrt(alpha_bar_t) * x_source + sqrt(1 - alpha_bar_t) * epsilon
  3. Denoise with new text prompt: x_0 = denoise(x_t, text_new, guidance=w)
  
  t controls edit strength: low t = subtle edit, high t = major change
  w controls text adherence: higher = more faithful to new prompt
```

---

## 8. Guidance Scale Selection Guide

### 8.1 By Task

| Task | Recommended w | Notes |
|------|--------------|-------|
| Text-to-image (SD v1.5) | 7-8 | Well-calibrated default |
| Text-to-image (SDXL) | 5-7 | Model is already well-conditioned |
| Text-to-image (Imagen) | 7-15 | With dynamic thresholding |
| Text-to-video (base) | 7-15 | High for text alignment |
| Text-to-video (SR) | 3-5 | Lower for quality preservation |
| Class-conditional (ImageNet) | 1.5-4.0 | Lower scale sufficient |
| Image editing (SDEdit) | 5-10 | Balance edit strength and quality |
| Text-to-3D (SDS) | 50-100 | Very high for 3D optimization |
| Inpainting | 3-8 | Lower to preserve context |

### 8.2 Tuning Guidance Scale

```
Signs of too low guidance (w < optimal):
  - Images look generic, do not match text well
  - Low CLIP score
  - Good diversity but poor specificity

Signs of too high guidance (w > optimal):
  - Oversaturated colors
  - Artifacts (checkerboard, color bleeding)
  - Loss of fine detail
  - Mode collapse (all samples look similar)
  - Unnaturally sharp edges

Tuning strategy:
  1. Start with recommended default for your model
  2. Generate batch of images at w, w-2, w+2
  3. Evaluate text alignment and visual quality
  4. Fine-tune in smaller increments
  5. Use dynamic thresholding if you need high w
```

---

## 9. Implementation Patterns

### 9.1 Standard CFG Implementation

```python
def guided_sample_step(model, x_t, t, text_emb, empty_emb, guidance_scale):
    # Batch both predictions
    x_input = torch.cat([x_t, x_t], dim=0)
    t_input = torch.cat([t, t], dim=0)
    c_input = torch.cat([empty_emb, text_emb], dim=0)
    
    # Single forward pass
    eps_both = model(x_input, t_input, c_input)
    eps_uncond, eps_cond = eps_both.chunk(2, dim=0)
    
    # Apply guidance
    eps_guided = eps_uncond + guidance_scale * (eps_cond - eps_uncond)
    
    return eps_guided
```

### 9.2 Negative Prompt Implementation

```python
def guided_sample_with_negative(model, x_t, t, text_emb, neg_emb, guidance_scale):
    x_input = torch.cat([x_t, x_t], dim=0)
    t_input = torch.cat([t, t], dim=0)
    c_input = torch.cat([neg_emb, text_emb], dim=0)
    
    eps_both = model(x_input, t_input, c_input)
    eps_neg, eps_cond = eps_both.chunk(2, dim=0)
    
    eps_guided = eps_neg + guidance_scale * (eps_cond - eps_neg)
    
    return eps_guided
```

### 9.3 Multi-Condition Composable Guidance

```python
def composable_guidance(model, x_t, t, conditions, scales, empty_emb):
    # conditions: list of conditioning embeddings
    # scales: list of guidance scales (one per condition)
    
    eps_uncond = model(x_t, t, empty_emb)
    eps_guided = eps_uncond.clone()
    
    for cond, scale in zip(conditions, scales):
        eps_cond = model(x_t, t, cond)
        eps_guided += scale * (eps_cond - eps_uncond)
    
    return eps_guided
```

---

## Source Citations

- Dhariwal, P. & Nichol, A. (2021). "Diffusion Models Beat GANs on Image Synthesis." NeurIPS 2021.
- Ho, J. & Salimans, T. (2022). "Classifier-Free Diffusion Guidance." NeurIPS 2021 Workshop.
- Saharia, C. et al. (2022). "Photorealistic Text-to-Image Diffusion Models with Deep Language Understanding." NeurIPS 2022.
- Rombach, R. et al. (2022). "High-Resolution Image Synthesis with Latent Diffusion Models." CVPR 2022.
- Zhang, L., Rao, A., & Agrawala, M. (2023). "Adding Conditional Control to Text-to-Image Diffusion Models." ICCV 2023.
- Peebles, W. & Xie, S. (2023). "Scalable Diffusion Models with Transformers." ICCV 2023.
- Singer, U. et al. (2022). "Make-A-Video." Meta AI. arXiv:2209.14792.
- Yang, L. et al. (2023). "Diffusion Models: A Comprehensive Survey." ACM Computing Surveys.
- Singh, A. (2023). "A Survey of AI Text-to-Image and AI Text-to-Video Generation."
