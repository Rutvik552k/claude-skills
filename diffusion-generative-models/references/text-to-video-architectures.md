# Text-to-Video Generation Architectures

## Overview

Text-to-video (T2V) generation extends text-to-image diffusion models to the temporal
domain. The key challenge is modeling temporal coherence while leveraging pretrained
T2I models. This reference covers Make-A-Video, Imagen Video, Phenaki, CogVideo,
GODIVA, and NUWA, with emphasis on their temporal modeling strategies.

---

## 1. Make-A-Video (Singer et al., Meta AI, 2022)

### 1.1 Core Insight

The central contribution of Make-A-Video is eliminating the need for paired text-video
training data. Instead, it decomposes the problem:
```
T2V = T2I model (learns visual quality + text alignment from image-text pairs)
    + temporal layers (learns motion from unlabeled video data)
```

This is significant because:
- Large-scale paired text-video datasets are scarce and expensive
- Billions of text-image pairs are available (LAION, etc.)
- Unlabeled video captures natural motion patterns adequately

### 1.2 Overall Pipeline

```
Input: text prompt

Stage 1: Generate base video (16 frames, 64x64)
  - Prior: CLIP text embedding -> CLIP image embedding (BPE -> CLIP -> prior diffusion)
  - Base T2V decoder: CLIP image embedding -> 16x64x64 video
  - Spatiotemporal U-Net with pseudo-3D layers

Stage 2: Frame interpolation
  - Interpolation network: 16 frames -> 76 frames (insert 4 frames between each pair)
  - Temporal super-resolution for smooth motion

Stage 3: Spatial super-resolution
  - SR network 1: 76x64x64 -> 76x256x256
  - SR network 2: 76x256x256 -> 76x768x768
  - Each SR network has temporal layers for video-aware upsampling

Output: 76 frames at 768x768, approximately 5 seconds at 16fps
```

### 1.3 Spatiotemporal Decomposition

The key architectural innovation: factorize every 3D operation into spatial + temporal.

**Pseudo-3D Convolution:**
```
Standard 3D conv: Conv3D(in_c, out_c, kernel=(k_t, k_h, k_w))
Pseudo-3D decomposition:
  Step 1: Conv2D(in_c, out_c, kernel=(k_h, k_w))     # spatial, applied per-frame
  Step 2: Conv1D(out_c, out_c, kernel=(k_t,))         # temporal, applied per-pixel
```

Implementation:
```
def pseudo_3d_conv(x, conv_spatial, conv_temporal):
    # x: [B, C, T, H, W]
    B, C, T, H, W = x.shape

    # Spatial convolution (per-frame)
    x = rearrange(x, 'b c t h w -> (b t) c h w')
    x = conv_spatial(x)                                # [B*T, C', H', W']
    x = rearrange(x, '(b t) c h w -> b c t h w', b=B)

    # Temporal convolution (per-spatial-location)
    x = rearrange(x, 'b c t h w -> (b h w) c t')
    x = conv_temporal(x)                               # [B*H*W, C', T']
    x = rearrange(x, '(b h w) c t -> b c t h w', b=B, h=H', w=W')

    return x
```

**Advantage**: Initialize spatial conv from pretrained T2I model, train only temporal conv.

**Pseudo-3D Attention:**
```
Standard 3D attention: Attention over all (T*H*W) tokens - O((THW)^2) complexity
Pseudo-3D decomposition:
  Step 1: Spatial self-attention per frame    # O(T * (HW)^2)
  Step 2: Temporal self-attention per position # O(HW * T^2)
```

Implementation:
```
def pseudo_3d_attention(x, spatial_attn, temporal_attn):
    # x: [B, T, H, W, D]  (D = embedding dimension)
    B, T, H, W, D = x.shape

    # Spatial attention (per-frame)
    x = rearrange(x, 'b t h w d -> (b t) (h w) d')
    x = spatial_attn(x)                               # self-attention over HW tokens
    x = rearrange(x, '(b t) (h w) d -> b t h w d', b=B, t=T, h=H, w=W)

    # Temporal attention (per-spatial-position)
    x = rearrange(x, 'b t h w d -> (b h w) t d')
    x = temporal_attn(x)                               # self-attention over T tokens
    x = rearrange(x, '(b h w) t d -> b t h w d', b=B, h=H, w=W)

    return x
```

