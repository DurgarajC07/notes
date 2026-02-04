# Part 4: Large Language Models (LLMs)

> Deep dive into LLM architecture, training, and the models powering modern AI

---

## What is a Large Language Model?

**Definition:** Neural network trained on massive text corpora to predict next tokens, resulting in emergent capabilities (reasoning, coding, conversation).

**Key characteristics:**

- **Decoder-only transformers** (mostly)
- **Billions of parameters** (7B-600B+)
- **Trained on trillions of tokens**
- **General purpose** (one model, many tasks)

**Why "large" matters:**

- **Scaling laws:** Bigger models = better performance
- **Emergence:** Capabilities appear at scale (reasoning, few-shot learning)
- **Generalization:** Larger models generalize better

---

## Tokenization

### Why Tokenization?

**Problem:** Neural networks need numbers, not text.

**Bad solution:** Character-level (inefficient, long sequences)
**Good solution:** Subword tokenization (balance between words and characters)

---

### Byte Pair Encoding (BPE)

**Core idea:** Start with characters, iteratively merge most frequent pairs.

**Algorithm:**

```
1. Start with vocabulary of all characters
2. Find most frequent pair of tokens (e.g., "th" appears often)
3. Merge into new token "th"
4. Repeat until vocabulary reaches target size (e.g., 50K)
```

**Example:**

```
Text: "lower lower lower lowest"

Initial: [l, o, w, e, r, space, l, o, w, e, s, t]

Step 1: Merge "l" + "o" → "lo" (most frequent)
Step 2: Merge "lo" + "w" → "low"
Step 3: Merge "low" + "e" → "lowe"
Step 4: Merge "lowe" + "r" → "lower"

Result: ["lower", "lower", "lower", "lowe", "s", "t"]
```

**Used in:** GPT-2, GPT-3, Codex

---

### WordPiece

**Similar to BPE** but uses different scoring (likelihood-based).

**Used in:** BERT, DistilBERT

---

### SentencePiece

**Key difference:** Treats spaces as regular characters (no pre-tokenization).

**Advantages:**

- **Language-agnostic:** Works with languages without spaces (Chinese, Japanese)
- **Reversible:** Can perfectly reconstruct original text
- **Two modes:** BPE or Unigram

**Used in:** T5, LLaMA, Mistral, Gemini

---

### Unigram Language Model

**Core idea:** Keep tokens that maximize likelihood of training data.

**Algorithm:**

```
1. Start with large vocabulary
2. Iteratively remove tokens that hurt likelihood least
3. Stop at target vocabulary size
```

**Used in:** T5 (SentencePiece Unigram)

---

### Tokenization Trade-offs

**Vocabulary size:**

- **Small (30K):** Longer sequences, more computation
- **Large (100K):** Shorter sequences, bigger embedding layer, rare tokens undertrained

**Example (GPT-4):**

```
"Hello world!" → [15496, 1917, 0]  # 3 tokens

"Supercalifragilisticexpialidocious" → [19841, 5531, 333, ...]  # Many tokens (rare word)
```

**Key insight:** Common words = 1 token, rare words = many tokens.

---

### Handling Multilingual Text

**Challenge:** English-centric tokenizers waste tokens on other languages.

**Example:**

```
English: "Hello" → 1 token
Hindi: "नमस्ते" → 5-10 tokens (inefficient!)
```

**Solutions:**

- **Multilingual tokenizers:** LLaMA 2 (32K vocab includes more non-English)
- **Language-specific models:** Qwen (Chinese), Yi (Chinese), Aya (multilingual)

---

### Special Tokens

**Common special tokens:**

```
<|endoftext|>  # End of document (GPT)
<|im_start|>   # Start of message (ChatML format)
<|im_end|>     # End of message
<pad>          # Padding token
<unk>          # Unknown token
<s>, </s>      # Start/end of sequence (LLaMA)
```

**Why needed:**

- Separate documents during training
- Structure conversations
- Handle padding in batches

---

## Embeddings

### Token Embeddings

**What:** Map token IDs to dense vectors.

**Implementation:**

```python
# Vocabulary size = 50K, embedding dimension = 768
embedding_layer = nn.Embedding(50000, 768)

# Token ID 1234 → [0.2, -0.5, 0.8, ...]
```

**Learned during training:** Embeddings capture semantic meaning.

**Example:**

```
"king" → [0.5, -0.2, 0.8, ...]
"queen" → [0.6, -0.1, 0.7, ...]  # Similar to king
"table" → [-0.3, 0.4, -0.6, ...]  # Different
```

---

### Positional Encodings (Revisited)

**Why needed:** Transformers have no notion of position without explicit encoding.

**Modern approach (LLaMA, Mistral):** **RoPE (Rotary Position Embedding)**

**How RoPE works:**

1. Rotate query and key vectors based on position
2. Encodes relative position (distance between tokens matters)
3. Generalizes to longer sequences

**Advantages over learned positional embeddings:**

- No fixed maximum length
- Better extrapolation to longer contexts
- More efficient

---

## Training LLMs

### Pretraining Objective

**Goal:** Learn language patterns from massive unlabeled text.

**Causal Language Modeling (CLM):**

- **Task:** Predict next token given previous tokens
- **Formula:** Maximize $P(x_t | x_1, ..., x_{t-1})$
- **Loss:** Cross-entropy between predicted and actual next token

**Training loop:**

```python
for batch in dataset:
    # Forward pass
    logits = model(batch.input_ids)  # (batch, seq_len, vocab_size)

    # Shift targets (predict next token)
    targets = batch.input_ids[:, 1:]  # Remove first token
    logits = logits[:, :-1, :]         # Remove last prediction

    # Compute loss
    loss = cross_entropy(logits, targets)

    # Backward pass
    loss.backward()
    optimizer.step()
```

