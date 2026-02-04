# Part 2: Deep Learning Core

> From neural networks to transformers—the architecture powering modern AI

---

## Neural Networks from Scratch (Conceptually)

### The Neuron

**Biological inspiration:**

- Neurons receive signals (dendrites)
- Process them (cell body)
- Send output (axon)

**Artificial neuron:**

```
Inputs: x1, x2, x3
Weights: w1, w2, w3
Bias: b

Output = activation(w1*x1 + w2*x2 + w3*x3 + b)
```

**Key insight:** A neuron is a **linear function** + **non-linear activation**.

---

### Multi-Layer Perceptron (MLP)

**Architecture:**

```
Input Layer → Hidden Layer(s) → Output Layer
```

**Why multiple layers?**

- 1 layer: Can only learn linear relationships
- 2+ layers: Can approximate **any** function (universal approximation theorem)

**Forward pass:**

```python
# Pseudocode
h1 = activation(W1 @ input + b1)  # Hidden layer 1
h2 = activation(W2 @ h1 + b2)     # Hidden layer 2
output = W3 @ h2 + b3             # Output layer
```

**Training:**

1. Forward pass: Compute output
2. Compute loss: Compare to target
3. Backward pass: Compute gradients (backpropagation)
4. Update weights: `w = w - lr * gradient`

---

## Activation Functions

**Why needed?** Without activation, stacking layers = one big linear function (useless).

---

### ReLU (Rectified Linear Unit)

**Formula:** `f(x) = max(0, x)`

**Properties:**

- Simple: Zeros out negatives
- Fast to compute
- Doesn't saturate for positive values
- **Dying ReLU problem:** Neurons can get stuck at 0

**Most common activation** for hidden layers (2012-2020).

```python
def relu(x):
    return max(0, x)
```

---

### GELU (Gaussian Error Linear Unit)

**Formula:** `f(x) = x * Φ(x)` where Φ is Gaussian CDF

**Intuition:** Smooth approximation of ReLU

**Why better than ReLU:**

- Smooth (differentiable everywhere)
- Non-monotonic (can be negative for small inputs)
- Better gradient flow

**Used in:** Transformers (BERT, GPT-2+)

---

### SiLU / Swish

**Formula:** `f(x) = x * sigmoid(x)`

**Properties:**

- Smooth, non-monotonic
- Self-gated
- Better performance than ReLU in some cases

**Used in:** Some modern architectures (EfficientNet, some LLMs)

---

### Sigmoid and Tanh (Legacy)

**Sigmoid:** `f(x) = 1 / (1 + e^(-x))` (outputs 0 to 1)
**Tanh:** `f(x) = (e^x - e^(-x)) / (e^x + e^(-x))` (outputs -1 to 1)

**Problems:**

- **Vanishing gradients:** Gradients very small for large/small inputs
- Slow training in deep networks

**Still used for:**

- Output layer (binary classification: sigmoid)
- Gates in LSTMs

---

## Normalization

**Problem:** Internal covariate shift—activations change distribution during training, slowing convergence.

---

### Batch Normalization (BatchNorm)

**What:** Normalize activations across batch dimension.

**Formula:**

```
mean = mean(batch)
variance = variance(batch)
x_normalized = (x - mean) / sqrt(variance + ε)
output = γ * x_normalized + β  # Learnable scale and shift
```

**Properties:**

- Speeds up training
- Allows higher learning rates
- Acts as regularization

**Used in:** CNNs

**Limitation:** Breaks with small batch sizes, doesn't work well for sequences (RNNs, Transformers).

---

### Layer Normalization (LayerNorm)

**What:** Normalize across feature dimension (not batch).

**Formula:**

```
mean = mean(features)
variance = variance(features)
x_normalized = (x - mean) / sqrt(variance + ε)
output = γ * x_normalized + β
```

**Why better for transformers:**

- Batch size independent
- Works with sequences
- Stable for variable-length inputs

**Used in:** **All transformers** (BERT, GPT, LLaMA, etc.)

```python
# PyTorch
layer_norm = nn.LayerNorm(d_model)
```

---

### RMSNorm (Root Mean Square Normalization)

**What:** Simplified LayerNorm (skips mean centering).

**Formula:**

```
rms = sqrt(mean(x^2))
output = x / rms * γ
```

**Advantages:**

- Faster (no mean computation)
- Slightly more stable
- Better performance in some cases

**Used in:** LLaMA, Mistral, modern LLMs

---

## Convolutional Neural Networks (CNNs)

### Why CNNs?

**Problem:** Images are large (e.g., 224×224×3 = 150K pixels). Fully connected layers would have millions of parameters.

**Solution:** **Convolution**—slide a small filter over the image.

---

### Convolution Operation

**Filter (kernel):**

```
3×3 filter:
[1  0 -1]
[1  0 -1]
[1  0 -1]
```

**Slide over image:**

