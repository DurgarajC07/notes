# 🌐 BASE Properties - Eventually Consistent Systems

> BASE properties represent a different approach to distributed data systems, trading strong consistency for higher availability and partition tolerance. Understanding BASE is essential for designing scalable NoSQL systems and making CAP theorem trade-offs.

---

## 📖 1. Concept Explanation

### What is BASE?

**BASE** is an acronym representing three properties of eventually consistent distributed systems:

1. **Basically Available** - System remains operational despite failures
2. **Soft state** - State may change over time without input
3. **Eventual consistency** - System will become consistent eventually

**Contrast with ACID:**

```
ACID (Traditional RDBMS)          BASE (Distributed NoSQL)
────────────────────────          ────────────────────────
Strong consistency                Eventual consistency
Immediate updates visible         Updates propagate over time
Lower availability                Higher availability
Vertical scaling                  Horizontal scaling
Single-node coordination          Distributed coordination
```

**Why BASE exists:**

- Enable massive horizontal scale (thousands of nodes)
- Maintain availability during network partitions
- Serve global users with low latency (geo-distribution)
- Handle billions of operations per day

---

### The Three Pillars

```
BASE Properties
│
├── Basically Available ────> System responds despite failures
│                              └─> Partial functionality OK
│
├── Soft State ────────────> State changes without input
│                              └─> Replicas may diverge temporarily
│
└── Eventual Consistency ──> All replicas converge eventually
                               └─> Given enough time, consistent
```

---

## 🧠 2. Why It Matters in Real Systems

### Without BASE - Scale Limits

**Problem: Coordinating 1000 nodes for strong consistency**

```
Operation: Update user's like count
1. Lock all replicas (1000 nodes)          ⏳ Wait...
2. Write to all nodes                      ⏳ Wait...
3. Confirm all writes succeeded            ⏳ Wait...
4. Release locks                           ⏳ Wait...

Result:
- High latency (network round-trips × 1000)
- Single failure blocks entire operation
- Can't scale beyond certain point
```

---

### With BASE - Massive Scale

**Solution: Accept temporary inconsistency**

```
Operation: Update user's like count
1. Write to local replica                  ✅ Instant!
2. Return success to user                  ✅ <10ms
3. Asynchronously propagate to others      ⏳ Background
4. Eventually all replicas consistent      ✅ Converged

Result:
- Low latency (no coordination)
- High availability (failures don't block)
- Horizontal scalability (1000s of nodes)
- User sees stale data temporarily (acceptable)
```

---

### Real-World Impact

| System Type           | ACID Approach              | BASE Approach               |
| --------------------- | -------------------------- | --------------------------- |
| **Social media feed** | Can't scale to billions    | Facebook, Twitter scale     |
| **Product catalog**   | Slow, limited availability | Amazon-scale throughput     |
| **Analytics**         | Single-node bottleneck     | Real-time at petabyte scale |
| **Shopping cart**     | High latency               | Low latency globally        |
| **DNS**               | Single point of failure    | Globally distributed        |

**BASE enables:**

- ✅ 99.99% availability (40min/year downtime)
- ✅ Sub-100ms latency globally
- ✅ Billions of requests per day
- ✅ Petabyte-scale data

**Cost of BASE:**

- ❌ Temporary inconsistency (stale reads)
- ❌ Complex conflict resolution
- ❌ Harder to reason about
- ❌ Application-level consistency logic

---

## ⚙️ 3. Internal Working

### Basically Available - How It Works

**Definition:** System continues operating despite failures, though possibly degraded.

**Implementation:**

```
Normal Operation:
Client → Load Balancer → [Node 1] → Success ✅
                         [Node 2]
                         [Node 3]

Node 1 Fails:
Client → Load Balancer → [Node 1] ❌
                         [Node 2] → Success ✅ (fallback)
                         [Node 3]

All Nodes in DC1 Fail:
Client → Load Balancer → [DC1: All nodes down] ❌
                         [DC2: Node 1] → Success ✅
```

**Cassandra Example:**

```cql
-- Write with replication factor = 3
INSERT INTO users (id, name, email)
VALUES (123, 'Alice', 'alice@example.com');

-- Write succeeds if ANY node responds (consistency = ONE)
-- Other 2 replicas written asynchronously
-- System remains available even if 2/3 nodes are down! ✅
```

**DynamoDB Example:**

```python
# Read from nearest replica (low latency)
response = dynamodb.get_item(
    TableName='users',
    Key={'id': 123},
    ConsistentRead=False  # Eventually consistent (faster)
)

# System serves request even if:
# - Some replicas are down
# - Network partition exists
# - Replication lag is high
```

