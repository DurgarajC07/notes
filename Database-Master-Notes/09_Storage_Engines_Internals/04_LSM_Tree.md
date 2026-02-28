# LSM-Tree - Log-Structured Merge Tree Architecture

## 1. Concept Explanation

**LSM-Tree (Log-Structured Merge Tree)** is a write-optimized storage structure used by databases like RocksDB, LevelDB, Cassandra, and HBase. Understanding LSM-Trees is critical for modern distributed systems and time-series databases.

**Core Idea: Sequential Writes > Random Writes**

```
Disk performance (HDD):
- Sequential write: 100 MB/s ← Fast (disk head moves in one direction)
- Random write: 1 MB/s ← Slow (disk head seeks to random locations)
100× difference! ✓

SSD performance:
- Sequential write: 500 MB/s
- Random write: 100 MB/s
5× difference (still significant)

LSM-Tree strategy: Convert random writes → sequential writes
```

**LSM-Tree Structure:**

```
+------------------+
|   Memtable       |  ← In-memory buffer (writes go here first)
|   (SkipList)     |  ← Red-Black tree or Skip List (sorted)
|   Size: 64 MB    |
+------------------+
        ↓ (when full, flush to disk)
+------------------+
|   L0 SSTables    |  ← Level 0 (Sorted String Tables)
|   [SST 1][SST 2] |  ← Immutable files on disk
+------------------+
        ↓ (compaction merges levels)
+------------------+
|   L1 SSTables    |  ← Level 1 (10× larger than L0)
|   [SST 1][SST 2][SST 3]...
+------------------+
        ↓
+------------------+
|   L2 SSTables    |  ← Level 2 (10× larger than L1)
|   [SST 1-100]    |
+------------------+
        ↓
...
+------------------+
|   L6 SSTables    |  ← Level 6 (largest level)
|   [SST 1-10000]  |  ← Bulk of data lives here
+------------------+

Key properties:
1. Writes: Always append to memtable (in-memory, fast)
2. Reads: Check memtable → L0 → L1 → ... → L6 (multiple levels, slower)
3. Compaction: Background process merges levels (keeps sorted order)
```

**Write Path:**

```
1. Write to WAL (Write-Ahead Log) for durability
   → Sequential append to log file (fast!)

2. Insert into memtable (in-memory sorted structure)
   → Red-Black tree or SkipList insertion (O(log n))

3. When memtable full (64 MB default):
   → Flush to disk as immutable SSTable (Level 0)
   → Sequential write (100 MB/s on HDD) ✓

4. Background compaction:
   → Merge overlapping SSTables from L0 → L1
   → Merge overlapping SSTables from L1 → L2
   → ...
   → Result: Data eventually migrates to deeper levels

Example write operation:
INSERT INTO users (id, name) VALUES (1000, 'Alice');

Step 1: WAL write (sequential)
  [WAL file] ← Append: "INSERT id=1000, name=Alice"

Step 2: Memtable insert
  Memtable (before): {id=1 → 'Bob', id=500 → 'Carol'}
  Memtable (after):  {id=1 → 'Bob', id=500 → 'Carol', id=1000 → 'Alice'}

Step 3: Memtable full (64 MB)
  → Flush to L0 as SST file
  SST_0001.sst: {id=1000 → 'Alice', ...}

Step 4: Compaction (background)
  L0 (SST_0001): {id=1000, ...}
  L1 (SST_0010): {id=900-1100, ...}
  → Merge → L1 (SST_0011): {id=900-1100 with id=1000 updated}
```

**Read Path:**

