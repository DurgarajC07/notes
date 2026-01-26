# Part 14: Tools & Ecosystem

## Table of Contents

1. [Model Providers](#model-providers)
2. [Embedding Models](#embedding-models)
3. [Vector Databases](#vector-databases)
4. [Frameworks & Libraries](#frameworks-libraries)
5. [Observability & Monitoring](#observability-monitoring)
6. [Deployment & Serving](#deployment-serving)
7. [Evaluation Tools](#evaluation-tools)
8. [Development Tools](#development-tools)
9. [Tool Selection Guide](#tool-selection)
10. [Cost Comparison](#cost-comparison)

---

## Model Providers {#model-providers}

### Commercial LLM APIs

**OpenAI**

**Models:**

- **GPT-4o:** Latest flagship, multimodal (text + vision)
- **GPT-4o-mini:** Fast, cheap, 128K context
- **o1-preview/o1-mini:** Reasoning models (chain-of-thought)
- **GPT-3.5-turbo:** Legacy, cheap for simple tasks

**Pricing (per 1M tokens):**

- GPT-4o: $2.50 input, $10 output
- GPT-4o-mini: $0.15 input, $0.60 output
- o1-preview: $15 input, $60 output

**Best for:**

- Function calling (best in class)
- Structured output (JSON mode)
- Wide ecosystem (LangChain, LlamaIndex)

**Weaknesses:**

- Rate limits (strict for free tier)
- Privacy concerns (logs data for 30 days)

---

**Anthropic Claude**

**Models:**

- **Claude 3.5 Sonnet:** Best for coding, analysis
- **Claude 3 Opus:** Most capable, expensive
- **Claude 3 Haiku:** Fast, cheap

**Pricing (per 1M tokens):**

- Sonnet: $3 input, $15 output
- Opus: $15 input, $75 output
- Haiku: $0.25 input, $1.25 output

**Best for:**

- Long context (200K tokens)
- Safety (less toxic, refuses harmful requests)
- Coding (better than GPT-4 for many tasks)

**Weaknesses:**

- Smaller ecosystem
- Function calling less robust than OpenAI

---

**Google Gemini**

**Models:**

- **Gemini 1.5 Pro:** 2M context window
- **Gemini 1.5 Flash:** Fast, cheap
- **Gemini Ultra:** Most capable (coming soon)

**Pricing (per 1M tokens):**

- Pro: $1.25 input, $5 output
- Flash: $0.075 input, $0.30 output

**Best for:**

- Huge context (2M tokens)
- Multimodal (native video understanding)
- Google Cloud integration

**Weaknesses:**

- Fewer tools in ecosystem
- Prompt engineering differs from GPT

---

**Open-Source Models**

**Meta LLaMA 3**

**Models:**

- **LLaMA 3.1 405B:** GPT-4 level performance
- **LLaMA 3.1 70B:** Claude 3 Sonnet level
- **LLaMA 3.1 8B:** Fast, runs on consumer GPU

**Best for:**

- Self-hosting (no API costs)
- Privacy (data stays on-premises)
- Fine-tuning (full model access)

**Weaknesses:**

- Requires GPU infrastructure
- No official support

---

**Tool Use Comparison**

| Model              | Function Calling           | Structured Output | JSON Mode | Context Length |
| ------------------ | -------------------------- | ----------------- | --------- | -------------- |
| **GPT-4o**         | ⭐⭐⭐⭐⭐ Best            | ✅ Yes            | ✅ Yes    | 128K           |
| **Claude 3.5**     | ⭐⭐⭐⭐ Good              | ✅ Yes            | ✅ Yes    | 200K           |
| **Gemini 1.5 Pro** | ⭐⭐⭐ Decent              | ✅ Yes            | ✅ Yes    | 2M             |
| **LLaMA 3.1**      | ⭐⭐⭐ OK (with prompting) | ⚠️ Manual         | ⚠️ Manual | 128K           |

**Recommendation:**

- **Production:** GPT-4o or Claude 3.5 Sonnet
- **Budget:** GPT-4o-mini or Gemini Flash
- **Privacy:** LLaMA 3.1 (self-hosted)
- **Reasoning:** o1-preview (complex problems)

---

## Embedding Models {#embedding-models}

### Commercial Embeddings

**OpenAI**

**Models:**

- **text-embedding-3-large:** 3072 dims, best quality
- **text-embedding-3-small:** 1536 dims, cheap

**Pricing:**

- Large: $0.13 per 1M tokens
- Small: $0.02 per 1M tokens

**Best for:**

- Production RAG (high quality)
- Semantic search

---

**Cohere**

**Models:**

- **embed-english-v3.0:** 1024 dims
- **embed-multilingual-v3.0:** 1024 dims

**Pricing:**

- $0.10 per 1M tokens

**Best for:**

- Multilingual (100+ languages)
- Classification (built-in classifier)

---

**Voyage AI**

**Models:**

- **voyage-large-2:** 1024 dims
- **voyage-code-2:** Code-specialized

**Pricing:**

- $0.12 per 1M tokens

**Best for:**

- Code search (voyage-code-2)
- Finance, legal (domain-specific models)

---

### Open-Source Embeddings

**Sentence-Transformers**

**Models:**

- **all-MiniLM-L6-v2:** 384 dims, fast
- **all-mpnet-base-v2:** 768 dims, quality

**Cost:** Free (self-hosted)

**Best for:**

- Local development
- Budget constraints

**Weaknesses:**

- Lower quality than commercial
- Requires GPU for speed

---

**Embedding Comparison**

| Model                      | Dimensions | Quality    | Speed      | Cost (1M tokens) | Use Case        |
| -------------------------- | ---------- | ---------- | ---------- | ---------------- | --------------- |
| **text-embedding-3-large** | 3072       | ⭐⭐⭐⭐⭐ | Fast       | $0.13            | Production RAG  |
| **text-embedding-3-small** | 1536       | ⭐⭐⭐⭐   | Fastest    | $0.02            | Budget RAG      |
| **Cohere embed-v3**        | 1024       | ⭐⭐⭐⭐   | Fast       | $0.10            | Multilingual    |
| **Voyage AI**              | 1024       | ⭐⭐⭐⭐   | Fast       | $0.12            | Domain-specific |
| **all-mpnet-base-v2**      | 768        | ⭐⭐⭐     | Slow (CPU) | Free             | Local dev       |

**Recommendation:**

- **Production:** text-embedding-3-small (best price/quality)
- **High-quality:** text-embedding-3-large or Voyage AI
- **Budget:** all-mpnet-base-v2 (self-hosted)
- **Multilingual:** Cohere embed-multilingual-v3

---

## Vector Databases {#vector-databases}

### Managed Vector DBs

**Pinecone**

**Pricing:**

- Serverless: $0.25 per 1M queries (pay-as-you-go)
- Pod-based: $70/month (1M vectors, 768 dims)

**Best for:**

- Production RAG (reliable, fast)
- Hybrid search (vector + metadata filtering)

**Pros:**

- Easy to use (no ops)
- Fast (<50ms latency)
- Hybrid search built-in

**Cons:**

- Expensive for large datasets (>10M vectors)

---

**Weaviate**

**Pricing:**

- Serverless: $0.095 per 1M queries
- Self-hosted: Free

**Best for:**

- Self-hosting (Kubernetes)
- Complex filters (GraphQL API)

**Pros:**

- Cheaper than Pinecone (serverless)
- Rich query language

**Cons:**

- More complex to set up

---

**Qdrant**

**Pricing:**

- Cloud: $0.10 per 1M queries
- Self-hosted: Free

**Best for:**

- High-performance (written in Rust)
- Self-hosting

**Pros:**

- Fast (Rust, no GC pauses)
- Free self-hosted option

**Cons:**

- Smaller ecosystem

---

### Database Extensions

**PostgreSQL (pgvector)**

**Pricing:** Free (add-on to PostgreSQL)

**Best for:**

- Existing PostgreSQL users
- Small datasets (<1M vectors)

**Pros:**

- No new database (use existing Postgres)
- ACID transactions

**Cons:**

- Slow for large datasets (>1M vectors)
- Limited scalability

---

**Elasticsearch**

**Pricing:**

- Cloud: $95/month (100GB)
- Self-hosted: Free

**Best for:**

- Hybrid search (keyword + vector)
- Existing Elasticsearch users

**Pros:**

- Battle-tested (logs, metrics)
- Rich full-text search

**Cons:**

- Complex to tune
- Resource-heavy

---

**Vector DB Comparison**

| Database          | Ease of Use | Speed      | Scalability | Cost | Best For      |
| ----------------- | ----------- | ---------- | ----------- | ---- | ------------- |
| **Pinecone**      | ⭐⭐⭐⭐⭐  | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐    | $$$  | Production    |
| **Weaviate**      | ⭐⭐⭐⭐    | ⭐⭐⭐⭐   | ⭐⭐⭐⭐    | $$   | Self-hosted   |
| **Qdrant**        | ⭐⭐⭐⭐    | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐  | $    | High-perf     |
| **pgvector**      | ⭐⭐⭐⭐⭐  | ⭐⭐⭐     | ⭐⭐        | Free | Small scale   |
| **Elasticsearch** | ⭐⭐⭐      | ⭐⭐⭐     | ⭐⭐⭐⭐    | $$   | Hybrid search |

**Recommendation:**

- **Production:** Pinecone (easiest) or Qdrant (best perf)
- **Budget:** Qdrant self-hosted or pgvector
- **Existing Postgres:** pgvector
- **Hybrid search:** Elasticsearch or Weaviate

---

## Frameworks & Libraries {#frameworks-libraries}

### LLM Orchestration

**LangChain**

**Language:** Python, JavaScript

**Best for:**

- RAG (document loaders, retrievers)
- Agents (ReAct, function calling)
- Quick prototyping

**Pros:**

- Largest ecosystem (100+ integrations)
- Good documentation
- Active community

**Cons:**

- Over-abstracted (hard to debug)
- Breaking changes (fast-moving)

**When to use:** Early prototyping, standard RAG

**Example:**

```python
from langchain.vectorstores import Pinecone
from langchain.chat_models import ChatOpenAI

# RAG in 5 lines
vectorstore = Pinecone.from_texts(texts, embeddings)
retriever = vectorstore.as_retriever()
llm = ChatOpenAI(model="gpt-4o")
chain = retriever | llm
response = chain.invoke("What's our refund policy?")
```

---

**LlamaIndex**

**Language:** Python

**Best for:**

- RAG (complex retrieval strategies)
- Knowledge graphs
- Multi-document reasoning

**Pros:**

- RAG-focused (better than LangChain for RAG)
- Advanced retrieval (hybrid, re-ranking)
- Good for enterprise

**Cons:**

- Smaller community than LangChain
- Less agent support

**When to use:** Production RAG, complex retrieval

**Example:**

```python
from llama_index import VectorStoreIndex, SimpleDirectoryReader

# RAG with 3 lines
documents = SimpleDirectoryReader("docs").load_data()
index = VectorStoreIndex.from_documents(documents)
response = index.as_query_engine().query("What's our refund policy?")
```

---

**LangGraph**

**Language:** Python

**Best for:**

- Multi-agent systems
- Complex workflows (state machines)
- Agentic applications

**Pros:**

- State management (checkpointing)
- Human-in-the-loop
- Production-ready (from LangChain team)

**Cons:**

- Steeper learning curve
- Newer (less mature)

**When to use:** Multi-agent, complex workflows

**Example:**

```python
from langgraph.graph import StateGraph

graph = StateGraph(AgentState)
graph.add_node("research", research_agent)
graph.add_node("analyze", analyze_agent)
graph.add_edge("research", "analyze")

workflow = graph.compile()
result = workflow.invoke({"query": "Plan a trip to Paris"})
```

---

**DSPy**

**Language:** Python

**Best for:**

- Prompt optimization (no manual prompt engineering)
- Research (Stanford project)

**Pros:**

- Automatic prompt optimization
- Composable (like PyTorch for prompts)

**Cons:**

- Experimental (not production-ready)
- Steep learning curve

**When to use:** Research, experimentation

---

**Framework Comparison**

| Framework      | Ease of Use | RAG        | Agents     | Production | Best For       |
| -------------- | ----------- | ---------- | ---------- | ---------- | -------------- |
| **LangChain**  | ⭐⭐⭐⭐    | ⭐⭐⭐⭐   | ⭐⭐⭐⭐   | ⭐⭐⭐     | Prototyping    |
| **LlamaIndex** | ⭐⭐⭐⭐    | ⭐⭐⭐⭐⭐ | ⭐⭐⭐     | ⭐⭐⭐⭐   | Production RAG |
| **LangGraph**  | ⭐⭐⭐      | ⭐⭐⭐     | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Agents         |
| **DSPy**       | ⭐⭐        | ⭐⭐⭐⭐   | ⭐⭐       | ⭐⭐       | Research       |

**Recommendation:**

- **Prototyping:** LangChain (fast to build)
- **Production RAG:** LlamaIndex (best retrieval)
- **Agents:** LangGraph (state management)
- **No frameworks:** Use OpenAI/Anthropic SDKs directly (less abstraction)

---

## Observability & Monitoring {#observability-monitoring}

### LLM Observability Tools

**LangSmith**

**Pricing:**

- Free: 5K traces/month
- Pro: $39/month (50K traces)

**Best for:**

- LangChain users (native integration)
- Debugging (trace every LLM call)

**Features:**

- **Tracing:** See every LLM call, latency, tokens
- **Datasets:** Store test cases
- **Evaluation:** Run evals on datasets
- **Prompt versioning:** Track prompt changes

**Pros:**

- Deep LangChain integration
- Easy to set up

**Cons:**

- LangChain-centric (less useful for other frameworks)

---

**Helicone**

**Pricing:**

- Free: 10K requests/month
- Pro: $20/month (100K requests)

**Best for:**

- OpenAI users (proxy-based)
- Cost tracking

**Features:**

- **Cost tracking:** Per-user, per-session
- **Caching:** 10-100x speed, cost savings
- **Rate limiting:** Prevent abuse
- **Analytics:** Latency, tokens, errors

**Pros:**

- Framework-agnostic (proxy-based)
- Simple integration (1 line code change)

**Cons:**

- Adds latency (proxy call)

---

**Arize AI**

**Pricing:**

- Free: 10K predictions/month
- Enterprise: Custom

**Best for:**

- ML teams (familiar with ML observability)
- Drift detection

**Features:**

- **Drift detection:** Prompt drift, output drift
- **Embeddings:** Visualize in 2D (UMAP)
- **Hallucination detection:** Automatic flagging
- **Root cause analysis:** Why did this fail?

**Pros:**

- Advanced analytics
- ML-focused

**Cons:**

- Overkill for simple RAG

---

**Phoenix (Arize Open-Source)**

**Pricing:** Free (self-hosted)

**Best for:**

- Open-source alternative
- Local development

**Features:**

- Tracing (OpenTelemetry)
- Embedding visualization
- Evaluation

**Pros:**

- Free
- No data leaves your server

**Cons:**

- Requires self-hosting

---

**Observability Comparison**

| Tool          | Ease of Setup | Tracing    | Cost Tracking | Evaluation | Best For     |
| ------------- | ------------- | ---------- | ------------- | ---------- | ------------ |
| **LangSmith** | ⭐⭐⭐⭐⭐    | ⭐⭐⭐⭐⭐ | ⭐⭐⭐        | ⭐⭐⭐⭐   | LangChain    |
| **Helicone**  | ⭐⭐⭐⭐⭐    | ⭐⭐⭐⭐   | ⭐⭐⭐⭐⭐    | ⭐⭐⭐     | Cost control |
| **Arize**     | ⭐⭐⭐        | ⭐⭐⭐⭐   | ⭐⭐⭐⭐      | ⭐⭐⭐⭐⭐ | ML teams     |
| **Phoenix**   | ⭐⭐⭐        | ⭐⭐⭐⭐   | ⭐⭐          | ⭐⭐⭐     | Self-hosted  |

**Recommendation:**

- **LangChain users:** LangSmith
- **Cost tracking:** Helicone
- **Advanced analytics:** Arize AI
- **Open-source:** Phoenix

---

## Deployment & Serving {#deployment-serving}

### Model Serving

**OpenAI API**

**Best for:**

- Prototyping (no infra needed)
- Standard use cases

**Pros:**

- Zero ops
- Reliable (99.9% uptime)

**Cons:**

- Vendor lock-in
- Privacy concerns

---

**vLLM**

**Best for:**

- Self-hosting open-source models (LLaMA, Mistral)

**Pros:**

- Fast (optimized inference)
- Supports many models

**Cons:**

- Requires GPU
- Ops overhead

**Example:**

```bash
# Serve LLaMA 3.1 70B
vllm serve meta-llama/Meta-Llama-3.1-70B \
  --tensor-parallel-size 4 \
  --dtype float16
```

---

**TGI (Text Generation Inference)**

**Best for:**

- Hugging Face models
- Production self-hosting

**Pros:**

- Production-ready (by Hugging Face)
- Good performance

**Cons:**

- GPU required
- Less flexible than vLLM

---

**Ollama**

**Best for:**

- Local development (run LLMs on laptop)

**Pros:**

- Easy to use (one command)
- Runs on CPU/GPU

**Cons:**

- Not for production (single-machine)

**Example:**

```bash
# Run LLaMA 3.1 8B locally
ollama run llama3.1
```

---

**Model Serving Comparison**

| Tool           | Ease of Use | Performance | Cost    | Best For       |
| -------------- | ----------- | ----------- | ------- | -------------- |
| **OpenAI API** | ⭐⭐⭐⭐⭐  | ⭐⭐⭐⭐⭐  | $$$     | Prototyping    |
| **vLLM**       | ⭐⭐⭐      | ⭐⭐⭐⭐⭐  | $ (GPU) | Production OSS |
| **TGI**        | ⭐⭐⭐⭐    | ⭐⭐⭐⭐    | $ (GPU) | Hugging Face   |
| **Ollama**     | ⭐⭐⭐⭐⭐  | ⭐⭐⭐      | Free    | Local dev      |

**Recommendation:**

- **Prototyping:** OpenAI API
- **Self-hosting:** vLLM (best performance)
- **Local dev:** Ollama

---

## Evaluation Tools {#evaluation-tools}

### Automated Evaluation

**RAGAS**

**Best for:** RAG evaluation

**Metrics:**

- **Faithfulness:** Is output grounded in context?
- **Answer relevance:** Does it answer the question?
- **Context precision:** Are retrieved docs relevant?

**Example:**

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy

result = evaluate(
    dataset=test_dataset,
    metrics=[faithfulness, answer_relevancy]
)
print(result)  # {"faithfulness": 0.92, "answer_relevancy": 0.87}
```

---

**LangChain Evaluators**

**Best for:** LangChain users

**Metrics:**

- Criteria-based (e.g., "Is output concise?")
- LLM-as-judge (GPT-4 rates outputs)

**Example:**

```python
from langchain.evaluation import load_evaluator

evaluator = load_evaluator("criteria", criteria="conciseness")
result = evaluator.evaluate_strings(
    prediction="Long response...",
    input="Short question"
)
print(result)  # {"score": 3/5}
```

---

**OpenAI Evals**

**Best for:** Systematic testing

**Features:**

- Eval templates
- LLM-as-judge
- Human eval interface

**Example:**

```python
from evals import run_eval

run_eval(
    eval_name="my_rag_eval",
    model="gpt-4o",
    dataset="test_questions.jsonl"
)
```

---

**Evaluation Tool Comparison**

| Tool                     | Ease of Use | Metrics      | Best For           |
| ------------------------ | ----------- | ------------ | ------------------ |
| **RAGAS**                | ⭐⭐⭐⭐    | RAG-specific | RAG systems        |
| **LangChain Evaluators** | ⭐⭐⭐⭐    | General      | LangChain users    |
| **OpenAI Evals**         | ⭐⭐⭐      | Flexible     | Systematic testing |

**Recommendation:**

- **RAG:** RAGAS (best metrics)
- **General:** LangChain Evaluators (easy to use)
- **Custom:** Build your own (most control)

---

## Development Tools {#development-tools}

### IDEs & Notebooks

**Jupyter Lab**

**Best for:** Experimentation, data analysis

**Pros:**

- Interactive
- Good for visualization

**Cons:**

- Not production code

---

**VS Code**

**Best for:** Production code

**Extensions:**

- **GitHub Copilot:** Code completion
- **Pylance:** Python type checking
- **Jupyter:** Run notebooks in VS Code

---

**Cursor**

**Best for:** AI-assisted coding

**Pros:**

- GPT-4 integrated (inline editing)
- Better than Copilot for complex tasks

**Cons:**

- Paid ($20/month)

---

### Prompt Playgrounds

**OpenAI Playground**

**Best for:** Testing OpenAI models

**Features:**

- Live testing
- Compare models
- Save prompts

---

**Anthropic Console**

**Best for:** Testing Claude

**Features:**

- Prompt caching
- Compare prompts
- Evaluate outputs

---

**PromptLayer**

**Best for:** Prompt versioning

**Features:**

- Track prompt changes
- A/B test prompts
- Analytics

---

## Tool Selection Guide {#tool-selection}

### Decision Framework

**1. Model Provider**

```
If (privacy is critical):
    Use self-hosted LLaMA 3.1
Else if (best quality):
    Use GPT-4o or Claude 3.5 Sonnet
Else if (budget is tight):
    Use GPT-4o-mini or Gemini Flash
Else if (huge context needed):
    Use Gemini 1.5 Pro (2M tokens)
```

---

**2. Embedding Model**

```
If (budget < $100/month):
    Use text-embedding-3-small
Else if (quality is critical):
    Use text-embedding-3-large or Voyage AI
Else if (multilingual):
    Use Cohere embed-multilingual-v3
Else if (self-hosted):
    Use all-mpnet-base-v2
```

---

**3. Vector Database**

```
If (vectors < 100K):
    Use pgvector (simplest)
Else if (ease of use is priority):
    Use Pinecone (managed)
Else if (best performance):
    Use Qdrant (self-hosted)
Else if (existing Elasticsearch):
    Use Elasticsearch vector search
```

---

**4. Framework**

```
If (just prototyping):
    Use LangChain (fast to build)
Else if (production RAG):
    Use LlamaIndex (best retrieval)
Else if (multi-agent system):
    Use LangGraph (state management)
Else if (simple use case):
    Use OpenAI/Anthropic SDK directly (no framework)
```

---

**5. Observability**

```
If (using LangChain):
    Use LangSmith (native integration)
Else if (cost tracking is priority):
    Use Helicone (proxy-based)
Else if (advanced analytics):
    Use Arize AI (drift detection)
Else if (budget = $0):
    Use Phoenix (open-source)
```

---

## Cost Comparison {#cost-comparison}

### RAG System Cost Breakdown

**Scenario:** 10K documents, 1000 queries/day

**Components:**

| Component                   | Tool                   | Monthly Cost   |
| --------------------------- | ---------------------- | -------------- |
| **Embedding** (one-time)    | text-embedding-3-small | $0.60          |
| **Vector DB**               | Pinecone Serverless    | $7.50          |
| **Retrieval**               | Top 5 chunks           | Included       |
| **LLM Generation**          | GPT-4o-mini            | $18            |
| **Re-ranking** (optional)   | Cohere Rerank          | $60            |
| **Observability**           | Helicone               | $20            |
| **Total (without re-rank)** |                        | **$46/month**  |
| **Total (with re-rank)**    |                        | **$106/month** |

**Per-query cost:** $0.0015 (without re-rank), $0.0035 (with re-rank)

---

### Optimization Strategies

**1. Caching (40% hit rate):**

- Saves 40% of LLM costs → $46 → $28/month

**2. Use gpt-4o-mini for simple queries:**

- 70% simple (mini), 30% complex (GPT-4o)
- Saves 50% of LLM costs → $18 → $9/month

**3. Self-host embeddings:**

- Use all-mpnet-base-v2 → Save $0.60 (one-time)

**4. Use pgvector (if <100K vectors):**

- No vector DB cost → Save $7.50/month

**Optimized total:** $9 (LLM) + $0 (vector DB) + $0 (observability) = **$9/month** ✅

---

### Enterprise Cost Example

**Scenario:** 10K employees, 100M queries/month

**Without optimization:**

- LLM: 100M × $0.005 (GPT-4o) = $500K/month ❌

**With optimization:**

- 70% gpt-4o-mini: 70M × $0.0003 = $21K
- 30% GPT-4o: 30M × $0.005 = $150K
- Caching (40% hit): $171K × 0.6 = $103K ✅

**Savings:** $500K → $103K (79% reduction)

---

## Tool Ecosystem Map

```
┌─────────────────────────────────────────────────────────┐
│                      USER INTERFACE                      │
│        (Web App, Mobile App, Chat Widget)               │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   ORCHESTRATION LAYER                    │
│    LangChain / LlamaIndex / LangGraph / Custom          │
└─────────────────────────────────────────────────────────┘
                            ↓
┌──────────────┬──────────────┬──────────────┬────────────┐
│  LLM Models  │  Embeddings  │  Vector DB   │   Tools    │
├──────────────┼──────────────┼──────────────┼────────────┤
│ GPT-4o       │ OpenAI       │ Pinecone     │ Web Search │
│ Claude 3.5   │ Cohere       │ Weaviate     │ Calculator │
│ Gemini       │ Voyage AI    │ Qdrant       │ Database   │
│ LLaMA 3.1    │ Local (SBERT)│ pgvector     │ APIs       │
└──────────────┴──────────────┴──────────────┴────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                 OBSERVABILITY & MONITORING               │
│    LangSmith / Helicone / Arize AI / Custom             │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   DEPLOYMENT & INFRA                     │
│      Docker / Kubernetes / Cloud (AWS/GCP/Azure)        │
└─────────────────────────────────────────────────────────┘
```

---

## Recommended Stacks

### Startup Stack (Budget < $500/month)

**Goal:** Build fast, spend little

- **Model:** GPT-4o-mini (gpt-4o for complex)
- **Embeddings:** text-embedding-3-small
- **Vector DB:** Pinecone Serverless or pgvector
- **Framework:** LangChain (fast prototyping)
- **Observability:** Helicone Free
- **Deployment:** Vercel / Railway

**Total cost:** ~$100/month

---

### Enterprise Stack (Production)

**Goal:** Reliability, scalability, compliance

- **Model:** GPT-4o + Claude 3.5 (fallback)
- **Embeddings:** text-embedding-3-large
- **Vector DB:** Qdrant (self-hosted) or Pinecone
- **Framework:** LlamaIndex (production RAG)
- **Observability:** Arize AI or LangSmith Pro
- **Deployment:** Kubernetes (AWS EKS)

**Total cost:** ~$5K-50K/month (depends on scale)

---

### Privacy-First Stack (Self-Hosted)

**Goal:** No data leaves your servers

- **Model:** LLaMA 3.1 70B (vLLM)
- **Embeddings:** all-mpnet-base-v2 (local)
- **Vector DB:** Qdrant (self-hosted)
- **Framework:** Custom (no external APIs)
- **Observability:** Phoenix (self-hosted)
- **Deployment:** On-premises Kubernetes

**Total cost:** GPU costs only (~$2K/month for 4× A100)

---

## Resources

**Official Docs:**

- [OpenAI Platform](https://platform.openai.com/docs)
- [Anthropic Claude](https://docs.anthropic.com)
- [LangChain Docs](https://docs.langchain.com)
- [LlamaIndex Docs](https://docs.llamaindex.ai)

**Community:**

- r/LocalLLaMA (open-source models)
- LangChain Discord
- Hugging Face Discord

**Benchmarks:**

- [LMSys Chatbot Arena](https://chat.lmsys.org) (model rankings)
- [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) (embeddings)

---

## Final Tips

**1. Start simple:** Don't use 5 tools when 2 will work.

**2. Evaluate before committing:** Test 2-3 options, measure results.

**3. Tools change fast:** Check for new releases quarterly.

**4. Framework-agnostic:** Don't lock into LangChain/LlamaIndex if not needed.

**5. Open-source first:** Try OSS before paying for commercial.

**6. Monitor costs:** Tools can get expensive at scale. Track spend weekly.

**7. Community matters:** Choose tools with active communities (easier debugging).

---

**Next:** [Part 15: Career Roadmap →](15_Career_Roadmap.md)

**Previous:** [← Part 13: Interview Preparation](13_Interview_Preparation.md)