---

### Soft State - State Changes Without Input

**Definition:** System state may change due to eventual consistency, not just user input.

**Mechanism:**

```
Time: T0
User A writes: counter = 5 (to Node 1)
User B writes: counter = 7 (to Node 2)

Time: T1 (User reads from Node 1)
└─> Sees counter = 5 (their write)

Time: T2 (User reads from Node 1 again)
└─> Sees counter = 7 (Node 2's write propagated!)
    State changed WITHOUT user input!
```

**Redis Cluster Example:**

```bash
# Write to master
SET user:123:profile '{"name": "Alice"}'

# Async replication to replicas
# Replica 1: Eventually receives update (50ms)
# Replica 2: Eventually receives update (100ms)

# Read from replica (before replication)
GET user:123:profile  # Returns old value (soft state)

# Read again (after replication)
GET user:123:profile  # Returns new value (state changed!)
```

**MongoDB Replica Set:**

```javascript
// Write to primary with w:1 (don't wait for replicas)
db.users.updateOne(
  { _id: 123 },
  { $set: { email: "newemail@example.com" } },
  { writeConcern: { w: 1 } }, // Return immediately
);

// Read from secondary (may see old email temporarily)
db.users.find({ _id: 123 }).readPref("secondary");
// → Returns old email (soft state)

// Read again 100ms later
db.users.find({ _id: 123 }).readPref("secondary");
// → Returns new email (eventually consistent)
```

---

### Eventual Consistency - Convergence

**Definition:** Given no new updates, all replicas will eventually return the same value.

**Convergence Mechanisms:**

**1. Anti-Entropy (Background Sync):**

```
Periodic reconciliation process:
Node 1: {"user:123": "Alice", "user:456": "Bob"}
Node 2: {"user:123": "Alice", "user:789": "Carol"}

Anti-Entropy Process:
1. Nodes exchange Merkle tree hashes
2. Detect differences
3. Sync missing/divergent data

After Sync:
Node 1: {"user:123": "Alice", "user:456": "Bob", "user:789": "Carol"}
Node 2: {"user:123": "Alice", "user:456": "Bob", "user:789": "Carol"}
└─> Converged! ✅
```

**2. Read Repair:**

```
Client reads user:123 from all replicas:
Node 1: {"name": "Alice", "email": "old@example.com"}
Node 2: {"name": "Alice", "email": "new@example.com"}  ← Newer
Node 3: {"name": "Alice", "email": "old@example.com"}

Read Repair Process:
1. Detect inconsistency (timestamp/version vector)
2. Return latest value to client (new@example.com)
3. Async update stale replicas (Node 1, Node 3)

Next Read: All replicas return "new@example.com" ✅
```

**3. Hinted Handoff:**

```
Write Operation (Node 1 is down):
Client → Write(key=123, value=Alice)
         ↓
         Node 1: ❌ Down
         Node 2: ✅ Stores data + hint for Node 1
         Node 3: ✅ Stores data

When Node 1 Recovers:
Node 2 → Replays hint → Node 1: ✅ Receives data
Eventually consistent! ✅
```

**Cassandra Tunable Consistency:**

```cql
-- Write with QUORUM (wait for majority)
INSERT INTO users (id, name) VALUES (123, 'Alice')
USING CONSISTENCY QUORUM;

-- Read with QUORUM (wait for majority)
SELECT * FROM users WHERE id = 123
USING CONSISTENCY QUORUM;

-- Guarantees: If write QUORUM + read QUORUM > replication_factor
--             Then reads see latest writes (strong consistency)
```

---

## ✅ 4. Best Practices

### 1. Design for Idempotency

```javascript
// ❌ Bad: Non-idempotent operation
function incrementLikeCount(postId) {
  db.update({ id: postId }, { $inc: { likes: 1 } });
  // If replayed due to retry → likes incremented multiple times!
}

// ✅ Good: Idempotent with unique ID
function incrementLikeCount(postId, requestId) {
  db.update(
    { id: postId },
    {
      $addToSet: { likeIds: requestId }, // Set (no duplicates)
    },
  );
  // likeCount = likeIds.length (computed field)
  // Replay safe! ✅
}
```

---

### 2. Use Vector Clocks for Conflict Resolution

