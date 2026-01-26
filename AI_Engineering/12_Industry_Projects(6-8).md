# Part 12: Industry Projects (Continued)

## Project 6: Multimodal Search Engine {#project-6-multimodal-search}

### Problem Statement

**Build a search engine that handles text queries and returns relevant images, and vice versa (image queries return relevant text/images).**

**Requirements:**

- Text → Image search ("find sunset photos")
- Image → Image search (upload photo, find similar)
- Image → Text search (upload product photo, find descriptions)
- Hybrid ranking (combine text and visual similarity)
- <1s search time
- Scale to 1M images + text documents
- <$0.001 per search

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      User Query                              │
│              (Text: "red sports car")                        │
│                    OR                                        │
│              (Image: uploaded photo)                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Query Encoder (CLIP/SigLIP)                     │
│         Text query → 512-dim embedding                       │
│         Image query → 512-dim embedding                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│           Vector Search (Pinecone/Qdrant)                    │
│      Search in unified embedding space                       │
│      (images + text documents as vectors)                    │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Image     │  │    Text     │  │   Hybrid    │
│  Results    │  │   Results   │  │  Re-ranker  │
│             │  │             │  │  (Optional) │
└─────────────┘  └─────────────┘  └─────────────┘
         │               │               │
         └───────────────┼───────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                 Return Top K Results                         │
│         (Images with captions, text snippets)                │
└─────────────────────────────────────────────────────────────┘
```

### Tech Stack

```yaml
Embeddings:
  - Model: CLIP (OpenAI ViT-L/14) or SigLIP (Google)
  - Dimension: 512 or 768
  - Framework: transformers (Hugging Face)

Vector DB:
  - Qdrant (supports payloads with metadata)
  - Alternative: Pinecone, Weaviate

Storage:
  - Images: AWS S3
  - Metadata: PostgreSQL (image URLs, captions, tags)

Backend:
  - FastAPI (Python)
  - Image preprocessing: PIL, torchvision

Monitoring:
  - Prometheus + Grafana
```

### Implementation

**Step 1: Index Images and Text**

```python
from transformers import CLIPProcessor, CLIPModel
import torch
from PIL import Image
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

# Load CLIP model
model = CLIPModel.from_pretrained("openai/clip-vit-large-patch14")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-large-patch14")

# Initialize Qdrant
client = QdrantClient(host="localhost", port=6333)

# Create collection
client.create_collection(
    collection_name="multimodal_search",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
)

def encode_image(image_path):
    """Encode image to CLIP embedding."""
    image = Image.open(image_path).convert("RGB")
    inputs = processor(images=image, return_tensors="pt")
    
    with torch.no_grad():
        image_features = model.get_image_features(**inputs)
        # Normalize
        image_features = image_features / image_features.norm(dim=-1, keepdim=True)
    
    return image_features[0].numpy()

def encode_text(text):
    """Encode text to CLIP embedding."""
    inputs = processor(text=[text], return_tensors="pt", padding=True)
    
    with torch.no_grad():
        text_features = model.get_text_features(**inputs)
        text_features = text_features / text_features.norm(dim=-1, keepdim=True)
    
    return text_features[0].numpy()

def index_image(image_id, image_path, caption, tags):
    """Index image with metadata."""
    embedding = encode_image(image_path)
    
    client.upsert(
        collection_name="multimodal_search",
        points=[
            PointStruct(
                id=image_id,
                vector=embedding.tolist(),
                payload={
                    "type": "image",
                    "image_url": upload_to_s3(image_path),
                    "caption": caption,
                    "tags": tags,
                },
            )
        ],
    )

def index_text_document(doc_id, text, title):
    """Index text document."""
    embedding = encode_text(text)
    
    client.upsert(
        collection_name="multimodal_search",
        points=[
            PointStruct(
                id=doc_id,
                vector=embedding.tolist(),
                payload={
                    "type": "text",
                    "title": title,
                    "content": text[:500],  # Store snippet
                },
            )
        ],
    )
```

**Step 2: Search Functions**

```python
def search_by_text(query, top_k=10, filter_type=None):
    """
    Search using text query.
    
    Args:
        query: Text query (e.g., "red sports car")
        top_k: Number of results
        filter_type: "image" or "text" or None (both)
    """
    # Encode query
    query_embedding = encode_text(query)
    
    # Build filter
    search_filter = None
    if filter_type:
        search_filter = {"type": filter_type}
    
    # Search
    results = client.search(
        collection_name="multimodal_search",
        query_vector=query_embedding.tolist(),
        limit=top_k,
        query_filter=search_filter,
    )
    
    return format_results(results)

def search_by_image(image_path, top_k=10):
    """Search using image query."""
    query_embedding = encode_image(image_path)
    
    results = client.search(
        collection_name="multimodal_search",
        query_vector=query_embedding.tolist(),
        limit=top_k,
    )
    
    return format_results(results)

