# 🗄️ Database Types Overview - Choosing the Right Tool for the Job

> The database landscape has evolved from relational-only to a rich ecosystem of specialized systems. Understanding different database types—relational, document, key-value, column-family, graph, time-series, and NewSQL—is essential for making informed architectural decisions.

---

## 📖 1. Concept Explanation

### The Database Landscape

```
Database Types (2026):

Relational (SQL)              Document              Key-Value
├─ PostgreSQL                 ├─ MongoDB            ├─ Redis
├─ MySQL                      ├─ Couchbase          ├─ Memcached
├─ Oracle                     ├─ RavenDB            ├─ DynamoDB
└─ SQL Server                 └─ DocumentDB         └─ Riak

Column-Family                 Graph                  Time-Series
├─ Cassandra                  ├─ Neo4j              ├─ InfluxDB
├─ HBase                      ├─ ArangoDB           ├─ TimescaleDB
├─ ScyllaDB                   ├─ JanusGraph         ├─ Prometheus
└─ BigTable                   └─ Amazon Neptune     └─ OpenTSDB

NewSQL                        Vector                 Multi-Model
├─ CockroachDB                ├─ Pinecone           ├─ ArangoDB
├─ TiDB                       ├─ Weaviate           ├─ CosmosDB
├─ Google Spanner             ├─ Milvus             ├─ FoundationDB
└─ YugabyteDB                 └─ Qdrant             └─ OrientDB
```

---

### Why Multiple Database Types Exist

**One Size Does NOT Fit All!**

```
Problem: Relational databases can't efficiently handle all workloads

Example: Social Network
├─ User profiles           → Document DB (flexible schema)
├─ Friend relationships    → Graph DB (traversal queries)
├─ Session cache           → Key-Value DB (fast lookups)
├─ Activity metrics        → Time-Series DB (metrics)
├─ Post feed               → Column-Family DB (wide rows)
└─ Recommendation engine   → Vector DB (similarity search)

Solution: Polyglot persistence (use right tool for each job)
```

---

### Core Characteristics

| Database Type     | Data Model             | Query Model    | Best For                         | Worst For                      |
| ----------------- | ---------------------- | -------------- | -------------------------------- | ------------------------------ |
| **Relational**    | Tables (rows, columns) | SQL            | Structured data, ACID            | Unstructured, horizontal scale |
| **Document**      | JSON-like documents    | Query API      | Semi-structured, flexible schema | Complex joins, transactions    |
| **Key-Value**     | Key → Value pairs      | Get/Set        | Caching, sessions                | Complex queries, relationships |
| **Column-Family** | Column families        | CQL            | Wide rows, write-heavy           | Ad-hoc queries, joins          |
| **Graph**         | Nodes & Edges          | Cypher/Gremlin | Relationships, traversal         | Aggregations, analytics        |
| **Time-Series**   | Time-stamped data      | PromQL/Flux    | Metrics, IoT                     | General purpose                |
| **NewSQL**        | Tables (SQL)           | SQL            | ACID + Scale                     | Latency-sensitive reads        |

---

## 🧠 2. Why It Matters in Real Systems

### Real Disaster: Uber (2016) - Wrong Database Choice

**Problem:** Used PostgreSQL (relational) for geospatial queries

**What Happened:**

```
Use Case: Find nearby drivers (lat/long queries)

PostgreSQL (Relational):
├─ Query: SELECT * FROM drivers
│          WHERE ST_DWithin(location, user_location, 5000)
│          ORDER BY ST_Distance(location, user_location) LIMIT 10;
│
├─ Performance: 500ms+ for hot areas (thousands of drivers)
├─ Index: B-Tree on lat/long (not optimal for geospatial)
└─ Result: Slow ride matching, frustrated users 💥

Root Cause: Wrong tool for the job!
- Relational DBs not optimized for geospatial queries
- B-Tree indexes inefficient for 2D range queries
```

**Solution:** Switched to specialized systems

```
New Architecture:
├─ Drivers' live locations → Redis (Key-Value with Geospatial index)
│  └─ GEORADIUS command: 10ms queries ✅
│
├─ Trip history → Cassandra (Column-Family for time-series data)
│  └─ Fast writes, efficient partitioning ✅
│
└─ User profiles → PostgreSQL (Relational for structured data)
   └─ ACID transactions for payments ✅

Result:
✅ 50x faster driver matching
✅ Handles 10M+ trips/day
✅ Right tool for each job
```

**Lesson:** Database choice has massive impact on performance and scale!

---

### Real Success: Netflix - Polyglot Persistence

**System:** Multiple databases for different workloads

**Architecture:**

```
Netflix Data Stack:

1. Cassandra (Column-Family):
   └─ Use: Viewing history, user activity
   └─ Why: Billions of writes/day, global replication
   └─ Scale: 2,500+ nodes, 420 TB data

2. MySQL (Relational):
   └─ Use: Billing, subscription management
   └─ Why: ACID transactions required
   └─ Scale: Hundreds of instances

3. DynamoDB (Key-Value):
   └─ Use: Session storage, user preferences
   └─ Why: Low latency, high throughput
   └─ Scale: Trillions of requests/month

4. S3 (Object Store):
   └─ Use: Video files, metadata
   └─ Why: Infinite scale, durability
   └─ Scale: Exabytes of data

5. ElasticSearch (Search Engine):
   └─ Use: Content search, recommendations
   └─ Why: Full-text search, analytics
   └─ Scale: Thousands of nodes

Benefits:
✅ Each workload uses optimal database
✅ No single point of failure
✅ Can scale each independently
✅ 200M+ subscribers globally
```

**Lesson:** World-class systems use multiple specialized databases!

---

### Real Disaster: SnapChat (2013) - Redis as Primary DB

**Problem:** Used Redis (Key-Value, in-memory) as primary database

**What Happened:**

```
Use Case: Store all user snaps

Design:
└─ Redis (In-Memory Key-Value)
   ├─ Fast: ✅ (microsecond latency)
   ├─ Durable: ❌ (in-memory, lost on crash)
   └─ Persistence: RDB snapshots (periodic)

Disaster:
1. AWS node failure during peak hours
2. Redis lost in-memory data (not yet persisted to disk)
3. Millions of snaps vanished 💥
4. Users complained: "My snaps disappeared!"

Impact:
- Reputation damage
- User trust lost
- PR nightmare
- Emergency migration to persistent storage
```

**Solution:** Moved to appropriate database

```
New Architecture:
├─ S3 (Object Storage) → Primary storage for snaps ✅
│  └─ Durable, persistent, replicated
│
└─ Redis (Cache) → Cache for recent snaps ✅
   └─ Fast access, but NOT primary storage

Lesson: In-memory DBs are caches, NOT primary storage!
```

---

## ⚙️ 3. Internal Working

### Relational Databases (RDBMS)

**Architecture:**

