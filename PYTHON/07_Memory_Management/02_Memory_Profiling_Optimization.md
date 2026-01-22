# Memory Profiling and Optimization

## 📖 Concept Overview

Memory profiling is the process of analyzing a program's memory usage to identify memory leaks, optimize allocation patterns, and reduce overall memory footprint.

## 🧠 Why It Matters

- **Performance**: Excessive memory usage slows applications
- **Scalability**: Memory constraints limit concurrent users
- **Cost**: Cloud resources cost more with higher memory
- **Stability**: Memory leaks cause crashes and downtime
- **Production Issues**: Memory problems often appear only under load

## ⚙️ Memory Profiling Tools

### 1. memory_profiler

Line-by-line memory usage analysis.

```python
# Install: pip install memory-profiler

from memory_profiler import profile

@profile
def load_data():
    """Profile memory usage line by line"""
    data = []
    for i in range(1000000):
        data.append(i)

    result = [x * 2 for x in data]
    return result

# Run with: python -m memory_profiler script.py
# Output:
# Line #    Mem usage    Increment  Line Contents
# ================================================
#      3     38.3 MiB     38.3 MiB   def load_data():
#      4     38.3 MiB      0.0 MiB       data = []
#      5     76.0 MiB     37.7 MiB       for i in range(1000000):
#      6     76.0 MiB      0.0 MiB           data.append(i)
#      7    114.0 MiB     38.0 MiB       result = [x * 2 for x in data]
```

### 2. tracemalloc (Built-in)

Python's built-in memory tracking module.

```python
import tracemalloc
import linecache

def display_top(snapshot, key_type='lineno', limit=10):
    """Display top memory consumers"""
    snapshot = snapshot.filter_traces((
        tracemalloc.Filter(False, "<frozen importlib._bootstrap>"),
        tracemalloc.Filter(False, "<unknown>"),
    ))
    top_stats = snapshot.statistics(key_type)

    print(f"Top {limit} memory consumers:")
    for index, stat in enumerate(top_stats[:limit], 1):
        frame = stat.traceback[0]
        filename = frame.filename
        lineno = frame.lineno
        line = linecache.getline(filename, lineno).strip()

        print(f"#{index}: {filename}:{lineno}: {stat.size / 1024:.1f} KiB")
        print(f"    {line}")

# Usage
tracemalloc.start()

# Your code here
data = [i for i in range(1000000)]
result = [x * 2 for x in data]

snapshot = tracemalloc.take_snapshot()
display_top(snapshot)

tracemalloc.stop()
```

### 3. objgraph

Visualize object references and memory leaks.

```python
# Install: pip install objgraph

import objgraph

# Show most common objects
objgraph.show_most_common_types()

# Track object growth
objgraph.show_growth()

# Find reference chains
x = []
y = [x, [x], dict(x=x)]
objgraph.show_chain(
    objgraph.find_backref_chain(
        x,
        objgraph.is_proper_module
    ),
    filename='chain.png'
)
```

### 4. guppy3 (heapy)

Heap analysis and memory profiling.

```python
# Install: pip install guppy3

from guppy import hpy

h = hpy()
print(h.heap())

# Output:
# Partition of a set of 50000 objects. Total size = 8000000 bytes.
#  Index  Count   %     Size   % Cumulative  % Kind (class / dict of class)
#      0  25000  50  2000000  25   2000000  25 str
#      1  10000  20  1600000  20   3600000  45 list
#      2   8000  16  1280000  16   4880000  61 dict
```

## 🔍 Common Memory Issues

### 1. Memory Leaks

**Problem**: Objects not being garbage collected.

```python
# ❌ Memory leak: Circular reference
class Node:
    def __init__(self, value):
        self.value = value
        self.parent = None
        self.children = []

    def add_child(self, child):
        child.parent = self  # Circular reference!
        self.children.append(child)

# Objects never freed even after del
root = Node(1)
child = Node(2)
root.add_child(child)
del root  # Memory not freed due to circular reference

# ✅ Solution 1: Use weakref
import weakref

class Node:
    def __init__(self, value):
        self.value = value
        self.parent = None  # Will use weakref
        self.children = []

    def add_child(self, child):
        child.parent = weakref.ref(self)  # Weak reference
        self.children.append(child)

# ✅ Solution 2: Explicit cleanup
class Node:
    def __init__(self, value):
        self.value = value
        self.parent = None
        self.children = []

    def add_child(self, child):
        child.parent = self
        self.children.append(child)

    def cleanup(self):
        """Explicit cleanup method"""
        self.parent = None
        for child in self.children:
            child.cleanup()
        self.children.clear()
```