def format_results(results):
    """Format search results."""
    formatted = []
    for hit in results:
        formatted.append({
            "id": hit.id,
            "score": hit.score,
            "type": hit.payload["type"],
            "data": hit.payload,
        })
    return formatted
```

**Step 3: Hybrid Re-ranking (Optional)**

```python
def hybrid_search(text_query, image_query=None, top_k=10):
    """
    Combine text and image query for hybrid search.
    Example: User types "sports car" and uploads a color reference image.
    """
    # Get candidates from text query
    text_results = search_by_text(text_query, top_k=50)
    
    if image_query:
        # Get candidates from image query
        image_results = search_by_image(image_query, top_k=50)
        
        # Merge and re-rank (weighted combination)
        combined_scores = {}
        
        for result in text_results:
            combined_scores[result["id"]] = result["score"] * 0.6
        
        for result in image_results:
            if result["id"] in combined_scores:
                combined_scores[result["id"]] += result["score"] * 0.4
            else:
                combined_scores[result["id"]] = result["score"] * 0.4
        
        # Sort by combined score
        ranked_ids = sorted(combined_scores.items(), key=lambda x: x[1], reverse=True)
        
        # Fetch top K
        final_results = []
        for doc_id, score in ranked_ids[:top_k]:
            # Retrieve full payload
            point = client.retrieve(
                collection_name="multimodal_search",
                ids=[doc_id],
            )[0]
            final_results.append({
                "id": doc_id,
                "score": score,
                "type": point.payload["type"],
                "data": point.payload,
            })
        
        return final_results
    
    return text_results[:top_k]
```

**Step 4: FastAPI Endpoints**

```python
from fastapi import FastAPI, File, UploadFile, Form
from pydantic import BaseModel

app = FastAPI()

class TextSearchRequest(BaseModel):
    query: str
    top_k: int = 10
    filter_type: str = None

@app.post("/search/text")
async def text_search(request: TextSearchRequest):
    """Text-based search."""
    results = search_by_text(
        query=request.query,
        top_k=request.top_k,
        filter_type=request.filter_type,
    )
    return {"results": results}

@app.post("/search/image")
async def image_search(
    file: UploadFile = File(...),
    top_k: int = Form(10),
):
    """Image-based search."""
    # Save uploaded image temporarily
    temp_path = f"/tmp/{file.filename}"
    with open(temp_path, "wb") as f:
        f.write(await file.read())
    
    results = search_by_image(temp_path, top_k=top_k)
    
    # Clean up
    os.remove(temp_path)
    
    return {"results": results}

@app.post("/search/hybrid")
async def hybrid_search_endpoint(
    text_query: str = Form(...),
    image: UploadFile = File(None),
    top_k: int = Form(10),
):
    """Hybrid text + image search."""
    image_path = None
    if image:
        image_path = f"/tmp/{image.filename}"
        with open(image_path, "wb") as f:
            f.write(await image.read())
    
    results = hybrid_search(text_query, image_path, top_k)
    
    if image_path:
        os.remove(image_path)
    
    return {"results": results}
```

**Step 5: Indexing Pipeline**

```python
def batch_index_images(image_dir, batch_size=100):
    """Index all images in a directory."""
    import os
    from tqdm import tqdm
    
    image_files = [f for f in os.listdir(image_dir) if f.endswith(('.jpg', '.png'))]
    
    for i in tqdm(range(0, len(image_files), batch_size)):
        batch = image_files[i:i + batch_size]
        
        for img_file in batch:
            image_path = os.path.join(image_dir, img_file)
            
            # Generate caption (optional: use BLIP or GPT-4V)
            caption = generate_caption(image_path)
            
            # Index
            index_image(
                image_id=img_file,
                image_path=image_path,
                caption=caption,
                tags=[],  # Extract from metadata or use vision model
            )
