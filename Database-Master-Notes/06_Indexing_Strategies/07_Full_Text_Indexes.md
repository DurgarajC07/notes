# Full-Text Indexes - Text Search and Inverted Indexes

## 1. Concept Explanation

**Full-text indexes** (FTS indexes) enable efficient text search across large documents by indexing words/tokens rather than entire column values. Unlike B-Tree indexes that match exact strings, full-text indexes support:
- **Word searches**: Find documents containing "database"
- **Phrase searches**: Find "relational database" as a phrase
- **Proximity searches**: Find "database" near "optimization"
- **Ranking**: Sort results by relevance score

Think of a full-text index like a book's index:
- **B-Tree index:** Looks up exact page numbers (WHERE title = 'Chapter 5')
- **Full-text index:** Looks up keywords → lists all pages mentioning that keyword (WHERE content CONTAINS 'optimization')

### Structure: Inverted Index

**Inverted indexes** map words → document IDs (opposite of documents → words).

```
Documents:
doc1: "database indexing strategies"
doc2: "database optimization techniques"  
doc3: "advanced indexing methods"

Inverted index:
"advanced"    → [doc3]
"database"    → [doc1, doc2]
"indexing"    → [doc1, doc3]
"methods"     → [doc3]
"optimization" → [doc2]
"strategies"  → [doc1]
"techniques"  → [doc2]

Query: Search "database indexing"
1. Lookup "database" → [doc1, doc2]
2. Lookup "indexing" → [doc1, doc3]
3. Intersect: [doc1, doc2] ∩ [doc1, doc3] = [doc1]
4. Result: doc1 matches both words ✅
```

**Key Properties:**
- **Tokenization**: Text split into words/tokens (stems, stopwords removed)
- **Posting list**: For each token, list of document IDs containing it
- **Ranking**: TF-IDF, BM25, or custom scoring for relevance
- **Storage**: Larger than original text (inverted index + metadata)

### PostgreSQL: GIN vs GiST Indexes

**GIN (Generalized Inverted Index):**
- Inverted index structure optimized for text search
- Fast reads, slower writes (larger index)
- Best for: Static or infrequently updated text

**GiST (Generalized Search Tree):**
- Tree structure with lossy compression
- Slower reads, faster writes (smaller index)
- Best for: Frequently updated text, geometric data

```sql
-- Table: documents
CREATE TABLE documents (
    doc_id SERIAL,
    content TEXT
);

-- GIN index (faster reads):
CREATE INDEX idx_documents_gin ON documents USING GIN(to_tsvector('english', content));

-- GiST index (faster writes):
CREATE INDEX idx_documents_gist ON documents USING GiST(to_tsvector('english', content));

-- Query:
SELECT doc_id, content
FROM documents
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database & optimization');
-- @@ operator: "matches" (full-text search)
-- &: AND, | : OR, !: NOT
```

---

## 2. Why It Matters

### Production Impact: Document Search Performance

**Scenario: Knowledge base with 1M articles**

```sql
-- Table: articles (1M rows, avg 5KB text per article)
CREATE TABLE articles (
    article_id INT,
    title VARCHAR(255),
    content TEXT,  -- avg 5000 characters
    created_at TIMESTAMP
);
```

**Option 1: LIKE Search (No Index)**
```sql
-- Query: Find articles mentioning "database" and "optimization"
SELECT article_id, title
FROM articles
WHERE content LIKE '%database%' 
  AND content LIKE '%optimization%';

-- Plan: Seq Scan on articles
-- - Scan all 1M rows
-- - Regex pattern matching on 5KB text per row → CPU intensive!
-- - No ranking (results unordered by relevance)

-- Performance:
-- - Time: 45 seconds ❌
-- - CPU: 100% (single-core regex matching)
-- - I/O: 5GB read (entire table)
-- - UX: Unusable search experience
```

**Option 2: B-Tree Index (Prefix Match)**
```sql
CREATE INDEX idx_content_btree ON articles(content);

-- Query:
SELECT * FROM articles WHERE content LIKE 'database%';  -- Prefix only!

-- Can use B-Tree for prefix: 'database%' ✅
-- Cannot use B-Tree for: '%database%' (wildcard at start) ❌

-- Limitations:
-- - Only prefix searches supported
-- - No word boundaries (matches "database" and "databases" inconsistently)
-- - No ranking
-- - Huge index size: 5GB (entire text indexed)
```

**Option 3: Full-Text GIN Index**
```sql
-- Create tsvector column (pre-computed search vector):
ALTER TABLE articles ADD COLUMN content_tsv TSVECTOR;
UPDATE articles SET content_tsv = to_tsvector('english', content);

-- GIN index on tsvector:
CREATE INDEX idx_content_fts ON articles USING GIN(content_tsv);

-- Query:
SELECT article_id, title, ts_rank(content_tsv, query) AS rank
FROM articles, to_tsquery('english', 'database & optimization') query
WHERE content_tsv @@ query
ORDER BY rank DESC
LIMIT 20;

-- Plan: Bitmap Index Scan using idx_content_fts
-- - Lookup "database" in GIN → Posting list [doc1, doc5, doc89, ...]
-- - Lookup "optimization" in GIN → Posting list [doc5, doc23, doc89, ...]
-- - Intersect bitmaps → [doc5, doc89] (both words present)
-- - Rank results by relevance (TF-IDF)

-- Performance:
-- - Time: 50ms (900× FASTER!) ✅
-- - CPU: 5% (index lookup, no full-text scan)
-- - I/O: 50MB (index pages only)
-- - Ranking: Results sorted by relevance ✅

-- Index size: 500MB (10% of original data, compressed tokens)
```

### Real Numbers: E-Commerce Product Search

| Approach | Query Time | Index Size | Ranking | Wildcards | Phrase Search |
|----------|------------|------------|---------|-----------|---------------|
| LIKE '%keyword%' | 30s | 0MB | No | Yes | No |
| B-Tree | 10s (prefix only) | 5GB | No | Prefix only | No |
| GIN full-text | 0.05s | 500MB | Yes (TF-IDF) | Yes | Yes |
| Elasticsearch | 0.01s | 1GB | Yes (BM25) | Yes | Yes |

**Winner: Full-text index (GIN or Elasticsearch)**
- 600× faster than LIKE
- Relevance ranking
- Phrase and proximity searches
- Smaller than B-Tree (compression)

---

## 3. Internal Working

### Tokenization and tsvector

**Tokenization Process:**

