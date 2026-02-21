# Hash Indexes - O(1) Lookups for Exact Matches

## 1. Concept Explanation

**Hash indexes** store key-value pairs in a hash table structure, providing theoretically O(1) constant-time lookups for exact equality matches. Unlike B-Tree indexes which maintain sorted order, hash indexes distribute keys pseudo-randomly across buckets using a hash function.

Think of a hash index like a library card catalog with numbered drawers:
- **Hash function:** Converts book title (key) to drawer number → hash("Moby Dick") = drawer #47
- **Buckets:** Each drawer (bucket) contains cards for books that hash to that number
- **Collisions:** Multiple books might hash to same drawer (handle via chaining)

### Hash Index vs B-Tree

```
Hash Index:
✅ Exact match: O(1) constant time (best case)
❌ Range queries: Not supported (keys not sorted)
❌ ORDER BY: Not supported (no ordering)
❌ Prefix match (LIKE 'A%'): Not supported

B-Tree Index:
✅ Exact match: O(log N)
✅ Range queries: O(log N + K)
✅ ORDER BY: Free (already sorted)
✅ Prefix match: Supported

Reality: Hash O(1) vs B-Tree O(log N) difference is marginal for typical database sizes
         (log 1B = 30, so B-Tree does ~30 comparisons vs hash's 1-2)
```

### Structure Visualization

```
Hash Function: h(key) = key mod 10

Keys: [5, 15, 25, 35, 45, 7, 17, 27, 9]

Hash Table Structure:

Bucket 0: NULL
Bucket 1: NULL
Bucket 2: NULL
Bucket 3: NULL
Bucket 4: NULL
Bucket 5: → [5] → [15] → [25] → [35] → [45]  ← Collision chain (linked list)
Bucket 6: NULL
Bucket 7: → [7] → [17] → [27]  ← Collision chain
Bucket 8: NULL
Bucket 9: → [9]

Lookup h(25):
1. Compute hash: 25 mod 10 = 5
2. Go to bucket 5
3. Scan chain: 5? No. 15? No. 25? YES! ← Found

Cost: 1 hash computation + 3 comparisons (chain scan)
```

**Key Properties:**
- **Deterministic:** Same key always hashes to same bucket
- **Uniform distribution:** Good hash function spreads keys evenly
- **Collisions inevitable:** Pigeonhole principle (infinite keys, finite buckets)
- **Not cryptographic:** Speed matters more than unpredictability

---

## 2. Why It Matters

### Theoretical vs Practical Performance

**Theory: Hash is Faster**
```
Hash index: O(1)
B-Tree index: O(log N)

For 1 billion rows: log₂(1B) ≈ 30
Hash should be 30× faster!
```

**Reality: Difference is Marginal**

```sql
-- Lookup by UUID (1B rows)

-- B-Tree index:
SELECT * FROM sessions WHERE session_id = 'f47ac10b-58cc-4372-a567-0e02b2c3d479';

-- Plan:
Index Scan using idx_sessions_btree
  Index Cond: (session_id = 'f47ac10b...')
  
-- I/O: 4 page reads (tree depth 4 for 1B rows)
-- Time: 0.4ms (4 pages × 0.1ms per page)

-- Hash index:
CREATE INDEX idx_sessions_hash ON sessions USING HASH (session_id);

SELECT * FROM sessions WHERE session_id = 'f47ac10b-58cc-4372-a567-0e02b2c3d479';

-- Plan:
Index Scan using idx_sessions_hash
  Index Cond: (session_id = 'f47ac10b...')
  
-- I/O: 2 page reads (hash bucket + overflow page if collision)
-- Time: 0.2ms (2 pages × 0.1ms per page)

-- Speedup: 2× (not 30×!)
-- Why? I/O dominates, not comparisons
-- B-Tree: 4 I/Os with 30 comparisons
-- Hash: 2 I/Os with 5 comparisons (collision chain)
-- Savings: 2 I/Os (200 microseconds) → Marginal
```

### When Hash Index Actually Helps

**Use Case: In-Memory Session Store**

```sql
-- Session lookups (10M active sessions, all in RAM)
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY,
    user_id BIGINT,
    data JSONB,
    expires_at TIMESTAMPTZ
);

-- Without index (using hash table in RAM):
-- Lookup: 100ns (direct memory access)

-- With B-Tree (in RAM):
-- Lookup: 500ns (4 pointer dereferences, 30 comparisons)

-- With Hash (in RAM):
-- Lookup: 150ns (hash computation + bucket lookup)

-- Speedup: 3× (B-Tree vs Hash in memory)

-- Real scenario: 100K requests/sec
-- Savings: 350ns × 100K = 35ms CPU per second
-- Impact: 3.5% CPU reduction (nice to have, not critical)
```

---

## 3. Internal Working

### Hash Function Design

**Requirements for Database Hash Function:**
1. **Fast to compute** (no cryptographic overhead)
2. **Uniform distribution** (avoid bucket clustering)
3. **Deterministic** (same input → same output)
4. **Low collision rate** (for common data distributions)

**Common Hash Functions in Databases:**

