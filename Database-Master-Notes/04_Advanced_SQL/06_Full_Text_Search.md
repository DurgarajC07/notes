# 🔍 Full-Text Search - Advanced Text Search Operations

> Full-Text Search (FTS) enables linguistic search across large text data with ranking, stemming, and fuzzy matching. Master FTS for search engines, content management, and text analytics.

---

## 📖 1. Concept Explanation

### What is Full-Text Search?

**Full-Text Search (FTS)** is a specialized indexing and search technique for natural language queries on text data.

**FTS vs LIKE:**
```sql
-- ❌ LIKE: Simple pattern matching
SELECT * FROM articles
WHERE content LIKE '%database%';
-- - Case-sensitive (PostgreSQL) or insensitive (MySQL)
-- - No ranking
-- - No stemming (database ≠ databases)
-- - Slow on large text (sequential scan)
-- - No linguistic support

-- ✅ Full-Text Search
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');
-- - Linguistic search (database = databases = DB)
-- - Ranked results (relevance scores)
-- - Fast (GIN/GiST indexes)
-- - Stemming, stop words, synonyms
-- - Fuzzy matching
```

**Database Support:**
```
FULL-TEXT SEARCH
│
├── PostgreSQL: to_tsvector, to_tsquery, GIN index ✅
├── MySQL: FULLTEXT index, MATCH...AGAINST ✅
├── SQL Server: FULLTEXT index, CONTAINS, FREETEXT ✅
├── Oracle: Oracle Text (formerly ConText) ✅
└── SQLite: FTS5 extension ✅
```

---

## 🧠 2. Why It Matters in Real Systems

### LIKE vs Full-Text Search Performance

```sql
-- Data: 1 million articles, 1KB text each

-- ❌ LIKE: Sequential scan
SELECT * FROM articles
WHERE content LIKE '%artificial intelligence%';
-- Time: 45 seconds ❌
-- Execution: Full table scan (1M rows × 1KB = 1GB scanned)
-- CPU: High (regex matching per row)

-- ✅ Full-Text Search with index
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'artificial & intelligence');
-- Time: 0.3 seconds ✅ (150x faster)
-- Execution: Index scan (GIN index lookup)
-- CPU: Low
-- Bonus: Ranked results, stemming (AI, artificially, intelligent match)
```

---

## ⚙️ 3. Internal Working

### PostgreSQL: tsvector and tsquery

```sql
-- to_tsvector: Convert text to searchable format
SELECT to_tsvector('english', 'The quick brown foxes jumped');
-- Output: 'brown':3 'fox':4 'jump':5 'quick':2
-- 
-- Process:
-- 1. Tokenize: ["The", "quick", "brown", "foxes", "jumped"]
-- 2. Lowercase: ["the", "quick", "brown", "foxes", "jumped"]
-- 3. Remove stop words: ["quick", "brown", "foxes", "jumped"]
-- 4. Stem: ["quick", "brown", "fox", "jump"]
-- 5. Store with positions: 'brown':3 (position 3 in original)

-- to_tsquery: Convert query to search format
SELECT to_tsquery('english', 'foxes & jumping');
-- Output: 'fox' & 'jump'
-- Both stemmed to match tsvector

-- Match operator: @@
SELECT to_tsvector('english', 'foxes jumped') @@ to_tsquery('english', 'fox & jump');
-- Returns: true (both terms match)
```

### GIN Index for Full-Text

```sql
-- GIN (Generalized Inverted Index)
CREATE INDEX idx_content_fts ON articles USING GIN (to_tsvector('english', content));

-- Index structure (simplified):
-- token  | article_ids
-- -------+-------------
-- 'fox'  | [1, 15, 234, 567, ...]
-- 'jump' | [1, 45, 123, 567, ...]
-- 'quick'| [1, 99, 234, ...]
--
-- Query: 'fox & jump'
-- 1. Lookup 'fox' → [1, 15, 234, 567, ...]
-- 2. Lookup 'jump' → [1, 45, 123, 567, ...]
-- 3. Intersect → [1, 567] (both present)
-- 4. Fetch rows 1 and 567
--
-- Time complexity: O(log N) lookup + O(K) intersection (K = result count)
```