```javascript
// ✅ Track causality with vector clocks
{
  "key": "user:123:cart",
  "value": ["item1", "item2"],
  "vector_clock": {
    "node_A": 5,  // Node A has seen 5 updates
    "node_B": 3   // Node B has seen 3 updates
  }
}

// Concurrent conflicting updates:
Node A: cart = ["item1", "item2", "item3"], clock = {A:6, B:3}
Node B: cart = ["item1", "item4"], clock = {A:5, B:4}

// Conflict detected! (neither clock dominates)
// Resolution strategy: Union (application-defined)
Merged: cart = ["item1", "item2", "item3", "item4"]
        clock = {A:6, B:4}
```

---

### 3. Implement Application-Level Consistency Checks

```python
# ✅ Read-your-writes consistency
class ShoppingCart:
    def add_item(self, user_id, item_id):
        # 1. Write to database
        write_result = db.update(
            {'user_id': user_id},
            {'$push': {'items': item_id}}
        )

        # 2. Store version in session
        session[user_id]['cart_version'] = write_result.version

    def get_cart(self, user_id):
        # 3. Read from replica with minimum version
        required_version = session.get(user_id, {}).get('cart_version', 0)

        cart = db.find_one(
            {'user_id': user_id, 'version': {'$gte': required_version}}
        )

        # Guaranteed to see own writes! ✅
        return cart
```

---

### 4. Set Appropriate Timeouts

```javascript
// ✅ Handle slow replicas with timeouts
const readOptions = {
  timeout: 100, // 100ms timeout
  readPreference: "nearest",
  maxStalenessSeconds: 90, // Max 90s lag
};

try {
  const user = await db.users.findOne({ id: 123 }, readOptions);
} catch (TimeoutError) {
  // Fallback: Read from any replica (accept stale data)
  const userStale = await db.users.findOne(
    { id: 123 },
    { timeout: 500, readPreference: "secondaryPreferred" },
  );
}
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Assuming Immediate Consistency

```python
# ❌ Bad: Expecting immediate consistency
def transfer_money(from_user, to_user, amount):
    # Debit from_user (writes to Node A)
    accounts.update_one(
        {'user_id': from_user},
        {'$inc': {'balance': -amount}}
    )

    # Immediate read (routes to Node B - stale!)
    balance = accounts.find_one({'user_id': from_user})['balance']

    if balance < 0:
        # BUG! May not see the debit yet! 💥
        raise InsufficientFundsError()

# ✅ Good: Read from same node or use sticky sessions
def transfer_money(from_user, to_user, amount):
    with db.session(causal_consistency=True) as session:
        # All reads within session see all prior writes
        accounts.update_one(
            {'user_id': from_user},
            {'$inc': {'balance': -amount}},
            session=session
        )

        balance = accounts.find_one(
            {'user_id': from_user},
            session=session  # Reads own writes ✅
        )['balance']
```

---

### Mistake 2: Ignoring Conflict Resolution

```javascript
// ❌ Bad: Last-write-wins (data loss!)
// User A adds item1 to cart
cart.items = ["item1"];  // timestamp: 10:00:00

// User B adds item2 to cart (concurrent)
cart.items = ["item2"];  // timestamp: 10:00:01

// Conflict resolution: Last write wins
// Result: cart = ["item2"]
// item1 LOST! 💥

// ✅ Good: Merge conflicts
{
  "cart": {
    "items": {
      "item1": { "added_by": "device_A", "timestamp": 10:00:00 },
      "item2": { "added_by": "device_B", "timestamp": 10:00:01 }
    }
  }
}
// Result: cart contains BOTH items ✅
```

---

### Mistake 3: Not Handling Replication Lag

```python
# ❌ Bad: Redirect to different server after write
@app.route('/post', methods=['POST'])
def create_post():
    post_id = db.posts.insert_one(request.json).inserted_id

    # Redirect to post page (may route to different server)
    return redirect(f'/posts/{post_id}')
    # 💥 Read from replica: "Post not found" (not replicated yet!)

# ✅ Good: Redirect with write confirmation or delay
@app.route('/post', methods=['POST'])
def create_post():
    post_id = db.posts.insert_one(
        request.json,
        write_concern={'w': 'majority'}  # Wait for majority
    ).inserted_id

    return redirect(f'/posts/{post_id}')  # Safe! ✅
```

---

### Mistake 4: Unbounded Eventual Consistency Window

```javascript
// ❌ Bad: No bounds on staleness
const user = await db.users.findOne(
  { id: 123 },
  { readPreference: "secondary" }, // Could be hours stale!
);

// ✅ Good: Bounded staleness
const user = await db.users.findOne(
  { id: 123 },
  {
    readPreference: "secondary",
    maxStalenessSeconds: 90, // Max 90 seconds stale
  },
);