```

### Challenges & Solutions

**Challenge 1: Scale (1M images)**

**Problem:** Embedding 1M images takes time; searching needs to be fast.

**Solution:**

- Offline indexing: Batch process images (use GPU for CLIP encoding)
- Vector DB: Qdrant with HNSW index (fast approximate search)
- Quantization: Use INT8 embeddings (2× smaller, minimal accuracy loss)

**Challenge 2: Cost (<$0.001 per search)**

```
Cost breakdown:
- CLIP inference (self-hosted): Free (one-time encoding)
- Vector search: $0.0001 per query (Qdrant Cloud)
- Total: $0.0001 ✅ (10× under budget)
```

**Challenge 3: Relevance**

**Problem:** Text query "dog" returns all dog images, but user wants "golden retriever running".

**Solution:**

- Fine-tune CLIP on domain-specific data
- Use detailed captions (BLIP-2 or GPT-4V for captioning)
- Add metadata filters (color, category, date)

### Evaluation

**Metrics:**

1. **Retrieval accuracy:**
   - Precision@10: 0.82 (82% of top 10 results are relevant)
   - Recall@100: 0.91 (91% of relevant items in top 100)

2. **Performance:**
   - Search latency: 250ms (p95) ✅
   - Index size: 1M vectors × 768 dims × 4 bytes = 3GB

3. **Cost:**
   - Indexing: 1M images × $0.00001 (CLIP) = $10 (one-time)
   - Search: $0.0001/query

### Interview Discussion Points

**Mid-level:**

- "Why CLIP instead of separate text and image encoders?"
  - CLIP is trained jointly (text-image pairs)
  - Same embedding space for both modalities
  - Enables cross-modal search (text query → image results)

- "How do you handle different image sizes?"
  - CLIP preprocessor resizes to 224×224
  - Maintains aspect ratio with center crop

**Senior-level:**

- "Scale to 100M images."
  - Sharding: Partition by category (products, photos, etc.)
  - Distributed Qdrant cluster
  - Approximate search (HNSW with lower ef_search for speed)
  - Cost: 100M vectors × $0.0001 = $10K/month

- "Add video search."
  - Extract keyframes (1 frame/sec)
  - Embed each frame with CLIP
  - Aggregate frame embeddings (average pooling)
  - Temporal metadata (timestamp for each frame)

---

## Project 7: Natural Language to SQL {#project-7-nl-to-sql}

### Problem Statement

**Build a system that converts natural language questions to SQL queries for business analytics.**

**Example:**
- User: "What were our top 5 products by revenue last month?"
- System: `SELECT product_name, SUM(revenue) FROM sales WHERE month = '2024-12' GROUP BY product_name ORDER BY SUM(revenue) DESC LIMIT 5;`

**Requirements:**

- Support complex queries (joins, aggregations, subqueries)
- Handle ambiguous questions (ask clarifying questions)
- Validate SQL before execution
- <3s response time
- 90%+ SQL accuracy
- Support multiple database schemas

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              User Question (Natural Language)                │
│         "What were top 5 products last month?"              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│           Schema Retrieval (Vector Search)                   │
│     Find relevant tables/columns for question                │
│     (Embed table names, descriptions)                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Few-Shot SQL Generation (GPT-4o)                │
│     Prompt: Schema + Examples + Question → SQL              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   SQL Validator                              │
│     1. Parse SQL (syntax check)                              │
│     2. Check permissions (SELECT only)                       │
│     3. Dry run (EXPLAIN query)                               │
└────────────────────────┬────────────────────────────────────┘
                         │
               ┌─────────┴─────────┐
               │                   │
               ▼                   ▼
          Valid SQL          Invalid SQL
               │                   │
               │                   ▼
               │          ┌─────────────────┐
               │          │  Self-Correct   │
               │          │  (Retry with    │
               │          │   error msg)    │
               │          └────────┬────────┘
               │                   │
               └───────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                Execute Query (Read-only DB)                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│          Format Results (Table/Chart/Summary)                │
└─────────────────────────────────────────────────────────────┘
```

### Tech Stack

```yaml
LLM:
  - SQL generation: gpt-4o
  - Schema retrieval: text-embedding-3-small

Database:
  - Production DB: PostgreSQL (read-only replica)
  - Metadata store: PostgreSQL (schema descriptions)

Vector DB:
  - Pinecone (embed table/column descriptions)

Backend:
  - FastAPI (Python)
  - SQL parsing: sqlparse
  - Query execution: psycopg2

Validation:
  - SQL syntax: sqlparse
  - Permissions: Custom middleware
```

### Implementation

**Step 1: Schema Embedding**

```python
import openai
from pinecone import Pinecone

# Initialize
pc = Pinecone(api_key="YOUR_KEY")
index = pc.Index("sql-schemas")

def embed_schema():
    """Embed all tables and columns for retrieval."""
    
    # Get schema from database
    schema = get_database_schema()  # {table_name: {columns: [...], description: "..."}}
    
    for table_name, table_info in schema.items():
        # Create text description
        text = f"Table: {table_name}\n"
        text += f"Description: {table_info['description']}\n"
        text += "Columns:\n"
        
        for col in table_info['columns']:
            text += f"  - {col['name']} ({col['type']}): {col['description']}\n"
        
        # Embed
        embedding = openai.Embedding.create(
            model="text-embedding-3-small",
            input=[text],
        ).data[0].embedding
        
        # Store in Pinecone
        index.upsert([(
            table_name,
            embedding,
            {"table_name": table_name, "description": text},
        )])

def get_database_schema():
    """Fetch schema from database."""
    # This is a simplified example
    return {
        "sales": {
            "description": "Sales transactions",
            "columns": [
                {"name": "id", "type": "INT", "description": "Primary key"},
                {"name": "product_name", "type": "VARCHAR", "description": "Product name"},
                {"name": "revenue", "type": "DECIMAL", "description": "Revenue in USD"},
                {"name": "month", "type": "DATE", "description": "Transaction month"},
            ],
        },
        "customers": {
            "description": "Customer information",
            "columns": [
                {"name": "id", "type": "INT", "description": "Customer ID"},
                {"name": "name", "type": "VARCHAR", "description": "Customer name"},
                {"name": "country", "type": "VARCHAR", "description": "Country"},
            ],
        },
    }
```

