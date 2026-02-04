# Part 6: Retrieval Augmented Generation (RAG)

> The most common architecture in production GenAI systems

---

## What is RAG?

**Definition:** Technique to enhance LLM responses with external knowledge by retrieving relevant documents before generation.

**Simple formula:**

```
RAG = Retrieval (find relevant docs) + Generation (LLM uses docs to answer)
```

---

## Why RAG Exists

### Problems RAG Solves

**1. Knowledge cutoff:**

- LLMs trained on data up to a certain date
- GPT-4: Training data cutoff September 2021
- RAG: Add real-time information

**2. Hallucinations:**

- LLMs sometimes make up facts
- RAG: Ground responses in actual documents

**3. Domain-specific knowledge:**

- LLMs lack your company's internal data
- RAG: Add proprietary documents, databases, wikis

**4. Source attribution:**

- Hard to verify LLM claims
- RAG: Cite specific documents

**5. Cost and control:**

- Fine-tuning is expensive and slow
- RAG: Update knowledge by updating documents (no retraining)

---

### RAG vs Fine-Tuning vs Prompting

| Approach        | Knowledge Update     | Cost | Use Case                 |
| --------------- | -------------------- | ---- | ------------------------ |
| **Prompting**   | Paste docs in prompt | Free | Small docs (<10 pages)   |
| **RAG**         | Update vector DB     | Low  | Large knowledge bases    |
| **Fine-tuning** | Retrain model        | High | Teach new behavior/style |

**Best practice:** Try prompting → RAG → Fine-tuning (in that order).

---

## RAG Architecture (End-to-End)

### High-Level Pipeline

```
Indexing (Offline):
Documents → Chunking → Embedding → Vector DB

Retrieval (Online):
User Query → Embedding → Similarity Search → Top-K Docs

Generation:
Query + Retrieved Docs → LLM → Answer
```

---

### Detailed Flow

```
┌─────────────────┐
│  Raw Documents  │ (PDF, text, HTML, code, etc.)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Load & Parse  │ (Extract text, handle formats)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Chunking     │ (Split into smaller pieces)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Embedding     │ (text → vectors)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Vector DB     │ (Store embeddings + metadata)
└────────┬────────┘
         │
         ▼
    [INDEXING COMPLETE]

┌─────────────────┐
│   User Query    │ "What's our refund policy?"
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Query Embedding │ (query → vector)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Similarity      │ (cosine similarity)
│ Search          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Top-K Chunks   │ (5-10 most relevant)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Re-ranking     │ (optional: improve relevance)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   LLM Prompt    │ (query + context chunks)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Generation    │ (LLM produces answer)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Answer      │ + citations
└─────────────────┘
```

---

## Embeddings in Practice

### What are Embeddings?

**Definition:** Dense vector representations of text that capture semantic meaning.

**Example:**

```python
text = "The cat sat on the mat"
embedding = [0.23, -0.15, 0.87, ..., 0.45]  # 768-1536 dimensions
```

**Key property:** Similar texts → similar embeddings (measured by cosine similarity).

---

### Embedding Models

#### OpenAI Embeddings

**text-embedding-3-small:**

- 1536 dimensions
- Fast, cost-effective
- Good for most use cases

**text-embedding-3-large:**

- 3072 dimensions
- Higher quality
- 2x cost

**text-embedding-ada-002 (legacy):**

- 1536 dimensions
- Being replaced by v3

**Usage:**

```python
from openai import OpenAI
client = OpenAI()

response = client.embeddings.create(
    model="text-embedding-3-small",
    input="Your text here"
)

embedding = response.data[0].embedding  # List of 1536 floats
```

**Cost (Jan 2025):**

- Small: $0.02 / 1M tokens
- Large: $0.13 / 1M tokens

---

#### Cohere Embeddings

**embed-english-v3.0:**

- Multilingual support
- Good retrieval performance

**embed-multilingual-v3.0:**

- 100+ languages

**Usage:**

```python
import cohere
co = cohere.Client("YOUR_API_KEY")

response = co.embed(
    texts=["Text 1", "Text 2"],
    model="embed-english-v3.0",
    input_type="search_document"  # or "search_query"
)

embeddings = response.embeddings
```

**Unique feature:** Different embeddings for documents vs queries (optimized asymmetric search).

---

#### Open-Source Embeddings

**BGE (BAAI General Embedding):**

- bge-small-en-v1.5: 384 dims, fast
- bge-base-en-v1.5: 768 dims, balanced
- bge-large-en-v1.5: 1024 dims, best quality

**E5 (Microsoft):**

- e5-small-v2
- e5-base-v2
- e5-large-v2

**Instructor:**

- Task-specific instructions
- "Represent the Science question for retrieval"

**GTE (General Text Embeddings - Alibaba):**

- Strong multilingual performance

**Sentence-BERT / all-MiniLM:**

- Lightweight, fast
- 384 dimensions
- Good for prototyping

**Usage (via Hugging Face):**

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('BAAI/bge-small-en-v1.5')
embeddings = model.encode(["Text 1", "Text 2"])
```

---

### Dimensionality Considerations

**Trade-offs:**

| Dimensions | Pros             | Cons                |
| ---------- | ---------------- | ------------------- |
| **384**    | Fast, low memory | Less nuanced        |
| **768**    | Balanced         | Standard choice     |
| **1536**   | High quality     | More memory/compute |
| **3072**   | Best quality     | Expensive           |

**General rule:** Start with 768, increase if quality matters more than speed/cost.

---

### Embedding Best Practices

**1. Use same model for indexing and querying:**

```python
# Bad: Different models
docs_embeddings = openai.embed(docs)
query_embedding = bge.embed(query)  # Won't match well!

# Good: Same model
docs_embeddings = openai.embed(docs)
query_embedding = openai.embed(query)
```

**2. Normalize embeddings (for cosine similarity):**

```python
import numpy as np

def normalize(vec):
    return vec / np.linalg.norm(vec)