```sql
-- Input text:
'PostgreSQL Database Indexing: Advanced Strategies for Optimization'

-- Step 1: Lowercase + split into words
['postgresql', 'database', 'indexing', 'advanced', 'strategies', 'for', 'optimization']

-- Step 2: Remove stopwords ('for', 'the', 'and', etc.)
['postgresql', 'database', 'indexing', 'advanced', 'strategies', 'optimization']

-- Step 3: Stemming (reduce to root form)
['postgresql', 'databas', 'index', 'advanc', 'strategi', 'optim']
-- Note: 'database' → 'databas', 'indexing' → 'index', 'optimization' → 'optim'

-- Step 4: Create tsvector with positions
'advanc':4 'databas':2 'index':3 'optim':7 'postgresql':1 'strategi':5

-- Format: 'word':position1,position2,...
-- Positions enable phrase and proximity searches
```

**tsvector in PostgreSQL:**

```sql
SELECT to_tsvector('english', 'The quick brown foxes jumped over the lazy dogs');

-- Result:
'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2

-- Observations:
-- - Stopwords ('the', 'over') removed
-- - Plurals stemmed: 'foxes' → 'fox', 'dogs' → 'dog'
-- - Verb tenses stemmed: 'jumped' → 'jump'
-- - Positions preserved: 'brown' at position 3 (3rd word)
```

### Inverted Index Structure (GIN)

**GIN B-Tree of Posting Lists:**

```
GIN Index Structure:

Root Node:
[Key: 'database', Pointer to posting list]
[Key: 'index', Pointer to posting list]
[Key: 'optimization', Pointer to posting list]

Posting Lists (stored separately):
'database':     [DocID: 1, 5, 89, 234, 890, ...]  (1000 docs)
'index':        [DocID: 1, 3, 5, 12, 89, ...]     (800 docs)
'optimization': [DocID: 5, 23, 89, 100, ...]      (200 docs)

Query: 'database & optimization'
1. Lookup 'database' → [1, 5, 89, 234, 890, ...]
2. Lookup 'optimization' → [5, 23, 89, 100, ...]
3. Intersect: [1,5,89,234,890] ∩ [5,23,89,100] = [5, 89]
4. Fetch documents 5 and 89

Cost:
- 2 B-Tree lookups: O(log N) each = 2 × log(10K) ≈ 26 comparisons
- Intersect posting lists: O(K) where K = shorter list length = 200
- Total: 26 + 200 = 226 operations
- vs seq scan: 1M document scans ❌
```

### Ranking: TF-IDF

**TF-IDF (Term Frequency × Inverse Document Frequency):**

```
TF (Term Frequency): How often does word appear in document?
  - doc1: "database" appears 5 times in 100 words → TF = 5/100 = 0.05

IDF (Inverse Document Frequency): How rare is the word across all documents?
  - "database" appears in 1000 of 10000 docs → IDF = log(10000/1000) = log(10) = 2.3
  - "the" appears in 9999 of 10000 docs → IDF = log(10000/9999) ≈ 0.0001 (stopword, low value)

TF-IDF Score: TF × IDF
  - "database" in doc1: 0.05 × 2.3 = 0.115
  - "the" in doc1: 0.10 × 0.0001 = 0.00001 (almost zero, irrelevant)

Ranking:
Document with highest TF-IDF sum across query terms ranks first.
```

**Example:**

```
Query: "database optimization"

doc1: "This article discusses database indexing and database optimization techniques."
  - "database": TF=2/10=0.2, IDF=2.3 → Score: 0.46
  - "optimization": TF=1/10=0.1, IDF=3.5 → Score: 0.35
  - Total: 0.46 + 0.35 = 0.81

doc2: "Database systems require optimization for performance."
  - "database": TF=1/8=0.125, IDF=2.3 → Score: 0.29
  - "optimization": TF=1/8=0.125, IDF=3.5 → Score: 0.44
  - Total: 0.29 + 0.44 = 0.73

Ranking: doc1 (0.81) > doc2 (0.73)
Reason: doc1 mentions "database" twice (higher TF)
```

**PostgreSQL ts_rank:**

```sql
SELECT 
    doc_id,
    ts_rank(content_tsv, query) AS rank
FROM documents, to_tsquery('english', 'database & optimization') query
WHERE content_tsv @@ query
ORDER BY rank DESC;

-- ts_rank: Built-in TF-IDF implementation
-- Higher rank = more relevant document
```

### GIN vs GiST Trade-Offs

| Aspect | GIN (Generalized Inverted Index) | GiST (Generalized Search Tree) |
|--------|----------------------------------|--------------------------------|
| **Structure** | B-Tree + posting lists (exact) | Tree with lossy compression |
| **Read Speed** | Fast (O(log N) + posting list scan) | Slower (recheck heap tuples) |
| **Write Speed** | Slow (update posting lists) | Fast (lossy, less precise updates) |
| **Index Size** | Large (100-200% of text) | Small (20-50% of text) |
| **Use Case** | Static/infrequent updates | Frequent updates |
| **Accuracy** | Exact matches | Lossy (false positives, recheck needed) |

**Example: INSERT Performance**

```sql
-- Table: 1M articles, avg 5KB text

-- GIN index:
INSERT INTO articles (content) VALUES ('...');
-- Process:
-- 1. Tokenize text → 500 words
-- 2. Update 500 posting lists in GIN index
-- 3. Potentially split B-Tree pages (if posting lists grow)
-- Time: 50ms per INSERT ⚠️

-- GiST index:
INSERT INTO articles (content) VALUES ('...');
-- Process:
-- 1. Hash text → Lossy signature (bitset)
-- 2. Update single tree node (no posting lists)
-- Time: 5ms per INSERT ✅ (10× faster)

-- Trade-off:
-- - GIN: Slower writes, faster reads
-- - GiST: Faster writes, slower reads
```

---

## 4. Best Practices

### Practice 1: Store tsvector in Separate Column for Performance

**❌ Bad: Compute tsvector on Every Query**
```sql
CREATE INDEX idx_content_fts ON articles USING GIN(to_tsvector('english', content));

-- Query:
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');

-- Problems:
-- - to_tsvector() computed for every matching row (CPU intensive)
-- - Index stores function result, but query recomputes (redundant)
-- - Slower query execution (tokenization overhead)
```

**✅ Good: Pre-Compute tsvector Column**
```sql
-- Add tsvector column:
ALTER TABLE articles ADD COLUMN content_tsv TSVECTOR;

-- Populate tsvector:
UPDATE articles SET content_tsv = to_tsvector('english', content);

-- Index on tsvector column:
CREATE INDEX idx_content_tsv ON articles USING GIN(content_tsv);

-- Auto-update trigger (keep tsvector in sync):
CREATE TRIGGER tsvector_update BEFORE INSERT OR UPDATE
ON articles FOR EACH ROW EXECUTE FUNCTION
tsvector_update_trigger(content_tsv, 'pg_catalog.english', content);

-- Query (no function call!):
SELECT * FROM articles
WHERE content_tsv @@ to_tsquery('english', 'database');

-- Benefits:
-- - No tokenization overhead during query ✅
-- - Faster queries: 50ms → 10ms (5× improvement)
-- - Trigger keeps tsvector updated automatically
```