```c
// PostgreSQL: Simple modulo hash for integers
uint32_t hash_int32(int32_t key, uint32_t num_buckets) {
    return (uint32_t)key % num_buckets;
}

// PostgreSQL: For strings (MurmurHash variant)
uint32_t hash_string(const char* str, size_t len, uint32_t num_buckets) {
    uint32_t hash = 0;
    for (size_t i = 0; i < len; i++) {
        hash = hash * 31 + str[i];  // Prime multiplier (31)
    }
    return hash % num_buckets;
}

// MySQL: FNV-1a hash (fast and good distribution)
uint64_t fnv1a_hash(const void* data, size_t len) {
    uint64_t hash = 0xcbf29ce484222325ULL;  // FNV offset basis
    const uint8_t* bytes = (const uint8_t*)data;
    for (size_t i = 0; i < len; i++) {
        hash ^= bytes[i];
        hash *= 0x100000001b3ULL;  // FNV prime
    }
    return hash;
}
```

**Hash Distribution Quality Example:**

```
Good hash (uniform):
Bucket 0: ██████ (6 keys)
Bucket 1: █████ (5 keys)
Bucket 2: ██████ (6 keys)
Bucket 3: █████ (5 keys)
Bucket 4: ██████ (6 keys)
→ Std deviation: 0.5 (low variance, good)
→ Max chain length: 6 (linear scan of 6 items worst case)

Bad hash (clustered):
Bucket 0: ██ (2 keys)
Bucket 1: ██████████████████ (18 keys) ← Hot spot!
Bucket 2: █ (1 key)
Bucket 3: ████████ (8 keys)
Bucket 4: █ (1 key)
→ Std deviation: 6.8 (high variance, bad)
→ Max chain length: 18 (degrades to O(N) scan)
```

### Collision Resolution: Chaining

**Linked List Chaining (Most Common):**

```
Bucket structure:
+----------------+
| Bucket 5       |
+----------------+
| First ptr: --------→ [Entry: key=5, rid=100] → [Entry: key=15, rid=200] → [Entry: key=25, rid=300] → NULL
+----------------+

Insert(35):
1. Hash: 35 mod 10 = 5
2. Go to bucket 5
3. Allocate new entry: [Entry: key=35, rid=400]
4. Prepend to chain: bucket[5] → new entry → old chain
   (Prepend is O(1), append would be O(k) where k = chain length)

After insert:
+----------------+
| Bucket 5       |
+----------------+
| First ptr: --------→ [Entry: key=35, rid=400] → [Entry: key=5, rid=100] → [Entry: key=15, rid=200] → [Entry: key=25, rid=300] → NULL
+----------------+

Lookup(15):
1. Hash: 15 mod 10 = 5
2. Go to bucket 5
3. Scan chain:
     key=35? No.
     key=5? No.
     key=15? YES! Return rid=200

Cost: O(k) where k = chain length
Best case: k=1 (no collisions) → O(1)
Worst case: k=N (all keys in one bucket) → O(N)
Average case: k = N/B (N keys, B buckets) → O(1) if B ≈ N
```

### Dynamic Resizing (Rehashing)

**Problem: Performance Degrades as Load Factor Increases**

```
Load factor α = N / B
- N = number of keys
- B = number of buckets

Average chain length = α
Lookup cost = O(α)

Example:
- Initial: 100 keys, 100 buckets → α = 1.0 → avg scan 1 item
- After growth: 1000 keys, 100 buckets → α = 10 → avg scan 10 items (10× slower!)
```

**Solution: Rehashing When α > Threshold**

```
Rehash trigger: α > 0.75 (typical threshold)

Rehash algorithm:
1. Allocate new bucket array (2× size)
2. For each key in old table:
     - Recompute hash with new bucket count
     - Insert into new table
3. Free old bucket array

Example:
Before rehash:
- 80 keys, 100 buckets → α = 0.8 (threshold exceeded)
- Bucket 5: [5, 15, 25, 35, 45] (chain length 5)

After rehash (200 buckets):
- 80 keys, 200 buckets → α = 0.4
- Recompute hashes:
    5 mod 200 = 5   → Bucket 5: [5]
    15 mod 200 = 15  → Bucket 15: [15]
    25 mod 200 = 25  → Bucket 25: [25]
    35 mod 200 = 35  → Bucket 35: [35]
    45 mod 200 = 45  → Bucket 45: [45]
- Chain lengths reduced from 5 → 1 (5× faster lookups!)

Cost: O(N) to rehash all keys
Amortized cost: O(1) per insert (rehash infrequent)
```

**PostgreSQL Implementation: Linear Hashing**

```
Instead of doubling all at once, gradually split buckets:

Initial: 4 buckets
Split pointer: 0

Insert triggers split:
1. Split bucket 0 into bucket 0 and bucket 4
2. Move half the keys from bucket 0 to bucket 4
3. Advance split pointer: 0 → 1

Next insert:
1. Split bucket 1 into bucket 1 and bucket 5
2. Advance split pointer: 1 → 2

Eventually all buckets split → Double capacity
Then restart with new split pointer

Benefit: Spread rehash cost across many inserts (no large pause)
```