---

## ✅ 4. Best Practices

### Use Generated tsvector Column

```sql
-- ❌ Compute tsvector in every query (slow)
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');
-- Computes tsvector for every row on each query ❌

-- ✅ Store tsvector in dedicated column
ALTER TABLE articles ADD COLUMN content_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;

CREATE INDEX idx_content_tsv ON articles USING GIN (content_tsv);

-- Now fast queries:
SELECT * FROM articles
WHERE content_tsv @@ to_tsquery('english', 'database');
-- Uses precomputed tsvector ✅
```

### Use Ranking for Relevance

```sql
-- ✅ Sort by relevance score
SELECT 
    title,
    ts_rank(content_tsv, query) as rank
FROM articles, to_tsquery('english', 'database & performance') query
WHERE content_tsv @@ query
ORDER BY rank DESC
LIMIT 10;

-- ts_rank: Frequency-based scoring
-- ts_rank_cd: Cover density (considers proximity)
```

### Combine with Regular Filters

```sql
-- ✅ Full-text + structured filters
SELECT title, published_at, ts_rank(content_tsv, query) as rank
FROM articles, to_tsquery('english', 'machine & learning') query
WHERE content_tsv @@ query
  AND published_at >= NOW() - INTERVAL '1 year'
  AND category = 'AI'
ORDER BY rank DESC
LIMIT 20;

-- Uses partial index if available:
CREATE INDEX idx_recent_ai ON articles(published_at, category)
WHERE category = 'AI';
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Not Using Index

```sql
-- ❌ Function in WHERE without functional index
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');
-- Sequential scan ❌

-- ✅ Create GIN index on expression
CREATE INDEX idx_content_fts ON articles USING GIN (to_tsvector('english', content));
-- Or use generated column (better):
ALTER TABLE articles ADD COLUMN content_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;
CREATE INDEX idx_content_tsv ON articles USING GIN (content_tsv);
```

### Mistake 2: Wrong Query Syntax

```sql
-- ❌ Using plain text (won't work)
WHERE content_tsv @@ 'database performance';
-- ERROR: operator @@ expects tsquery

-- ✅ Use to_tsquery or plainto_tsquery
WHERE content_tsv @@ to_tsquery('english', 'database & performance');

-- Or plainto_tsquery for natural language
WHERE content_tsv @@ plainto_tsquery('english', 'database performance');
-- Converts to: 'database' & 'performance'
```

### Mistake 3: Ignoring Language Configuration

```sql
-- ❌ Wrong language or missing language
to_tsvector(content)  -- Uses default (often 'simple', no stemming)

-- ✅ Specify language for stemming
to_tsvector('english', content)  -- English stemming
to_tsvector('spanish', content)  -- Spanish stemming

-- Check current default:
SHOW default_text_search_config;  -- Likely 'pg_catalog.english'
```

---

## 🔐 6. Security Considerations

### Prevent Query Injection

```sql
-- ❌ Concatenating user input
DECLARE @userQuery VARCHAR(100) = 'database'' OR 1=1--';
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', @userQuery);
-- Error or unexpected behavior

-- ✅ Use plainto_tsquery (escapes automatically)
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ plainto_tsquery('english', @userQuery);
-- Treats entire input as search terms ✅

-- ✅ Or sanitize with websearch_to_tsquery (PostgreSQL 11+)
SELECT * FROM articles
WHERE content_tsv @@ websearch_to_tsquery('english', @userQuery);
-- Supports quotes, AND/OR, minus (like Google search)
```

---

## 🚀 7. Performance Optimization

### GIN vs GiST Indexes

```sql
-- GIN (Generalized Inverted Index)
CREATE INDEX idx_gin ON articles USING GIN (content_tsv);
-- Pros: Faster queries ✅ (3-5x)
-- Cons: Slower inserts/updates (2-3x), larger size (~50% of text data)
-- Use for: Read-heavy workloads ✅

-- GiST (Generalized Search Tree)
CREATE INDEX idx_gist ON articles USING GiST (content_tsv);
-- Pros: Faster inserts/updates ✅, smaller size
-- Cons: Slower queries
-- Use for: Write-heavy workloads

-- Benchmark (1M rows, read-heavy):
-- GIN query: 0.3s ✅
-- GiST query: 1.2s
-- GIN recommended for most use cases ✅
```

### Partial Indexes for Filtered Queries

```sql
-- ✅ Index only recent articles
CREATE INDEX idx_recent_content_tsv ON articles USING GIN (content_tsv)
WHERE published_at >= '2023-01-01';

-- Smaller index, faster queries on recent data ✅
SELECT * FROM articles
WHERE content_tsv @@ to_tsquery('english', 'database')
  AND published_at >= '2023-01-01';
```

---

## 🧪 8. Examples

### PostgreSQL: Basic Full-Text Search

```sql
-- Create table with tsvector column
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    content_tsv tsvector GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''))
    ) STORED
);