```
PostgreSQL Internal Structure:

Storage Layer:
├─ Data Files (heap files)
│  └─ Tables stored as fixed-size pages (8KB)
│
├─ Indexes (B-Tree, Hash, GiST, GIN)
│  └─ B-Tree: Balanced tree for range queries
│
└─ WAL (Write-Ahead Log)
   └─ Durability via sequential writes

Query Processing:
1. Parser: SQL → Parse tree
2. Planner: Generate execution plan (cost-based)
3. Executor: Execute plan (index scan, seq scan, joins)

Transaction Manager:
├─ MVCC (Multi-Version Concurrency Control)
│  └─ Readers don't block writers
│
├─ Lock Manager
│  └─ Row-level, table-level, advisory locks
│
└─ WAL + Checkpointing
   └─ Crash recovery
```

**Strengths:**

- ✅ ACID transactions
- ✅ Complex queries (joins, aggregations)
- ✅ Referential integrity (foreign keys)

**Weaknesses:**

- ❌ Hard to scale horizontally (sharding complex)
- ❌ Fixed schema (migrations expensive)
- ❌ Not optimal for unstructured data

---

### Document Databases (MongoDB)

**Architecture:**

```
MongoDB Internal Structure:

Storage Layer:
├─ Collections (like tables)
│  └─ Documents stored as BSON (binary JSON)
│
├─ Indexes (B-Tree, Text, Geospatial, TTL)
│  └─ Can index nested fields: {"user.address.city": 1}
│
└─ WiredTiger Storage Engine
   └─ Compression, snapshots, checkpoints

Sharding:
├─ Shard Key: Partition data (e.g., user_id)
│  └─ Each shard is a replica set
│
├─ Config Servers: Store metadata
│
└─ mongos Router: Query routing to correct shard

Replication:
└─ Replica Sets: Primary + Secondaries
   ├─ Writes → Primary
   ├─ Reads → Primary or Secondary (tunable)
   └─ Automatic failover (elect new primary)
```

**Strengths:**

- ✅ Flexible schema (no migrations)
- ✅ Nested documents (avoid joins)
- ✅ Horizontal scaling (sharding built-in)

**Weaknesses:**

- ❌ No multi-document ACID (before v4.0)
- ❌ Joins expensive (prefer embedding)
- ❌ Data duplication (denormalization)

---

### Key-Value Databases (Redis)

**Architecture:**

```
Redis Internal Structure:

In-Memory Storage:
├─ Hash Table: Key → Value mapping
│  └─ O(1) lookups, inserts, deletes
│
├─ Data Structures:
│  ├─ Strings (text, numbers)
│  ├─ Lists (linked lists)
│  ├─ Sets (unique values)
│  ├─ Sorted Sets (ordered by score)
│  ├─ Hashes (nested key-value)
│  └─ Streams (log-like data)
│
└─ Persistence (optional):
   ├─ RDB: Periodic snapshots (full dump)
   └─ AOF: Append-only log (every write)

Single-Threaded Event Loop:
└─> Atomic operations (no locks needed)
    └─> INCR, LPUSH, SADD, etc. are atomic

Replication:
└─ Master-Slave replication
   ├─ Asynchronous (eventual consistency)
   └─ Sentinel for automatic failover
```

**Strengths:**

- ✅ Extremely fast (microsecond latency)
- ✅ Rich data structures (lists, sets, sorted sets)
- ✅ Pub/Sub messaging built-in

**Weaknesses:**

- ❌ Limited by memory (expensive to scale)
- ❌ No complex queries (no filtering, no joins)
- ❌ Not durable by default (in-memory)

---

### Column-Family Databases (Cassandra)

**Architecture:**

```
Cassandra Internal Structure:

Data Model:
├─ Keyspace (database)
│  └─ Table (column family)
│     └─ Partition Key + Clustering Columns
│
├─ Storage: Wide Rows
│  └─ Each row can have different columns (sparse)

Example:
CREATE TABLE user_activity (
    user_id UUID,
    timestamp TIMESTAMP,
    event_type TEXT,
    data BLOB,
    PRIMARY KEY (user_id, timestamp)
);

Partition Key: user_id (determines which node stores data)
Clustering Column: timestamp (orders data within partition)

Distributed Architecture:
├─ No master (peer-to-peer)
│  └─ Any node can handle any request
│
├─ Consistent Hashing: Determine data placement
│  └─ Virtual nodes for even distribution
│
├─ Replication Factor: 3 (default)
│  └─ Data copied to 3 nodes
│
└─ Tunable Consistency:
   ├─ ONE: Fast, eventual consistency
   ├─ QUORUM: Balanced (2/3 nodes)
   └─ ALL: Strong consistency, slow

Write Path:
1. Write to commit log (sequential, durable)
2. Write to memtable (in-memory)
3. Periodic flush to SSTable (on disk)
4. Compaction: Merge SSTables (background)
```

**Strengths:**

- ✅ Massive write throughput (linear scale)
- ✅ No single point of failure (P2P)
- ✅ Tunable consistency (AP or CP)

**Weaknesses:**

- ❌ No joins (denormalized data required)
- ❌ Limited secondary indexes
- ❌ Compaction overhead (disk I/O)

---

### Graph Databases (Neo4j)

**Architecture:**

```
Neo4j Internal Structure:

Data Model:
├─ Nodes: Entities (User, Product, City)
│  └─ Properties: {name: "Alice", age: 30}
│
└─ Relationships: Connections between nodes
   ├─ Type: FRIENDS_WITH, PURCHASED, LOCATED_IN
   └─ Properties: {since: "2020-01-01", weight: 0.8}

Example:
(Alice:User {age: 30})-[:FRIENDS_WITH {since: "2020"}]->(Bob:User)

Storage:
├─ Node Store: Fixed-size records
│  └─ Direct pointers to relationships (fast traversal)
│
├─ Relationship Store: Doubly-linked list
│  └─ Source node ⇄ Relationship ⇄ Target node
│
└─ Property Store: Key-value pairs

Query (Cypher):
MATCH (alice:User {name: "Alice"})-[:FRIENDS_WITH*1..3]-(friend)
RETURN friend.name

└─> Finds friends up to 3 hops away (index-free adjacency)

Transactions:
└─ ACID compliant (single instance)
    └─ Causal Clustering: Eventual consistency (distributed)
```

**Strengths:**

- ✅ Traversal queries (friends-of-friends in milliseconds)
- ✅ Expressive query language (Cypher)
- ✅ Pattern matching (find complex relationships)

**Weaknesses:**

- ❌ Not ideal for aggregations (sum, avg)
- ❌ Harder to shard (relationships cross partitions)
- ❌ Smaller ecosystem than relational DBs

---

### Time-Series Databases (InfluxDB)

**Architecture:**