---

## 4. Best Practices

### Practice 1: Use Hash Indexes Only for Exact Equality

**❌ Bad: Hash Index for Range Query**
```sql
-- Hash index won't help here!
CREATE INDEX idx_orders_total_hash ON orders USING HASH (total_amount);

-- Query:
SELECT * FROM orders WHERE total_amount > 100.00;

-- Plan:
Seq Scan on orders  ← Hash index NOT used!
  Filter: (total_amount > 100.00)

-- Why?
-- Hash keys not sorted, can't do range scan
-- Optimizer ignores hash index for range queries
```

**✅ Good: Hash Index for Exact Match**
```sql
CREATE INDEX idx_sessions_id_hash ON sessions USING HASH (session_id);

-- Query:
SELECT * FROM sessions WHERE session_id = 'abc123';

-- Plan:
Index Scan using idx_sessions_id_hash
  Index Cond: (session_id = 'abc123')

-- Hash index used efficiently!
```

### Practice 2: Prefer B-Tree Unless Proven Need for Hash

**Default Choice: B-Tree**
```sql
-- Start with B-Tree (versatile)
CREATE INDEX idx_users_email ON users(email);

-- Supports all query types:
WHERE email = 'alice@example.com'  ✅ Exact match
WHERE email LIKE 'alice%'          ✅ Prefix match
WHERE email > 'a@'                 ✅ Range
ORDER BY email                      ✅ Sorting
```

**Switch to Hash Only If:**
1. **Proven bottleneck:** Profiling shows B-Tree lookup is slowest operation
2. **Only exact match queries:** Never need ranges, sorting, or prefix matching
3. **High-cardinality keys:** UUID, hash values, long strings (where hash saves space)
4. **Benchmarked improvement:** Measured >20% speedup with hash vs B-Tree

**Example Decision Process:**
```sql
-- Scenario: Session lookup by UUID (10M sessions)

-- Step 1: Profile with B-Tree
CREATE INDEX idx_sessions_btree ON sessions(session_id);
EXPLAIN ANALYZE SELECT * FROM sessions WHERE session_id = 'f47ac10b...';
-- Result: 0.5ms per lookup

-- Step 2: Check if B-Tree features needed
-- Q: Ever query ranges? (WHERE session_id > 'a...')  → NO
-- Q: Ever sort by session_id? → NO
-- Q: Ever prefix match? → NO
-- Conclusion: Hash candidate ✅

-- Step 3: Benchmark hash
CREATE INDEX idx_sessions_hash ON sessions USING HASH (session_id);
EXPLAIN ANALYZE SELECT * FROM sessions WHERE session_id = 'f47ac10b...';
-- Result: 0.3ms per lookup (40% faster)

-- Step 4: Measure impact at scale
-- Load test: 100K requests/sec
-- Savings: 0.2ms × 100K = 20 seconds of CPU per second
-- Impact: 2000% CPU reduction → Worth it! ✅

-- Decision: Use hash index
DROP INDEX idx_sessions_btree;
```

### Practice 3: Monitor Hash Index Fill Factor

**Problem: Hash Buckets Fill Over Time**
```sql
-- Check hash index statistics (PostgreSQL)
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE indexname LIKE '%_hash';

-- Check for long collision chains (requires pg_buffercache)
SELECT
    c.relname,
    count(*) AS buffers,
    avg(pg_buffercache.usagecount) AS avg_usage
FROM pg_buffercache
JOIN pg_class c ON pg_buffercache.relfilenode = pg_relation_filenode(c.oid)
WHERE c.relname LIKE '%_hash'
GROUP BY c.relname;
```

**Rebuild Hash Index Periodically:**
```sql
-- After significant data growth (e.g., 2× rows since index created)
REINDEX INDEX idx_sessions_hash;

-- Why rebuild?
-- - Rehashing optimizes bucket count for current data size
-- - Reduces collision chain lengths
-- - Improves cache locality

-- Schedule: Quarterly or after 50% data growth
```

### Practice 4: Avoid Hash Indexes in Replicated/Distributed Systems

**Problem: Hash Values Not Portable**
```sql
-- Primary server: Linux x86_64
CREATE INDEX idx_users_hash ON users USING HASH (email);
hash('alice@example.com') = 0x3FA29BC1  (32-bit hash)

-- Replica server: Linux ARM64
hash('alice@example.com') = 0x7E4C21A9  (different implementation!)

-- Result: Queries on replica return wrong results or fail! ❌
```

**Why This Happens:**
- Hash functions may differ by CPU architecture (endianness, word size)
- Compiler optimizations change hash computation
- Database versions use different hash algorithms

**Solution:**
```sql
-- Use B-Tree indexes for replicated systems
CREATE INDEX idx_users_email ON users(email);  -- B-Tree (default)

-- B-Tree key comparison is portable:
-- 'alice@example.com' < 'bob@example.com' is same on all platforms
```

---

## 5. Common Mistakes

### Mistake 1: Using Hash Index for Multi-Column Queries