### 1.4 Initialization Strategy

Critical for leveraging pretrained T2I knowledge:
```
Spatial layers: initialized from pretrained T2I model (frozen or low learning rate)
Temporal layers: initialized to identity/zero (so initial model = T2I applied per-frame)
  - Conv1D temporal: initialize kernel to [0, ..., 0, 1, 0, ..., 0] (identity in center)
  - Temporal attention: initialize output projection to zero

This ensures:
  - At initialization, the model generates identical frames (static video)
  - Temporal layers gradually learn motion during training
  - Pretrained image quality is preserved
```

### 1.5 Frame Interpolation Network

```
Input: 16 frames at 64x64 (from base model)
Architecture: U-Net with temporal layers
  - Masked diffusion: condition on known frames (frame 0 and frame 1)
  - Generate intermediate frames between each pair
  - Can be applied recursively for higher frame rates

Output: 76 frames at 64x64
  - 16 original + 15*4 interpolated = 76 frames
```

### 1.6 Spatial Super-Resolution

```
SR Network 1 (64 -> 256):
  Input: low-res frame (upsampled) + noise
  Architecture: U-Net with temporal layers
  Conditioning: CLIP image embedding + low-res via concatenation
  Noise augmentation: add noise to low-res conditioning

SR Network 2 (256 -> 768):
  Similar architecture, processes higher resolution
  Temporal layers ensure frame-to-frame consistency during upsampling
```

### 1.7 Training Data

```
Image-text data: LAION-5B subset (image-text pairs)
  - Used for spatial (T2I) pretraining
  - ~2.3B pairs after filtering

Video data: WebVid-10M + HD-VILA-100M (unlabeled video)
  - Used for temporal layer training
  - No text labels needed for video
  - Random clips of 16 frames

Training procedure:
  1. Train T2I model on image-text data
  2. Add temporal layers, initialize to identity
  3. Train temporal layers on video data (spatial layers frozen or low LR)
  4. Fine-tune frame interpolation and SR networks on video data
```

### 1.8 Key Results

```
UCF-101 (zero-shot): FVD = 367.23, IS = 33.00
MSR-VTT: CLIPSIM = 0.3049
Resolution: 768x768, 16fps, ~5 seconds
Qualitative: strong motion dynamics, good text alignment
```

---

## 2. Imagen Video (Ho et al., Google Brain, 2022)

### 2.1 Architecture Overview

Imagen Video uses a massive cascade of 7 diffusion models:
```
1. Base video model:      16 frames x 40x24    (text -> low-res short video)
2. Temporal SR 1:         16 -> 64 frames      (increase frame rate)
3. Spatial SR 1:          64 frames x 80x48    (spatial upsampling)
4. Temporal SR 2:         64 -> 128 frames     (further frame rate increase)
5. Spatial SR 2:          128 frames x 320x192 (spatial upsampling)
6. Spatial SR 3:          128 frames x 1280x768 (final spatial upsampling)
7. Temporal SR 3:         ~128 frames (optional final temporal refinement)
```

Final output: approximately 128 frames at 1280x768, ~5.3 seconds at 24fps.

### 2.2 Text Conditioning

Uses T5-XXL (same as Imagen for images):
```
Text -> T5-XXL encoder -> token embeddings [B, L, 4096]

Conditioning in base model:
  - Cross-attention to T5 embeddings (full sequence)
  - Pooled text embedding added to timestep embedding

Conditioning in SR models:
  - Cross-attention to T5 embeddings
  - Low-resolution video via concatenation
  - Noise augmentation on low-res conditioning
```

### 2.3 v-Prediction Parameterization

Imagen Video uses v-prediction instead of epsilon-prediction:
```
v = alpha_t * epsilon - sigma_t * x_0

where:
  alpha_t = sqrt(alpha_bar_t)      (signal coefficient)
  sigma_t = sqrt(1 - alpha_bar_t)  (noise coefficient)
```