---

### Masked Language Modeling (MLM)

**Alternative (BERT-style):**

- Mask 15% of tokens
- Predict masked tokens (bidirectional context)

**Example:**

```
Input:  "The [MASK] sat on the mat"
Target: "cat"
```

**Not used for generative models** (decoder-only LLMs use CLM).

---

### Scaling Laws

**Chinchilla Scaling Laws (2022):**

**Key finding:** Models are undertrained. For optimal performance:
$$\text{Tokens} = 20 \times \text{Parameters}$$

**Example:**

- **GPT-3 (175B):** Trained on 300B tokens → **undertrained**
- **Optimal:** Should train on 3.5T tokens
- **LLaMA (65B):** Trained on 1.4T tokens → **better than GPT-3** despite being smaller

**Implications:**

- **Compute-optimal:** Balance model size and training data
- **Smaller models, more data:** More cost-effective
- **Most LLMs were undertrained** (pre-2022)

**Modern trend:** Train longer (LLaMA 2: 2T tokens, LLaMA 3: 15T tokens)

---

### Data Mixtures

**Training data sources:**

1. **Web crawl:** CommonCrawl (noisy, diverse)
2. **Books:** High-quality long-form text
3. **Code:** GitHub, StackOverflow (programming ability)
4. **Scientific papers:** ArXiv, PubMed (reasoning, knowledge)
5. **Conversations:** Reddit, forums (dialogue)
6. **Wikipedia:** Factual knowledge

**Data quality matters more than quantity:**

- **Filtering:** Remove duplicates, low-quality, toxic content
- **Deduplication:** Avoid memorization, overfitting
- **Data mixing:** Balance different domains

**Example (LLaMA):**

- 67% CommonCrawl
- 15% C4 (cleaned web)
- 4.5% GitHub
- 4.5% Wikipedia
- 4.5% Books
- 2.5% ArXiv
- 2.5% StackExchange

---

### Distributed Training

**Problem:** Models don't fit on single GPU.

**GPT-3 (175B parameters):**

- Each parameter: 2 bytes (FP16) or 4 bytes (FP32)
- Memory needed: 350GB-700GB (just for weights!)
- Plus: Gradients, optimizer states, activations → **Several TB**

**Solutions:**

---

#### Data Parallelism (DDP)

**Idea:** Replicate model on multiple GPUs, split data.

```
GPU 0: Model copy 1, batch 1
GPU 1: Model copy 1, batch 2
GPU 2: Model copy 1, batch 3
GPU 3: Model copy 1, batch 4

After forward/backward: Average gradients across GPUs
```

**Limitation:** Model must fit on single GPU.

---

#### Model Parallelism

**Idea:** Split model across GPUs.

**Tensor parallelism:**

- Split individual layers (e.g., split attention heads across GPUs)
- Used in: Megatron-LM

**Pipeline parallelism:**

- Split layers across GPUs (Layer 1-10 on GPU0, Layer 11-20 on GPU1, etc.)
- Process microbatches in pipeline

---

#### Fully Sharded Data Parallel (FSDP)

**Idea:** Shard (partition) model parameters, gradients, and optimizer states across GPUs.

**Process:**

1. Each GPU holds 1/N of model parameters
2. During forward pass: All-gather parameters as needed
3. During backward: Reduce-scatter gradients
4. Update local shard only

**Advantage:** Scales to 100s of GPUs efficiently.

**Used in:** PyTorch FSDP, Meta's LLaMA training

---

#### DeepSpeed (Microsoft)

**Features:**

- **ZeRO (Zero Redundancy Optimizer):** Partition optimizer states, gradients, parameters
- **ZeRO-Offload:** Offload to CPU/NVMe
- **3D parallelism:** Combine data + model + pipeline

**Stages:**

- **ZeRO-1:** Partition optimizer states
- **ZeRO-2:** Partition gradients
- **ZeRO-3:** Partition model parameters

**Used in:** Many open-source LLM training projects

---

#### Megatron-LM (NVIDIA)

**Specializes in tensor and pipeline parallelism.**

**Features:**

- Efficient attention implementation
- Flash Attention integration
- Optimized for NVIDIA GPUs

**Used in:** NVIDIA's Megatron models, some open-source projects

---

### Training Optimizations

**1. Mixed precision training (FP16/BF16):**

- Speeds up training 2-3x
- Reduces memory usage
- BF16 preferred for LLMs (better numerical stability)

**2. Gradient accumulation:**

- Simulate large batch sizes with limited memory
- Forward/backward multiple times before optimizer step

**3. Gradient checkpointing:**

- Trade compute for memory
- Recompute activations during backward pass instead of storing

**4. Flash Attention:**

- Faster, memory-efficient attention implementation
- 2-4x speedup
- Standard in modern LLM training

**5. Learning rate warmup:**

- Start with small LR, increase linearly
- Prevents early instability
- Typical: 2000-5000 steps warmup

---

## Distributed Training Deep Dive

### ZeRO Optimization: Mathematical Foundation

**Problem:** Training 175B parameter model requires ~700GB memory (FP32).

**Memory breakdown per parameter:**

```
Component               | Memory per Parameter | For 175B model
------------------------|----------------------|----------------
Model parameters        | 4 bytes (FP32)       | 700 GB
Gradients              | 4 bytes              | 700 GB
Optimizer states (Adam)| 8 bytes (m, v)       | 1400 GB
Total                  | 16 bytes             | 2.8 TB
```

**ZeRO partitioning:**

$$
\text{Memory per GPU} = \frac{\text{Total Memory}}{N_{GPUs}}
$$

---

### ZeRO Stage Breakdown

**ZeRO-1: Optimizer State Partitioning**