**Problem:**
```sql
-- Composite hash index (PostgreSQL doesn't support! But hypothetically...)
CREATE INDEX idx_users_country_city_hash ON users USING HASH (country, city);

-- Query with partial match:
SELECT * FROM users WHERE country = 'US';  -- Missing city!

-- Hash computes on BOTH columns:
hash(country='US', city=???) = ???  ← Can't compute without city value!

-- Result: Index NOT used, seq scan instead ❌
```

**Why Hash Fails for Partial Matches:**
```
Hash function: h(country, city) = hash(concat(country, city))

h('US', 'NYC') = 0x1A2B3C4D
h('US', 'LA')  = 0x9E8F7D6C  ← Completely different hash!

Knowing country='US' doesn't help narrow down buckets
vs B-Tree: Can traverse subtree for country='US', then filter cities
```

**Solution:**
```sql
-- Use B-Tree for composite queries
CREATE INDEX idx_users_country_city_btree ON users(country, city);

-- Query:
SELECT * FROM users WHERE country = 'US';

-- Plan:
Index Scan using idx_users_country_city_btree
  Index Cond: (country = 'US')
  -- Can use leftmost prefix of composite B-Tree ✅
```

### Mistake 2: Hash Index on Nullable Column

**Problem:**
```sql
CREATE INDEX idx_orders_ref_hash ON orders USING HASH (external_reference);
-- external_reference: VARCHAR, nullable (50% of orders have NULL)

-- Query:
SELECT * FROM orders WHERE external_reference = 'REF-12345';

-- Hash index stores:
-- Bucket 0: ...
-- Bucket 5: [key='REF-12345', ...], [key='REF-67890', ...], [key=NULL, ...], [key=NULL, ...], ...
              ↑ Many NULLs in various buckets

-- Problem: hash(NULL) is defined but not useful
-- - All NULLs hash to same bucket (bucket #0 often)
-- - Creates hot spot (unbalanced load)
-- - NULLs don't match equality (NULL != NULL in SQL)
```

**Solution:**
```sql
-- Option 1: Partial index excluding NULLs
CREATE INDEX idx_orders_ref_hash ON orders USING HASH (external_reference)
WHERE external_reference IS NOT NULL;

-- Only indexes 50% of rows (those with external_reference)
-- No NULL collision issue ✅

-- Option 2: Use B-Tree (handles NULLs efficiently)
CREATE INDEX idx_orders_ref_btree ON orders(external_reference);
-- B-Tree stores NULLs at beginning/end (sorted), not scattered
```

### Mistake 3: Treating Hash Index as "Always Faster"

**Benchmark Gone Wrong:**
```sql
-- Developer: "Hash is O(1), let's use it everywhere!"

-- Table: 1000 rows
CREATE TABLE products (product_id INT PRIMARY KEY, name VARCHAR);

-- Create hash index
CREATE INDEX idx_products_id_hash ON products USING HASH (product_id);

-- Benchmark:
EXPLAIN ANALYZE SELECT * FROM products WHERE product_id = 42;

-- Hash result: 0.05ms
-- B-Tree result: 0.07ms

-- Developer: "Hash is 40% faster! Ship it!" ❌

-- Problem: For small tables, difference is noise
-- - 0.02ms difference is insignificant
-- - Lost B-Tree features (ranges, sorting)
-- - Hash index size may be larger (collision overhead)
```

**Correct Approach:**
```sql
-- Benchmark on production-scale data (100M rows)
-- Measure end-to-end query time (not just index lookup)

-- Example:
-- B-Tree: 0.5ms index + 0.3ms heap fetch = 0.8ms total
-- Hash: 0.2ms index + 0.3ms heap fetch = 0.5ms total
-- Savings: 0.3ms (37% faster)

-- At scale (10K queries/sec):
-- Savings: 0.3ms × 10K = 3000ms = 3 CPU-seconds per second
-- Impact: 300% CPU reduction → Meaningful! ✅

-- Decision criteria:
-- - Absolute time save > 0.1ms (measurable impact)
-- - Relative save > 20% (not noise)
-- - Query frequency > 1K/sec (high traffic)
-- ALL THREE → Consider hash index
```

### Mistake 4: Ignoring Write Amplification

**Problem:**
```sql
-- Insert-heavy table (logs, events)
CREATE TABLE events (
    event_id BIGSERIAL PRIMARY KEY,
    user_id BIGINT,
    event_type VARCHAR,
    timestamp TIMESTAMPTZ
);

-- Developer adds hash indexes:
CREATE INDEX idx_events_user_hash ON events USING HASH (user_id);
CREATE INDEX idx_events_type_hash ON events USING HASH (event_type);

-- Insert benchmark:
-- Without indexes: 100K inserts/sec
-- With hash indexes: 60K inserts/sec (40% slower!)

-- Why?
-- - Each insert updates 2 hash indexes
-- - Hash collisions require scanning chains (write cost)
-- - Hash bucket page locks contention (many writers)
```

**Solution:**
```sql
-- For write-heavy tables, minimize indexes
DROP INDEX idx_events_type_hash;  -- event_type has low cardinality (not selective)

-- Keep only essential indexes
-- If reads are infrequent, seq scan acceptable

-- Alternative: Batch writes with periodic index rebuild
-- - Insert without indexes (fast)
-- - Build indexes after batch complete (amortized cost)

CREATE INDEX idx_events_user_hash ON events USING HASH (user_id);
```

