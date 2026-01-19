# 🗑️ Garbage Collection and Memory Management

## Overview

Understanding Python's memory management is crucial for writing efficient applications and debugging memory leaks.

---

## Reference Counting

### How It Works

```python
import sys

# Reference counting basics
x = [1, 2, 3]
print(sys.getrefcount(x))  # 2 (x + getrefcount parameter)

y = x  # Reference count increases
print(sys.getrefcount(x))  # 3

del y  # Reference count decreases
print(sys.getrefcount(x))  # 2

# Object is deleted when ref count reaches 0
def create_list():
    temp = [1, 2, 3]  # Created
    return temp       # Returned (ref count = 1)
# temp goes out of scope but object still exists (returned)

result = create_list()  # ref count = 2
del result              # ref count = 0 -> deallocated

# Reference count doesn't change identity
a = [1, 2, 3]
b = [1, 2, 3]
print(a is b)  # False (different objects)
print(a == b)  # True (same values)

c = a
print(a is c)  # True (same object)
```

### Circular References Problem

```python
# Problem: Circular references prevent deallocation
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

# Create circular reference
node1 = Node(1)
node2 = Node(2)
node1.next = node2
node2.next = node1  # Circular!

# Even if we delete variables, objects remain in memory
del node1
del node2
# Objects still reference each other (memory leak without GC)

# Solution 1: Break cycle manually
node1 = Node(1)
node2 = Node(2)
node1.next = node2
node2.next = node1

node1.next = None  # Break cycle
node2.next = None

del node1
del node2

# Solution 2: Use weak references
import weakref

class NodeWithWeak:
    def __init__(self, value):
        self.value = value
        self._next = None

    @property
    def next(self):
        """Get next node"""
        return self._next() if self._next else None

    @next.setter
    def next(self, node):
        """Set next node as weak reference"""
        self._next = weakref.ref(node) if node else None

# Now circular reference won't prevent GC
node1 = NodeWithWeak(1)
node2 = NodeWithWeak(2)
node1.next = node2
node2.next = node1

del node1  # Can be garbage collected
del node2
```

---

## Garbage Collection

### Generational Garbage Collection

```python
import gc

# Python's GC collects circular references
# Uses generational approach (gen 0, 1, 2)

# Check GC status
print(gc.isenabled())  # True by default

# Get GC stats
stats = gc.get_stats()
print(f"Collections: {stats}")

# Get GC thresholds (gen0, gen1, gen2)
print(gc.get_threshold())  # (700, 10, 10)

# Generation 0: Young objects (most frequent collection)
# Generation 1: Survived one collection
# Generation 2: Survived multiple collections (least frequent)

# Manually trigger garbage collection
collected = gc.collect()
print(f"Collected {collected} objects")

# Get all tracked objects
objects = gc.get_objects()
print(f"Total tracked objects: {len(objects)}")

# Disable/enable GC
gc.disable()  # Manual control
# ... critical section ...
gc.enable()

# Set custom thresholds
gc.set_threshold(1000, 15, 15)  # Less frequent collection
```

### Debugging Memory Leaks

```python
import gc
import weakref

# Find circular references
class LeakyClass:
    """Class with potential circular reference"""
    instances = []

    def __init__(self, name):
        self.name = name
        self.ref = None
        LeakyClass.instances.append(self)  # Keeps reference!

    def __repr__(self):
        return f"LeakyClass({self.name})"

# Create objects
obj1 = LeakyClass("obj1")
obj2 = LeakyClass("obj2")
obj1.ref = obj2
obj2.ref = obj1

# Delete variables
del obj1
del obj2

# Objects still exist due to class-level list
print(f"Instances: {LeakyClass.instances}")

# Find garbage
gc.collect()  # Force collection
garbage = gc.garbage
print(f"Garbage: {garbage}")

# Find referrers (what's keeping object alive)
obj = LeakyClass.instances[0]
referrers = gc.get_referrers(obj)
print(f"Referrers: {len(referrers)}")

# Fix: Use weak references
class NonLeakyClass:
    """Fixed version with weak references"""
    _instances = []

    def __init__(self, name):
        self.name = name
        self.ref = None
        NonLeakyClass._instances.append(weakref.ref(self))

    @classmethod
    def get_instances(cls):
        """Get alive instances"""
        # Remove dead references
        cls._instances = [ref for ref in cls._instances if ref() is not None]
        return [ref() for ref in cls._instances]
```