```python
# Traditional: All GPUs store full optimizer state
# Memory: 8 bytes/param × N_params

# ZeRO-1: Partition optimizer states
# Memory per GPU: 8 bytes/param × N_params / N_GPUs

# Example: 175B model, 64 GPUs
Traditional: 1.4 TB per GPU
ZeRO-1: 21.9 GB per GPU  # 64x reduction in optimizer memory
```

**Implementation concept:**

```python
class ZeRO1Optimizer:
    def __init__(self, params, rank, world_size):
        self.rank = rank
        self.world_size = world_size

        # Each GPU only stores 1/world_size of optimizer states
        self.param_partition = params[rank::world_size]
        self.optimizer = Adam(self.param_partition)

    def step(self):
        # 1. All-reduce gradients (sum across GPUs)
        for param in all_params:
            dist.all_reduce(param.grad)

        # 2. Each GPU updates its partition
        self.optimizer.step()

        # 3. All-gather updated parameters
        for i in range(self.world_size):
            dist.broadcast(params[i::world_size], src=i)
```

**ZeRO-2: Gradient Partitioning**

```python
# Additional: Partition gradients
# Memory savings: 4 bytes/param × N_params / N_GPUs

# Example: 175B model, 64 GPUs
ZeRO-1: 21.9 GB optimizer states
ZeRO-2: + 10.9 GB gradients saved = 32.8 GB total savings
```

**ZeRO-3: Parameter Partitioning**

```python
# Ultimate: Even model parameters are partitioned
# Each GPU only holds 1/world_size of parameters

class ZeRO3Layer:
    def __init__(self, layer, rank, world_size):
        self.rank = rank
        self.world_size = world_size

        # Only store my partition
        self.param_partition = self.partition_params(layer.weight)

    def forward(self, x):
        # All-gather full parameters for forward pass
        full_param = dist.all_gather(self.param_partition)

        # Compute forward
        output = F.linear(x, full_param)

        # Discard full_param (free memory)
        del full_param

        return output

    def backward(self, grad_output):
        # All-gather parameters again for backward
        full_param = dist.all_gather(self.param_partition)

        # Compute backward
        grad = compute_grad(grad_output, full_param)

        # Reduce-scatter: Each GPU gets its gradient partition
        grad_partition = dist.reduce_scatter(grad)

        return grad_partition

# Memory per GPU: ~16 bytes/param / N_GPUs
# Example: 175B model, 64 GPUs
# ZeRO-3: 43.75 GB per GPU (down from 2.8 TB)
```

---

### Tensor Parallelism: Column and Row Splitting

**Column-parallel linear layer:**

```python
class ColumnParallelLinear(nn.Module):
    """
    Split output dimension across GPUs
    Weight: (in_features, out_features)
    Split out_features across world_size GPUs
    """
    def __init__(self, in_features, out_features, rank, world_size):
        super().__init__()
        self.rank = rank
        self.world_size = world_size

        # Each GPU holds out_features / world_size columns
        self.out_features_per_partition = out_features // world_size

        self.weight = nn.Parameter(
            torch.randn(in_features, self.out_features_per_partition)
        )

    def forward(self, x):
        # x: (batch, seq_len, in_features)
        # output: (batch, seq_len, out_features / world_size)

        # Local matrix multiplication
        output = F.linear(x, self.weight)

        # No communication needed (each GPU has different output slice)
        return output

# Usage: Split MLP projection
# Y = XW where W is split column-wise
# GPU 0: Y[:, :d/2] = X @ W[:, :d/2]
# GPU 1: Y[:, d/2:] = X @ W[:, d/2:]
```

**Row-parallel linear layer:**

```python
class RowParallelLinear(nn.Module):
    """
    Split input dimension across GPUs
    Weight: (in_features, out_features)
    Split in_features across world_size GPUs
    """
    def __init__(self, in_features, out_features, rank, world_size):
        super().__init__()
        self.rank = rank
        self.world_size = world_size

        # Each GPU holds in_features / world_size rows
        self.in_features_per_partition = in_features // world_size

        self.weight = nn.Parameter(
            torch.randn(self.in_features_per_partition, out_features)
        )

    def forward(self, x):
        # x: (batch, seq_len, in_features / world_size)
        # Each GPU computes partial result

        # Local matmul
        partial_output = F.linear(x, self.weight)

        # All-reduce to sum partial results
        output = dist.all_reduce(partial_output, op=dist.ReduceOp.SUM)

        return output

# Usage: Combine results after split
# Y = XW where X is split row-wise
# GPU 0: Y_partial_0 = X[:, :d/2] @ W[:d/2, :]
# GPU 1: Y_partial_1 = X[:, d/2:] @ W[d/2:, :]
# Final: Y = Y_partial_0 + Y_partial_1 (all-reduce)
```

**Multi-head attention with tensor parallelism:**

```python
class TensorParallelMultiHeadAttention(nn.Module):
    """
    Split attention heads across GPUs
    """
    def __init__(self, d_model, num_heads, rank, world_size):
        super().__init__()
        assert num_heads % world_size == 0

        self.num_heads = num_heads
        self.world_size = world_size
        self.heads_per_partition = num_heads // world_size
        self.d_k = d_model // num_heads

        # Column-parallel: Split heads
        self.W_q = ColumnParallelLinear(d_model, d_model, rank, world_size)
        self.W_k = ColumnParallelLinear(d_model, d_model, rank, world_size)
        self.W_v = ColumnParallelLinear(d_model, d_model, rank, world_size)

        # Row-parallel: Combine heads
        self.W_o = RowParallelLinear(d_model, d_model, rank, world_size)

    def forward(self, x):
        # Each GPU computes subset of heads
        Q = self.W_q(x)  # Local heads only
        K = self.W_k(x)
        V = self.W_v(x)

        # Attention computation (local heads)
        attn_output = self.attention(Q, K, V)

        # Combine heads across GPUs (all-reduce in W_o)
        output = self.W_o(attn_output)

        return output

# Communication pattern:
# 1. W_q, W_k, W_v: No communication (column-parallel)
# 2. Local attention: No communication
# 3. W_o: All-reduce (row-parallel)
# Total: 1 all-reduce per layer
```