embedding = normalize(embedding)
```

**3. Consider query vs document asymmetry:**

- Queries: Short ("refund policy")
- Documents: Long (entire policy document)
- Some models (Cohere) handle this explicitly

**4. Prepend instructions (for Instructor embeddings):**

```python
query = "Represent this query for searching relevant passages: " + user_query
```

---

## Vector Databases

### What is a Vector Database?

**Purpose:** Store and search high-dimensional vectors efficiently.

**Core operation:** Find K nearest neighbors (KNN) to query vector.

**Why not regular DB?**

- Traditional DBs: Exact match (WHERE id = 123)
- Vector DBs: Similarity search (find similar vectors)

---

### Similarity Metrics

**1. Cosine Similarity:**
$$\text{cosine}(a, b) = \frac{a \cdot b}{\|a\| \|b\|}$$

Range: -1 to 1 (1 = identical, 0 = orthogonal, -1 = opposite)

**Most common for RAG.**

**2. Dot Product:**
$$\text{dot}(a, b) = a \cdot b$$

Faster than cosine (no normalization), but depends on magnitude.

**3. Euclidean Distance:**
$$\text{dist}(a, b) = \sqrt{\sum (a_i - b_i)^2}$$

Measures "distance" rather than "similarity" (lower = more similar).

**Which to use?**

- **Cosine similarity:** Default choice (normalized)
- **Dot product:** If embeddings are pre-normalized
- **Euclidean:** Rare for embeddings

---

### Vector Database Options

---

#### FAISS (Facebook AI Similarity Search)

**Type:** Library (local, not a server)

**Pros:**

- Extremely fast
- Production-proven (used by Facebook)
- Supports exact and approximate search
- Free and open-source

**Cons:**

- No built-in persistence (you manage storage)
- No multi-user support
- No filtering/metadata queries (basic version)

**When to use:**

- Prototyping
- Small datasets (<1M vectors)
- Local development
- Don't need managed service

**Usage:**

```python
import faiss
import numpy as np

# Create index
dimension = 768
index = faiss.IndexFlatL2(dimension)  # Exact search (L2 distance)

# Add vectors
vectors = np.random.rand(1000, dimension).astype('float32')
index.add(vectors)

# Search
query = np.random.rand(1, dimension).astype('float32')
k = 5
distances, indices = index.search(query, k)
```

**Advanced: Approximate search (faster for large datasets):**

```python
# IVF (Inverted File Index) + PQ (Product Quantization)
nlist = 100  # Number of clusters
m = 8        # Number of subquantizers
quantizer = faiss.IndexFlatL2(dimension)
index = faiss.IndexIVFPQ(quantizer, dimension, nlist, m, 8)

index.train(vectors)  # Required for IVF
index.add(vectors)

# Search with nprobe (check more clusters = better recall, slower)
index.nprobe = 10
distances, indices = index.search(query, k)
```

---

#### Pinecone

**Type:** Managed vector database (cloud)

**Pros:**

- Fully managed (no infrastructure)
- Metadata filtering
- Hybrid search (sparse + dense)
- Production-ready
- Easy to scale

**Cons:**

- Not free (pay per usage)
- Vendor lock-in
- Cold start on free tier

**When to use:**

- Production RAG systems
- Don't want to manage infrastructure
- Need metadata filtering
- Scale to millions of vectors

**Usage:**

```python
import pinecone

pinecone.init(api_key="YOUR_API_KEY", environment="us-west1-gcp")

# Create index
pinecone.create_index(
    name="rag-index",
    dimension=1536,
    metric="cosine"
)

index = pinecone.Index("rag-index")

# Upsert vectors
index.upsert(vectors=[
    ("id1", [0.1, 0.2, ...], {"text": "...", "source": "doc1.pdf"}),
    ("id2", [0.3, 0.4, ...], {"text": "...", "source": "doc2.pdf"}),
])

# Query with metadata filter
results = index.query(
    vector=[0.5, 0.6, ...],
    top_k=5,
    filter={"source": "doc1.pdf"}
)
```

**Pricing (Jan 2025):**

- Free tier: 1 index, 1GB storage
- Paid: $0.096/hour per pod

---

#### Weaviate

**Type:** Open-source vector database

**Pros:**

- Hybrid search (BM25 + vector)
- Flexible schema
- GraphQL API
- Self-hostable or managed cloud
- Built-in modules (OpenAI, Cohere, HuggingFace)

**Cons:**

- More complex setup than Pinecone
- Steeper learning curve

**When to use:**

- Need hybrid search
- Want open-source
- Complex metadata queries
- Self-hosting preferred

**Usage:**

```python
import weaviate

client = weaviate.Client("http://localhost:8080")

# Create schema
schema = {
    "class": "Document",
    "vectorizer": "text2vec-openai",
    "properties": [
        {"name": "content", "dataType": ["text"]},
        {"name": "source", "dataType": ["string"]},
    ]
}
client.schema.create_class(schema)

# Add documents (auto-vectorization)
client.data_object.create(
    class_name="Document",
    data_object={"content": "...", "source": "doc1.pdf"}
)

# Query (hybrid search)
result = client.query.get("Document", ["content", "source"]) \
    .with_hybrid(query="What's the refund policy?", alpha=0.5) \
    .with_limit(5) \
    .do()
```

---

#### Chroma

**Type:** Open-source, embedded vector database

**Pros:**

- Super simple to use
- Embeds in your app (like SQLite)
- Great for prototyping
- Free and open-source

**Cons:**

- Limited scale (not for millions of vectors)
- Basic features
- Not production-grade for large systems

**When to use:**

- Prototyping
- Small projects
- Learning RAG
- Don't need managed service

**Usage:**

```python
import chromadb

client = chromadb.Client()
collection = client.create_collection("docs")

# Add documents
collection.add(
    documents=["Doc 1 text", "Doc 2 text"],
    metadatas=[{"source": "doc1"}, {"source": "doc2"}],
    ids=["id1", "id2"]
)

# Query
results = collection.query(
    query_texts=["What's the policy?"],
    n_results=5
)
```

---

#### Qdrant

**Type:** Open-source vector database

**Pros:**

- Rust-based (very fast)
- Rich filtering
- Good documentation
- Self-hostable or cloud
- Payload storage (store original text)

**Cons:**

- Smaller community than Pinecone/Weaviate
- Less mature ecosystem

**Usage:**

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams

client = QdrantClient(":memory:")  # or "http://localhost:6333"

# Create collection
client.create_collection(
    collection_name="docs",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE)
)

# Upsert points
client.upsert(
    collection_name="docs",
    points=[
        {"id": 1, "vector": [0.1, 0.2, ...], "payload": {"text": "..."}},
        {"id": 2, "vector": [0.3, 0.4, ...], "payload": {"text": "..."}},
    ]
)

# Search
results = client.search(
    collection_name="docs",
    query_vector=[0.5, 0.6, ...],
    limit=5
)
```

---

#### Milvus