---

## Memory Profiling

### Using memory_profiler

```python
from memory_profiler import profile

@profile
def memory_intensive_function():
    """Profile memory usage line by line"""
    # Large list
    big_list = [i for i in range(1_000_000)]

    # Large dict
    big_dict = {i: i**2 for i in range(1_000_000)}

    # Process data
    result = sum(big_list)

    return result

# Run with: python -m memory_profiler script.py
# Output shows memory usage per line

# Manual memory tracking
import tracemalloc

def track_memory():
    """Track memory allocations"""
    # Start tracing
    tracemalloc.start()

    # Take snapshot before
    snapshot1 = tracemalloc.take_snapshot()

    # Code to profile
    data = [i for i in range(1_000_000)]

    # Take snapshot after
    snapshot2 = tracemalloc.take_snapshot()

    # Compare snapshots
    top_stats = snapshot2.compare_to(snapshot1, 'lineno')

    print("Top memory allocations:")
    for stat in top_stats[:5]:
        print(stat)

    # Stop tracing
    tracemalloc.stop()

# Get current memory usage
def get_memory_usage():
    """Get current process memory"""
    import psutil
    import os

    process = psutil.Process(os.getpid())
    mem_info = process.memory_info()

    return {
        'rss': mem_info.rss / 1024 / 1024,  # MB
        'vms': mem_info.vms / 1024 / 1024   # MB
    }
```

### Memory Optimization Techniques

```python
# 1. Use generators instead of lists
def get_numbers_list(n):
    """Returns list - loads all in memory"""
    return [i for i in range(n)]

def get_numbers_gen(n):
    """Returns generator - one at a time"""
    for i in range(n):
        yield i

# Memory difference
import sys
list_obj = get_numbers_list(1_000_000)
print(f"List size: {sys.getsizeof(list_obj) / 1024 / 1024:.2f} MB")

gen_obj = get_numbers_gen(1_000_000)
print(f"Generator size: {sys.getsizeof(gen_obj)} bytes")

# 2. Use __slots__ to reduce memory
class RegularClass:
    """Regular class with __dict__"""
    def __init__(self, x, y):
        self.x = x
        self.y = y

class SlottedClass:
    """Class with __slots__ (no __dict__)"""
    __slots__ = ['x', 'y']

    def __init__(self, x, y):
        self.x = x
        self.y = y

# Memory comparison
regular = RegularClass(1, 2)
slotted = SlottedClass(1, 2)

print(f"Regular: {sys.getsizeof(regular) + sys.getsizeof(regular.__dict__)} bytes")
print(f"Slotted: {sys.getsizeof(slotted)} bytes")  # Much smaller

# 3. Use array for numeric data
from array import array

# List of integers
int_list = [i for i in range(1_000_000)]
print(f"List: {sys.getsizeof(int_list) / 1024 / 1024:.2f} MB")

# Array of integers
int_array = array('i', range(1_000_000))
print(f"Array: {sys.getsizeof(int_array) / 1024 / 1024:.2f} MB")

# 4. Delete large objects explicitly
def process_large_data():
    """Process and clean up"""
    large_data = [i for i in range(10_000_000)]

    # Process data
    result = sum(large_data)

    # Delete large object
    del large_data

    # Force garbage collection
    gc.collect()

    return result

# 5. Use context managers for resources
class MemoryIntensiveResource:
    """Resource that allocates memory"""

    def __init__(self):
        self.data = [i for i in range(1_000_000)]

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # Clean up
        del self.data
        gc.collect()

    def process(self):
        return sum(self.data)

# Automatic cleanup
with MemoryIntensiveResource() as resource:
    result = resource.process()
# Memory freed here
```

---

## Object Interning

### String and Integer Interning