---

### Pipeline Parallelism: GPipe and 1F1B

**Naive pipeline (inefficient):**

```
GPU 0: [Layer 1-4]   ████____████____
GPU 1: [Layer 5-8]       ████____████
GPU 2: [Layer 9-12]          ████____
GPU 3: [Layer 13-16]             ████

Bubbles (idle time): 50% GPU utilization
```

**GPipe (improved):**

```python
# Split batch into microbatches
batch_size = 64
num_microbatches = 4
microbatch_size = 16

# Forward pass: Fill pipeline
for mb in microbatches:
    gpu_0_forward(mb)  # → GPU 1
    gpu_1_forward(mb)  # → GPU 2
    gpu_2_forward(mb)  # → GPU 3
    gpu_3_forward(mb)  # → loss

# Backward pass: Drain pipeline
for mb in reversed(microbatches):
    gpu_3_backward(mb)  # ← GPU 2
    gpu_2_backward(mb)  # ← GPU 1
    gpu_1_backward(mb)  # ← GPU 0
    gpu_0_backward(mb)

# Update weights (synchronized)
optimizer.step()
```

**1F1B (One Forward One Backward) - Most efficient:**

```
GPU 0: F0 F1 F2 F3 B0 B1 B2 B3
GPU 1:    F0 F1 F2 F3 B0 B1 B2 B3
GPU 2:       F0 F1 F2 F3 B0 B1 B2 B3
GPU 3:          F0 F1 F2 F3 B0 B1 B2 B3

Bubbles reduced to ~20% (vs 50%)
```

**Implementation:**

```python
class PipelineParallel:
    def __init__(self, layers, rank, world_size, num_microbatches=4):
        self.layers = layers[rank]  # My stage's layers
        self.rank = rank
        self.world_size = world_size
        self.num_microbatches = num_microbatches

    def forward_pass(self, batch):
        microbatches = torch.chunk(batch, self.num_microbatches)

        for mb_id, mb in enumerate(microbatches):
            # Forward
            if self.rank == 0:
                # First stage: Start from input
                activations = self.layers(mb)
            else:
                # Wait for activations from previous stage
                activations = self.recv_from_prev_stage()
                activations = self.layers(activations)

            # Send to next stage
            if self.rank < self.world_size - 1:
                self.send_to_next_stage(activations)

            # Start backward as soon as possible (1F1B)
            if mb_id >= self.world_size - 1 - self.rank:
                self.backward_pass(mb_id)

    def backward_pass(self, mb_id):
        if self.rank == self.world_size - 1:
            # Last stage: Compute loss gradient
            grad = compute_loss_grad()
        else:
            # Wait for gradient from next stage
            grad = self.recv_grad_from_next_stage()

        # Backward through my layers
        grad = self.layers.backward(grad)

        # Send gradient to previous stage
        if self.rank > 0:
            self.send_grad_to_prev_stage(grad)
```

---

### 3D Parallelism: Combining Everything

**Megatron-LM style:**

```python
# Combine all three:
# - Data parallelism: Across data batches
# - Tensor parallelism: Within layers
# - Pipeline parallelism: Across layers

# Example: 512 GPUs training 175B model
# 8-way data parallel (DP=8)
# 8-way tensor parallel (TP=8)
# 8-way pipeline parallel (PP=8)
# Total: 8 × 8 × 8 = 512 GPUs

class ThreeDParallelism:
    def __init__(self, dp_size=8, tp_size=8, pp_size=8):
        self.dp_size = dp_size  # Data parallel
        self.tp_size = tp_size  # Tensor parallel
        self.pp_size = pp_size  # Pipeline parallel

        world_size = dp_size * tp_size * pp_size
        assert world_size == dist.get_world_size()

        # Assign each GPU to a group
        self.setup_process_groups()

    def setup_process_groups(self):
        """
        Create process groups for each parallelism dimension
        """
        # Example GPU layout (512 GPUs):
        # GPU_id = dp_rank * (tp_size * pp_size) + tp_rank * pp_size + pp_rank

        # Data parallel group: GPUs with same model replica
        # Tensor parallel group: GPUs in same layer, same pipeline stage
        # Pipeline parallel group: GPUs across pipeline stages

        self.dp_group = dist.new_group(dp_ranks)
        self.tp_group = dist.new_group(tp_ranks)
        self.pp_group = dist.new_group(pp_ranks)

# Communication overhead:
# - Data parallel: All-reduce gradients (once per iteration)
# - Tensor parallel: All-reduce per layer (many times)
# - Pipeline parallel: Point-to-point between stages
#
# Latency hierarchy:
# TP (lowest latency) → PP → DP (highest latency)
#
# Best practice:
# - TP: Within node (NVLink/InfiniBand)
# - PP: Across nodes
# - DP: Outer dimension
```

---

### Scaling Laws: Chinchilla and Beyond

**Kaplan Scaling Laws (GPT-3 era, 2020):**

$$
L(N, D) = \left(\frac{N_c}{N}\right)^{\alpha_N} + \left(\frac{D_c}{D}\right)^{\alpha_D}
$$

Where:

- $L$ = Loss
- $N$ = Number of parameters
- $D$ = Dataset size (tokens)
- $\alpha_N \approx 0.076$, $\alpha_D \approx 0.095$

**Key insight:** Loss scales as power laws with both model size and data.

---

**Chinchilla Scaling Laws (2022):**

**Optimal allocation of compute:**