// Even better: Use read concerns
const user = await db.users.findOne(
  { id: 123 },
  { readConcern: { level: "majority" } }, // Majority confirmed
);
```

---

## 🔐 6. Security Considerations

### 1. Eventual Consistency Security Gaps

```python
# ⚠️ Security Risk: Permission revocation delay
def revoke_admin_access(user_id):
    # Write to primary: Remove admin role
    db.users.update_one(
        {'id': user_id},
        {'$pull': {'roles': 'admin'}}
    )
    # ✅ Primary updated immediately

    # ⚠️ Problem: Replicas lag behind (up to 90 seconds)
    # User can still perform admin actions if request routes to stale replica!

# ✅ Mitigation: Force strong consistency for security operations
def revoke_admin_access(user_id):
    db.users.update_one(
        {'id': user_id},
        {'$pull': {'roles': 'admin'}},
        write_concern={'w': 'majority'},  # Wait for majority
        read_concern={'level': 'majority'}
    )

    # Invalidate all sessions immediately
    session_store.delete_all(user_id)
```

---

### 2. Audit Log Integrity

```python
# ✅ Use strongly consistent writes for audit logs
def log_security_event(event_type, user_id, details):
    audit_log.insert_one(
        {
            'event': event_type,
            'user_id': user_id,
            'timestamp': datetime.utcnow(),
            'details': details
        },
        write_concern={'w': 'majority', 'j': True}  # Journaled + majority
    )
    # Ensures audit trail is durable and consistent
```

---

### 3. Session Fixation with Stale Reads

```python
# ⚠️ Risk: Read stale session after logout
def logout(session_id):
    sessions.delete_one({'session_id': session_id})
    # Write to primary, but replicas lag

# Attacker immediately makes request to stale replica
# Session still exists on replica → Access granted! 💥

# ✅ Mitigation: Session tokens with version numbers
def logout(session_id):
    sessions.update_one(
        {'session_id': session_id},
        {'$set': {'revoked': True, 'version': version + 1}},
        write_concern={'w': 'majority'}
    )

    # All reads check version number
    # Even stale reads see "revoked: true"
```

---

## 🚀 7. Performance Optimization Techniques

### 1. Read From Nearest Replica

```javascript
// ✅ Geo-distributed low-latency reads
const readPreference = {
  mode: "nearest", // Lowest network latency
  tags: { region: user.region }, // Read from same region
};

const posts = await db.posts.find({ user_id: 123 }, { readPreference });

// Result:
// US user → reads from US replica (10ms)
// EU user → reads from EU replica (10ms)
// vs. all reading from US primary (200ms for EU users)
```

---

### 2. Async Writes for Non-Critical Data

```python
# ✅ Fire-and-forget for analytics events
def track_page_view(user_id, page_url):
    # Don't wait for write confirmation (fastest)
    analytics_events.insert_one(
        {
            'user_id': user_id,
            'page': page_url,
            'timestamp': datetime.utcnow()
        },
        write_concern={'w': 0}  # Unacknowledged write
    )
    # Returns immediately, no latency! ⚡

# ⚠️ Trade-off: Slight chance of data loss if node crashes
# Acceptable for non-critical analytics data
```

---

### 3. Batch Writes for Efficiency

```javascript
// ✅ Batch updates to reduce round-trips
const updates = users.map((user) => ({
  updateOne: {
    filter: { id: user.id },
    update: { $set: { last_active: new Date() } },
  },
}));

await db.users.bulkWrite(updates, {
  ordered: false, // Parallel execution
  writeConcern: { w: 1 },
});

// 1000 users updated in single round-trip vs. 1000 individual writes
```

---

### 4. Cache Invalidation Strategies

```python
# ✅ Handle cache staleness with TTL
@cache(ttl=60)  # Cache for 60 seconds
def get_user_profile(user_id):
    return db.users.find_one({'id': user_id})

# Alternative: Proactive cache invalidation
def update_user_profile(user_id, updates):
    db.users.update_one(
        {'id': user_id},
        {'$set': updates},
        write_concern={'w': 'majority'}
    )

    # Invalidate cache across all servers
    cache.delete(f'user:{user_id}')

    # Even better: Update cache with new value
    cache.set(f'user:{user_id}', updated_profile, ttl=300)
```

---

## 🧪 8. Examples

### Example 1: Social Media Like Counter

```javascript
// ✅ Eventually consistent like counter
// Optimized for write throughput over accuracy

