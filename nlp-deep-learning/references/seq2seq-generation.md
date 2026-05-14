# Sequence-to-Sequence and Text Generation

## Encoder-Decoder Framework

### Components
1. **Encoder**: processes source sequence into contextualized representations
2. **Decoder**: generates target sequence token-by-token, conditioned on encoder output
3. **Attention**: dynamic alignment between decoder state and encoder positions

### Training
- **Teacher forcing**: feed ground truth y_{t-1} as input to decoder at step t
  - Pro: faster convergence, stable training
  - Con: exposure bias — at inference, model sees its own (possibly wrong) predictions
- **Scheduled sampling**: probability p of using ground truth, (1-p) of using model prediction
  - p starts at 1.0, linearly decays to ~0.25 over training
  - Bridges gap between training and inference

### Inference Modes

## Greedy Decoding
```
At each step: y_t = argmax P(y | y_1, ..., y_{t-1}, x)
```
- O(T * V) per sequence (T = max length, V = vocab size)
- No backtracking — can produce suboptimal sequences
- Fast, good enough for many applications

## Beam Search
```
Maintain B candidate sequences (beams)
At each step:
  For each beam, compute P(y | beam_history)
  Expand each beam by top-B tokens → B*B candidates
  Keep top-B candidates by cumulative log-probability
Continue until all beams produce <EOS> or max_len reached
Return highest-scoring complete sequence
```

### Key Parameters
| Parameter | Typical | Effect |
|-----------|---------|--------|
| Beam width B | 4-10 | Larger = better quality, slower |
| Length penalty alpha | 0.6-0.7 | Score / length^alpha prevents short bias |
| Max length | 1.5x source length | Prevents infinite generation |
| Early stopping | Yes | Stop beam when top candidate has <EOS> |

### Length Normalization
```
score(y) = log P(y_1, ..., y_T) / T^alpha

Without normalization: shorter sequences have higher raw probability
Alpha = 0: no normalization (favors short)
Alpha = 1: divide by length (favors long)
Alpha = 0.6-0.7: sweet spot for most tasks
```

## Sampling Strategies for Open-Ended Generation

### Temperature Sampling
```
P(y_i) = exp(z_i / T) / sum_j exp(z_j / T)

T = 1.0: standard softmax (original distribution)
T < 1.0: sharper distribution (more deterministic, less diverse)
T > 1.0: flatter distribution (more diverse, more random)
T → 0: argmax (greedy)
T → inf: uniform distribution
```

### Top-k Sampling
```
1. Compute logits for all vocabulary tokens
2. Keep only top-k tokens by probability
3. Renormalize probabilities over k tokens
4. Sample from this truncated distribution
```
- k = 10-50 typical
- Problem: fixed k doesn't adapt to distribution shape (confident vs uncertain contexts)

### Nucleus (Top-p) Sampling
```
1. Sort tokens by probability (descending)
2. Find smallest set S where sum of probabilities >= p
3. Renormalize over S
4. Sample from S
```
- p = 0.9-0.95 typical
- Dynamically adjusts: narrow distribution → few tokens; flat → many tokens
- Generally preferred over top-k

### Combined Strategy
```
Best practice: temperature (0.7-0.9) + top-p (0.9-0.95) + top-k (50)
Apply temperature first, then top-k, then top-p, then sample
```

## Evaluation Metrics

### BLEU (Bilingual Evaluation Understudy)
```
BLEU-N = BP * exp(sum_{n=1}^{N} w_n * log(precision_n))

precision_n = matched n-grams / total n-grams in candidate
BP = min(1, exp(1 - ref_len/cand_len))  # brevity penalty

BLEU-4 (standard): w_1 = w_2 = w_3 = w_4 = 0.25
```
- Range: 0-1 (higher = better)
- Measures n-gram precision vs reference
- Limitations: doesn't capture meaning, penalizes valid paraphrases

### ROUGE (Recall-Oriented Understudy for Gisting Evaluation)
```
ROUGE-N: recall of n-grams from reference in candidate
ROUGE-L: longest common subsequence (LCS) based
ROUGE-1: unigram recall (most common for summarization)
```

### Perplexity (Language Model Evaluation)
```
PPL = exp(-1/N * sum_{i=1}^{N} log P(w_i | w_1, ..., w_{i-1}))
```
- Lower = better
- Measures how well model predicts held-out text
- 1 = perfect prediction, larger = worse

### Human Evaluation
- **Fluency**: is the output grammatical and natural?
- **Adequacy**: does it preserve the source meaning?
- **Coherence**: does it make logical sense?
- Gold standard but expensive and slow

## Practical Tips

### Training
1. Start with teacher forcing, transition to scheduled sampling after 5-10 epochs
2. Use gradient clipping (max norm 1.0-5.0) to prevent exploding gradients
3. Learning rate: start 1e-3 for Adam, reduce on plateau
4. Batch by similar lengths to minimize padding waste
5. Label smoothing (0.1) can improve generation quality

### Inference
1. Beam search for translation/summarization (quality matters)
2. Sampling with temperature+top-p for creative generation (diversity matters)
3. Always set max generation length to prevent infinite loops
4. Consider repetition penalty for long generation (penalize already-generated tokens)

### Debugging
- If output is repetitive: lower temperature, add repetition penalty
- If output is generic/boring: raise temperature, increase top-k/top-p
- If output cuts off early: increase max length, adjust length penalty
- If BLEU is low: check tokenization consistency between candidate and reference