$$
N_{\text{optimal}} \propto C^{0.5}
$$

$$
D_{\text{optimal}} \propto C^{0.5}
$$

Where $C$ = Compute budget (FLOPs)

**Practical rule:**

$$
D_{\text{optimal}} = 20 \times N
$$

**Example calculations:**

```python
def chinchilla_optimal_tokens(params):
    """
    Compute optimal training tokens for given parameter count
    """
    return 20 * params

# Examples:
print(f"7B model: {chinchilla_optimal_tokens(7e9) / 1e12:.1f}T tokens")   # 0.14T
print(f"13B model: {chinchilla_optimal_tokens(13e9) / 1e12:.1f}T tokens")  # 0.26T
print(f"70B model: {chinchilla_optimal_tokens(70e9) / 1e12:.1f}T tokens")  # 1.4T
print(f"405B model: {chinchilla_optimal_tokens(405e9) / 1e12:.1f}T tokens") # 8.1T

# Reality check:
# LLaMA 2 70B: Trained on 2T tokens ✓ (slightly overtrained)
# GPT-3 175B: Trained on 300B tokens ✗ (severely undertrained)
# LLaMA 3 405B: Trained on 15T tokens ✓ (overtrained for better quality)
```

**Compute calculation:**

$$
C \approx 6ND
$$

Where:

- $C$ = FLOPs
- $N$ = Parameters
- $D$ = Tokens

```python
def compute_flops(params, tokens):
    """
    Estimate FLOPs for training
    """
    # Forward pass: 2 * params * tokens (multiply-accumulate)
    # Backward pass: ~2x forward
    # Total: ~6 * params * tokens
    return 6 * params * tokens

# Example: Train LLaMA 2 70B on 2T tokens
flops = compute_flops(70e9, 2e12)
print(f"Total FLOPs: {flops:.2e}")  # 8.4e23 FLOPs

# Convert to GPU-days:
# A100 (80GB): ~312 TFLOPS (BF16)
a100_flops_per_sec = 312e12
seconds = flops / a100_flops_per_sec
gpu_days = seconds / (24 * 3600)
print(f"GPU-days on A100: {gpu_days:.0f}")  # ~31,000 GPU-days

# With 2048 GPUs: ~15 days training time
```

---

### Modern Training Best Practices (2025)

**1. Overtrain for quality:**

```
Chinchilla optimal: 20 × N tokens
Production reality: 40-100 × N tokens

Why? Better downstream performance worth the extra compute.
```

**2. Cosine learning rate schedule:**

```python
def get_cosine_schedule_with_warmup(optimizer, warmup_steps, total_steps):
    """
    LR schedule: Warmup → Cosine decay → Min LR
    """
    def lr_lambda(current_step):
        if current_step < warmup_steps:
            # Linear warmup
            return float(current_step) / float(max(1, warmup_steps))

        # Cosine decay
        progress = (current_step - warmup_steps) / (total_steps - warmup_steps)
        return 0.5 * (1.0 + math.cos(math.pi * progress))

    return LambdaLR(optimizer, lr_lambda)

# Typical values:
# Peak LR: 1e-4 to 6e-4 (larger models use smaller LR)
# Warmup: 2000 steps
# Min LR: 10% of peak LR
# Decay: Cosine from peak to min over training
```

**3. Batch size scaling:**

```python
# Critical batch size (where returns diminish):
C_critical = (loss_scale ** 2) / (grad_noise_scale)

# Rule of thumb:
# Start: Small batch size (256-512)
# Scale up: Increase linearly with #GPUs until critical batch size
# Large models: 4M-16M tokens per batch

# Example: LLaMA 2 70B
# Batch size: 4M tokens
# Context length: 4096
# Sequences per batch: 4M / 4096 = 976 sequences
```

**4. Gradient clipping:**

```python
# Prevent gradient explosions
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

**5. Weight decay:**

```python
# AdamW with weight decay
optimizer = torch.optim.AdamW(
    model.parameters(),
    lr=3e-4,
    betas=(0.9, 0.95),
    weight_decay=0.1  # Typical for LLMs
)
```

---

## Fine-Tuning LLMs

### Supervised Fine-Tuning (SFT)

**Goal:** Adapt pretrained model to follow instructions.

**Data format:**

```json
{
  "instruction": "Summarize this article",
  "input": "Article text...",
  "output": "Summary text..."
}
```

**Process:**

1. Start with pretrained base model (e.g., LLaMA 2 base)
2. Fine-tune on instruction-following dataset
3. Optimize same objective (predict next token)
4. Result: Model learns to follow instructions

**Key insight:** Base models know language but need to learn instruction format.

---

### Instruction Tuning

**What:** Fine-tuning on diverse instruction-following tasks.

**Datasets:**

- **FLAN:** 1000+ tasks reformulated as instructions
- **Dolly:** 15K human-generated instructions
- **Alpaca:** 52K GPT-3.5-generated instructions
- **ShareGPT:** Real ChatGPT conversations

**Example:**

```
Instruction: "Translate to French: Hello, how are you?"
Output: "Bonjour, comment allez-vous?"

Instruction: "Write a poem about AI"
Output: "In circuits bright and code so clean..."
```

**Result:** Model becomes general-purpose assistant.

---

## Alignment

### Why Alignment?

**Problem:** Base models don't align with human values:

- Generate toxic content
- Follow harmful instructions
- Don't refuse inappropriate requests

**Goal:** Make models helpful, harmless, honest.

---

### RLHF (Reinforcement Learning from Human Feedback)

**3-step process:**

---

#### Step 1: Supervised Fine-Tuning (SFT)

Train on high-quality demonstrations.

**Data:**

```
Prompt: "How do I bake a cake?"
Response: "Here's a step-by-step guide: 1. Preheat oven..."
```

---

#### Step 2: Reward Modeling

**Goal:** Train reward model to predict human preferences.

**Process:**

1. Generate multiple outputs for same prompt
2. Humans rank outputs (best to worst)
3. Train reward model to predict rankings

**Example:**

```
Prompt: "Explain quantum computing"