```
Image:        Filter:       Output:
[1 2 3 4]     [1 0]         [...]
[5 6 7 8]  →  [0 1]      →
[9 0 1 2]
```

**Key properties:**

- **Local connectivity:** Each output depends on small region
- **Weight sharing:** Same filter across entire image (fewer parameters)
- **Translation invariance:** Detects features regardless of position

---

### CNN Architecture

**Typical structure:**

```
Input → Conv → ReLU → Pool → Conv → ReLU → Pool → Flatten → FC → Output
```

**Layers:**

1. **Conv layer:** Extract features (edges, textures, patterns)
2. **Activation:** Non-linearity (ReLU)
3. **Pooling:** Downsample (MaxPool, AvgPool)
4. **Fully connected:** Final classification

**Famous architectures:** AlexNet, VGG, ResNet, EfficientNet

---

### CNNs in GenAI

**Vision models:**

- **DALL-E:** Uses CNN-like components in VAE
- **Stable Diffusion:** U-Net (CNN-based) for diffusion
- **Vision Transformers (ViT):** Replace CNNs with transformers

**Modern trend:** Transformers replacing CNNs for vision tasks.

---

## Recurrent Neural Networks (RNNs)

### Why RNNs?

**Problem:** Process sequences (text, time-series). Need memory of previous inputs.

**Solution:** **Recurrence**—feed output back as input for next step.

---

### RNN Architecture

**Formula:**

```
h_t = tanh(W_h * h_{t-1} + W_x * x_t + b)
```

**Diagram:**

```
h_0 → [RNN] → h_1 → [RNN] → h_2 → [RNN] → h_3
       ↑            ↑            ↑
       x_1          x_2          x_3
```

**Key idea:** Hidden state `h_t` carries information from previous timesteps.

---

### LSTM (Long Short-Term Memory)

**Why:** Vanilla RNNs suffer from vanishing gradients (can't learn long-range dependencies).

**Solution:** **Gates** to control information flow.

**Gates:**

1. **Forget gate:** What to forget from cell state
2. **Input gate:** What new info to add
3. **Output gate:** What to output

**Formula (simplified):**

```
f_t = σ(W_f @ [h_{t-1}, x_t])  # Forget gate
i_t = σ(W_i @ [h_{t-1}, x_t])  # Input gate
o_t = σ(W_o @ [h_{t-1}, x_t])  # Output gate
C_t = f_t * C_{t-1} + i_t * tanh(W_c @ [h_{t-1}, x_t])  # Cell state
h_t = o_t * tanh(C_t)  # Hidden state
```

**Used in:** Older NLP systems (pre-transformer era), some time-series tasks.

---

### GRU (Gated Recurrent Unit)

**What:** Simplified LSTM (fewer gates).

**Advantages:**

- Faster than LSTM
- Fewer parameters
- Often similar performance

**Gates:**

1. **Update gate:** How much to update hidden state
2. **Reset gate:** How much past info to forget

---

### Why RNNs Failed at Scale

**Problems:**

1. **Sequential processing:** Can't parallelize (slow for long sequences)
2. **Vanishing gradients:** Even LSTMs struggle with very long sequences
3. **Limited context:** Hard to capture dependencies 100+ tokens apart

**Solution:** **Transformers** (2017)—parallel processing, attention mechanism.

**RNNs today:** Mostly obsolete for NLP. Transformers dominate.

---

## Attention Mechanism

### The Problem

**RNN bottleneck:** Entire input sequence compressed into fixed-size hidden state.

**Example:**

```
Input: "The cat sat on the mat"
RNN: Compresses to h_final (fixed size)
Decoder: Generate translation from h_final
```

**Issue:** Long sentences lose information.

---

### Attention Solution

**Key insight:** Let decoder **attend** to all encoder states, not just the last one.

**Formula:**

```
Attention(Q, K, V) = softmax(score(Q, K)) * V
```

**Steps:**

1. **Compute scores:** How relevant is each encoder state?
2. **Softmax:** Convert to probabilities (attention weights)
3. **Weighted sum:** Combine values using attention weights

**Visual:**

```
Query (decoder state):  "Where is the subject?"
Keys (encoder states):  ["The", "cat", "sat", "on", "the", "mat"]
Scores:                 [0.1,  0.7,  0.05, 0.05, 0.05, 0.05]
Output:                 Focus on "cat"
```

---

### Scaled Dot-Product Attention

**Formula:**
$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

**Components:**

- **Q (Query):** What am I looking for?
- **K (Key):** What do I have?
- **V (Value):** The actual content

**Scaling factor:** $\frac{1}{\sqrt{d_k}}$ prevents softmax saturation.

**Why dot product?**

- Fast (matrix multiplication)
- Measures similarity

---

## Transformer Architecture

**Paper:** "Attention Is All You Need" (2017)

**Paradigm shift:** Replace recurrence with **attention**.

---

### High-Level Architecture

```
Input Embedding
    ↓
Positional Encoding
    ↓
[Encoder Blocks] ×N
    ↓
[Decoder Blocks] ×N
    ↓
Output Projection
```

---

### Encoder Block

```
Input
  ↓
Multi-Head Self-Attention
  ↓
Add & Norm (Residual + LayerNorm)
  ↓
Feed-Forward Network
  ↓
Add & Norm
  ↓
Output
```

**Components:**

1. **Multi-head self-attention:** Attend to all positions in input
2. **Residual connection:** `output = layer(input) + input`
3. **Layer normalization:** Stabilize training
4. **Feed-forward network:** 2-layer MLP with GELU

**Repeated N times** (e.g., N=12 for BERT-base, N=96 for GPT-3).

---

### Decoder Block

```
Input
  ↓
Masked Multi-Head Self-Attention
  ↓
Add & Norm
  ↓
Multi-Head Cross-Attention (attend to encoder)
  ↓
Add & Norm
  ↓
Feed-Forward Network
  ↓
Add & Norm
  ↓
Output
```

**Key difference:** **Masked attention**—can only attend to previous tokens (for autoregressive generation).

---

### Multi-Head Attention

**Why multiple heads?**

- Different heads learn different patterns
- Head 1: Subject-verb agreement
- Head 2: Coreference resolution
- Head 3: Syntactic structure

**Formula:**

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) @ W_O