**Advantages for video:**
- More stable training at high noise levels
- Better for progressive distillation (enables faster sampling)
- Smoother loss landscape across timesteps
- Recovery: x_0 = alpha_t * x_t - sigma_t * v_theta
           epsilon = sigma_t * x_t + alpha_t * v_theta

### 2.4 Temporal Architecture

The base model and SR models use 3D U-Nets with:
```
Temporal attention: full attention across the time axis
  - Applied at each spatial resolution in the U-Net
  - Temporal position embeddings for frame ordering

Temporal convolutions: 1D convolutions along time axis
  - Applied after spatial convolutions
  - Capture local temporal patterns

Joint spatial-temporal blocks:
  ResBlock -> SpatialAttn -> TemporalAttn -> CrossAttn(text) -> ResBlock
```

### 2.5 Progressive Distillation for Video

Apply progressive distillation to each model in the cascade:
```
Original: each model uses ~100-256 DDPM steps
After distillation: each model uses ~8 steps

Total NFE reduction:
  Original: 7 models * ~150 steps = ~1050 forward passes
  Distilled: 7 models * ~8 steps = ~56 forward passes
  ~18x speedup
```

v-prediction is critical for stable distillation in the video setting.

### 2.6 Joint Image-Video Training

Train base model on both images (single frame) and videos:
```
Image training: treat image as 1-frame video
  - Same architecture, T=1
  - Access to large image-text datasets

Video training: standard multi-frame training
  - Smaller video-text datasets

Benefits:
  - Better per-frame quality from image data
  - Better motion from video data
  - Regularization prevents overfitting to small video datasets
```

### 2.7 Classifier-Free Guidance in Video

```
Guidance applied at each level of the cascade:
  eps_guided = eps_uncond + w * (eps_cond - eps_uncond)

Guidance scales (typical):
  Base model: w = 7-15 (high for text alignment)
  Temporal SR: w = 3-5 (lower, text alignment already established)
  Spatial SR: w = 3-7 (moderate)
```

### 2.8 Key Results

```
Resolution: 1280x768 at 24fps
Duration: ~5.3 seconds (128 frames)
Quality: high-fidelity, temporally coherent
Limitations: not publicly released, high compute requirements
Evaluation: human preference studies, FVD on benchmarks
```

---

## 3. Phenaki (Villegas et al., Google Research, 2023)

### 3.1 Architecture Overview

Phenaki generates variable-length videos from time-indexed text prompts:
```
Input: sequence of text prompts with time boundaries
  "A cat sits on a windowsill" (0-3s)
  "The cat jumps down" (3-5s)
  "The cat walks across the room" (5-8s)

Output: continuous video with scene transitions matching prompts
```

### 3.2 C-ViViT (Causal Video ViT)

Video tokenizer based on Vision Transformer:
```
Architecture:
  Spatial encoder: ViT applied per-frame -> spatial tokens
  Causal temporal encoder: transformer with causal attention over time
    - Each frame attends only to current and previous frames
    - Enables autoregressive video extension

Quantization: learned codebook for discrete tokens
  - Spatial tokens quantized to discrete vocabulary
  - Enables token-based generation

Decoder: reverse process (tokens -> frames)
  Temporal decoder (causal) -> spatial decoder (per-frame)
```

### 3.3 Bidirectional Masked Transformer

Generation model (not diffusion):
```
Input: text embeddings (T5-XXL) + partial video tokens (masked)
Architecture: bidirectional transformer
  - Predict masked video tokens given text and visible tokens
  - MaskGIT-style iterative decoding:
    Iteration 1: predict all tokens, keep most confident
    Iteration 2: re-mask uncertain tokens, predict again
    ... repeat for N iterations (typically 12-24)

Conditioning: T5-XXL text embeddings via cross-attention
Time conditioning: which text prompt applies to which frames
```

### 3.4 Variable-Length Generation

```
Autoregressive in time:
  1. Generate first chunk of K frames
  2. Condition on last M frames (context window)
  3. Generate next chunk of K frames
  4. Repeat until desired length

Context window allows:
  - Arbitrary video length (not limited by model capacity)
  - Scene transitions guided by new text prompts
  - Temporal coherence via causal attention in C-ViViT
```

### 3.5 Training