Output A: "Quantum computing uses qubits..." (detailed, accurate) → Score: 0.9
Output B: "It's like regular computers but quantum." → Score: 0.3
```

**Reward model:** `reward = RM(prompt, output)`

---

#### Step 3: RL Fine-Tuning (PPO)

**Goal:** Optimize policy to maximize reward while staying close to original model.

**Objective:**
$$\mathcal{L} = \mathbb{E}[\text{reward}(y)] - \beta \cdot \text{KL}(\pi_\theta \| \pi_{\text{ref}})$$

- **Reward term:** Maximize reward model score
- **KL term:** Prevent model from deviating too much (prevents mode collapse)

**Algorithm (PPO - Proximal Policy Optimization):**

1. Generate outputs with current policy
2. Compute rewards
3. Update policy to increase reward (with constraints)
4. Repeat

**Why KL regularization matters:**

- Without it: Model learns to "hack" reward model (generates nonsense with high reward)
- With it: Model stays reasonable while improving

---

### Challenges with RLHF

**Problems:**

1. **Expensive:** Requires human annotations, reward model, RL training
2. **Unstable:** RL can be tricky (divergence, mode collapse)
3. **Reward hacking:** Model exploits reward model weaknesses
4. **Scalability:** Human feedback is bottleneck

---

### DPO (Direct Preference Optimization)

**Key innovation:** Skip reward model and RL, directly optimize on preferences.

**Advantages:**

- **Simpler:** No reward model, no RL (just supervised learning)
- **Stable:** More stable than PPO
- **Efficient:** Faster training

**How it works:**
Use preference data directly:

```
Prompt: "Explain AI"
Preferred: "AI is artificial intelligence..." (clear, helpful)
Rejected: "AI is when computers think." (vague)
```

**Loss function encourages preferred outputs over rejected ones.**

**Used in:** Mistral, Zephyr, many recent models

---

### RLAIF (RL from AI Feedback)

**Idea:** Use AI (GPT-4) instead of humans for feedback.

**Advantages:**

- **Scalable:** No human bottleneck
- **Cheap:** API calls instead of human annotators
- **Fast:** Generate millions of preferences quickly

**Process:**

1. Generate multiple outputs
2. Use GPT-4 to rank them (based on rubric)
3. Train reward model or use DPO

**Trade-off:** AI feedback might miss nuances humans catch.

---

### Constitutional AI (Anthropic)

**Idea:** Guide model with explicit principles ("constitution").

**Process:**

1. Define principles (e.g., "Be helpful and harmless")
2. Generate self-critiques and revisions
3. Train on revised outputs

**Example:**

```
Original: "Here's how to hack a website..."
Critique: "This violates principle of harmlessness."
Revision: "I can't help with that, but I can explain ethical security testing."
```

**Used in:** Claude models

---

### PPO (Proximal Policy Optimization)

**Most common RL algorithm for RLHF.**

**Key idea:** Limit policy updates to prevent large changes (stability).

**Clipping objective:**
Restrict ratio of new policy to old policy: $\frac{\pi_\theta}{\pi_{\text{old}}} \in [1-\epsilon, 1+\epsilon]$

**Why it matters:** Prevents catastrophic forgetting, maintains stability.

---

### GRPO (Group Relative Policy Optimization)

**Recent variant of PPO.**

**Key idea:** Compare outputs within same group (same prompt).

**Advantage:** More stable than vanilla PPO.

---

## Inference vs Training

### Key Differences

| Aspect         | Training                    | Inference            |
| -------------- | --------------------------- | -------------------- |
| **Goal**       | Learn parameters            | Generate outputs     |
| **Batch size** | Large (512-2048)            | Small (1-32)         |
| **Memory**     | Huge (gradients, optimizer) | Smaller (just model) |
| **Bottleneck** | Compute                     | Memory bandwidth     |
| **Precision**  | FP32, BF16, FP16            | FP16, INT8, INT4     |
| **Speed**      | Days/weeks                  | Milliseconds/seconds |

---

### Autoregressive Decoding

**Process:**

```python
def generate(prompt, max_tokens=100):
    tokens = tokenize(prompt)

    for _ in range(max_tokens):
        # Forward pass
        logits = model(tokens)  # (1, seq_len, vocab_size)

        # Get next token logits
        next_token_logits = logits[0, -1, :]  # (vocab_size,)

        # Sample next token
        next_token = sample(next_token_logits)

        # Append to sequence
        tokens.append(next_token)

        # Stop if EOS token
        if next_token == EOS_TOKEN:
            break

    return detokenize(tokens)