where head_i = Attention(Q @ W_Q^i, K @ W_K^i, V @ W_V^i)
```

**Parameters:**

- `h`: Number of heads (e.g., 12, 32, 64)
- `d_model`: Model dimension (e.g., 768, 1024, 4096)
- `d_k = d_model / h`: Per-head dimension

**Example (GPT-2):**

- `d_model = 768`
- `h = 12`
- `d_k = 64`

---

### Feed-Forward Network

**Formula:**

```
FFN(x) = GELU(x @ W1 + b1) @ W2 + b2
```

**Dimensions:**

```
Input:  (batch, seq_len, d_model)
W1:     (d_model, d_ff)         # Typically d_ff = 4 * d_model
W2:     (d_ff, d_model)
Output: (batch, seq_len, d_model)
```

**Key insight:** FFN is applied **independently** to each position (position-wise).

---

### Residual Connections

**Formula:**

```
output = LayerNorm(input + Sublayer(input))
```

**Why critical:**

- Prevents vanishing gradients
- Allows very deep networks (100+ layers)
- Enables gradient flow

**Without residuals:** Deep networks don't train well.

---

### Layer Normalization

**Applied after each sublayer:**

```
x = x + Attention(x)
x = LayerNorm(x)

x = x + FFN(x)
x = LayerNorm(x)
```

**Two variants:**

- **Post-LN:** Original transformer (above)
- **Pre-LN:** Modern LLMs (more stable)

```
# Pre-LN (used in GPT, LLaMA)
x = x + Attention(LayerNorm(x))
x = x + FFN(LayerNorm(x))
```

---

## Self-Attention vs Cross-Attention

### Self-Attention

**What:** Each token attends to all tokens in the **same** sequence.

**Used in:**

- Encoder (BERT)
- Decoder (GPT)

**Example:**

```
Sentence: "The cat sat"
Token "sat" attends to: ["The", "cat", "sat"]
```

**Enables:** Capturing relationships within a sequence.

---

### Cross-Attention

**What:** Decoder attends to **encoder** outputs.

**Used in:**

- Encoder-decoder models (T5, BART)
- Vision-language models (CLIP, Flamingo)

**Example:**

```
Encoder: "Le chat est assis" (French)
Decoder generating: "The cat"
Token "cat" attends to: ["Le", "chat", "est", "assis"]
```

**Enables:** Conditioning on external information.

---

## Positional Encodings

**Problem:** Attention is **permutation-invariant**—doesn't know token order.

**Example:**

```
"dog bites man" and "man bites dog"
Without positional encoding: Same representation!
```

**Solution:** Add positional information.

---

### Absolute Positional Encoding (Original Transformer)

**Formula (sinusoidal):**

```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

**Properties:**

- Fixed (not learned)
- Unique for each position
- Generalizes to longer sequences

---

### Learned Positional Embeddings

**What:** Learn position embeddings during training (like token embeddings).

**Used in:** BERT, GPT-2

**Limitation:** Fixed max sequence length (e.g., 512, 1024).

---

### Rotary Position Embedding (RoPE)

**What:** Encode position by **rotating** query and key vectors.

**Why better:**

- Relative position encoding (generalizes better)
- Infinite sequence length (theoretically)
- Better long-context performance

**Used in:** LLaMA, GPT-NeoX, Mistral, modern LLMs

**Extension:** RoPE scaling allows extending context window (e.g., 4K → 32K tokens).

---

### ALiBi (Attention with Linear Biases)

**What:** Add **bias** to attention scores based on distance.

**Formula:**

```
Attention(Q, K) = softmax(QK^T + bias) @ V
bias[i,j] = -m * |i - j|  # Linear penalty for distance
```

