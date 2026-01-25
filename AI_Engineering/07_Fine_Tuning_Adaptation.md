# Part 7: Fine-tuning & Adaptation

## Table of Contents

1. [Introduction to Fine-tuning](#introduction)
2. [Full Fine-tuning](#full-fine-tuning)
3. [Parameter-Efficient Fine-Tuning (PEFT)](#peft)
4. [LoRA & QLoRA](#lora-qlora)
5. [Quantization Techniques](#quantization)
6. [Continued Pretraining vs Instruction Tuning](#pretraining-vs-instruction)
7. [When NOT to Fine-tune](#when-not-to-fine-tune)
8. [Tools & Frameworks](#tools-frameworks)
9. [Cost & Performance Trade-offs](#cost-performance)
10. [Advanced Topics](#advanced-topics)

---

## Introduction to Fine-tuning {#introduction}

### What is Fine-tuning?

**Fine-tuning** = Further training a pretrained model on task-specific data.

**Why fine-tune?**

- Adapt model to domain-specific language (legal, medical, code)
- Improve performance on specific tasks
- Change model behavior/style (tone, format)
- Inject proprietary knowledge (not suitable for RAG)
- Reduce prompt length (behavior baked into weights)

**Fine-tuning vs other approaches:**

```
Approach         | Use Case                    | Cost   | Speed  | Complexity
-----------------|-----------------------------|---------|---------|-----------
Prompt Eng       | General tasks, quick wins   | Low     | Fast    | Low
RAG              | External knowledge, facts   | Medium  | Fast    | Medium
Fine-tuning      | Behavior change, style      | High    | Slow    | High
Pretraining      | New language, domain        | Very High | Very Slow | Very High
```

**Key principle:**

> Try prompt engineering first → Add RAG if needed → Fine-tune only if necessary

---

## Full Fine-tuning {#full-fine-tuning}

### What is Full Fine-tuning?

**Definition:** Update ALL parameters of the model during training.

**Process:**

1. Load pretrained model weights
2. Replace/add task-specific head (optional)
3. Train on task data with lower learning rate
4. Save updated weights

**Example: GPT-2 full fine-tuning**

```python
from transformers import GPT2LMHeadModel, GPT2Tokenizer, Trainer, TrainingArguments

# Load pretrained model
model = GPT2LMHeadModel.from_pretrained("gpt2")  # 124M parameters
tokenizer = GPT2Tokenizer.from_pretrained("gpt2")

# Prepare dataset
train_dataset = ...  # Your task-specific data

# Training config
training_args = TrainingArguments(
    output_dir="./gpt2-finetuned",
    per_device_train_batch_size=4,
    learning_rate=5e-5,  # Lower than pretraining (1e-4)
    num_train_epochs=3,
    save_steps=1000,
    logging_steps=100,
    fp16=True,  # Mixed precision
)

# Train
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
)

trainer.train()
```

### Memory Requirements

**Full fine-tuning memory cost:**

```
Component           | Memory per Billion Parameters
--------------------|--------------------------------
Model weights       | 2 GB (fp16) or 4 GB (fp32)
Gradients           | 2 GB (fp16) or 4 GB (fp32)
Optimizer states    | 8 GB (Adam: 2x gradients)
Activations         | 4-8 GB (depends on batch size)
--------------------|--------------------------------
TOTAL               | ~16 GB per billion parameters (fp16)
```

**Example: LLaMA 3 8B full fine-tuning**

- Model: 8B parameters × 2 bytes = 16 GB
- Gradients: 16 GB
- Optimizer (Adam): 32 GB
- Activations: 10 GB
- **Total: ~74 GB VRAM**

**Impossible to fine-tune 8B+ models on consumer GPUs (24 GB).**

### When to use Full Fine-tuning

✅ Use when:

- You have large datasets (100K+ examples)
- Significant domain shift (e.g., pretraining on English → fine-tune on Python)
- You need maximum performance
- You have compute resources

❌ Don't use when:

- Small dataset (<10K examples) → Overfitting risk
- Limited GPU memory
- Quick iteration needed
- Behavior change, not knowledge injection

---

## Parameter-Efficient Fine-Tuning (PEFT) {#peft}

### What is PEFT?

**PEFT** = Fine-tuning methods that update only a small subset of parameters.

**Key idea:**

- Freeze most pretrained weights
- Train small adapter modules
- Reduce memory and compute drastically

**PEFT methods:**

```
Method               | Trainable Params | Memory | Speed | Performance
---------------------|------------------|--------|-------|-------------
LoRA                 | 0.1-1%          | Low    | Fast  | ~99% of full FT
QLoRA                | 0.1-1%          | Very Low | Fast | ~98% of full FT
Adapter Layers       | 1-5%            | Low    | Medium | ~95% of full FT
Prefix Tuning        | 0.01-0.1%       | Very Low | Fast | ~90% of full FT
Prompt Tuning        | 0.001-0.01%     | Very Low | Very Fast | ~85% of full FT
```

### Adapter Layers

**Concept:** Insert small trainable layers between frozen transformer blocks.

**Architecture:**

```
Original Transformer Block:
Input → Self-Attention → LayerNorm → FFN → Output

With Adapter:
Input → Self-Attention → Adapter → LayerNorm → FFN → Adapter → Output
```

**Adapter structure:**

```python
class Adapter(nn.Module):
    def __init__(self, d_model, bottleneck_size):
        super().__init__()
        self.down_project = nn.Linear(d_model, bottleneck_size)
        self.up_project = nn.Linear(bottleneck_size, d_model)
        self.activation = nn.ReLU()

    def forward(self, x):
        residual = x
        x = self.down_project(x)
        x = self.activation(x)
        x = self.up_project(x)
        return x + residual  # Skip connection
```

**Parameters:**

- If `d_model = 4096`, `bottleneck = 64`
- Down: 4096 × 64 = 262K params
- Up: 64 × 4096 = 262K params
- **Total per adapter: ~524K params**
- For 32-layer model with 2 adapters per layer: 32 × 2 × 524K = **33.5M trainable params**
- Compared to 7B base model: **0.5% trainable**

### Prefix Tuning

**Concept:** Prepend trainable "prefix" tokens to each layer's input.

**How it works:**

```
Input: "Translate to French: Hello"
Prefix: [P1] [P2] [P3] [P4] [P5]
Actual input to model: [P1] [P2] [P3] [P4] [P5] Translate to French: Hello
```

- Prefix tokens are virtual (not real tokens)
- Each layer has its own prefix embeddings
- Only prefix embeddings are trained

**Parameters:**

- Prefix length: 20 tokens
- Model dimension: 4096
- Number of layers: 32
- **Total: 20 × 4096 × 32 = 2.6M trainable params**
- For 7B model: **0.037% trainable**

### Prompt Tuning

**Simpler version of prefix tuning:**

- Add soft prompts only to the input layer
- Even fewer parameters

**Example:**

```python
from peft import PromptTuningConfig, get_peft_model

config = PromptTuningConfig(
    task_type="CAUSAL_LM",
    num_virtual_tokens=20,  # Number of soft prompt tokens
)

model = AutoModelForCausalLM.from_pretrained("gpt2")
model = get_peft_model(model, config)

print(model.print_trainable_parameters())
# Output: trainable params: 81,920 || all params: 124,521,728 || trainable%: 0.066%
```

---

## LoRA & QLoRA {#lora-qlora}

### LoRA (Low-Rank Adaptation)

**Key insight:** Weight updates during fine-tuning are low-rank.

**Mathematical formulation:**

```
Original forward pass:
h = W_0 * x

With LoRA:
h = W_0 * x + (B * A) * x
```

Where:

- `W_0` = Original frozen weights (d × d)
- `A` = Trainable matrix (d × r)
- `B` = Trainable matrix (r × d)
- `r` = Rank (typically 4-64, much smaller than d)

**Parameter reduction:**

```
Original: d × d parameters
LoRA: d × r + r × d = 2 × d × r parameters

Example (d=4096, r=16):
- Original: 4096 × 4096 = 16.8M params
- LoRA: 2 × 4096 × 16 = 131K params
- Reduction: 128×
```

**Implementation:**

```python
import torch
import torch.nn as nn

class LoRALayer(nn.Module):
    def __init__(self, in_features, out_features, rank=8, alpha=16):
        super().__init__()

        # Frozen original weight
        self.weight = nn.Parameter(torch.randn(out_features, in_features))
        self.weight.requires_grad = False

        # LoRA matrices
        self.lora_A = nn.Parameter(torch.randn(rank, in_features))
        self.lora_B = nn.Parameter(torch.zeros(out_features, rank))

        # Scaling factor
        self.scaling = alpha / rank

    def forward(self, x):
        # Original transformation
        result = F.linear(x, self.weight)

        # LoRA transformation
        lora_result = (x @ self.lora_A.T) @ self.lora_B.T

        return result + lora_result * self.scaling
```

### LoRA Hyperparameters

**Rank (r):**

- Lower rank = Fewer parameters, faster training, less capacity
- Higher rank = More parameters, slower training, more capacity
- **Typical values: 4, 8, 16, 32, 64**
- **Rule of thumb:** Start with r=16

**Alpha (α):**

- Scaling factor for LoRA updates
- **Typical: α = 2 × r** (e.g., r=16 → α=32)
- Higher α = Stronger LoRA influence

**Target modules:**

- Apply LoRA to which layers?
- **Common choices:**
  - Query & Value projections: `q_proj`, `v_proj`
  - All attention: `q_proj`, `k_proj`, `v_proj`, `o_proj`
  - Attention + FFN: All attention + `gate_proj`, `up_proj`, `down_proj`

**Example config:**

```python
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,                      # Rank
    lora_alpha=32,             # Scaling
    target_modules=[           # Which layers to adapt
        "q_proj",
        "v_proj",
        "k_proj",
        "o_proj",
    ],
    lora_dropout=0.05,         # Dropout for regularization
    bias="none",               # Don't train biases
    task_type="CAUSAL_LM",
)

model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b-hf")
model = get_peft_model(model, lora_config)

model.print_trainable_parameters()
# Output: trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.062%
```

### QLoRA (Quantized LoRA)

**Key innovation:** Combine LoRA with 4-bit quantization.

**Memory savings:**

```
Full FT (LLaMA 7B):
- 7B × 2 bytes (fp16) = 14 GB
- Optimizer states = 28 GB
- Total: ~42 GB

LoRA (LLaMA 7B):
- Base model: 14 GB (fp16)
- LoRA adapters: 30 MB
- Optimizer (only for adapters): 60 MB
- Total: ~14.1 GB

QLoRA (LLaMA 7B):
- Base model: 3.5 GB (4-bit)
- LoRA adapters: 30 MB
- Optimizer: 60 MB
- Total: ~3.6 GB ✅ Fits on consumer GPU!
```

**QLoRA components:**

1. **4-bit NormalFloat (NF4):** Better quantization for normally-distributed weights
2. **Double quantization:** Quantize the quantization constants
3. **Paged optimizers:** Use CPU RAM when GPU runs out

**Usage:**

```python
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model

# Quantization config
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)

# Load quantized model
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=bnb_config,
    device_map="auto",
)

# Add LoRA
lora_config = LoraConfig(r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"])
model = get_peft_model(model, lora_config)

# Train normally
trainer = Trainer(model=model, ...)
trainer.train()
```

### LoRA vs QLoRA Comparison

```
Metric               | LoRA (fp16)      | QLoRA (4-bit)
---------------------|------------------|------------------
7B model memory      | 14 GB            | 3.5 GB
Training speed       | Faster (1×)      | Slower (1.3-1.5×)
Performance          | Better           | Slightly worse (~1-2%)
GPU required         | A100 (40 GB)     | RTX 3090 (24 GB) ✅
Use case             | Best performance | Limited GPU
```

**Rule of thumb:**

- LoRA if you have GPU memory
- QLoRA if memory-constrained

---

## Quantization Techniques {#quantization}

### What is Quantization?

**Quantization** = Reduce precision of model weights from 32-bit/16-bit to 8-bit/4-bit.

**Why quantize?**

- Reduce model size (4× for 8-bit, 8× for 4-bit)
- Reduce memory usage
- Faster inference (sometimes)
- Enable deployment on edge devices

**Quantization awareness:**

- **Post-Training Quantization (PTQ):** Quantize after training
- **Quantization-Aware Training (QAT):** Train with quantization simulation

### Quantization Formats

**INT8 (8-bit integer):**

- Each weight represented as 8-bit integer (-128 to 127)
- Scale & zero-point for dequantization
- **Size reduction: 4× (from fp32) or 2× (from fp16)**

**INT4 (4-bit integer):**

- Each weight in 4 bits (16 possible values)
- **Size reduction: 8× (from fp32) or 4× (from fp16)**
- More aggressive, higher accuracy loss

**NF4 (4-bit NormalFloat):**

- Optimized for normally-distributed weights (typical in LLMs)
- Better quality than uniform INT4

### Quantization Methods

#### 1. GPTQ (Accurate Post-Training Quantization)

**Method:** Layer-wise quantization with Hessian-based error compensation.

**Characteristics:**

- High accuracy retention (~1-2% loss)
- Requires calibration data
- One-time quantization (not on-the-fly)

**Usage:**

```python
from transformers import AutoModelForCausalLM, GPTQConfig

gptq_config = GPTQConfig(
    bits=4,
    dataset="c4",  # Calibration dataset
    tokenizer="meta-llama/Llama-2-7b-hf",
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=gptq_config,
    device_map="auto",
)
```

**Pros:**

- Good accuracy
- Fast inference (optimized kernels)

**Cons:**

- Slow quantization process (hours)
- Requires GPU for inference

#### 2. AWQ (Activation-aware Weight Quantization)

**Key insight:** Protect salient weights (weights with large activations).

**Method:**

1. Run calibration data through model
2. Identify important weights based on activations
3. Apply mixed precision (keep important weights at higher precision)

**Characteristics:**

- Better accuracy than GPTQ for 4-bit
- Faster quantization than GPTQ
- Good for inference

**Usage:**

```python
from transformers import AutoModelForCausalLM, AwqConfig

awq_config = AwqConfig(
    bits=4,
    group_size=128,
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=awq_config,
    device_map="auto",
)
```

#### 3. GGUF (GPT-Generated Unified Format)

**Format by llama.cpp for CPU inference.**

**Quantization levels:**

```
Format   | Bits | Size (7B) | Quality | Use Case
---------|------|-----------|---------|------------------
Q4_0     | 4    | 3.5 GB    | Good    | Standard
Q4_K_M   | 4    | 4.1 GB    | Better  | Recommended
Q5_K_M   | 5    | 4.6 GB    | Great   | Balance
Q8_0     | 8    | 7.0 GB    | Excellent | High quality
FP16     | 16   | 13 GB     | Perfect | Full precision
```

**Usage:**

```bash
# Convert model to GGUF
python convert.py --model meta-llama/Llama-2-7b-hf --outtype q4_k_m

# Run with llama.cpp
./main -m llama-2-7b.q4_k_m.gguf -p "Hello, how are you?"
```

**Pros:**

- CPU inference (no GPU needed)
- Very efficient
- Great for local deployment

**Cons:**

- Slower than GPU inference
- Limited to llama.cpp ecosystem

#### 4. bitsandbytes (QLoRA quantization)

**Built for QLoRA fine-tuning.**

**Features:**

- 4-bit NF4 quantization
- 8-bit INT8 quantization
- Double quantization
- Paged optimizers

**Usage (covered in QLoRA section):**

```python
from transformers import BitsAndBytesConfig

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
)
```

### Quantization Comparison

```
Method         | Target    | Bits | Quality | Speed  | Use Case
---------------|-----------|------|---------|--------|------------------
bitsandbytes   | Training  | 4/8  | Good    | Fast   | QLoRA fine-tuning
GPTQ           | Inference | 4    | Great   | Fast   | GPU inference
AWQ            | Inference | 4    | Excellent | Fast | GPU inference
GGUF           | Inference | 4-8  | Good    | Medium | CPU inference
```

**Decision tree:**

```
Fine-tuning?
  └─ Yes → bitsandbytes (QLoRA)
  └─ No → Inference only
      └─ GPU available?
          └─ Yes → AWQ or GPTQ
          └─ No → GGUF (llama.cpp)
```

---

## Continued Pretraining vs Instruction Tuning {#pretraining-vs-instruction}

### Continued Pretraining

**Definition:** Continue training on unlabeled text (next-token prediction).

**Use case:** Adapt to new domain or language.

**Example:**

- Pretrain on general web text
- Continue pretraining on medical papers → Medical domain expert

**Data format:**

```
Raw text (no instructions):

"Myocardial infarction (MI), commonly known as a heart attack, occurs when
blood flow decreases or stops to a part of the heart..."

"Atrial fibrillation is an irregular and often rapid heart rate that can
increase the risk of stroke..."
```

**Characteristics:**

- Large datasets needed (10B+ tokens)
- Expensive (weeks of training)
- Changes knowledge, not behavior

### Instruction Tuning

**Definition:** Fine-tune on instruction-response pairs.

**Use case:** Teach model to follow instructions and respond helpfully.

**Data format:**

```json
[
  {
    "instruction": "Explain what a heart attack is to a 10-year-old.",
    "response": "A heart attack happens when part of your heart doesn't get enough blood..."
  },
  {
    "instruction": "List 3 symptoms of atrial fibrillation.",
    "response": "1. Irregular heartbeat\n2. Shortness of breath\n3. Fatigue"
  }
]
```

**Characteristics:**

- Smaller datasets (10K-1M examples)
- Faster training (hours to days)
- Changes behavior and format

### Supervised Fine-Tuning (SFT)

**Supervised Fine-Tuning (SFT)** = Instruction tuning with high-quality examples.

**Process:**

1. Start with pretrained base model (e.g., LLaMA 3 8B Base)
2. Fine-tune on instruction datasets
3. Get instruction-following model (e.g., LLaMA 3 8B Instruct)

**Example datasets:**

- **OpenAssistant:** 161K human conversations
- **ShareGPT:** Conversations from ChatGPT
- **Alpaca:** 52K instruction-response pairs (GPT-3.5 generated)
- **Dolly:** 15K human-written examples

**Training:**

```python
from datasets import load_dataset
from transformers import Trainer, TrainingArguments

# Load instruction dataset
dataset = load_dataset("tatsu-lab/alpaca")

def format_instruction(example):
    return f"### Instruction:\n{example['instruction']}\n\n### Response:\n{example['output']}"

# Apply LoRA
lora_config = LoraConfig(r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"])
model = get_peft_model(model, lora_config)

# Train
training_args = TrainingArguments(
    per_device_train_batch_size=4,
    learning_rate=2e-5,
    num_train_epochs=3,
    logging_steps=10,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=dataset["train"],
)

trainer.train()
```

### Multi-Task Fine-Tuning

**Definition:** Fine-tune on multiple tasks simultaneously.

**Benefits:**

- Model generalizes better
- Single model for multiple tasks
- Reduces catastrophic forgetting

**Example:**

```json
[
  {
    "task": "summarization",
    "input": "Long article...",
    "output": "Summary..."
  },
  { "task": "translation", "input": "Hello", "output": "Bonjour" },
  {
    "task": "QA",
    "input": "What is the capital of France?",
    "output": "Paris"
  },
  { "task": "classification", "input": "I love this!", "output": "Positive" }
]
```

**Instruction format:**

```
Task: Summarization
Input: [Long article]
Output: [Summary]

Task: Translation (English to French)
Input: Hello
Output: Bonjour
```

### Comparison

```
Method               | Data Size | Cost  | Changes     | Use Case
---------------------|-----------|-------|-------------|------------------
Continued Pretraining | 10B+ tokens | High | Knowledge | New domain/language
Instruction Tuning   | 10K-1M    | Low   | Behavior  | Follow instructions
Multi-Task FT        | 100K-10M  | Medium| Generalization | Multiple tasks
```

---

## When NOT to Fine-tune {#when-not-to-fine-tune}

### Fine-tuning is Overrated

**Common mistakes:**

1. Fine-tuning when prompt engineering would work
2. Fine-tuning to inject facts (use RAG instead)
3. Fine-tuning without proper evaluation
4. Fine-tuning with tiny datasets

### Decision Tree

```
Need to change model behavior?
├─ No → Prompt engineering
└─ Yes → Need external knowledge?
    ├─ Yes → RAG
    └─ No → Change style/format?
        ├─ Yes → Instruction tuning
        └─ No → Domain shift?
            ├─ Yes → Continued pretraining
            └─ No → Maybe fine-tuning not needed
```

### When Fine-tuning Makes Sense

✅ **Use fine-tuning when:**

1. **Consistent style/format needed:**

   ```
   Example: Medical diagnosis formatting
   - Model must always output: Diagnosis | Confidence | Reasoning
   - 10K examples → Fine-tune to enforce format
   ```

2. **Domain-specific jargon:**

   ```
   Example: Legal contract generation
   - Base model doesn't know legal terms well
   - Fine-tune on legal corpus
   ```

3. **Reduce prompt length:**

   ```
   Before fine-tuning: 500-token system prompt
   After fine-tuning: 10-token prompt (behavior in weights)
   ```

4. **Insufficient in-context examples:**

   ```
   Task requires 50 examples to work (exceeds context)
   → Fine-tune instead
   ```

5. **Latency critical:**
   ```
   RAG adds 100ms retrieval latency
   → Fine-tune to remove retrieval step
   ```

### When Fine-tuning Doesn't Make Sense

❌ **Don't fine-tune when:**

1. **Injecting facts:**

   ```
   Wrong: Fine-tune to memorize "CEO of Acme Corp is John Doe"
   Right: Use RAG to retrieve from database

   Why? Facts change, model becomes stale.
   ```

2. **Small dataset (<1000 examples):**

   ```
   Wrong: Fine-tune LLaMA 7B on 100 examples
   Result: Overfitting, loss of general knowledge

   Right: Few-shot prompting with examples
   ```

3. **Prompt engineering not tried:**

   ```
   Wrong: Immediately jump to fine-tuning
   Right: Try these first:
     1. Zero-shot with clear instructions
     2. Few-shot with examples
     3. Chain of Thought prompting
     4. Prompt chaining
   ```

4. **Evaluation unclear:**

   ```
   Wrong: Fine-tune without knowing target metrics
   Right: Define success criteria first:
     - Accuracy on test set
     - Human preference
     - Business metrics
   ```

5. **Can't maintain model:**
   ```
   Fine-tuned model = responsibility
   - Need to retrain when base model updates
   - Need to maintain training pipeline
   - Need to version control models
   ```

### Cost-Benefit Analysis

**Example: Customer support chatbot**

**Option 1: RAG**

- Cost: $100 vector DB + $0.02/query
- Latency: 150ms
- Maintenance: Update documents
- Pros: Easy to update knowledge

**Option 2: Fine-tuning**

- Cost: $200 training + $50/month hosting
- Latency: 50ms
- Maintenance: Retrain every quarter
- Pros: Faster, consistent style

**Decision:**

- If knowledge changes weekly → RAG
- If style consistency critical + knowledge stable → Fine-tune

---

## Tools & Frameworks {#tools-frameworks}

### Hugging Face PEFT

**Most popular library for parameter-efficient fine-tuning.**

**Supported methods:**

- LoRA / QLoRA
- Prefix Tuning
- Prompt Tuning
- Adapter Layers

**Installation:**

```bash
pip install peft transformers accelerate bitsandbytes
```

**Basic usage:**

```python
from peft import LoraConfig, get_peft_model, TaskType
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b-hf")

lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type=TaskType.CAUSAL_LM,
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.062%

# Train normally with Trainer
trainer = Trainer(model=model, ...)
trainer.train()

# Save only adapter weights (small)
model.save_pretrained("./lora-adapters")  # ~30 MB

# Load adapter
from peft import PeftModel
base_model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b-hf")
model = PeftModel.from_pretrained(base_model, "./lora-adapters")
```

### Axolotl

**Full-featured fine-tuning framework with configs.**

**Features:**

- YAML-based configuration
- Built-in LoRA/QLoRA support
- Multi-GPU training
- Flash Attention 2
- Popular datasets preloaded

**Installation:**

```bash
git clone https://github.com/OpenAccess-AI-Collective/axolotl
cd axolotl
pip install -e .
```

**Example config:**

```yaml
# config.yaml
base_model: meta-llama/Llama-2-7b-hf
model_type: LlamaForCausalLM

# LoRA config
adapter: lora
lora_r: 16
lora_alpha: 32
lora_dropout: 0.05
lora_target_modules:
  - q_proj
  - v_proj

# Training data
datasets:
  - path: tatsu-lab/alpaca
    type: alpaca

# Training params
micro_batch_size: 4
gradient_accumulation_steps: 4
num_epochs: 3
learning_rate: 0.0002
lr_scheduler: cosine
warmup_steps: 100

# Output
output_dir: ./lora-out

# Quantization (QLoRA)
load_in_4bit: true
bnb_4bit_compute_dtype: bfloat16
```

**Training:**

```bash
accelerate launch -m axolotl.cli.train config.yaml
```

### LLaMA Factory

**GUI-based fine-tuning tool.**

**Features:**

- Web UI for non-coders
- Supports LLaMA, Mistral, Qwen, etc.
- LoRA / QLoRA / Full fine-tuning
- Built-in datasets
- Evaluation & inference

**Installation:**

```bash
git clone https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory
pip install -e .
```

**Launch UI:**

```bash
python src/train_web.py
```

**Opens browser with GUI to:**

- Select model
- Choose dataset
- Configure LoRA
- Train with one click
- Evaluate & chat with model

### Unsloth

**Fastest training library (2-5× faster than Hugging Face).**

**Features:**

- Optimized kernels for LoRA
- Reduces memory usage
- Support for LLaMA, Mistral, Phi, Gemma
- Drop-in replacement for PEFT

**Installation:**

```bash
pip install unsloth
```

**Usage:**

```python
from unsloth import FastLanguageModel

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/llama-3-8b-bnb-4bit",  # Pre-quantized
    max_seq_length=2048,
    dtype=None,
    load_in_4bit=True,
)

# Add LoRA
model = FastLanguageModel.get_peft_model(
    model,
    r=16,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    lora_alpha=16,
    lora_dropout=0,
    bias="none",
)

# Train 2× faster
trainer = Trainer(model=model, ...)
trainer.train()
```

**Speed comparison:**

```
Method          | Training Time (LLaMA 7B, 50K examples)
----------------|----------------------------------------
Standard PEFT   | 8 hours
Unsloth         | 4 hours (2× faster) ✅
```

### OpenAI Fine-tuning API

**Managed fine-tuning for GPT-3.5/GPT-4.**

**Pros:**

- No infrastructure setup
- Easy to use
- Automatic scaling

**Cons:**

- Expensive ($8-$24 per 1M tokens)
- Black box (can't inspect weights)
- Vendor lock-in

**Usage:**

```bash
# Prepare data (JSONL)
{"messages": [{"role": "user", "content": "What is 2+2?"}, {"role": "assistant", "content": "4"}]}

# Upload file
openai tools fine_tunes.prepare_data -f train.jsonl

# Start fine-tuning
openai api fine_tunes.create -t train.jsonl -m gpt-3.5-turbo

# Use fine-tuned model
openai api chat.completions.create \
  -m ft:gpt-3.5-turbo:my-org:custom-suffix:id \
  --messages '[{"role": "user", "content": "Hello"}]'
```

**Pricing (2025):**

- GPT-3.5 Turbo: $8 per 1M training tokens
- GPT-4o mini: $12 per 1M training tokens
- GPT-4: $24 per 1M training tokens

### Tool Comparison

```
Tool             | Ease of Use | Speed  | Cost   | Flexibility | Best For
-----------------|-------------|--------|--------|-------------|------------------
Hugging Face PEFT| Medium      | Medium | Free   | High        | Research, custom
Axolotl          | Easy (YAML) | Medium | Free   | High        | Production
LLaMA Factory    | Very Easy   | Medium | Free   | Medium      | Non-coders
Unsloth          | Medium      | Fast   | Free   | High        | Speed-critical
OpenAI API       | Very Easy   | N/A    | High   | Low         | Quick prototypes
```

---

## Cost & Performance Trade-offs {#cost-performance}

### Training Costs

**Cost factors:**

- Model size
- Dataset size
- Training duration
- Hardware (GPU type)
- Method (full FT vs PEFT)

**Example: LLaMA 3 8B fine-tuning**

**Full fine-tuning:**

```
Hardware: 8× A100 80GB ($30/hr each)
Duration: 24 hours
Cost: 8 × $30 × 24 = $5,760
```

**LoRA (PEFT):**

```
Hardware: 1× A100 40GB ($2.5/hr)
Duration: 8 hours
Cost: 1 × $2.5 × 8 = $20 ✅
```

**QLoRA:**

```
Hardware: 1× RTX 4090 24GB (own GPU)
Duration: 12 hours
Cost: Electricity ~$2 ✅✅
```

### Performance vs Cost

**Hypothetical benchmark (Accuracy on task):**

```
Method               | Accuracy | Cost    | Training Time
---------------------|----------|---------|---------------
GPT-4 (no FT)        | 75%      | $0      | 0
Few-shot prompting   | 78%      | $0      | 0
Full FT              | 95%      | $5,760  | 24h
LoRA (r=64)          | 94%      | $40     | 12h
LoRA (r=16)          | 92%      | $20     | 8h
QLoRA (r=16)         | 90%      | $2      | 12h
Prompt tuning        | 85%      | $10     | 4h
```

**Key insight:**

- Diminishing returns after LoRA r=16
- Full FT costs 288× more for 3% accuracy gain

### Memory-Performance Trade-off

**LLaMA 3 8B on different GPUs:**

```
Method          | VRAM   | GPU             | Batch Size | Speed
----------------|--------|-----------------|------------|-------
Full FT (fp32)  | 160 GB | 4× A100 80GB    | 8          | 1×
Full FT (fp16)  | 80 GB  | 1× A100 80GB    | 4          | 1.2×
LoRA (fp16)     | 20 GB  | 1× A100 40GB    | 8          | 1.5×
QLoRA (4-bit)   | 8 GB   | 1× RTX 4090 24GB| 4          | 0.8×
```

**Consumer GPUs:**

```
GPU              | VRAM  | Can train (LLaMA 3 8B)
-----------------|-------|------------------------
RTX 4090         | 24 GB | QLoRA ✅
RTX 4080         | 16 GB | QLoRA (small batch) ✅
RTX 3090         | 24 GB | QLoRA ✅
RTX 3080         | 10 GB | ❌ (too small)
Apple M2 Max     | 64 GB | LoRA (unified memory) ✅
```

### Dataset Size vs Performance

**Instruction tuning effectiveness:**

```
Dataset Size | Performance  | Observations
-------------|--------------|------------------------------------------
100          | Poor         | Severe overfitting
1,000        | Okay         | Model learns format, limited generalization
10,000       | Good         | Solid performance on similar tasks
100,000      | Great        | Strong generalization
1,000,000+   | Excellent    | Diminishing returns, very costly
```

**Rule of thumb:**

- **Minimum: 1,000 examples** (for LoRA)
- **Sweet spot: 10K-100K examples**
- **Beyond 1M: Marginal gains**

### Catastrophic Forgetting

**Problem:** Fine-tuning causes model to forget original capabilities.

**Example:**

```
Base LLaMA 3 8B:
Q: What is the capital of France?
A: Paris. [Correct]

After fine-tuning on Python coding (10K examples):
Q: What is the capital of France?
A: def capital_of_france(): return "Paris" [Wrong format!]
```

**Mitigation strategies:**

1. **Mix general data:**

   ```
   80% task-specific + 20% general knowledge examples
   ```

2. **Lower learning rate:**

   ```
   Use 1e-5 instead of 1e-4
   ```

3. **Fewer epochs:**

   ```
   Train for 1-2 epochs instead of 5+
   ```

4. **Regularization:**

   ```python
   training_args = TrainingArguments(
       weight_decay=0.01,  # L2 regularization
   )
   ```

5. **Replay buffer:**
   ```
   Keep 10% random samples from pretraining data
   ```

---

## Advanced Topics {#advanced-topics}

### Merging LoRA Adapters

**Problem:** You fine-tuned multiple LoRAs for different tasks.

**Solution:** Merge them into base model.

**Method 1: Linear combination**

```python
from peft import PeftModel

base_model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b-hf")

# Load first adapter
model = PeftModel.from_pretrained(base_model, "./lora-task1")

# Merge into base
merged_model = model.merge_and_unload()

# Now it's a standard model (no adapters)
merged_model.save_pretrained("./merged-model")
```

**Method 2: Multi-adapter (switch at runtime)**

```python
from peft import PeftModel

base_model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b-hf")

# Load multiple adapters
model = PeftModel.from_pretrained(base_model, "./lora-task1", adapter_name="task1")
model.load_adapter("./lora-task2", adapter_name="task2")

# Switch adapters
model.set_adapter("task1")
output1 = model.generate(...)

model.set_adapter("task2")
output2 = model.generate(...)
```

### DARE (Drop And REscale)

**Method to merge multiple LoRAs with minimal interference.**

**Steps:**

1. Randomly drop some weights from each adapter
2. Rescale remaining weights
3. Merge into base model

**Result:** Better performance than simple averaging.

### Task Arithmetic

**Concept:** Add/subtract task vectors.

**Example:**

```
Model for Task A = Base + LoRA_A
Model for Task B = Base + LoRA_B

Multi-task model = Base + 0.5 * LoRA_A + 0.5 * LoRA_B
```

**Remove unwanted behavior:**

```
Model without toxicity = Base - LoRA_toxic
```

### Mixture of LoRA Experts (MoLE)

**Concept:** Route inputs to different LoRA adapters based on task.

**Architecture:**

```
Input → Router (classifier) → LoRA_1, LoRA_2, or LoRA_3 → Output
```

**Use case:** Single model for multiple tasks, efficient switching.

### DPO with LoRA

**Combine Direct Preference Optimization (DPO) with LoRA.**

**Why:**

- DPO aligns model with human preferences
- LoRA makes it memory-efficient

**Usage:**

```python
from trl import DPOTrainer

# Preference dataset
# {"chosen": "Good response", "rejected": "Bad response"}

dpo_trainer = DPOTrainer(
    model=model,  # LoRA-enabled model
    ref_model=ref_model,  # Reference model (frozen)
    train_dataset=preference_dataset,
    beta=0.1,  # DPO temperature
)

dpo_trainer.train()
```

### Quantization-Aware LoRA Training

**Train LoRA while simulating quantization effects.**

**Result:** Better accuracy after quantization to 4-bit.

```python
from transformers import BitsAndBytesConfig

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=bnb_config,
)

# Train LoRA on quantized model
# Adapters learn to compensate for quantization errors
```

---

## Practical Takeaways

### For Beginners (0-2 years)

**Learn:**

- Understand the difference between full fine-tuning and PEFT
- Use Hugging Face PEFT for LoRA experiments
- Start with QLoRA on consumer GPU (RTX 3090/4090)
- Fine-tune on small datasets (Alpaca 52K)

**Avoid:**

- Full fine-tuning without GPU cluster
- Fine-tuning when prompting works
- Tiny datasets (<1K examples)

### For Mid-Level (2-5 years)

**Learn:**

- Implement LoRA from scratch to understand internals
- Experiment with different ranks and target modules
- Use Axolotl or Unsloth for production training
- Measure catastrophic forgetting
- Merge LoRA adapters

**Focus:**

- When to fine-tune vs RAG vs prompting
- Cost optimization (QLoRA, smaller models)
- Evaluation strategies

### For Senior (5+ years)

**Learn:**

- Multi-task fine-tuning strategies
- Task arithmetic and adapter merging
- DPO/RLHF with PEFT
- Scaling fine-tuning to multiple GPUs
- Custom quantization methods

**Focus:**

- Architecture decisions (fine-tune vs other approaches)
- Cost-performance trade-offs
- Production fine-tuning pipelines
- Fine-tuning as part of larger system (RAG + fine-tuning)

---

## Interview Questions

### Junior Level

**Q: What is the difference between full fine-tuning and LoRA?**

**A:**
Full fine-tuning updates all model parameters (expensive, requires lots of GPU memory). LoRA adds small trainable matrices to frozen layers, updating only ~0.1% of parameters (cheap, fits on consumer GPUs, 99% of full FT performance).

**Q: What is QLoRA?**

**A:**
QLoRA = LoRA + 4-bit quantization. Reduces memory 4× (e.g., LLaMA 7B from 14GB to 3.5GB), enabling fine-tuning on RTX 3090/4090 GPUs. Slightly slower training, ~1-2% accuracy loss.

**Q: When should you fine-tune vs use RAG?**

**A:**

- RAG: Need external/changing knowledge (facts, documents)
- Fine-tuning: Need consistent style/format, domain-specific behavior
- Try prompting first, RAG second, fine-tune last

### Mid Level

**Q: Explain how LoRA works mathematically.**

**A:**
LoRA decomposes weight updates into low-rank matrices:

- Original: `h = W * x`
- LoRA: `h = W * x + (B * A) * x`
- `W` (d×d) frozen, `A` (d×r) and `B` (r×d) trainable, r ≪ d
- For d=4096, r=16: 16.8M params → 131K (128× reduction)

**Q: What are the trade-offs between LoRA rank values?**

**A:**

- **Low rank (r=4-8):** Fewer params, faster, less capacity, may underfit
- **High rank (r=64-128):** More params, slower, more capacity, may overfit
- **Sweet spot: r=16-32** for most tasks
- Higher rank if large dataset + complex task

**Q: How do you prevent catastrophic forgetting during fine-tuning?**

**A:**

1. Mix 20% general data with task data
2. Use lower learning rate (1e-5 vs 1e-4)
3. Train for fewer epochs (1-2 vs 5+)
4. Add regularization (weight decay)
5. Use PEFT (less forgetting than full FT)

### Senior Level

**Q: Design a fine-tuning strategy for a company with 50 different internal tasks.**

**A:**
**Architecture:**

1. Base LLaMA 3 8B model (shared)
2. Task-specific LoRA adapters (50× ~30MB = 1.5GB total)
3. Router model to select adapter based on task

**Training:**

- Multi-task fine-tuning on all 50 tasks simultaneously
- Prevents forgetting, improves generalization
- Use mixture of experts approach (route to relevant LoRAs)

**Serving:**

- Load base model once (8GB VRAM)
- Swap LoRA adapters dynamically (<5ms overhead)
- Scales to 100s of tasks without memory increase

**Trade-offs:**

- More complex than single model
- Router adds latency (~10ms)
- Easier to update specific task without retraining all

**Q: You fine-tuned a model, but inference is too slow. What do you do?**

**A:**
**Diagnose:**

1. Merge LoRA adapters into base model (removes adapter overhead)
2. Quantize merged model (GPTQ/AWQ for 4-bit)
3. Use faster inference engine (vLLM, TGI)
4. Enable Flash Attention 2
5. Reduce context length if possible
6. Use smaller model (Mistral 7B vs LLaMA 13B)
7. Consider distillation (teach small model from large)

**If still slow:**

- Batch multiple requests
- Use speculative decoding
- Switch to smaller base model + better fine-tuning

---

## Summary

**Core concepts:**

1. **Fine-tuning:** Further training pretrained models on specific data
2. **Full FT:** Expensive, updates all parameters
3. **PEFT:** Efficient, updates <1% parameters
4. **LoRA:** Adds low-rank matrices, ~99% of full FT performance
5. **QLoRA:** LoRA + 4-bit quantization, fits on consumer GPUs

**Key trade-offs:**

- **Full FT:** Best performance, high cost, needs big GPUs
- **LoRA:** Great performance, low cost, fits on smaller GPUs
- **QLoRA:** Good performance, very low cost, consumer GPUs

**Decision tree:**

1. Try prompt engineering first
2. Add RAG for external knowledge
3. Fine-tune for style/format/behavior
4. Use LoRA by default (not full FT)
5. Use QLoRA if memory-constrained

**Tools:**

- **PEFT:** General-purpose PEFT library
- **Axolotl:** Config-based training
- **LLaMA Factory:** GUI for non-coders
- **Unsloth:** Fastest training
- **OpenAI API:** Managed, expensive

**Best practices:**

- Start with r=16 for LoRA rank
- Target Q, K, V projections at minimum
- Use 10K+ examples for good results
- Monitor for catastrophic forgetting
- Evaluate properly before deployment

**Next:** Part 8 - GenAI System Design (agents, tools, frameworks)