**Step 2: Retrieve Relevant Schema**

```python
def retrieve_relevant_schema(question, top_k=3):
    """Find relevant tables for the question."""
    
    # Embed question
    embedding = openai.Embedding.create(
        model="text-embedding-3-small",
        input=[question],
    ).data[0].embedding
    
    # Search Pinecone
    results = index.query(
        vector=embedding,
        top_k=top_k,
        include_metadata=True,
    )
    
    # Extract table descriptions
    relevant_tables = []
    for match in results.matches:
        relevant_tables.append(match.metadata["description"])
    
    return "\n\n".join(relevant_tables)
```

**Step 3: SQL Generation with Few-Shot**

```python
FEW_SHOT_EXAMPLES = """
Example 1:
Question: Show me total revenue by product
SQL: SELECT product_name, SUM(revenue) AS total_revenue FROM sales GROUP BY product_name ORDER BY total_revenue DESC;

Example 2:
Question: Which customers from USA bought more than $1000?
SQL: SELECT c.name, SUM(s.revenue) AS total FROM customers c JOIN sales s ON c.id = s.customer_id WHERE c.country = 'USA' GROUP BY c.name HAVING SUM(s.revenue) > 1000;

Example 3:
Question: What's the average order value last quarter?
SQL: SELECT AVG(revenue) FROM sales WHERE month >= '2024-10-01' AND month <= '2024-12-31';
"""

def generate_sql(question, schema):
    """Generate SQL using GPT-4o."""
    
    prompt = f"""
You are a SQL expert. Convert natural language questions to SQL queries.

Database Schema:
{schema}

{FEW_SHOT_EXAMPLES}

Question: {question}
SQL:"""

    response = openai.ChatCompletion.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "You are a SQL expert. Generate only valid SQL queries."},
            {"role": "user", "content": prompt},
        ],
        temperature=0,
        max_tokens=300,
    )
    
    sql = response.choices[0].message.content.strip()
    
    # Clean SQL (remove markdown code blocks if present)
    sql = sql.replace("```sql", "").replace("```", "").strip()
    
    return sql
```

**Step 4: SQL Validation**

```python
import sqlparse
import re

def validate_sql(sql):
    """Validate SQL query."""
    
    # 1. Parse SQL (syntax check)
    try:
        parsed = sqlparse.parse(sql)
        if not parsed:
            return False, "Invalid SQL syntax"
    except Exception as e:
        return False, f"Parse error: {str(e)}"
    
    # 2. Check for dangerous operations
    dangerous_keywords = ["DROP", "DELETE", "UPDATE", "INSERT", "TRUNCATE", "ALTER"]
    sql_upper = sql.upper()
    
    for keyword in dangerous_keywords:
        if re.search(rf'\b{keyword}\b', sql_upper):
            return False, f"Forbidden operation: {keyword}"
    
    # 3. Ensure SELECT only
    if not sql_upper.strip().startswith("SELECT"):
        return False, "Query must start with SELECT"
    
    # 4. Dry run (EXPLAIN)
    try:
        with psycopg2.connect(**DB_CONFIG) as conn:
            with conn.cursor() as cur:
                cur.execute(f"EXPLAIN {sql}")
    except Exception as e:
        return False, f"Query execution error: {str(e)}"
    
    return True, "Valid"
```

**Step 5: Self-Correction Loop**

```python
def generate_sql_with_retry(question, schema, max_retries=3):
    """Generate SQL with self-correction."""
    
    for attempt in range(max_retries):
        # Generate SQL
        sql = generate_sql(question, schema)
        
        # Validate
        is_valid, error_msg = validate_sql(sql)
        
        if is_valid:
            return sql, None
        
        # If invalid, ask LLM to fix
        if attempt < max_retries - 1:
            sql = fix_sql(sql, error_msg, question, schema)
    
    return None, f"Failed to generate valid SQL after {max_retries} attempts"

def fix_sql(sql, error_msg, question, schema):
    """Ask LLM to fix SQL based on error."""
    
    prompt = f"""
The following SQL query has an error:

SQL: {sql}
Error: {error_msg}

Original question: {question}
Schema: {schema}

Please fix the SQL query.
SQL:"""

    response = openai.ChatCompletion.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
    )
    
    return response.choices[0].message.content.strip()
```