### 2. Large Object in Memory

**Problem**: Keeping large objects in memory unnecessarily.

```python
# ❌ Bad: Loading entire file into memory
def process_large_file(filename):
    with open(filename, 'r') as f:
        data = f.read()  # Loads entire file!

    lines = data.split('\n')
    for line in lines:
        process_line(line)

# ✅ Good: Stream processing
def process_large_file(filename):
    with open(filename, 'r') as f:
        for line in f:  # Read line by line
            process_line(line.strip())

# ✅ Good: Generator for large datasets
def read_large_file(filename):
    """Generator yields lines one at a time"""
    with open(filename, 'r') as f:
        for line in f:
            yield line.strip()

for line in read_large_file('huge.txt'):
    process_line(line)
```

### 3. Unnecessary Data Copies

**Problem**: Creating unnecessary copies of data.

```python
import numpy as np

# ❌ Bad: Creates copy
def process_array(arr):
    new_arr = arr.copy()  # Unnecessary copy
    new_arr *= 2
    return new_arr

# ✅ Good: In-place operation
def process_array(arr):
    arr *= 2  # Modifies in place
    return arr

# ✅ Good: View instead of copy
large_array = np.arange(1000000)
view = large_array[::2]  # View, not copy
copy = large_array[::2].copy()  # Explicit copy when needed
```

### 4. String Concatenation

**Problem**: Repeated string concatenation creates many intermediate objects.

```python
# ❌ Bad: Creates n intermediate strings
def build_string(items):
    result = ""
    for item in items:
        result += str(item) + ","  # Creates new string each time
    return result

# ✅ Good: Use join
def build_string(items):
    return ",".join(str(item) for item in items)

# ✅ Good: Use StringIO for complex building
from io import StringIO

def build_complex_string(items):
    buffer = StringIO()
    for item in items:
        buffer.write(str(item))
        buffer.write(",")
    return buffer.getvalue()
```

## 🏗️ Real-World Example: Data Processing Pipeline

```python
import sys
from memory_profiler import profile
from typing import Iterator, List

class DataPipeline:
    """Memory-efficient data processing"""

    @staticmethod
    @profile
    def process_inefficient(filename: str) -> List[dict]:
        """❌ Inefficient: Loads everything into memory"""
        # Load all data
        with open(filename, 'r') as f:
            lines = f.readlines()  # Entire file in memory

        # Parse all at once
        data = [parse_line(line) for line in lines]

        # Filter
        filtered = [item for item in data if item['value'] > 100]

        # Transform
        result = [transform(item) for item in filtered]

        return result

    @staticmethod
    @profile
    def process_efficient(filename: str) -> Iterator[dict]:
        """✅ Efficient: Streaming with generators"""
        def read_lines(f):
            for line in f:
                yield line.strip()

        def parse_items(lines):
            for line in lines:
                yield parse_line(line)

        def filter_items(items):
            for item in items:
                if item['value'] > 100:
                    yield item

        def transform_items(items):
            for item in items:
                yield transform(item)

        # Pipeline: read -> parse -> filter -> transform
        with open(filename, 'r') as f:
            pipeline = transform_items(
                filter_items(
                    parse_items(
                        read_lines(f)
                    )
                )
            )

            # Process one item at a time
            for item in pipeline:
                yield item

def parse_line(line: str) -> dict:
    parts = line.split(',')
    return {'id': int(parts[0]), 'value': int(parts[1])}

def transform(item: dict) -> dict:
    return {**item, 'processed': True, 'double': item['value'] * 2}

# Compare memory usage
if __name__ == '__main__':
    filename = 'large_data.csv'

    # Inefficient version
    print("Inefficient version:")
    result1 = DataPipeline.process_inefficient(filename)
    print(f"Memory used: {sys.getsizeof(result1) / 1024 / 1024:.2f} MB")

    # Efficient version
    print("\nEfficient version:")
    result2 = list(DataPipeline.process_efficient(filename))
    print(f"Peak memory much lower (see profile)")
```

