# Word Embedding Methods

## Overview

Word embeddings map words to dense, low-dimensional vectors where semantic similarity corresponds to geometric proximity.

## Word2Vec

### CBOW (Continuous Bag of Words)
- **Goal**: Predict center word from context words
- **Input**: One-hot vectors of context words (window size w)
- **Architecture**: Context one-hots → shared embedding → average → output projection → softmax
- **Objective**: maximize P(w_center | w_context)
- **Faster training** than Skip-gram, better for frequent words

### Skip-gram
- **Goal**: Predict context words from center word
- **Input**: One-hot of center word
- **Architecture**: Center one-hot → embedding → output projection → softmax for each context position
- **Objective**: maximize P(w_context | w_center)
- **Better for rare words**, produces higher-quality embeddings for small datasets

### Negative Sampling
- Full softmax over vocabulary V is expensive (O(|V|) per example)
- Instead: for each positive (word, context) pair, sample k negative (word, random_word) pairs
- Binary classification: is this a real co-occurrence or noise?
- Typical k: 5-20 for small datasets, 2-5 for large

### Hyperparameters
| Parameter | Typical Values | Effect |
|-----------|---------------|--------|
| Embedding dimension | 100-300 | Larger = more expressive, more data needed |
| Window size | 5-10 | Larger = more semantic; smaller = more syntactic |
| Min count | 5-10 | Filter rare words to reduce noise |
| Negative samples | 5-15 | More = better for small data, slower |
| Learning rate | 0.025 (initial) | Linear decay during training |

## GloVe (Global Vectors)

### Key Insight
Co-occurrence ratios encode meaning. P(ice|solid) / P(ice|gas) is large because solid relates to ice.

### Algorithm
1. Build global word-word co-occurrence matrix X from corpus
2. Minimize weighted least-squares objective:
   J = sum_{i,j} f(X_ij) * (w_i^T * w_j + b_i + b_j - log(X_ij))^2
3. f(x) is weighting function that caps at x_max (typically 100):
   f(x) = (x/x_max)^0.75 if x < x_max, else 1

### Properties
- Combines advantages of count-based (LSA) and predictive (Word2Vec) methods
- Directly optimizes for encoding co-occurrence statistics
- Final embedding = w + w_tilde (sum of word and context vectors)

## FastText

### Key Innovation
- Represents each word as a bag of character n-grams (typically 3-6)
- Word "where" with n=3: `<wh`, `whe`, `her`, `ere`, `re>`
- Word vector = sum of all its n-gram vectors

### Advantages Over Word2Vec
| Feature | Word2Vec | FastText |
|---------|----------|----------|
| OOV handling | No (returns UNK) | Yes (compose from n-grams) |
| Morphology | Ignores | Captures (un+happy, happi+ness) |
| Rare words | Poor representations | Better (shared n-grams help) |
| Typos | Fails | Partially robust |
| Training speed | Faster | Slower (more parameters) |

## Contextual Embeddings (ELMo/BERT)

### ELMo
- Bidirectional LSTM language model (forward + backward)
- Word representation = weighted sum of all layer outputs
- Each layer captures different linguistic information (lower = syntax, upper = semantics)
- Task-specific learned weights combine layers

### BERT
- Transformer encoder, pretrained with Masked Language Model (15% tokens masked) + Next Sentence Prediction
- WordPiece tokenization handles any input
- Fine-tuning: add task-specific head on top of [CLS] token or token outputs
- Produces different embeddings for same word in different contexts

## Evaluation

### Intrinsic Evaluation
- **Word similarity**: Spearman correlation with human judgments (SimLex-999, WS-353)
- **Word analogy**: "king - man + woman = queen" accuracy
- **Categorization**: clustering words into semantic categories

### Extrinsic Evaluation
- Performance on downstream tasks (NER, sentiment, translation)
- This is what ultimately matters

## Practical Recommendations

| Scenario | Recommendation |
|----------|---------------|
| Large English corpus, general use | GloVe 300d or Word2Vec 300d |
| Morphologically rich language | FastText |
| Limited data, specific domain | Pretrained + fine-tune on domain data |
| Context matters (polysemy) | BERT or contextual embeddings |
| Speed-critical inference | Static embeddings (Word2Vec/GloVe) |
| Custom vocabulary (code, medical) | Train FastText on domain corpus |