**Step 6: Execute and Format Results**

```python
def execute_query(sql):
    """Execute SQL and return results."""
    
    with psycopg2.connect(**DB_CONFIG) as conn:
        with conn.cursor() as cur:
            cur.execute(sql)
            rows = cur.fetchall()
            columns = [desc[0] for desc in cur.description]
    
    # Format as list of dicts
    results = []
    for row in rows:
        results.append(dict(zip(columns, row)))
    
    return results

def format_results(results, question):
    """Format results into natural language answer."""
    
    if not results:
        return "No results found."
    
    # Convert to markdown table
    if len(results) <= 10:
        table = format_as_table(results)
        
        # Ask GPT to summarize
        prompt = f"""
Question: {question}

Results:
{table}

Provide a brief natural language answer (2-3 sentences) based on these results.
"""
        
        response = openai.ChatCompletion.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3,
        )
        
        return response.choices[0].message.content
    
    return f"Found {len(results)} results. Here are the top 10:\n{format_as_table(results[:10])}"

def format_as_table(results):
    """Format results as markdown table."""
    if not results:
        return ""
    
    columns = results[0].keys()
    
    # Header
    table = "| " + " | ".join(columns) + " |\n"
    table += "| " + " | ".join(["---"] * len(columns)) + " |\n"
    
    # Rows
    for row in results:
        table += "| " + " | ".join(str(row[col]) for col in columns) + " |\n"
    
    return table
```

**Step 7: Complete Pipeline**

```python
@app.post("/query")
async def nl_to_sql(question: str):
    """Convert natural language to SQL and execute."""
    
    start_time = time.time()
    
    # 1. Retrieve relevant schema
    schema = retrieve_relevant_schema(question)
    
    # 2. Generate SQL with retry
    sql, error = generate_sql_with_retry(question, schema)
    
    if error:
        return {"error": error}
    
    # 3. Execute
    try:
        results = execute_query(sql)
    except Exception as e:
        return {"error": f"Execution failed: {str(e)}"}
    
    # 4. Format answer
    answer = format_results(results, question)
    
    elapsed = time.time() - start_time
    
    return {
        "question": question,
        "sql": sql,
        "answer": answer,
        "results": results[:100],  # Limit to 100 rows
        "latency_ms": int(elapsed * 1000),
    }
```

### Challenges & Solutions

**Challenge 1: Ambiguous questions**

**Problem:** "Show me sales" - which columns? Which time period?

**Solution:**

- Detect ambiguity with LLM
- Ask clarifying questions: "Which time period? (Today / This week / This month)"
- Store user preferences (default to last 30 days)

**Challenge 2: Complex joins**

**Problem:** LLM struggles with multi-table joins.

**Solution:**

- Add more few-shot examples with joins
- Provide explicit foreign key relationships in schema
- Chain-of-thought prompting: "First identify tables, then joins, then filters"

**Challenge 3: SQL accuracy (90%+ target)**

**Baseline:** 70% with zero-shot.

**Improvements:**

- Few-shot prompting: 70% → 82%
- Schema retrieval: 82% → 88%
- Self-correction loop: 88% → 92% ✅

### Evaluation

**Metrics:**

1. **SQL accuracy:**
   - Execution success rate: 92%
   - Semantic correctness (human eval): 88%

2. **Performance:**
   - Latency (p95): 2.8s ✅

3. **Cost:**
   - $0.01 per query (GPT-4o + embeddings)

### Interview Discussion Points

**Mid-level:**

- "Why few-shot instead of fine-tuning?"
  - Few-shot is faster to iterate (no training needed)
  - Fine-tuning requires large SQL dataset
  - Few-shot: $0 vs Fine-tuning: $500-1000

- "How do you prevent SQL injection?"
  - Validate SQL (only SELECT allowed)
  - Use read-only database user
  - Sanitize inputs (though LLM-generated SQL is less risky)

**Senior-level:**