```
Query: SELECT * FROM users WHERE id = 1000;

Step 1: Check memtable (in-memory, fastest)
  → If found: Return immediately ✓
  → If not found: Continue to disk

Step 2: Check L0 SSTables (newest data)
  → Search SST_0001, SST_0002, ..., SST_000N
  → Use bloom filters (probabilistic, 99% false positive reduction)
  → If found: Return ✓
  → If not found: Continue to L1

Step 3: Check L1 SSTables
  → Search SST_0010, SST_0011, ...
  → If found: Return ✓
  → If not found: Continue to L2

... (repeat for L2, L3, ..., L6)

Worst case: Check memtable + L0 (10 files) + L1 (10 files) + ... + L6 (100 files)
→ 100+ disk reads (slow!) ✗

Optimization: Bloom filters reduce disk reads by 99%
→ Typical read: 1-5 disk IOs ✓
```

---

## 2. Why It Matters

### Production Success: Facebook's RocksDB at Scale

**Scenario: Facebook Social Graph (Billions of Operations/Second)**

```
Date: 2013-present
Company: Meta (Facebook)
Database: RocksDB (LSM-Tree based on LevelDB)
Scale: Petabytes of data, billions of operations/second

Use case: Social graph (friends, likes, comments, reactions)

Workload characteristics:
- Writes: 1 billion writes/second (new posts, likes, comments)
- Reads: 10 billion reads/second (news feed generation)
- Data: 100 petabytes (user profiles, relationships, content)
- Write pattern: Append-heavy (new content, no UPDATE of old posts)

Why LSM-Tree (RocksDB)?

Problem with B-Tree (traditional InnoDB):
-- Write operation (e.g., new "like" on post)
INSERT INTO likes (user_id, post_id, timestamp) VALUES (123, 456, NOW());

B-Tree behavior:
1. Find leaf page for new row (random disk read) ← Slow!
2. Modify leaf page in memory
3. Flush modified page to disk (random write) ← Slow!

Result:
- Random I/O: 1 MB/s (HDD) or 100 MB/s (SSD)
- 1 billion writes/second → Impossible! ✗

LSM-Tree (RocksDB) behavior:
1. Append to WAL (sequential write: 100 MB/s HDD, 500 MB/s SSD) ✓
2. Insert into memtable (in-memory, nanoseconds)
3. Background compaction (batched, sequential I/O)

Result:
- Write throughput: 10× higher than B-Tree ✓
- 1 billion writes/second → Achievable with 10,000 RocksDB instances ✓

Facebook's RocksDB deployment:

Configuration:
write_buffer_size = 67108864          # 64 MB memtable
max_write_buffer_number = 3           # 3 memtables (1 active, 2 flushing)
level0_file_num_compaction_trigger = 4 # Compact L0 when 4 files
level0_slowdown_writes_trigger = 20    # Slow writes at 20 L0 files
level0_stop_writes_trigger = 36        # Stop writes at 36 L0 files

target_file_size_base = 67108864      # 64 MB SSTable size
max_bytes_for_level_base = 268435456  # 256 MB for L1
max_bytes_for_level_multiplier = 10   # Each level 10× larger

# Bloom filters (reduce read amplification)
filter_policy = rocksdb.BloomFilterPolicy(10)  # 10 bits per key

Performance metrics (production):

Write performance:
- Throughput: 100,000 writes/second per RocksDB instance
- Latency: p50 = 0.5ms, p99 = 5ms (in-memory write)
- Durability: WAL fsync every 1 second (configurable)

Read performance:
- Throughput: 1,000,000 reads/second per instance
- Latency: p50 = 1ms, p99 = 10ms (bloom filters + cache)
- Cache hit ratio: 95% (memtable + block cache)

Compaction:
- Background threads: 16 (parallel compaction)
- I/O bandwidth: 100 MB/s (sequential writes)
- CPU usage: 20% (mostly compaction)

Business impact:
- News feed latency: 200ms p99 (fast despite billions of posts)
- Write capacity: 1 billion writes/second globally
- Storage efficiency: 50% space savings (compression in SSTables)
- Operational simplicity: Automatic compaction (no manual VACUUM like PostgreSQL)

Alternative approaches considered:

MySQL InnoDB (B-Tree):
- Write throughput: 10,000 writes/second per instance (100× worse)
- Verdict: Cannot scale to 1 billion writes/second ✗

Cassandra (LSM-Tree):
- Write throughput: Similar to RocksDB ✓
- Read performance: Slower (no in-process optimization like RocksDB)
- Verdict: RocksDB chosen for better read latency

PostgreSQL (B-Tree heap):
- Write throughput: 5,000 writes/second per instance (200× worse)
- Verdict: Cannot scale ✗

Result: RocksDB powering Facebook, Instagram, WhatsApp (billions of users)
```