```
InfluxDB Internal Structure:

Data Model:
├─ Measurement: Like a table (e.g., "cpu_usage")
│
├─ Tags: Indexed metadata (e.g., {host: "server1", region: "us-west"})
│  └─> Used for filtering (WHERE host = 'server1')
│
├─ Fields: Actual values (e.g., {usage: 75.3, temp: 68.2})
│  └─> Not indexed (stored as values)
│
└─ Timestamp: Nanosecond precision

Example:
cpu_usage,host=server1,region=us-west usage=75.3,temp=68.2 1609459200000000000

Storage (TSM Engine):
├─ WAL (Write-Ahead Log): Durable writes
│
├─ Cache: In-memory for recent data
│
├─ TSM Files: Columnar storage on disk
│  └─ Compressed (delta encoding, Gorilla compression)
│  └─> Can achieve 100:1 compression ratio!
│
└─ Compaction: Merge small TSM files
   └─> Optimize for time-range queries

Query (Flux):
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r.host == "server1")
  |> aggregateWindow(every: 1m, fn: mean)

Retention Policies:
└─ Auto-delete old data (e.g., keep 30 days)
```

**Strengths:**

- ✅ Optimized for time-range queries
- ✅ Extreme compression (10x-100x)
- ✅ Continuous queries (real-time aggregation)

**Weaknesses:**

- ❌ No updates (immutable data)
- ❌ No joins (single table queries)
- ❌ Not general-purpose

---

### NewSQL Databases (CockroachDB)

**Architecture:**

```
CockroachDB Internal Structure:

Goal: ACID + Horizontal Scale (best of both worlds)

Distributed SQL:
├─ SQL Layer: Parse, optimize, execute SQL
│
├─ Transaction Layer: MVCC + distributed transactions
│  └─> Uses Raft consensus for consistency
│
├─ Distribution Layer: Shard data (range-based)
│  └─> Automatic rebalancing
│
└─ Replication Layer: 3 replicas per range
   └─> Raft consensus: Majority quorum

Consistency:
├─ Serializable isolation (strongest)
│  └─> Uses HLC (Hybrid Logical Clocks)
│
└─ Distributed transactions via 2PC (Two-Phase Commit)

Example:
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT,
    balance DECIMAL
);

-- Distributed transaction (spans multiple nodes):
BEGIN;
UPDATE users SET balance = balance - 100 WHERE id = 'alice';
UPDATE users SET balance = balance + 100 WHERE id = 'bob';
COMMIT;

└─> Guaranteed ACID across nodes ✅
```

**Strengths:**

- ✅ ACID transactions (like PostgreSQL)
- ✅ Horizontal scaling (like Cassandra)
- ✅ SQL interface (familiar)

**Weaknesses:**

- ❌ Higher latency than NoSQL (consensus overhead)
- ❌ More complex to tune
- ❌ Newer (smaller community vs PostgreSQL)

---

## ✅ 4. Best Practices

### 1. Choose Based on Access Patterns

```python
# ✅ Map use case to database type

# Use Case 1: User authentication
# Access Pattern: Lookup by email (exact match)
# Database: Key-Value (Redis) or Relational (PostgreSQL)
class AuthService:
    def login(self, email, password):
        # Redis (cached session):
        session = redis.get(f'session:{email}')
        if session:
            return session

        # PostgreSQL (persistent storage):
        user = db.query('SELECT * FROM users WHERE email = ?', email)
        if verify_password(user.password_hash, password):
            session = create_session(user)
            redis.set(f'session:{email}', session, ex=3600)  # 1-hour TTL
            return session

# Use Case 2: Social network (friends-of-friends)
# Access Pattern: Graph traversal (relationships)
# Database: Graph (Neo4j)
class SocialService:
    def find_mutual_friends(self, user_id1, user_id2):
        # Cypher query (Neo4j):
        query = """
        MATCH (user1:User {id: $user_id1})-[:FRIENDS_WITH]-(mutual)-[:FRIENDS_WITH]-(user2:User {id: $user_id2})
        RETURN mutual
        """
        return neo4j.run(query, user_id1=user_id1, user_id2=user_id2)

# Use Case 3: IoT sensor data (time-series metrics)
# Access Pattern: Range queries by time
# Database: Time-Series (InfluxDB)
class MetricsService:
    def get_sensor_data(self, sensor_id, start_time, end_time):
        # Flux query (InfluxDB):
        query = f"""
        from(bucket: "iot")
          |> range(start: {start_time}, stop: {end_time})
          |> filter(fn: (r) => r.sensor_id == "{sensor_id}")
        """
        return influxdb.query(query)

# Use Case 4: Product catalog (flexible schema, nested data)
# Access Pattern: Flexible queries, frequent schema changes
# Database: Document (MongoDB)
class ProductService:
    def get_product(self, product_id):
        # MongoDB query:
        product = mongo.products.find_one(
            {'product_id': product_id},
            {
                'name': 1,
                'price': 1,
                'attributes': 1,  # Flexible schema (different per product)
                'reviews': 1       # Embedded documents (no join)
            }
        )
        return product
```

---

### 2. Use Polyglot Persistence

```python
# ✅ Use multiple databases in same system
class EcommerceSystem:
    def __init__(self):
        self.postgres = PostgreSQLClient()    # Transactions
        self.mongo = MongoClient()             # Product catalog
        self.redis = RedisClient()             # Cache
        self.elasticsearch = ESClient()        # Search
        self.influxdb = InfluxDBClient()       # Analytics

    def place_order(self, user_id, product_ids):
        # 1. Check product availability (MongoDB)
        products = self.mongo.products.find({'_id': {'$in': product_ids}})

        # 2. Create order with transaction (PostgreSQL)
        with self.postgres.transaction():
            order_id = self.postgres.execute(
                'INSERT INTO orders (user_id, total) VALUES (?, ?) RETURNING id',
                user_id, sum(p['price'] for p in products)
            )

            for product in products:
                self.postgres.execute(
                    'INSERT INTO order_items (order_id, product_id, price) VALUES (?, ?, ?)',
                    order_id, product['_id'], product['price']
                )

        # 3. Invalidate cache (Redis)
        self.redis.delete(f'user:{user_id}:orders')

        # 4. Index for search (Elasticsearch)
        self.elasticsearch.index(
            index='orders',
            id=order_id,
            body={'user_id': user_id, 'products': product_ids}
        )

        # 5. Track metrics (InfluxDB)
        self.influxdb.write_points([{
            'measurement': 'orders',
            'tags': {'user_id': user_id},
            'fields': {'total': sum(p['price'] for p in products)}
        }])

        return order_id

# Benefits:
# ✅ Each database optimized for its workload
# ✅ Can scale each independently
# ✅ No single point of failure
```

---

### 3. Implement Data Consistency Patterns

```python
# ✅ Eventual consistency between databases

# Saga Pattern (orchestrated transactions across databases)
class OrderSaga:
    def execute(self, order_data):
        try:
            # Step 1: Reserve inventory (PostgreSQL)
            inventory_reserved = self.inventory_service.reserve(order_data['product_ids'])

            # Step 2: Process payment (External API)
            payment_processed = self.payment_service.charge(order_data['amount'])

            # Step 3: Create order (MongoDB)
            order_created = self.order_service.create(order_data)

            # Step 4: Update cache (Redis)
            self.cache_service.invalidate(order_data['user_id'])

            return order_created

        except Exception as e:
            # Compensating transactions (rollback)
            if inventory_reserved:
                self.inventory_service.release(order_data['product_ids'])

            if payment_processed:
                self.payment_service.refund(order_data['amount'])

            raise

# Event Sourcing (propagate changes via events)
class EventDrivenSystem:
    def place_order(self, order_data):
        # 1. Write to primary database (PostgreSQL)
        order_id = self.postgres.create_order(order_data)

        # 2. Publish event (Kafka)
        event = {
            'event_type': 'OrderPlaced',
            'order_id': order_id,
            'user_id': order_data['user_id'],
            'timestamp': datetime.utcnow()
        }
        self.kafka.publish('orders', event)

        # 3. Consumers update other databases (eventual consistency)
        # - MongoDB consumer: Update product inventory
        # - Redis consumer: Invalidate cache
        # - Elasticsearch consumer: Index for search
        # - InfluxDB consumer: Record metrics
```

