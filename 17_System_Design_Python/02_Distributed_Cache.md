# System Design: Distributed Cache

## 📖 Problem Statement

Design a distributed caching system like Redis/Memcached that:

- Stores key-value pairs in memory
- Distributes data across multiple cache servers
- Handles high throughput (millions of operations/second)
- Provides low latency (<1ms for cache hits)
- Supports cache eviction policies (LRU, LFU)
- Maintains consistency across replicas
- Handles node failures gracefully

## 🎯 Requirements

### Functional Requirements

1. PUT(key, value) - Store key-value pair
2. GET(key) - Retrieve value by key
3. DELETE(key) - Remove key-value pair
4. Support TTL (time-to-live) for auto-expiration
5. Support multiple data types (strings, lists, sets, hashes)
6. Support atomic operations
7. Persist data to disk (optional)

### Non-Functional Requirements

1. **Low Latency**: <1ms for cache hits, <5ms for misses
2. **High Throughput**: 100K+ operations/second per node
3. **Scalability**: Horizontal scaling (add more nodes)
4. **Availability**: 99.9% uptime
5. **Consistency**: Eventually consistent
6. **Partition Tolerance**: Continue working during network splits

## 📊 Capacity Estimation

### Memory Requirements

- **Cache size per node**: 64 GB RAM
- **Average key-value size**: 1 KB
- **Entries per node**: 64 GB / 1 KB = 64M entries
- **Total nodes**: 10 nodes = 640M total entries

### Throughput

- **Read operations**: 90% of traffic = 900K reads/second
- **Write operations**: 10% of traffic = 100K writes/second
- **Per node**: 100K operations/second

## 🏗️ Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│  Consistent Hash    │  ← Determines which node
│      Router         │
└──────────┬──────────┘
           │
    ┌──────┴───────┬────────┬────────┐
    ▼              ▼        ▼        ▼
┌────────┐    ┌────────┐ ┌────────┐ ┌────────┐
│ Node 1 │    │ Node 2 │ │ Node 3 │ │ Node N │
│(Master)│    │(Master)│ │(Master)│ │(Master)│
└───┬────┘    └───┬────┘ └───┬────┘ └───┬────┘
    │             │          │          │
    ▼             ▼          ▼          ▼
┌────────┐    ┌────────┐ ┌────────┐ ┌────────┐
│Replica │    │Replica │ │Replica │ │Replica │
└────────┘    └────────┘ └────────┘ └────────┘
```

## 🔑 Core Components

### 1. In-Memory Cache with LRU Eviction

```python
from collections import OrderedDict
from typing import Optional, Any
import time

class LRUCache:
    """
    LRU (Least Recently Used) Cache

    - Evicts least recently accessed items when full
    - O(1) get, put, delete operations
    - Uses OrderedDict for efficient LRU tracking
    """

    def __init__(self, capacity: int):
        """
        Args:
            capacity: Maximum number of entries
        """
        self.capacity = capacity
        self.cache = OrderedDict()
        self.ttl_map = {}  # key -> expiration_time

    def get(self, key: str) -> Optional[Any]:
        """
        Get value by key

        Returns:
            Value if found and not expired, None otherwise
        """
        # Check expiration
        if key in self.ttl_map:
            if time.time() > self.ttl_map[key]:
                # Expired
                self.delete(key)
                return None

        if key not in self.cache:
            return None

        # Move to end (mark as recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: str, value: Any, ttl: Optional[int] = None):
        """
        Store key-value pair

        Args:
            key: Cache key
            value: Value to store
            ttl: Time-to-live in seconds (optional)
        """
        # If key exists, move to end
        if key in self.cache:
            self.cache.move_to_end(key)

        # Add new entry
        self.cache[key] = value

        # Set TTL if provided
        if ttl:
            self.ttl_map[key] = time.time() + ttl
        elif key in self.ttl_map:
            del self.ttl_map[key]

        # Evict if over capacity
        if len(self.cache) > self.capacity:
            # Remove oldest (least recently used)
            oldest_key = next(iter(self.cache))
            self.delete(oldest_key)

    def delete(self, key: str) -> bool:
        """
        Delete key from cache

        Returns:
            True if deleted, False if not found
        """
        if key in self.cache:
            del self.cache[key]
            if key in self.ttl_map:
                del self.ttl_map[key]
            return True
        return False

    def clear_expired(self):
        """Clear all expired entries"""
        now = time.time()
        expired_keys = [
            key for key, expiry in self.ttl_map.items()
            if now > expiry
        ]

        for key in expired_keys:
            self.delete(key)

    def size(self) -> int:
        """Get current cache size"""
        return len(self.cache)