**Why better:**

- No explicit position embeddings
- Extrapolates to longer sequences easily
- Saves parameters

**Used in:** BLOOM, some research models

---

## Encoder vs Decoder vs Encoder-Decoder

### Encoder-Only (BERT-style)

**Architecture:** Stacked encoder blocks

**Training:** Masked language modeling (predict masked tokens)

**Use case:**

- Classification (sentiment, NER)
- Embedding generation
- Understanding tasks

**Examples:** BERT, RoBERTa, ALBERT

**Attention:** Bidirectional (can see all tokens)

---

### Decoder-Only (GPT-style)

**Architecture:** Stacked decoder blocks (with masked attention)

**Training:** Causal language modeling (predict next token)

**Use case:**

- Text generation
- Code generation
- Chat, assistants

**Examples:** GPT-2, GPT-3, GPT-4, LLaMA, Mistral

**Attention:** Causal (can only see previous tokens)

**Why dominant:** Scales better, simpler architecture, generation is key use case.

---

### Encoder-Decoder (T5-style)

**Architecture:** Encoder + Decoder with cross-attention

**Training:** Seq2seq tasks (translate, summarize)

**Use case:**

- Translation
- Summarization
- Question answering (input → output)

**Examples:** T5, BART, mT5

**Attention:** Encoder (bidirectional) + Decoder (causal + cross-attention)

**When to use:** When you have clear input-output pairs (not open-ended generation).

---

## Why Transformers Won

### vs RNNs

| RNN                 | Transformer                 |
| ------------------- | --------------------------- |
| Sequential (slow)   | Parallel (fast)             |
| Limited context     | Full context via attention  |
| Vanishing gradients | Stable training (residuals) |
| Complex (gates)     | Simple (attention + FFN)    |

---

### Computational Trade-offs

**Attention complexity:** $O(n^2 \cdot d)$ where $n$ = sequence length, $d$ = dimension

**Problem:** Quadratic scaling with sequence length.

**Implications:**

- 1K tokens: Manageable
- 10K tokens: Expensive
- 100K tokens: Very expensive

**Solutions:**

- Sparse attention (only attend to subset)
- Linear attention (approximations)
- Alternative architectures (Mamba, RWKV)

---

## Modern Architectures (2024-2025)

### Mamba (State Space Models)

**Key idea:** Replace attention with **selective state space models**.

**Advantages:**

- **Linear complexity:** $O(n \cdot d)$ instead of $O(n^2 \cdot d)$
- Fast inference
- Handles very long sequences

**Trade-offs:**

- Newer (less proven)
- Different training dynamics
- Some tasks still favor attention

**Used in:** Mamba, Mamba-2 (research, early adoption)

---

### RWKV (Receptance Weighted Key Value)

**Key idea:** RNN-like architecture with transformer-like performance.

**Advantages:**

- Linear complexity
- Constant memory during inference
- Fast generation

**Trade-offs:**

- Less expressive than full attention
- Training tricks needed

**Used in:** RWKV models (open-source community)

---

### RetNet (Retentive Networks)

**Key idea:** Combine benefits of transformers and RNNs.

**Advantages:**

- Training parallelism (like transformer)
- Inference efficiency (like RNN)
- Linear complexity option

**Status:** Research (Microsoft, 2023)

---

### Practical Reality (2025)

**For production:**

- **Transformers still dominate** (GPT, LLaMA, Mistral)
- New architectures are promising but unproven at scale
- Most companies stick with transformer-based models

**Watch:** Mamba gaining traction for long-context tasks.

---

## Transformer Mathematics: Complete Walkthrough

### Step-by-Step Attention Computation

**Let's compute attention for a simple example:**

**Input sentence:** "I love AI"

**Step 1: Tokenization & Embedding**

```
Tokens: ["I", "love", "AI"]
Token IDs: [100, 523, 892]

Embedding lookup (assume d_model=4 for simplicity):
x_0 = [0.1, 0.2, 0.3, 0.4]  # "I"
x_1 = [0.5, 0.6, 0.7, 0.8]  # "love"
x_2 = [0.9, 1.0, 1.1, 1.2]  # "AI"

X = [[0.1, 0.2, 0.3, 0.4],
     [0.5, 0.6, 0.7, 0.8],
     [0.9, 1.0, 1.1, 1.2]]  # Shape: (3, 4)
```

**Step 2: Create Q, K, V matrices**

```python
# Simplified weight matrices (normally initialized randomly)
W_Q = [[1, 0, 1, 0],
       [0, 1, 0, 1],
       [1, 1, 0, 0],
       [0, 0, 1, 1]]  # Shape: (4, 4)

W_K = W_Q  # For simplicity, same weights
W_V = W_Q

# Compute Q, K, V
Q = X @ W_Q  # Shape: (3, 4)
K = X @ W_K  # Shape: (3, 4)
V = X @ W_V  # Shape: (3, 4)

# Example Q matrix:
Q = [[0.4, 0.6, 0.4, 0.6],
     [1.2, 1.4, 1.2, 1.4],
     [2.0, 2.2, 2.0, 2.2]]
```