---

### 4. Monitor Database Performance

```python
# ✅ Track metrics for each database

class DatabaseMonitoring:
    def monitor_all_databases(self):
        metrics = {}

        # PostgreSQL metrics
        metrics['postgres'] = {
            'query_latency': self.postgres_latency(),
            'connection_count': self.postgres_connections(),
            'cache_hit_ratio': self.postgres_cache_hit_ratio(),
            'transaction_rate': self.postgres_tps()
        }

        # MongoDB metrics
        metrics['mongo'] = {
            'query_latency': self.mongo_latency(),
            'document_scans': self.mongo_doc_scans(),
            'index_usage': self.mongo_index_usage(),
            'replication_lag': self.mongo_replication_lag()
        }

        # Redis metrics
        metrics['redis'] = {
            'hit_rate': self.redis_hit_rate(),
            'memory_usage': self.redis_memory(),
            'eviction_count': self.redis_evictions(),
            'command_rate': self.redis_ops_per_sec()
        }

        # Cassandra metrics
        metrics['cassandra'] = {
            'write_latency': self.cassandra_write_latency(),
            'read_latency': self.cassandra_read_latency(),
            'compaction_queue': self.cassandra_compaction(),
            'pending_tasks': self.cassandra_pending()
        }

        return metrics
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Using Single Database for Everything

```python
# ❌ Bad: Force-fitting PostgreSQL for all use cases
class BadArchitecture:
    def __init__(self):
        self.postgres = PostgreSQLClient()  # Only database

    def cache_session(self, user_id, session_data):
        # Using PostgreSQL as cache (inefficient!)
        self.postgres.execute(
            'INSERT INTO sessions (user_id, data, expires_at) VALUES (?, ?, ?)',
            user_id, session_data, datetime.now() + timedelta(hours=1)
        )
        # Problem: PostgreSQL not optimized for high-throughput writes
        # Problem: No automatic expiration (must manually delete)
        # Problem: Overkill (ACID not needed for sessions)

    def search_products(self, query):
        # Using PostgreSQL for full-text search (inefficient!)
        results = self.postgres.execute(
            'SELECT * FROM products WHERE name ILIKE ?',
            f'%{query}%'
        )
        # Problem: Slow for fuzzy search, typos, relevance ranking
        # Problem: Can't search nested JSON fields easily

# ✅ Good: Use specialized databases
class GoodArchitecture:
    def __init__(self):
        self.postgres = PostgreSQLClient()       # Transactional data
        self.redis = RedisClient()               # Cache
        self.elasticsearch = ESClient()          # Search

    def cache_session(self, user_id, session_data):
        # Redis optimized for caching:
        self.redis.setex(
            f'session:{user_id}',
            3600,  # Auto-expires in 1 hour
            session_data
        )
        # ✅ 100x faster than PostgreSQL
        # ✅ Automatic expiration (TTL)

    def search_products(self, query):
        # Elasticsearch optimized for search:
        results = self.elasticsearch.search(
            index='products',
            body={
                'query': {
                    'multi_match': {
                        'query': query,
                        'fields': ['name^2', 'description', 'tags'],
                        'fuzziness': 'AUTO'
                    }
                }
            }
        )
        # ✅ Fast fuzzy search
        # ✅ Relevance ranking
        # ✅ Typo tolerance
```

---

### Mistake 2: Not Understanding Database Trade-offs

```python
# ❌ Bad: Choosing database based on hype
"""
"Everyone uses MongoDB, so we'll use it too!"

Problem: MongoDB not ideal for:
- Financial transactions (ACID required)
- Complex joins (denormalization required)
- Strong consistency (eventual by default)
"""

# ✅ Good: Choose based on requirements
class DatabaseSelection:
    def choose_database(self, requirements):
        # Requirement: ACID transactions, strong consistency
        if requirements['acid'] and requirements['consistency'] == 'strong':
            return 'PostgreSQL'  # Relational database

        # Requirement: Horizontal scale, high write throughput
        elif requirements['write_heavy'] and requirements['scale'] == 'horizontal':
            return 'Cassandra'  # Column-family database

        # Requirement: Low latency, high throughput, simple queries
        elif requirements['latency'] < 10 and requirements['query_complexity'] == 'simple':
            return 'Redis'  # Key-value database

        # Requirement: Relationship traversal
        elif requirements['query_type'] == 'graph_traversal':
            return 'Neo4j'  # Graph database

        # Requirement: Time-series data, analytics
        elif requirements['data_type'] == 'time_series':
            return 'InfluxDB'  # Time-series database

        else:
            return 'PostgreSQL'  # Default to relational (safest choice)
```

---

### Mistake 3: Ignoring Database Limits

```python
# ❌ Bad: Exceeding database design limits
class BadDesign:
    def store_large_blob(self, blob_data):
        # Storing 100MB file in PostgreSQL:
        self.postgres.execute(
            'INSERT INTO files (data) VALUES (?)',
            blob_data  # 100MB BYTEA column
        )
        # Problems:
        # - PostgreSQL row limit: 1GB (but slow for large blobs)
        # - TOAST overhead (chunking large values)
        # - Backup/restore slow
        # - Memory pressure

# ✅ Good: Use object storage for large files
class GoodDesign:
    def store_large_blob(self, blob_data):
        # 1. Upload to S3 (object storage)
        blob_key = self.s3.upload(blob_data)

        # 2. Store reference in PostgreSQL
        self.postgres.execute(
            'INSERT INTO files (s3_key, size, uploaded_at) VALUES (?, ?, ?)',
            blob_key, len(blob_data), datetime.utcnow()
        )

        # Benefits:
        # ✅ S3 optimized for large files
        # ✅ PostgreSQL stores metadata only (small rows)
        # ✅ Can serve files via CDN
```

---

### Mistake 4: Not Planning for Growth

```python
# ❌ Bad: No sharding strategy
class UnscalableDesign:
    def create_user(self, user_data):
        # Single PostgreSQL instance:
        self.postgres.execute(
            'INSERT INTO users (name, email) VALUES (?, ?)',
            user_data['name'], user_data['email']
        )
        # Problem: Hits single-node limit (vertical scaling expensive)