**Type:** Open-source, production-grade vector database

**Pros:**

- Handles billions of vectors
- GPU support
- High performance
- Kubernetes-native

**Cons:**

- Complex deployment
- Heavy (needs more resources)

**When to use:** Large-scale production (100M+ vectors)

---

#### pgvector (PostgreSQL extension)

**Type:** Vector search extension for PostgreSQL

**Pros:**

- Use existing PostgreSQL infrastructure
- Combine vector + traditional queries
- ACID transactions
- Familiar SQL interface

**Cons:**

- Slower than specialized vector DBs
- Limited to ~1M vectors efficiently

**When to use:**

- Already using PostgreSQL
- Small to medium datasets
- Need transactional guarantees

**Usage:**

```sql
-- Enable extension
CREATE EXTENSION vector;

-- Create table with vector column
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)
);

-- Insert
INSERT INTO documents (content, embedding) VALUES
    ('Doc text', '[0.1, 0.2, ...]');

-- Search by cosine similarity
SELECT content
FROM documents
ORDER BY embedding <=> '[0.5, 0.6, ...]'  -- <=> is cosine distance
LIMIT 5;
```

---

#### LanceDB

**Type:** Open-source, embedded vector database

**Pros:**

- Serverless (no separate service)
- Columnar storage (efficient)
- Version control for vectors
- Good for ML workflows

**Cons:**

- Newer (less mature)
- Smaller community

---

### Vector DB Comparison

| Database     | Best For                   | Scale | Hosting  | Cost     |
| ------------ | -------------------------- | ----- | -------- | -------- |
| **FAISS**    | Local dev, prototyping     | <1M   | Local    | Free     |
| **Pinecone** | Production, managed        | 100M+ | Cloud    | $$$      |
| **Weaviate** | Hybrid search, open-source | 10M+  | Both     | Free/$   |
| **Chroma**   | Prototyping, learning      | <100K | Embedded | Free     |
| **Qdrant**   | Production, self-host      | 10M+  | Both     | Free/$   |
| **Milvus**   | Large-scale, enterprise    | 1B+   | Both     | Free/$$$ |
| **pgvector** | Existing PostgreSQL        | <1M   | Self     | Free     |
| **LanceDB**  | ML workflows, versioning   | 10M+  | Embedded | Free     |

---

## Chunking Strategies

### Why Chunking Matters

**Problem:** Documents are too long for embedding models (token limits) and retrievers (too much irrelevant info).

**Solution:** Split documents into smaller chunks.

**Trade-off:**

- **Small chunks:** Precise retrieval, but lose context
- **Large chunks:** More context, but diluted relevance

---

### Fixed-Size Chunking

**Method:** Split by character/token count with overlap.

**Example:**

```python
def chunk_text(text, chunk_size=500, overlap=50):
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start += (chunk_size - overlap)
    return chunks
```

**Pros:**

- Simple
- Predictable chunk sizes
- Fast

**Cons:**

- May split mid-sentence or mid-paragraph
- Loses semantic boundaries

**When to use:** Quick prototyping, uniform text (e.g., legal documents)

---

### Semantic Chunking

**Method:** Split at natural boundaries (paragraphs, sentences, sections).

**Example:**

```python
def semantic_chunk(text):
    # Split by double newline (paragraphs)
    paragraphs = text.split('\n\n')

    chunks = []
    current_chunk = ""

    for para in paragraphs:
        if len(current_chunk) + len(para) < 1000:
            current_chunk += para + "\n\n"
        else:
            chunks.append(current_chunk.strip())
            current_chunk = para + "\n\n"

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks
```

**Pros:**

- Respects document structure
- Better readability
- More coherent chunks

**Cons:**

- Variable chunk sizes
- More complex

**When to use:** Structured documents (articles, reports)

---

### Recursive Character Splitting

**Method:** Try splitting by paragraphs, then sentences, then characters (fallback).

**LangChain implementation:**

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""]  # Try these in order
)

chunks = splitter.split_text(text)
```

**Why recursive:** Finds best split point while respecting boundaries.

---

### Document-Specific Chunking

**Markdown:**

```python
# Split by headers
## Section 1
Content...

## Section 2
Content...
```

**Code:**

```python
# Split by functions/classes
def function1():
    ...

def function2():
    ...
```

**HTML:**

```python
# Split by tags (<div>, <section>, <article>)
```

---

### Overlapping Windows

**Why:** Prevent losing context at chunk boundaries.

**Example:**

```
Chunk 1: [Word 1-500] with overlap 50
Chunk 2: [Word 451-950] with overlap 50
Chunk 3: [Word 901-1400]
```

**Overlap size:** Typically 10-20% of chunk size.

---

### Parent-Child Chunking

**Idea:** Retrieve small chunks, provide large chunks to LLM.

**Process:**

1. Split document into large chunks (parent)
2. Further split into small chunks (child)
3. Embed small chunks
4. When retrieving: Return parent chunk (more context)

**Example:**

```python
parent_chunk = "Full section of document..."  # 2000 tokens
child_chunks = [
    "First paragraph...",    # 200 tokens
    "Second paragraph...",   # 200 tokens
    # ...
]

# Index: child chunks (precise retrieval)
# Return: parent chunk (full context)
```

---

### Best Practices

**1. Experiment with chunk size:**

- Start with 500-1000 characters
- Test retrieval quality
- Adjust based on results

**2. Include metadata:**

```python
{
    "text": "Chunk content...",
    "source": "document.pdf",
    "page": 5,
    "section": "Refund Policy",
    "chunk_id": "doc1_chunk_5"
}
```

**3. Add context (page numbers, section titles):**

```
[Document: Refund Policy, Page 3]
Customers can request refunds within 30 days...
```

**4. Preserve structure:**

- Don't split bullet points across chunks
- Keep tables together
- Maintain header hierarchy

---

## Similarity Search

### Naive Approach (Brute Force)

```python
def brute_force_search(query_vec, all_vecs, k=5):
    similarities = []
    for i, doc_vec in enumerate(all_vecs):
        sim = cosine_similarity(query_vec, doc_vec)
        similarities.append((sim, i))

    similarities.sort(reverse=True)
    return similarities[:k]