---

## 3. Internal Working

### SSTable (Sorted String Table) Format

**SSTable File Structure:**

```
SSTable file: 000123.sst (64 MB)

+------------------+
| Data Block 1     |  ← 4 KB block (compressed)
|  key1: value1    |  ← Sorted key-value pairs
|  key2: value2    |
|  ...             |
+------------------+
| Data Block 2     |  ← Another 4 KB block
|  key100: value   |
|  ...             |
+------------------+
| ...              |
+------------------+
| Filter Block     |  ← Bloom filter (probabilistic presence check)
|  "Does key exist?|
|   Bloom[hash1]   |
|   Bloom[hash2]"  |
+------------------+
| Meta Index Block |  ← Points to filter block
+------------------+
| Index Block      |  ← Points to data blocks
|  "key1-key99: Block 1"
|  "key100-key199: Block 2"
|  ...             |
+------------------+
| Footer           |  ← Metaindex and index block offsets
|  magic number    |
|  version         |
+------------------+

Read operation:
1. Read footer (last 48 bytes)
2. Read index block → Find data block containing key
3. Check bloom filter → 99% "key not present" filtered out (skip disk read)
4. Read data block → Binary search for key
5. Decompress and return value
```

### Compaction Strategies

**1. Size-Tiered Compaction (Cassandra Default):**

```
Strategy: Merge SSTables of similar size

Level 0: [1MB] [1MB] [1MB] [1MB]  ← 4 files of ~1MB
  → Merge when 4 files → Create L1 file (4MB)

Level 1: [4MB] [4MB] [4MB] [4MB]  ← 4 files of ~4MB
  → Merge when 4 files → Create L2 file (16MB)

Level 2: [16MB] [16MB] [16MB]
  → Merge when 4 files → Create L3 file (64MB)

Pros:
✓ Write amplification: 2× (each byte written 2 times on average)
✓ Simple algorithm

Cons:
✗ Space amplification: 2× (temporary space during compaction)
✗ Read amplification: High (many overlapping SSTables)
```

**2. Leveled Compaction (RocksDB Default):**

```
Strategy: Each level has non-overlapping SSTables

L0: [SST1: keys 1-100] [SST2: keys 50-150] ← Overlapping (from memtable flushes)

L1: [SST1: keys 1-100] [SST2: keys 101-200] [SST3: keys 201-300] ← Non-overlapping!

L2: [SST1: keys 1-1000] [SST2: keys 1001-2000] ... ← Non-overlapping, 10× larger than L1

Compaction process:
1. Select SSTable from L0 (e.g., keys 1-100)
2. Find overlapping SSTables in L1 (keys 1-100)
3. Merge L0[1-100] + L1[1-100] → L1_new[1-100]
4. Delete old SSTables

Pros:
✓ Read amplification: 1 SSTable per level (fast reads)
✓ Space amplification: 1.1× (minimal temporary space)

Cons:
✗ Write amplification: 10-50× (data rewritten through multiple levels)

Example write amplification calculation:
Write 1 MB to memtable
→ Flush to L0: 1 MB written
→ Compact to L1: 1 MB written (2 MB total)
→ Compact to L2: 1 MB written (3 MB total)
→ ... repeat for L3-L6
→ Total: 1 MB logical write → 10-50 MB physical writes ✗

Trade-off: Write amplification (10×) for read performance (1 SSTable per level)
```

---

## 4. Best Practices

### Practice 1: Tune Memtable Size