CREATE INDEX idx_content_tsv ON articles USING GIN (content_tsv);

-- Insert data
INSERT INTO articles (title, content) VALUES
('Introduction to Databases', 'Databases store and manage data efficiently...'),
('Advanced Database Performance', 'Optimizing database queries for speed...');

-- Simple search
SELECT title FROM articles
WHERE content_tsv @@ to_tsquery('english', 'database');

-- Output:
-- Introduction to Databases
-- Advanced Database Performance
```

### PostgreSQL: Query Operators

```sql
-- AND operator: &
SELECT title FROM articles
WHERE content_tsv @@ to_tsquery('english', 'database & performance');
-- Matches: "database" AND "performance"

-- OR operator: |
SELECT title FROM articles
WHERE content_tsv @@ to_tsquery('english', 'mysql | postgresql');
-- Matches: "mysql" OR "postgresql"

-- NOT operator: !
SELECT title FROM articles
WHERE content_tsv @@ to_tsquery('english', 'database & !nosql');
-- Matches: "database" but NOT "nosql"

-- Phrase search: <->
SELECT title FROM articles
WHERE content_tsv @@ to_tsquery('english', 'data <-> science');
-- Matches: "data science" (adjacent words in order)

-- Proximity: <N>
SELECT title FROM articles
WHERE content_tsv @@ to_tsquery('english', 'data <2> science');
-- Matches: "data science" or "data big science" (within 2 positions)
```

### PostgreSQL: Ranking Results

```sql
-- ts_rank: Frequency-based ranking
SELECT 
    title,
    ts_rank(content_tsv, query) as rank
FROM articles, to_tsquery('english', 'database & optimization') query
WHERE content_tsv @@ query
ORDER BY rank DESC;

-- ts_rank with weights (title > content)
SELECT 
    title,
    ts_rank(
        setweight(to_tsvector('english', title), 'A') ||  -- Weight A (highest)
        setweight(to_tsvector('english', content), 'B'),  -- Weight B
        query
    ) as rank
FROM articles, to_tsquery('english', 'database') query
WHERE to_tsvector('english', title || ' ' || content) @@ query
ORDER BY rank DESC;

-- ts_rank_cd: Cover density (considers word proximity)
SELECT 
    title,
    ts_rank_cd(content_tsv, query) as rank
FROM articles, to_tsquery('english', 'machine & learning') query
WHERE content_tsv @@ query
ORDER BY rank DESC;
```

### PostgreSQL: Highlighting Matches

```sql
-- ts_headline: Highlight matching terms
SELECT 
    title,
    ts_headline('english', content, query, 'MaxFragments=3, MaxWords=50, MinWords=25') as snippet
FROM articles, to_tsquery('english', 'database & performance') query
WHERE content_tsv @@ query;

-- Output:
-- title: "Advanced Database Performance"
-- snippet: "Optimizing <b>database</b> queries for <b>performance</b> requires understanding..."
```

### MySQL: Full-Text Search

```sql
-- Create FULLTEXT index
CREATE TABLE articles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title TEXT,
    content TEXT,
    FULLTEXT KEY idx_content (title, content)
);