### Practice 2: Use GIN for Read-Heavy Workloads, GiST for Write-Heavy

**Decision Matrix:**

| Workload | Reads:Writes Ratio | Index Type | Reason |
|----------|-------------------|------------|--------|
| Knowledge base | 1000:1 | GIN | Reads dominate, need fast search |
| Live chat | 1:10 | GiST | Writes dominate, frequent message inserts |
| Blog comments | 10:1 | GIN | More reads than writes (comments read often) |
| Product catalog | 100:1 | GIN | Static data, rare updates |

**Example: Chat Application**

```sql
-- Table: messages (10K inserts/sec)
CREATE TABLE messages (
    message_id BIGSERIAL,
    user_id BIGINT,
    content TEXT,
    sent_at TIMESTAMP
);

-- GIN index (bad for high-write):
CREATE INDEX idx_messages_gin ON messages USING GIN(to_tsvector('english', content));
-- INSERT time: 50ms (GIN update overhead)
-- Throughput: 10K inserts/sec × 50ms = 500 CPU-sec/sec → 50,000% CPU ❌

-- GiST index (good for high-write):
CREATE INDEX idx_messages_gist ON messages USING GiST(to_tsvector('english', content));
-- INSERT time: 5ms (GiST faster writes)
-- Throughput: 10K inserts/sec × 5ms = 50 CPU-sec/sec → 5000% CPU ✅

-- Trade-off:
-- - Query performance: GIN 10ms, GiST 30ms (3× slower reads)
-- - Acceptable for chat (search less frequent than messaging)
```

### Practice 3: Index Only Searchable Columns

**Avoid Indexing Large Non-Searchable Text:**

```sql
-- Table: articles
-- Columns:
--   title: 100 chars (frequently searched)
--   summary: 500 chars (occasionally searched)
--   content: 50KB (rarely searched, mostly displayed)

-- ❌ Bad: Index everything
ALTER TABLE articles ADD COLUMN all_text_tsv TSVECTOR;
UPDATE articles SET all_text_tsv = 
    to_tsvector('english', coalesce(title,'') || ' ' || coalesce(summary,'') || ' ' || coalesce(content,''));
CREATE INDEX idx_all_text ON articles USING GIN(all_text_tsv);
-- Index size: 5GB (includes 50KB content per article × 1M articles)
-- Problem: content rarely searched, wasting 90% of index space

-- ✅ Good: Index only searched columns
ALTER TABLE articles ADD COLUMN searchable_tsv TSVECTOR;
UPDATE articles SET searchable_tsv = 
    to_tsvector('english', coalesce(title,'') || ' ' || coalesce(summary,''));
CREATE INDEX idx_searchable ON articles USING GIN(searchable_tsv);
-- Index size: 500MB (only title + summary indexed)

-- Benefits:
-- - 10× smaller index (5GB → 500MB)
-- - Fits in cache (faster queries)
-- - 10× faster INSERTs (less text to tokenize)

-- Full-content search (rare):
-- Use separate column/index or accept slower LIKE search
```

### Practice 4: Tune GIN Fastupdate for Write Performance

**GIN Fastupdate Buffer:**

```sql
-- Default GIN: Every INSERT updates index immediately (slow)

-- Fastupdate enabled (buffer pending entries):
CREATE INDEX idx_content_gin ON articles USING GIN(content_tsv) WITH (fastupdate = on);

-- Process:
-- 1. INSERT: Add to fastupdate pending list (fast, in-memory)
-- 2. Periodic flush: Merge pending list into GIN index (batch operation)

-- Performance:
-- - Single INSERT: 50ms → 5ms (10× faster) ✅
-- - Bulk INSERT: 10K inserts → 1 flush → Amortized cost lower

-- Trade-off:
-- - Queries: Slightly slower (scan pending list + index)
-- - Write throughput: 10× improvement

-- Configuration:
ALTER INDEX idx_content_gin SET (fastupdate = on);
ALTER INDEX idx_content_gin SET (gin_pending_list_limit = 4096);  -- 4MB buffer (default 1MB)

-- When to use:
-- - High-write workload (>1000 INSERTs/sec)
-- - Acceptable for queries to check pending list
-- - Need write throughput over instant query visibility
```

---

## 5. Common Mistakes

### Mistake 1: Using LIKE Instead of Full-Text Search

**Problem:**

```sql
-- Developer uses LIKE for text search:
SELECT * FROM articles
WHERE content LIKE '%database%' 
  AND content LIKE '%optimization%';

-- Performance: 30 seconds (seq scan 1M rows)
-- Problems:
-- - No index support ('%' wildcard at start)
-- - No ranking (results unordered by relevance)
-- - Case-sensitive (unless ILIKE, but still no index)
```

**Solution:**

```sql
-- Create full-text index:
CREATE INDEX idx_content_fts ON articles USING GIN(to_tsvector('english', content));

-- Query:
SELECT *, ts_rank(to_tsvector('english', content), query) AS rank
FROM articles, to_tsquery('english', 'database & optimization') query
WHERE to_tsvector('english', content) @@ query
ORDER BY rank DESC;

-- Performance: 50ms (600× faster!)
-- Benefits: Ranking, stemming, stopword removal ✅
```

### Mistake 2: Not Using tsquery Properly (Injection Risk)

**Problem: User Input Directly in tsquery**

```sql
-- User search input: "database & optimization"
-- Query (vulnerable):
SELECT * FROM articles
WHERE content_tsv @@ to_tsquery('english', $user_input);

-- If user input is: "'; DROP TABLE articles; --"
-- tsquery injection similar to SQL injection!

-- Or user input: "database&optimization"  (no spaces)
-- to_tsquery('database&optimization') → Syntax error (& requires spaces)
```

**Solution: Use plainto_tsquery or websearch_to_tsquery**

```sql
-- Option 1: plainto_tsquery (safest, auto-tokenizes)
SELECT * FROM articles
WHERE content_tsv @@ plainto_tsquery('english', $user_input);

-- Handles: "database optimization" → Converts to: 'database' & 'optimization'
-- Escapes: Special characters automatically handled
-- No injection risk ✅

-- Option 2: websearch_to_tsquery (Google-like syntax)
SELECT * FROM articles
WHERE content_tsv @@ websearch_to_tsquery('english', $user_input);

-- Supports:
-- - "database optimization" → AND search
-- - "database OR indexing" → OR search  
-- - "-performance" → NOT performance
-- - "relational database" (quotes) → Phrase search

-- User-friendly syntax + injection-safe ✅
```

### Mistake 3: Forgetting to Update tsvector After Content Changes

**Problem:**

