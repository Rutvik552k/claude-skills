---
name: "nlp-deep-learning"
description: "NLP with deep learning and PyTorch: text preprocessing, word embeddings (Word2Vec, GloVe, FastText), CNNs/RNNs/LSTMs for text, attention mechanisms, sequence-to-sequence models, and text generation. Use when building NLP pipelines, choosing embedding strategies, implementing sequence models, or generating text."
origin: ECC
---

# NLP Deep Learning

Comprehensive knowledge base for building NLP applications using deep learning. Covers the full pipeline from text preprocessing through embedding, sequence modeling, and generation — with practical PyTorch implementation patterns.

## When to Activate

- Building text classification, sentiment analysis, or NER systems
- Choosing between embedding methods (Word2Vec vs GloVe vs learned)
- Implementing RNN/LSTM/GRU models for sequence tasks
- Building encoder-decoder or attention-based models
- Text generation with beam search or sampling strategies
- PyTorch NLP implementation patterns (DataLoader, embedding layers, training loops)
- Understanding the conceptual bridge from RNNs to transformers

## Topics Index

### Foundations
- **Supervised Learning for NLP**: observation/target pairs, train/eval/test splits, loss functions (cross-entropy for classification, MSE for regression), evaluation metrics (accuracy, precision, recall, F1)
- **Computational Graphs**: static vs dynamic graphs, PyTorch autograd, automatic gradient computation, backward pass
- **PyTorch Basics**: tensors and operations, GPU acceleration (`.to(device)`), Dataset/DataLoader, `nn.Module`, `optim`, training loop pattern (zero_grad → forward → loss → backward → step)

### Text Preprocessing
- **Tokenization**: word-level, subword (BPE, WordPiece, SentencePiece), character-level, trade-offs between vocabulary size and coverage
- **Normalization**: lowercasing, stemming (Porter, Snowball), lemmatization (WordNet), stopword removal
- **Bag-of-Words & TF-IDF**: document-term matrix, term frequency, inverse document frequency (IDF = log(N/df)), TF-IDF weighting
- **N-gram Language Models**: unigram, bigram, trigram, smoothing (Laplace, Kneser-Ney), perplexity evaluation

### NLP Tasks
- **Text Classification**: sentiment analysis, spam detection, topic classification — single label per document
- **Part-of-Speech Tagging**: POS tag sets (Penn Treebank), sequence labeling formulation
- **Named Entity Recognition**: entity types (PER, ORG, LOC, MISC), BIO/BIOES tagging scheme, sequence labeling
- **Machine Translation**: parallel corpora, alignment, BLEU score evaluation

### Embeddings & Representation
- **One-Hot Encoding**: sparse representation (V-dimensional), no semantic similarity, memory-intensive
- **Word2Vec**: CBOW (predict center from context), Skip-gram (predict context from center), negative sampling, window size, embedding dimension trade-offs
- **GloVe**: global co-occurrence matrix, weighted least-squares objective, combines global statistics with local context
- **FastText**: character n-gram embeddings, handles OOV words, morphologically rich languages
- **Embedding Layers**: `nn.Embedding(vocab_size, embed_dim)`, pretrained loading, fine-tuning vs frozen embeddings
- **Contextual vs Static**: Word2Vec/GloVe produce one vector per word; ELMo/BERT produce context-dependent vectors

### Feed-Forward Networks for NLP
- **MLP for Text Classification**: input (BoW or averaged embeddings) → hidden layers → softmax output
- **Activation Functions**: ReLU (default), sigmoid (binary), tanh (centered), GELU (modern transformers)
- **Regularization**: dropout (randomly zero activations), batch normalization, weight decay (L2)
- **Architecture Patterns**: embedding → pooling (mean/max) → FC → ReLU → dropout → FC → softmax

### Convolutional Networks for NLP
- **1D Convolutions Over Text**: filter slides over word sequence, detects local n-gram patterns
- **Multiple Filter Sizes**: parallel convolutions with different kernel widths (e.g., 3, 4, 5) capture different n-gram lengths
- **Max-Over-Time Pooling**: extract strongest feature activation per filter — produces fixed-size representation
- **TextCNN Architecture**: embedding → parallel conv1d (multiple sizes) → max pool → concat → FC → softmax

### Recurrent Neural Networks
- **Vanilla RNN**: h_t = tanh(W_hh * h_{t-1} + W_xh * x_t + b). Sequential processing, hidden state carries context.
- **Vanishing/Exploding Gradients**: gradient magnitude shrinks/grows exponentially through time steps. Solved by gating (LSTM/GRU) or gradient clipping.
- **LSTM**: forget gate (what to discard from cell state), input gate (what new info to store), output gate (what to output from cell state). Cell state provides "highway" for gradients.
- **GRU**: reset gate (how much past to forget), update gate (how much to update). Fewer parameters than LSTM, similar performance.
- **Bidirectional RNN**: forward RNN (left-to-right) + backward RNN (right-to-left), concatenated output captures both past and future context
- **Packed Sequences**: `pack_padded_sequence` / `pad_packed_sequence` in PyTorch for variable-length batches, avoids wasted computation on padding