```

**Complexity:** O(N) where N = number of documents

**Problem:** Slow for large datasets (millions of vectors).

---

### Approximate Nearest Neighbor (ANN)

**Goal:** Trade accuracy for speed.

**Algorithms:**

- **HNSW (Hierarchical Navigable Small World):** Graph-based, very fast
- **IVF (Inverted File Index):** Clustering-based
- **Product Quantization (PQ):** Compression-based

**Speed:** O(log N) instead of O(N)

**Trade-off:** May miss some relevant results (typically 95-99% recall).

---

### HNSW

**How it works:**

1. Build multi-layer graph
2. Navigate from top layer (coarse) to bottom (fine)
3. Find nearest neighbors efficiently

**Pros:**

- Fastest search
- High recall (98-99%)

**Cons:**

- Higher memory usage
- Slower indexing

**Used in:** Faiss, Qdrant, Weaviate

---

### IVF (Inverted File Index)

**How it works:**

1. Cluster vectors (e.g., 100 clusters)
2. Assign each vector to nearest cluster
3. Search: Find nearest clusters, search within them

**Pros:**

- Good balance speed/memory
- Configurable (nprobe = clusters to search)

**Cons:**

- Requires training
- Lower recall than HNSW

**Used in:** Faiss

---

### Product Quantization (PQ)

**How it works:**

1. Split vector into subvectors
2. Quantize each subvector (reduce precision)
3. Use lookup tables for fast distance computation

**Pros:**

- Massive compression (10-100x)
- Fast search

**Cons:**

- Lower accuracy
- Lossy compression

**Used in:** Faiss (often combined with IVF)

---

## Hybrid Search

### What is Hybrid Search?

**Idea:** Combine **sparse** (keyword) and **dense** (semantic) retrieval.

**Sparse (BM25):**

- Keyword matching (like Google search)
- "refund policy" matches "refund" and "policy"
- Fast, interpretable

**Dense (Vector):**

- Semantic matching
- "refund policy" matches "money back guarantee"
- Captures meaning, not just keywords

**Hybrid:** Best of both worlds.

---

### BM25

**What:** Probabilistic keyword matching (TF-IDF on steroids).

**Formula (simplified):**
$$\text{score} = \sum \text{TF}(term) \times \text{IDF}(term)$$

**Strengths:**

- Exact keyword matches
- Fast
- Works for abbreviations, codes (e.g., "Model XYZ")

**Weaknesses:**

- No semantic understanding
- Misses synonyms

---

### Reciprocal Rank Fusion (RRF)

**Problem:** How to combine BM25 and vector scores?

**Solution: RRF**

**Algorithm:**

```python
def reciprocal_rank_fusion(bm25_results, vector_results, k=60):
    scores = {}

    # Add BM25 scores
    for rank, doc_id in enumerate(bm25_results):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)

    # Add vector scores
    for rank, doc_id in enumerate(vector_results):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)

    # Sort by combined score
    ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return ranked
```

**Why it works:**

- Ranks matter, not absolute scores
- Robust to different score scales
- Simple and effective

---

### When to Use Hybrid Search

**Use hybrid when:**

- Documents have specific terms (product codes, acronyms)
- Users search with keywords (not natural language)
- Need best of both worlds

**Use vector-only when:**

- Conversational queries
- Semantic understanding is key
- No special terminology

---

## Re-ranking

### Why Re-ranking?

**Problem:** Initial retrieval (vector search) casts wide net. Results may have irrelevant docs.

**Solution:** Re-rank with more expensive but accurate model.

**Two-stage retrieval:**

1. **Stage 1 (fast):** Vector search → Get top 100 candidates
2. **Stage 2 (accurate):** Re-rank top 100 → Return top 5

---

### Cross-Encoder Rerankers

**Difference from embeddings:**

**Bi-encoder (embeddings):**

```
query_vec = encode(query)
doc_vec = encode(doc)
score = similarity(query_vec, doc_vec)
```

- Independent encodings
- Fast (can pre-compute doc embeddings)
- Less accurate

**Cross-encoder:**

```
score = model(query + doc)  # Joint encoding
```

- Attends to both query and doc
- Slow (must run for each pair)
- More accurate

---

### Popular Rerankers

**1. Cohere Rerank:**

```python
import cohere
co = cohere.Client("YOUR_API_KEY")

results = co.rerank(
    model="rerank-english-v2.0",
    query="What's the refund policy?",
    documents=candidate_docs,
    top_n=5
)
```

**2. BGE Reranker:**

```python
from sentence_transformers import CrossEncoder

model = CrossEncoder('BAAI/bge-reranker-large')
scores = model.predict([
    (query, doc1),
    (query, doc2),
    (query, doc3),
])

# Sort by score
ranked = sorted(zip(docs, scores), key=lambda x: x[1], reverse=True)
```

---

### When to Rerank

**Use re-ranking when:**

- Quality matters more than latency
- Have budget for extra API calls
- Initial retrieval has noise

**Skip re-ranking when:**

- Latency-critical (<200ms)
- High-quality initial retrieval
- Cost-constrained

---

## Advanced RAG Techniques

### Query Transformation

**Problem:** User queries are often poorly phrased for retrieval.

---

#### HyDE (Hypothetical Document Embeddings)

**Idea:** Generate hypothetical answer, then search for it.

**Process:**

1. Query: "What causes diabetes?"
2. LLM generates hypothetical answer: "Diabetes is caused by..."
3. Embed hypothetical answer
4. Search for similar documents

**Why it works:** Answers are more similar to documents than questions.

```python
# 1. Generate hypothetical answer
prompt = f"Answer this question: {query}"
hypothetical_answer = llm.generate(prompt)

# 2. Embed and search
query_embedding = embed(hypothetical_answer)
results = vector_db.search(query_embedding)
```

---

#### Multi-Query

**Idea:** Generate multiple variations of query, retrieve for all, merge results.

**Process:**

```python
original = "What's the return policy?"

# Generate variations
queries = llm.generate(f"Generate 3 variations of this query: {original}")
# → ["How do I return a product?", "Can I get my money back?", "Refund process?"]

# Retrieve for each
all_results = []
for q in queries:
    results = retrieve(q)
    all_results.extend(results)

# Deduplicate and re-rank
final_results = deduplicate_and_rank(all_results)
```

**Why it works:** Increases recall (catches more relevant docs).

---

### Contextual Compression

**Problem:** Retrieved chunks contain noise (irrelevant sentences).

**Solution:** Use LLM to extract only relevant parts.

**Process:**

```python
# 1. Retrieve
context = retrieve(query)  # Long document chunk

# 2. Compress
prompt = f"""
Extract only the sentences relevant to answering: {query}

Context:
{context}

Relevant sentences:
"""
compressed = llm.generate(prompt)

