# Part 3: Generative AI Fundamentals

> Understanding different types of generative models and when to use each

---

## What is Generative AI?

**Definition:** AI systems that **create new content** (text, images, code, audio, video) rather than just classify or predict.

**Key shift:** From "classify this image" to "generate an image of a cat wearing a hat."

---

## Discriminative vs Generative Models

### Discriminative Models

**Goal:** Learn boundary between classes.

**Formula:** Model $P(y|x)$ (probability of label given input)

**Example:**

- Input: Email text
- Output: Spam or Not Spam

**Models:** Logistic regression, SVM, BERT (for classification)

**Use case:** Classification, prediction

---

### Generative Models

**Goal:** Learn data distribution, generate new samples.

**Formula:** Model $P(x)$ or $P(x|y)$

**Example:**

- Input: "A cat wearing a wizard hat"
- Output: Generated image

**Models:** GPT, DALL-E, Stable Diffusion

**Use case:** Content generation, data augmentation

---

### Why Generative Models Are Harder

1. **Higher dimensional output:** Generate entire images/text, not just labels
2. **Must capture full distribution:** Not just decision boundary
3. **Evaluation is subjective:** No single "correct" answer
4. **Mode collapse risk:** Model generates limited variety

---

## Types of Generative Models

```
Generative Models
├── Autoregressive (GPT, LLaMA)
├── Autoencoders & VAEs
├── GANs (StyleGAN, BigGAN)
├── Diffusion (Stable Diffusion, DALL-E 2)
├── Flow-based (RealNVP, Glow)
└── Hybrid (VQ-VAE, Parti)
```

---

## 1. Autoregressive Models

### What

**Core idea:** Generate data **one element at a time**, conditioned on previous elements.

**Formula:**
$$P(x) = P(x_1) \cdot P(x_2|x_1) \cdot P(x_3|x_1, x_2) \cdot ... \cdot P(x_n|x_1,...,x_{n-1})$$

---

### For Text (GPT)

**Process:**

```
Input:  "The cat"
Generate: "sat" (sample from P(·|"The cat"))
Input:  "The cat sat"
Generate: "on" (sample from P(·|"The cat sat"))
...
```

**Architecture:** Decoder-only transformer (masked self-attention)

**Training:**

- Objective: Predict next token
- Loss: Cross-entropy

---

### For Images (PixelCNN, ImageGPT)

**Process:** Generate pixel by pixel (top-left to bottom-right).

**Challenge:** Very slow (must generate each pixel sequentially).

**Why not popular:** Diffusion models are better for images.

---

### Strengths

- **Simple training:** Just predict next token/pixel
- **High quality:** GPT-4 shows amazing results
- **Flexible:** Can condition on any prefix
- **Exact likelihood:** Can compute $P(x)$ exactly

---

### Weaknesses