```sql
-- Initial setup:
ALTER TABLE articles ADD COLUMN content_tsv TSVECTOR;
UPDATE articles SET content_tsv = to_tsvector('english', content);
CREATE INDEX idx_content_tsv ON articles USING GIN(content_tsv);

-- Later: Content updated, but tsvector NOT updated
UPDATE articles SET content = 'Updated content about PostgreSQL indexing'
WHERE article_id = 12345;

-- Problem: content_tsv still has old tokens!
-- Query: Search "PostgreSQL" → Does NOT find article 12345 (stale tsvector) ❌
```

**Solution: Trigger to Auto-Update tsvector**

```sql
-- Create trigger function:
CREATE FUNCTION articles_tsvector_update() RETURNS TRIGGER AS $$
BEGIN
    NEW.content_tsv := to_tsvector('english', coalesce(NEW.content,''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Attach trigger:
CREATE TRIGGER tsvector_update
BEFORE INSERT OR UPDATE OF content
ON articles
FOR EACH ROW
EXECUTE FUNCTION articles_tsvector_update();

-- Or use built-in trigger:
CREATE TRIGGER tsvector_update 
BEFORE INSERT OR UPDATE
ON articles FOR EACH ROW 
EXECUTE FUNCTION tsvector_update_trigger(content_tsv, 'pg_catalog.english', content);

-- Now: UPDATE content → Automatically updates content_tsv ✅
```

### Mistake 4: Not Tuning Language Configuration

**Problem: Wrong Language or Missing Custom Words**

```sql
-- Default: English stopwords and stemming
-- Query: Search "PostgreSQL" in English-configured index

-- Problem 1: Custom technical terms not recognized
SELECT to_tsvector('english', 'PostgreSQL database indexing');
-- Result: 'databas':2 'index':3 'postgresql':1
-- Issue: 'PostgreSQL' kept as-is (not stemmed, case-insensitive)

-- Problem 2: Company-specific terms treated as stopwords
SELECT to_tsvector('english', 'ACME Corp database solutions');
-- Result: 'acm':1 'corp':2 'databas':3 'solut':4
-- Issue: 'ACME' stemmed to 'acm' (may not match user searches for "ACME")
```

**Solution: Custom Language Configuration**

```sql
-- Create custom dictionary:
CREATE TEXT SEARCH DICTIONARY custom_dict (
    TEMPLATE = simple,  -- No stemming for custom words
    STOPWORDS = custom_stopwords  -- Custom stopword list
);

-- Create custom configuration:
CREATE TEXT SEARCH CONFIGURATION custom_english (COPY = pg_catalog.english);
ALTER TEXT SEARCH CONFIGURATION custom_english
    ALTER MAPPING FOR asciiword WITH custom_dict, english_stem;

-- Add custom stopwords file:
-- custom_stopwords.txt:
-- a
-- an
-- the
-- (exclude company-specific terms like "ACME")

-- Use custom configuration:
CREATE INDEX idx_content_custom ON articles USING GIN(to_tsvector('custom_english', content));

-- Query:
SELECT * FROM articles
WHERE to_tsvector('custom_english', content) @@ to_tsquery('custom_english', 'PostgreSQL');
```

---

## 6. Security Considerations

### 1. Full-Text Search Injection

**Risk:**

```sql
-- User input: database'; DELETE FROM articles; --

-- Vulnerable query:
-- Directly embedding user input in SQL:
sql = f"SELECT * FROM articles WHERE to_tsvector('english', content) @@ to_tsquery('english', '{user_input}')"

-- Result: SQL injection (DROP TABLE, data theft, etc.)
```

**Mitigation:**

```sql
-- Use parameterized queries:
-- Application code (Python example):
query = "SELECT * FROM articles WHERE content_tsv @@ plainto_tsquery('english', %s)"
cursor.execute(query, (user_input,))

-- plainto_tsquery escapes special characters automatically ✅

-- Or use prepared statements (PostgreSQL PREPARE):
PREPARE search_articles (TEXT) AS
    SELECT * FROM articles WHERE content_tsv @@ plainto_tsquery('english', $1);

EXECUTE search_articles('user input here');
```

### 2. Information Leakage via Ranking

**Risk:**

```sql
-- Attacker searches "confidential salary bonus"

-- Query returns no results (documents not accessible)
-- But ranking scores leak info:

SELECT doc_id, ts_rank(content_tsv, query) AS rank
FROM documents, to_tsquery('confidential & salary & bonus') query
WHERE content_tsv @@ query;

-- If rank = 0.9 → Document contains all three words (high confidence)
-- If rank = 0.1 → Document contains one word (low confidence)

-- Attacker inference: "Document 12345 discusses confidential salaries" (sensitive info leak)
```

**Mitigation:**

- **Access control**: Check permissions before showing ranks
- **Redaction**: Hide ranks for unauthorized documents
- **Thresholding**: Only show results above rank threshold (hide low-confidence leaks)

```sql
-- Secure query:
SELECT doc_id, ts_rank(content_tsv, query) AS rank
FROM documents, to_tsquery('confidential & salary') query
WHERE content_tsv @@ query
  AND user_has_access(doc_id, current_user_id())  -- Permission check
ORDER BY rank DESC;
```

---

## 7. Performance Optimization

### Optimization 1: Covering GIN Index with INCLUDE (PostgreSQL 13+)

**Standard GIN: Still Requires Heap Fetch**

```sql
CREATE INDEX idx_content_tsv ON articles USING GIN(content_tsv);

SELECT article_id, title FROM articles
WHERE content_tsv @@ to_tsquery('database');

-- Plan:
Bitmap Heap Scan on articles
  Recheck Cond: (content_tsv @@ ...)
  -> Bitmap Index Scan on idx_content_tsv
  -- Index provides matching doc IDs
  -- Heap fetch required to get article_id, title ❌
```

**Covering GIN: Include Columns (PostgreSQL 13+)**

```sql
CREATE INDEX idx_content_tsv_covering 
ON articles USING GIN(content_tsv) INCLUDE (article_id, title);

SELECT article_id, title FROM articles
WHERE content_tsv @@ to_tsquery('database');

-- Plan (PostgreSQL 13+):
Index Only Scan using idx_content_tsv_covering  ✅
  -- All columns in index (content_tsv, article_id, title)
  -- No heap fetch needed!

-- Performance:
-- - Standard: 50ms (1000 heap fetches)
-- - Covering: 10ms (0 heap fetches, 5× faster!)
```

### Optimization 2: Partition Tables + Per-Partition Indexes

**Scenario: 100M articles, full-text search**