### Attention Mechanisms
- **Motivation**: fixed-size bottleneck in encoder final state limits information flow for long sequences
- **Additive (Bahdanau) Attention**: score = v^T * tanh(W_1 * h_enc + W_2 * h_dec). Learned alignment between encoder and decoder states.
- **Dot-Product (Luong) Attention**: score = h_dec^T * h_enc. Simpler, faster. Scaled version: score / sqrt(d_k).
- **Self-Attention**: query, key, value all from same sequence. Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V
- **Context Vector**: weighted sum of encoder states using attention weights — dynamic, position-specific summary

### Sequence-to-Sequence Models
- **Encoder-Decoder Architecture**: encoder processes input → final hidden state → decoder generates output token by token
- **Teacher Forcing**: during training, feed ground-truth previous token to decoder (faster convergence, exposure bias)
- **Scheduled Sampling**: gradually shift from teacher forcing to model predictions during training
- **Attention-Augmented Seq2Seq**: decoder attends to all encoder states, not just final — handles long sequences

### Text Generation
- **Greedy Decoding**: always pick highest-probability next token. Fast but suboptimal (can miss better overall sequences).
- **Beam Search**: maintain top-k partial sequences, expand each, keep top-k overall. Width 4-10 typical. Length normalization prevents short-sequence bias.
- **Temperature Sampling**: divide logits by temperature T before softmax. T<1 = sharper (more deterministic), T>1 = flatter (more diverse).
- **Top-k Sampling**: sample from top k most probable tokens only. Prevents low-probability garbage.
- **Nucleus (Top-p) Sampling**: sample from smallest set of tokens whose cumulative probability exceeds p. Dynamically adjusts k per step.

### Transfer Learning for NLP
- **Pretraining**: train language model on large corpus (unsupervised). Learns syntax, semantics, world knowledge.
- **Fine-Tuning**: adapt pretrained model to downstream task with task-specific head. Usually lower learning rate.
- **Feature Extraction**: freeze pretrained weights, use as fixed feature extractor. Faster, less data needed.
- **Transformer Bridge**: self-attention replaces recurrence. Positional encoding adds sequence order. Foundation for BERT, GPT, T5.

## Key Concepts

### Embedding Method Comparison
| Method | Type | OOV Handling | Training | Best For |
|--------|------|-------------|----------|----------|
| One-Hot | Sparse | No | None | Baseline |
| Word2Vec | Dense, static | No | Predictive (CBOW/SG) | General word similarity |
| GloVe | Dense, static | No | Count-based + regression | Analogy tasks |
| FastText | Dense, static | Yes (subword) | Predictive + char n-grams | Morphologically rich languages |
| ELMo | Dense, contextual | Partial | BiLSTM language model | Tasks needing polysemy |
| BERT | Dense, contextual | Yes (WordPiece) | Masked LM + NSP | Most NLP tasks (fine-tune) |

### Sequence Model Selection
| Task | Best Architecture | Why |
|------|------------------|-----|
| Text classification | TextCNN or LSTM + pooling | CNN: fast, captures n-grams. LSTM: handles long-range. |
| Sequence labeling (NER, POS) | BiLSTM-CRF | Bidirectional context + CRF enforces valid label transitions |
| Machine translation | Attention Seq2Seq or Transformer | Attention handles alignment; Transformer parallelizes |
| Text generation | Autoregressive RNN or Transformer | Generate one token at a time, condition on previous |
| Sentiment (short text) | CNN or averaged embeddings + MLP | Short context, n-gram features suffice |

## Problem-Solving Patterns

| Pattern | When to Use | Key Insight |
|---------|-------------|-------------|
| Pretrained embeddings → fine-tune | Limited labeled data, standard NLP task | Transfer learning saves data; freeze initially, then unfreeze |
| BiLSTM + CRF | Sequence labeling with label dependencies | CRF layer models transition constraints (e.g., I-PER can't follow B-LOC) |
| Multi-filter CNN | Short text classification | Parallel filters of size 3,4,5 capture different n-gram patterns simultaneously |
| Attention + copy mechanism | Summarization, rare word handling | Copy attention allows directly copying input tokens to output |
| Curriculum learning | Training on noisy or varying-difficulty data | Start with easy examples, gradually increase difficulty |

## Common Pitfalls

| Pitfall | Correct Approach |
|---------|-----------------|
| Not padding/packing variable-length sequences | Use `pad_sequence` + `pack_padded_sequence` for efficient batching |
| Training embeddings from scratch on small data | Use pretrained (Word2Vec, GloVe) and fine-tune |
| Ignoring class imbalance in classification | Use weighted loss, oversampling, or focal loss |
| Fixed teacher forcing ratio throughout training | Use scheduled sampling — gradually reduce teacher forcing |
| Using vanilla RNN for long sequences | Use LSTM or GRU to handle vanishing gradients |
| Not using gradient clipping with RNNs | Clip gradient norm to 1.0-5.0 to prevent exploding gradients |
| Beam search without length normalization | Divide score by length^alpha (alpha ~ 0.6-0.7) |

## Source Material

- *Natural Language Processing with PyTorch* by Delip Rao and Brian McMahan (O'Reilly, 2019)
