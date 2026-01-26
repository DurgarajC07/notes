# Part 13: Interview Preparation

## Table of Contents

1. [Beginner-Level Questions](#beginner-level)
2. [Mid-Level Questions](#mid-level)
3. [Senior/Staff System Design](#senior-staff)
4. [Behavioral Questions](#behavioral)
5. [How to Explain Projects](#explain-projects)
6. [Common Mistakes to Avoid](#common-mistakes)

---

## Beginner-Level Questions {#beginner-level}

### Conceptual Understanding

**Q1: What is an embedding? How is it different from one-hot encoding?**

**Answer:**

An embedding is a dense vector representation of data (text, images, etc.) in continuous space.

**Example:**
```
Word: "cat"
One-hot: [0, 0, 1, 0, 0, ...] (sparse, 50K dimensions)
Embedding: [0.24, -0.51, 0.83, ...] (dense, 768 dimensions)
```

**Key differences:**

- One-hot: Sparse (mostly zeros), high-dimensional, no semantic meaning
- Embedding: Dense (all values meaningful), lower-dimensional, captures semantics

**Real-world use:**
- Similar words have similar embeddings: `embed("cat")` ≈ `embed("kitten")`
- Used in: RAG (document search), recommendation systems, semantic search

---

**Q2: Explain transformer architecture at a high level.**

**Answer:**

Transformers process sequences (text, images) using self-attention to understand relationships between elements.

**Core components:**

1. **Input:** Tokens → Embeddings (token + positional)
2. **Self-Attention:** Each token attends to all other tokens (learns context)
3. **Feed-Forward Network:** Process each token independently
4. **Layer Norm + Residual:** Stabilize training
5. **Output:** Contextualized embeddings

**Example (text):**
```
Input: "The cat sat on the mat"
Token "cat" attends to: "The" (less), "sat" (more), "mat" (medium)
Output: Context-aware embedding for "cat"
```

**Why it won:**
- Parallel processing (vs sequential RNNs)
- Long-range dependencies (attention across entire sequence)
- Scalable (GPT-4, Claude use transformers)

---

**Q3: What is prompt engineering? Give examples of good vs bad prompts.**

**Answer:**

Prompt engineering is designing inputs to LLMs to get desired outputs.

**Bad prompt:**
```
"Tell me about dogs."
```
- Vague, no context, unpredictable output

**Good prompt:**
```
You are a veterinary expert. Explain to a 10-year-old why dogs need vaccines.
Use simple language and include 2-3 examples.
```
- Clear role, specific task, constraints

**Techniques:**
- Few-shot: Provide examples
- Chain-of-thought: "Let's think step by step"
- Delimiters: Use XML tags `<context>...</context>`

**Interview tip:** Show you understand prompting is engineering work, not magic.

---

**Q4: What's the difference between GPT and BERT?**

**Answer:**

| Aspect | GPT | BERT |
|--------|-----|------|
| **Type** | Decoder-only (autoregressive) | Encoder-only |
| **Training** | Next-token prediction | Masked language modeling |
| **Use case** | Text generation | Text understanding |
| **Direction** | Left-to-right | Bidirectional |
| **Example** | ChatGPT, code completion | Search, classification |

**GPT:** "Given 'The cat sat on the', predict 'mat'"

**BERT:** "Given 'The [MASK] sat on the mat', predict 'cat'"

**Real-world:**
- GPT: Chatbots, code generation, creative writing
- BERT: Google Search, sentiment analysis, Q&A

---

**Q5: How does RAG work?**

**Answer:**

RAG = Retrieval-Augmented Generation. Combines retrieval (search) with generation (LLM).

**Pipeline:**

1. **User query:** "What's our refund policy?"
2. **Embed query:** Convert to vector
3. **Search:** Find similar docs in vector DB
4. **Retrieve:** Get top K relevant chunks
5. **Augment:** Add retrieved docs to LLM context
6. **Generate:** LLM answers based on retrieved info

**Why RAG?**
- Solves knowledge cutoff (LLM knows nothing after training)
- Reduces hallucinations (grounds LLM in facts)
- Domain-specific (add company docs without fine-tuning)

**Example:**
```
Without RAG: "I don't know your refund policy (trained in 2023)."
With RAG: "Our refund policy allows 30-day returns (retrieved from docs)."
```

---

## Mid-Level Questions {#mid-level}

### System Design

**Q1: Design a RAG system for a company's internal documentation (10K documents).**

**Answer:**

**Requirements clarification:**
- Document types? (PDFs, Confluence, code)
- Users? (1000 employees)
- Latency target? (<2s)
- Budget? (<$0.01 per query)

**Architecture:**

```
User Query
    ↓
Embedding (text-embedding-3-small)
    ↓
Vector Search (Pinecone: 10K docs × 512 dims)
    ↓
Hybrid Search (BM25 + Vector → RRF)
    ↓
Re-rank top 20 (Cohere Rerank)
    ↓
LLM Generation (GPT-4o)
    ↓
Response + Citations
```

**Components:**

1. **Ingestion:**
   - Parse docs (Unstructured.io)
   - Chunk (1000 chars, 200 overlap)
   - Embed (text-embedding-3-small: $0.00001/1K tokens)
   - Store in Pinecone

2. **Retrieval:**
   - Hybrid search (semantic + keyword)
   - Re-rank top 20 (Cohere: $0.002 per query)

3. **Generation:**
   - GPT-4o with retrieved context
   - Citations (return source docs)

4. **Caching:**
   - Redis for frequent queries (50% hit rate → 50% cost savings)

**Cost:**
- Embedding: $0.00001
- Search: $0.00005 (Pinecone)
- Re-rank: $0.002
- LLM: $0.005
- **Total: $0.0071 per query** ✅

**Evaluation:**
- Precision@5 (are top 5 results relevant?)
- Human eval (sample 100 queries)
- User feedback (thumbs up/down)

---

**Q2: How would you reduce hallucinations in an LLM?**

**Answer:**

Hallucinations = LLM generates false information.

**Strategies:**

1. **RAG (Ground in facts)**
   - Retrieve source documents
   - LLM cites sources
   - Verify claims against retrieved docs

2. **Prompt engineering**
   - "Only use information from provided context."
   - "If you don't know, say 'I don't know.'"
   - Use XML tags to separate context from instructions

3. **Structured output**
   - Force JSON schema (can't hallucinate keys)
   - Validate outputs programmatically

4. **Fact-checking layer**
   - After generation, check claims against knowledge base
   - Flag low-confidence statements

5. **Model selection**
   - Use models trained for truthfulness (Claude, GPT-4)
   - Avoid overly creative models for factual tasks

6. **Temperature = 0**
   - Deterministic outputs (less creativity, more accuracy)

7. **Evaluation**
   - RAGAS faithfulness metric
   - Human eval (% of hallucinated claims)

**Example:**
```
Without mitigation:
Q: "What's the capital of Atlantis?"
A: "The capital of Atlantis is Poseidonis." (hallucination)

With mitigation:
A: "I don't have information about Atlantis as it's a fictional place."
```

---

**Q3: Explain different chunking strategies and trade-offs.**

**Answer:**

Chunking = splitting documents for embedding/retrieval.

| Strategy | Description | Pros | Cons | Use Case |
|----------|-------------|------|------|----------|
| **Fixed-size** | 500 chars per chunk | Simple, consistent | Breaks mid-sentence | Generic docs |
| **Sentence-based** | Split on `.` `!` `?` | Semantic units | Variable size | Clean text |
| **Paragraph** | Split on `\n\n` | Natural boundaries | Too large (>1000 chars) | Articles, blogs |
| **Recursive** | Try `\n\n` → `\n` → `. ` | Hierarchical | Complex | Mixed formats |
| **Semantic** | Split when meaning changes (LLM-based) | Highest quality | Expensive | High-value docs |
| **Overlapping** | 1000 chars, 200 overlap | Context continuity | Redundancy | Technical docs |

**Trade-offs:**

- **Small chunks (200 chars):** Precise, but miss context
- **Large chunks (2000 chars):** More context, but noise in search
- **Optimal:** 500-1000 chars with 10-20% overlap

**Real-world:**
- Code: Function-level chunking
- Legal: Clause-level
- FAQs: Q&A pairs

**Interview tip:** There's no one-size-fits-all. Explain trade-offs based on document type.

---

**Q4: When would you fine-tune vs use RAG?**

**Answer:**

| Scenario | RAG | Fine-tuning | Why |
|----------|-----|-------------|-----|
| **New knowledge** (docs updated weekly) | ✅ Yes | ❌ No | RAG: Easy updates<br>Fine-tuning: Expensive retraining |
| **Domain language** (medical jargon) | ❌ No | ✅ Yes | Fine-tuning learns terminology |
| **Factual Q&A** (support chatbot) | ✅ Yes | ❌ No | RAG retrieves facts |
| **Style/tone** (formal business writing) | ❌ No | ✅ Yes | Fine-tuning learns style |
| **Low latency** (<100ms) | ❌ No | ✅ Yes | Fine-tuned models are faster (no retrieval) |
| **Budget** (<$1000) | ✅ Yes | ❌ No | RAG is cheaper |

**Combined approach:**
- Fine-tune base model for domain language
- Use RAG for knowledge retrieval

**Example:**
- Medical chatbot: Fine-tune on medical text + RAG for latest research papers

**Interview tip:** Don't say "always use RAG" or "always fine-tune". Explain trade-offs.

---

**Q5: How do you evaluate an LLM's outputs?**

**Answer:**

**1. Automated metrics (offline):**

- **Perplexity:** Lower = better (measures model confidence)
- **BLEU/ROUGE:** Compare to reference (not ideal for LLMs)
- **Exact match:** For structured outputs (SQL, JSON)
- **RAGAS:** Faithfulness, relevance, context precision

**2. LLM-as-a-judge:**

- Use GPT-4 to rate outputs (1-5 scale)
- Pairwise comparison (which output is better?)
- Rubric-based (evaluate on criteria: clarity, accuracy, helpfulness)

**3. Human evaluation:**

- Annotators rate sample outputs (100-500 examples)
- Metrics: Accuracy, helpfulness, harmfulness
- Cost: $0.10-1 per eval (expensive but gold standard)

**4. Business metrics (online):**

- User satisfaction (CSAT, thumbs up/down)
- Task completion rate (did user find answer?)
- Engagement (time on page, follow-up questions)

**5. A/B testing:**

- Model A vs Model B (50/50 traffic split)
- Measure: Conversion, retention, revenue

**Example pipeline:**
1. Offline: RAGAS score > 0.8 before deployment
2. Shadow deploy: Compare new model to production (1 week)
3. A/B test: 10% traffic to new model
4. Full rollout: If metrics improve

**Interview tip:** Emphasize you can't rely on one metric. Use a mix.

---

## Senior/Staff System Design {#senior-staff}

### Large-Scale Architecture

**Q1: Design an AI customer support system handling 10M users.**

**Expected discussion:**

**1. Requirements clarification:**
- Concurrent users? (10K concurrent)
- Languages? (English only or multilingual?)
- Latency? (<2s)
- Cost budget? ($10K/month)
- SLA? (99.9% uptime)

**2. High-level architecture:**

```
Users (10M) → Load Balancer → API Gateway → Microservices

Microservices:
├── Intent Classifier (gpt-4o-mini)
├── Retrieval Service (RAG)
├── Agent Orchestrator (LangGraph)
├── Tool Services (Ticketing, Orders, Payments)
└── LLM Generation (GPT-4o)

Supporting:
├── Cache Layer (Redis Cluster)
├── Vector DB (Pinecone Serverless)
├── SQL DB (RDS Multi-AZ)
└── Monitoring (Datadog, LangSmith)
```

**3. Key decisions:**

**Scalability:**
- Horizontal scaling (Kubernetes HPA: 10-100 replicas)
- Async processing (Celery for non-realtime)
- Database: Read replicas for heavy queries

**Cost optimization:**
- Tiered models: gpt-4o-mini for 70% of queries
- Caching: 40% hit rate (semantic cache)
- Batch API calls where possible

**Latency:**
- Edge caching (CloudFront for static content)
- Streaming responses (first token in 300ms)
- Parallel tool calls

**Reliability:**
- Fallback models (if GPT-4o down, use Claude)
- Circuit breakers (prevent cascading failures)
- Graceful degradation (rule-based responses if all LLMs down)

**4. Data flow:**

```
User: "I want to return my order"
    ↓
Intent: "order_return" (confidence: 0.95)
    ↓
Retrieve: Get order details (API call)
    ↓
Agent: Check return policy (RAG)
    ↓
Decision: "Eligible for return"
    ↓
Action: Create return label + ticket
    ↓
Response: "Here's your return label. Refund in 3-5 days."
```

**5. Monitoring:**

- **Input:** Query volume, intent distribution
- **Process:** LLM latency, tool call success rate, cache hit rate
- **Output:** User satisfaction, resolution rate, escalation rate
- **Business:** Cost per conversation, revenue impact

**6. Security:**

- Authentication: OAuth2 + JWT
- PII handling: Mask in logs, encrypt in DB
- Rate limiting: 10 requests/min per user
- Content filtering: Toxicity check on user inputs

**7. Scaling estimate:**

- 10M users × 10 queries/month = 100M queries/month
- Peak: 10K concurrent → 30K requests/min
- API servers: 50 replicas (600 req/min each)
- Cost: 100M × $0.005 = $500K/month (LLM)
  - Optimization: Use mini for simple queries → $150K/month ✅

**Interview tip:** Walk through this systematically. Don't jump to implementation details.

---

**Q2: Build a code review assistant that scales across 1000 repositories.**

**Expected discussion:**

**1. Architecture:**

```
GitHub Webhook
    ↓
Queue (SQS: 1000 PRs/day)
    ↓
Worker Pool (50 workers)
    ↓
Code Analyzer
    ├── Bug Detection (GPT-4o)
    ├── Style Check (gpt-4o-mini)
    ├── Security Scan (GPT-4o + Semgrep)
    └── Performance Analysis (GPT-4o)
    ↓
Aggregator (GPT-4o)
    ↓
Post to GitHub
```

**2. Challenges:**

**Large PRs (500+ lines):**
- Problem: Doesn't fit in context
- Solution: Chunk by file, summarize large files

**Rate limits:**
- GitHub API: 5K requests/hour
- OpenAI API: 10K requests/min (tier 5)
- Solution: Use multiple API keys, implement backoff

**Cost:**
- 1000 PRs/day × $0.037 = $37/day = $1,110/month
- Budget: $5K/month ✅

**3. Quality:**

- False positives: 25% → Train on past PRs (manual labels)
- Critical bugs: Auto-block PR merge
- Low-confidence flags: Mark as "suggestion"

**4. Optimizations:**

- Cache file-level analysis (don't re-analyze unchanged files)
- Skip generated code (package-lock.json, migrations)
- Parallel analysis (all checks run simultaneously)

**Interview tip:** Emphasize you've thought about edge cases (large PRs, API limits).

---

**Q3: Design a multi-agent system for travel booking.**

**Expected discussion:**

**1. Agents:**

```
User: "Plan a 5-day trip to Paris for 2 people, budget $3000"
    ↓
Coordinator Agent (o1-preview for planning)
    ├── Research Agent (web search: flights, hotels)
    ├── Budget Agent (track spending)
    ├── Booking Agent (API calls: Booking.com, Expedia)
    └── Itinerary Agent (generate day-by-day plan)
    ↓
Aggregator (combine all results)
    ↓
User: "Here's your trip plan (flights, hotel, itinerary)"
```

**2. Inter-agent communication:**

- Message passing (JSON events)
- Shared memory (Redis for current state)
- Checkpointing (save progress, resume if failure)

**3. Workflow (LangGraph):**

```python
from langgraph.graph import StateGraph

graph = StateGraph(TravelState)

graph.add_node("research", research_agent)
graph.add_node("budget", budget_agent)
graph.add_node("booking", booking_agent)

graph.add_edge("research", "budget")
graph.add_edge("budget", "booking")

workflow = graph.compile()
```

**4. Failure handling:**

- If flight API fails → Try alternative API
- If over budget → Suggest cheaper options
- If no availability → Suggest nearby dates

**5. Evaluation:**

- Task completion: 85% (15% require human intervention)
- User satisfaction: 4.2/5
- Time saved: 2 hours → 5 minutes

**Interview tip:** Explain how agents coordinate, not just what each does.

---

**Q4: Architect a GenAI platform for an enterprise (multi-tenancy, security, cost).**

**Expected discussion:**

**1. Requirements:**

- 10K employees across 20 departments
- Each department has separate data (HR, Finance, Engineering)
- Compliance: GDPR, SOC 2
- Budget: $50K/month

**2. Architecture:**

```
User → Auth (SSO) → Tenant Router
    ↓
Department-specific namespace:
    ├── Vector DB (per-tenant index)
    ├── LLM Gateway (shared, with tenant ID)
    └── Data Store (isolated per tenant)

Shared services:
    ├── Monitoring (Datadog)
    ├── Prompt Registry (versioned prompts)
    └── Cost Tracking (per tenant)
```

**3. Multi-tenancy:**

- **Data isolation:** Each department has separate Pinecone namespace
- **Permissions:** RBAC (role-based access control)
- **Cost attribution:** Track usage per department

**4. Security:**

- **Authentication:** SSO (Okta, Azure AD)
- **Authorization:** Check user permissions before RAG retrieval
- **Data residency:** EU users → EU region
- **Audit logs:** All queries logged (who, what, when)

**5. Cost controls:**

- **Quotas:** 1000 queries/user/month
- **Model routing:** Simple queries → gpt-4o-mini
- **Caching:** 50% hit rate (save $25K/month)
- **Alerts:** Notify if department exceeds budget

**6. Compliance:**

- **GDPR:** Right to deletion (delete user data)
- **Data retention:** Logs kept for 90 days
- **Encryption:** At-rest (AES-256), in-transit (TLS)

**7. Monitoring:**

- **Per-tenant metrics:** Usage, cost, latency, errors
- **Anomaly detection:** Spike in usage (potential abuse)
- **Dashboards:** Executives see cost breakdown

**Interview tip:** This is a broad question. Focus on one area (security OR cost OR architecture), then expand if asked.

---

**Q5: How would you build a ChatGPT competitor?**

**Expected discussion:**

**This is a very open-ended question. Break it down:**

**1. Clarify scope:**
- "Are we building from scratch (training LLM) or using existing models (OpenAI API)?"
- "Target users? (Consumers, enterprises, developers)"
- "Differentiation? (Privacy, cost, domain-specific)"

**Assume:** Build for enterprises using existing LLMs.

**2. Core features:**

- Multi-turn conversations
- Tool use (web search, calculator, code execution)
- File uploads (PDFs, images)
- Conversation history
- API for developers

**3. Architecture:**

```
User → Web/Mobile App → API Gateway
    ↓
Conversation Manager
    ├── LLM Router (GPT-4o, Claude, Gemini)
    ├── Tool Executor (web search, code execution)
    ├── Memory Manager (conversation history)
    └── Safety Layer (content moderation)
    ↓
Storage:
    ├── Conversations (PostgreSQL)
    ├── File uploads (S3)
    └── User data (encrypted)
```

**4. Differentiators:**

**Privacy:**
- Self-hosted option (for sensitive data)
- Zero data retention mode

**Cost:**
- Model routing (use cheaper models for simple queries)
- Offer credits instead of subscriptions

**Customization:**
- Custom instructions per workspace
- Domain-specific models (legal, medical)

**5. Scaling:**

- 1M users, 100M conversations/month
- API servers: 100 replicas (auto-scaling)
- Database: Sharded by user ID
- Cost: 100M × $0.01 = $1M/month (LLM costs)

**6. Business model:**

- Freemium (10 queries/day free)
- Pro ($20/month, unlimited)
- Enterprise ($500/month/team, custom models)

**Interview tip:** This question tests strategic thinking, not just coding. Focus on product decisions.

---

## Behavioral Questions {#behavioral}

### STAR Framework

**Situation → Task → Action → Result**

---

**Q1: Describe a time you debugged a hallucination issue.**

**Answer (STAR):**

**Situation:**
"At my previous company, our RAG chatbot was hallucinating product features that didn't exist. Users complained it gave incorrect information."

**Task:**
"I was tasked with reducing hallucinations to <5% (measured by human eval)."

**Action:**
1. "I analyzed 100 hallucinated responses. Found 3 patterns:
   - 40% from poor retrieval (wrong docs retrieved)
   - 35% from LLM creativity (ignoring context)
   - 25% from ambiguous user queries"

2. "I implemented fixes:
   - Improved chunking (500 chars → 1000 chars for better context)
   - Added re-ranking (Cohere rerank on top 20 results)
   - Prompt engineering (strict instruction: 'Only use provided context')
   - Lowered temperature (0.7 → 0.1)"

3. "Deployed to 10% of users (A/B test) for 1 week."

**Result:**
"Hallucination rate dropped from 12% → 4%. User satisfaction increased from 3.8/5 → 4.3/5. Rolled out to 100% of users."

---

**Q2: How do you stay updated with AI research?**

**Answer:**

"I use a layered approach:

**1. Curated sources (daily, 15 min):**
- Twitter/X: Follow Andrej Karpathy, Andrew Ng, AI researchers
- Newsletters: The Batch, TLDR AI, Last Week in AI

**2. Deep dives (weekly, 2 hours):**
- Read 1-2 papers (focus on abstracts + conclusions)
- Watch conference talks (NeurIPS, ICLR on YouTube)

**3. Hands-on (monthly):**
- Build projects with new models (e.g., tried GPT-4V when it launched)
- Experiment with new frameworks (LangGraph, DSPy)

**4. Community (ongoing):**
- r/LocalLLaMA, Hugging Face Discord
- Contribute to open-source (submitted PR to LangChain)

**Recent example:**
'When o1 launched, I read the blog, tried it for reasoning tasks, shared findings with my team. We decided NOT to use it yet (too expensive for our use case).'

**Interview tip:** Show you balance breadth (staying aware) and depth (experimenting).

---

**Q3: Tell me about a GenAI project that failed and what you learned.**

**Answer (STAR):**

**Situation:**
"I built a code generation tool to automate React component creation. Goal: 50% time savings for developers."

**Task:**
"Generate production-ready React components from natural language descriptions."

**Action:**
"I used GPT-4 with few-shot prompting. Deployed to 10 developers (internal beta)."

**Result (Failed):**
"Adoption was only 20%. Developers said:
- Generated code was inconsistent (different styles)
- Missing edge cases (error handling, accessibility)
- Faster to write from scratch than debug AI code"

**What I learned:**

1. **User research first:** I assumed developers wanted code generation. They actually wanted code suggestions (autocomplete, not full generation).

2. **Start smaller:** Full component generation was too ambitious. Should have started with helper functions.

3. **Iterate with users:** I built for 2 months alone. Should have shown mockups to users in week 1.

4. **Evaluation matters:** I measured "time to generate," not "time to production-ready code" (which included debugging).

**What I did next:**
"Pivoted to a simpler tool: Generate unit tests (not components). This had 70% adoption because it solved a real pain point."

**Interview tip:** Show self-awareness and learning, not defensiveness.

---

**Q4: How do you balance speed vs quality when shipping AI features?**

**Answer:**

"I use a phased approach:

**Phase 1: MVP (1 week)**
- Goal: Prove concept works
- Quality bar: 70% accuracy (good enough for internal demo)
- Example: RAG chatbot with basic chunking, no re-ranking

**Phase 2: Internal beta (2 weeks)**
- Goal: Get real user feedback
- Quality bar: 80% accuracy (usable but rough edges)
- Deploy to 10-20 internal users

**Phase 3: Production (4 weeks)**
- Goal: High quality
- Quality bar: 90%+ accuracy
- Add re-ranking, evaluation pipeline, monitoring

**Key principles:**

1. **Define 'good enough':** For exploratory features, 70% is fine. For user-facing, 90%+ required.

2. **Fail fast:** If MVP doesn't work, kill project (don't polish a bad idea).

3. **Incremental quality:** Start simple (GPT-3.5), upgrade later (GPT-4) if needed.

4. **Automate regression tests:** Catch quality regressions early.

**Example:**
'We shipped a summarization feature in 3 days with GPT-3.5 (80% quality). Users loved it. Later upgraded to GPT-4 (95% quality) when we had budget.'

**Interview tip:** Show you're pragmatic, not a perfectionist.

---

**Q5: Explain a complex AI concept to a non-technical stakeholder.**

**Answer:**

"I explain RAG like this:

**Analogy:**
'Imagine you're a librarian. Someone asks you a question about ancient Rome.

Without RAG (just LLM):
- You answer from memory (might be wrong or outdated).

With RAG:
- You search the library (retrieval),
- Find relevant books,
- Read key passages,
- Answer based on those books (generation).

RAG = Search + Answer, not just Answer.'

**Why it matters (business value):**
'RAG lets our chatbot answer questions about our products using the latest documentation. Without RAG, the chatbot is stuck in 2023 knowledge.'

**Trade-offs (honest):**
'RAG is slower (2s vs 1s) but much more accurate (90% vs 60%). For customer support, accuracy is more important than speed.'

**Follow-up (budget):**
'RAG costs $0.01 per query (search + LLM). We handle 10K queries/month = $100/month. This is much cheaper than hiring more support staff ($5K/month per person).'

**Interview tip:** Use analogies, focus on business impact, be honest about trade-offs."

---

## How to Explain Projects {#explain-projects}

### STAR Framework for Projects

**S (Situation):** Context, problem, why it mattered

**T (Task):** Your role, what you were responsible for

**A (Action):** What you built, technical decisions, challenges

**R (Result):** Impact (metrics), what you learned

---

### Example: RAG System

**Bad explanation:**
"I built a RAG system using LangChain and Pinecone. It has embeddings and retrieval."

**Good explanation (STAR):**

**Situation:**
"Our company had 10K internal docs (Confluence, PDFs, Slack). Employees spent 30 min/day searching for information manually."

**Task:**
"I was tasked with building an internal knowledge base chatbot. Requirements: <2s response time, 90%+ accuracy, <$0.01 per query."

**Action:**
"I designed a RAG system:
- **Ingestion:** Parsed 10K docs (Unstructured.io), chunked into 1000-char pieces (50K chunks), embedded with text-embedding-3-small.
- **Retrieval:** Hybrid search (BM25 + vector search), re-ranked top 20 with Cohere.
- **Generation:** GPT-4o with retrieved context, citations.
- **Challenges:**
  - Chunking strategy: Fixed-size broke tables → Switched to semantic chunking (10% better retrieval).
  - Cost: Initial design was $0.02/query → Optimized to $0.007 (used gpt-4o-mini for simple queries, cached frequent searches).
  - Accuracy: Baseline 70% → Added re-ranking → 88% → Few-shot prompting → 92%."

**Result:**
"Deployed to 1000 employees. Metrics:
- Usage: 500 queries/day
- User satisfaction: 4.3/5
- Time saved: 20 min/employee/day → 12K hours/year saved
- Cost: $105/month (vs $50K/year for manual search)
- Accuracy: 92% (human eval on 100 queries)

**What I learned:**
- Chunking is critical (spent 50% of time on this).
- Evaluation is hard (built custom eval set of 200 Q&A pairs).
- Users want citations (added source links, increased trust)."

---

### Key Principles

**1. Lead with business impact:**
- "Saved 12K hours/year"