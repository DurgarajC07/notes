# Part 1: Foundations (Critical but Practical)

> Math and ML basics for AI Engineers—explained with intuition, not heavy proofs

---

## Philosophy

**You don't need a PhD in math to be an AI Engineer.**

What you need:

- **Intuition:** Understand _why_ things work
- **Practical knowledge:** Know _when_ to use techniques
- **Enough theory:** Debug problems, read papers, interview well

Skip:

- Rigorous proofs
- Advanced theorems
- Pure mathematics

---

## 1. Linear Algebra for Deep Learning

### Why it matters

Neural networks are **matrix multiplications**. Transformers are **attention matrices**. Embeddings are **vectors**. Understanding linear algebra means understanding how AI actually works under the hood.

---

### Vectors

**What:** Ordered list of numbers (e.g., `[0.2, -0.5, 0.8]`)

**In AI:**

- **Word embeddings:** "cat" → `[0.2, -0.1, 0.5, ...]` (768 dimensions for BERT)
- **Hidden states:** Internal representations in neural networks
- **Attention scores:** Similarity between tokens

**Operations:**

```python
# Vector addition
v1 = [1, 2, 3]
v2 = [4, 5, 6]
v1 + v2 = [5, 7, 9]  # Element-wise

# Scalar multiplication
2 * [1, 2, 3] = [2, 4, 6]
```

**Magnitude (length):**
$$\|v\| = \sqrt{v_1^2 + v_2^2 + ... + v_n^2}$$

---

### Dot Product

**Formula:**
$$a \cdot b = a_1b_1 + a_2b_2 + ... + a_nb_n$$

**Example:**

```python
a = [1, 2, 3]
b = [4, 5, 6]
dot_product = 1*4 + 2*5 + 3*6 = 32
```

**Intuition:** Measures **similarity** between vectors

- **Positive:** Vectors point in similar directions
- **Zero:** Perpendicular (orthogonal)
- **Negative:** Opposite directions

**In AI:**

- **Attention mechanism:** Dot product between query and key vectors
- **Similarity search:** Find similar embeddings (e.g., RAG retrieval)
- **Neural network layers:** `output = weight · input`

---

### Cosine Similarity

**Formula:**
$$\text{cosine similarity} = \frac{a \cdot b}{\|a\| \|b\|}$$

**Range:** -1 (opposite) to +1 (identical)

**Why:** Dot product depends on vector magnitude. Cosine similarity **normalizes** (only cares about direction, not length).

**Example:**

```python
import numpy as np

a = np.array([1, 2, 3])
b = np.array([4, 5, 6])

# Cosine similarity
cos_sim = np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
# Result: 0.974 (very similar)
```

**In AI:**

- **Vector search:** Find most similar document embeddings
- **RAG retrieval:** Match query embedding to document embeddings
- Most vector databases use cosine similarity by default

---

### Matrices

**What:** 2D array of numbers

**Example:**

```
A = [[1, 2, 3],
     [4, 5, 6]]
```

**Dimensions:** 2×3 (2 rows, 3 columns)

**In AI:**

- **Weight matrices:** Store learned parameters
- **Attention matrices:** Query-Key interactions
- **Batch processing:** Multiple inputs at once

**Matrix multiplication:**

```python
# A (2×3) × B (3×2) = C (2×2)
A = [[1, 2, 3],
     [4, 5, 6]]

B = [[7, 8],
     [9, 10],
     [11, 12]]

# Result C[i,j] = sum of (row i of A) * (column j of B)
```

**Key rule:** Inner dimensions must match: (m×**n**) × (**n**×p) = (m×p)

---

### Why Embeddings Work

**Problem:** Computers can't understand words. They need numbers.

**Bad solution:** One-hot encoding

```
"cat"  → [1, 0, 0, 0, 0, ...]  (10,000 dimensions)
"dog"  → [0, 1, 0, 0, 0, ...]
```

- **Issue:** No notion of similarity. "cat" and "dog" are equally distant from "table".

**Good solution:** Dense embeddings

```
"cat" → [0.2, -0.5, 0.8, 0.1, ...]  (768 dimensions)
"dog" → [0.3, -0.4, 0.7, 0.2, ...]  (close to cat!)
"table" → [-0.8, 0.1, -0.3, 0.9, ...]  (far from cat)
```