# ✅ Good: Design for horizontal scale
class ScalableDesign:
    def create_user(self, user_data):
        # Sharded architecture (Cassandra):
        user_id = uuid.uuid4()

        self.cassandra.execute(
            '''
            INSERT INTO users (user_id, name, email)
            VALUES (?, ?, ?)
            ''',
            user_id, user_data['name'], user_data['email']
        )

        # Benefits:
        # ✅ Horizontal scaling (add more nodes)
        # ✅ No single point of failure
        # ✅ Linear scale (2x nodes = 2x throughput)

        # OR: Use NewSQL for ACID + scale
        self.cockroachdb.execute(
            'INSERT INTO users (name, email) VALUES (?, ?)',
            user_data['name'], user_data['email']
        )
        # Benefits:
        # ✅ ACID transactions (like PostgreSQL)
        # ✅ Horizontal scaling (like Cassandra)
```

---

## 🔐 6. Security Considerations

### 1. Database-Specific Security Features

```python
# ✅ Leverage database security features

# PostgreSQL: Row-Level Security (RLS)
class PostgreSQLSecurity:
    def setup_rls(self):
        # Enable RLS for multi-tenant isolation:
        self.postgres.execute('''
            ALTER TABLE users ENABLE ROW LEVEL SECURITY;

            CREATE POLICY tenant_isolation ON users
            USING (tenant_id = current_setting('app.tenant_id')::UUID);
        ''')

        # Set tenant context:
        self.postgres.execute("SET app.tenant_id = 'tenant-123'")

        # Now queries automatically filtered:
        users = self.postgres.execute('SELECT * FROM users')
        # Only returns users for tenant-123 ✅

# MongoDB: Field-Level Encryption
class MongoDBSecurity:
    def setup_encryption(self):
        # Enable field-level encryption for PII:
        client = MongoClient(
            auto_encryption_opts=AutoEncryptionOpts(
                key_vault_namespace='encryption.__keyVault',
                kms_providers={'local': {'key': master_key}},
                schema_map={
                    'mydb.users': {
                        'bsonType': 'object',
                        'encryptMetadata': {
                            'keyId': [binary_key_id]
                        },
                        'properties': {
                            'ssn': {
                                'encrypt': {
                                    'bsonType': 'string',
                                    'algorithm': 'AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic'
                                }
                            }
                        }
                    }
                }
            )
        )
        # SSN automatically encrypted at rest and in transit ✅

# Redis: ACL (Access Control Lists)
class RedisSecurity:
    def setup_acl(self):
        # Create read-only user:
        self.redis.execute_command(
            'ACL SETUSER readonly on >readonly_password ~* -@all +@read'
        )

        # Create write-only user:
        self.redis.execute_command(
            'ACL SETUSER writeonly on >writeonly_password ~* -@all +@write'
        )
```

---

### 2. Encrypt Data Across Databases

```python
# ✅ Consistent encryption across different databases

class UnifiedEncryption:
    def __init__(self):
        self.cipher = Fernet(encryption_key)

    def store_sensitive_data(self, user_id, ssn):
        encrypted_ssn = self.cipher.encrypt(ssn.encode())

        # PostgreSQL:
        self.postgres.execute(
            'UPDATE users SET ssn = ? WHERE user_id = ?',
            encrypted_ssn, user_id
        )

        # MongoDB:
        self.mongo.users.update_one(
            {'user_id': user_id},
            {'$set': {'ssn': encrypted_ssn}}
        )

        # Redis (cache):
        self.redis.setex(
            f'user:{user_id}:ssn',
            3600,
            encrypted_ssn
        )
```

---

### 3. Audit Logging

```python
# ✅ Centralized audit logging for all databases

class AuditLogger:
    def log_database_access(self, database, operation, user, query):
        # Write to immutable audit log (time-series DB):
        self.influxdb.write_points([{
            'measurement': 'database_audit',
            'tags': {
                'database': database,
                'operation': operation,
                'user': user
            },
            'fields': {
                'query': query,
                'timestamp': datetime.utcnow().isoformat()
            }
        }])

    def query_database(self, database, query, user):
        # Log before executing:
        self.log_database_access(database, 'SELECT', user, query)

        # Execute query:
        if database == 'postgres':
            return self.postgres.execute(query)
        elif database == 'mongo':
            return self.mongo.execute(query)
```

---

## 🚀 7. Performance Optimization Techniques

### 1. Use Caching Layer

```python
# ✅ Redis as cache in front of primary databases

class CachedDataAccess:
    def get_user(self, user_id):
        # 1. Check Redis cache:
        cached_user = self.redis.get(f'user:{user_id}')
        if cached_user:
            return json.loads(cached_user)  # Cache hit ✅

        # 2. Cache miss, query PostgreSQL:
        user = self.postgres.execute(
            'SELECT * FROM users WHERE user_id = ?',
            user_id
        ).fetchone()

        # 3. Store in cache (5-minute TTL):
        self.redis.setex(
            f'user:{user_id}',
            300,
            json.dumps(user)
        )

        return user

# Cache invalidation:
def update_user(self, user_id, updates):
    # 1. Update PostgreSQL:
    self.postgres.execute(
        'UPDATE users SET name = ? WHERE user_id = ?',
        updates['name'], user_id
    )

    # 2. Invalidate cache:
    self.redis.delete(f'user:{user_id}')
```

---

### 2. Read Replicas for Scale

```python
# ✅ Route reads to replicas, writes to primary

class ReplicatedDataAccess:
    def __init__(self):
        self.postgres_primary = PostgreSQLClient(host='primary.db')
        self.postgres_replicas = [
            PostgreSQLClient(host='replica1.db'),
            PostgreSQLClient(host='replica2.db'),
            PostgreSQLClient(host='replica3.db')
        ]

    def write(self, query, *args):
        # Always write to primary:
        return self.postgres_primary.execute(query, *args)

    def read(self, query, *args):
        # Round-robin load balancing across replicas:
        replica = random.choice(self.postgres_replicas)
        return replica.execute(query, *args)

# Usage:
# Writes: Sent to primary
db.write('INSERT INTO users (name) VALUES (?)', 'Alice')

# Reads: Distributed across replicas
user = db.read('SELECT * FROM users WHERE user_id = ?', 123)
```

---

### 3. Denormalize Data Strategically

```python
# ✅ Denormalize in document/column databases for performance

# Normalized (Relational):
# - users table
# - orders table (foreign key to users)
# - Requires JOIN for user + orders

# Denormalized (Document DB):
{
    "_id": "user_123",
    "name": "Alice",
    "email": "alice@example.com",
    "orders": [  # Embedded (no join needed)
        {
            "order_id": "order_456",
            "total": 99.99,
            "items": ["product_1", "product_2"]
        },
        {
            "order_id": "order_789",
            "total": 149.99,
            "items": ["product_3"]
        }
    ]
}