# Example usage
cache = LRUCache(capacity=3)

cache.put("a", 1)
cache.put("b", 2)
cache.put("c", 3)
print(cache.get("a"))  # 1 (moves "a" to end)

cache.put("d", 4)  # Evicts "b" (least recently used)
print(cache.get("b"))  # None (evicted)
print(cache.get("a"))  # 1 (still in cache)
```

### 2. Consistent Hashing

```python
import hashlib
from bisect import bisect_right
from typing import List, Optional

class ConsistentHash:
    """
    Consistent Hashing for distributed cache

    - Minimizes key redistribution when nodes added/removed
    - Uses virtual nodes for better distribution
    - O(log N) lookup time
    """

    def __init__(self, nodes: List[str], virtual_nodes: int = 150):
        """
        Args:
            nodes: List of cache node addresses
            virtual_nodes: Number of virtual nodes per physical node
        """
        self.virtual_nodes = virtual_nodes
        self.ring = {}  # hash -> node_address
        self.sorted_keys = []

        for node in nodes:
            self.add_node(node)

    def _hash(self, key: str) -> int:
        """Hash function (MD5)"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """Add node to hash ring"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)

            self.ring[hash_value] = node
            self.sorted_keys.append(hash_value)

        self.sorted_keys.sort()

    def remove_node(self, node: str):
        """Remove node from hash ring"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)

            if hash_value in self.ring:
                del self.ring[hash_value]
                self.sorted_keys.remove(hash_value)

    def get_node(self, key: str) -> Optional[str]:
        """
        Get node responsible for key

        Returns:
            Node address
        """
        if not self.ring:
            return None

        hash_value = self._hash(key)

        # Find first node clockwise from hash value
        index = bisect_right(self.sorted_keys, hash_value)

        if index == len(self.sorted_keys):
            index = 0

        return self.ring[self.sorted_keys[index]]

# Example usage
nodes = ["cache1:6379", "cache2:6379", "cache3:6379"]
ch = ConsistentHash(nodes)

# Route keys to nodes
keys = ["user:123", "post:456", "session:789"]
for key in keys:
    node = ch.get_node(key)
    print(f"{key} -> {node}")

# Add new node (minimal redistribution)
ch.add_node("cache4:6379")
print("\nAfter adding cache4:")
for key in keys:
    node = ch.get_node(key)
    print(f"{key} -> {node}")
```

### 3. Distributed Cache Client

```python
import redis
from typing import Any, Optional, List
import logging

logger = logging.getLogger(__name__)

class DistributedCache:
    """
    Distributed cache client

    - Routes keys to nodes using consistent hashing
    - Handles node failures
    - Supports replication
    """

    def __init__(self, nodes: List[str], replicas: int = 2):
        """
        Args:
            nodes: List of Redis node addresses (host:port)
            replicas: Number of replica nodes per key
        """
        self.nodes = {}
        self.replicas = replicas

        # Connect to all nodes
        for node in nodes:
            host, port = node.split(':')
            try:
                client = redis.Redis(
                    host=host,
                    port=int(port),
                    decode_responses=True,
                    socket_timeout=1,
                    socket_connect_timeout=1
                )
                # Test connection
                client.ping()
                self.nodes[node] = client
            except Exception as e:
                logger.error(f"Failed to connect to {node}: {e}")

        # Initialize consistent hash
        self.hash_ring = ConsistentHash(list(self.nodes.keys()))

    def get(self, key: str) -> Optional[Any]:
        """
        Get value from cache

        Returns:
            Value if found, None otherwise
        """
        node_address = self.hash_ring.get_node(key)
        if not node_address:
            return None

        try:
            client = self.nodes[node_address]
            return client.get(key)
        except Exception as e:
            logger.error(f"Failed to get {key} from {node_address}: {e}")

            # Try replica nodes
            return self._get_from_replicas(key, node_address)

    def _get_from_replicas(self, key: str, failed_node: str) -> Optional[Any]:
        """Try to get value from replica nodes"""
        for node_address, client in self.nodes.items():
            if node_address != failed_node:
                try:
                    value = client.get(key)
                    if value is not None:
                        return value
                except:
                    continue
        return None

    def put(self, key: str, value: Any, ttl: Optional[int] = None):
        """
        Store key-value pair

        Args:
            key: Cache key
            value: Value to store
            ttl: Time-to-live in seconds
        """
        node_address = self.hash_ring.get_node(key)
        if not node_address:
            raise Exception("No nodes available")

        try:
            client = self.nodes[node_address]

            if ttl:
                client.setex(key, ttl, value)
            else:
                client.set(key, value)

            # Replicate to other nodes
            self._replicate(key, value, ttl, node_address)

        except Exception as e:
            logger.error(f"Failed to put {key} to {node_address}: {e}")
            raise

    def _replicate(self, key: str, value: Any, ttl: Optional[int], primary_node: str):
        """Replicate data to replica nodes"""
        replica_count = 0

        for node_address, client in self.nodes.items():
            if node_address != primary_node and replica_count < self.replicas:
                try:
                    if ttl:
                        client.setex(key, ttl, value)
                    else:
                        client.set(key, value)
                    replica_count += 1
                except Exception as e:
                    logger.error(f"Failed to replicate {key} to {node_address}: {e}")

    def delete(self, key: str) -> bool:
        """
        Delete key from cache

        Returns:
            True if deleted, False otherwise
        """
        node_address = self.hash_ring.get_node(key)
        if not node_address:
            return False

        deleted = False

        # Delete from all nodes (primary + replicas)
        for node_address, client in self.nodes.items():
            try:
                if client.delete(key):
                    deleted = True
            except Exception as e:
                logger.error(f"Failed to delete {key} from {node_address}: {e}")

        return deleted

    def add_node(self, node_address: str):
        """Add new cache node"""
        host, port = node_address.split(':')

        try:
            client = redis.Redis(
                host=host,
                port=int(port),
                decode_responses=True
            )
            client.ping()

            self.nodes[node_address] = client
            self.hash_ring.add_node(node_address)

            logger.info(f"Added node: {node_address}")

        except Exception as e:
            logger.error(f"Failed to add node {node_address}: {e}")
            raise

    def remove_node(self, node_address: str):
        """Remove cache node"""
        if node_address in self.nodes:
            self.nodes[node_address].close()
            del self.nodes[node_address]
            self.hash_ring.remove_node(node_address)

            logger.info(f"Removed node: {node_address}")

# Example usage
cache = DistributedCache(
    nodes=["localhost:6379", "localhost:6380", "localhost:6381"],
    replicas=2
)

# Store data
cache.put("user:123", "John Doe", ttl=3600)
cache.put("session:abc", "active")

# Retrieve data
user = cache.get("user:123")
print(f"User: {user}")

# Delete data
cache.delete("session:abc")
```

## 🔍 Cache Strategies

### 1. Cache-Aside (Lazy Loading)

```python
class CacheAsidePattern:
    """
    Cache-Aside (Read-Through) pattern

    - Application checks cache first
    - On miss, load from database and populate cache
    - Best for read-heavy workloads
    """

    def __init__(self, cache: DistributedCache, db):
        self.cache = cache
        self.db = db

    def get_user(self, user_id: int):
        """Get user with caching"""
        cache_key = f"user:{user_id}"

        # Try cache first
        cached_user = self.cache.get(cache_key)
        if cached_user:
            return cached_user

        # Cache miss - load from database
        user = self.db.get_user(user_id)

        if user:
            # Populate cache
            self.cache.put(cache_key, user, ttl=3600)

        return user

    def update_user(self, user_id: int, data: dict):
        """Update user and invalidate cache"""
        # Update database
        self.db.update_user(user_id, data)

        # Invalidate cache
        cache_key = f"user:{user_id}"
        self.cache.delete(cache_key)
```

### 2. Write-Through Cache

```python
class WriteThroughPattern:
    """
    Write-Through pattern

    - Write to cache and database simultaneously
    - Ensures cache always consistent
    - Higher write latency
    """

    def __init__(self, cache: DistributedCache, db):
        self.cache = cache
        self.db = db

    def create_user(self, user_id: int, data: dict):
        """Create user in database and cache"""
        # Write to database
        user = self.db.create_user(user_id, data)

        # Write to cache
        cache_key = f"user:{user_id}"
        self.cache.put(cache_key, user, ttl=3600)

        return user

    def update_user(self, user_id: int, data: dict):
        """Update user in database and cache"""
        # Update database
        user = self.db.update_user(user_id, data)

        # Update cache
        cache_key = f"user:{user_id}"
        self.cache.put(cache_key, user, ttl=3600)

        return user
```

### 3. Write-Behind (Write-Back)

```python
import queue
import threading
import time

class WriteBehindPattern:
    """
    Write-Behind pattern

    - Write to cache immediately
    - Write to database asynchronously
    - Lower write latency
    - Risk of data loss if cache fails
    """

    def __init__(self, cache: DistributedCache, db):
        self.cache = cache
        self.db = db
        self.write_queue = queue.Queue()

        # Start background writer thread
        self.writer_thread = threading.Thread(target=self._background_writer, daemon=True)
        self.writer_thread.start()

    def update_user(self, user_id: int, data: dict):
        """Update user (fast write to cache)"""
        cache_key = f"user:{user_id}"

        # Write to cache immediately
        self.cache.put(cache_key, data, ttl=3600)

        # Queue database write
        self.write_queue.put((user_id, data))

    def _background_writer(self):
        """Background thread for database writes"""
        while True:
            try:
                # Get pending write
                user_id, data = self.write_queue.get(timeout=1)

                # Write to database
                self.db.update_user(user_id, data)

                self.write_queue.task_done()

            except queue.Empty:
                continue
            except Exception as e:
                logger.error(f"Background write failed: {e}")
```

## ❓ Interview Questions

### Q1: How does consistent hashing minimize redistribution?

**Answer**:
Consistent hashing maps both keys and nodes to a hash ring. When a node is added/removed, only keys between that node and its predecessor need redistribution (~1/N keys), not all keys. Virtual nodes ensure even distribution.

### Q2: What's the difference between LRU and LFU eviction?

**Answer**:

- **LRU** (Least Recently Used): Evicts items not accessed for longest time. Good for time-based access patterns.
- **LFU** (Least Frequently Used): Evicts items accessed least often. Good for frequency-based patterns.
- **Trade-off**: LRU simpler, LFU better for long-term popular items.

### Q3: How do you handle cache stampede?

**Answer**:
Cache stampede occurs when many requests for expired key hit database simultaneously.

**Solutions**:

1. **Lock-based**: First request locks, others wait
2. **Probabilistic early expiration**: Refresh before expiration
3. **Background refresh**: Refresh popular keys proactively
4. **Stale-while-revalidate**: Serve stale data while refreshing

### Q4: How do you ensure cache consistency?

**Answer**:

1. **TTL**: Set appropriate expiration times
2. **Invalidation**: Delete cache on writes
3. **Write-through**: Update cache and database together
4. **Event-driven**: Use message queue for invalidation
5. **Versioning**: Add version to cache keys

## 📚 Summary

**Key Takeaways**:

1. **LRU Cache**: O(1) operations using OrderedDict
2. **Consistent Hashing**: Minimizes redistribution (use virtual nodes)
3. **Replication**: Improve availability (primary + replicas)
4. **Cache-Aside**: Best for read-heavy workloads
5. **Write-Through**: Best for consistency
6. **Write-Behind**: Best for write performance
7. **TTL**: Always set expiration times
8. **Fail Open**: Handle cache failures gracefully
9. **Monitoring**: Track hit rate, latency, evictions
10. **Sharding**: Distribute load across nodes

Distributed caching is critical for scalable systems!