**Step 3: Compute attention scores**

```python
# Scaled dot-product
d_k = 4
scores = (Q @ K.T) / sqrt(d_k)  # Shape: (3, 3)

# scores[i,j] = how much token i attends to token j
scores = [[0.65, 1.95, 3.25],    # "I" attending to ["I", "love", "AI"]
          [1.95, 5.85, 9.75],    # "love" attending to ["I", "love", "AI"]
          [3.25, 9.75, 16.25]]   # "AI" attending to ["I", "love", "AI"]
```

**Step 4: Apply softmax**

```python
import numpy as np

attention_weights = np.softmax(scores, axis=-1)

# attention_weights (after softmax):
[[0.01, 0.11, 0.88],   # "I" mostly attends to "AI"
 [0.00, 0.01, 0.99],   # "love" mostly attends to "AI"
 [0.00, 0.00, 1.00]]   # "AI" only attends to itself
```

**Step 5: Weighted sum of values**

```python
output = attention_weights @ V  # Shape: (3, 4)

# Each output token is weighted combination of all V vectors
output[0] = 0.01*V[0] + 0.11*V[1] + 0.88*V[2]
```

**Complete code:**

```python
import torch
import torch.nn.functional as F

def attention(Q, K, V, mask=None):
    """
    Args:
        Q: Queries (batch, seq_len, d_k)
        K: Keys (batch, seq_len, d_k)
        V: Values (batch, seq_len, d_v)
        mask: Optional mask (batch, seq_len, seq_len)
    Returns:
        output: (batch, seq_len, d_v)
        attention_weights: (batch, seq_len, seq_len)
    """
    d_k = Q.size(-1)

    # Compute scores: (batch, seq_len, seq_len)
    scores = torch.matmul(Q, K.transpose(-2, -1)) / torch.sqrt(torch.tensor(d_k, dtype=torch.float32))

    # Apply mask (for causal/padding)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, -1e9)

    # Softmax
    attention_weights = F.softmax(scores, dim=-1)

    # Weighted sum
    output = torch.matmul(attention_weights, V)

    return output, attention_weights

# Example usage
batch_size, seq_len, d_model = 2, 5, 64
Q = torch.randn(batch_size, seq_len, d_model)
K = torch.randn(batch_size, seq_len, d_model)
V = torch.randn(batch_size, seq_len, d_model)

output, weights = attention(Q, K, V)
print(f"Output shape: {output.shape}")  # (2, 5, 64)
print(f"Attention weights shape: {weights.shape}")  # (2, 5, 5)
```

---

### Multi-Head Attention Implementation

**Mathematical formulation:**

$$
\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, ..., \text{head}_h)W^O
$$

where:

$$
\text{head}_i = \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)
$$

**Complete PyTorch implementation:**

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.1):
        """
        Args:
            d_model: Model dimension (e.g., 512, 768)
            num_heads: Number of attention heads (e.g., 8, 12)
            dropout: Dropout probability
        """
        super().__init__()
        assert d_model % num_heads == 0, "d_model must be divisible by num_heads"

        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads  # Dimension per head

        # Linear projections
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)

        self.dropout = nn.Dropout(dropout)

    def split_heads(self, x):
        """
        Split last dimension into (num_heads, d_k)
        Args:
            x: (batch, seq_len, d_model)
        Returns:
            (batch, num_heads, seq_len, d_k)
        """
        batch_size, seq_len, d_model = x.size()
        x = x.view(batch_size, seq_len, self.num_heads, self.d_k)
        return x.transpose(1, 2)  # (batch, num_heads, seq_len, d_k)

    def combine_heads(self, x):
        """
        Inverse of split_heads
        Args:
            x: (batch, num_heads, seq_len, d_k)
        Returns:
            (batch, seq_len, d_model)
        """
        batch_size, num_heads, seq_len, d_k = x.size()
        x = x.transpose(1, 2)  # (batch, seq_len, num_heads, d_k)
        return x.contiguous().view(batch_size, seq_len, self.d_model)

    def forward(self, query, key, value, mask=None):
        """
        Args:
            query: (batch, seq_len_q, d_model)
            key: (batch, seq_len_k, d_model)
            value: (batch, seq_len_v, d_model)
            mask: (batch, 1, seq_len_q, seq_len_k) or (batch, 1, 1, seq_len_k)
        Returns:
            output: (batch, seq_len_q, d_model)
            attention_weights: (batch, num_heads, seq_len_q, seq_len_k)
        """
        batch_size = query.size(0)

        # Linear projections
        Q = self.W_q(query)  # (batch, seq_len_q, d_model)
        K = self.W_k(key)    # (batch, seq_len_k, d_model)
        V = self.W_v(value)  # (batch, seq_len_v, d_model)

        # Split into multiple heads
        Q = self.split_heads(Q)  # (batch, num_heads, seq_len_q, d_k)
        K = self.split_heads(K)  # (batch, num_heads, seq_len_k, d_k)
        V = self.split_heads(V)  # (batch, num_heads, seq_len_v, d_k)

        # Scaled dot-product attention
        # scores: (batch, num_heads, seq_len_q, seq_len_k)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)

        # Apply mask
        if mask is not None:
            scores = scores.masked_fill(mask == 0, -1e9)

        # Softmax and dropout
        attention_weights = F.softmax(scores, dim=-1)
        attention_weights = self.dropout(attention_weights)

        # Apply attention to values
        # context: (batch, num_heads, seq_len_q, d_k)
        context = torch.matmul(attention_weights, V)

        # Combine heads
        context = self.combine_heads(context)  # (batch, seq_len_q, d_model)

        # Final linear projection
        output = self.W_o(context)

        return output, attention_weights