# Benefits:
# ✅ Single query (no join)
# ✅ Fast reads
# ❌ Duplication (user name repeated in orders)
```

---

## 🧪 8. Examples

### Example 1: E-Commerce Platform (Polyglot Persistence)

```python
class EcommercePlatform:
    def __init__(self):
        self.postgres = PostgreSQLClient()      # Orders, payments (ACID)
        self.mongo = MongoClient()               # Product catalog (flexible schema)
        self.redis = RedisClient()               # Session cache, cart
        self.elasticsearch = ESClient()          # Product search
        self.cassandra = CassandraClient()       # User activity (time-series)
        self.neo4j = Neo4jClient()               # Recommendations (graph)

    def user_flow(self, user_id):
        # 1. Session management (Redis)
        session = self.redis.get(f'session:{user_id}')

        # 2. Browse products (MongoDB + Elasticsearch)
        products = self.elasticsearch.search(
            index='products',
            body={'query': {'match': {'category': 'electronics'}}}
        )

        # 3. Add to cart (Redis)
        self.redis.lpush(f'cart:{user_id}', product_id)

        # 4. Checkout (PostgreSQL transaction)
        with self.postgres.transaction():
            order_id = self.postgres.execute(
                'INSERT INTO orders (user_id, total) VALUES (?, ?) RETURNING id',
                user_id, total
            )

            # Payment processing...

        # 5. Track activity (Cassandra)
        self.cassandra.execute(
            '''
            INSERT INTO user_activity (user_id, event_type, timestamp)
            VALUES (?, ?, ?)
            ''',
            user_id, 'purchase', datetime.utcnow()
        )

        # 6. Update recommendations (Neo4j)
        self.neo4j.run(
            '''
            MATCH (user:User {id: $user_id}), (product:Product {id: $product_id})
            CREATE (user)-[:PURCHASED]->(product)
            ''',
            user_id=user_id, product_id=product_id
        )
```

---

### Example 2: Ride-Sharing App (Specialized Databases)

```python
class RideSharingApp:
    def __init__(self):
        self.postgres = PostgreSQLClient()       # User accounts, billing
        self.redis = RedisClient()               # Driver locations (geospatial)
        self.cassandra = CassandraClient()       # Trip history (time-series)
        self.influxdb = InfluxDBClient()         # Metrics (analytics)

    def find_nearby_drivers(self, user_lat, user_lon):
        # Redis geospatial queries:
        drivers = self.redis.georadius(
            'drivers:locations',
            user_lon, user_lat,
            5000,  # 5km radius
            unit='m',
            withdist=True,
            sort='ASC'
        )
        return drivers  # Sorted by distance ✅

    def update_driver_location(self, driver_id, lat, lon):
        # Redis geospatial update (fast):
        self.redis.geoadd('drivers:locations', lon, lat, driver_id)

    def record_trip(self, trip_data):
        # Cassandra (wide-column storage for time-series):
        self.cassandra.execute(
            '''
            INSERT INTO trips (driver_id, trip_id, start_time, end_time, distance, fare)
            VALUES (?, ?, ?, ?, ?, ?)
            ''',
            trip_data['driver_id'],
            trip_data['trip_id'],
            trip_data['start_time'],
            trip_data['end_time'],
            trip_data['distance'],
            trip_data['fare']
        )

    def track_metrics(self, metric_data):
        # InfluxDB (time-series metrics):
        self.influxdb.write_points([{
            'measurement': 'rides',
            'tags': {'city': metric_data['city']},
            'fields': {
                'completed_rides': metric_data['count'],
                'avg_wait_time': metric_data['avg_wait']
            }
        }])
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: LinkedIn - Multi-Database Architecture

**Requirements:**

- Social graph (connections)
- User profiles
- Activity feed
- Messaging
- Search

**Database Stack:**

```
LinkedIn Data Stack:

1. Graph Database (Custom - Voldemort):
   └─ Use: Social connections, network graph
   └─ Scale: 850M+ users, billions of connections

2. Document Database (Espresso):
   └─ Use: User profiles, company pages
   └─ Scale: Distributed across datacenters

3. Search Engine (Galene - built on Lucene):
   └─ Use: People search, job search
   └─ Scale: Billions of documents

4. Key-Value (Venice):
   └─ Use: Derived data, machine learning features
   └─ Scale: Read-optimized, globally replicated

5. Time-Series (Pinot):
   └─ Use: Analytics, who-viewed-your-profile
   └─ Scale: Real-time analytics on billions of events

6. Relational (MySQL):
   └─ Use: Transactions, billing
   └─ Scale: Sharded across thousands of instances

Benefits:
✅ Each database optimized for specific use case
✅ Independent scaling per workload
✅ Best-in-class performance
```

---

### Use Case 2: Airbnb - Event-Driven Polyglot Persistence

**Requirements:**

- Listing search
- Booking transactions
- Messaging
- Reviews
- Analytics

**Architecture:**

```
Airbnb Data Stack:

1. MySQL (Relational):
   └─ Use: Bookings, payments (ACID transactions)
   └─ Scale: Vitess (MySQL sharding)

2. Elasticsearch:
   └─ Use: Listing search, autocomplete
   └─ Scale: Thousands of nodes

3. Redis:
   └─ Use: Session storage, rate limiting
   └─ Scale: Redis Cluster

4. Amazon S3:
   └─ Use: Listing photos, user uploads
   └─ Scale: Petabytes

5. Druid:
   └─ Use: Real-time analytics dashboards
   └─ Scale: Billions of events/day

6. Kafka:
   └─ Use: Event streaming (CDC - Change Data Capture)
   └─ Scale: Connects all databases via events

Data Flow:
1. Booking created (MySQL) → Event published (Kafka)
2. Kafka consumers update:
   ├─ Elasticsearch (update availability)
   ├─ Redis (invalidate cache)
   └─ Druid (record metrics)

Benefits:
✅ Eventual consistency across databases
✅ Decoupled architecture
✅ Can replay events (rebuild any database)
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What database would you choose for a social media application?

**Answer:**

**I'd use polyglot persistence with multiple specialized databases:**

**1. User profiles → Document Database (MongoDB)**

- **Why:** Flexible schema (users have varying attributes)
- **Example:** Some users have "company", some don't; easy to add new fields
- **Trade-off:** No joins, must denormalize friend counts

**2. Social graph → Graph Database (Neo4j)**

- **Why:** Friend relationships, friend-of-friend queries
- **Example:** "Find mutual friends" in milliseconds
- **Trade-off:** Harder to shard (relationships cross partitions)

**3. Activity feed → Column-Family Database (Cassandra)**

- **Why:** Write-heavy (millions of posts/minute), time-series data
- **Example:** Each user has wide row with recent posts
- **Trade-off:** Denormalized (store author name with each post)

**4. Session management → Key-Value Database (Redis)**

- **Why:** Fast lookups, auto-expiration (TTL)
- **Example:** Session tokens with 1-hour expiration
- **Trade-off:** In-memory (expensive at scale)

**5. Payments → Relational Database (PostgreSQL)**

- **Why:** ACID transactions (money involved!)
- **Example:** Deduct balance, record transaction atomically
- **Trade-off:** Harder to scale horizontally

**Architecture:**

```
User Action: Post a status update

1. Write post (Cassandra):
   └─> Fanout to followers' feeds

2. Update graph (Neo4j):
   └─> User posted, increment post_count