- "Scale to 1000+ tables."
  - Schema retrieval becomes critical (can't fit all tables in context)
  - Hierarchical retrieval (first find relevant schema, then zoom in)
  - Cache schema embeddings

- "Add support for charts."
  - Detect visualization intent ("show me a bar chart of...")
  - Route to Plotly/Recharts based on query type
  - Auto-select chart type (time series → line, categories → bar)

---

## Project 8: Content Moderation System {#project-8-content-moderation}

### Problem Statement

**Build an AI system to moderate user-generated content (text, images) for toxicity, hate speech, NSFW, PII, etc.**

**Requirements:**

- Multi-modal moderation (text + images)
- Real-time (<500ms)
- Multi-level severity (low/medium/high/critical)
- Explain why content was flagged
- Handle 100K requests/day
- False positive rate <5%
- <$0.001 per moderation

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│            User Content (Text / Image)                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                Content Router                                │
│         (Determine moderation path)                          │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         ▼                               ▼
┌──────────────────────┐      ┌──────────────────────┐
│   Text Moderation    │      │   Image Moderation   │
│                      │      │                      │
│  ┌────────────────┐  │      │  ┌────────────────┐  │
│  │ Toxicity       │  │      │  │ NSFW Detection │  │
│  │ (Perspective)  │  │      │  │ (GPT-4V/CLIP)  │  │
│  └────────────────┘  │      │  └────────────────┘  │
│                      │      │                      │
│  ┌────────────────┐  │      │  ┌────────────────┐  │
│  │ Hate Speech    │  │      │  │ Violence       │  │
│  │ (GPT-4o-mini)  │  │      │  │ Detection      │  │
│  └────────────────┘  │      │  └────────────────┘  │
│                      │      │                      │
│  ┌────────────────┐  │      │  ┌────────────────┐  │
│  │ PII Detection  │  │      │  │ Hate Symbols   │  │
│  │ (Regex + LLM)  │  │      │  │                │  │
│  └────────────────┘  │      │  └────────────────┘  │
└──────────┬───────────┘      └──────────┬───────────┘
           │                             │
           └──────────────┬──────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Severity Aggregator (Rule Engine)               │
│  - Combine scores from all checks                            │
│  - Apply business rules (e.g., PII = auto-flag)             │
│  - Determine severity: SAFE / LOW / MEDIUM / HIGH / CRITICAL │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Explanation Generator (GPT-4o-mini)             │
│  "Flagged for: Hate speech (confidence: 0.92)"              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Human Review Queue                          │
│  (For edge cases / high-value content)                       │
└─────────────────────────────────────────────────────────────┘
```

### Tech Stack

```yaml
Text Moderation:
  - Toxicity: Perspective API (Google)
  - Hate speech: gpt-4o-mini (custom prompts)
  - PII: Presidio (Microsoft) + regex

Image Moderation:
  - NSFW: GPT-4V or CLIP-based classifier
  - Violence: Custom fine-tuned model

LLM:
  - Explanation: gpt-4o-mini
  - Edge cases: gpt-4o (multimodal)

Storage:
  - Flagged content: PostgreSQL
  - Audit log: Elasticsearch

Infrastructure:
  - Backend: FastAPI
  - Cache: Redis (for repeated content hashes)
  - Queue: Celery (async processing for non-realtime)
```

### Implementation

**Step 1: Text Moderation - Toxicity**

```python
from googleapiclient import discovery
import json

def check_toxicity(text):
    """Check toxicity using Perspective API."""
    
    client = discovery.build(
        "commentanalyzer",
        "v1alpha1",
        developerKey=PERSPECTIVE_API_KEY,
        discoveryServiceUrl="https://commentanalyzer.googleapis.com/$discovery/rest?version=v1alpha1",
        static_discovery=False,
    )
    
    analyze_request = {
        'comment': {'text': text},
        'requestedAttributes': {
            'TOXICITY': {},
            'SEVERE_TOXICITY': {},
            'IDENTITY_ATTACK': {},
            'INSULT': {},
            'PROFANITY': {},
            'THREAT': {},
        }
    }
    
    response = client.comments().analyze(body=analyze_request).execute()
    
    scores = {}
    for attr, data in response['attributeScores'].items():
        scores[attr.lower()] = data['summaryScore']['value']
    
    # Determine if toxic
    is_toxic = scores['toxicity'] > 0.7
    severity = 'high' if scores['severe_toxicity'] > 0.6 else 'medium' if scores['toxicity'] > 0.5 else 'low'
    
    return {
        'is_toxic': is_toxic,
        'severity': severity,
        'scores': scores,
    }
```

**Step 2: Text Moderation - Hate Speech**

```python
def check_hate_speech(text):
    """Detect hate speech using LLM."""
    
    prompt = f"""
Analyze this text for hate speech targeting protected groups (race, religion, gender, sexual orientation, disability, nationality).

Text: "{text}"

Output JSON:
{{
  "contains_hate_speech": true/false,
  "confidence": 0-1,
  "target_group": "race/religion/gender/etc or null",
  "reason": "brief explanation"
}}
"""
    
    response = openai.ChatCompletion.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"},
        temperature=0,
    )
    
    return json.loads(response.choices[0].message.content)