```python
# Small integers are interned (-5 to 256)
a = 100
b = 100
print(a is b)  # True (same object)

a = 1000
b = 1000
print(a is b)  # False (different objects)

# String interning
s1 = "hello"
s2 = "hello"
print(s1 is s2)  # True (interned)

# Dynamic strings not always interned
s1 = "hello world"
s2 = "hello world"
print(s1 is s2)  # May be False

# Force string interning
import sys

s1 = sys.intern("hello world")
s2 = sys.intern("hello world")
print(s1 is s2)  # True (now interned)

# When to use interning
# ✅ Many identical strings (save memory)
# ✅ Fast equality checks (identity instead of value)
# ❌ Large strings (wastes memory)
# ❌ Short-lived strings (no benefit)
```

---

## Memory Pools

### Python Memory Allocator

```python
"""
Python's memory architecture:

1. Object-specific allocators (int, float, etc.)
2. PyMalloc (Python's memory pool for objects < 512 bytes)
3. malloc/free (system allocator for large objects)

PyMalloc Structure:
- Arena: 256 KB chunk
- Pool: 4 KB chunk within arena
- Block: Fixed-size allocation within pool

Benefits:
- Reduces fragmentation
- Faster than system malloc
- Reuses memory
"""

# Small objects use pymalloc
class SmallObject:
    """< 512 bytes, uses pymalloc"""
    def __init__(self):
        self.data = [0] * 10

# Large objects use system allocator
class LargeObject:
    """≥ 512 bytes, uses malloc"""
    def __init__(self):
        self.data = [0] * 1000

# Check memory allocator
import sys
small = SmallObject()
large = LargeObject()

# Object size
print(f"Small: {sys.getsizeof(small)} bytes")
print(f"Large: {sys.getsizeof(large)} bytes")
```

---

## Best Practices

### ✅ Do's:

1. **Let GC work** - don't disable unless necessary
2. **Use weak references** for caches and observers
3. **Break circular references** manually if possible
4. **Use generators** for large sequences
5. **Use `__slots__`** for classes with many instances
6. **Profile memory** usage in production
7. **Delete large objects** explicitly in loops
8. **Use context managers** for cleanup

### ❌ Don'ts:

1. **Don't rely solely** on `__del__` (may not be called)
2. **Don't keep references** in class variables unnecessarily
3. **Don't ignore memory leaks** (they compound)
4. **Don't create circular** references without weak refs
5. **Don't call `gc.collect()`** too frequently (expensive)

---

## Interview Questions

### Q1: Explain Python's garbage collection.

**Answer**: Reference counting + generational GC:

- **Reference counting**: Deallocate when count = 0
- **Generational GC**: Collects circular references
- Three generations: 0 (young), 1, 2 (old)
- Gen 0 collected most frequently

### Q2: What are weak references?

**Answer**: References that don't increase ref count:

- Allow object to be garbage collected
- Use for: Caches, observer pattern, backlinks
- `weakref.ref()` creates weak reference
- Access with `ref()` call (returns None if dead)

### Q3: What is `__slots__`?

**Answer**: Class attribute defining allowed attributes:

- Prevents `__dict__` creation
- Saves memory (no dynamic attributes)
- Faster attribute access
- Trade-off: Less flexibility

### Q4: How do you debug memory leaks?

**Answer**:

- **tracemalloc**: Track allocations
- **memory_profiler**: Line-by-line profiling
- **objgraph**: Find reference chains
- **gc.get_referrers()**: What keeps object alive
- Look for: Circular refs, global caches, event handlers

### Q5: Explain reference counting limitations.

**Answer**:

- **Circular references**: Not handled (needs GC)
- **Thread safety**: Requires GIL (overhead)
- **Overhead**: Every reference change updates count
- **Predictability**: Deallocation timing uncertain with GC

---

## Summary

Python memory management:

- **Reference counting**: Automatic deallocation
- **Garbage collection**: Handles circular references
- **Weak references**: Cache without preventing GC
- **Memory profiling**: tracemalloc, memory_profiler
- **Optimization**: Generators, `__slots__`, arrays
- **Best practices**: Let GC work, profile, break cycles

Understand memory for efficient applications! 🗑️