-- Natural language search
SELECT title, MATCH(title, content) AGAINST ('database performance') as score
FROM articles
WHERE MATCH(title, content) AGAINST ('database performance' IN NATURAL LANGUAGE MODE)
ORDER BY score DESC;

-- Boolean mode (AND, OR, NOT)
SELECT title FROM articles
WHERE MATCH(title, content) AGAINST ('+database +performance -nosql' IN BOOLEAN MODE);
-- + required, - excluded

-- Query expansion (finds related terms)
SELECT title FROM articles
WHERE MATCH(title, content) AGAINST ('database' WITH QUERY EXPANSION);
```

### SQL Server: Full-Text Search

```sql
-- Create FULLTEXT catalog and index
CREATE FULLTEXT CATALOG ft_catalog AS DEFAULT;

CREATE FULLTEXT INDEX ON articles(title, content)
KEY INDEX PK_articles;

-- CONTAINS: Boolean search
SELECT title FROM articles
WHERE CONTAINS(content, 'database AND performance');

-- CONTAINS with proximity
SELECT title FROM articles
WHERE CONTAINS(content, 'NEAR((data, science), 5)');
-- Within 5 words

-- FREETEXT: Natural language
SELECT title, KEY_TBL.RANK
FROM articles
INNER JOIN CONTAINSTABLE(articles, content, 'database performance optimization') AS KEY_TBL
    ON articles.id = KEY_TBL.[KEY]
ORDER BY KEY_TBL.RANK DESC;

-- FREETEXTTABLE: Natural language with ranking
SELECT title, RANK
FROM articles
INNER JOIN FREETEXTTABLE(articles, content, 'machine learning artificial intelligence') AS FT
    ON articles.id = FT.[KEY]
WHERE RANK > 10
ORDER BY RANK DESC;
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: E-commerce Product Search

```sql
-- Product catalog with full-text search
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    description TEXT,
    category VARCHAR(50),
    price NUMERIC(10,2),
    search_vector tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(name, '')), 'A') ||  -- Name (highest weight)
        setweight(to_tsvector('english', coalesce(description, '')), 'B') ||  -- Description
        setweight(to_tsvector('english', coalesce(category, '')), 'C')  -- Category
    ) STORED
);

CREATE INDEX idx_product_search ON products USING GIN (search_vector);

-- Search query with filters
SELECT 
    name,
    price,
    ts_rank(search_vector, query) as relevance
FROM products, websearch_to_tsquery('english', 'wireless bluetooth headphones') query
WHERE search_vector @@ query
  AND price BETWEEN 50 AND 200
  AND category = 'Electronics'
ORDER BY relevance DESC
LIMIT 20;
```

### Case Study 2: Documentation/Knowledge Base Search

```sql
-- Knowledge base articles
CREATE TABLE kb_articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    category VARCHAR(50),
    views INT DEFAULT 0,
    helpful_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    search_tsv tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(content, '')), 'B')
    ) STORED
);

CREATE INDEX idx_kb_search ON kb_articles USING GIN (search_tsv);

-- Search with popularity boost
SELECT 
    id,
    title,
    ts_headline('english', content, query, 'MaxWords=50') as snippet,
    (ts_rank(search_tsv, query) * (1 + LOG(views + 1) * 0.1)) as score
FROM kb_articles, websearch_to_tsquery('english', 'how to reset password') query
WHERE search_tsv @@ query
ORDER BY score DESC
LIMIT 10;
```

### Case Study 3: Job Board Search