```
Video tokenizer (C-ViViT):
  - Trained on large video dataset
  - Reconstruction loss + codebook commitment loss
  - Perceptual loss for quality

Masked transformer:
  - Trained on tokenized videos with T5 text conditioning
  - Random masking ratio (cosine schedule from high to low)
  - Joint training on image (single frame) and video data
```

### 3.6 Key Results

```
Capabilities: variable-length (minutes), multi-scene, diverse styles
Resolution: 256x256 (base), potentially higher with SR
Frame rate: 8fps base
Unique strength: multi-prompt time-indexed generation
Limitation: lower resolution than Imagen Video / Make-A-Video
```

---

## 4. CogVideo (Hong et al., 2022)

### 4.1 Architecture Overview

CogVideo extends CogView2 (T2I transformer) to video:
```
Architecture: 9.4B parameter transformer
  - Inherits CogView2's text-image understanding
  - Extended with temporal modeling via frame-level attention
```

### 4.2 Multi-Frame-Rate Hierarchical Training

```
Stage 1: Key frame generation (low frame rate)
  - Generate 5 key frames at ~2fps
  - Full text-image attention for quality

Stage 2: Frame interpolation (high frame rate)
  - Interpolate between key frames
  - Dual-channel attention:
    Channel 1: spatial attention within each frame
    Channel 2: temporal attention across frames at same position
  - Produces 33 frames at ~8fps
```

### 4.3 Dual-Channel Attention

```
Given video tokens V of shape [T, H*W]:

Channel 1 - Spatial (within-frame):
  For each frame t:
    Q, K, V = linear(V[t])
    attn_spatial[t] = softmax(Q @ K^T / sqrt(d)) @ V

Channel 2 - Temporal (across-frame):
  For each spatial position (h, w):
    Q, K, V = linear(V[:, h*W+w])
    attn_temporal[:, h*W+w] = softmax(Q @ K^T / sqrt(d)) @ V

Output: alpha * attn_spatial + (1 - alpha) * attn_temporal
```

### 4.4 Training Data

```
Dataset: 5.4M text-video pairs
  - Chinese text descriptions (primarily)
  - Filtered from web-scraped video data
  - Additional image-text data for pretraining

Tokenization:
  - CogView2's VQ-VAE: 160x160 image -> 20x20 tokens (400 tokens per frame)
  - 33 frames = 13,200 video tokens + text tokens
```

### 4.5 Inference

```
Text encoding: CogLM tokenizer
Key frame generation: autoregressive with nucleus sampling
Frame interpolation: parallel decoding with dual-channel attention
Total generation time: ~2 minutes for 4-second video on 8 GPUs
Resolution: 480x480 at 8fps, 4 seconds (32 frames)
```

---

## 5. GODIVA (Wu et al., Microsoft, 2021)

### 5.1 Architecture Overview

One of the earliest T2V models using sparse attention transformers:
```
Architecture: VQ-VAE tokenizer + autoregressive transformer
Pipeline: text -> GODIVA transformer -> video tokens -> VQ-VAE decoder -> video
```

### 5.2 3D Sparse Attention

Full 3D attention is O((T*H*W)^2) which is prohibitive for video.
GODIVA uses factored sparse attention:
```
Attention pattern 1 - Row attention:
  Each token attends to all tokens in the same row (spatial horizontal)

Attention pattern 2 - Column attention:
  Each token attends to all tokens in the same column (spatial vertical)

Attention pattern 3 - Temporal attention:
  Each token attends to the same spatial position in all frames

Combined: stack these three attention patterns in each transformer layer
Complexity: O(T*H*W * max(H, W, T)) instead of O((T*H*W)^2)
```

### 5.3 Generation Process

```
1. Encode text with pretrained text encoder
2. Autoregressive generation of video tokens:
   - Row by row, frame by frame
   - Conditioned on text via cross-attention
   - Sparse attention for tractable computation
3. Decode video tokens with VQ-VAE decoder
```

### 5.4 Training

```
Dataset: HowTo100M subset + MSR-VTT
Tokenization: VQ-VAE with 8192 vocabulary
Video tokens: 5 frames x 16x16 = 1280 tokens per video
Text tokens: BPE tokenizer, max 64 tokens
Training: cross-entropy loss on next-token prediction
```