```

**Key insight:** Each token generation requires full forward pass → Sequential, slow.

---

### Sampling Strategies

**1. Greedy decoding:**

```python
next_token = argmax(logits)
```

- **Pros:** Deterministic, fast
- **Cons:** Repetitive, boring

**2. Temperature sampling:**

```python
logits = logits / temperature
probs = softmax(logits)
next_token = sample(probs)
```

- **temperature < 1:** More confident, focused
- **temperature > 1:** More random, creative
- **temperature = 0:** Greedy

**3. Top-k sampling:**

```python
top_k_logits, indices = topk(logits, k=50)
probs = softmax(top_k_logits)
next_token = sample(probs)
```

- Keep only top k tokens
- Prevents sampling rare/nonsense tokens

**4. Top-p (nucleus) sampling:**

```python
sorted_probs, indices = sort(softmax(logits), descending=True)
cumsum = cumsum(sorted_probs)
nucleus_mask = cumsum <= p
```

- Keep smallest set of tokens with cumulative probability ≥ p
- Adaptive: Adjusts number of candidates based on confidence

**Best practice:** Top-p (p=0.9) + temperature (T=0.7)

---

## Context Windows and Extensions

### Context Window Limits

**What:** Maximum number of tokens model can process at once.

**Common sizes (2025):**

- GPT-3.5: 4K-16K tokens
- GPT-4: 8K-128K tokens
- Claude 3: 200K tokens
- Gemini 1.5: 1M tokens
- LLaMA 2: 4K tokens
- LLaMA 3: 8K-128K tokens (with extensions)

**Why limitations exist:**

- **Quadratic attention:** $O(n^2)$ complexity
- **Memory:** Storing KV cache grows linearly with sequence length

---

### KV Cache Optimization

**Problem:** Recomputing attention for all previous tokens is wasteful.

**Solution:** Cache key and value projections.

**Without KV cache:**

```python
for t in range(seq_len):
    # Recompute keys and values for ALL previous tokens
    K = compute_keys(tokens[:t+1])  # Wasteful!
    V = compute_values(tokens[:t+1])
    attention = softmax(Q @ K.T) @ V
```

**With KV cache:**

```python
K_cache = []
V_cache = []

for t in range(seq_len):
    # Only compute for NEW token
    K_new = compute_keys(tokens[t])
    V_new = compute_values(tokens[t])

    K_cache.append(K_new)
    V_cache.append(V_new)

    # Use cached values
    attention = softmax(Q @ K_cache) @ V_cache
```

**Speedup:** 10-100x faster inference

**Memory cost:** Linear in sequence length

---

### Context Window Extensions

**1. RoPE Scaling (Position Interpolation):**

- Interpolate position encodings to fit longer sequences
- Example: 4K → 32K by dividing position indices by 8

**2. YaRN (Yet another RoPE extensioN):**

- Improved scaling method
- Better extrapolation

**3. ALiBi:**

- No learned positions → Inherently extends better

**4. Sparse attention:**

- Only attend to subset of tokens
- Sliding window + global tokens

**5. Streaming (infinite context):**

- Process in chunks
- Compress old context into summary tokens

---

## Model Families

### GPT (OpenAI)

**GPT-3 (2020):**

- 175B parameters
- 300B tokens training
- Zero-shot, few-shot learning

**GPT-3.5 (2022):**

- ChatGPT initial model
- Instruction-tuned + RLHF
- Conversational

**GPT-4 (2023):**

- Multimodal (text + images)
- Significantly better reasoning
- ~1.7T parameters (rumored, not confirmed)

**GPT-4 Turbo (2023):**

- 128K context window
- Cheaper, faster

**GPT-4o (2024):**

- "Omni" - text, vision, audio
- Faster, cheaper than GPT-4
- Real-time voice

**o1 / o1-preview (2024):**

- Chain-of-thought reasoning
- Test-time compute
- Best for math, coding, reasoning

**o3 (2025):**

- Next-gen reasoning model
- Even better test-time scaling

**Closed-source:** No weights, API-only

---

### LLaMA (Meta)

**LLaMA 1 (2023):**

- 7B, 13B, 33B, 65B variants
- Trained on 1-1.4T tokens
- Open weights (research-only license)

**LLaMA 2 (2023):**

- 7B, 13B, 70B variants
- 2T tokens training
- **Commercial license**
- LLaMA 2 Chat: Instruction-tuned + RLHF

**LLaMA 3 (2024):**

- 8B, 70B variants
- 15T tokens training
- Better tokenizer (128K vocab)
- **405B model** (most capable open model)

**LLaMA 3.1 (2024):**

- Extended context (128K tokens)
- Multilingual
- Better tool use

**LLaMA 3.2 (2024):**

- Lightweight models (1B, 3B) for edge
- Multimodal variants (vision)

**LLaMA 3.3 (2024):**

- 70B model matching 405B performance on many tasks
- Efficiency improvements

**Key advantage:** Best open-source models, competitive with GPT-4.

---

### Mistral (Mistral AI)

**Mistral 7B (2023):**

- 7B parameters
- Outperforms LLaMA 2 13B
- Sliding window attention (32K tokens)
- Apache 2.0 license

**Mixtral 8x7B (2023):**

- **Mixture of Experts (MoE)**
- 8 expert networks, 2 active per token
- 47B total, 13B active
- Performance of 70B model, speed of 13B

**Mixtral 8x22B (2024):**

- Larger MoE
- 141B total, 39B active
- GPT-4 class performance

**Mistral Large (2024):**

- Closed model (API)
- Competes with GPT-4

**Mistral Small, Medium:**

- Cheaper API options

**Key advantage:** Efficient MoE architecture, open weights for smaller models.

---

### Claude (Anthropic)

**Claude 1 (2023):**

- Constitutional AI
- 100K context window (first to offer)

**Claude 2 (2023):**

- Improved reasoning
- 100K context

**Claude 3 family (2024):**

- **Haiku:** Fast, cheap (similar to GPT-3.5)
- **Sonnet:** Balanced (similar to GPT-4)
- **Opus:** Most capable (best reasoning)

**Claude 3.5 Sonnet (2024):**

- Best Claude model (as of 2024)
- Better than Opus on many tasks
- 200K context window
- Excellent for coding

**Claude 4 (2025-2026):**

- Rumored next generation

**Key advantage:** Long context, safety, excellent at following instructions.

---

### Gemini (Google)

**Gemini 1.0 (2023):**

- **Pro:** Mid-tier (GPT-3.5 level)
- **Ultra:** High-end (GPT-4 level)
- Multimodal from scratch

**Gemini 1.5 (2024):**

- **1M token context window** (largest)
- Pro and Flash variants

**Gemini 2.0 Flash (2025):**

- Faster, cheaper
- State-of-the-art multimodal
- Agent-oriented (tool use, grounding)

**Key advantage:** Longest context window, native multimodal, Google integration.

---

### Qwen (Alibaba)

**Qwen series:**

- Strong multilingual (especially Chinese)
- 0.5B to 72B variants
- Open weights

**Qwen 2.5:**

- Competitive with LLaMA 3
- Excellent coding performance

---

### DeepSeek (Chinese)

**DeepSeek-V2:**

- MoE architecture
- Cost-effective training
- Strong coding

**DeepSeek-R1 (2025):**

- Reasoning model (like o1)
- Open weights
- Competitive reasoning performance

---

### Phi (Microsoft)

**Phi-1, Phi-2, Phi-3:**

- **Small Language Models (SLMs)**
- 1-7B parameters
- Trained on high-quality synthetic data
- Outperforms much larger models on benchmarks

**Key advantage:** Efficient, runs on edge devices.

---

### Gemma (Google)

**Gemma 2B, 7B:**

- Open weights
- Distilled from Gemini
- On-device deployment

---

## Open vs Closed Models

### Closed Models (API-only)

**Examples:** GPT-4, Claude, Gemini Pro/Ultra

**Pros:**

- **Cutting-edge performance**
- **No infrastructure needed**
- **Rapid updates**
- **Easy to use**

**Cons:**

- **Cost:** Pay per token (can get expensive)
- **Vendor lock-in**
- **No customization** (can't fine-tune)
- **Privacy concerns** (data sent to third party)
- **Rate limits**
- **API changes**

**When to use:** Prototyping, low-volume apps, need best quality

---

### Open Models (Self-hosted)

**Examples:** LLaMA, Mistral, Qwen

**Pros:**

- **Full control**
- **Privacy** (data stays in-house)
- **Customizable** (fine-tune, modify)
- **Cost-effective at scale** (no per-token fees)
- **No rate limits**

**Cons:**

- **Infrastructure complexity** (GPUs, serving, scaling)
- **Upfront cost** (hardware)
- **Maintenance burden**
- **Quality gap** (vs GPT-4, Claude)

**When to use:** High volume, sensitive data, need customization, cost-conscious at scale

---

## Mixture of Experts (MoE)

### What is MoE?

**Core idea:** Have multiple "expert" networks, route each token to subset of experts.

**Architecture:**

```
Input token
    ↓