---

## 6. Security Considerations

### 1. Hash Collision Attacks (Denial of Service)

**Vulnerability:**
```sql
-- Attacker knows hash function (predictable)
-- Crafts keys that all hash to same bucket

-- Example: PostgreSQL uses h(key) = key mod num_buckets

-- Attacker inserts:
INSERT INTO users (user_id, email) VALUES
  (5, 'attacker1@example.com'),
  (1005, 'attacker2@example.com'),
  (2005, 'attacker3@example.com'),
  ...
  (1000005, 'attacker1000@example.com');

-- All user_ids mod 1000 = 5 → Bucket #5 has 1000 entries!

-- Victim query:
SELECT * FROM users WHERE user_id = 5;
-- Scans 1000-entry chain → O(N) instead of O(1)
-- Query time: 0.5ms → 50ms (100× slower)
-- Repeat 1000× → 50 seconds of CPU (DOS!)
```

**Mitigation:**

1. **Use cryptographic hash with secret seed:**
```sql
-- PostgreSQL: Use pg_crypto extension
CREATE EXTENSION pgcrypto;

-- Hash with secret key (not predictable by attacker)
CREATE INDEX idx_users_hash ON users USING HASH (hmac(user_id::text, 'SECRET_KEY', 'sha256'));

-- Attacker can't predict bucket without knowing SECRET_KEY
```

2. **Rate limiting:**
```
-- Application layer: Limit queries per IP
-- Database layer: Set statement_timeout
SET statement_timeout = '100ms';  -- Kill queries exceeding 100ms
```

3. **Monitor anomalous hash chain lengths:**
```sql
-- Alert if any bucket has >10× average chain length
-- Indicates collision attack or poor hash function
```

### 2. Timing Attacks for Hash Existence

**Risk:**
```sql
-- Hash index lookup timing reveals if key exists

-- Key exists:
SELECT * FROM users WHERE api_key = 'valid-key-abc123';
-- Time: 0.5ms (hash lookup + heap fetch)

-- Key doesn't exist:
SELECT * FROM users WHERE api_key = 'invalid-key-xyz789';
-- Time: 0.3ms (hash lookup only, early termination)

-- Attacker can enumerate valid API keys via timing
```

**Mitigation:**
- Use constant-time comparison in application
- Add random delay jitter
- Use B-Tree (timing more uniform due to log N tree traversal)

---

## 7. Performance Optimization

### Optimization 1: Increase Hash Bucket Count for Large Tables

**Problem: Default Bucket Count Too Small**
```sql
-- PostgreSQL initial buckets: 256 (default)
CREATE INDEX idx_sessions_hash ON sessions USING HASH (session_id);

-- After growth: 10M sessions
-- Load factor: 10M / 256 = 39,000 keys per bucket!
-- Collision chains: Average 39K entries per bucket → O(N) scan!

-- Query performance:
-- Expected: 0.1ms (O(1) hash)
-- Actual: 500ms (O(N) chain scan)
```

**Solution: Rebuild with More Buckets**
```sql
-- PostgreSQL: Rehash automatically, but can rebuild to optimize
REINDEX INDEX idx_sessions_hash;

-- After reindex: Buckets grow to accommodate load factor α < 1
-- New bucket count: ~10M buckets (α = 1.0)
-- Collision chains: Average 1 entry per bucket → O(1) ✅

-- Query performance restored:
-- Time: 0.1ms (true O(1) lookup)
```

### Optimization 2: Use Hash Partitioning with Multiple Hash Indexes

**Scenario: 1B row table, hot hash index**

**Naive Approach:**
```sql
-- Single hash index on 1B rows
CREATE INDEX idx_events_user_hash ON events USING HASH (user_id);

-- Problems:
-- - Index size: 10GB (doesn't fit in RAM)
-- - Cache thrashing: Frequent index page evictions
-- - Contention: Many writers lock hash bucket pages
```

**Optimized: Hash Partitioning + Local Indexes**
```sql
-- Partition table by hash of user_id (16 partitions)
CREATE TABLE events (
    event_id BIGSERIAL,
    user_id BIGINT,
    data JSONB
) PARTITION BY HASH (user_id);

CREATE TABLE events_p0 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 0);
CREATE TABLE events_p1 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 1);
...
CREATE TABLE events_p15 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 15);

-- Create local hash index on each partition
CREATE INDEX idx_events_p0_user_hash ON events_p0 USING HASH (user_id);
CREATE INDEX idx_events_p1_user_hash ON events_p1 USING HASH (user_id);
...
CREATE INDEX idx_events_p15_user_hash ON events_p15 USING HASH (user_id);

-- Benefits:
-- - Each index: 10GB / 16 = 625MB (fits in RAM)
-- - Parallel queries: 16 partitions can be scanned concurrently
-- - Less contention: Writes distributed across 16 indexes

-- Query performance:
SELECT * FROM events WHERE user_id = 12345;

-- Plan:
-- 1. Router: user_id 12345 mod 16 = 9 → Query partition events_p9 only
-- 2. Index scan on idx_events_p9_user_hash
-- Total: 1/16th of data scanned → 16× faster!
```