- **Slow generation:** Sequential (can't parallelize)
- **Exposure bias:** Errors compound (generated tokens as input)
- **Left-to-right bias:** Can't revise earlier parts

---

### Used In

- **LLMs:** GPT-2, GPT-3, GPT-4, LLaMA, Mistral, Claude
- **Code:** Codex, GitHub Copilot
- **Audio:** WaveNet (older, not used much now)

**Verdict:** **Dominant for text generation.**

---

## 2. Autoencoders

### What

**Core idea:** Compress data to lower-dimensional representation, then reconstruct.

**Architecture:**

```
Input (x)
    ↓
Encoder → Latent code (z)
    ↓
Decoder → Reconstruction (x')
```

**Training:** Minimize reconstruction error: $\|x - x'\|^2$

---

### Vanilla Autoencoder

**Not generative!** Just learns to compress and reconstruct. Can't generate new samples.

**Use case:**

- Dimensionality reduction
- Denoising
- Anomaly detection

---

### Variational Autoencoder (VAE)

**Key innovation:** Make latent space **smooth and continuous**.

**Architecture:**

```
Input (x)
    ↓
Encoder → μ, σ (mean and variance)
    ↓
Sample z ~ N(μ, σ)  # Reparameterization trick
    ↓
Decoder → Reconstruction (x')
```

**Training objective:**
$$\text{Loss} = \text{Reconstruction Loss} + \text{KL}(q(z|x) \| p(z))$$

- **Reconstruction loss:** How well decoder reconstructs input
- **KL divergence:** Regularizes latent space to be close to N(0, 1)

---

### Why VAE Works for Generation

**Smooth latent space:**

- Encode image of "cat" → $z_1 = [0.5, -0.2, ...]$
- Encode image of "dog" → $z_2 = [0.6, -0.1, ...]$
- Interpolate: $z_3 = 0.5 * z_1 + 0.5 * z_2$
- Decode $z_3$ → "cat-dog hybrid"

**Generation:**

1. Sample $z \sim N(0, 1)$ (random point in latent space)
2. Decode $z$ → new image

---

### Strengths

- **Fast sampling:** Single forward pass through decoder
- **Smooth latent space:** Good for interpolation
- **Principled (probabilistic):** Bayesian framework

---

### Weaknesses

- **Blurry outputs:** Reconstruction loss prefers averaging
- **Less diverse:** Regularization can limit expressiveness
- **Not SOTA:** Diffusion models produce better images

---

### Used In

- **Latent Diffusion (Stable Diffusion):** VAE compresses image before diffusion
- **Older image generation:** Before GANs/Diffusion became popular
- **Representation learning:** Learn good features

**Verdict:** **Component of modern systems** (e.g., Stable Diffusion), not standalone.

---

## 3. GANs (Generative Adversarial Networks)

### What

**Core idea:** Two networks compete:

- **Generator:** Creates fake data
- **Discriminator:** Distinguishes real from fake

**Training:**

1. Generator tries to fool discriminator
2. Discriminator tries to catch generator
3. Both improve through adversarial training

---

### Architecture

```
Random noise (z)
    ↓
Generator (G) → Fake image (G(z))
    ↓
Discriminator (D) → Real or Fake?

Real images also fed to D
```

**Objective:**

- Generator: Maximize $\log D(G(z))$ (fool discriminator)
- Discriminator: Maximize $\log D(x) + \log(1 - D(G(z)))$ (classify correctly)

**Minimax game:** $\min_G \max_D V(D, G)$

---

### Training Process

```
for epoch in range(epochs):
    # Train discriminator
    real_images = get_batch()
    fake_images = generator(noise)
    d_loss = -log(D(real)) - log(1 - D(fake))
    update_discriminator()

    # Train generator
    fake_images = generator(noise)
    g_loss = -log(D(fake))  # Want D(fake) to be high
    update_generator()
```

---

### Strengths

- **Sharp outputs:** No blurriness (unlike VAEs)
- **High quality:** StyleGAN produces photorealistic faces
- **No explicit likelihood:** Don't need to model $P(x)$ directly

---

### Weaknesses

- **Training instability:** Hard to balance G and D
- **Mode collapse:** Generator produces limited variety
- **No latent space structure:** Unlike VAEs, z is not interpretable
- **Evaluation is hard:** No good metrics (FID, IS are imperfect)

---

### Famous GAN Architectures

**StyleGAN (2019):**

- State-of-the-art face generation
- Controls style at different scales
- Used in: thispersondoesnotexist.com

**BigGAN (2018):**

- Large-scale ImageNet generation
- High-resolution, diverse images

**Pix2Pix, CycleGAN:**

- Image-to-image translation
- Used in: Colorization, style transfer

---

### Used In (2025)

**Declining in popularity:**

- Diffusion models now produce better images
- GANs still used for:
  - Fast generation (single forward pass)
  - Super-resolution
  - Face swapping, deepfakes
  - Some video generation

**Verdict:** **Being replaced by diffusion models**, but still relevant for specific applications.

---

## 4. Diffusion Models

### What

**Core idea:** Learn to **reverse a noise corruption process**.

**Forward process (add noise):**

```
Real image → +noise → +noise → +noise → ... → Pure noise
```

**Reverse process (denoise):**

```
Pure noise → -noise → -noise → -noise → ... → Real image
```

**Training:** Learn to predict and remove noise at each step.

---

### DDPM (Denoising Diffusion Probabilistic Models)

**Forward diffusion:**
Add Gaussian noise over $T$ timesteps:
$$x_t = \sqrt{\alpha_t} x_{t-1} + \sqrt{1 - \alpha_t} \epsilon$$

**Reverse diffusion:**
Learn $p(x_{t-1} | x_t)$ (how to denoise):
$$x_{t-1} = \mu_\theta(x_t, t) + \sigma_t z$$

**Training objective:**
Predict noise $\epsilon$ added at each timestep:
$$\mathcal{L} = \mathbb{E}[\|\epsilon - \epsilon_\theta(x_t, t)\|^2]$$

---

### Generation Process

```python
# Start with pure noise
x_T = torch.randn(1, 3, 256, 256)

# Iteratively denoise
for t in reversed(range(T)):
    # Predict noise
    noise_pred = model(x_t, t)
    # Remove noise
    x_{t-1} = denoise(x_t, noise_pred, t)

# x_0 is the generated image
```

---

### Latent Diffusion (Stable Diffusion)

**Problem:** Diffusion on raw pixels is slow (high resolution).

**Solution:** Compress image to latent space (VAE), then diffuse.

**Architecture:**

```
Text prompt
    ↓
CLIP Text Encoder → Text embedding
    ↓
U-Net (diffusion) → Latent code
    ↓
VAE Decoder → Image
```

**Why faster:**

- Diffuse in 64×64 latent space instead of 512×512 pixel space
- 10-100x faster

---

### Classifier-Free Guidance

**Problem:** How to condition on text prompts?

**Solution:** Train model with and without conditioning.

**Formula:**
$$\epsilon_\theta(x_t, t, c) = \epsilon_\theta(x_t, t, \emptyset) + w \cdot (\epsilon_\theta(x_t, t, c) - \epsilon_\theta(x_t, t, \emptyset))$$

- $c$: Condition (text prompt)
- $w$: Guidance scale (controls how much to follow prompt)
- Higher $w$: More faithful to prompt, less diverse

---

### Strengths

- **State-of-the-art image quality**
- **Stable training:** No adversarial dynamics
- **Flexible conditioning:** Text, images, sketches
- **Diverse outputs:** Different noise → different images

---

### Weaknesses

- **Slow generation:** Must iterate through many timesteps (50-1000)
- **Memory intensive:** Need to store intermediate states
- **Hard to control:** Prompts are imprecise

**Speedups:**

- **DDIM:** Fewer steps (50 instead of 1000)
- **LCM (Latent Consistency Models):** 4-8 steps
- **Distillation:** Train fast student model

---

### Used In

- **Stable Diffusion:** Open-source text-to-image (most popular)
- **DALL-E 2:** OpenAI's text-to-image
- **Midjourney:** Commercial image generation (likely diffusion-based)
- **Imagen (Google):** Text-to-image
- **Video generation:** Runway Gen-2, Pika, Stable Video

**Verdict:** **Dominant for image generation** (2023-2025).

---

## 5. Flow-Based Models

### What

**Core idea:** Learn **invertible transformations** from data distribution to simple distribution (e.g., Gaussian).

**Forward:** $z = f(x)$ (data → noise)
**Inverse:** $x = f^{-1}(z)$ (noise → data)

**Key property:** Exact likelihood computation (unlike GANs).

---

### Strengths

- **Exact likelihood:** Can compute $P(x)$ exactly
- **Exact inference:** $z = f(x)$ (no sampling needed)
- **Stable training:** No adversarial dynamics

---

### Weaknesses

- **Architectural constraints:** Must use invertible layers (limiting expressiveness)
- **Slower than GANs:** More complex architecture
- **Less popular:** Diffusion models produce better results

---

### Used In

- **RealNVP, Glow:** Image generation (older)
- **WaveGlow:** Audio synthesis
- **Mostly research:** Not widely used in production

**Verdict:** **Niche**, mostly replaced by diffusion models.

---

## 6. Hybrid Approaches

### VQ-VAE (Vector Quantized VAE)

**Idea:** Discrete latent codes (like a codebook).

**Architecture:**

```
Image → Encoder → Continuous latent → Vector Quantization → Discrete codes
                                                                    ↓
                                            Decoder ← Lookup in codebook
```

**Why useful:**

- Discrete codes work well with autoregressive models
- Used in DALL-E (original)

---

### Parti (Google)

**Idea:** Treat images as sequences of tokens.

**Process:**

1. Tokenize image (VQ-VAE)
2. Train autoregressive model (like GPT) on image tokens
3. Generate image tokens autoregressively
4. Decode tokens to image

**Advantage:** Leverage powerful transformer models.

---

## When to Use Each Model Type

| Model Type               | Best For                   | Avoid When           |
| ------------------------ | -------------------------- | -------------------- |
| **Autoregressive (GPT)** | Text, code                 | Images (too slow)    |
| **VAE**                  | Latent representations     | Need sharp outputs   |
| **GAN**                  | Fast generation, deepfakes | Need stable training |
| **Diffusion**            | High-quality images        | Need fast generation |
| **Flow**                 | Research, exact likelihood | Need SOTA results    |

---

## Industry Reality (2025)

### Text generation:

- **Autoregressive transformers dominate** (GPT, LLaMA, Claude)
- No competition from other paradigms

### Image generation:

- **Diffusion models dominate** (Stable Diffusion, DALL-E, Midjourney)
- GANs declining (except niche uses)

### Video generation:

- **Early days:** Diffusion-based (Runway, Pika, Sora)
- Autoregressive (emerging)

### Audio:

- **Autoregressive + Diffusion:** AudioLM, MusicLM, Stable Audio

---

## Interview Talking Points

**Beginner:**

- Explain autoregressive generation (next-token prediction)
- What's the difference between discriminative and generative?
- High-level: How does Stable Diffusion work?

**Mid-level:**

- Compare VAE vs GAN vs Diffusion
- Explain classifier-free guidance
- Why are diffusion models better than GANs for images?
- How does VQ-VAE enable DALL-E?

**Senior:**

- Design a hybrid model for a specific use case
- Trade-offs between generation speed and quality
- How would you optimize diffusion for real-time generation?
- Explain mathematical foundations of diffusion models

---

## Key Takeaways

1. **Autoregressive = SOTA for text** (GPT-style models)
2. **Diffusion = SOTA for images** (Stable Diffusion)
3. **GANs declining** but still fast for some tasks
4. **VAEs** are components, not standalone systems
5. **Training stability:** Diffusion > VAE > GAN
6. **Generation speed:** GAN (fast) > VAE > Diffusion (slow) > Autoregressive (very slow for images)
7. **Output quality (images):** Diffusion > GAN > VAE
8. **Output quality (text):** Autoregressive (no competition)

---

**Next:** [Part 4: Large Language Models](04_Large_Language_Models.md) (Deep dive into LLMs)