```sql
-- Partitioned by date (old articles rarely searched):
CREATE TABLE articles (
    article_id BIGSERIAL,
    content TEXT,
    content_tsv TSVECTOR,
    created_at DATE
) PARTITION BY RANGE (created_at);

-- Partitions:
CREATE TABLE articles_2023 PARTITION OF articles FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
CREATE TABLE articles_2024 PARTITION OF articles FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- GIN index only on recent partition:
CREATE INDEX idx_articles_2024_tsv ON articles_2024 USING GIN(content_tsv);
-- No index on articles_2023 (historical, rare searches)

-- Query recent articles:
SELECT * FROM articles
WHERE created_at > '2024-01-01'
  AND content_tsv @@ to_tsquery('database');

-- Plan:
-- - Partition pruning: Only scan articles_2024 (90M rows excluded)
-- - GIN index: Fast search within 2024 partition (10M rows)
-- - No index overhead on historical data ✅

-- Benefits:
-- - Index size: 5GB → 500MB (10× smaller, only 2024 indexed)
-- - INSERT performance: No index update for 2023 partition
-- - Query speed: Partition pruning + smaller index = faster
```

### Optimization 3: Use Parallel Query for Large Result Sets

**Enable Parallel GIN Scans:**

```sql
-- PostgreSQL configuration:
SET max_parallel_workers_per_gather = 4;
SET parallel_setup_cost = 100;
SET parallel_tuple_cost = 0.01;

-- Query:
SELECT * FROM articles
WHERE content_tsv @@ to_tsquery('database');
-- Expected: 100K matching documents

-- Plan:
Gather (workers=4)  ← Parallel execution ✅
  Workers Planned: 4
  -> Parallel Bitmap Heap Scan on articles
       -> Bitmap Index Scan on idx_content_tsv

-- Performance:
-- - Single-threaded: 2 seconds (100K docs, single core)
-- - Parallel (4 workers): 0.5 seconds (4× faster!)

-- Speedup: Linear scaling with worker count (if I/O not bottleneck)
```

---

## 8. Examples

### Example 1: Blog Search with Phrase Matching

**Scenario: Search blog posts for exact phrase "database optimization"**

```sql
CREATE TABLE blog_posts (
    post_id SERIAL,
    title VARCHAR(255),
    content TEXT,
    content_tsv TSVECTOR
);

-- Index:
CREATE INDEX idx_blog_posts_tsv ON blog_posts USING GIN(content_tsv);

-- Query: Phrase search (words must be adjacent)
SELECT post_id, title, ts_rank(content_tsv, query) AS rank
FROM blog_posts, phraseto_tsquery('english', 'database optimization') query
WHERE content_tsv @@ query
ORDER BY rank DESC;

-- phraseto_tsquery: Matches "database optimization" as phrase (adjacent words)
-- vs plainto_tsquery: Matches "database" AND "optimization" anywhere in text
```

---

### Example 2: E-Commerce Product Search with Boosting

**Scenario: Search products, boost title matches over description**

```sql
CREATE TABLE products (
    product_id SERIAL,
    title VARCHAR(255),
    description TEXT,
    title_tsv TSVECTOR,
    description_tsv TSVECTOR
);

-- Separate indexes:
CREATE INDEX idx_products_title_tsv ON products USING GIN(title_tsv);
CREATE INDEX idx_products_desc_tsv ON products USING GIN(description_tsv);

-- Query with boosting:
SELECT 
    product_id,
    title,
    (ts_rank(title_tsv, query) * 2.0 +  -- 2× weight for title matches
     ts_rank(description_tsv, query)) AS rank
FROM products, to_tsquery('laptop') query
WHERE title_tsv @@ query OR description_tsv @@ query
ORDER BY rank DESC
LIMIT 20;

-- Boosting: Title matches ranked higher (2× multiplier)
-- Example:
--   Product A: Title match (rank 0.5), Description match (rank 0.1) → Total: 0.5*2 + 0.1 = 1.1
--   Product B: Description match only (rank 0.3) → Total: 0 + 0.3 = 0.3
--   Product A ranks higher ✅
```

---

## 9. Real-World Use Cases

### Use Case 1: Stack Overflow - Question Search

**Context:**
- 50M questions with title + body text
- Full-text search on both fields, ranking by relevance

**Implementation:**

```sql
CREATE TABLE questions (
    question_id BIGSERIAL,
    title VARCHAR(255),
    body TEXT,
    search_vector TSVECTOR GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
        setweight(to_tsvector('english', coalesce(body,'')), 'B')
    ) STORED
);

CREATE INDEX idx_questions_search ON questions USING GIN(search_vector);

-- Query:
SELECT question_id, title, ts_rank(search_vector, query) AS rank
FROM questions, websearch_to_tsquery('english', 'postgresql indexing') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 25;

-- setweight: 'A' = title (higher weight), 'B' = body (lower weight)
-- Title matches rank higher than body matches ✅
```

### Use Case 2: Airbnb - Listing Description Search

**Context:**
- Search listing descriptions for amenities ("pool", "wifi", etc.)
- Multilingual support

**Implementation:**

```sql
CREATE TABLE listings (
    listing_id BIGINT,
    title VARCHAR(255),
    description TEXT,
    language VARCHAR(10),  -- 'en', 'es', 'fr', etc.
    search_vector TSVECTOR
);

-- Trigger: Set tsvector based on language
CREATE FUNCTION update_listing_search_vector() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := to_tsvector(
        NEW.language::regconfig,  -- Use language-specific config
        coalesce(NEW.title,'') || ' ' || coalesce(NEW.description,'')
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER listing_search_vector_update
BEFORE INSERT OR UPDATE OF title, description, language
ON listings FOR EACH ROW
EXECUTE FUNCTION update_listing_search_vector();

CREATE INDEX idx_listings_search ON listings USING GIN(search_vector);

-- Query:
SELECT listing_id, title
FROM listings
WHERE language = 'en'
  AND search_vector @@ plainto_tsquery('english', 'pool wifi parking');

-- Benefits:
-- - Language-specific stemming ('pools' → 'pool' in English, 'piscinas' → 'piscina' in Spanish)
-- - Global search across all languages with appropriate tokenization ✅
```

---

## 10. Interview Questions

### Q1: How does an inverted index differ from a B-Tree index? When would you choose each?

**Senior Answer:**