# Example usage
d_model = 512
num_heads = 8
batch_size = 2
seq_len = 10

mha = MultiHeadAttention(d_model, num_heads)
x = torch.randn(batch_size, seq_len, d_model)

output, weights = mha(x, x, x)  # Self-attention
print(f"Output shape: {output.shape}")  # (2, 10, 512)
print(f"Attention weights shape: {weights.shape}")  # (2, 8, 10, 10)
```

---

### KV Cache: Fast Autoregressive Generation

**Problem:** Generating tokens is slow because we recompute attention for all previous tokens every time.

**Without KV cache:**

```python
# Generate token by token (inefficient)
for i in range(max_length):
    # Recompute attention over ALL tokens (0 to i)
    logits = model(input_ids[:, :i+1])  # Expensive!
    next_token = logits[:, -1].argmax()
    input_ids = torch.cat([input_ids, next_token.unsqueeze(-1)], dim=1)
```

**Time complexity:** O(n²) for generating n tokens

**With KV cache:**

```python
# Cache Key and Value matrices
past_key_values = None

for i in range(max_length):
    # Only process NEW token
    logits, past_key_values = model(
        input_ids[:, i:i+1],      # Only current token!
        past_key_values=past_key_values  # Cached K,V from previous steps
    )
    next_token = logits[:, -1].argmax()
    input_ids = torch.cat([input_ids, next_token.unsqueeze(-1)], dim=1)
```

**Time complexity:** O(n) for generating n tokens

**Implementation:**

```python
class MultiHeadAttentionWithKVCache(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads

        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)

    def forward(self, query, key, value, past_kv=None, use_cache=False):
        """
        Args:
            query: (batch, seq_len_q, d_model)
            key: (batch, seq_len_k, d_model)
            value: (batch, seq_len_v, d_model)
            past_kv: Tuple of (past_key, past_value) or None
                     past_key: (batch, num_heads, past_seq_len, d_k)
                     past_value: (batch, num_heads, past_seq_len, d_k)
            use_cache: Whether to return K,V for caching
        Returns:
            output: (batch, seq_len_q, d_model)
            present_kv: Tuple of (K, V) if use_cache else None
        """
        # Project to Q, K, V
        Q = self.W_q(query)
        K = self.W_k(key)
        V = self.W_v(value)

        # Split heads
        Q = self.split_heads(Q)  # (batch, num_heads, seq_len_q, d_k)
        K = self.split_heads(K)  # (batch, num_heads, seq_len_k, d_k)
        V = self.split_heads(V)  # (batch, num_heads, seq_len_v, d_k)

        # Concatenate with past K,V if provided
        if past_kv is not None:
            past_key, past_value = past_kv
            K = torch.cat([past_key, K], dim=2)  # Concatenate along sequence dimension
            V = torch.cat([past_value, V], dim=2)

        # Compute attention
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        attention_weights = F.softmax(scores, dim=-1)
        context = torch.matmul(attention_weights, V)

        # Combine heads
        context = self.combine_heads(context)
        output = self.W_o(context)

        # Return current K,V for next iteration
        present_kv = (K, V) if use_cache else None

        return output, present_kv

    def split_heads(self, x):
        batch_size, seq_len, _ = x.size()
        x = x.view(batch_size, seq_len, self.num_heads, self.d_k)
        return x.transpose(1, 2)

    def combine_heads(self, x):
        batch_size, _, seq_len, _ = x.size()
        x = x.transpose(1, 2).contiguous()
        return x.view(batch_size, seq_len, self.d_model)

# Usage example: Efficient generation
model = MultiHeadAttentionWithKVCache(d_model=512, num_heads=8)

input_ids = torch.randint(0, 1000, (1, 1))  # Start with one token
past_kv = None