---

## 8. Examples

### Example 1: In-Memory Session Store (PostgreSQL)

**Scenario:** Web app with 10M active sessions, all lookups by session_id (UUID)

```sql
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY,
    user_id BIGINT NOT NULL,
    data JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ
);

-- Workload:
-- - 100K requests/sec
-- - Each request: SELECT * FROM sessions WHERE session_id = ?
-- - Never query ranges, sorts, or filters by other columns

-- Baseline: B-Tree index (PRIMARY KEY creates B-Tree)
EXPLAIN ANALYZE 
SELECT * FROM sessions WHERE session_id = 'f47ac10b-58cc-4372-a567-0e02b2c3d479';

-- Plan:
Index Scan using sessions_pkey on sessions  (cost=0.43..8.45 rows=1)
  Index Cond: (session_id = 'f47ac10b-58cc-4372-a567-0e02b2c3d479')
-- Execution: 0.521ms

-- Create hash index:
CREATE INDEX idx_sessions_hash ON sessions USING HASH (session_id);

-- Drop B-Tree (keep only hash):
ALTER TABLE sessions DROP CONSTRAINT sessions_pkey;
ALTER TABLE sessions ADD CONSTRAINT sessions_pkey PRIMARY KEY USING INDEX idx_sessions_hash;
-- (Note: PostgreSQL 15+ allows hash primary keys)

-- Retest:
EXPLAIN ANALYZE 
SELECT * FROM sessions WHERE session_id = 'f47ac10b-58cc-4372-a567-0e02b2c3d479';

-- Plan:
Index Scan using idx_sessions_hash on sessions  (cost=0.00..8.01 rows=1)
  Index Cond: (session_id = 'f47ac10b-58cc-4372-a567-0e02b2c3d479')
-- Execution: 0.312ms (40% faster)

-- Impact at scale:
-- Savings: 0.21ms × 100K queries/sec = 21 seconds CPU per second
-- Result: 2100% CPU reduction → Major win! ✅
```

---

### Example 2: Deduplication with Hash Index (MySQL)

**Scenario:** ETL pipeline needs to detect duplicate records

```sql
CREATE TABLE staging_data (
    record_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    external_id VARCHAR(255),
    data TEXT,
    imported_at DATETIME
);

-- Goal: Detect duplicates by external_id (1M records/day)

-- Naive approach: Self-join
SELECT s1.record_id, s2.record_id
FROM staging_data s1
JOIN staging_data s2 ON s1.external_id = s2.external_id
WHERE s1.record_id < s2.record_id;

-- Plan: Nested Loop Join (O(N²))
-- Execution: 15 minutes (terrible!)

-- Optimized: Hash index for equality check
-- MySQL 8.0+ supports hash indexes on InnoDB (via adaptive hash index, but we'll use explicit HASH)
-- Note: MySQL hash indexes only supported on MEMORY tables, so we'll use B-Tree

-- Actually, best approach: Use GROUP BY with HAVING
SELECT external_id, COUNT(*) AS duplicate_count
FROM staging_data
GROUP BY external_id
HAVING COUNT(*) > 1;

-- With B-Tree index:
CREATE INDEX idx_staging_external ON staging_data(external_id);

-- Plan: Index Scan + GroupAggregate
-- Execution: 2 seconds (450× faster!)

-- For true hash deduplication, use application-side hash table:
-- - Read all external_ids into dict/HashMap
-- - Check for duplicates in O(N) time
```

---

## 9. Real-World Use Cases

### Use Case 1: Discord - Message ID Lookups

**Context:** 
- Discord has billions of messages
- Each message has unique Snowflake ID (64-bit integer)
- Primary query: Fetch message by ID (exact match only)

**Implementation:**
```sql
-- Messages partitioned by channel_id (shard key)
CREATE TABLE messages (
    message_id BIGINT PRIMARY KEY,
    channel_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    content TEXT,
    created_at TIMESTAMPTZ
) PARTITION BY HASH (channel_id);

-- Each partition has hash index on message_id
-- (For read-heavy workload with no range queries)

-- Query pattern:
SELECT * FROM messages WHERE message_id = 1234567890123456789;

-- Performance:
-- B-Tree: 0.8ms (4-level tree for 10B messages)
-- Hash: 0.3ms (direct bucket lookup)
-- Savings: 0.5ms × 10K queries/sec = 5 CPU-seconds/sec → 500% CPU reduction

-- Trade-off accepted:
-- - No message range queries needed (fetch by ID or channel+time)
-- - Hash index 30% smaller than B-Tree (saves disk space)
```

### Use Case 2: Memcached - Key-Value Store

**Implementation:**
- Memcached doesn't use SQL, but internal structure is hash table
- 100% hash-based lookups (no B-Tree)

**Why Hash Works:**
- **Workload:** Only GET/SET by exact key (never ranges)
- **In-memory:** No disk I/O (hash O(1) vs log N matters more)
- **Simple keys:** Short strings or integers (hash fast to compute)