# 3. Use compressed context for final answer
```

**Trade-off:** Extra LLM call, but better context quality.

---

### Metadata Filtering

**Idea:** Filter by document attributes before/after retrieval.

**Example:**

```python
# Filter by date
results = vector_db.search(
    query_embedding,
    filter={"date": {"$gte": "2024-01-01"}}
)

# Filter by source
results = vector_db.search(
    query_embedding,
    filter={"source": "official_docs"}
)

# Filter by user permissions
results = vector_db.search(
    query_embedding,
    filter={"access_level": {"$lte": user.access_level}}
)
```

**Why it matters:**

- Enforce access control
- Time-sensitive queries
- Department-specific knowledge

---

### Agentic RAG

**Idea:** Let agent decide when/how to retrieve.

**Process:**

```python
def agentic_rag(query):
    # Agent decides: Do I need to retrieve?
    if agent.needs_retrieval(query):
        context = retrieve(query)

        # Agent decides: Is this enough?
        if agent.is_sufficient(context):
            return generate(query, context)
        else:
            # Refine query and retrieve again
            refined_query = agent.refine_query(query, context)
            additional_context = retrieve(refined_query)
            return generate(query, context + additional_context)
    else:
        # Answer from parametric knowledge
        return generate(query)
```

**Benefits:**

- Adaptive retrieval
- Handles complex queries
- Self-correcting

---

### GraphRAG

**Idea:** Build knowledge graph, use graph traversal + RAG.

**Architecture:**

```
Documents → Extract entities & relationships → Knowledge Graph

Query → Entity extraction → Graph traversal → Relevant subgraph → LLM
```

**Example:**

```
Query: "Who did Alice work with on Project X?"

Graph:
Alice -[WORKED_ON]-> Project X
Bob -[WORKED_ON]-> Project X
Project X -[RELATED_TO]-> Project Y

Traversal: Find all nodes connected to Alice and Project X
Result: Bob
```

**When to use:**

- Highly interconnected knowledge
- Need relationship reasoning
- Complex multi-hop questions

**Trade-off:** Requires graph construction (expensive), but powerful for complex queries.

---

### Self-RAG

**Idea:** Model decides when to retrieve and self-critiques outputs.

**Process:**

1. Generate initial answer
2. Critique: "Is this accurate? Do I need more info?"
3. If needed: Retrieve and revise
4. Repeat until satisfied

**Research-stage** (not widely used in production yet).

---

## Advanced RAG Patterns & Optimization

### Query Understanding and Transformation

**Problem:** User queries are often vague, ambiguous, or poorly phrased.

**Solution:** Transform query before retrieval for better results.

---

#### Query Classification

```python
class QueryClassifier:
    """
    Classify query type to route to appropriate retrieval strategy
    """
    def classify(self, query):
        prompt = f"""
        Classify this query into one of the following categories:
        - FACTUAL: Looking for specific facts
        - EXPLORATORY: Open-ended research question
        - PROCEDURAL: How-to or step-by-step instructions
        - COMPARISON: Comparing multiple things
        - TROUBLESHOOTING: Problem-solving

        Query: {query}
        Category:
        """
        return llm.generate(prompt)

# Usage
query = "How do I reset my password?"
category = classifier.classify(query)  # PROCEDURAL

# Route based on category
if category == "PROCEDURAL":
    # Use chunking that preserves step sequences
    chunks = semantic_chunking_with_steps(docs)
elif category == "COMPARISON":
    # Retrieve multiple relevant docs for comparison
    chunks = multi_doc_retrieval(query)
```

---

#### Query Expansion with LLMs

```python
def expand_query(query):
    """
    Generate multiple query variations to improve recall
    """
    prompt = f"""
    Given this user query, generate 3 semantically similar queries
    that could help find relevant information.

    Original: {query}

    Variations:
    1.
    2.
    3.
    """
    variations = llm.generate(prompt).split("\n")
    return [query] + variations

# Example
original = "How to improve model accuracy?"
expanded = expand_query(original)
# [
#   "How to improve model accuracy?",
#   "What techniques increase machine learning model performance?",
#   "Methods to reduce model error rate",
#   "Best practices for improving prediction accuracy"
# ]

# Retrieve with all variations
all_results = []
for q in expanded:
    results = vector_search(q, top_k=20)
    all_results.extend(results)

# Deduplicate and rank
final_results = deduplicate_and_rank(all_results, top_k=10)
```

---

#### Query Decomposition

**For complex multi-part questions:**

```python
def decompose_query(complex_query):
    """
    Break complex query into simpler sub-questions
    """
    prompt = f"""
    Break this complex question into simpler sub-questions.

    Complex question: {complex_query}

    Sub-questions:
    1.
    2.
    3.
    """
    return llm.generate(prompt).split("\n")

# Example
complex_q = "Compare the performance of GPT-4 and Claude 3 on coding tasks and explain which is better for production use"

sub_questions = decompose_query(complex_q)
# [
#   "What is GPT-4's performance on coding tasks?",
#   "What is Claude 3's performance on coding tasks?",
#   "What are the criteria for production use of LLMs?"
# ]

# Retrieve for each sub-question
sub_contexts = []
for sub_q in sub_questions:
    context = retrieve(sub_q, top_k=5)
    sub_contexts.append((sub_q, context))

# Synthesize final answer
final_answer = synthesize_answer(complex_q, sub_contexts)
```

---

### Advanced Chunking Strategies

#### Semantic Chunking with Embeddings

```python
def semantic_chunking(text, max_chunk_size=512, similarity_threshold=0.75):
    """
    Chunk based on semantic similarity between sentences
    """
    sentences = split_into_sentences(text)
    embeddings = [embed(s) for s in sentences]

    chunks = []
    current_chunk = [sentences[0]]
    current_chunk_embedding = embeddings[0]

    for i in range(1, len(sentences)):
        # Compute similarity with current chunk
        similarity = cosine_similarity(current_chunk_embedding, embeddings[i])

        # If similar and not too long, add to current chunk
        if similarity > similarity_threshold and len(" ".join(current_chunk)) < max_chunk_size:
            current_chunk.append(sentences[i])
            # Update chunk embedding (running average)
            current_chunk_embedding = np.mean([current_chunk_embedding, embeddings[i]], axis=0)
        else:
            # Start new chunk
            chunks.append(" ".join(current_chunk))
            current_chunk = [sentences[i]]
            current_chunk_embedding = embeddings[i]

    # Add last chunk
    chunks.append(" ".join(current_chunk))

    return chunks