// Write path: Fire-and-forget
async function likePost(userId, postId) {
  // 1. Increment counter (async, no wait)
  await redis.incr(`post:${postId}:likes`, {
    w: 0, // Don't wait for confirmation
  });

  // 2. Add to user's liked posts (async)
  await redis.sadd(`user:${userId}:liked`, postId);

  // Return immediately ⚡
  return { success: true };
}

// Read path: Accept staleness
async function getPostLikes(postId) {
  // Read from nearest replica (may be slightly stale)
  const likes = await redis.get(`post:${postId}:likes`, {
    readPreference: "nearest",
  });

  // Display: "1.2K likes" (approximate is fine)
  return formatNumber(likes);
}

// Reconciliation: Background job syncs to persistent store
setInterval(async () => {
  const postIds = await redis.keys("post:*:likes");

  for (const key of postIds) {
    const postId = key.match(/post:(\d+):likes/)[1];
    const count = await redis.get(key);

    // Batch update to database (less frequent, persistent)
    await db.posts.updateOne({ id: postId }, { $set: { like_count: count } });
  }
}, 60000); // Every minute
```

---

### Example 2: Shopping Cart with Conflict Resolution

```python
# ✅ Shopping cart with last-write-wins per item
class ShoppingCart:
    def add_item(self, user_id, item):
        """Add item with timestamp for conflict resolution"""
        cart_doc = {
            'user_id': user_id,
            f'items.{item.id}': {
                'product_id': item.id,
                'quantity': item.quantity,
                'price': item.price,
                'added_at': datetime.utcnow(),
                'device_id': request.device_id
            }
        }

        # Upsert with last-write-wins
        db.carts.update_one(
            {'user_id': user_id},
            {'$set': cart_doc},
            upsert=True,
            write_concern={'w': 1}  # Fast write
        )

    def get_cart(self, user_id):
        """Merge cart from all replicas"""
        # Read from multiple replicas
        carts = db.carts.find(
            {'user_id': user_id},
            read_preference=ReadPreference.SECONDARY_PREFERRED
        ).limit(3)

        # Merge strategy: Union of items, latest timestamp wins
        merged_items = {}

        for cart in carts:
            for item_id, item_data in cart.get('items', {}).items():
                if item_id not in merged_items:
                    merged_items[item_id] = item_data
                else:
                    # Keep item with latest timestamp
                    if item_data['added_at'] > merged_items[item_id]['added_at']:
                        merged_items[item_id] = item_data

        return {'user_id': user_id, 'items': merged_items}
```

---

### Example 3: DNS-Style Eventually Consistent System

```python
# ✅ DNS-like system with TTL-based consistency
class DNSCache:
    def update_record(self, domain, ip_address, ttl=300):
        """Update DNS record with TTL"""
        record = {
            'domain': domain,
            'ip': ip_address,
            'ttl': ttl,
            'updated_at': time.time(),
            'version': uuid.uuid4()
        }

        # Write to all replicas asynchronously
        for replica in replicas:
            replica.set(domain, record, fire_and_forget=True)

        return record['version']

    def resolve(self, domain):
        """Resolve domain with cache"""
        # 1. Check local cache
        cached = local_cache.get(domain)

        if cached and not self._is_expired(cached):
            return cached['ip']  # ⚡ Fast path

        # 2. Query nearest replica
        record = nearest_replica.get(domain)

        # 3. Update local cache
        local_cache.set(domain, record, ttl=record['ttl'])

        return record['ip']

    def _is_expired(self, record):
        age = time.time() - record['updated_at']
        return age > record['ttl']

# Characteristics:
# - Reads are fast (local cache)
# - Writes propagate asynchronously
# - Stale data is acceptable (bounded by TTL)
# - Scales to billions of queries/day ✅
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: Facebook News Feed

**Requirements:**

- Billions of posts/day
- Sub-second latency globally
- High availability (always accessible)
- Consistency not critical (stale feed OK)

**BASE Design:**

```python
class NewsFeed:
    def publish_post(self, user_id, content):
        """Write to local datacenter only"""
        post_id = generate_snowflake_id()

        # 1. Write to local datacenter (fast)
        local_db.posts.insert_one({
            'id': post_id,
            'user_id': user_id,
            'content': content,
            'timestamp': datetime.utcnow()
        }, write_concern={'w': 1})  # Local write only

        # 2. Async fan-out to followers
        celery.send_task('fanout_post', args=[post_id, user_id])

        # 3. Cross-datacenter replication (background)
        # No waiting! ⚡

        return post_id

    def get_feed(self, user_id):
        """Read from nearest replica"""
        # Read from local/nearest datacenter
        posts = local_db.feeds.find(
            {'user_id': user_id},
            limit=20
        ).sort('timestamp', -1)

        # May show posts out of order temporarily
        # May miss recent posts for 1-2 seconds
        # Acceptable trade-off! ✅

        return posts

# Results:
# - 500M+ users
# - <100ms latency worldwide
# - 99.99% availability
# - Eventual consistency (posts appear within seconds)
```