## 🏢 Real-World Example: Caching Strategy

```python
from functools import lru_cache
import weakref
from typing import Optional

class MemoryEfficientCache:
    """Cache with memory management"""

    def __init__(self, max_size: int = 1000):
        self.max_size = max_size
        self.cache = {}
        self.access_count = {}

    def get(self, key: str) -> Optional[any]:
        """Get from cache"""
        if key in self.cache:
            self.access_count[key] += 1
            return self.cache[key]
        return None

    def set(self, key: str, value: any):
        """Set in cache with eviction"""
        # Evict if at max size
        if len(self.cache) >= self.max_size:
            self._evict_lru()

        self.cache[key] = value
        self.access_count[key] = 1

    def _evict_lru(self):
        """Evict least recently used item"""
        if not self.access_count:
            return

        lru_key = min(self.access_count, key=self.access_count.get)
        del self.cache[lru_key]
        del self.access_count[lru_key]

    def memory_usage(self) -> dict:
        """Report memory usage"""
        import sys
        return {
            'cache_size': len(self.cache),
            'cache_bytes': sys.getsizeof(self.cache),
            'avg_item_bytes': (
                sys.getsizeof(self.cache) / len(self.cache)
                if self.cache else 0
            )
        }

# Using weakref for caching objects
class ObjectCache:
    """Cache using weak references"""

    def __init__(self):
        self.cache = weakref.WeakValueDictionary()

    def get_or_create(self, key: str, factory):
        """Get from cache or create if not exists"""
        obj = self.cache.get(key)
        if obj is None:
            obj = factory()
            self.cache[key] = obj
        return obj

# Usage
cache = ObjectCache()

class HeavyObject:
    def __init__(self, data):
        self.data = data

# Objects automatically cleaned up when no longer referenced
obj1 = cache.get_or_create('key1', lambda: HeavyObject([1, 2, 3]))
obj2 = cache.get_or_create('key1', lambda: HeavyObject([4, 5, 6]))
print(obj1 is obj2)  # True - same object

del obj1, obj2  # Objects can now be garbage collected
```

## 🎯 Optimization Techniques

### 1. Use **slots**

```python
import sys

# Without __slots__
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

# With __slots__
class OptimizedPoint:
    __slots__ = ['x', 'y']

    def __init__(self, x, y):
        self.x = x
        self.y = y

# Memory comparison
p1 = Point(1, 2)
p2 = OptimizedPoint(1, 2)

print(f"Point: {sys.getsizeof(p1)} bytes")
print(f"OptimizedPoint: {sys.getsizeof(p2)} bytes")

# Creating many instances
points = [Point(i, i) for i in range(1000000)]
optimized = [OptimizedPoint(i, i) for i in range(1000000)]

# Memory savings: ~40-50% reduction
```

### 2. Use Generators Instead of Lists

```python
# ❌ List: All in memory
def get_squares_list(n):
    return [x**2 for x in range(n)]

squares = get_squares_list(1000000)  # ~8MB memory

# ✅ Generator: One at a time
def get_squares_gen(n):
    return (x**2 for x in range(n))

squares = get_squares_gen(1000000)  # ~80 bytes memory
```

### 3. Delete Large Objects Explicitly

```python
import gc

# Process large dataset
large_data = load_large_dataset()
process(large_data)

# Explicitly free memory
del large_data
gc.collect()  # Force garbage collection
```

### 4. Use Memory-Efficient Data Structures

```python
from array import array
from collections import deque

# ❌ List of integers: More memory
numbers_list = [i for i in range(1000000)]

# ✅ Array: Less memory for homogeneous types
numbers_array = array('i', range(1000000))

# ✅ Deque for queues: O(1) append/pop on both ends
queue = deque(maxlen=1000)  # Automatically removes old items
```

## ✅ Best Practices

### 1. Profile Before Optimizing