```

---

#### Hierarchical Chunking

```python
class HierarchicalChunker:
    """
    Create multi-level chunks: Document → Section → Paragraph
    """
    def chunk_document(self, document):
        # Level 1: Document summary
        doc_summary = summarize(document)
        doc_embedding = embed(doc_summary)

        # Level 2: Sections
        sections = split_into_sections(document)
        section_chunks = []

        for section in sections:
            section_summary = summarize(section)
            section_embedding = embed(section_summary)

            # Level 3: Paragraphs
            paragraphs = split_into_paragraphs(section)
            para_chunks = []

            for para in paragraphs:
                para_embedding = embed(para)
                para_chunks.append({
                    "text": para,
                    "embedding": para_embedding,
                    "section": section_summary,
                    "document": doc_summary
                })

            section_chunks.append({
                "summary": section_summary,
                "embedding": section_embedding,
                "paragraphs": para_chunks
            })

        return {
            "summary": doc_summary,
            "embedding": doc_embedding,
            "sections": section_chunks
        }

    def hierarchical_retrieval(self, query, top_k=5):
        """
        Two-stage retrieval: First find relevant sections, then paragraphs
        """
        query_embedding = embed(query)

        # Stage 1: Find relevant sections
        section_scores = []
        for doc in self.documents:
            for section in doc["sections"]:
                score = cosine_similarity(query_embedding, section["embedding"])
                section_scores.append((score, section))

        top_sections = sorted(section_scores, key=lambda x: x[0], reverse=True)[:10]

        # Stage 2: Find relevant paragraphs within top sections
        para_scores = []
        for score, section in top_sections:
            for para in section["paragraphs"]:
                para_score = cosine_similarity(query_embedding, para["embedding"])
                para_scores.append((para_score, para))

        top_paras = sorted(para_scores, key=lambda x: x[0], reverse=True)[:top_k]

        return [para["text"] for score, para in top_paras]
```

---

### Re-ranking Models: Deep Dive

#### Cross-Encoder Re-ranking

```python
from sentence_transformers import CrossEncoder

class Reranker:
    def __init__(self, model_name="cross-encoder/ms-marco-MiniLM-L-6-v2"):
        """
        Load cross-encoder model for re-ranking
        """
        self.model = CrossEncoder(model_name)

    def rerank(self, query, candidates, top_k=5):
        """
        Rerank candidates using cross-encoder

        Args:
            query: User query string
            candidates: List of candidate documents
            top_k: Number of top results to return

        Returns:
            List of (score, document) tuples
        """
        # Create query-document pairs
        pairs = [(query, doc) for doc in candidates]

        # Score all pairs
        scores = self.model.predict(pairs)

        # Sort by score
        ranked = sorted(zip(scores, candidates), key=lambda x: x[0], reverse=True)

        return ranked[:top_k]

# Usage in RAG pipeline
def rag_with_reranking(query, top_k_retrieve=100, top_k_final=5):
    # Stage 1: Fast vector search (high recall)
    candidates = vector_search(query, top_k=top_k_retrieve)

    # Stage 2: Re-rank with cross-encoder (high precision)
    reranker = Reranker()
    reranked = reranker.rerank(query, candidates, top_k=top_k_final)

    # Stage 3: Generate answer
    context = "\n\n".join([doc for score, doc in reranked])
    answer = llm.generate(f"Context: {context}\n\nQuestion: {query}\n\nAnswer:")

    return answer

# Performance comparison:
# Without reranking: ~65% relevant docs in top-5
# With reranking: ~85% relevant docs in top-5
# Cost: +50ms latency, +$0.001 per query
```

---

#### LLM-as-Reranker

```python
def llm_rerank(query, candidates, top_k=5):
    """
    Use LLM to score and rerank candidates
    """
    scores = []

    for doc in candidates:
        prompt = f"""
        Rate the relevance of this document to the query on a scale of 0-10.

        Query: {query}

        Document: {doc[:500]}...

        Relevance score (0-10):
        """
        score = float(llm.generate(prompt, max_tokens=5))
        scores.append((score, doc))

    # Sort by score
    ranked = sorted(scores, key=lambda x: x[0], reverse=True)

    return ranked[:top_k]

# Trade-offs:
# Pros: Very accurate, understands nuance
# Cons: Expensive ($0.01-0.05 per query), slow (2-5s)
# Use case: High-value queries where accuracy is critical
```

---

### Hybrid Search Implementation

```python
class HybridSearch:
    def __init__(self, vector_db, bm25_index):
        self.vector_db = vector_db
        self.bm25_index = bm25_index

    def hybrid_search(self, query, top_k=10, alpha=0.5):
        """
        Combine vector search and BM25

        Args:
            query: Search query
            top_k: Number of results
            alpha: Weight for vector search (1-alpha for BM25)
                   0.0 = pure BM25, 1.0 = pure vector

        Returns:
            List of ranked documents
        """
        # Vector search
        vector_results = self.vector_db.search(query, top_k=top_k*2)
        vector_scores = {doc_id: score for doc_id, score in vector_results}

        # BM25 search
        bm25_results = self.bm25_index.search(query, top_k=top_k*2)
        bm25_scores = {doc_id: score for doc_id, score in bm25_results}

        # Normalize scores to [0, 1]
        vector_scores = normalize_scores(vector_scores)
        bm25_scores = normalize_scores(bm25_scores)

        # Combine scores
        all_doc_ids = set(vector_scores.keys()) | set(bm25_scores.keys())
        combined_scores = {}

        for doc_id in all_doc_ids:
            v_score = vector_scores.get(doc_id, 0.0)
            b_score = bm25_scores.get(doc_id, 0.0)
            combined_scores[doc_id] = alpha * v_score + (1 - alpha) * b_score

        # Rank by combined score
        ranked = sorted(combined_scores.items(), key=lambda x: x[1], reverse=True)

        return ranked[:top_k]

def normalize_scores(scores):
    """Min-max normalization"""
    if not scores:
        return {}
    min_score = min(scores.values())
    max_score = max(scores.values())
    if max_score == min_score:
        return {k: 1.0 for k in scores}
    return {
        k: (v - min_score) / (max_score - min_score)
        for k, v in scores.items()
    }

# Example usage
hybrid = HybridSearch(vector_db, bm25_index)

