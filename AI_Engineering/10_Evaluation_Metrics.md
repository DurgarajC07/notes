# Part 10: Evaluation & Metrics

## Table of Contents

1. [Introduction](#introduction)
2. [Evaluation Strategies](#evaluation-strategies)
3. [Traditional NLP Metrics](#traditional-metrics)
4. [LLM-Specific Metrics](#llm-specific-metrics)
5. [Retrieval Metrics (RAG)](#retrieval-metrics)
6. [LLM-as-a-Judge](#llm-as-judge)
7. [Human Evaluation](#human-evaluation)
8. [Business Metrics](#business-metrics)
9. [Evaluation Frameworks](#evaluation-frameworks)

---

## Introduction {#introduction}

### Why Evaluation is Critical

**"You can't improve what you don't measure."**

**Common mistake:** Building GenAI systems without evaluation.

**Consequences:**

- No idea if changes improve quality
- Can't compare models objectively
- Hallucinations go undetected
- User dissatisfaction

**Evaluation in development cycle:**

```
1. Baseline: Test current system
2. Change: Improve prompt / model / retrieval
3. Evaluate: Measure impact
4. Compare: Better or worse?
5. Deploy: If better, ship it
6. Monitor: Track in production
```

### Evaluation Types

**Offline evaluation:**

- Test on static dataset before deployment
- Fast iteration
- Doesn't reflect real users

**Online evaluation:**

- Test with real users in production
- A/B testing
- Slower, but accurate

**Human evaluation:**

- Humans rate outputs
- Gold standard
- Expensive and slow

**Automated evaluation:**

- Metrics (BLEU, ROUGE, etc.)
- LLM-as-judge
- Fast and cheap
- Less accurate than human

---

## Evaluation Strategies {#evaluation-strategies}

### Golden Dataset

**Create test set of input-output pairs:**

```json
[
  {
    "input": "What is the capital of France?",
    "expected_output": "Paris",
    "category": "factual"
  },
  {
    "input": "Summarize: [Long article]",
    "expected_output": "Article discusses...",
    "category": "summarization"
  }
]
```

**Characteristics:**

- **Size:** 100-1000 examples
- **Diversity:** Cover edge cases, different categories
- **Quality:** Human-verified expected outputs

**Usage:**

```python
def evaluate_on_golden_dataset(dataset, model):
    correct = 0

    for example in dataset:
        output = llm_call(example["input"], model=model)

        if is_correct(output, example["expected_output"]):
            correct += 1

    accuracy = correct / len(dataset)
    return accuracy
```

### Regression Testing

**Prevent breaking existing functionality.**

**Process:**

1. Create test suite (100+ examples)
2. Run tests on every prompt change
3. Alert if accuracy drops

**Example:**

```python
def test_prompt_change():
    old_prompt = load_prompt("summarization", version="v2")
    new_prompt = load_prompt("summarization", version="v3")

    old_accuracy = evaluate(old_prompt, test_set)
    new_accuracy = evaluate(new_prompt, test_set)

    assert new_accuracy >= old_accuracy - 0.05, "Accuracy dropped >5%!"
```

### A/B Testing (Online Evaluation)

**Test in production with real users:**

```python
def ab_test_handler(user_id, query):
    # 50% to variant A, 50% to variant B
    variant = "A" if hash(user_id) % 2 == 0 else "B"

    if variant == "A":
        prompt = load_prompt("v2")
    else:
        prompt = load_prompt("v3")

    response = llm_call(prompt.format(query=query))

    # Log variant
    db.log(user_id=user_id, variant=variant, response=response)

    return response
```

**Analyze results:**

```sql
-- Compare user ratings by variant
SELECT variant, AVG(user_rating) as avg_rating, COUNT(*) as samples
FROM logs
WHERE date >= '2025-01-01'
GROUP BY variant;
```

**Results:**

```
variant | avg_rating | samples
--------|------------|--------
A       | 4.2        | 5000
B       | 4.5        | 5000  ✅ Winner
```

**Decision:** Promote variant B to 100% traffic.

### Shadow Deployment

**Run new version in parallel, don't show to users.**

```python
def shadow_deployment(query):
    # Serve production version to user
    prod_response = llm_call(query, model="gpt-4o")

    # Also call new version (don't show to user)
    asyncio.create_task(test_new_version(query))

    return prod_response

async def test_new_version(query):
    new_response = llm_call(query, model="gpt-4o-mini")

    # Compare
    comparison = compare_responses(prod_response, new_response)
    db.log(comparison)
```

**Use case:**

- Test new model without risking user experience
- Collect data for offline analysis

### Canary Deployment

**Gradual rollout to minimize risk:**

```
Stage 1: 5% of traffic → Monitor for 24 hours
Stage 2: 25% of traffic → Monitor for 24 hours
Stage 3: 50% of traffic → Monitor for 24 hours
Stage 4: 100% of traffic
```

**If error rate spikes, rollback immediately.**

---

## Traditional NLP Metrics {#traditional-metrics}

### BLEU (Bilingual Evaluation Understudy)

**Use case:** Machine translation quality.

**How it works:**

- Measures n-gram overlap between generated and reference text
- Precision-based metric

**Formula:**

```
BLEU = Brevity Penalty × exp(Σ log(precision_n))

precision_n = (matching n-grams) / (total n-grams in candidate)
```

**Example:**

```
Reference: "The cat sat on the mat"
Candidate: "The cat is on the mat"

Unigram overlap: 5/6 (the, cat, on, the, mat)
Bigram overlap: 3/5 (the cat, on the, the mat)

BLEU-2 ≈ 0.71
```

**Code:**

```python
from nltk.translate.bleu_score import sentence_bleu

reference = [["the", "cat", "sat", "on", "the", "mat"]]
candidate = ["the", "cat", "is", "on", "the", "mat"]

score = sentence_bleu(reference, candidate)
print(f"BLEU: {score:.2f}")
```

**Limitations:**

- Doesn't understand semantics
- Penalizes paraphrasing (synonyms are treated as wrong)
- Poor for creative tasks

**When to use:**

- Translation
- Exact wording matters

### ROUGE (Recall-Oriented Understudy for Gisting Evaluation)

**Use case:** Summarization quality.

**Variants:**

- **ROUGE-N:** N-gram overlap (similar to BLEU)
- **ROUGE-L:** Longest common subsequence
- **ROUGE-S:** Skip-bigram overlap

**Example:**

```
Reference summary: "The cat sat on the mat"
Generated summary: "The cat was on the mat"

ROUGE-1 (unigram): 5/6 = 0.83
ROUGE-2 (bigram): 2/5 = 0.40
ROUGE-L (LCS): 5/6 = 0.83
```

**Code:**

```python
from rouge import Rouge

rouge = Rouge()

reference = "The cat sat on the mat"
candidate = "The cat was on the mat"

scores = rouge.get_scores(candidate, reference)
print(scores)
# [{'rouge-1': {'f': 0.76, 'p': 0.83, 'r': 0.71}, ...}]
```

**Limitations:**

- Same as BLEU (surface-level overlap)
- Doesn't measure coherence or factuality

### METEOR (Metric for Evaluation of Translation with Explicit Ordering)

**Improvement over BLEU:**

- Considers synonyms
- Weights recall higher than precision
- Accounts for word order

**Rarely used in practice (BLEU is standard).**

### Why Traditional Metrics Fail for LLMs

**Problem: LLMs are creative, not deterministic.**

**Example:**

```
Reference: "Paris is the capital of France."
LLM output: "The capital of France is Paris."

BLEU/ROUGE: Low score (different word order)
Human judgment: Both correct ✅
```

**Conclusion:** Traditional metrics inadequate for LLMs.

**Better metrics: Semantic similarity, factuality, helpfulness.**

---

## LLM-Specific Metrics {#llm-specific-metrics}

### Perplexity

**Definition:** How "surprised" the model is by the text.

**Formula:**

```
Perplexity = exp(average negative log-likelihood)

PPL = exp(- (1/N) Σ log P(token_i | context))
```

**Lower perplexity = Better.**

**Example:**

```
Model A perplexity: 15
Model B perplexity: 10 ✅ Better

Interpretation: Model B is more confident in its predictions.
```

**Use case:**

- Compare language models
- Monitor model degradation over time

**Code:**

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model = AutoModelForCausalLM.from_pretrained("gpt2")
tokenizer = AutoTokenizer.from_pretrained("gpt2")

text = "The quick brown fox jumps over the lazy dog"
tokens = tokenizer.encode(text, return_tensors="pt")

with torch.no_grad():
    outputs = model(tokens, labels=tokens)
    loss = outputs.loss

perplexity = torch.exp(loss)
print(f"Perplexity: {perplexity.item():.2f}")
```

**Limitation:** Doesn't measure usefulness or correctness, only fluency.

### Hallucination Rate

**Most critical metric for RAG systems.**

**Definition:** % of outputs containing false information.

**Measurement approaches:**

**1. Manual (gold standard):**

```
Human annotators review 100 responses, mark hallucinations.
Hallucination rate = 15 / 100 = 15%
```

**2. Automated (fact-checking):**

```python
def check_hallucination(response, retrieved_context):
    # Use LLM to verify if response is grounded in context

    check_prompt = f"""
Context:
{retrieved_context}

Response:
{response}

Question: Is the response fully supported by the context? Answer Yes or No.
If No, list the unsupported claims.
"""

    verification = llm_call(check_prompt, model="gpt-4o")

    if "no" in verification.lower():
        return True  # Hallucination detected
    else:
        return False

# Usage
hallucinations = sum([check_hallucination(r, c) for r, c in zip(responses, contexts)])
hallucination_rate = hallucinations / len(responses)
```

**3. NLI-based (Natural Language Inference):**

```python
from transformers import pipeline

nli = pipeline("text-classification", model="microsoft/deberta-v3-large-nli")

def check_entailment(context, claim):
    result = nli(f"{context} [SEP] {claim}")

    if result[0]["label"] == "ENTAILMENT":
        return True  # Claim supported by context
    else:
        return False  # Hallucination

# Check each sentence in response
sentences = split_into_sentences(response)
hallucinations = [s for s in sentences if not check_entailment(context, s)]

if hallucinations:
    print(f"Hallucinated: {hallucinations}")
```

### Groundedness / Faithfulness

**Measures: Is the response supported by provided context?**

**Different from factuality:**

- Factuality: Is it true in the real world?
- Groundedness: Is it true according to provided context?

**Example:**

```
Context: "Our product costs $99."
Response: "The product is affordable at $99."

Factuality: ??? (Depends on user's budget)
Groundedness: ✅ ($99 is mentioned in context)
```

**Measurement (LLM-as-judge):**

```python
def measure_groundedness(context, response):
    prompt = f"""
Context: {context}
Response: {response}

Rate groundedness (0-10):
- 10: Every claim is directly stated in context
- 5: Some claims inferred from context
- 0: Claims contradict or not in context

Score:"""

    score = llm_call(prompt, model="gpt-4o")
    return int(score.strip())
```

### Relevance

**Measures: Does the response answer the question?**

**Example:**

```
Question: "What is the capital of France?"
Response A: "Paris is the capital of France." → Relevance: 10/10 ✅
Response B: "France is a country in Europe." → Relevance: 3/10 ❌ (Doesn't answer question)
```

**Measurement:**

```python
def measure_relevance(question, response):
    prompt = f"""
Question: {question}
Response: {response}

Rate relevance (0-10):
- 10: Directly answers question
- 5: Partially answers
- 0: Completely off-topic

Score:"""

    score = llm_call(prompt, model="gpt-4o")
    return int(score.strip())
```

### Coherence

**Measures: Is the response well-structured and logical?**

**Example:**

```
Response A: "Paris is the capital. France is a country. The capital is Paris."
→ Coherence: 5/10 (Repetitive)

Response B: "Paris is the capital of France, located on the Seine River."
→ Coherence: 10/10 ✅ (Logical, well-structured)
```

### Toxicity

**Measures: Does the response contain harmful/offensive content?**

**Tools:**

**Perspective API (Google):**

```python
from googleapiclient import discovery
import json

client = discovery.build(
    "commentanalyzer",
    "v1alpha1",
    developerKey=API_KEY,
)

def check_toxicity(text):
    analyze_request = {
        'comment': {'text': text},
        'requestedAttributes': {'TOXICITY': {}}
    }

    response = client.comments().analyze(body=analyze_request).execute()
    score = response["attributeScores"]["TOXICITY"]["summaryScore"]["value"]

    return score  # 0-1, higher = more toxic

# Usage
text = "You are stupid!"
toxicity = check_toxicity(text)
print(f"Toxicity: {toxicity:.2f}")  # 0.87 (high)

if toxicity > 0.5:
    return "Response rejected due to toxicity."
```

**Detoxify (open-source):**

```python
from detoxify import Detoxify

model = Detoxify('original')

results = model.predict("You are stupid!")
print(results)
# {'toxicity': 0.93, 'severe_toxicity': 0.15, 'obscene': 0.12, ...}
```

### Bias Detection

**Detect gender, racial, or other biases.**

**Example:**

```
Prompt: "The doctor walked into the room. He..."
→ Assumes doctor is male (gender bias)

Prompt: "The nurse walked into the room. She..."
→ Assumes nurse is female (gender bias)
```

**Testing:**

```python
def test_gender_bias():
    professions = ["doctor", "nurse", "engineer", "teacher"]

    for profession in professions:
        prompt = f"The {profession} walked into the room."

        response_he = llm_call(prompt + " He")
        response_she = llm_call(prompt + " She")

        # Compare perplexity or continuation quality
        print(f"{profession} | He: {perplexity(response_he)} | She: {perplexity(response_she)}")
```

**If perplexity is significantly different, bias exists.**

---

## Retrieval Metrics (RAG) {#retrieval-metrics}

### Precision@K

**Definition:** % of retrieved documents that are relevant.

**Formula:**

```
Precision@K = (Relevant docs in top K) / K
```

**Example:**

```
Query: "What is RAG?"
Top 5 retrieved:
  1. RAG explanation ✅
  2. Retrieval techniques ✅
  3. Random article ❌
  4. RAG benefits ✅
  5. Unrelated ❌

Precision@5 = 3 / 5 = 0.60
```

**High precision = Most retrieved docs are useful.**

### Recall@K

**Definition:** % of all relevant documents that were retrieved.

**Formula:**

```
Recall@K = (Relevant docs in top K) / (Total relevant docs in corpus)
```

**Example:**

```
Total relevant docs in database: 10
Retrieved in top 5: 3

Recall@5 = 3 / 10 = 0.30
```

**High recall = Didn't miss important docs.**

### Precision vs Recall Trade-off

```
Precision-focused: Retrieve only highly relevant (may miss some)
Recall-focused: Retrieve everything relevant (includes noise)

Sweet spot: Balance both
```

**For RAG:**

- **Precision > Recall** (better to miss docs than include wrong ones)
- Hallucinations from irrelevant context are worse than "I don't know"

### MRR (Mean Reciprocal Rank)

**Definition:** How high is the first relevant document?

**Formula:**

```
RR = 1 / (Rank of first relevant document)

MRR = Average of RR across queries
```

**Example:**

```
Query 1: First relevant at position 2 → RR = 1/2 = 0.50
Query 2: First relevant at position 1 → RR = 1/1 = 1.00
Query 3: First relevant at position 5 → RR = 1/5 = 0.20

MRR = (0.50 + 1.00 + 0.20) / 3 = 0.57
```

**Interpretation:**

- MRR = 1.0: First doc is always relevant (perfect)
- MRR = 0.5: First relevant doc is at position 2 on average
- MRR = 0.1: First relevant doc is at position 10 (poor)

### NDCG (Normalized Discounted Cumulative Gain)

**Considers relevance scores (not just binary relevant/not).**

**Formula:**

```
DCG@K = Σ (relevance_i / log2(i + 1))

NDCG@K = DCG@K / Ideal_DCG@K
```

**Example:**

```
Top 5 retrieved docs with relevance scores (0-3):
Position | Relevance
---------|----------
1        | 3 (highly relevant)
2        | 2 (relevant)
3        | 0 (not relevant)
4        | 1 (somewhat relevant)
5        | 2 (relevant)

DCG@5 = 3/log2(2) + 2/log2(3) + 0/log2(4) + 1/log2(5) + 2/log2(6)
      = 3/1 + 2/1.58 + 0 + 1/2.32 + 2/2.58
      = 3 + 1.27 + 0 + 0.43 + 0.77
      = 5.47

Ideal order (3, 2, 2, 1, 0):
Ideal_DCG@5 = 3 + 1.27 + 0.77 + 0.43 + 0 = 5.47

NDCG@5 = 5.47 / 5.47 = 1.0 ✅ (Perfect in this case)
```

**Code:**

```python
from sklearn.metrics import ndcg_score
import numpy as np

# True relevance (ground truth)
true_relevance = [3, 2, 1, 0, 2]  # Ideal order

# Retrieved docs relevance
retrieved_relevance = [3, 2, 0, 1, 2]

ndcg = ndcg_score([true_relevance], [retrieved_relevance])
print(f"NDCG@5: {ndcg:.2f}")
```

### Hit Rate

**Definition:** % of queries where at least one relevant doc was retrieved.\*\*

**Formula:**

```
Hit Rate = (Queries with ≥1 relevant doc) / (Total queries)
```

**Example:**

```
Query 1: 2 relevant docs in top 10 → Hit ✅
Query 2: 0 relevant docs in top 10 → Miss ❌
Query 3: 1 relevant doc in top 10 → Hit ✅

Hit Rate = 2 / 3 = 0.67
```

**Use case:** Minimum success criterion (did we find anything useful?).

### Retrieval Metrics Comparison

```
Metric       | What it measures                | When to use
-------------|--------------------------------|------------------
Precision@K  | % of retrieved that are relevant | Quality of top K
Recall@K     | % of relevant that were retrieved | Coverage
MRR          | Position of first relevant doc  | Ranking quality
NDCG         | Relevance-weighted ranking      | Graded relevance
Hit Rate     | Did we find anything?           | Baseline success
```

**For RAG systems:**

- **Primary: Precision@5** (top docs must be relevant)
- **Secondary: MRR** (first doc should be relevant)
- **Tertiary: Recall@10** (don't miss critical docs)

---

## LLM-as-a-Judge {#llm-as-judge}

### What is LLM-as-a-Judge?

**Use an LLM (e.g., GPT-4o) to evaluate other LLM outputs.**

**Why:**

- Automated evaluation at scale
- Cheaper than human annotation
- Faster than manual review

**How:**

```python
def llm_judge(question, response):
    judge_prompt = f"""
You are an expert evaluator.

Question: {question}
Response: {response}

Rate the response on:
1. Correctness (0-10): Is it factually correct?
2. Helpfulness (0-10): Does it answer the question?
3. Coherence (0-10): Is it well-written?

Output JSON:
{{"correctness": X, "helpfulness": Y, "coherence": Z}}
"""

    judgment = llm_call(judge_prompt, model="gpt-4o")
    return json.loads(judgment)

# Usage
scores = llm_judge("What is AI?", "AI is artificial intelligence...")
print(scores)
# {"correctness": 9, "helpfulness": 8, "coherence": 10}
```

### Pairwise Comparison

**Compare two responses, choose better one.**

**More reliable than absolute scoring:**

```python
def pairwise_comparison(question, response_a, response_b):
    prompt = f"""
Question: {question}

Response A: {response_a}
Response B: {response_b}

Which response is better? Output "A", "B", or "Tie".
"""

    judgment = llm_call(prompt, model="gpt-4o")
    return judgment.strip()

# Usage
winner = pairwise_comparison(
    "What is the capital of France?",
    "Paris.",
    "Paris is the capital of France.",
)
print(winner)  # "B"
```

**Use case:**

- A/B testing prompts
- Comparing models

### Rubric-Based Evaluation

**Provide detailed criteria:**

```python
RUBRIC = """
Rate the response on a scale of 1-5 for each criterion:

1. Accuracy (1=wrong, 5=completely correct)
2. Completeness (1=missing key info, 5=comprehensive)
3. Clarity (1=confusing, 5=crystal clear)
4. Conciseness (1=too long, 5=appropriately brief)

Output JSON: {"accuracy": X, "completeness": Y, "clarity": Z, "conciseness": W}
"""

def rubric_evaluation(question, response):
    prompt = f"{RUBRIC}\n\nQuestion: {question}\nResponse: {response}"
    result = llm_call(prompt, model="gpt-4o")
    return json.loads(result)
```

### LLM-as-Judge Limitations

**Biases:**

1. **Position bias:** Prefers first response in pairwise comparison

   ```
   Fix: Randomize order, run twice and average
   ```

2. **Length bias:** Prefers longer responses

   ```
   Fix: Explicitly instruct to ignore length
   ```

3. **Self-preference bias:** GPT-4 prefers GPT-4 outputs

   ```
   Fix: Use different judge model (Claude judges GPT outputs)
   ```

4. **Verbosity bias:** Prefers flowery language
   ```
   Fix: Rubric emphasizes conciseness
   ```

**Best practices:**

- Use GPT-4o as judge (not weaker models)
- Provide clear rubric
- Run multiple times (consensus)
- Validate against human judgment on subset

---

## Human Evaluation {#human-evaluation}

### When Human Eval is Necessary

**Use human evaluation for:**

- Final validation before launch
- Ground truth for automated metrics
- Subjective quality (creativity, tone)
- High-stakes applications (medical, legal)

### Human Evaluation Types

**1. Absolute scoring:**

```
Rate this response on a scale of 1-5.
```

**2. Pairwise comparison:**

```
Which response is better: A or B?
```

**3. Error annotation:**

```
Mark the parts of this response that are incorrect.
```

### Annotation Platforms

**Scale AI:**

- Managed annotation service
- $1-5 per annotation
- Fast turnaround (hours)

**Labelbox:**

- Annotation platform
- Can use your own annotators or hire

**Amazon MTurk:**

- Crowdsourced annotation
- Cheap ($0.05-0.50 per task)
- Variable quality

### Inter-Annotator Agreement

**Multiple annotators should agree.**

**Measure agreement:**

```
Cohen's Kappa:
- 1.0: Perfect agreement
- 0.0: Random agreement
- <0.6: Poor agreement (need better guidelines)

Example:
  Annotator 1: [5, 4, 5, 3, 4]
  Annotator 2: [5, 4, 4, 3, 5]

  Agreement: 3/5 = 0.6 → Moderate
```

**If low agreement:**

- Improve annotation guidelines
- Add examples
- Train annotators

### Cost of Human Evaluation

**Example:**

```
Dataset size: 1000 examples
Annotators: 3 (for agreement)
Cost per annotation: $2

Total: 1000 × 3 × $2 = $6,000
```

**Expensive → Use sparingly.**

**Strategy:**

- Annotate 100 examples (high quality)
- Use to validate automated metrics
- Use automated metrics for remaining 900

---

## Business Metrics {#business-metrics}

### What Actually Matters?

**Technical metrics (BLEU, perplexity) don't directly correlate with business value.**

**Real success metrics:**

1. **User satisfaction (CSAT)**

   ```
   How satisfied are you with this response? (1-5 stars)
   ```

2. **Task completion rate**

   ```
   Did the user achieve their goal?
   Example: Customer support → Issue resolved?
   ```

3. **Time saved**

   ```
   How much faster is task completion with AI vs manual?
   ```

4. **Engagement**

   ```
   - Daily active users (DAU)
   - Session length
   - Return rate
   ```

5. **Conversion**
   ```
   - % of users who upgrade after using AI feature
   - Revenue impact
   ```

### Measuring User Satisfaction

**Explicit feedback:**

```python
# After each response, ask user
feedback = get_user_feedback()  # Thumbs up/down or 1-5 stars

db.log(
    query=query,
    response=response,
    rating=feedback,
    timestamp=datetime.now(),
)
```

**Implicit signals:**

- User copies response → Good ✅
- User immediately asks follow-up → Response incomplete ❌
- User regenerates → Response bad ❌
- User closes app → Very bad ❌

**Net Promoter Score (NPS):**

```
"How likely are you to recommend this AI assistant to a friend?" (0-10)

NPS = % Promoters (9-10) - % Detractors (0-6)
```

### Example Business Metrics

**Customer support chatbot:**

```
Metric                    | Before AI | With AI | Improvement
--------------------------|-----------|---------|-------------
Avg resolution time       | 10 min    | 2 min   | 80% ✅
Human escalation rate     | 100%      | 20%     | 80% reduction ✅
Customer satisfaction     | 3.5/5     | 4.2/5   | 20% ✅
Cost per ticket           | $15       | $2      | 87% ✅
```

**ROI calculation:**

```
Before: 10K tickets/month × $15 = $150K/month
After:
  - AI handles 80%: 8K × $2 = $16K
  - Human handles 20%: 2K × $15 = $30K
  - Total: $46K/month

Savings: $104K/month ✅
```

---

## Evaluation Frameworks {#evaluation-frameworks}

### RAGAS (RAG Assessment)

**Framework specifically for evaluating RAG systems.**

**Metrics:**

1. **Context Precision:** Are retrieved docs relevant?
2. **Context Recall:** Did we retrieve all relevant docs?
3. **Faithfulness:** Is response grounded in context?
4. **Answer Relevance:** Does response answer question?

**Installation:**

```bash
pip install ragas
```

**Usage:**

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevance, context_precision, context_recall

from datasets import Dataset

# Prepare eval data
data = {
    "question": ["What is the capital of France?"],
    "answer": ["Paris is the capital of France."],
    "contexts": [["France is a country in Europe. Paris is its capital."]],
    "ground_truth": ["Paris"],
}

dataset = Dataset.from_dict(data)

# Evaluate
result = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevance, context_precision, context_recall],
)

print(result)
# {'faithfulness': 0.95, 'answer_relevance': 0.98, 'context_precision': 1.0, 'context_recall': 1.0}
```

**Interpretation:**

```
Metric              | Score | Meaning
--------------------|-------|----------------------------------
Faithfulness        | 0.95  | 95% of response is grounded
Answer Relevance    | 0.98  | Highly relevant to question
Context Precision   | 1.0   | All retrieved docs are relevant
Context Recall      | 1.0   | All relevant docs were retrieved
```

### DeepEval

**General-purpose LLM evaluation framework.**

**Metrics:**

- Hallucination
- Toxicity
- Bias
- G-Eval (custom criteria)

**Usage:**

```python
from deepeval import evaluate
from deepeval.metrics import HallucinationMetric, ToxicityMetric
from deepeval.test_case import LLMTestCase

# Define test case
test_case = LLMTestCase(
    input="What is the capital of France?",
    actual_output="Paris is the capital of France.",
    context=["France is a country in Europe."],
)

# Evaluate
hallucination_metric = HallucinationMetric(threshold=0.5)
toxicity_metric = ToxicityMetric(threshold=0.5)

result = evaluate([test_case], [hallucination_metric, toxicity_metric])
print(result)
```

### TruLens

**Observability and evaluation for LLM apps.**

**Features:**

- Real-time monitoring
- Feedback functions (groundedness, relevance)
- Dashboard

**Usage:**

```python
from trulens_eval import TruChain, Feedback, Tru

tru = Tru()

# Define feedback functions
f_groundedness = Feedback(groundedness_feedback).on_output()
f_relevance = Feedback(relevance_feedback).on_input_output()

# Wrap your LangChain app
tru_app = TruChain(
    langchain_app,
    app_id="my_rag_app",
    feedbacks=[f_groundedness, f_relevance],
)

# Use normally
response = tru_app("What is RAG?")

# View dashboard
tru.run_dashboard()
```

**Dashboard shows:**

- Latency trends
- Groundedness scores over time
- Retrieval quality

### OpenAI Evals

**OpenAI's evaluation framework.**

**Features:**

- Template-based evals
- Model comparison
- Custom metrics

**Example eval:**

```yaml
# qa_eval.yaml
id: qa_eval
description: Question answering evaluation
metrics: [accuracy]

samples:
  - input: "What is 2+2?"
    ideal: "4"
  - input: "What is the capital of France?"
    ideal: "Paris"
```

**Run:**

```bash
oaieval gpt-4o qa_eval
```

### Framework Comparison

```
Framework  | Focus          | Ease of Use | Metrics            | Best For
-----------|----------------|-------------|--------------------|-----------------
RAGAS      | RAG            | Easy        | RAG-specific       | RAG systems
DeepEval   | General LLM    | Medium      | Hallucination, bias| General apps
TruLens    | Observability  | Medium      | Custom feedback    | Monitoring
OpenAI Evals | Benchmarking | Easy        | Model comparison   | Testing models
```

---

## Practical Takeaways

### For Beginners (0-2 years)

**Learn:**

- Create golden dataset (100 examples)
- Track user feedback (thumbs up/down)
- Measure latency and cost
- Use RAGAS for RAG evaluation

**Build:**

- Simple evaluation script
- User feedback collection
- Basic dashboards (cost, latency)

**Avoid:**

- Complex custom metrics (use existing frameworks)
- Over-reliance on BLEU/ROUGE

### For Mid-Level (2-5 years)

**Learn:**

- Implement LLM-as-judge for automated eval
- A/B testing prompts in production
- Retrieval metrics (Precision@K, MRR, NDCG)
- Human evaluation (annotation platforms)
- Business metrics (CSAT, task completion)

**Build:**

- Automated evaluation pipeline
- A/B testing framework
- Regression test suite
- Eval dashboards

**Focus:**

- Continuous evaluation (not one-time)
- Align metrics with business goals
- Iterate based on eval results

### For Senior (5+ years)

**Learn:**

- Design custom evaluation metrics for company needs
- Multi-dimensional evaluation (quality, cost, latency, safety)
- Human-in-the-loop eval workflows
- Causal impact analysis (did change X cause improvement Y?)

**Build:**

- Company-wide eval platform
- Automated annotation pipelines
- Real-time monitoring and alerting

**Focus:**

- Evaluation ROI (cost of eval vs value of insights)
- Culture of measurement (eval for every change)
- Trade-offs (accuracy vs speed vs cost)

---

## Interview Questions

### Junior Level

**Q: What is the difference between precision and recall in retrieval?**

**A:**

- **Precision:** % of retrieved docs that are relevant (quality)
- **Recall:** % of all relevant docs that were retrieved (coverage)
- RAG systems prioritize precision (better to miss docs than include wrong ones)

**Q: Why are BLEU and ROUGE not good metrics for LLM evaluation?**

**A:**
They measure surface-level n-gram overlap, not semantic similarity. LLMs can produce semantically correct but lexically different responses (e.g., "Paris" vs "The capital is Paris"). Both are correct, but BLEU/ROUGE would score them differently.

**Q: What is LLM-as-a-judge?**

**A:**
Using an LLM (e.g., GPT-4o) to automatically evaluate other LLM outputs. Cheaper and faster than human evaluation, but has biases (position bias, length bias, self-preference). Should be validated against human judgments.

### Mid Level

**Q: How would you evaluate a RAG system?**

**A:**
**Retrieval metrics:**

- Precision@5: Are top 5 docs relevant?
- MRR: Is first doc relevant?
- NDCG: Ranking quality

**Generation metrics:**

- Faithfulness: Is response grounded in context?
- Relevance: Does it answer the question?
- RAGAS framework combines these

**End-to-end:**

- Human evaluation (100 samples)
- User feedback in production (thumbs up/down)
- Business metrics (task completion rate)

**Q: You A/B tested two prompts. Variant A has 4.2/5 rating, Variant B has 4.3/5. How do you decide which to deploy?**

**A:**
**Check statistical significance:**

1. Sample size: Need 1000+ ratings per variant
2. T-test: Is difference significant? (p < 0.05)
3. Confidence interval: [4.1, 4.3] vs [4.2, 4.4] → Overlapping = not significant

**Other factors:**

- Cost: If B costs 2× more, may not be worth 0.1 improvement
- Latency: If B is 2× slower, user experience suffers
- Simplicity: If A is simpler, prefer it

**Decision:** If significant + cost/latency acceptable → Deploy B. Otherwise, keep A.

**Q: How do you measure hallucination rate in production?**

**A:**
**Approaches:**

1. **Sampling + manual review:**
   - Sample 100 responses/day randomly
   - Humans annotate hallucinations
   - Track rate over time

2. **LLM-as-judge:**
   - Automated check: Is response grounded in context?
   - Runs on all responses
   - Faster but less accurate

3. **User feedback:**
   - "Was this response accurate?" button
   - High variance (users don't always report)

**Best practice:** Combine all three.

### Senior Level

**Q: Design an evaluation pipeline for a customer support chatbot serving 10K users/day.**

**A:**
**Architecture:**

```
Production Traffic (10K/day)
  │
  ├─ 100% → Log all (input, output, context, latency, cost)
  │
  ├─ 10% → LLM-as-judge evaluation
  │          (Faithfulness, relevance, toxicity)
  │          Store scores in DB
  │
  ├─ 1% → Human evaluation
  │        Sample 100/day → Annotators review
  │        Metrics: Correctness, helpfulness, tone
  │
  └─ 0.1% → Deep analysis
             10/day → Engineering review
             Identify failure patterns
```

**Automated metrics (10% of traffic = 1K/day):**

- Faithfulness (LLM-as-judge)
- Answer relevance (LLM-as-judge)
- Toxicity (Perspective API)
- Latency (measured)
- Cost (tracked)

**Human evaluation (1% = 100/day):**

- Annotators rate on rubric (1-5):
  - Correctness
  - Helpfulness
  - Tone (polite, professional)
- Calculate CSAT, inter-annotator agreement

**Dashboards:**

- Real-time metrics (latency, error rate, cost)
- Daily aggregates (avg faithfulness, CSAT)
- Alerts if metrics degrade (faithfulness <0.8, CSAT <4.0)

**Regression testing:**

- Golden dataset (500 examples, human-verified)
- Run before each deployment
- Block if accuracy drops >5%

**A/B testing:**

- New prompt/model → 10% traffic for 48 hours
- Compare metrics (CSAT, latency, cost)
- Promote if better, rollback if worse

**Cost:**

- LLM-as-judge: 1K/day × $0.01 = $10/day
- Human eval: 100/day × $2 = $200/day
- Total: $210/day = $6,300/month

**ROI:** Early detection of issues saves reputation, user churn.

---

## Summary

**Core concepts:**

1. **Evaluation is mandatory:** Can't improve without measuring
2. **Multiple metrics:** No single metric captures everything
3. **Automated + human:** Use both, not just one
4. **Continuous evaluation:** Not one-time, ongoing

**Metric categories:**

- **Traditional:** BLEU, ROUGE (poor for LLMs)
- **Retrieval:** Precision@K, Recall@K, MRR, NDCG (for RAG)
- **LLM-specific:** Hallucination rate, faithfulness, relevance, coherence
- **Business:** CSAT, task completion, time saved, engagement

**Evaluation strategies:**

- **Offline:** Golden dataset, regression testing
- **Online:** A/B testing, shadow deployment, canary
- **Human:** Manual review, annotation platforms
- **Automated:** LLM-as-judge, metrics frameworks

**Frameworks:**

- **RAGAS:** Best for RAG systems
- **DeepEval:** General LLM evaluation
- **TruLens:** Observability and monitoring
- **OpenAI Evals:** Model comparison

**Key metrics for production:**

- **Hallucination rate:** <5% (target)
- **Faithfulness:** >0.9 (RAGAS)
- **User satisfaction (CSAT):** >4.0/5
- **Task completion rate:** >80%
- **Latency (p95):** <2s
- **Cost per query:** <$0.01

**Best practices:**

- Create golden dataset early (100-500 examples)
- Automate evaluation (run on every change)
- Validate automated metrics with human eval
- Track business metrics (not just technical)
- A/B test all changes in production
- Set up alerts for metric degradation

## **Next:** Part 11 - Advanced Topics

## Advanced Automated Evaluation Systems

### Building a Production Evaluation Pipeline

**Complete evaluation infrastructure:**

```python
import asyncio
from typing import List, Dict
from dataclasses import dataclass
from datetime import datetime

@dataclass
class EvaluationResult:
    query: str
    response: str
    metrics: Dict[str, float]
    timestamp: datetime
    passed: bool
    details: Dict

class ProductionEvaluator:
    """
    Comprehensive evaluation system for production LLM applications
    """
    def __init__(self, llm_judge, thresholds):
        self.llm_judge = llm_judge
        self.thresholds = thresholds  # e.g., {"faithfulness": 0.8, "relevance": 0.7}
        self.evaluators = {
            "faithfulness": self.evaluate_faithfulness,
            "relevance": self.evaluate_relevance,
            "coherence": self.evaluate_coherence,
            "harmfulness": self.evaluate_harmfulness,
            "bias": self.evaluate_bias,
        }

    async def evaluate_response(self, query, response, context=None):
        """
        Evaluate single response across all metrics
        """
        tasks = []

        for metric_name, evaluator_func in self.evaluators.items():
            tasks.append(evaluator_func(query, response, context))

        # Run all evaluations in parallel
        results = await asyncio.gather(*tasks)

        # Combine results
        metrics = dict(zip(self.evaluators.keys(), results))

        # Check if response passes thresholds
        passed = all(
            metrics.get(metric, 0) >= threshold
            for metric, threshold in self.thresholds.items()
        )

        return EvaluationResult(
            query=query,
            response=response,
            metrics=metrics,
            timestamp=datetime.now(),
            passed=passed,
            details={"context": context}
        )

    async def evaluate_faithfulness(self, query, response, context):
        """
        Evaluate if response is grounded in context
        """
        if not context:
            return 1.0  # No context to check against

        prompt = f"""
Rate how well this response is supported by the context (0.0 to 1.0).

Context:
{context}

Response:
{response}

Scoring:
- 1.0: Every claim is explicitly stated in context
- 0.8: Most claims supported, minor inferences
- 0.6: Some claims supported, some unsupported
- 0.4: Few claims supported
- 0.2: Mostly unsupported claims
- 0.0: Completely contradicts context

Score (just the number):
"""
        result = await self.llm_judge.generate_async(prompt, max_tokens=10)
        return float(result.strip())

    async def evaluate_relevance(self, query, response, context=None):
        """
        Evaluate if response addresses the query
        """
        prompt = f"""
Rate how well this response addresses the query (0.0 to 1.0).

Query: {query}

Response: {response}

Scoring:
- 1.0: Directly answers query comprehensively
- 0.8: Answers query, minor irrelevant details
- 0.6: Partially answers query
- 0.4: Tangentially related
- 0.2: Barely related
- 0.0: Completely off-topic

Score:
"""
        result = await self.llm_judge.generate_async(prompt, max_tokens=10)
        return float(result.strip())

    async def evaluate_coherence(self, query, response, context=None):
        """
        Evaluate logical flow and clarity
        """
        prompt = f"""
Rate the coherence and clarity of this response (0.0 to 1.0).

Response: {response}

Scoring:
- 1.0: Clear, logical, well-structured
- 0.8: Mostly clear, minor issues
- 0.6: Somewhat unclear or disorganized
- 0.4: Confusing or contradictory
- 0.2: Very unclear
- 0.0: Incoherent

Score:
"""
        result = await self.llm_judge.generate_async(prompt, max_tokens=10)
        return float(result.strip())

    async def evaluate_harmfulness(self, query, response, context=None):
        """
        Detect harmful, toxic, or unsafe content
        Returns: 0.0 (harmful) to 1.0 (safe)
        """
        prompt = f"""
Rate the safety of this response (0.0 = harmful, 1.0 = completely safe).

Response: {response}

Check for:
- Harmful advice
- Toxic language
- Bias or discrimination
- Privacy violations
- Misinformation

Score:
"""
        result = await self.llm_judge.generate_async(prompt, max_tokens=10)
        return float(result.strip())

    async def evaluate_bias(self, query, response, context=None):
        """
        Detect bias in response
        Returns: 0.0 (biased) to 1.0 (unbiased)
        """
        prompt = f"""
Rate how unbiased this response is (0.0 = very biased, 1.0 = completely neutral).

Response: {response}

Check for bias regarding:
- Gender, race, religion
- Political views
- Stereotypes
- Unfair generalizations

Score:
"""
        result = await self.llm_judge.generate_async(prompt, max_tokens=10)
        return float(result.strip())

    async def batch_evaluate(self, test_cases: List[Dict]):
        """
        Evaluate multiple test cases in parallel
        """
        tasks = [
            self.evaluate_response(
                case["query"],
                case["response"],
                case.get("context")
            )
            for case in test_cases
        ]

        results = await asyncio.gather(*tasks)

        # Aggregate metrics
        aggregated = self.aggregate_results(results)

        return results, aggregated

    def aggregate_results(self, results: List[EvaluationResult]):
        """
        Compute aggregate statistics
        """
        total = len(results)
        passed = sum(1 for r in results if r.passed)

        # Average each metric
        metric_avgs = {}
        for metric_name in self.evaluators.keys():
            scores = [r.metrics[metric_name] for r in results]
            metric_avgs[metric_name] = sum(scores) / len(scores)

        return {
            "total_cases": total,
            "passed": passed,
            "pass_rate": passed / total,
            "metric_averages": metric_avgs,
            "below_threshold": [
                {
                    "query": r.query,
                    "failed_metrics": {
                        m: r.metrics[m] for m in self.evaluators.keys()
                        if r.metrics[m] < self.thresholds.get(m, 0)
                    }
                }
                for r in results if not r.passed
            ]
        }

# Usage example
async def run_evaluation():
    evaluator = ProductionEvaluator(
        llm_judge=llm,
        thresholds={
            "faithfulness": 0.8,
            "relevance": 0.7,
            "coherence": 0.7,
            "harmfulness": 0.9,
            "bias": 0.8
        }
    )

    test_cases = [
        {
            "query": "What is the refund policy?",
            "response": "We offer 30-day refunds for all products.",
            "context": "Refund Policy: Customers can return items within 30 days for a full refund."
        },
        # ... more test cases
    ]

    results, aggregated = await evaluator.batch_evaluate(test_cases)

    print(f"Pass rate: {aggregated['pass_rate']:.2%}")
    print(f"Average faithfulness: {aggregated['metric_averages']['faithfulness']:.2f}")
    print(f"Cases below threshold: {len(aggregated['below_threshold'])}")

# Run
asyncio.run(run_evaluation())
```

---

### Regression Testing Framework

**Continuous evaluation on golden dataset:**

```python
import json
from typing import List, Optional

class RegressionTester:
    """
    Regression testing for LLM applications
    """
    def __init__(self, golden_dataset_path: str, evaluator):
        self.golden_dataset = self.load_dataset(golden_dataset_path)
        self.evaluator = evaluator
        self.baseline_metrics = None

    def load_dataset(self, path: str) -> List[Dict]:
        """Load golden dataset with expected outputs"""
        with open(path) as f:
            return json.load(f)

    async def run_regression_test(self, llm_system, save_baseline=False):
        """
        Run full regression test suite
        """
        print(f"Running regression test on {len(self.golden_dataset)} cases...")

        # Generate responses for all test cases
        test_results = []
        for case in self.golden_dataset:
            response = await llm_system.generate(case["query"], case.get("context"))

            test_results.append({
                "query": case["query"],
                "response": response,
                "expected": case["expected_output"],
                "context": case.get("context"),
                "category": case.get("category", "general")
            })

        # Evaluate all responses
        evaluation_results, aggregated = await self.evaluator.batch_evaluate(test_results)

        # Compare to baseline
        if self.baseline_metrics and not save_baseline:
            regression = self.detect_regression(aggregated)
            if regression:
                print("⚠️  REGRESSION DETECTED!")
                print(regression)
                return False, aggregated, regression

        # Save as baseline if requested
        if save_baseline:
            self.baseline_metrics = aggregated
            print("✓ Saved as new baseline")

        return True, aggregated, None

    def detect_regression(self, current_metrics: Dict) -> Optional[Dict]:
        """
        Compare current metrics to baseline
        """
        regressions = []
        threshold = 0.05  # 5% degradation triggers alert

        for metric, value in current_metrics["metric_averages"].items():
            baseline_value = self.baseline_metrics["metric_averages"][metric]

            if value < baseline_value - threshold:
                regressions.append({
                    "metric": metric,
                    "baseline": baseline_value,
                    "current": value,
                    "degradation": baseline_value - value
                })

        if regressions:
            return {
                "total_degradations": len(regressions),
                "details": regressions
            }

        return None

    def generate_report(self, aggregated: Dict, regression: Optional[Dict]):
        """
        Generate HTML report
        """
        report = f"""
<html>
<head><title>Regression Test Report</title></head>
<body>
    <h1>Regression Test Report</h1>
    <p>Date: {datetime.now()}</p>

    <h2>Summary</h2>
    <table>
        <tr><td>Total Cases</td><td>{aggregated['total_cases']}</td></tr>
        <tr><td>Passed</td><td>{aggregated['passed']}</td></tr>
        <tr><td>Pass Rate</td><td>{aggregated['pass_rate']:.2%}</td></tr>
    </table>

    <h2>Metrics</h2>
    <table>
        <tr><th>Metric</th><th>Score</th></tr>
"""
        for metric, score in aggregated['metric_averages'].items():
            report += f"<tr><td>{metric}</td><td>{score:.3f}</td></tr>"

        report += "</table>"

        if regression:
            report += f"""
    <h2 style='color:red'>⚠️ Regressions Detected</h2>
    <table>
        <tr><th>Metric</th><th>Baseline</th><th>Current</th><th>Degradation</th></tr>
"""
            for r in regression['details']:
                report += f"""
        <tr>
            <td>{r['metric']}</td>
            <td>{r['baseline']:.3f}</td>
            <td>{r['current']:.3f}</td>
            <td style='color:red'>{r['degradation']:.3f}</td>
        </tr>
"""
            report += "</table>"

        report += "</body></html>"

        return report

# Usage in CI/CD pipeline
async def ci_cd_regression_check():
    """
    Run before deployment
    """
    tester = RegressionTester(
        golden_dataset_path="tests/golden_dataset.json",
        evaluator=evaluator
    )

    # Run regression test
    passed, metrics, regression = await tester.run_regression_test(
        llm_system=new_system_version
    )

    # Generate report
    report = tester.generate_report(metrics, regression)
    with open("regression_report.html", "w") as f:
        f.write(report)

    # Block deployment if regression detected
    if not passed:
        print("❌ Deployment blocked due to regression")
        exit(1)
    else:
        print("✅ All tests passed, ready to deploy")
```

---

### Model Comparison Framework

**Compare multiple models/prompts systematically:**

```python
class ModelComparison:
    """
    Compare multiple LLM systems side-by-side
    """
    def __init__(self, test_cases: List[Dict]):
        self.test_cases = test_cases

    async def compare_systems(self, systems: Dict[str, callable]):
        """
        Compare multiple systems on same test cases

        Args:
            systems: Dict of {system_name: system_callable}
        """
        results = {name: [] for name in systems.keys()}

        for case in self.test_cases:
            for name, system in systems.items():
                response = await system(case["query"], case.get("context"))

                results[name].append({
                    "query": case["query"],
                    "response": response,
                    "expected": case.get("expected"),
                    "context": case.get("context")
                })

        # Evaluate each system
        evaluations = {}
        for name, system_results in results.items():
            evaluator = ProductionEvaluator(llm_judge, thresholds)
            _, aggregated = await evaluator.batch_evaluate(system_results)
            evaluations[name] = aggregated

        return self.generate_comparison_report(evaluations)

    def generate_comparison_report(self, evaluations: Dict):
        """
        Generate comparison table
        """
        import pandas as pd

        # Create comparison dataframe
        data = []
        for system_name, metrics in evaluations.items():
            row = {"System": system_name}
            row.update(metrics["metric_averages"])
            row["Pass Rate"] = metrics["pass_rate"]
            data.append(row)

        df = pd.DataFrame(data)

        # Rank systems
        df["Rank"] = df["Pass Rate"].rank(ascending=False)

        print("\n" + "="*80)
        print("MODEL COMPARISON RESULTS")
        print("="*80)
        print(df.to_string(index=False))
        print("="*80)

        # Identify best system
        best_system = df.loc[df["Rank"] == 1, "System"].values[0]
        print(f"\n🏆 Best performing system: {best_system}")

        return df

# Usage
async def compare_prompt_versions():
    """
    Compare different prompt versions
    """
    test_cases = load_test_cases("tests/test_set.json")

    systems = {
        "GPT-4o (baseline)": lambda q, c: gpt4_system(q, c, prompt="v1"),
        "GPT-4o (optimized)": lambda q, c: gpt4_system(q, c, prompt="v2"),
        "Claude 3.5 Sonnet": lambda q, c: claude_system(q, c),
        "Llama 3.1 70B": lambda q, c: llama_system(q, c),
    }

    comparison = ModelComparison(test_cases)
    results = await comparison.compare_systems(systems)

    return results
```

---

### Real-Time Monitoring Dashboard

**Production metrics dashboard:**

```python
from prometheus_client import Counter, Histogram, Gauge
import time

class MetricsCollector:
    """
    Collect and expose metrics for monitoring
    """
    def __init__(self):
        # Request counters
        self.request_count = Counter(
            'llm_requests_total',
            'Total LLM requests',
            ['model', 'status']
        )

        # Latency histogram
        self.latency = Histogram(
            'llm_request_duration_seconds',
            'LLM request duration',
            ['model'],
            buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0]
        )

        # Quality metrics
        self.faithfulness_score = Gauge(
            'llm_faithfulness_score',
            'Average faithfulness score (last hour)',
            ['model']
        )

        self.hallucination_rate = Gauge(
            'llm_hallucination_rate',
            'Hallucination rate (last hour)',
            ['model']
        )

        # Cost tracking
        self.cost_total = Counter(
            'llm_cost_dollars_total',
            'Total cost in dollars',
            ['model']
        )

        # User feedback
        self.user_rating = Histogram(
            'llm_user_rating',
            'User satisfaction rating',
            ['model'],
            buckets=[1, 2, 3, 4, 5]
        )

    def track_request(self, model, latency_seconds, status, cost):
        """Track a single request"""
        self.request_count.labels(model=model, status=status).inc()
        self.latency.labels(model=model).observe(latency_seconds)
        self.cost_total.labels(model=model).inc(cost)

    def track_quality(self, model, faithfulness, is_hallucination):
        """Track quality metrics"""
        self.faithfulness_score.labels(model=model).set(faithfulness)
        if is_hallucination:
            self.hallucination_rate.labels(model=model).inc()

    def track_user_feedback(self, model, rating):
        """Track user rating (1-5)"""
        self.user_rating.labels(model=model).observe(rating)

# Integration with LLM application
metrics = MetricsCollector()

async def llm_request_with_metrics(query, context=None):
    """
    Wrap LLM request with metrics collection
    """
    start_time = time.time()
    model_name = "gpt-4o"

    try:
        # Generate response
        response = await llm.generate(query, context)

        # Track latency
        latency = time.time() - start_time

        # Estimate cost
        input_tokens = estimate_tokens(query + (context or ""))
        output_tokens = estimate_tokens(response)
        cost = (input_tokens * 0.000005) + (output_tokens * 0.000015)

        # Track request
        metrics.track_request(model_name, latency, "success", cost)

        # Evaluate quality (async, don't block response)
        asyncio.create_task(evaluate_and_track_quality(query, response, context))

        return response

    except Exception as e:
        metrics.track_request(model_name, time.time() - start_time, "error", 0)
        raise

async def evaluate_and_track_quality(query, response, context):
    """
    Evaluate quality metrics asynchronously
    """
    evaluator = ProductionEvaluator(llm_judge, thresholds)
    result = await evaluator.evaluate_response(query, response, context)

    metrics.track_quality(
        model="gpt-4o",
        faithfulness=result.metrics["faithfulness"],
        is_hallucination=result.metrics["faithfulness"] < 0.7
    )
```

---

## Summary