```sql
-- Job listings
CREATE TABLE jobs (
    id SERIAL PRIMARY KEY,
    title TEXT,
    description TEXT,
    skills TEXT[],  -- Array of skills
    location VARCHAR(100),
    salary_min INT,
    salary_max INT,
    posted_at TIMESTAMP DEFAULT NOW(),
    search_vector tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(description, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(array_to_string(skills, ' '), '')), 'A')
    ) STORED
);

CREATE INDEX idx_job_search ON jobs USING GIN (search_vector);

-- Search jobs
SELECT 
    title,
    location,
    salary_max,
    ts_rank(search_vector, query) as relevance
FROM jobs, websearch_to_tsquery('english', 'senior python developer machine learning') query
WHERE search_vector @@ query
  AND posted_at >= NOW() - INTERVAL '30 days'
  AND location ILIKE '%San Francisco%'
ORDER BY relevance DESC
LIMIT 20;
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between LIKE and Full-Text Search?

**Answer:**

| Feature | LIKE | Full-Text Search |
|---------|------|------------------|
| **Performance** | Sequential scan ❌ | Indexed lookups ✅ |
| **Stemming** | No (database ≠ databases) | Yes ✅ (database = databases) |
| **Ranking** | No | Yes ✅ (relevance scores) |
| **Linguistic** | No | Yes ✅ (synonyms, stop words) |
| **Fuzzy Match** | No | Yes ✅ (typo tolerance) |
| **Case Sensitive** | Depends on collation | Normalized ✅ |
| **Speed (1M rows)** | 45s ❌ | 0.3s ✅ |
| **Use Case** | Exact pattern match | Natural language search ✅ |

**Example:**
```sql
-- LIKE: Only exact substring
SELECT * FROM articles WHERE content LIKE '%database%';
-- Matches: "database"
-- Doesn't match: "databases", "DB", "data base"

-- FTS: Linguistic matching
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');
-- Matches: "database", "databases", "DB" (through stemming/synonyms) ✅
```

**Recommendation:** Use FTS for search features, LIKE for exact pattern matching (emails, IDs).

---

### Q2: Explain how full-text indexing works in PostgreSQL.

**Answer:**

**1. tsvector creation:**
```sql
to_tsvector('english', 'The quick brown foxes jumped')
-- Process:
-- a. Tokenize: ["The", "quick", "brown", "foxes", "jumped"]
-- b. Normalize: ["the", "quick", "brown", "foxes", "jumped"]
-- c. Remove stop words: ["quick", "brown", "foxes", "jumped"]
-- d. Stem: ["quick", "brown", "fox", "jump"]
-- e. Store with positions: 'brown':3 'fox':4 'jump':5 'quick':2
```

**2. GIN index structure:**
```
Inverted index (token → document IDs):
'fox'   → [1, 15, 234, 567, ...]
'jump'  → [1, 45, 123, 567, ...]
'quick' → [1, 99, 234, ...]
```

**3. Query processing:**
```sql
to_tsquery('english', 'fox & jump')
-- Stems to: 'fox' & 'jump'
-- Index lookup:
--   'fox'  → [1, 15, 234, 567, ...]
--   'jump' → [1, 45, 123, 567, ...]
-- Intersect: [1, 567]
-- Return: documents 1 and 567
```

**Time complexity:** O(log N) for index lookup + O(K) for intersection (K = result size)

---

### Q3: How do you rank search results by relevance?

**Answer:**

**PostgreSQL: ts_rank and ts_rank_cd**

```sql
-- ts_rank: Frequency-based
SELECT 
    title,
    ts_rank(content_tsv, query) as rank
FROM articles, to_tsquery('english', 'database') query
WHERE content_tsv @@ query
ORDER BY rank DESC;

-- Higher rank = more occurrences of search terms

// ts_rank_cd: Cover density (considers proximity)
SELECT 
    title,
    ts_rank_cd(content_tsv, query) as rank
FROM articles, to_tsquery('english', 'machine & learning') query
WHERE content_tsv @@ query
ORDER BY rank DESC;

-- Higher rank = terms closer together
```

**Weighted ranking (prioritize title):**
```sql
SELECT 
    title,
    ts_rank(
        setweight(to_tsvector('english', title), 'A') ||  -- Weight 1.0
        setweight(to_tsvector('english', content), 'B'),  -- Weight 0.4
        query
    ) as rank
