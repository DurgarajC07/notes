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
