# Sequence Model Architectures

## RNN Cell Equations

### Vanilla RNN
```
h_t = tanh(W_hh * h_{t-1} + W_xh * x_t + b_h)
y_t = W_hy * h_t + b_y
```
- Parameters: W_hh (hidden×hidden), W_xh (hidden×input), W_hy (output×hidden)
- Problem: vanishing gradients for long sequences (gradient = product of Jacobians shrinks exponentially)

### LSTM (Long Short-Term Memory)
```
f_t = sigmoid(W_f * [h_{t-1}, x_t] + b_f)     # forget gate
i_t = sigmoid(W_i * [h_{t-1}, x_t] + b_i)     # input gate
c_tilde = tanh(W_c * [h_{t-1}, x_t] + b_c)    # candidate cell state
c_t = f_t * c_{t-1} + i_t * c_tilde            # cell state update
o_t = sigmoid(W_o * [h_{t-1}, x_t] + b_o)     # output gate
h_t = o_t * tanh(c_t)                          # hidden state
```

**Why it works**: Cell state c_t provides an uninterrupted gradient path. Forget gate f_t controls how much to retain. When f_t ≈ 1, gradients flow unchanged through time.

**Parameter count**: 4 * (hidden_dim * (hidden_dim + input_dim) + hidden_dim) for gates

### GRU (Gated Recurrent Unit)
```
r_t = sigmoid(W_r * [h_{t-1}, x_t] + b_r)     # reset gate
z_t = sigmoid(W_z * [h_{t-1}, x_t] + b_z)     # update gate
h_tilde = tanh(W_h * [r_t * h_{t-1}, x_t] + b_h)  # candidate hidden
h_t = (1 - z_t) * h_{t-1} + z_t * h_tilde     # interpolate
```

**Fewer parameters** than LSTM (3 gates vs 4, no separate cell state). Similar performance in practice.

## Architecture Comparison

| Feature | Vanilla RNN | LSTM | GRU |
|---------|-------------|------|-----|
| Gates | 0 | 3 (forget, input, output) | 2 (reset, update) |
| State variables | h | h, c | h |
| Parameters | ~H^2 | ~4H^2 | ~3H^2 |
| Long-range memory | Poor | Excellent | Good |
| Training speed | Fast | Slowest | Middle |
| When to use | Short sequences, simple tasks | Default for most tasks | When LSTM is too slow |

## Bidirectional RNN

```
Forward:   h_f_t = RNN_f(x_t, h_f_{t-1})   # process left-to-right
Backward:  h_b_t = RNN_b(x_t, h_b_{t+1})   # process right-to-left
Combined:  h_t = [h_f_t ; h_b_t]             # concatenate (2 * hidden_dim)
```

- Captures both past and future context at each position
- Use for sequence labeling (NER, POS) where whole sentence is available
- Do NOT use for generation (can't see future during generation)

## Multi-Layer (Stacked) RNN

```
Layer 1: h_1_t = RNN_1(x_t, h_1_{t-1})
Layer 2: h_2_t = RNN_2(h_1_t, h_2_{t-1})     # input = previous layer's output
...
Layer L: h_L_t = RNN_L(h_{L-1}_t, h_L_{t-1})
```

- 2-3 layers typical. Diminishing returns beyond 4.
- Dropout between layers (not within timesteps): `dropout` parameter in PyTorch LSTM

## Encoder-Decoder (Seq2Seq)

### Without Attention
```
Encoder: processes input sequence → final hidden state (h_T)
Decoder: initializes with h_T, generates output autoregressively

Problem: entire input compressed into single fixed vector — information bottleneck
```

### With Attention (Bahdanau)
```
For each decoder step t:
  1. Compute attention scores: e_{t,i} = v^T * tanh(W_1 * h_enc_i + W_2 * h_dec_t)
  2. Normalize: alpha_{t,i} = softmax(e_{t,i})
  3. Context vector: c_t = sum_i(alpha_{t,i} * h_enc_i)
  4. Decoder input: [c_t ; y_{t-1}] → decoder RNN
```

### With Attention (Luong)
```
Three score functions:
  dot:     score = h_dec^T * h_enc
  general: score = h_dec^T * W * h_enc
  concat:  score = v^T * tanh(W * [h_dec ; h_enc])

Context: c_t = sum_i(softmax(score_i) * h_enc_i)
Output:  h_tilde = tanh(W_c * [c_t ; h_dec_t])
```

## Self-Attention (Transformer Building Block)

```
Q = X * W_Q    # queries  (seq_len, d_k)
K = X * W_K    # keys     (seq_len, d_k)
V = X * W_V    # values   (seq_len, d_v)

Attention(Q, K, V) = softmax(Q * K^T / sqrt(d_k)) * V
```

- Multi-head: run h parallel attention heads with different W_Q, W_K, W_V, concatenate outputs
- Positional encoding: sine/cosine functions add position information (transformers have no inherent order)
- Parallelizable: all positions computed simultaneously (unlike RNN sequential processing)

## Architecture Selection Guide

| Task | Input → Output | Recommended |
|------|---------------|-------------|
| Text classification | Sequence → single label | BiLSTM + mean pool, or TextCNN |
| Sequence labeling | Sequence → label per token | BiLSTM-CRF |
| Translation | Sequence → sequence | Attention Seq2Seq or Transformer |
| Summarization | Long sequence → short sequence | Attention Seq2Seq with copy mechanism |
| Language modeling | Sequence → next token | Unidirectional LSTM or Transformer |
| Question answering | (question, context) → span | BiLSTM or BERT with span prediction head |