**Why it works:**

1. **Learned representations:** Neural networks learn that "cat" and "dog" are similar
2. **Semantic meaning:** Embeddings capture meaning, not just identity
3. **Continuous space:** Small changes in embedding = small changes in meaning

**Magic property:** Vector arithmetic sometimes works!

```
king - man + woman ≈ queen
Paris - France + Italy ≈ Rome
```

**In practice:**

- GPT models: 768-12288 dimensions
- Sentence embeddings: 384-1536 dimensions
- Trade-off: Higher dimensions = more expressiveness, more computation

---

### Eigenvalues and Eigenvectors (Light Touch)

**Intuition:** Special vectors that don't change direction when transformed by a matrix.

**Formula:**
$$Av = \lambda v$$

- $A$: Matrix
- $v$: Eigenvector (direction that stays the same)
- $\lambda$: Eigenvalue (how much it gets scaled)

**Why it matters:**

- **PCA (dimensionality reduction):** Find principal components (eigenvectors)
- **Understanding transformations:** What does a matrix "do"?
- **Graph algorithms:** PageRank uses eigenvectors

**You probably won't implement this yourself** (use NumPy/PyTorch), but understanding helps with debugging and advanced techniques.

---

### Matrix Operations in Transformers

**Self-Attention formula:**
$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

**What's happening:**

1. **$QK^T$:** Matrix multiplication (computes similarity between all tokens)
   - Result: Attention scores matrix (N×N for N tokens)
2. **$\frac{1}{\sqrt{d_k}}$:** Scaling (prevents softmax saturation)
3. **Softmax:** Convert scores to probabilities (each row sums to 1)
4. **Multiply by $V$:** Weighted sum of value vectors

**Shape tracking:**

```
Q: (batch_size, seq_len, d_model)  # Queries
K: (batch_size, seq_len, d_model)  # Keys
V: (batch_size, seq_len, d_model)  # Values

QK^T: (batch_size, seq_len, seq_len)  # Attention weights
Output: (batch_size, seq_len, d_model)  # Weighted values
```

**Why this matters:** Debugging transformer implementations requires tracking matrix shapes.

---

## 2. Probability & Statistics for AI Engineers

### Why it matters

LLMs are **probabilistic models**. They output probability distributions over tokens. Understanding probability helps you understand uncertainty, sampling, and model behavior.

---

### Probability Basics

**Probability:** Likelihood of an event (0 to 1)

**Key concepts:**

- **P(A):** Probability of event A
- **P(A|B):** Probability of A _given_ B happened (conditional probability)
- **P(A, B):** Probability of both A _and_ B (joint probability)

**Chain rule:**
$$P(A, B) = P(A) \cdot P(B|A)$$

**In LLMs:**
Language models compute:
$$P(\text{"The cat sat on the mat"}) = P(\text{The}) \cdot P(\text{cat}|\text{The}) \cdot P(\text{sat}|\text{The cat}) \cdot ...$$

This is **autoregressive generation:** Predict next token given previous tokens.

---

### Distributions Used in ML

**1. Normal (Gaussian) Distribution**

- Bell curve
- Mean $\mu$ and standard deviation $\sigma$
- Appears in weight initialization, noise, VAEs

**2. Bernoulli Distribution**

- Binary outcome (0 or 1)
- Coin flip
- Used in binary classification

**3. Categorical Distribution**

- Multiple outcomes (not just 2)
- Example: Next token prediction (vocabulary of 50K tokens)
- LLM output is a categorical distribution

**4. Uniform Distribution**

- All outcomes equally likely
- Used in random sampling

---

### Maximum Likelihood Estimation (MLE)

**Goal:** Find model parameters that make observed data most likely.

**Intuition:**

- You have data: `["cat", "dog", "cat", "cat"]`
- What's the best estimate for P(cat)? **3/4 = 0.75** (frequency)

**In LLMs:**
Training objective: Maximize likelihood of training data.
$$\text{Maximize } P(\text{training data} | \text{model parameters})$$

Equivalently: **Minimize negative log-likelihood** (cross-entropy loss).