### 5.5 Results and Limitations

```
Resolution: 128x128 at low frame rate
Duration: short clips (~2-3 seconds)
Quality: pioneering but limited by small model and data scale
Historical significance: early demonstration of transformer-based T2V
```

---

## 6. NUWA (Wu et al., Microsoft, 2022)

### 6.1 Architecture Overview

Unified visual generation model for multiple tasks:
```
Tasks:
  - Text-to-image (T2I)
  - Text-to-video (T2V)
  - Video prediction (V -> V)
  - Video manipulation
  - Sketch-to-image
  - Image completion
  - Video completion
  - Image manipulation

Architecture: 3D-Aware Transformer Encoder-Decoder
```

### 6.2 3D Nearby Attention (3DNA)

NUWA's key innovation for efficient 3D attention:
```
Standard 3D attention: attend to all (T, H, W) positions
3DNA: attend only to a local 3D neighborhood

For a token at position (t, h, w):
  Attend to: {(t', h', w') : |t-t'| <= r_t, |h-h'| <= r_h, |w-w'| <= r_w}

Window sizes:
  r_t = 1 (1 frame before, current frame, 1 frame after)
  r_h = r_w = 3 (7x7 spatial window)

Receptive field per layer: (3, 7, 7)
Stack N layers: effective receptive field grows with depth
```

### 6.3 Adaptive Encoder

```
Text encoder: shared across all tasks
  - Pretrained transformer encoder
  - Output: text feature sequence

Visual encoder: 3D convolution + 3D nearby attention
  - Encodes input visual conditions (sketch, low-res, partial video)
  - Adapts to different modalities via task-specific input heads

Cross-modal attention: text features attend to visual features
```

### 6.4 Decoder with 3DNA

```
Autoregressive decoder with 3D nearby attention:
  - Generate tokens in raster scan order (left-to-right, top-to-bottom, frame-by-frame)
  - Causal masking ensures autoregressive property
  - 3DNA limits attention to local 3D window for efficiency
  - Cross-attention to encoder outputs (text + visual conditions)
```

### 6.5 Training

```
Datasets:
  - Image-text: 14M pairs (Conceptual Captions, SBU, etc.)
  - Video-text: 2M pairs (WebVid subset, HowTo100M subset)
  - Task-specific data for each of the 8 tasks

Training strategy:
  - Multi-task training with task tokens
  - Shared parameters across tasks
  - Task-specific input/output heads
  - VQ-GAN tokenizer (discrete tokens)
```

### 6.6 NUWA-Infinity Extension

```
Autoregressive visual generation at arbitrary resolution and duration:
  - Patch-by-patch generation with overlapping context
  - Can generate very long videos or very high-resolution images
  - Consistency maintained via overlapping attention windows
```

---

## 7. Temporal Modeling Comparison

### 7.1 Temporal Attention Patterns

| Model | Temporal Mechanism | Complexity | Init Strategy |
|-------|-------------------|------------|---------------|
| Make-A-Video | Pseudo-3D (factored spatial+temporal) | O(T^2) per position | Zero/identity from T2I |
| Imagen Video | Full temporal attention | O(T^2) per resolution | Joint image-video training |
| Phenaki | Causal temporal in C-ViViT | O(T^2) with causal mask | Separate tokenizer training |
| CogVideo | Dual-channel attention | O(T^2) per position | Inherited from CogView2 |
| GODIVA | Sparse temporal attention | O(T) per position | From scratch |
| NUWA | 3D nearby attention | O(r_t * r_h * r_w) | From scratch |

### 7.2 Resolution and Duration Comparison

| Model | Max Resolution | Max Duration | Frame Rate | Frames |
|-------|---------------|-------------|-----------|--------|
| Make-A-Video | 768x768 | ~5s | 16fps | 76 |
| Imagen Video | 1280x768 | ~5.3s | 24fps | 128 |
| Phenaki | 256x256 | Minutes | 8fps | Variable |
| CogVideo | 480x480 | ~4s | 8fps | 32 |
| GODIVA | 128x128 | ~2-3s | Low | ~15 |
| NUWA | 256x256 | ~3-4s | Low | ~32 |