3. Cache (Redis):
   └─> Store recent posts for fast access

4. Invalidate cache (Redis):
   └─> Delete cached profile data

5. Analytics (InfluxDB):
   └─> Track engagement metrics
```

**Key principle:** No single database handles all workloads well!

---

### Q2: Relational vs NoSQL - when to use each?

**Answer:**

**Use Relational (PostgreSQL, MySQL) when:**

1. **ACID transactions required**
   - Banking, payments, inventory
   - Example: Transfer money (atomic, consistent)

2. **Complex queries with joins**
   - Reporting, analytics
   - Example: "Total sales by region, product category, and month"

3. **Fixed schema (well-understood data)**
   - CRM, ERP systems
   - Example: Employee records (predictable structure)

4. **Strong consistency required**
   - Compliance, audit trails
   - Example: Medical records (must be accurate)

**Use NoSQL when:**

**Document DB (MongoDB) when:**

- Flexible/evolving schema
- Example: Product catalog (attributes differ per product type)
- Nested data (avoid joins)

**Key-Value (Redis) when:**

- Simple lookups by key
- Example: Session storage, caching
- Low latency (<1ms) required

**Column-Family (Cassandra) when:**

- Massive write throughput
- Example: IoT sensor data, time-series logs
- Horizontal scale (petabytes)

**Graph (Neo4j) when:**

- Relationship queries
- Example: Social network, fraud detection
- Traversal ("friends of friends")

**Comparison:**

| Requirement       | Relational    | NoSQL         |
| ----------------- | ------------- | ------------- |
| ACID Transactions | ✅ Strong     | ❌ Limited    |
| Horizontal Scale  | ❌ Hard       | ✅ Easy       |
| Complex Joins     | ✅ Fast       | ❌ Slow/None  |
| Flexible Schema   | ❌ Migrations | ✅ Schemaless |
| Consistency       | ✅ Strong     | ⚠️ Eventual   |

**Real-world example:**

```
E-commerce system:
├─ Orders, payments → PostgreSQL (ACID)
├─ Product catalog → MongoDB (flexible schema)
├─ Cart, session → Redis (fast lookups)
└─ Recommendations → Neo4j (graph)
```

**Key takeaway:** Use both! Modern systems combine relational and NoSQL (polyglot persistence).

---

### Q3: How do you handle consistency across multiple databases?

**Answer:**

**Challenge:** Maintaining data consistency when using multiple databases (polyglot persistence).

**Solutions:**

**1. Eventual Consistency (Most Common)**

```python
# Accept temporary inconsistency (eventual consistency)

# Example: Order placed
def place_order(order_data):
    # 1. Write to primary database (PostgreSQL):
    order_id = postgres.create_order(order_data)  # Source of truth

    # 2. Publish event (Kafka):
    kafka.publish('orders', {
        'event': 'OrderPlaced',
        'order_id': order_id,
        'user_id': order_data['user_id']
    })

    # 3. Consumers update other databases (async):
    # - MongoDB: Update product inventory
    # - Redis: Invalidate user cache
    # - Elasticsearch: Index order for search

    # Result: Eventually consistent (lag: 100ms - 1s)

# Trade-offs:
# ✅ Fast writes (don't wait for all databases)
# ✅ Independent failure (one DB down doesn't block others)
# ❌ Temporary inconsistency (acceptable for most use cases)
```

**2. Saga Pattern (Distributed Transactions)**

```python
# Orchestrated transactions across databases

class OrderSaga:
    def execute(self, order_data):
        steps = []

        try:
            # Step 1: Reserve inventory (PostgreSQL)
            self.inventory_service.reserve(order_data['product_ids'])
            steps.append('inventory_reserved')

            # Step 2: Process payment (External API)
            self.payment_service.charge(order_data['amount'])
            steps.append('payment_processed')

            # Step 3: Create order (MongoDB)
            order_id = self.order_service.create(order_data)
            steps.append('order_created')

            return order_id

        except Exception as e:
            # Rollback (compensating transactions):
            if 'order_created' in steps:
                self.order_service.delete(order_id)

            if 'payment_processed' in steps:
                self.payment_service.refund(order_data['amount'])

            if 'inventory_reserved' in steps:
                self.inventory_service.release(order_data['product_ids'])

            raise
```

**3. Event Sourcing (Rebuild from Events)**

```python
# Store events (immutable), derive state

# Events (append-only log):
events = [
    {'event': 'UserCreated', 'user_id': '123', 'name': 'Alice'},
    {'event': 'OrderPlaced', 'order_id': '456', 'user_id': '123'},
    {'event': 'PaymentProcessed', 'order_id': '456', 'amount': 99.99}
]

# Projections (derived from events):
# - PostgreSQL: Current orders
# - MongoDB: User profiles
# - Elasticsearch: Search index

# Benefits:
# ✅ Can rebuild any database from events
# ✅ Complete audit trail
# ✅ Time travel (query state at any point)
```

**4. Two-Phase Commit (2PC) - Avoid if Possible**

```python
# Distributed transaction (slow, blocking)

def transfer_money_2pc(from_account, to_account, amount):
    # Phase 1: Prepare
    db1.prepare('UPDATE accounts SET balance = balance - 100 WHERE id = ?', from_account)
    db2.prepare('UPDATE accounts SET balance = balance + 100 WHERE id = ?', to_account)

    # Phase 2: Commit (all or nothing)
    if db1.prepared and db2.prepared:
        db1.commit()
        db2.commit()
    else:
        db1.rollback()
        db2.rollback()

# Problems:
# ❌ Blocking (locks held during prepare)
# ❌ Single point of failure (coordinator)
# ❌ Slow (multiple round trips)

# When to use:
# Only when strong consistency absolutely required (rare)
```

**Recommendation:** Use eventual consistency (event-driven) for most use cases!

---

### Q4: What is polyglot persistence and when should you use it?

**Answer:**

**Polyglot Persistence:** Using multiple database types in a single application (each optimized for specific use case).

**When to use:**

1. **Different data models**
   - Example: User profiles (document), social graph (graph), metrics (time-series)

2. **Different performance requirements**
   - Example: Transactions need ACID (relational), analytics need speed (column-family)

3. **Different scale requirements**
   - Example: User data (moderate scale, relational), logs (massive scale, time-series)

**Real-world example: Netflix**

```
Netflix Polyglot Persistence:

1. Cassandra (Column-Family):
   └─ Viewing history (billions of writes/day)
   └─ Why: Write-heavy, time-series data

2. MySQL (Relational):
   └─ Billing, subscriptions
   └─ Why: ACID transactions required

3. DynamoDB (Key-Value):
   └─ Session storage
   └─ Why: Low latency, high throughput

4. S3 (Object Store):
   └─ Video files, metadata
   └─ Why: Infinite scale, durability

5. ElasticSearch (Search Engine):
   └─ Content search
   └─ Why: Full-text search, relevance ranking

Benefits:
✅ Each database optimized for workload
✅ Independent scaling
✅ No single point of failure