---

### Use Case 2: Amazon Product Catalog

**Challenge:** Display 500M products with high availability

**BASE Implementation:**

```javascript
// Write path: Update product info
async function updateProduct(productId, updates) {
  // 1. Write to primary
  await db.products.updateOne(
    { id: productId },
    { $set: updates },
    { writeConcern: { w: "majority" } }, // Wait for majority
  );

  // 2. Invalidate CDN cache (async)
  cdn.purge(`/products/${productId}`);

  // 3. Replication to read replicas (automatic, async)
  // US, EU, Asia replicas updated within 1-2 seconds
}

// Read path: Serve from nearest replica
async function getProduct(productId, userRegion) {
  // 1. Try CDN cache first (edge location)
  const cached = await cdn.get(`/products/${productId}`);
  if (cached) return cached; // ⚡ <10ms

  // 2. Read from regional replica
  const replica = selectReplica(userRegion);
  const product = await replica.products.findOne({ id: productId });

  // 3. Cache at CDN (TTL: 5 minutes)
  cdn.set(`/products/${productId}`, product, (ttl = 300));

  return product;
}

// Characteristics:
// - Product updates visible within seconds
// - 99.999% read availability
// - <50ms latency globally
// - Stale product info acceptable (prices updated eventually)
```

---

### Use Case 3: Distributed Session Store

**Requirements:**

- Multi-datacenter deployments
- Low-latency session reads
- Session consistency eventually

**Implementation:**

```python
class DistributedSessionStore:
    def create_session(self, user_id):
        """Create session in local datacenter"""
        session_id = generate_secure_token()
        session_data = {
            'id': session_id,
            'user_id': user_id,
            'created_at': datetime.utcnow(),
            'expires_at': datetime.utcnow() + timedelta(days=30),
            'version': 1
        }

        # Write to local Redis cluster
        redis_local.setex(
            f'session:{session_id}',
            ttl=30*24*3600,
            value=json.dumps(session_data)
        )

        # Async replicate to other datacenters
        for dc in other_datacenters:
            celery.send_task('replicate_session',
                args=[dc, session_id, session_data])

        return session_id

    def get_session(self, session_id):
        """Read from local datacenter"""
        # Try local first (fast)
        session = redis_local.get(f'session:{session_id}')

        if not session:
            # Fallback: Try other datacenters
            for dc in other_datacenters:
                session = redis[dc].get(f'session:{session_id}')
                if session:
                    # Copy to local for future reads
                    redis_local.setex(
                        f'session:{session_id}',
                        ttl=30*24*3600,
                        value=session
                    )
                    break

        return json.loads(session) if session else None

    def update_session(self, session_id, updates):
        """Update with version check"""
        session = self.get_session(session_id)

        if not session:
            raise SessionNotFoundError()

        # Optimistic locking with version
        session['version'] += 1
        session.update(updates)

        # Write to local
        redis_local.setex(
            f'session:{session_id}',
            ttl=30*24*3600,
            value=json.dumps(session)
        )

        # Async propagate to other DCs
        for dc in other_datacenters:
            celery.send_task('replicate_session',
                args=[dc, session_id, session])
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: Explain BASE properties with a real-world example.

**Answer:**

Consider Twitter's timeline:

**Basically Available:** If some servers fail, Twitter remains accessible. You can still read tweets, though maybe not the absolute latest ones.

**Soft State:** Your timeline updates without you refreshing. As new tweets propagate through Twitter's distributed system, your feed changes even though you didn't request an update.

**Eventual Consistency:** When you tweet, followers don't see it instantly. It appears in their timelines over 1-5 seconds as it propagates across datacenters. Eventually, all followers see it.

This design allows Twitter to:

- Serve 500M+ users globally
- Handle 6,000 tweets/second
- Maintain <100ms latency
- Stay available during outages

The trade-off: Tweets don't appear instantly everywhere, but users accept this for speed and reliability.

---

### Q2: How does eventual consistency differ from strong consistency?

**Answer:**

| Aspect              | Strong Consistency              | Eventual Consistency           |
| ------------------- | ------------------------------- | ------------------------------ |
| **Read guarantee**  | Always sees latest write        | May see stale data temporarily |
| **Write latency**   | High (wait for all replicas)    | Low (write locally)            |
| **Availability**    | Lower (nodes must coordinate)   | Higher (no coordination)       |
| **Implementation**  | 2PC, Paxos, Raft                | Async replication              |
| **Scalability**     | Limited (coordination overhead) | Unlimited (horizontal)         |
| **Example systems** | Bank accounts, inventory        | Social feeds, analytics        |

**Code example:**

```python
# Strong consistency (ACID)
with transaction():
    db.update({'id': 1}, {'balance': 500})
    # All replicas updated before returning
    # OR transaction aborts
    # Guarantee: Next read sees 500