```
**B-Tree Index:**

Structure: Sorted tree of column values → Row IDs
  - Example: Index on email column
    ['alice@example.com', RID1]
    ['bob@example.com', RID2]
    ['charlie@example.com', RID3]

Query: WHERE email = 'alice@example.com'
  → Navigate B-Tree → Find RID1 → Fast O(log N)

Query: WHERE email LIKE '%example.com'
  → Cannot use B-Tree efficiently (wildcard at start) → Seq scan ❌

Best for:
  - Exact matches (=, IN)
  - Range queries (<, >, BETWEEN)
  - Prefix matches (LIKE 'alice%')
  - Sorted results (ORDER BY)

---

**Inverted Index (Full-Text):**

Structure: Token → List of document IDs containing token
  - Example: Index on article content
    'database' → [doc1, doc5, doc89]
    'indexing' → [doc1, doc3, doc89]
    'optimization' → [doc5, doc23, doc89]

Query: Search "database indexing"
  → Lookup 'database' → [doc1, doc5, doc89]
  → Lookup 'indexing' → [doc1, doc3, doc89]
  → Intersect → [doc1, doc89] ✅

Best for:
  - Text search (contains word)
  - Multiple word AND/OR/NOT queries
  - Phrase searches ("exact phrase")
  - Relevance ranking (TF-IDF)

---

**Decision Matrix:**

| Use Case | Index Type | Reason |
|----------|------------|--------|
| WHERE email = 'alice@example.com' | B-Tree | Exact match on full value |
| WHERE status IN ('pending', 'active') | B-Tree | Discrete values, exact matches |
| WHERE content LIKE '%database%' | Full-Text (GIN) | Substring search (wildcard both sides) |
| Search "database AND optimization" | Full-Text (GIN) | Multi-word text search + ranking |
| WHERE price BETWEEN 100 AND 500 | B-Tree | Range query |
| Autocomplete: title LIKE 'data%' | B-Tree | Prefix match |

---

**Real-World Example:**

At my previous company, we had a products table with 'name' and 'description' columns. Initial implementation used B-Tree indexes on both.

Problem: User search "laptop charger" → Query: WHERE name LIKE '%laptop%' OR description LIKE '%charger%'
  → Performance: 30 seconds (seq scan, B-Tree unusable with leading wildcard)

Solution: Migrated to GIN full-text index on (name || description)
  → Performance: 50ms (600× faster!)
  → Added ranking: Users saw most relevant products first (MacBook Pro charger before generic laptop accessories)

Lesson: B-Tree for structured exact/range queries. Full-text inverted index for natural language search across large text fields.
```

---

### Q2: Explain GIN vs GiST indexes in PostgreSQL. When would you choose each?

**Staff Answer:**

```
**GIN (Generalized Inverted Index):**

**Structure:**
- B-Tree of tokens → Posting lists (exact document IDs)
- Example:
  Token: 'database' → Posting list: [doc1, doc5, doc89, doc234, ...]
  Token: 'optimization' → Posting list: [doc5, doc23, doc89, ...]

**Characteristics:**
- Exact posting lists (no false positives)
- Large index size (100-200% of original text, due to inverted structure)
- Slow writes (update all affected posting lists for each INSERT)
- Fast reads (direct lookup → posting list → result)

**Query Process:**
1. Lookup token 'database' → Retrieve posting list [doc1, doc5, ...]
2. Lookup token 'optimization' → Retrieve posting list [doc5, doc23, ...]
3. Intersect lists → [doc5, doc89]
4. Fetch documents (no recheck needed, exact matches) ✅

**Write Performance:**
INSERT with 500 words → Update 500 posting lists → 50ms
  - Each token: Append doc ID to posting list
  - Potential page splits in B-Tree
  - Expensive for high-write workloads ❌

**Best For:**
- Read-heavy (>100:1 read:write ratio)
- Static or infrequently updated text (knowledge base, historical data)
- Need fast, accurate search (no false positives)

---

**GiST (Generalized Search Tree):**

**Structure:**
- Tree with lossy signatures (bitsets/hashes, not exact tokens)
- Example:
  Node: Signature for docs [doc1, doc5, doc89] → Bitset: 0x3FA2...
  (Lossy: Multiple tokens hash to same bits)

**Characteristics:**
- Lossy compression (false positives possible → recheck heap tuples)
- Small index size (20-50% of text, compressed signatures)
- Fast writes (update single tree node, not multiple posting lists)
- Slower reads (traverse tree + recheck heap tuples for false positives)

**Query Process:**
1. Traverse GiST tree (check signatures match query tokens)
2. Reach leaf nodes → Get candidate doc IDs
3. **Recheck heap tuples** (signature match != exact match) ⚠️
4. Filter false positives → Return exact matches

**Write Performance:**
INSERT with 500 words → Hash to signature → Update 1 tree node → 5ms
  - No per-token posting list updates
  - 10× faster writes than GIN ✅

**Best For:**
- Write-heavy (>1:10 read:write ratio)
- Frequently updated text (chat messages, live feeds)
- Acceptable slower reads (recheck overhead tolerable)
- Geometric data (PostGIS uses GiST for spatial indexes)

---

**Comparison Table:**

| Aspect | GIN | GiST |
|--------|-----|------|
| **Read Speed** | Fast (50ms) | Slower (150ms, recheck overhead) |
| **Write Speed** | Slow (50ms) | Fast (5ms) |
| **Index Size** | Large (500MB for 100MB text) | Small (50MB for 100MB text) |
| **Accuracy** | Exact (no false positives) | Lossy (false positives, recheck) |
| **Use Case** | Read-heavy, static text | Write-heavy, dynamic text |
| **Cache Efficiency** | Lower (large index) | Higher (small index fits in cache) |

---

**Decision Framework:**

**Choose GIN When:**
- Reads >> Writes (>100:1 ratio)
  - Example: Knowledge base, documentation, blog archives
- Need fastest possible search (<10ms)
- Storage not constrained (can afford 2× text size for index)
- Data relatively static (daily/weekly updates acceptable)

**Choose GiST When:**
- Writes >> Reads (>1:10 ratio)
  - Example: Chat messages (10K inserts/sec, search rare)
- Write throughput critical (need <5ms INSERTs)
- Storage constrained (small index preferred)
- Acceptable slower search (100-200ms OK)

**Choose Both (Hybrid):**
- Recent data: GiST (high write volume)
- Historical data: GIN (read-heavy, no writes)
- Partition table by date, different index types per partition ✅

---

**Real-World Example:**

Company: Messaging platform (Slack-like)
Table: messages (10M messages/day, 10K inserts/sec)

Initial design: GIN index on message content
  Problem: INSERT time 50ms → 10K qps needs 500 CPU-sec/sec → 50,000% CPU ❌
  Result: Database overwhelmed, replication lag 10 minutes

Solution: Switched to GiST index
  - INSERT time: 50ms → 5ms (10× faster)
  - CPU usage: 50,000% → 5,000% (manageable)
  - Search performance: 50ms → 150ms (3× slower reads, but search queries <1% of traffic)

Trade-off: Slightly slower search (acceptable) for 10× write throughput (critical).

Optimization: After 30 days, move messages to historical partition with GIN index
  - Recent messages (<30 days): GiST (high write, occasional search)
  - Old messages (>30 days): GIN (no writes, fast search when needed)

Result: Best of both worlds—fast writes for recent data, fast reads for historical search.

Lesson: **Choose index type based on workload. Read-heavy? GIN. Write-heavy? GiST. Time-series? Hybrid approach with partitioning.**
```

---

### Q3: How would you implement multi-language full-text search?

**Principal Answer:**