FROM articles, to_tsquery('english', 'database') query
WHERE to_tsvector('english', title || ' ' || content) @@ query
ORDER BY rank DESC;
```

**Custom scoring (combine FTS + popularity):**
```sql
SELECT 
    title,
    (ts_rank(content_tsv, query) * (1 + LOG(views + 1) * 0.1)) as custom_score
FROM articles, to_tsquery('english', 'database') query
WHERE content_tsv @@ query
ORDER BY custom_score DESC;
```

---

### Q4: What's the difference between GIN and GiST indexes for full-text search?

**Answer:**

| Feature | GIN | GiST |
|---------|-----|------|
| **Query Speed** | Faster ✅ (3-5x) | Slower |
| **Insert/Update** | Slower (2-3x) | Faster ✅ |
| **Index Size** | Larger (~50% of text) | Smaller ✅ |
| **Exact Match** | Excellent ✅ | Good |
| **Partial Match** | Good | Better ✅ |
| **Use Case** | Read-heavy ✅ | Write-heavy |

**Benchmark (1M documents):**
```
Operation        | GIN     | GiST
-----------------+---------+--------
Build index      | 120s    | 45s ✅
Query (simple)   | 0.3s ✅ | 1.2s
Query (complex)  | 0.8s ✅ | 3.5s
Insert 10K rows  | 15s     | 6s ✅
Update 10K rows  | 18s     | 7s ✅
Index size       | 250MB   | 150MB ✅
```

**Recommendation:**
- ✅ **Use GIN** for most applications (queries matter more than writes)
- ✅ **Use GiST** for high-write workloads (real-time logs, streaming data)

**Syntax:**
```sql
-- GIN (default choice)
CREATE INDEX idx_gin ON articles USING GIN (content_tsv);

-- GiST
CREATE INDEX idx_gist ON articles USING GiST (content_tsv);
```

---

### Q5: How do you implement autocomplete/typeahead search?

**Answer:**

**PostgreSQL: Prefix matching with pg_trgm**

```sql
-- Install trigram extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create GIN trigram index
CREATE INDEX idx_title_trgm ON products USING GIN (title gin_trgm_ops);

-- Autocomplete query
SELECT title
FROM products
WHERE title ILIKE 'lapt%'  -- Prefix matches "laptop", "laptops"
ORDER BY similarity(title, 'lapt') DESC
LIMIT 10;

-- Or use full-text prefix search
SELECT title
FROM products
WHERE to_tsvector('english', title) @@ to_tsquery('english', 'lapt:*');
-- 'lapt:*' matches any word starting with "lapt"
```

**Optimized autocomplete table:**
```sql
-- Pre-compute search suggestions
CREATE TABLE search_suggestions (
    prefix VARCHAR(50) PRIMARY KEY,
    suggestions TEXT[],  -- Top suggestions
    search_count INT DEFAULT 0  -- Popularity
);

CREATE INDEX idx_prefix_trgm ON search_suggestions USING GIN (prefix gin_trgm_ops);

-- Populate from search logs
INSERT INTO search_suggestions (prefix, suggestions, search_count)
SELECT 
    LOWER(LEFT(search_term, 5)) as prefix,
    ARRAY_AGG(search_term ORDER BY count DESC) FILTER (WHERE rn <= 10) as suggestions,
    SUM(count) as search_count
FROM (
    SELECT 
        search_term,
        count,
        ROW_NUMBER() OVER (PARTITION BY LOWER(LEFT(search_term, 5)) ORDER BY count DESC) as rn
    FROM search_logs
    GROUP BY search_term, count
) sub
GROUP BY prefix;

-- Fast autocomplete lookup
SELECT suggestions[1:10]
FROM search_suggestions
WHERE prefix = LOWER(LEFT('laptop', 5));
-- Returns: ["laptop", "laptops", "laptop bags", ...]
```

---

## 🧩 11. Design Patterns

### Pattern 1: Multi-Field Weighted Search

```sql
-- Weight title > description > tags
ALTER TABLE products ADD COLUMN search_vector tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||  -- Weight: 1.0
    setweight(to_tsvector('english', coalesce(description, '')), 'B') ||  -- Weight: 0.4
    setweight(to_tsvector('english', coalesce(array_to_string(tags, ' '), '')), 'C')  -- Weight: 0.2
) STORED;