Router (learned) → Decides which experts to use
    ↓
Expert 1, Expert 2, ..., Expert 8
    ↓
Combine outputs (weighted sum)
    ↓
Output
```

---

### Advantages

**1. Efficiency:**

- Total parameters: 47B (Mixtral 8x7B)
- Active parameters: 13B per token
- **Result:** Speed of 13B model, quality of 47B model

**2. Specialization:**

- Different experts specialize in different patterns
- Expert 1: Math
- Expert 2: Code
- Expert 3: Creative writing
- Etc.

**3. Scaling:**

- Add more experts without increasing inference cost proportionally

---

### Challenges

**1. Load balancing:**

- Some experts get overused, others underused
- Need auxiliary loss to encourage balanced routing

**2. Training complexity:**

- More hyperparameters
- Need careful tuning

**3. Memory:**

- All experts must fit in memory (even if inactive)

---

### Examples

**Mixtral 8x7B:**

- 8 experts, 2 active per token
- Open weights
- Best quality-to-cost ratio

**GPT-4 (rumored):**

- Possibly MoE (16 experts, unconfirmed)

---

## Small Language Models (SLMs)

### Why SLMs?

**Need:** Run models on devices (phones, laptops, edge).

**Constraints:**

- Limited memory (< 8GB)
- Limited compute
- Battery life

---

### Examples

**Phi-3 (Microsoft):**

- 3.8B parameters
- Fits on phone
- Competitive with 7B models

**Gemma 2B:**

- Runs on mobile
- Distilled from Gemini

**LLaMA 3.2 (1B, 3B):**

- Meta's edge models

---

### Techniques

**1. Knowledge distillation:**

- Train small model to mimic large model

**2. High-quality data:**

- Textbook-quality synthetic data (Phi approach)

**3. Quantization:**

- INT4, INT8 (4-8x compression)

---

## Key Takeaways

1. **Tokenization matters:** BPE/SentencePiece are standard, vocab size is trade-off
2. **Pretraining = massive compute:** Scaling laws guide optimal training
3. **Alignment is critical:** RLHF/DPO make models useful
4. **KV cache = fast inference:** Cache keys/values to avoid recomputation
5. **Decoder-only won:** GPT-style models dominate
6. **Open models catching up:** LLaMA 3, Mistral competitive with GPT-4
7. **MoE = efficiency:** Mixtral shows MoE works
8. **Context windows expanding:** 4K → 1M tokens in 2 years
9. **SLMs for edge:** Phi, Gemma enable on-device AI
10. **API vs self-hosted:** Trade-off between convenience and control

---

## Interview Questions

**Junior:**

- Explain tokenization and why it's needed
- What's the difference between GPT and BERT?
- What is RLHF at a high level?
- How does KV cache speed up inference?

**Mid-level:**

- Compare different tokenization algorithms (BPE, WordPiece, SentencePiece)
- Explain the three steps of RLHF in detail
- What are scaling laws and why do they matter?
- How does Mixture of Experts work?
- Compare open vs closed models for production use

**Senior:**

- Design a training pipeline for a 70B model (distributed training, optimizations)
- Explain trade-offs between DPO and RLHF
- How would you extend context window from 4K to 32K?
- Design serving infrastructure for high-throughput LLM inference
- When would you choose to self-host vs use API?

---

**Next:** [Part 5: Prompt Engineering](05_Prompt_Engineering.md)