```
**Challenge:** Different languages have different:
- Stopwords ('the' in English, 'le/la' in French, 'der/die/das' in German)
- Stemming rules ('running' → 'run' in English, 'courant' → 'cour' in French)
- Character sets (Latin, Cyrillic, CJK, Arabic)
- Tokenization (spaces in English, katakana in Japanese, no spaces in Chinese)

**Strategy 1: Language Column + Language-Specific Indexes**

```sql
CREATE TABLE documents (
    doc_id BIGSERIAL,
    content TEXT,
    language VARCHAR(10),  -- 'en', 'es', 'fr', 'de', 'ja', 'zh', etc.
    content_tsv_en TSVECTOR,  -- English tsvector
    content_tsv_es TSVECTOR,  -- Spanish tsvector
    content_tsv_fr TSVECTOR,  -- French tsvector
    ...
);

-- Trigger: Populate correct tsvector based on language
CREATE FUNCTION update_document_tsvector() RETURNS TRIGGER AS $$
BEGIN
    CASE NEW.language
        WHEN 'en' THEN
            NEW.content_tsv_en := to_tsvector('english', NEW.content);
        WHEN 'es' THEN
            NEW.content_tsv_es := to_tsvector('spanish', NEW.content);
        WHEN 'fr' THEN
            NEW.content_tsv_fr := to_tsvector('french', NEW.content);
        ELSE
            NEW.content_tsv_en := to_tsvector('simple', NEW.content);  -- Fallback: no stemming
    END CASE;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER doc_tsvector_update BEFORE INSERT OR UPDATE OF content, language
ON documents FOR EACH ROW EXECUTE FUNCTION update_document_tsvector();

-- Indexes per language:
CREATE INDEX idx_docs_tsv_en ON documents USING GIN(content_tsv_en) WHERE language = 'en';
CREATE INDEX idx_docs_tsv_es ON documents USING GIN(content_tsv_es) WHERE language = 'es';
CREATE INDEX idx_docs_tsv_fr ON documents USING GIN(content_tsv_fr) WHERE language = 'fr';

-- Query:
SELECT doc_id, content
FROM documents
WHERE language = 'en'
  AND content_tsv_en @@ plainto_tsquery('english', 'database optimization');
```

**Pros:**
- Accurate language-specific stemming and stopwords
- Fast (separate indexes, no cross-language confusion)
- Simple query logic (filter by language first)

**Cons:**
- Storage overhead (multiple tsvector columns if many languages)
- Complex triggers (handle many languages)
- Cannot search across languages easily

---

**Strategy 2: Single tsvector + Dynamic Configuration**

```sql
CREATE TABLE documents (
    doc_id BIGSERIAL,
    content TEXT,
    language VARCHAR(10),
    content_tsv TSVECTOR  -- Single tsvector column
);

-- Trigger: Use language to select configuration dynamically
CREATE FUNCTION update_document_tsvector_dynamic() RETURNS TRIGGER AS $$
BEGIN
    NEW.content_tsv := to_tsvector(NEW.language::regconfig, NEW.content);
    RETURN NEW;
EXCEPTION
    WHEN OTHERS THEN
        -- Fallback: If language config doesn't exist (e.g., 'zh'), use 'simple'
        NEW.content_tsv := to_tsvector('simple', NEW.content);
        RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER doc_tsvector_update_dynamic BEFORE INSERT OR UPDATE OF content, language
ON documents FOR EACH ROW EXECUTE FUNCTION update_document_tsvector_dynamic();

CREATE INDEX idx_docs_tsv ON documents USING GIN(content_tsv);

-- Query:
SELECT doc_id, content
FROM documents
WHERE language = 'en'
  AND content_tsv @@ plainto_tsquery('english', 'database optimization');
```

**Pros:**
- Single tsvector column (less storage)
- Simpler schema (one index)

**Cons:**
- Query must know document language (requires language filter)
- Mixed-language search difficult (different token forms in same index)

---

**Strategy 3: External Search Engine (Elasticsearch)**

For complex multi-language requirements, use dedicated search engine:

```
Elasticsearch configuration:

Index: documents
Mappings:
{
  "properties": {
    "content": {
      "type": "text",
      "fields": {
        "en": {
          "type": "text",
          "analyzer": "english"
        },
        "es": {
          "type": "text",
          "analyzer": "spanish"
        },
        "fr": {
          "type": "text",
          "analyzer": "french"
        }
      }
    },
    "language": {
      "type": "keyword"
    }
  }
}

Query:
GET /documents/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "language": "en" } },
        { "match": { "content.en": "database optimization" } }
      ]
    }
  }
}
```

**Pros:**
- Best-in-class full-text search (BM25 ranking, phrase search, fuzzy matching)
- Language analyzers for 50+ languages (built-in)
- Horizontal scalability (shard across nodes)
- Advanced features: Highlighting, suggestions, aggregations

**Cons:**
- Operational complexity (separate infrastructure)
- Eventual consistency (sync lag between PostgreSQL → Elasticsearch)
- Additional cost (Elasticsearch cluster)

---

**Strategy 4: Hybrid (PostgreSQL + Language Detection)**

Automatically detect language, then apply correct configuration:

```sql
-- Install language detection extension (external, e.g., CLD2, langdetect via plpython)
CREATE EXTENSION plpython3u;

CREATE FUNCTION detect_language(text TEXT) RETURNS TEXT AS $$
    import langdetect
    try:
        return langdetect.detect(text)
    except:
        return 'en'  # Default to English
$$ LANGUAGE plpython3u;

-- Trigger: Auto-detect language, set tsvector
CREATE FUNCTION update_document_tsvector_auto() RETURNS TRIGGER AS $$
BEGIN
    NEW.language := detect_language(NEW.content);  -- Auto-detect
    NEW.content_tsv := to_tsvector((NEW.language || '_text')::regconfig, NEW.content);
    RETURN NEW;
EXCEPTION
    WHEN OTHERS THEN
        NEW.content_tsv := to_tsvector('simple', NEW.content);
        RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER doc_tsvector_auto_detect BEFORE INSERT OR UPDATE OF content