```python
# RocksDB configuration

import rocksdb

opts = rocksdb.Options()

# Memtable size (trade-off: memory vs flush frequency)
opts.write_buffer_size = 67108864  # 64 MB (default)

# Larger memtable (256 MB):
opts.write_buffer_size = 268435456  # 256 MB
# Pros: Fewer flushes (less I/O), better compression (larger blocks)
# Cons: More memory usage, longer recovery (replay larger WAL)

# Smaller memtable (16 MB):
opts.write_buffer_size = 16777216  # 16 MB
# Pros: Less memory, faster recovery
# Cons: Frequent flushes (more L0 files, more compaction)

# Recommendation:
# - Write-heavy workload: 128-256 MB memtable (reduce flush frequency)
# - Memory-constrained: 32-64 MB memtable
# - Fast recovery critical: 16-32 MB memtable
```

### Practice 2: Configure Bloom Filters

```python
# Bloom filter reduces read amplification

opts.filter_policy = rocksdb.BloomFilterPolicy(10)  # 10 bits per key

# How it works:
# Bloom filter answers: "Does key exist in this SSTable?"
# - If "No" (guaranteed): Skip SSTable (avoid disk read) ✓
# - If "Yes" (99% accurate): Read SSTable (1% false positive)

# Bloom filter size calculation:
# 10 bits per key × 1M keys = 10 Mb = 1.25 MB filter per SSTable

# Trade-off:
# - More bits/key: Lower false positive rate (fewer wasted disk reads)
# - Fewer bits/key: Smaller filter (less memory)

# Benchmark:
# 10 bits: 1% false positive (99% of "not found" queries skip disk read)
# 20 bits: 0.01% false positive (99.99% skip)

# Recommendation: 10 bits/key (good balance)
```

---

## 5. Common Mistakes

### Mistake 1: Not Monitoring Write Amplification

**Problem:**
```python
# Default RocksDB config (leveled compaction)
# Write 1 GB of data

Logical writes: 1 GB
Physical writes: 30 GB (30× write amplification!) ✗

# Impact on SSD lifespan:
# SSD rated for 1000 TB writes (TBW)
# With 30× amplification: 1000 TB / 30 = 33 TB usable writes
# If writing 1 TB/day: 33 days until SSD wears out! ✗
```

**Fix:**
```python
# Monitor write amplification
stats = db.get_property(b'rocksdb.stats')
# Look for:
# - "cumulative writes" (physical writes)
# - "cumulative WAL writes" (logical writes)
# - Write amp = cumulative writes / WAL writes

# If write amp > 50×:
# 1. Reduce number of levels: opts.num_levels = 4 (default 7)
# 2. Increase SSTable size: opts.target_file_size_base = 134217728  # 128 MB (default 64 MB)
# 3. Switch to size-tiered compaction (lower write amp, higher read amp)
```

---

## 6. Security Considerations

### Encryption at Rest (SSTable Files)

```python
# RocksDB encryption (Linux LUKS or OS-level)

# Option 1: Encrypt entire disk (LUKS)
# Transparent to RocksDB, but encrypts all files

# Option 2: RocksDB encryption plugin (per-SSTable encryption)
from rocksdb import EncryptionOptions

encryption = EncryptionOptions(
    provider=AESEncryptionProvider(key_manager),
    method='AES256'
)

opts.encryption_options = encryption

# What's encrypted:
# - SSTable files (data blocks)
# - WAL files (write-ahead log)

# What's NOT encrypted:
# - Memtable (in-memory, OS protects)
# - MANIFEST files (metadata, low sensitivity)
```

---

## 7. Performance Optimization

### Benchmark: LSM-Tree vs B-Tree

**Test: Write-Heavy Workload (90% writes, 10% reads)**

| Engine | Writes/sec | Reads/sec | Write Amp | Read Amp | Disk Usage |
|--------|------------|-----------|-----------|----------|------------|
| **RocksDB (LSM)** | 100,000 | 50,000 | 30× | 10 | 1.2× |
| **InnoDB (B-Tree)** | 10,000 | 500,000 | 2× | 1 | 1.5× |