for i in range(20):  # Generate 20 tokens
    # Only process the LAST token (not all previous)
    x = embedding(input_ids[:, -1:])  # (1, 1, 512)

    # Forward pass with KV cache
    output, past_kv = model(x, x, x, past_kv=past_kv, use_cache=True)

    # Sample next token
    logits = output_head(output)  # (1, 1, vocab_size)
    next_token = logits.argmax(dim=-1)

    # Append to sequence
    input_ids = torch.cat([input_ids, next_token], dim=1)

print(f"Generated {input_ids.size(1)} tokens efficiently!")
```

**Memory savings:**

```
Without cache: Store all activations for all tokens
  └─ Memory: O(n² * d_model) for n tokens

With cache: Only store K,V matrices (reuse them)
  └─ Memory: O(n * d_model * num_layers) for n tokens
  └─ Speedup: ~10-50x for long sequences
```

**Real-world impact:**

- GPT-4: Can generate 100 tokens/sec with KV cache vs 2-5 tokens/sec without
- Production inference servers (vLLM, TGI) heavily rely on KV cache optimization

---

### Causal Masking for Decoder

**Why needed:** Prevent decoder from "cheating" by looking at future tokens.

**Causal mask:**

```python
def create_causal_mask(seq_len):
    """
    Create lower-triangular mask
    Returns: (seq_len, seq_len) boolean tensor
    """
    mask = torch.tril(torch.ones(seq_len, seq_len))
    return mask  # [[1, 0, 0],
                 #  [1, 1, 0],
                 #  [1, 1, 1]]

# In attention computation:
scores = scores.masked_fill(mask == 0, -1e9)
# After softmax, masked positions become 0
```

**Visual representation:**

```
Token 0 (position 0): Can only attend to [token 0]
Token 1 (position 1): Can only attend to [token 0, token 1]
Token 2 (position 2): Can only attend to [token 0, token 1, token 2]
...

Attention matrix (after masking):
       Q
     0 1 2
K  0 [✓ ✗ ✗]
   1 [✓ ✓ ✗]
   2 [✓ ✓ ✓]

✓ = Allowed to attend
✗ = Masked (cannot attend to future)
```

**Complete implementation:**

```python
def forward_with_causal_mask(self, x):
    """
    Args:
        x: (batch, seq_len, d_model)
    """
    batch_size, seq_len, _ = x.size()

    # Create causal mask
    causal_mask = torch.tril(torch.ones(seq_len, seq_len, device=x.device))
    causal_mask = causal_mask.unsqueeze(0).unsqueeze(0)  # (1, 1, seq_len, seq_len)

    # Self-attention with causal mask
    output, _ = self.attention(x, x, x, mask=causal_mask)

    return output
```

---

### Memory and Compute Optimization

**1. Flash Attention**

**Problem:** Standard attention materializes full attention matrix (seq_len × seq_len).

**Solution:** Compute attention in blocks without materializing full matrix.

**Memory:** O(n) instead of O(n²)

**Speed:** 2-4x faster

```python
# Standard attention: O(n²) memory
scores = Q @ K.T  # Materialize (seq_len, seq_len) matrix ❌

# Flash Attention: O(n) memory
# Compute in tiles, accumulate results on-the-fly ✓
from flash_attn import flash_attn_func

output = flash_attn_func(Q, K, V)  # Never stores full attention matrix
```

**2. Gradient Checkpointing**

**Trade compute for memory:**

```python
import torch.utils.checkpoint as checkpoint

class TransformerLayer(nn.Module):
    def forward(self, x):
        # Without checkpointing: Store all activations (high memory)
        # x = self.attention(x)
        # x = self.ffn(x)

        # With checkpointing: Recompute during backward (low memory)
        x = checkpoint.checkpoint(self.attention, x, use_reentrant=False)
        x = checkpoint.checkpoint(self.ffn, x, use_reentrant=False)
        return x
```

**Impact:** Train 2-3x larger models on same GPU.

**3. Mixed Precision (FP16/BF16)**

```python
from torch.cuda.amp import autocast, GradScaler

model = TransformerModel()
optimizer = torch.optim.AdamW(model.parameters())
scaler = GradScaler()

for batch in dataloader:
    with autocast():  # Use FP16 for forward/backward
        loss = model(batch)

    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

**Benefits:**

- 2x faster training
- 2x less memory
- **BF16 preferred for LLMs** (better numerical stability)

---

### Transformer Block: Complete Implementation