# Eventual consistency (BASE)
db.update({'id': 1}, {'balance': 500}, w=1)
# Returns immediately after local write
# Read might see old balance for 1-2 seconds
# Eventually all replicas show 500
```

---

### Q3: What are the challenges of eventual consistency?

**Answer:**

**1. Read-your-writes inconsistency:**

```python
# User updates profile
db.users.update({'id': 123}, {'name': 'Alice'})

# Immediate redirect
redirect('/profile')

# Read from different replica
user = db.users.find_one({'id': 123})
# Still shows old name! 💥
```

**Solution:** Sticky sessions or read from primary after write.

**2. Concurrent conflict resolution:**

```python
# Two users update same document simultaneously
User A: cart.items = ['item1', 'item2']
User B: cart.items = ['item1', 'item3']

# Who wins? Need conflict resolution strategy!
```

**Solution:** CRDTs, vector clocks, application-specific merge logic.

**3. Unbounded lag:**

```python
# Replica can be arbitrarily stale
# Security risk: Permission revoked but still works for minutes
```

**Solution:** Bounded staleness, read concerns, monitoring.

**4. Debugging complexity:**

```python
# "Works on my machine" → different replica states
# Hard to reproduce bugs
# Need distributed tracing
```

---

### Q4: When should you use BASE vs ACID?

**Answer:**

**Use ACID when:**

- ✅ Financial transactions (money, payments, billing)
- ✅ Inventory systems (prevent overselling)
- ✅ Reservation systems (prevent double-booking)
- ✅ Regulatory compliance (audit trails)
- ✅ Data correctness > speed/availability

**Use BASE when:**

- ✅ Social media (feeds, likes, comments)
- ✅ Analytics (click tracking, metrics)
- ✅ Product catalogs (price updates can lag)
- ✅ Content delivery (news, videos)
- ✅ Speed/availability > immediate consistency

**Hybrid approach (common in practice):**

```python
class EcommerceSystem:
    # ACID for critical operations
    def place_order(self, order):
        with transaction():
            # Strong consistency required
            db.orders.insert(order)
            db.inventory.decrement(order.items)
            db.payments.charge(order.total)

    # BASE for non-critical operations
    def update_view_count(self, product_id):
        # Eventual consistency OK
        redis.incr(f'product:{product_id}:views', w=0)
```

---

### Q5: How do you test eventually consistent systems?

**Answer:**

**1. Chaos engineering:**

```python
# Inject network latency
with chaos.network_delay(100, 500):  # 100-500ms
    result = db.users.find_one({'id': 123})
    # Verify system handles slow replicas

# Inject replica lag
with chaos.replication_lag(10):  # 10 seconds
    # Verify stale reads are handled correctly
```

**2. Concurrent operation testing:**

```python
# Simulate concurrent writes from different clients
threads = []
for i in range(100):
    t = threading.Thread(target=lambda: db.update({'id': 1}, {'$inc': {'counter': 1}}))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

# Verify final state is consistent
# Check for lost updates, race conditions
```

**3. Time-travel testing:**

```python
# Simulate reading at different points in consistency window
write_time = time.time()
db.update({'id': 1}, {'value': 'new'})

# Read immediately (may be stale)
value_t0 = db.find_one({'id': 1})

time.sleep(1)  # Wait for propagation

# Read after 1 second (should be consistent)
value_t1 = db.find_one({'id': 1})

assert value_t1['value'] == 'new'
```

**4. Monitoring for divergence:**

```python
# Check replica consistency
def check_replica_consistency():
    values = []
    for replica in all_replicas:
        value = replica.find_one({'id': 1})
        values.append(value)

    if len(set(values)) > 1:
        alert('Replicas diverged!', values)