**Observation:**
- RocksDB: 10× better write throughput (LSM optimized for writes)
- InnoDB: 10× better read throughput (B-Tree optimized for reads)
- Trade-off: LSM = write-optimized, B-Tree = read-optimized

**Test: Read-Heavy Workload (10% writes, 90% reads)**

| Engine | Writes/sec | Reads/sec | Latency (p99) |
|--------|------------|-----------|---------------|
| **RocksDB (LSM)** | 100,000 | 200,000 | 10ms |
| **InnoDB (B-Tree)** | 10,000 | 1,000,000 | 5ms |

**Observation:**
- B-Tree wins for read-heavy workloads (5× better read throughput)
- LSM-Tree still achieves 10× better write throughput

---

## 8. Examples

### Example: RocksDB Python API

```python
import rocksdb

# Create database
opts = rocksdb.Options()
opts.create_if_missing = True
opts.write_buffer_size = 67108864  # 64 MB memtable
opts.max_write_buffer_number = 3
opts.target_file_size_base = 67108864  # 64 MB SSTables
opts.filter_policy = rocksdb.BloomFilterPolicy(10)  # Bloom filter

db = rocksdb.DB("/tmp/mydb", opts)

# Write operations (fast, sequential)
db.put(b'key1', b'value1')  # Append to WAL + insert into memtable
db.put(b'key2', b'value2')

# Batch write (atomic, efficient)
batch = rocksdb.WriteBatch()
batch.put(b'key3', b'value3')
batch.put(b'key4', b'value4')
batch.delete(b'key1')  # Delete (tombstone marker)
db.write(batch)

# Read operations
value = db.get(b'key2')  # Check memtable → L0 → L1 → ... → L6
print(value)  # b'value2'

# Range scan (sorted iteration)
it = db.iteritems()
it.seek_to_first()
for key, value in it:
    print(f"{key} → {value}")

# Cleanup
del db
```

---

## 9. Real-World Use Cases

### Use Case: LinkedIn Kafka (LSM-Tree for Commit Log)

```python
# LinkedIn's Kafka uses log-structured storage (similar to LSM-Tree)

class KafkaLogStructuredStorage:
    """
    Kafka's append-only log (LSM-Tree principles)
    """
    @staticmethod
    def write_message(topic, message):
        """
        Append message to log (sequential write)
        """
        # Kafka log structure:
        # topic-0/
        #   00000000000000000000.log (1 GB segment)
        #   00000000001000000000.log (next 1 GB segment)
        #   ...
        
        # Write operation (O(1), sequential):
        # 1. Append to active segment (in-memory buffer)
        # 2. Flush to disk when buffer full (sequential write: 500 MB/s) ✓
        # 3. No compaction needed (append-only)
        
        # Performance:
        # - Write throughput: 1 million messages/second per broker
        # - Latency: p99 < 5ms (in-memory buffer + sequential flush)
        # - Disk: 100% sequential I/O (no random writes) ✓
        
        return {
            'offset': 1000000,  # Message position in log
            'timestamp': '2024-02-01T12:00:00Z'
        }
    
    @staticmethod
    def read_message(topic, offset):
        """
        Read message by offset (O(1), sequential read if cached)
        """
        # Read operation:
        # 1. Calculate segment file: offset // 1GB
        # 2. Seek to offset within segment
        # 3. Read message
        
        # Page cache optimization:
        # - OS caches hot segments in memory (95% cache hit)
        # - Cold messages: Sequential disk read (100 MB/s) ✓
        
        # Performance:
        # - Read throughput: 10 million messages/second (from cache)
        # - Latency: p99 < 1ms (cache hit), p99 < 50ms (disk read)
        
        pass
    
    @staticmethod
    def compaction():
        """
        Kafka compaction (similar to LSM-Tree compaction)
        """
        # Kafka log compaction:
        # - Keep only latest value for each key
        # - Merge old segments → New compacted segment
        # - Delete old segments
        
        # Example:
        # Old segments:
        #   Segment 1: [key1:v1, key2:v1]
        #   Segment 2: [key1:v2, key3:v1]
        #   Segment 3: [key2:v2, key1:v3]
        
        # Compacted segment:
        #   [key1:v3, key2:v2, key3:v1]  ← Latest values only
        
        # Benefits:
        # - 70% disk space savings (LinkedIn production data)
        # - Faster catch-up (consumers replay only latest state)
        
        pass

# LinkedIn Kafka scale:
# - 10 trillion messages/day
# - 10 petabytes of data
# - 7 million writes/second globally
# - Write amplification: 1× (append-only, no compaction for most topics)
# - Result: LSM-Tree principles enable Kafka's massive write throughput ✓
```