```python
# Always measure first
import tracemalloc

tracemalloc.start()

# Your code
result = expensive_operation()

current, peak = tracemalloc.get_traced_memory()
print(f"Current: {current / 1024 / 1024:.1f}MB")
print(f"Peak: {peak / 1024 / 1024:.1f}MB")

tracemalloc.stop()
```

### 2. Use Context Managers for Resources

```python
# ✅ Automatic cleanup
with open('file.txt', 'r') as f:
    data = f.read()
# File automatically closed, memory freed

# ✅ Custom context manager
class ManagedResource:
    def __enter__(self):
        self.resource = acquire_resource()
        return self.resource

    def __exit__(self, exc_type, exc_val, exc_tb):
        release_resource(self.resource)
        del self.resource  # Explicit cleanup
```

### 3. Batch Processing for Large Datasets

```python
def process_in_batches(items, batch_size=1000):
    """Process large dataset in batches"""
    batch = []
    for item in items:
        batch.append(item)

        if len(batch) >= batch_size:
            process_batch(batch)
            batch.clear()  # Free memory

    # Process remaining items
    if batch:
        process_batch(batch)
```

## ❌ Common Mistakes

### 1. Not Closing File Handles

```python
# ❌ Bad: File handle kept open
f = open('file.txt')
data = f.read()
# Never closed!

# ✅ Good: Use context manager
with open('file.txt') as f:
    data = f.read()
```

### 2. Keeping References to Large Objects

```python
# ❌ Bad: Keeping unnecessary reference
class DataProcessor:
    def __init__(self):
        self.results = []  # Keeps growing!

    def process(self, data):
        result = expensive_operation(data)
        self.results.append(result)  # Memory leak!

# ✅ Good: Don't store if not needed
class DataProcessor:
    def process(self, data):
        result = expensive_operation(data)
        save_result(result)  # Save and discard
        return result
```

### 3. Not Using Pagination for Queries

```python
# ❌ Bad: Load everything
def get_all_users():
    return User.objects.all()  # Could be millions!

# ✅ Good: Paginate
def get_users_paginated(page=1, per_page=100):
    offset = (page - 1) * per_page
    return User.objects.all()[offset:offset + per_page]
```

## ❓ Interview Questions

### Q1: How do you identify memory leaks in Python?

**Answer**:

1. **Use tracemalloc**: Track memory allocations
2. **Monitor object growth**: Use `objgraph.show_growth()`
3. **Check circular references**: Use `gc.get_referrers()`
4. **Profile with memory_profiler**: Line-by-line analysis

Common causes:

- Circular references
- Unclosed file handles
- Global caches without eviction
- Event listeners not removed

### Q2: What is the difference between memory leak and memory bloat?

**Answer**:

- **Memory leak**: Objects that should be freed but aren't (bugs)
- **Memory bloat**: Legitimate memory usage that's excessive (inefficiency)

Memory leak: Circular references, unclosed resources
Memory bloat: Loading entire file when streaming would work

### Q3: How does **slots** save memory?

**Answer**:
Without `__slots__`, each instance has a `__dict__` (hash table) for attributes.
With `__slots__`, attributes stored in fixed-size array.

Savings: ~40-50% for objects with few attributes
Trade-off: Can't add new attributes dynamically

### Q4: When should you use generators vs lists?

**Answer**:
**Use generators**:

- Large datasets that don't fit in memory
- Processing streams
- One-time iteration
- Pipeline processing

**Use lists**:

- Need random access
- Multiple iterations
- Small datasets
- Need to modify elements

## 📚 Summary

### Key Takeaways

1. **Profile first** - Measure before optimizing
2. **Use generators** for large datasets
3. **Close resources** with context managers
4. **Avoid circular references** or use weakref
5. **Batch processing** for large operations
6. ****slots**** for many small objects
7. **Monitor production** memory usage

### Memory Optimization Checklist

- ✅ Profile with tracemalloc or memory_profiler
- ✅ Use generators instead of lists for large data
- ✅ Implement proper resource cleanup
- ✅ Use **slots** for frequently created objects
- ✅ Avoid keeping unnecessary references
- ✅ Implement pagination for queries
- ✅ Use appropriate data structures
- ✅ Monitor memory in production

Memory optimization is about finding the right balance between memory usage, performance, and code complexity. Always measure first, then optimize the proven bottlenecks.