```

---

## 🧩 11. Design Patterns

### Pattern 1: CQRS (Command Query Responsibility Segregation)

**Problem:** Read and write workloads have different consistency requirements

**Solution:**

```python
# Write path: Strong consistency
class CommandService:
    def create_order(self, order_data):
        with transaction():
            order_id = db_primary.orders.insert(order_data)
            db_primary.inventory.decrement(order_data['items'])
            return order_id

# Read path: Eventual consistency (optimized replicas)
class QueryService:
    def get_order_history(self, user_id):
        # Read from denormalized, read-optimized replica
        return db_readonly.order_history.find({'user_id': user_id})

    def get_dashboard_stats(self):
        # Read from pre-aggregated materialized views
        return db_analytics.stats.find_one({'type': 'dashboard'})
```

**Pros:**

- ✅ Optimized read and write paths independently
- ✅ Scale reads and writes separately
- ✅ Different consistency models per use case

**Cons:**

- ❌ Increased complexity (two models)
- ❌ Synchronization overhead

---

### Pattern 2: Saga Pattern (Distributed Transactions)

**Problem:** ACID transactions across microservices are expensive/impossible

**Solution:**

```python
class OrderSaga:
    def execute(self, order_data):
        compensations = []

        try:
            # Step 1: Reserve inventory
            inventory_id = inventory_service.reserve(order_data['items'])
            compensations.append(lambda: inventory_service.release(inventory_id))

            # Step 2: Charge payment
            payment_id = payment_service.charge(order_data['total'])
            compensations.append(lambda: payment_service.refund(payment_id))

            # Step 3: Create shipment
            shipment_id = shipping_service.create(order_data)
            compensations.append(lambda: shipping_service.cancel(shipment_id))

            # All steps succeeded!
            return {'order_id': order_id, 'status': 'confirmed'}

        except Exception as e:
            # Rollback: Execute compensating transactions
            for compensate in reversed(compensations):
                try:
                    compensate()
                except Exception as rollback_error:
                    # Log and alert (manual intervention may be needed)
                    logger.error(f'Compensation failed: {rollback_error}')

            raise OrderFailedError('Order saga failed', original_error=e)
```

**Pros:**

- ✅ Enables distributed transactions
- ✅ Each service maintains local ACID
- ✅ Loosely coupled services

**Cons:**

- ❌ Eventual consistency (temporary inconsistent states)
- ❌ Compensations can fail (need monitoring)

---

### Pattern 3: CRDTs (Conflict-Free Replicated Data Types)

**Problem:** Merging concurrent updates without conflicts

**Solution:**

```python
class GCounter:
    """Grow-only counter (CRDT)"""
    def __init__(self, node_id):
        self.node_id = node_id
        self.counts = {}  # node_id → count

    def increment(self, amount=1):
        self.counts[self.node_id] = self.counts.get(self.node_id, 0) + amount

    def value(self):
        return sum(self.counts.values())

    def merge(self, other):
        """Merge with another replica (idempotent, commutative)"""
        for node_id, count in other.counts.items():
            self.counts[node_id] = max(
                self.counts.get(node_id, 0),
                count
            )

# Usage across replicas
node_a = GCounter('A')
node_a.increment(5)  # {A: 5}, value = 5

node_b = GCounter('B')
node_b.increment(3)  # {B: 3}, value = 3

# Merge replicas (order doesn't matter!)
node_a.merge(node_b)  # {A: 5, B: 3}, value = 8
node_b.merge(node_a)  # {A: 5, B: 3}, value = 8

# Conflict-free! ✅
```

**Pros:**

- ✅ Automatic conflict resolution
- ✅ Commutative and idempotent
- ✅ No coordination needed

**Cons:**

- ❌ Limited data types
- ❌ Can grow in size (tombstones)

---

## 📚 Summary

**BASE Properties:**

- ✅ **Basically Available:** System stays operational despite failures
- ✅ **Soft State:** State can change without input (replication)
- ✅ **Eventual Consistency:** Replicas converge over time

**Trade-offs:**

- ✅ High availability and partition tolerance
- ✅ Horizontal scalability to thousands of nodes
- ✅ Low latency globally
- ❌ Temporary inconsistency
- ❌ Complex conflict resolution
- ❌ Application-level consistency logic

**When to use BASE:**

- Social media platforms
- Content delivery networks
- Analytics systems
- Product catalogs
- Session stores

**When to use ACID:**

- Financial transactions
- Inventory systems
- Reservation systems
- Compliance-critical data

---

**Key Takeaway:** BASE is not "worse" than ACID—it's a different set of trade-offs optimized for availability and scale at the cost of immediate consistency. Choose based on your requirements, and remember many systems use both (hybrid approach).