---

## 10. Interview Questions

### Q1: Explain write amplification in LSM-Trees. How does it impact SSD lifespan, and what strategies can reduce it?

**Answer:**

**Write Amplification Definition:**
```
Logical write: 1 MB application data
Physical writes: 30 MB (written to disk through compaction)
Write amplification: 30× ✗

Calculation:
1. Write 1 MB to memtable
2. Flush to L0: 1 MB written (cumulative: 1 MB)
3. Compact L0 → L1: Merge 1 MB + 10 MB existing L1 data → Write 11 MB (cumulative: 12 MB)
4. Compact L1 → L2: Merge 11 MB + 100 MB existing L2 data → Write 111 MB (cumulative: 123 MB)
5. ... (repeat for deeper levels)

Total: 1 MB logical → 123 MB physical (123× write amplification!) ✗
```

**SSD Lifespan Impact:**
```
SSD specifications:
- Capacity: 1 TB
- Endurance: 1000 TBW (terabytes written)
- Expected lifespan: 1000 TB / (1 GB/s × 86400s/day) = 11.6 days without write amp ✗

With 30× write amplification:
- Effective endurance: 1000 TB / 30 = 33 TB
- Workload: 1 TB/day writes
- Lifespan: 33 days ✗ (vs 1000 days without amplification)

Industry problem:
- Facebook RocksDB: Billions of writes/sec
- SSD replacement: Every 6-12 months (hardware cost)
- Solution: Reduce write amplification or use over-provisioned SSDs
```

**Strategies to Reduce Write Amplification:**

**1. Increase SSTable Size:**
```python
opts.target_file_size_base = 134217728  # 128 MB (default 64 MB)
# Larger SSTables → Fewer compactions → Lower write amp (30× → 20×)
```

**2. Reduce Number of Levels:**
```python
opts.num_levels = 4  # Default 7
# Fewer levels → Less compaction → Lower write amp (30× → 15×)
# Trade-off: Higher space amplification (more overlap)
```

**3. Use Size-Tiered Compaction:**
```python
# RocksDB: Switch to universal compaction (size-tiered)
opts.compaction_style = rocksdb.CompactionStyle.universal
# Write amp: 30× → 5× ✓
# Trade-off: Read performance degrades (more overlapping SSTables)
```

**4. Tune Compaction Triggers:**
```python
opts.level0_file_num_compaction_trigger = 8  # Default 4
# Delay compaction → Batch more files → Lower write amp
# Trade-off: More L0 files → Slower reads
```

**Interview insight:**
"At [Company], we reduced write amplification from 40× to 12× by increasing SSTable size to 256 MB and using universal compaction for write-heavy tables. This tripled our SSD lifespan from 4 months to 12 months, saving $200K annually in hardware costs. However, read latency increased 20% (p99: 5ms → 6ms), which was acceptable for our write-heavy workload (95% writes). We monitor write amp weekly using `rocksdb.stats` and replace SSDs proactively before endurance limits."

---

## 11. Summary

**LSM-Tree Architecture:**

```
Components:
1. Memtable (RAM): Sorted structure (SkipList/Red-Black tree)
2. WAL: Write-ahead log (durability)
3. SSTables: Immutable sorted files (L0-L6)
4. Compaction: Background merge process

Write path: WAL → Memtable → Flush to L0 → Compact to L1-L6
Read path: Memtable → L0 → L1 → ... → L6 (check all levels)
```