**Performance:**
```
Benchmark: 1M keys in memory

Hash table:
- GET: 50,000 ops/sec per CPU core
- SET: 40,000 ops/sec per CPU core
- Latency: 20 microseconds (50K/sec)

Hypothetical B-Tree (Redis uses for sorted sets):
- GET: 30,000 ops/sec per CPU core (40% slower)
- SET: 25,000 ops/sec per CPU core (38% slower)
- Latency: 33 microseconds

Hash advantage: 40% more throughput (in-memory workload)
```

---

## 10. Interview Questions

### Q1: Why don't databases use hash indexes as default instead of B-Tree?

**Senior Answer:**

```
**Reasons B-Tree is default:**

1. **Versatility:**
   - B-Tree: Single index supports =, >, <, BETWEEN, LIKE 'prefix%', ORDER BY
   - Hash: Only supports = (equality)
   - Most queries need ranges at some point (e.g., WHERE date > '2024-01-01')

2. **Marginal performance difference on disk:**
   - Hash theoretical: O(1)
   - B-Tree theoretical: O(log N)
   - For 1B rows: log₂(1B) = 30 comparisons
   - Actual difference: 2-4 page reads vs 3-5 page reads
   - Savings: 1-2 I/Os = 0.1-0.2ms (marginal)
   - I/O dominates, not comparison count

3. **Cache-friendliness:**
   - B-Tree: Sequential access patterns (prefetching works)
   - Hash: Random bucket access (poor cache locality, especially for range queries — oh wait, hash doesn't support ranges!)
   - B-Tree internal nodes stay hot in cache (ref: top 2 levels of tree)

4. **Collision handling overhead:**
   - Hash: Collision chains degrade to O(N) worst case
   - B-Tree: Always O(log N), predictable performance

5. **Ordering preservation:**
   - B-Tree: Keys sorted, ORDER BY free
   - Hash: Random order, must sort results

**Real-world example:**

E-commerce product catalog:

```sql
-- Queries on products table:
WHERE sku = 'LAPTOP-X1'           -- Exact match (hash would help)
WHERE price > 100                  -- Range (hash fails)
WHERE name LIKE 'Apple%'           -- Prefix (hash fails)
ORDER BY price DESC LIMIT 10       -- Sort (hash fails)

-- B-Tree handles ALL queries with one index
-- Hash would require 4 different indexes or fallback to seq scan
```

**When hash IS better:**

- In-memory databases (Redis, Memcached): I/O not a factor, O(1) vs O(log N) matters
- Session stores: Only exact lookups by session ID
- Deduplication: Hash into buckets for grouping

**Interview follow-up:**

"In my experience, I've only created 2 hash indexes in production: (1) session_id lookups in a caching layer (all in-memory, only exact matches), and (2) UUID-based API key validation (billion API keys, only equality checks). For 99% of cases, B-Tree's versatility outweighs hash's marginal speed advantage."
```

---

### Q2: Explain collision resolution in hash indexes and how it affects performance.

**Staff Answer:**

```
**Collision Resolution Strategies:**

**1. Chaining (Most Common in Databases):**

Structure:
- Array of buckets
- Each bucket → Linked list of entries with same hash

```
Bucket 5: → [key=5, rid=100] → [key=1005, rid=200] → [key=2005, rid=300] → NULL
             ↑ First entry     ↑ Collision         ↑ Another collision
```

**Insert algorithm:**
```
INSERT(key, value):
  bucket_idx = hash(key) % num_buckets
  new_entry = {key, value}
  new_entry.next = buckets[bucket_idx]  // Prepend (O(1))
  buckets[bucket_idx] = new_entry
```

**Lookup algorithm:**
```
LOOKUP(key):
  bucket_idx = hash(key) % num_buckets
  entry = buckets[bucket_idx]
  while entry != NULL:
    if entry.key == key:
      return entry.value  // Found
    entry = entry.next
  return NOT_FOUND
```

**Performance characteristics:**

- **Best case (no collisions):** O(1) — single comparison
- **Average case:** O(1 + α) where α = load factor (N/B)
- **Worst case (all keys in one bucket):** O(N) — linear scan of entire list

**Example:**

```
1M keys, 1M buckets → α = 1.0
- Average chain length: 1
- Average comparisons: 1 + 1 = 2 (hash + 1 comparison)

1M keys, 100K buckets → α = 10
- Average chain length: 10
- Average comparisons: 1 + 5 = 6 (hash + half of 10-entry chain)
- Performance: 3× slower

1M keys, 10K buckets → α = 100
- Average chain length: 100
- Average comparisons: 1 + 50 = 51
- Performance: 25× slower (degrades to near-sequential scan!)
```

**2. Open Addressing (Less Common):**

Structure:
- Single array, no linked lists
- On collision, probe for next empty slot

**Linear probing:**
```
INSERT(key, value):
  idx = hash(key) % num_buckets
  while buckets[idx] != NULL:
    if buckets[idx].key == key:
      buckets[idx].value = value  // Update
      return
    idx = (idx + 1) % num_buckets  // Next slot
  buckets[idx] = {key, value}