# Query with specific term (favor BM25)
results1 = hybrid.hybrid_search("product SKU-12345", alpha=0.3)  # 70% BM25

# Semantic query (favor vector)
results2 = hybrid.hybrid_search("How to improve customer satisfaction?", alpha=0.8)  # 80% vector

# Balanced
results3 = hybrid.hybrid_search("company policy on refunds", alpha=0.5)  # 50/50
```

---

### RAG Evaluation Framework

```python
class RAGEvaluator:
    """
    Comprehensive RAG evaluation metrics
    """
    def __init__(self, llm):
        self.llm = llm

    def evaluate_retrieval(self, query, retrieved_docs, ground_truth_docs):
        """
        Evaluate retrieval quality
        """
        # Precision: How many retrieved docs are relevant?
        precision = len(set(retrieved_docs) & set(ground_truth_docs)) / len(retrieved_docs)

        # Recall: How many relevant docs were retrieved?
        recall = len(set(retrieved_docs) & set(ground_truth_docs)) / len(ground_truth_docs)

        # F1 score
        f1 = 2 * (precision * recall) / (precision + recall) if (precision + recall) > 0 else 0

        # MRR (Mean Reciprocal Rank)
        for i, doc in enumerate(retrieved_docs):
            if doc in ground_truth_docs:
                mrr = 1 / (i + 1)
                break
        else:
            mrr = 0

        return {
            "precision": precision,
            "recall": recall,
            "f1": f1,
            "mrr": mrr
        }

    def evaluate_answer_quality(self, query, answer, ground_truth_answer):
        """
        Evaluate generated answer quality using LLM
        """
        # 1. Faithfulness: Is answer supported by retrieved context?
        faithfulness_prompt = f"""
        Rate how well this answer is supported by the context (0-10).

        Context: {context}
        Answer: {answer}

        Score (0-10):
        """
        faithfulness = float(self.llm.generate(faithfulness_prompt, max_tokens=5))

        # 2. Relevance: Does answer address the question?
        relevance_prompt = f"""
        Rate how well this answer addresses the question (0-10).

        Question: {query}
        Answer: {answer}

        Score (0-10):
        """
        relevance = float(self.llm.generate(relevance_prompt, max_tokens=5))

        # 3. Correctness: Compare to ground truth
        correctness_prompt = f"""
        Compare this answer to the ground truth. Rate accuracy (0-10).

        Ground truth: {ground_truth_answer}
        Generated answer: {answer}

        Score (0-10):
        """
        correctness = float(self.llm.generate(correctness_prompt, max_tokens=5))

        return {
            "faithfulness": faithfulness,
            "relevance": relevance,
            "correctness": correctness,
            "average": (faithfulness + relevance + correctness) / 3
        }

    def evaluate_end_to_end(self, test_cases):
        """
        Evaluate complete RAG pipeline on test set
        """
        results = []

        for case in test_cases:
            query = case["query"]
            ground_truth_docs = case["relevant_docs"]
            ground_truth_answer = case["answer"]

            # Run RAG pipeline
            retrieved_docs = rag_retrieve(query)
            generated_answer = rag_generate(query, retrieved_docs)

            # Evaluate retrieval
            retrieval_metrics = self.evaluate_retrieval(
                query, retrieved_docs, ground_truth_docs
            )

            # Evaluate answer
            answer_metrics = self.evaluate_answer_quality(
                query, generated_answer, ground_truth_answer
            )

            results.append({
                "query": query,
                "retrieval": retrieval_metrics,
                "answer": answer_metrics
            })

        # Aggregate metrics
        avg_precision = np.mean([r["retrieval"]["precision"] for r in results])
        avg_recall = np.mean([r["retrieval"]["recall"] for r in results])
        avg_answer_quality = np.mean([r["answer"]["average"] for r in results])

        return {
            "avg_precision": avg_precision,
            "avg_recall": avg_recall,
            "avg_f1": 2 * (avg_precision * avg_recall) / (avg_precision + avg_recall),
            "avg_answer_quality": avg_answer_quality,
            "detailed_results": results
        }

# Usage
evaluator = RAGEvaluator(llm)

test_cases = [
    {
        "query": "What is the refund policy?",
        "relevant_docs": ["doc1", "doc5"],
        "answer": "Refunds are available within 30 days..."
    },
    # ... more test cases
]

evaluation = evaluator.evaluate_end_to_end(test_cases)
print(f"Precision: {evaluation['avg_precision']:.2f}")
print(f"Recall: {evaluation['avg_recall']:.2f}")
print(f"Answer Quality: {evaluation['avg_answer_quality']:.2f}")
```

---

### Production RAG Optimization

#### Caching Strategy

```python
class RAGCache:
    """
    Multi-level caching for RAG pipeline
    """
    def __init__(self, redis_client):
        self.redis = redis_client
        self.query_cache = {}  # In-memory cache
        self.embedding_cache = {}

    def get_cached_answer(self, query):
        """
        Check if we've answered this exact query before
        """
        # Level 1: In-memory (fastest)
        if query in self.query_cache:
            return self.query_cache[query]

        # Level 2: Redis (fast)
        cached = self.redis.get(f"answer:{query}")
        if cached:
            answer = json.loads(cached)
            self.query_cache[query] = answer  # Populate L1 cache
            return answer

        return None

    def cache_answer(self, query, answer, ttl=3600):
        """
        Cache answer at multiple levels
        """
        # L1: In-memory
        self.query_cache[query] = answer

        # L2: Redis with TTL
        self.redis.setex(
            f"answer:{query}",
            ttl,
            json.dumps(answer)
        )

    def get_similar_cached_queries(self, query_embedding, threshold=0.95):
        """
        Semantic cache: Find similar queries we've answered
        """
        for cached_query, cached_embedding in self.embedding_cache.items():
            similarity = cosine_similarity(query_embedding, cached_embedding)
            if similarity > threshold:
                return self.get_cached_answer(cached_query)

        return None

# Usage in RAG pipeline
cache = RAGCache(redis_client)

def rag_with_cache(query):
    # Check exact match cache
    cached_answer = cache.get_cached_answer(query)
    if cached_answer:
        return cached_answer  # Cache hit!

    # Check semantic cache
    query_embedding = embed(query)
    similar_answer = cache.get_similar_cached_queries(query_embedding)
    if similar_answer:
        return similar_answer

    # Cache miss: Run full RAG pipeline
    context = retrieve(query)
    answer = generate(query, context)

    # Cache for future
    cache.cache_answer(query, answer)

    return answer