**Performance Characteristics:**

| Metric | LSM-Tree | B-Tree | Winner |
|--------|----------|--------|--------|
| **Write throughput** | 100,000/s | 10,000/s | LSM (10×) |
| **Read throughput** | 200,000/s | 1,000,000/s | B-Tree (5×) |
| **Write amplification** | 10-50× | 2× | B-Tree |
| **Read amplification** | 1-10 | 1 | B-Tree |
| **Space amplification** | 1.1× | 1.5× | LSM |

**Use Cases:**

**LSM-Tree Best For:**
```
✓ Write-heavy workloads (90%+ writes)
✓ Time-series data (append-only logs)
✓ Event streaming (Kafka, RocksDB)
✓ Large-scale distributed systems (Cassandra, HBase)
```

**B-Tree Best For:**
```
✓ Read-heavy workloads (90%+ reads)
✓ OLTP (balanced read/write)
✓ Point queries (find by primary key)
✓ Range scans with low latency requirements
```

**RocksDB Configuration (Production):**

```python
opts = rocksdb.Options()

# Memory
opts.write_buffer_size = 134217728       # 128 MB memtable
opts.max_write_buffer_number = 3         # 3 memtables

# SSTables
opts.target_file_size_base = 134217728   # 128 MB SSTables
opts.max_bytes_for_level_base = 1073741824  # 1 GB L1

# Compaction
opts.num_levels = 6
opts.level0_file_num_compaction_trigger = 4

# Bloom filters
opts.filter_policy = rocksdb.BloomFilterPolicy(10)  # 10 bits/key

# Compression
opts.compression = rocksdb.CompressionType.lz4_compression
```

**Write Amplification Strategies:**

```
Problem: 30× write amplification → SSD wears out in 33 days

Solutions:
1. Increase SSTable size (64 MB → 256 MB): 30× → 20× ✓
2. Reduce levels (7 → 4): 30× → 15× ✓
3. Universal compaction: 30× → 5× ✓
4. Over-provisioned SSDs: 1 TB → 2 TB (2× lifespan)
```

**Real-World Deployments:**

```
Facebook (RocksDB):
- Scale: 1 billion writes/second
- Data: 100 petabytes
- Write amp: 20-30×
- Read latency: p99 < 10ms

LinkedIn (Kafka):
- Scale: 10 trillion messages/day
- Write amp: 1× (append-only)
- Write throughput: 7 million/second

Cassandra:
- Compaction: Size-tiered (default)
- Write amp: 5-10×
- Use case: Time-series, write-heavy
```

**Common Pitfalls:**

```
1. Not monitoring write amplification:
   Problem: SSD wears out unexpectedly
   Fix: Monitor rocksdb.stats, track cumulative writes

2. Default memtable size (64 MB) for write-heavy:
   Problem: Frequent flushes (high I/O)
   Fix: Increase to 128-256 MB

3. No bloom filters:
   Problem: Read performance 10× slower (check all levels)
   Fix: Enable 10 bits/key bloom filter

4. Using LSM for read-heavy workload:
   Problem: 5× slower reads than B-Tree
   Fix: Use B-Tree (InnoDB, PostgreSQL) for read-heavy
```

**The One Thing to Remember:** "LSM-Trees trade write amplification (10-50×) for write throughput (10× better than B-Trees). This makes them ideal for write-heavy workloads (Facebook's 1 billion writes/sec) but problematic for SSD lifespan (30× amplification reduces endurance from 1000 TB to 33 TB). The key decision: LSM for write-optimized (time-series, event streams), B-Tree for read-optimized (OLTP, analytics). No one-size-fits-all—Netflix uses both (Cassandra for writes, MySQL for reads)."

---

**Next:** [05_BTree_vs_LSM.md](05_BTree_vs_LSM.md) - Architecture comparison and decision framework