```

**Performance:**

- **Best case:** O(1)
- **Average case:** O(1 / (1 - α))
  - α = 0.5 → 2 probes
  - α = 0.75 → 4 probes
  - α = 0.9 → 10 probes (clustering!)
- **Worst case:** O(N) — entire table is probed

**Problem: Primary clustering**

```
Initial:
[NULL, key1, key2, NULL, NULL, key3, NULL]

Insert key4 (hashes to index 1):
- Slot 1 full → Probe 2
- Slot 2 full → Probe 3
- Slot 3 free → Insert at 3

Result:
[NULL, key1, key2, key4, NULL, key3, NULL]
         ↑____________↑____↑  Long cluster!

Next insertion anywhere in cluster → Must scan entire cluster
Clustering grows! (cascading effect)
```

**Real-world impact:**

PostgreSQL uses chaining:
- Pros: Predictable performance (O(1 + α))
- Pros: No clustering issues
- Cons: Extra memory for pointers

MySQL InnoDB uses adaptive hash index (hidden):
- Auto-creates hash index for hot B-Tree pages
- Uses chaining internally
- Transparent to user

**Performance monitoring:**

```sql
-- Detect high collision rates:
-- - Average chain length > 5 → Need more buckets
-- - Max chain length > 50 → Hot spot or bad hash function
-- - Variance in chain lengths > 10× → Poor distribution

-- Solution: REINDEX to rebuild with more buckets
```

**Interview insight:**

"In production, I've debugged a hash index slowdown caused by skewed data. Our user_id hash was h(x) = x mod 1000, but user IDs were auto-incrementing sequential integers. This meant every 1000th user hashed to the same bucket, creating hot spots. We switched to a better hash function (MurmurHash) that distributed sequential IDs uniformly, reducing average chain length from 50 → 2 (25× speedup)."
```

---

## 11. Summary

### Hash Index Key Principles

1. **O(1) Lookups (Best Case)**  
   Direct bucket access → Constant time  
   **BUT:** Only for exact equality, not ranges

2. **Collision Handling Critical**  
   Load factor α < 1 maintains performance  
   α > 2 degrades to O(N) chain scans

3. **Limited Query Support**  
   ✅ WHERE x = ?  
   ❌ WHERE x > ?  
   ❌ ORDER BY x  
   ❌ LIKE 'prefix%'

4. **B-Tree Usually Better**  
   Marginal difference on disk (2-4 vs 3-5 I/Os)  
   B-Tree versatility > hash's narrow advantage

### When to Use Hash Index

**✅ Use Hash Index When:**
- **Only exact equality queries** (never ranges, sorts, prefix matches)
- **High-cardinality keys** (UUID, long strings) where hash saves space
- **Proven bottleneck** (profiled >20% speedup)
- **High-frequency queries** (>1K/sec) where 0.2ms savings = 200ms CPU/sec

**❌ Avoid Hash Index When:**
- Any range queries needed (`WHERE x > ?`)
- Need sorted output (`ORDER BY x`)
- Prefix matching (`LIKE 'A%'`)
- Multi-column queries (hash doesn't support partial keys)
- Write-heavy workload (collision updates expensive)

### Decision Matrix

| Scenario | Recommendation | Reason |
|----------|----------------|--------|
| Session ID lookup | Hash | Only exact match, high frequency |
| User email lookup | B-Tree | Need LIKE 'prefix%' for search |
| Order ID range query | B-Tree | Need WHERE order_id > ? |
| JOIN on foreign key | B-Tree | Need ranges for join algorithms |
| API key validation | Hash (maybe) | Exact match only, if proven bottleneck |
| Date filtering | B-Tree | Always need ranges for dates |

### Performance Reality Check

```
**Theoretical:**
Hash: O(1)
B-Tree: O(log N)

For 1B rows: B-Tree = 30 comparisons
→ Hash should be 30× faster!

**Actual (on disk):**
Hash: 2-3 page reads
B-Tree: 3-5 page reads

Difference: 1-2 I/Os = 0.1-0.2ms
→ Hash is 1.5-2× faster (not 30×!)

**Why?**
I/O dominates, not comparison count.
```

### Quick Reference

**PostgreSQL:**
```sql
-- Create hash index
CREATE INDEX idx_name ON table USING HASH (column);

-- Rebuild (after data growth)
REINDEX INDEX idx_name;
```

**MySQL:**
- InnoDB: No explicit hash indexes (uses adaptive hash internally)
- MEMORY engine: `CREATE INDEX USING HASH`

**SQL Server:**
- No hash indexes (B-Tree only called "nonclustered index")

### Key Takeaway

**"Hash indexes are a micro-optimization. Start with B-Tree. Switch to hash only if profiling shows index lookup (not heap access, not network) is the bottleneck AND you never need ranges/sorting."**

In 15 years of database work, B-Tree indexes solve 99% of performance problems. Hash indexes are for the remaining 1% where you've optimized everything else and need that last 0.2ms.

---

**Next file:** `03_Bitmap_Indexes.md` - Optimizing low-cardinality columns in data warehouses