```

---

#### Batch Processing for Efficiency

```python
def batch_rag(queries, batch_size=32):
    """
    Process multiple queries efficiently
    """
    all_answers = []

    for i in range(0, len(queries), batch_size):
        batch = queries[i:i+batch_size]

        # Batch embedding (much faster than one-by-one)
        embeddings = embed_batch(batch)

        # Parallel retrieval
        with ThreadPoolExecutor(max_workers=8) as executor:
            contexts = list(executor.map(retrieve, embeddings))

        # Batch LLM generation
        prompts = [
            f"Context: {ctx}\n\nQuestion: {q}\n\nAnswer:"
            for q, ctx in zip(batch, contexts)
        ]
        answers = llm.generate_batch(prompts)

        all_answers.extend(answers)

    return all_answers

# Speedup: 5-10x faster than sequential processing
```

---

## Production RAG Pitfalls

### Common Issues

**1. Chunking errors:**

- Splitting mid-sentence
- Losing context
- **Fix:** Use semantic chunking, add overlap

**2. Embedding mismatch:**

- Different models for indexing vs query
- **Fix:** Use same model consistently

**3. Poor retrieval:**

- Top-k results not relevant
- **Fix:** Increase k, try hybrid search, add reranking

**4. Hallucinations:**

- LLM ignores context, makes stuff up
- **Fix:** Prompt "Only use provided context", add grounding checks

**5. Context window overflow:**

- Too many chunks exceed context limit
- **Fix:** Limit top-k, compress context, use longer context models

**6. Metadata loss:**

- Can't cite sources
- **Fix:** Include source metadata, add citation instructions

**7. Stale data:**

- Vector DB not updated
- **Fix:** Implement refresh pipeline

---

### Handling Hallucinations

**Strategies:**

**1. Prompt engineering:**

```
You are a helpful assistant. Answer based ONLY on the provided context.
If the context doesn't contain enough information, say "I don't have enough information to answer that."

Context:
{context}

Question: {question}
```

**2. Grounding check:**

```python
def is_grounded(answer, context):
    prompt = f"""
    Is this answer supported by the context?

    Answer: {answer}
    Context: {context}

    Output: Yes or No
    """
    result = llm.generate(prompt)
    return result.strip().lower() == "yes"
```

**3. Citation requirement:**

```
Answer the question and cite sources using [1], [2], etc.

Context:
[1] Document 1...
[2] Document 2...

Question: {question}
Answer with citations:
```

---

### Evaluation Strategies

**Retrieval metrics:**

- **Precision@K:** % of retrieved docs that are relevant
- **Recall@K:** % of relevant docs that were retrieved
- **MRR (Mean Reciprocal Rank):** How early is the first relevant doc?
- **NDCG:** Ranking quality

**Generation metrics:**

- **Groundedness:** Is answer supported by context?
- **Relevance:** Does answer address question?
- **Coherence:** Is answer well-written?
- **Citation accuracy:** Are citations correct?

**End-to-end:**

- Human evaluation (gold standard)
- LLM-as-judge (GPT-4 scores outputs)

---

## When NOT to Use RAG

**Don't use RAG when:**

1. **Knowledge already in model:**
   - General facts (capitals, dates)
   - Common sense reasoning
   - Use prompting

2. **Behavior change needed (not knowledge):**
   - Tone, style, format
   - Use fine-tuning or prompting

3. **Small dataset:**
   - <10 documents
   - Just paste in prompt

4. **Real-time data with API:**
   - Weather, stock prices
   - Use function calling / tool use

5. **Latency-critical:**
   - RAG adds 100-500ms
   - Consider caching or hybrid approach

---

## RAG System Checklist

**Offline (Indexing):**

- [ ] Load documents (PDF, HTML, Markdown, etc.)
- [ ] Clean and preprocess text
- [ ] Chunk appropriately (size, overlap, semantic boundaries)
- [ ] Add metadata (source, date, section, access control)
- [ ] Generate embeddings (same model as query)
- [ ] Store in vector DB with metadata

**Online (Query):**

- [ ] Preprocess query (lowercase, remove stopwords?)
- [ ] Embed query
- [ ] Retrieve top-K candidates (K=20-100)
- [ ] (Optional) Filter by metadata
- [ ] (Optional) Rerank top candidates
- [ ] Select top-N for LLM (N=3-10)
- [ ] Construct prompt (system + context + query)
- [ ] Generate answer
- [ ] (Optional) Validate grounding
- [ ] Return answer + citations

**Monitoring:**

- [ ] Log queries, retrieved docs, answers
- [ ] Track latency (retrieval, generation, total)
- [ ] Monitor relevance (user feedback)
- [ ] Detect hallucinations
- [ ] Measure cost (embedding + LLM calls)

---

## Interview Questions

**Junior:**

- What is RAG and why is it useful?
- Explain the RAG pipeline at a high level
- What's the difference between RAG and fine-tuning?
- Why do we chunk documents?
- What's cosine similarity?

**Mid-level:**

- Design a RAG system for company documentation (10K documents)
- Explain different chunking strategies and trade-offs
- What's hybrid search and when should you use it?
- How does re-ranking improve RAG?
- Describe query transformation techniques (HyDE, multi-query)
- How would you handle hallucinations in RAG?

**Senior:**

- Design a RAG system for 10M+ documents with multi-tenancy
- Compare vector databases (Pinecone vs Weaviate vs pgvector)
- Optimize RAG for latency (p99 < 500ms)
- Implement access control in RAG
- Design evaluation strategy for RAG system
- When would you choose fine-tuning over RAG?

---

## Key Takeaways

1. **RAG = most common production pattern** (90% of GenAI apps use it)
2. **Pipeline: Chunk → Embed → Store → Retrieve → Generate**
3. **Embeddings capture meaning:** Similar text → similar vectors
4. **Vector DBs enable fast search:** Pinecone, Weaviate, FAISS
5. **Chunking is an art:** Balance context vs precision
6. **Hybrid search > pure vector:** Combine BM25 + embeddings
7. **Re-ranking improves quality:** Worth the extra cost
8. **Metadata matters:** Filtering, citations, access control
9. **Hallucinations are real:** Prompt engineering, grounding checks
10. **RAG before fine-tuning:** Cheaper, faster, easier to update

---

**Next:** [Part 7: Fine-tuning & Adaptation](07_Fine_Tuning_Adaptation.md)