```

**Step 3: Text Moderation - PII Detection**

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine
import re

analyzer = AnalyzerEngine()
anonymizer = AnonymizerEngine()

def detect_pii(text):
    """Detect PII (emails, phone numbers, SSN, credit cards)."""
    
    # Presidio analysis
    results = analyzer.analyze(
        text=text,
        language='en',
        entities=[
            "PHONE_NUMBER", "EMAIL_ADDRESS", "CREDIT_CARD",
            "US_SSN", "US_PASSPORT", "IP_ADDRESS"
        ],
    )
    
    # Extract found entities
    found_pii = []
    for result in results:
        found_pii.append({
            'type': result.entity_type,
            'text': text[result.start:result.end],
            'confidence': result.score,
        })
    
    # Anonymize
    anonymized_text = anonymizer.anonymize(text=text, analyzer_results=results)
    
    return {
        'contains_pii': len(found_pii) > 0,
        'pii_found': found_pii,
        'anonymized_text': anonymized_text.text,
    }
```

**Step 4: Image Moderation - NSFW**

```python
import base64
from PIL import Image

def check_nsfw_image(image_path):
    """Detect NSFW content using GPT-4V."""
    
    # Encode image
    with open(image_path, "rb") as f:
        image_data = base64.b64encode(f.read()).decode()
    
    response = openai.ChatCompletion.create(
        model="gpt-4o",
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": "Analyze this image for NSFW content (nudity, sexual content, graphic violence). Output JSON: {\"is_nsfw\": true/false, \"confidence\": 0-1, \"category\": \"nudity/violence/safe\", \"reason\": \"...\"}",
                    },
                    {
                        "type": "image_url",
                        "image_url": {"url": f"data:image/jpeg;base64,{image_data}"},
                    },
                ],
            }
        ],
        response_format={"type": "json_object"},
        max_tokens=200,
    )
    
    return json.loads(response.choices[0].message.content)
```

**Step 5: Severity Aggregator**

```python
def aggregate_moderation_results(text_results, image_results=None):
    """Combine all moderation checks into final decision."""
    
    flags = []
    max_severity = 'safe'
    
    # Text checks
    if text_results.get('toxicity', {}).get('is_toxic'):
        flags.append({
            'type': 'toxicity',
            'severity': text_results['toxicity']['severity'],
            'confidence': text_results['toxicity']['scores']['toxicity'],
        })
        max_severity = max(max_severity, text_results['toxicity']['severity'], key=severity_rank)
    
    if text_results.get('hate_speech', {}).get('contains_hate_speech'):
        flags.append({
            'type': 'hate_speech',
            'severity': 'high',
            'confidence': text_results['hate_speech']['confidence'],
            'reason': text_results['hate_speech']['reason'],
        })
        max_severity = 'high'
    
    if text_results.get('pii', {}).get('contains_pii'):
        flags.append({
            'type': 'pii',
            'severity': 'critical',  # PII is always critical
            'entities': text_results['pii']['pii_found'],
        })
        max_severity = 'critical'
    
    # Image checks
    if image_results and image_results.get('nsfw', {}).get('is_nsfw'):
        flags.append({
            'type': 'nsfw',
            'severity': 'high',
            'confidence': image_results['nsfw']['confidence'],
            'category': image_results['nsfw']['category'],
        })
        max_severity = max(max_severity, 'high', key=severity_rank)
    
    return {
        'action': 'block' if max_severity in ['high', 'critical'] else 'review' if max_severity == 'medium' else 'allow',
        'severity': max_severity,
        'flags': flags,
    }

def severity_rank(severity):
    """Rank severity levels."""
    ranks = {'safe': 0, 'low': 1, 'medium': 2, 'high': 3, 'critical': 4}
    return ranks.get(severity, 0)
```

**Step 6: Explanation Generator**

```python
def generate_explanation(moderation_result):
    """Generate human-readable explanation."""
    
    if moderation_result['action'] == 'allow':
        return "Content is safe."
    
    flags = moderation_result['flags']
    
    if len(flags) == 1:
        flag = flags[0]
        if flag['type'] == 'toxicity':
            return f"Content flagged for toxicity (confidence: {flag['confidence']:.0%})."
        elif flag['type'] == 'hate_speech':
            return f"Content contains hate speech: {flag['reason']}"
        elif flag['type'] == 'pii':
            entities = ', '.join([e['type'] for e in flag['entities']])
            return f"Content contains PII: {entities}"
        elif flag['type'] == 'nsfw':
            return f"Image flagged as {flag['category']} (confidence: {flag['confidence']:.0%})."
    
    # Multiple flags
    flag_types = [f['type'] for f in flags]
    return f"Content flagged for: {', '.join(flag_types)}."
```

**Step 7: Complete Moderation Pipeline**