```python
class TransformerBlock(nn.Module):
    """
    Complete transformer encoder/decoder block
    """
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1, is_decoder=False):
        super().__init__()

        self.is_decoder = is_decoder

        # Self-attention
        self.self_attn = MultiHeadAttention(d_model, num_heads, dropout)
        self.norm1 = nn.LayerNorm(d_model)
        self.dropout1 = nn.Dropout(dropout)

        # Cross-attention (decoder only)
        if is_decoder:
            self.cross_attn = MultiHeadAttention(d_model, num_heads, dropout)
            self.norm2 = nn.LayerNorm(d_model)
            self.dropout2 = nn.Dropout(dropout)

        # Feed-forward network
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.GELU(),
            nn.Dropout(dropout),
            nn.Linear(d_ff, d_model),
        )
        self.norm3 = nn.LayerNorm(d_model)
        self.dropout3 = nn.Dropout(dropout)

    def forward(self, x, encoder_output=None, self_attn_mask=None, cross_attn_mask=None):
        """
        Args:
            x: (batch, seq_len, d_model)
            encoder_output: (batch, src_len, d_model) - for decoder cross-attention
            self_attn_mask: Mask for self-attention
            cross_attn_mask: Mask for cross-attention
        """
        # Self-attention with residual + norm (Pre-LN style)
        residual = x
        x = self.norm1(x)
        x, _ = self.self_attn(x, x, x, mask=self_attn_mask)
        x = self.dropout1(x)
        x = residual + x

        # Cross-attention (decoder only)
        if self.is_decoder and encoder_output is not None:
            residual = x
            x = self.norm2(x)
            x, _ = self.cross_attn(x, encoder_output, encoder_output, mask=cross_attn_mask)
            x = self.dropout2(x)
            x = residual + x

        # Feed-forward with residual + norm
        residual = x
        x = self.norm3(x)
        x = self.ffn(x)
        x = self.dropout3(x)
        x = residual + x

        return x

# Example: Build a 12-layer transformer
class Transformer(nn.Module):
    def __init__(self, vocab_size=50000, d_model=768, num_heads=12,
                 num_layers=12, d_ff=3072, max_seq_len=2048, dropout=0.1):
        super().__init__()

        self.embedding = nn.Embedding(vocab_size, d_model)
        self.pos_encoding = nn.Embedding(max_seq_len, d_model)

        self.layers = nn.ModuleList([
            TransformerBlock(d_model, num_heads, d_ff, dropout, is_decoder=True)
            for _ in range(num_layers)
        ])

        self.ln_f = nn.LayerNorm(d_model)
        self.lm_head = nn.Linear(d_model, vocab_size, bias=False)

        # Weight tying (share embedding and output projection)
        self.lm_head.weight = self.embedding.weight

    def forward(self, input_ids, attention_mask=None):
        batch_size, seq_len = input_ids.size()

        # Token + positional embeddings
        positions = torch.arange(seq_len, device=input_ids.device).unsqueeze(0)
        x = self.embedding(input_ids) + self.pos_encoding(positions)

        # Create causal mask
        causal_mask = torch.tril(torch.ones(seq_len, seq_len, device=x.device))
        causal_mask = causal_mask.unsqueeze(0).unsqueeze(0)

        # Stack transformer blocks
        for layer in self.layers:
            x = layer(x, self_attn_mask=causal_mask)

        # Final layer norm and projection
        x = self.ln_f(x)
        logits = self.lm_head(x)

        return logits

# Instantiate GPT-2 sized model
model = Transformer(
    vocab_size=50257,
    d_model=768,
    num_heads=12,
    num_layers=12,
    d_ff=3072,
    max_seq_len=1024
)

print(f"Parameters: {sum(p.numel() for p in model.parameters()) / 1e6:.1f}M")
# Output: Parameters: 124.4M (same as GPT-2)
```

---

## Real-World Use Cases

### Encoder models (BERT):

- Document classification
- Named entity recognition
- Embedding generation for RAG
- Sentence similarity

### Decoder models (GPT):

- **Chatbots** (most common use case)
- Code generation
- Content creation
- Agents, tool use

### Encoder-decoder models (T5):

- Translation
- Summarization (input document → summary)
- Grammar correction

**Trend:** Decoder-only models taking over (can do everything with prompting).

---

## Interview Questions

**Junior:**

- Explain transformer architecture at a high level
- What's the difference between self-attention and cross-attention?
- Why do we need positional encodings?
- What's the difference between BERT and GPT?

**Mid-level:**

- Walk through the attention mechanism mathematically
- Explain multi-head attention and why it's useful
- What are residual connections and why do they matter?
- Compare encoder-only vs decoder-only architectures

**Senior:**

- Explain the computational complexity of transformers and how to address it
- Compare different positional encoding schemes (absolute, RoPE, ALiBi)
- Design a custom transformer variant for a specific use case
- Discuss trade-offs between transformer and alternative architectures (Mamba, RWKV)

---

## Key Takeaways

1. **Transformers = Attention + FFN + Residuals + LayerNorm**
2. **Attention mechanism** is the key innovation (parallel processing, long-range dependencies)
3. **Decoder-only models (GPT-style)** dominate for generative tasks
4. **Positional encodings** are critical (RoPE is modern standard)
5. **O(n²) complexity** is the main limitation (Mamba/RWKV try to address this)
6. **Multi-head attention** learns diverse patterns
7. **Residual connections** enable deep networks
8. **RNNs/LSTMs** are mostly obsolete for NLP

---

**Next:** [Part 3: Generative AI Fundamentals](03_Generative_AI_Fundamentals.md)