CREATE INDEX idx_search_vector ON products USING GIN (search_vector);
```

### Pattern 2: Language-Specific Search

```sql
-- Multi-language content
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    language VARCHAR(10),  -- 'en', 'es', 'fr'
    search_tsv_en tsvector,
    search_tsv_es tsvector,
    search_tsv_fr tsvector
);

-- Triggers to maintain language-specific tsvectors
CREATE OR REPLACE FUNCTION articles_search_update() RETURNS trigger AS $$
BEGIN
    CASE NEW.language
        WHEN 'en' THEN NEW.search_tsv_en := to_tsvector('english', NEW.title || ' ' || NEW.content);
        WHEN 'es' THEN NEW.search_tsv_es := to_tsvector('spanish', NEW.title || ' ' || NEW.content);
        WHEN 'fr' THEN NEW.search_tsv_fr := to_tsvector('french', NEW.title || ' ' || NEW.content);
    END CASE;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tsvector_update BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION articles_search_update();

-- Indexes per language
CREATE INDEX idx_search_en ON articles USING GIN (search_tsv_en);
CREATE INDEX idx_search_es ON articles USING GIN (search_tsv_es);
CREATE INDEX idx_search_fr ON articles USING GIN (search_tsv_fr);
```

### Pattern 3: Search Analytics

```sql
-- Track search queries for optimization
CREATE TABLE search_logs (
    id SERIAL PRIMARY KEY,
    query TEXT,
    results_count INT,
    clicked_result_id INT,
    user_id INT,
    searched_at TIMESTAMP DEFAULT NOW()
);

-- Analyze zero-result queries
SELECT query, COUNT(*) as frequency
FROM search_logs
WHERE results_count = 0
GROUP BY query
ORDER BY frequency DESC
LIMIT 20;

-- Popular searches
SELECT query, COUNT(*) as frequency
FROM search_logs
WHERE results_count > 0
GROUP BY query
ORDER BY frequency DESC
LIMIT 50;
```

---

## 📚 Summary

### Key Takeaways

1. **FTS vs LIKE**: FTS faster (150x), linguistic, ranked; LIKE for exact patterns
2. **Components**: to_tsvector (text → searchable), to_tsquery (query format), @@ (match operator)
3. **Indexing**: GIN (fast queries, read-heavy ✅), GiST (fast writes, write-heavy)
4. **Generated columns**: Store tsvector for performance (avoid recomputing)
5. **Ranking**: ts_rank (frequency), ts_rank_cd (proximity), custom weights
6. **Operators**: & (AND), | (OR), ! (NOT), <-> (phrase), :* (prefix)
7. **Performance**: 1M rows: LIKE 45s vs FTS 0.3s (150x faster)
8. **Highlighting**: ts_headline for snippet generation
9. **Security**: Use plainto_tsquery or websearch_to_tsquery (prevent injection)
10. **Real-world**: Product search, documentation, job boards, content management

**PostgreSQL Quick Reference:**

```sql
-- Create searchable column
ALTER TABLE articles ADD COLUMN search_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || content)) STORED;

-- Create GIN index
CREATE INDEX idx_search ON articles USING GIN (search_tsv);

-- Search query
SELECT title, ts_rank(search_tsv, query) as rank
FROM articles, to_tsquery('english', 'database & performance') query
WHERE search_tsv @@ query
ORDER BY rank DESC
LIMIT 10;

-- Highlight matches
SELECT ts_headline('english', content, query)
FROM articles, to_tsquery('english', 'database') query
WHERE search_tsv @@ query;
```

---

**Next:** [07_Spatial_Queries.md](07_Spatial_Queries.md) - Geographic/spatial operations  
**Related:** [../06_Indexing_Performance/03_Index_Types.md](../06_Indexing_Performance/03_Index_Types.md) - Index strategies

---

*Last Updated: February 20, 2026*