---

### KL Divergence

**What:** Measures how different two probability distributions are.

**Formula:**
$$D_{KL}(P \| Q) = \sum_x P(x) \log \frac{P(x)}{Q(x)}$$

**Intuition:**

- $P$: True distribution
- $Q$: Approximation
- KL divergence: How much information is lost when using $Q$ instead of $P$

**Properties:**

- Always ≥ 0
- KL(P || Q) = 0 if P = Q (identical distributions)
- **Not symmetric:** KL(P || Q) ≠ KL(Q || P)

**In AI:**

- **RLHF:** Regularize fine-tuned model to stay close to base model
  - Minimize: `loss + β * KL(policy || base_model)`
  - Prevents model from deviating too much (catastrophic forgetting)
- **Variational inference:** Approximate complex distributions
- **Distillation:** Train small model to match large model's output distribution

**Interview point:** "Why do we add KL divergence in RLHF?"
Answer: To keep the fine-tuned model from deviating too much from the base model, preventing instability and maintaining general capabilities.

---

### Bayesian Thinking

**Bayes' Theorem:**
$$P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)}$$

**Intuition:** Update beliefs given new evidence.

**Example:**

- **P(spam|"free money"):** Probability email is spam given it contains "free money"
- **P("free money"|spam):** Probability spam contains "free money" (high)
- **P(spam):** Prior probability email is spam (10%)
- **P("free money"):** Probability any email contains "free money"

**In AI:**

- **Bayesian neural networks:** Model uncertainty
- **Active learning:** Choose which data to label next
- **Prompt engineering:** Update strategy based on observed outputs

---

## 3. Optimization

### Why it matters

Training neural networks = **optimization problem**. Adjust weights to minimize loss. Understanding optimization helps debug training issues (non-convergence, overfitting, instability).

---

### Gradient Descent

**Goal:** Find minimum of a function (loss function).

**Intuition:** Roll a ball down a hill. It moves in the direction of steepest descent.

**Algorithm:**

```
1. Start with random weights
2. Compute loss (how bad are predictions?)
3. Compute gradient (which direction reduces loss?)
4. Update weights: w = w - learning_rate * gradient
5. Repeat until converged
```

**Visual analogy:**

```
Loss
 ^
 |     ___/\_
 |   _/      \___
 |  /            \___
 | /                  \
 +----------------------> Weights
```

Start somewhere → follow gradient → reach minimum.

---

### Learning Rate

**What:** How big a step to take in gradient direction.

**Too large:** Overshoot, oscillate, diverge

```
         *
      *     *
   *           *
(never converges)
```

**Too small:** Slow progress, may get stuck

```
         *
        * *(stuck here)
      *
(takes forever)
```

**Just right:** Converge efficiently

**In practice:**

- Start with 1e-4 to 1e-3
- Use learning rate schedulers (decrease over time)
- Warmup: Start small, increase, then decrease

---

### Gradient Descent Variants

**1. Stochastic Gradient Descent (SGD)**

- Update weights after **each sample**
- Fast but noisy

**2. Mini-Batch Gradient Descent**

- Update after a **batch** of samples (e.g., 32, 64)
- Balance between speed and stability
- **Standard in deep learning**

**3. SGD with Momentum**

- Add "velocity" term (remember previous gradients)
- Helps escape local minima and speed up convergence

```python
velocity = 0.9 * velocity - learning_rate * gradient
weights = weights + velocity
```

**4. Adam (Adaptive Moment Estimation)**

- **Most popular optimizer**
- Adapts learning rate for each parameter
- Combines momentum + adaptive learning rates

```python
# Simplified
m = beta1 * m + (1 - beta1) * gradient  # First moment (momentum)
v = beta2 * v + (1 - beta2) * gradient^2  # Second moment (variance)
weights = weights - learning_rate * m / sqrt(v)
```

**When to use:**

- **Adam:** Default choice (works well most of the time)
- **AdamW:** Adam with weight decay (fixes regularization issue, better for transformers)
- **SGD with momentum:** Sometimes better for CNNs
- **AdaFactor:** Memory-efficient (used for large models)

---

### Backpropagation