ON documents FOR EACH ROW EXECUTE FUNCTION update_document_tsvector_auto();
```

**Pros:**
- No manual language tagging required (automatic)
- Adapts to multilingual inputs

**Cons:**
- Language detection accuracy ~90% (may misdetect short texts)
- Performance overhead (detection on every INSERT)

---

**Recommendation Decision Tree:**

1. **Few languages (2-3), language known upfront:**
   → Strategy 1 (Language-specific tsvector columns)
   → Simple, accurate, fast

2. **Many languages (10+), language known upfront:**
   → Strategy 2 (Dynamic configuration) or Strategy 3 (Elasticsearch)
   → Avoids schema explosion

3. **Language unknown (user-generated content):**
   → Strategy 4 (Auto-detect) or Strategy 3 (Elasticsearch with multi-field analyzers)

4. **Massive scale (100M+ documents, complex queries):**
   → Strategy 3 (Elasticsearch)
   → PostgreSQL full-text suitable for <50M documents, beyond that Elasticsearch scales better

---

**Real-World Example:**

Company: International knowledge base (Wikipedia-like)
Documents: 10M articles in 25 languages

Initial implementation: Single tsvector with 'simple' config (no stemming)
  Problem: Poor search quality ('running' didn't match 'run', language-agnostic)

Solution: Strategy 1 with top 5 languages (English, Spanish, French, German, Japanese)
  - 5 tsvector columns (en, es, fr, de, ja)
  - Other 20 languages: Use 'simple' config (acceptable degradation for <5% traffic)
  - Query: Filter by language first, then search appropriate tsvector column

Result:
  - Search quality: 90% precision/recall (up from 60%)
  - Performance: 50ms P99 (same as before, partitioned by language)
  - Storage: +30% for 5 tsvector columns (acceptable for quality gain)

Lesson: **For multi-language search, invest in language-specific configurations for dominant languages (Pareto principle: 80% traffic = 20% languages). Accept simpler search for long-tail languages.**
```

---

## 11. Summary

### Full-Text Index Key Principles

**1. Inverted Index Structure**
- Maps tokens → document IDs (opposite of document → content)
- Enables fast multi-word searches (AND/OR/NOT operations)
- Supports ranking (TF-IDF, BM25) for relevance sorting

**2. Tokenization + Stemming**
- Text split into tokens (words), positions preserved
- Stopwords removed ('the', 'a', 'and')
- Words stemmed to root form ('running' → 'run', 'databases' → 'databas')
- Language-specific rules (English, Spanish, French, etc.)

**3. GIN vs GiST Trade-Off**
- GIN: Fast reads, slow writes, large index (read-heavy workloads)
- GiST: Faster writes, slower reads, small index (write-heavy workloads)
- Choose based on read:write ratio

**4. Ranking and Relevance**
- TF-IDF: Term frequency × Inverse document frequency
- Higher score = more relevant document
- Essential for user-facing search (order matters!)

### When to Use Full-Text Indexes

**✅ Create Full-Text Index:**
- Text search with wildcards ('%keyword%')
- Multi-word AND/OR queries
- Need relevance ranking
- Large text fields (articles, descriptions, comments)
- User-facing search features
- Better alternative to LIKE for substring matching

**❌ Use B-Tree Instead:**
- Exact value lookups (email = 'user@example.com')
- Prefix searches only (name LIKE 'John%')
- Small fixed vocabulary (status IN ('pending', 'active'))
- Range queries (price BETWEEN 100 AND 500)

### Implementation Best Practices

| Practice | Benefit |
|----------|---------|
| Pre-compute tsvector column | 5-10× faster queries (no on-the-fly tokenization) |
| Auto-update trigger | Keeps tsvector in sync with content changes |
| Use plainto_tsquery() | Injection-safe user input handling |
| Add covering INCLUDE columns (PG13+) | Index-only scans (2-5× speedup) |
| Tune GIN fastupdate | 10× faster INSERTs (batch updates) |
| Separate indexes for title vs body | Boost title matches (better ranking) |

### Performance Characteristics

| Metric | LIKE '%keyword%' | B-Tree (prefix) | GIN Full-Text |
|--------|------------------|-----------------|---------------|
| Query time (1M docs) | 30s (seq scan) | 10s (partial) | 0.05s (index) |
| Index size | 0MB | 500MB | 100MB (compressed) |
| Wildcards | ✅ All | ✅ Prefix only | ✅ All |
| Ranking | ❌ No | ❌ No | ✅ Yes (TF-IDF) |
| Multi-word AND | ❌ Multiple LIKEs | ❌ Not supported | ✅ Fast (bitwise ops) |
| Phrase search | ❌ No | ❌ No | ✅ Yes |

### Quick Wins

**1. Convert LIKE to full-text (2 hours, 100-1000× speedup)**
```sql
-- Before:
WHERE content LIKE '%database%'  -- 30s (seq scan)

-- After:
ALTER TABLE articles ADD COLUMN content_tsv TSVECTOR;
UPDATE articles SET content_tsv = to_tsvector('english', content);
CREATE INDEX idx_content_tsv ON articles USING GIN(content_tsv);

WHERE content_tsv @@ plainto_tsquery('english', 'database')  -- 0.05s (index)
```

**2. Add tsvector trigger (30 min, prevents stale indexes)**
```sql
CREATE TRIGGER tsvector_update BEFORE INSERT OR UPDATE
ON articles FOR EACH ROW EXECUTE FUNCTION
tsvector_update_trigger(content_tsv, 'pg_catalog.english', content);
```

**3. Enable GIN fastupdate for high-write workloads (5 min, 10× faster INSERTs)**
```sql
ALTER INDEX idx_content_tsv SET (fastupdate = on, gin_pending_list_limit = 4096);
```

### Database-Specific Implementations

**PostgreSQL:**
```sql
-- GIN (recommended):
CREATE INDEX idx_name ON table USING GIN(to_tsvector('english', content));

-- GiST (write-heavy):
CREATE INDEX idx_name ON table USING GiST(to_tsvector('english', content));

-- Query:
WHERE to_tsvector('english', content) @@ to_tsquery('database & optimization');
```

**MySQL:**
```sql
-- Full-text index (MyISAM, InnoDB 5.6+):
CREATE FULLTEXT INDEX idx_name ON table(content);

-- Query:
WHERE MATCH(content) AGAINST('database optimization' IN BOOLEAN MODE);
```

**SQL Server:**
```sql
-- Full-text index:
CREATE FULLTEXT INDEX ON table(content) KEY INDEX pk_table;

-- Query:
WHERE CONTAINS(content, 'database AND optimization');
```

**Elasticsearch (External):**
```
PUT /articles/_mapping
{
  "properties": {
    "content": {
      "type": "text",
      "analyzer": "english"
    }
  }
}

GET /articles/_search
{
  "query": {
    "match": {
      "content": "database optimization"
    }
  }
}
```

### Key Takeaway

**"Full-text indexes (inverted indexes) transform slow LIKE searches into sub-second ranked results. Use GIN for read-heavy workloads, GiST for write-heavy. Pre-compute tsvector columns for maximum performance. For enterprise-scale search (100M+ documents), consider Elasticsearch."**

Example decision: Search 1M blog articles for "database optimization"
- LIKE '%database%': 30 seconds (seq scan)
- GIN full-text: 0.05 seconds (600× faster!)
- GIN + covering index (title included): 0.01 seconds (3000× faster!)
- Result: Fast search, relevance ranking, scalable to 10M+ documents ✅

---

**Next:** `08_Index_Maintenance.md` - Fragmentation, REINDEX strategies, and monitoring