### 7.3 Training Data Requirements

| Model | Text-Image Pairs | Text-Video Pairs | Unlabeled Video |
|-------|-----------------|-----------------|-----------------|
| Make-A-Video | ~2.3B | 0 | 20M+ clips |
| Imagen Video | Large (Imagen) | ~14M | Joint training |
| Phenaki | Large | Moderate | Yes |
| CogVideo | ~30M | ~5.4M | No |
| GODIVA | Small | ~100K | No |
| NUWA | ~14M | ~2M | No |

### 7.4 Key Design Decisions

```
Diffusion vs Autoregressive:
  - Diffusion (Make-A-Video, Imagen Video): higher quality, slower generation
  - Autoregressive (CogVideo, GODIVA, NUWA): faster generation, lower quality
  - Hybrid (Phenaki): masked bidirectional for quality + autoregressive for length

Pixel vs Latent vs Token:
  - Pixel space (Make-A-Video base, Imagen Video): highest fidelity, most compute
  - Latent space (emerging): efficient, good quality
  - Token space (CogVideo, GODIVA, NUWA): discrete, fast, lower fidelity

T2I Transfer:
  - Strong transfer (Make-A-Video): explicit T2I -> T2V pipeline
  - Joint training (Imagen Video): simultaneous image and video training
  - Inherited (CogVideo): extended from T2I model architecture
  - From scratch (GODIVA, NUWA): no explicit T2I transfer
```

---

## 8. Common Challenges in T2V

### 8.1 Temporal Coherence

```
Problem: flickering, jittering, sudden changes between frames
Solutions:
  - Temporal attention/convolution layers
  - Shared noise across frames (correlated noise initialization)
  - Temporal super-resolution (interpolation-based smoothing)
  - Progressive training: frames -> short clips -> longer clips
```

### 8.2 Motion Quality

```
Problem: static videos, unnatural motion, motion blur
Solutions:
  - Large-scale video pretraining
  - Motion-specific loss terms
  - Frame interpolation networks
  - Multi-frame-rate training (CogVideo)
```

### 8.3 Text-Video Alignment

```
Problem: generated video does not match text description
Solutions:
  - Strong text encoders (T5-XXL, large CLIP)
  - Classifier-free guidance (high scale for base model)
  - Joint text-video attention mechanisms
  - Multi-prompt conditioning (Phenaki)
```

### 8.4 Computational Cost

```
Problem: video generation is extremely expensive
  - 16 frames at 256x256 = 16x the pixels of a single image
  - Full 3D attention is O(T^2 * H^2 * W^2)

Solutions:
  - Factored attention (Make-A-Video pseudo-3D)
  - Sparse attention (GODIVA, NUWA 3DNA)
  - Latent space generation (emerging approaches)
  - Progressive distillation (Imagen Video)
  - Cascaded pipelines (low-res -> high-res)
```

---

## Source Citations

- Singer, U., Polyak, A., Hayes, T., et al. (2022). "Make-A-Video: Text-to-Video Generation without Text-Video Data." Meta AI. arXiv:2209.14792.
- Ho, J., Chan, W., Saharia, C., et al. (2022). "Imagen Video: High Definition Video Generation with Diffusion Models." Google Brain. arXiv:2210.02303.
- Villegas, R., Babaeizadeh, M., Kindermans, P.-J., et al. (2023). "Phenaki: Variable Length Video Generation from Open Domain Textual Descriptions." ICML 2023.
- Hong, W., Ding, M., Zheng, W., Liu, X., & Tang, J. (2022). "CogVideo: Large-scale Pretraining for Text-to-Video Generation via Transformers." ICLR 2023.
- Wu, C., Liang, J., Ji, L., et al. (2021). "GODIVA: Generating Open-DomaIn Videos from nAtural Descriptions." arXiv:2104.14806.
- Wu, C., Liang, J., Hu, X., et al. (2022). "NUWA: Visual Synthesis Pre-training for Neural Visual World Creation." ECCV 2022.
- Singh, A. (2023). "A Survey of AI Text-to-Image and AI Text-to-Video Generation."
- Yang, L. et al. (2023). "Diffusion Models: A Comprehensive Survey." ACM Computing Surveys.