**What:** Algorithm to compute gradients efficiently.

**Key insight:** **Chain rule** from calculus.

**Forward pass:**

```
Input → Layer 1 → Layer 2 → ... → Output → Loss
```

**Backward pass:**

```
Loss → ∂Loss/∂Output → ∂Loss/∂Layer2 → ... → ∂Loss/∂Layer1 → ∂Loss/∂Input
```

**Chain rule:**
$$\frac{\partial \text{Loss}}{\partial w_1} = \frac{\partial \text{Loss}}{\partial \text{output}} \cdot \frac{\partial \text{output}}{\partial w_1}$$

**You don't implement this yourself** (PyTorch/TensorFlow do it automatically), but understanding helps debug training issues.

---

### Vanishing and Exploding Gradients

**Problem:** In deep networks, gradients can become too small or too large during backpropagation.

**Vanishing gradients:**

- Gradients shrink exponentially with each layer
- Early layers don't learn (gradient ≈ 0)
- **Causes:** Deep networks, sigmoid/tanh activations

**Solution:**

- Use **ReLU** activation (doesn't saturate)
- **Residual connections** (skip connections, ResNet)
- **Layer normalization**
- **Gradient clipping**

**Exploding gradients:**

- Gradients grow exponentially
- Weights update too much, training becomes unstable
- **Causes:** Poor initialization, high learning rates

**Solution:**

- **Gradient clipping:** Cap gradient magnitude

```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

- Better initialization (Xavier, He)
- Lower learning rate

**In transformers:**

- Residual connections help prevent vanishing gradients
- Layer normalization stabilizes training
- Gradient clipping is standard

---

### Learning Rate Scheduling

**Why:** Fixed learning rate is suboptimal. Start large (fast progress), then decrease (fine-tune).

**Common schedules:**

**1. Step decay:**

```python
# Reduce LR by factor every N epochs
lr = initial_lr * (0.1 ** (epoch // 30))
```

**2. Exponential decay:**

```python
lr = initial_lr * (0.95 ** epoch)
```

**3. Cosine annealing:**

```python
# Smooth decrease following cosine curve
lr = min_lr + 0.5 * (max_lr - min_lr) * (1 + cos(π * t / T))
```

**4. Warmup + decay:**

```python
# Common in transformers
# Warmup: Increase LR linearly for first N steps
# Then: Decrease (inverse sqrt or cosine)
```

**In practice:**

- Transformers often use warmup + cosine decay
- AdamW with warmup is standard for LLMs

---

## 4. Information Theory

### Why it matters

LLMs are **information processing systems**. They compress information (training), then decompress (generation). Understanding entropy and cross-entropy explains _why_ language modeling works.

---

### Entropy

**What:** Measures **uncertainty** or **surprise** in a probability distribution.

**Formula:**
$$H(P) = -\sum_x P(x) \log_2 P(x)$$

**Intuition:**

- **Low entropy:** Predictable (e.g., P(heads) = 0.99 for biased coin)
- **High entropy:** Unpredictable (e.g., P(heads) = 0.5 for fair coin)

**Example:**

```python
# Fair coin: P(H) = 0.5, P(T) = 0.5
H = -0.5 * log2(0.5) - 0.5 * log2(0.5) = 1 bit

# Biased coin: P(H) = 0.9, P(T) = 0.1
H = -0.9 * log2(0.9) - 0.1 * log2(0.1) ≈ 0.47 bits
```

**In language:**

- English text has ~1-2 bits of entropy per character
- LLMs compress this by learning patterns

---

### Cross-Entropy Loss

**What:** Measures how well model predictions match true distribution.

**Formula:**
$$H(P, Q) = -\sum_x P(x) \log Q(x)$$

- $P(x)$: True distribution (one-hot encoded label)
- $Q(x)$: Model's predicted distribution

**Intuition:**

- If model predicts correct class with high probability → low loss
- If model predicts wrong class → high loss

**Example:**

```python
# True label: "cat" (index 2)
P = [0, 0, 1, 0, 0]  # One-hot

# Model prediction (after softmax)
Q = [0.1, 0.2, 0.6, 0.05, 0.05]

# Cross-entropy loss
loss = -log(Q[2]) = -log(0.6) ≈ 0.51
```

**Why this is used in LLMs:**

- Training objective: Minimize cross-entropy loss
- Equivalent to maximizing likelihood
- Forces model to assign high probability to correct next token

---

### Why Language Models Optimize Next-Token Prediction

**Key insight:** Modeling P(next token | context) is sufficient to capture all language patterns.

**Autoregressive decomposition:**
$$P(\text{sentence}) = P(w_1) \cdot P(w_2|w_1) \cdot P(w_3|w_1, w_2) \cdot ...$$

**By learning to predict next token:**

- Model learns grammar (syntax)
- Model learns facts (knowledge)
- Model learns reasoning (implicit in patterns)

**Training:**

```
Input:  "The cat sat on the"
Target: "mat"

Model predicts: P("mat" | "The cat sat on the")
Loss: Cross-entropy between prediction and true token
```

**Why this works:**

- Simple objective (predict next token)
- Scales to massive data (trillions of tokens)
- Emergent capabilities (reasoning, coding, math)

---

### Perplexity

**What:** Metric for evaluating language models. Measures how "confused" the model is.

**Formula:**
$$\text{Perplexity} = 2^{H(P,Q)} = 2^{\text{cross-entropy}}$$

Or equivalently:
$$\text{Perplexity} = \exp(\text{average negative log-likelihood})$$

**Intuition:**

- **Low perplexity:** Model is confident and accurate
- **High perplexity:** Model is confused

**Example:**

- Perplexity = 10 means model is "as confused as if it had to choose uniformly from 10 words"
- GPT-3: Perplexity ~20 on validation data
- Random baseline: Perplexity = vocabulary size (50K+)

**In practice:**

- Used to compare models (lower is better)
- Not perfect (doesn't capture downstream task performance)
- Useful for pretraining evaluation

**Interview point:**
"Perplexity measures how well a model predicts a held-out dataset. It's the exponential of cross-entropy loss. Lower perplexity means better predictions."

---

## Practical Takeaways

### What you actually need to know:

**Linear algebra:**

- Vectors are embeddings
- Dot product = similarity
- Cosine similarity for vector search
- Matrix shapes matter (debug transformers)

**Probability:**

- LLMs output probability distributions
- MLE = maximize likelihood of data
- KL divergence keeps models from deviating (RLHF)

**Optimization:**

- Adam/AdamW is default
- Learning rate matters (tune it)
- Gradient clipping prevents explosions
- Warmup + decay for transformers

**Information theory:**

- Cross-entropy loss = training objective
- Perplexity = evaluation metric
- Next-token prediction learns everything

---

### When this knowledge helps:

**Debugging training:**

- Loss not decreasing? Check learning rate
- Gradients exploding? Add clipping
- Model diverging? KL divergence too low

**Designing systems:**

- RAG retrieval? Use cosine similarity
- Fine-tuning? Understand KL regularization
- Evaluation? Perplexity for model comparison

**Interviews:**

- Explain attention mechanism (matrix math)
- Why cross-entropy loss? (Maximum likelihood)
- How does Adam work? (Adaptive learning rates)
- What's perplexity? (Exponential of cross-entropy)

---

## Interview Questions

**Junior:**

- What is an embedding?
- Explain dot product and why it measures similarity
- What's the difference between SGD and Adam?
- Why do we use cross-entropy loss for classification?

**Mid-level:**

- Explain attention mechanism mathematically
- What is KL divergence and why is it used in RLHF?
- How do you prevent vanishing gradients?
- What's perplexity and how do you interpret it?

**Senior:**

- Design a training pipeline for a 7B parameter model (optimization, stability, scaling)
- Explain trade-offs between different optimizers for LLM fine-tuning
- How would you debug training instability in a transformer?

---

## Next Steps

Now that you understand the math foundations, let's build on this:

- [Part 2: Deep Learning Core](02_Deep_Learning_Core.md) - Neural networks and transformers
- [Part 4: Large Language Models](04_Large_Language_Models.md) - Apply these concepts to LLMs

**Remember:** You learned this to understand AI systems, not to become a mathematician. Focus on intuition and application, not rigorous proofs.