Challenges:
❌ Operational complexity (multiple systems)
❌ Data consistency (eventual consistency)
❌ Developer training (multiple query languages)
```

**When NOT to use:**

- Small applications (overhead not justified)
- Team lacks expertise (operational burden)
- Strong consistency required everywhere (distributed transactions hard)

**Best practices:**

1. **Start simple** (single database initially)
2. **Add databases as needed** (when bottlenecks proven)
3. **Use event-driven architecture** (decouple databases via events)
4. **Monitor all databases** (unified observability)

---

### Q5: How do you choose between CockroachDB, Cassandra, and PostgreSQL?

**Answer:**

| Database        | Type          | Consistency        | Scale      | When to Use                                           |
| --------------- | ------------- | ------------------ | ---------- | ----------------------------------------------------- |
| **PostgreSQL**  | Relational    | Strong (ACID)      | Vertical   | ACID + complex queries, single-region                 |
| **CockroachDB** | NewSQL        | Strong (ACID)      | Horizontal | ACID + horizontal scale, multi-region                 |
| **Cassandra**   | Column-Family | Eventual (tunable) | Horizontal | Massive writes, multi-region, eventual consistency OK |

**Decision tree:**

```
Do you need ACID transactions?
│
├─ YES:
│  │
│  ├─ Need horizontal scale (multi-region)?
│  │  │
│  │  ├─ YES: CockroachDB (NewSQL)
│  │  └─ NO: PostgreSQL (Relational)
│  │
│  └─ Can you tolerate higher latency?
│     ├─ YES: CockroachDB
│     └─ NO: PostgreSQL
│
└─ NO (Eventual consistency acceptable):
   │
   └─ Write-heavy workload (>100k writes/sec)?
      │
      ├─ YES: Cassandra (Column-Family)
      └─ NO: Consider PostgreSQL or CockroachDB
```

**Real-world examples:**

**Choose PostgreSQL:**

- Use case: E-commerce checkout (ACID required, single-region)
- Scale: <10k transactions/second
- Why: Mature, rich features, excellent tooling

**Choose CockroachDB:**

- Use case: Global banking system (ACID + multi-region)
- Scale: 10k-100k transactions/second, globally distributed
- Why: ACID guarantees + horizontal scale

**Choose Cassandra:**

- Use case: IoT sensor data (billions of writes/day)
- Scale: >100k writes/second, eventually consistent
- Why: Linear scalability, tunable consistency

**Key insight:** No single best database—choose based on requirements!

---

## 🧩 11. Design Patterns

### Pattern 1: Cache-Aside Pattern (Redis + Database)

**Problem:** Database queries too slow, need caching layer

**Solution:**

```python
class CacheAsidePattern:
    def get(self, key):
        # 1. Check cache first:
        value = redis.get(key)
        if value:
            return value  # Cache hit ✅

        # 2. Cache miss, query database:
        value = database.query(key)

        # 3. Populate cache:
        redis.setex(key, 300, value)  # 5-minute TTL

        return value

    def set(self, key, value):
        # 1. Update database:
        database.update(key, value)

        # 2. Invalidate cache:
        redis.delete(key)
```

---

### Pattern 2: CQRS (Command Query Responsibility Segregation)

**Problem:** Different databases optimized for reads vs writes

**Solution:**

```python
class CQRS:
    def __init__(self):
        self.write_db = PostgreSQLClient()  # Normalized, transactional
        self.read_db = MongoDBClient()       # Denormalized, fast reads

    # Commands (writes):
    def create_order(self, order_data):
        # Write to PostgreSQL (ACID):
        order_id = self.write_db.execute(
            'INSERT INTO orders (user_id, total) VALUES (?, ?) RETURNING id',
            order_data['user_id'], order_data['total']
        )

        # Publish event:
        kafka.publish('orders', {'event': 'OrderCreated', 'order_id': order_id})

        return order_id

    # Queries (reads):
    def get_order_details(self, order_id):
        # Read from MongoDB (denormalized):
        order = self.read_db.find_one(
            {'order_id': order_id},
            {
                'order_id': 1,
                'user_name': 1,      # Denormalized
                'items': 1,           # Denormalized
                'total': 1
            }
        )
        return order

# Event handler updates read database:
def on_order_created(event):
    order_id = event['order_id']

    # Fetch data from write DB:
    order = write_db.query(order_id)
    user = write_db.query(order['user_id'])

    # Denormalize and store in read DB:
    read_db.insert({
        'order_id': order_id,
        'user_name': user['name'],  # Denormalized
        'items': order['items'],
        'total': order['total']
    })
```

---

### Pattern 3: Data Lake Pattern (Multi-Database Analytics)

**Problem:** Need to analyze data across multiple databases

**Solution:**

```python
# Centralized data warehouse/lake for analytics

Architecture:
├─ Operational Databases:
│  ├─ PostgreSQL (orders, users)
│  ├─ MongoDB (products)
│  └─ Cassandra (activity logs)
│
├─ Change Data Capture (CDC):
│  └─ Stream changes to data lake via Kafka
│
└─ Data Lake (S3 + Redshift/Snowflake):
   └─ Unified analytics across all databases

# ETL Pipeline:
def sync_to_data_lake():
    # 1. Extract from operational databases:
    orders = postgres.query('SELECT * FROM orders WHERE updated_at > ?', last_sync)
    products = mongo.find({'updated_at': {'$gt': last_sync}})
    activity = cassandra.query('SELECT * FROM activity WHERE time > ?', last_sync)

    # 2. Transform (denormalize, clean):
    transformed = transform(orders, products, activity)

    # 3. Load into data warehouse:
    redshift.bulk_insert(transformed)

# Analytics queries (Redshift/Snowflake):
SELECT
    u.name,
    COUNT(o.order_id) as order_count,
    SUM(o.total) as revenue
FROM orders o
JOIN users u ON o.user_id = u.user_id
GROUP BY u.name
ORDER BY revenue DESC;
```

---

## 📚 Summary

**Database Types:**

- ✅ **Relational:** PostgreSQL, MySQL (ACID, complex queries)
- ✅ **Document:** MongoDB, Couchbase (flexible schema, nested data)
- ✅ **Key-Value:** Redis, DynamoDB (fast lookups, caching)
- ✅ **Column-Family:** Cassandra, HBase (massive writes, time-series)
- ✅ **Graph:** Neo4j, ArangoDB (relationships, traversal)
- ✅ **Time-Series:** InfluxDB, TimescaleDB (metrics, IoT)
- ✅ **NewSQL:** CockroachDB, TiDB (ACID + horizontal scale)

**When to use each:**

- **ACID transactions:** Relational or NewSQL
- **Horizontal scale:** NoSQL or NewSQL
- **Flexible schema:** Document DB
- **Graph queries:** Graph DB
- **Time-series data:** Time-Series DB
- **Caching:** Key-Value DB

**Polyglot persistence:**

- ✅ Use multiple databases (each optimized for use case)
- ✅ Event-driven architecture (eventual consistency)
- ✅ Monitor all databases (unified observability)

**Key takeaway:** No single database fits all use cases—choose based on data model, consistency, scale, and performance requirements!