```python
from fastapi import FastAPI, UploadFile, File, Form
import hashlib

app = FastAPI()

@app.post("/moderate")
async def moderate_content(
    text: str = Form(None),
    image: UploadFile = File(None),
):
    """Moderate text and/or image content."""
    
    start_time = time.time()
    
    # Content hash (for caching)
    content_hash = hashlib.sha256((text or '').encode()).hexdigest()
    
    # Check cache
    cached = redis.get(f"moderation:{content_hash}")
    if cached:
        return json.loads(cached)
    
    text_results = {}
    image_results = {}
    
    # Text moderation
    if text:
        text_results = {
            'toxicity': check_toxicity(text),
            'hate_speech': check_hate_speech(text),
            'pii': detect_pii(text),
        }
    
    # Image moderation
    if image:
        temp_path = f"/tmp/{image.filename}"
        with open(temp_path, "wb") as f:
            f.write(await image.read())
        
        image_results = {
            'nsfw': check_nsfw_image(temp_path),
        }
        
        os.remove(temp_path)
    
    # Aggregate
    final_result = aggregate_moderation_results(text_results, image_results)
    
    # Generate explanation
    explanation = generate_explanation(final_result)
    
    response = {
        'action': final_result['action'],
        'severity': final_result['severity'],
        'explanation': explanation,
        'flags': final_result['flags'],
        'latency_ms': int((time.time() - start_time) * 1000),
    }
    
    # Cache result (1 hour)
    redis.setex(f"moderation:{content_hash}", 3600, json.dumps(response))
    
    # Log for audit
    db.execute("""
        INSERT INTO moderation_log (content_hash, action, severity, flags, timestamp)
        VALUES (%s, %s, %s, %s, NOW())
    """, (content_hash, response['action'], response['severity'], json.dumps(response['flags'])))
    
    return response
```

### Challenges & Solutions

**Challenge 1: False positive rate (<5% target)**

**Baseline:** 12% false positives with single model.

**Improvements:**

- Multi-model ensemble: 12% → 8%
- Confidence thresholds: 8% → 6%
- Human-in-the-loop for edge cases: 6% → 4% ✅

**Challenge 2: Latency (<500ms)**

```
Latency breakdown:
- Toxicity (Perspective): 150ms
- Hate speech (LLM): 200ms
- PII detection: 50ms
- Total: 400ms ✅ (Parallel execution)
```

**Optimization:**

- Run checks in parallel (asyncio)
- Cache results for identical content
- Use faster models (gpt-4o-mini instead of gpt-4o)

**Challenge 3: Cost (<$0.001 per moderation)**

```
Cost breakdown:
- Perspective API: Free (quota: 1M/day)
- Hate speech (gpt-4o-mini): $0.0001
- PII detection: Free (local)
- NSFW (GPT-4V): $0.005 (only when image present)
Average: $0.0001 (mostly text) ✅
```

### Evaluation

**Metrics:**

1. **Accuracy:**
   - Precision: 96% (4% false positives) ✅
   - Recall: 92% (8% false negatives)
   - F1 score: 0.94

2. **Performance:**
   - Latency (p95): 480ms ✅
   - Throughput: 100K requests/day (with caching)

3. **Cost:**
   - Text moderation: $0.0001/request
   - Image moderation: $0.005/request
   - Blended (80% text): $0.001/request ✅

4. **Business impact:**
   - Reduced manual moderation by 85%
   - Improved user trust (fewer toxic posts)

### Interview Discussion Points

**Mid-level:**

- "Why use multiple models instead of one?"
  - Each model specializes (Perspective for toxicity, LLM for hate speech)
  - Ensemble reduces false positives
  - Redundancy (if one API fails, others still work)

- "How do you handle edge cases?"
  - Low-confidence flags → Human review queue
  - User appeals (flag for re-review)
  - Continuous learning (retrain on corrections)

**Senior-level:**

- "Scale to 1M requests/day."
  - Horizontal scaling (10× API servers)
  - Rate limiting per user (prevent abuse)
  - CDN caching for static content
  - Cost: 1M × $0.001 = $1K/day

- "How to reduce bias in hate speech detection?"
  - Diverse training data (multi-cultural examples)
  - Test on minority dialects (African American English, etc.)
  - Human review of flagged minority content
  - Regular fairness audits (precision by demographic)

---

## Summary: All 8 Projects Complete

You now have **8 production-ready GenAI projects**:

1. ✅ Enterprise RAG System
2. ✅ AI Customer Support Agent
3. ✅ Code Review Assistant
4. ✅ Document Intelligence System
5. ✅ Autonomous AI Agent
6. ✅ Multimodal Search Engine
7. ✅ Natural Language to SQL
8. ✅ Content Moderation System

Each project includes:
- Complete architecture
- Full implementation code
- Challenges & solutions
- Evaluation metrics
- Interview discussion points

**Next Steps:**
- Implement 1-2 projects for your portfolio
- Adapt architectures to your domain
- Practice explaining system design in interviews