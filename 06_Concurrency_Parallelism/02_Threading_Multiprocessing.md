# 🧵 Threading and Multiprocessing

## Overview

Python provides multiple ways to achieve concurrency: threading for I/O-bound tasks and multiprocessing for CPU-bound tasks. Understanding the GIL is crucial for choosing the right approach.

---

## The Global Interpreter Lock (GIL)

### What is the GIL?

```python
import threading
import time

# The GIL prevents multiple threads from executing Python bytecode simultaneously
# Only ONE thread can execute Python code at a time

def cpu_bound_task(n):
    """CPU-intensive task"""
    count = 0
    for i in range(n):
        count += i ** 2
    return count

# Threading - does NOT improve CPU-bound tasks due to GIL
def test_threading():
    """Test with threads"""
    start = time.time()

    threads = []
    for _ in range(4):
        t = threading.Thread(target=cpu_bound_task, args=(10_000_000,))
        t.start()
        threads.append(t)

    for t in threads:
        t.join()

    print(f"Threading: {time.time() - start:.2f}s")  # ~4 seconds (sequential)

# Multiprocessing - DOES improve CPU-bound tasks
from multiprocessing import Process

def test_multiprocessing():
    """Test with processes"""
    start = time.time()

    processes = []
    for _ in range(4):
        p = Process(target=cpu_bound_task, args=(10_000_000,))
        p.start()
        processes.append(p)

    for p in processes:
        p.join()

    print(f"Multiprocessing: {time.time() - start:.2f}s")  # ~1 second (parallel)
```

---

## Threading

### Basic Threading

```python
import threading
import time

def worker(name, delay):
    """Worker thread function"""
    print(f"{name} starting")
    time.sleep(delay)
    print(f"{name} finished")

# Create and start threads
threads = []
for i in range(5):
    t = threading.Thread(
        target=worker,
        args=(f"Thread-{i}", i * 0.5)
    )
    t.start()
    threads.append(t)

# Wait for all threads to complete
for t in threads:
    t.join()

print("All threads completed")
```

### Thread with Return Value

```python
import threading
import queue

def fetch_url(url, result_queue):
    """Fetch URL and put result in queue"""
    import requests
    try:
        response = requests.get(url, timeout=5)
        result_queue.put((url, response.status_code))
    except Exception as e:
        result_queue.put((url, str(e)))

def fetch_multiple_urls(urls):
    """Fetch multiple URLs concurrently"""
    result_queue = queue.Queue()
    threads = []

    for url in urls:
        t = threading.Thread(target=fetch_url, args=(url, result_queue))
        t.start()
        threads.append(t)

    # Wait for all threads
    for t in threads:
        t.join()

    # Collect results
    results = []
    while not result_queue.empty():
        results.append(result_queue.get())

    return results

# Usage
urls = [
    "https://api.github.com",
    "https://api.github.com/users",
    "https://api.github.com/repos"
]
results = fetch_multiple_urls(urls)
```

### Thread Pool Executor

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests

def fetch_url(url):
    """Fetch single URL"""
    response = requests.get(url, timeout=5)
    return url, response.status_code

def fetch_urls_with_pool(urls, max_workers=5):
    """Fetch URLs using thread pool"""
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        # Submit all tasks
        future_to_url = {
            executor.submit(fetch_url, url): url
            for url in urls
        }

        # Process as they complete
        results = []
        for future in as_completed(future_to_url):
            url = future_to_url[future]
            try:
                result = future.result()
                results.append(result)
            except Exception as e:
                print(f"Error fetching {url}: {e}")

        return results

# Usage with map (simpler for uniform tasks)
def simple_fetch_with_pool(urls):
    """Simple fetch with map"""
    with ThreadPoolExecutor(max_workers=10) as executor:
        results = list(executor.map(fetch_url, urls))
    return results
```

### Thread Synchronization

```python
import threading

# Lock - mutual exclusion
lock = threading.Lock()
counter = 0

def increment_with_lock():
    """Thread-safe increment"""
    global counter
    for _ in range(100000):
        with lock:  # Only one thread can execute this block
            counter += 1

# RLock - reentrant lock (can be acquired multiple times by same thread)
rlock = threading.RLock()

def recursive_function(n):
    """Recursive function with RLock"""
    with rlock:
        if n > 0:
            recursive_function(n - 1)

# Semaphore - limit concurrent access
semaphore = threading.Semaphore(3)  # Max 3 concurrent

def limited_resource():
    """Access limited resource"""
    with semaphore:
        print(f"{threading.current_thread().name} acquired semaphore")
        time.sleep(1)
        print(f"{threading.current_thread().name} released semaphore")

# Event - signal between threads
event = threading.Event()

def wait_for_event():
    """Wait for event"""
    print("Waiting for event...")
    event.wait()  # Blocks until event is set
    print("Event received!")

def trigger_event():
    """Trigger event"""
    time.sleep(2)
    print("Setting event")
    event.set()

# Condition - complex synchronization
condition = threading.Condition()
items = []

def producer():
    """Produce items"""
    with condition:
        items.append("item")
        condition.notify()  # Wake up waiting consumer

def consumer():
    """Consume items"""
    with condition:
        while not items:
            condition.wait()  # Wait for item
        item = items.pop()
        return item
```

---

## Multiprocessing

### Basic Multiprocessing

```python
from multiprocessing import Process, Queue, Pool
import os

def worker_process(name):
    """Worker process"""
    print(f"Process {name} (PID: {os.getpid()}) starting")
    time.sleep(2)
    print(f"Process {name} finished")

# Create processes
processes = []
for i in range(4):
    p = Process(target=worker_process, args=(f"Worker-{i}",))
    p.start()
    processes.append(p)

# Wait for all processes
for p in processes:
    p.join()
```

### Process Pool

```python
from multiprocessing import Pool

def square(x):
    """Square a number"""
    return x ** 2

def parallel_computation(numbers):
    """Compute squares in parallel"""
    with Pool(processes=4) as pool:
        results = pool.map(square, numbers)
    return results

# Usage
numbers = list(range(1000))
results = parallel_computation(numbers)

# With more control
def parallel_with_starmap(data_pairs):
    """Use starmap for multiple arguments"""
    def add(x, y):
        return x + y

    with Pool(processes=4) as pool:
        results = pool.starmap(add, data_pairs)
    return results

# Async execution
def async_parallel():
    """Async process execution"""
    with Pool(processes=4) as pool:
        # Start tasks asynchronously
        result1 = pool.apply_async(square, (10,))
        result2 = pool.apply_async(square, (20,))

        # Get results when ready
        print(result1.get(timeout=5))
        print(result2.get(timeout=5))
```

### Inter-Process Communication

```python
from multiprocessing import Process, Queue, Pipe

# Queue - thread and process safe
def producer_process(queue):
    """Produce items into queue"""
    for i in range(10):
        queue.put(f"item-{i}")
        time.sleep(0.1)
    queue.put(None)  # Sentinel

def consumer_process(queue):
    """Consume items from queue"""
    while True:
        item = queue.get()
        if item is None:
            break
        print(f"Consumed: {item}")

# Usage
queue = Queue()
p1 = Process(target=producer_process, args=(queue,))
p2 = Process(target=consumer_process, args=(queue,))
p1.start()
p2.start()
p1.join()
p2.join()

# Pipe - two-way communication
def pipe_example():
    """Use pipe for communication"""
    parent_conn, child_conn = Pipe()

    def child_process(conn):
        """Child process"""
        conn.send("Hello from child")
        msg = conn.recv()
        print(f"Child received: {msg}")
        conn.close()

    p = Process(target=child_process, args=(child_conn,))
    p.start()

    msg = parent_conn.recv()
    print(f"Parent received: {msg}")
    parent_conn.send("Hello from parent")

    p.join()
```

### Shared Memory

```python
from multiprocessing import Process, Value, Array, Manager

# Value - shared primitive
def increment_shared_value(shared_val):
    """Increment shared value"""
    with shared_val.get_lock():
        for _ in range(1000):
            shared_val.value += 1

# Usage
shared_value = Value('i', 0)  # 'i' = integer
processes = [
    Process(target=increment_shared_value, args=(shared_value,))
    for _ in range(4)
]

for p in processes:
    p.start()
for p in processes:
    p.join()

print(f"Final value: {shared_value.value}")  # 4000

# Array - shared array
shared_array = Array('d', [1.0, 2.0, 3.0, 4.0])  # 'd' = double

def modify_array(arr, index, value):
    """Modify shared array"""
    arr[index] = value

# Manager - more complex shared objects
def use_manager():
    """Use Manager for complex types"""
    with Manager() as manager:
        # Shared dict
        shared_dict = manager.dict()

        # Shared list
        shared_list = manager.list()

        def worker(shared_d, shared_l, name):
            shared_d[name] = os.getpid()
            shared_l.append(name)

        processes = [
            Process(target=worker, args=(shared_dict, shared_list, f"Worker-{i}"))
            for i in range(4)
        ]

        for p in processes:
            p.start()
        for p in processes:
            p.join()

        print(f"Dict: {dict(shared_dict)}")
        print(f"List: {list(shared_list)}")
```

---

## Choosing the Right Approach

### Decision Matrix

```python
"""
┌─────────────────────┬──────────────┬─────────────────┬──────────────────┐
│ Task Type           │ Threading    │ Multiprocessing │ Async/Await      │
├─────────────────────┼──────────────┼─────────────────┼──────────────────┤
│ I/O-bound           │ ✅ Good      │ ⚠️ Overkill    │ ✅ Best          │
│ CPU-bound           │ ❌ Poor      │ ✅ Best         │ ❌ Poor          │
│ Network calls       │ ✅ Good      │ ⚠️ Overkill    │ ✅ Best          │
│ Database queries    │ ✅ Good      │ ⚠️ Maybe       │ ✅ Best          │
│ File I/O            │ ✅ Good      │ ⚠️ Maybe       │ ✅ Good          │
│ Heavy computation   │ ❌ Poor      │ ✅ Best         │ ❌ Poor          │
│ Shared state        │ ⚠️ Complex   │ ❌ Very Hard    │ ✅ Easy          │
└─────────────────────┴──────────────┴─────────────────┴──────────────────┘
"""

# Example: I/O-bound task
def io_bound_task():
    """Choose async or threading"""
    # Best: Async
    async def fetch_async():
        async with httpx.AsyncClient() as client:
            await client.get(url)

    # Good: Threading
    def fetch_threaded():
        with ThreadPoolExecutor() as executor:
            executor.submit(requests.get, url)

# Example: CPU-bound task
def cpu_bound_task():
    """Choose multiprocessing"""
    # Best: Multiprocessing
    def compute_parallel():
        with Pool() as pool:
            pool.map(expensive_computation, data)

    # Poor: Threading (GIL limits performance)
    def compute_threaded():
        with ThreadPoolExecutor() as executor:
            executor.map(expensive_computation, data)
```

---

## Real-World Patterns

### 1. **Web Scraping**

```python
from concurrent.futures import ThreadPoolExecutor
import requests
from bs4 import BeautifulSoup

class WebScraper:
    """Concurrent web scraper"""

    def __init__(self, max_workers=10):
        self.max_workers = max_workers

    def scrape_page(self, url):
        """Scrape single page"""
        try:
            response = requests.get(url, timeout=10)
            soup = BeautifulSoup(response.content, 'html.parser')
            return {
                'url': url,
                'title': soup.title.string if soup.title else None,
                'status': response.status_code
            }
        except Exception as e:
            return {'url': url, 'error': str(e)}

    def scrape_urls(self, urls):
        """Scrape multiple URLs concurrently"""
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            results = list(executor.map(self.scrape_page, urls))
        return results
```

### 2. **Data Processing Pipeline**

```python
from multiprocessing import Pool, cpu_count
import pandas as pd

def process_chunk(chunk):
    """Process data chunk"""
    # CPU-intensive transformations
    chunk['processed'] = chunk['value'].apply(lambda x: x ** 2)
    return chunk

def parallel_dataframe_processing(df, chunk_size=1000):
    """Process large DataFrame in parallel"""
    # Split into chunks
    chunks = [df[i:i + chunk_size] for i in range(0, len(df), chunk_size)]

    # Process in parallel
    with Pool(cpu_count()) as pool:
        results = pool.map(process_chunk, chunks)

    # Combine results
    return pd.concat(results, ignore_index=True)
```

### 3. **Background Task Queue**

```python
import threading
import queue

class BackgroundTaskQueue:
    """Background task processor"""

    def __init__(self, num_workers=3):
        self.task_queue = queue.Queue()
        self.workers = []

        for i in range(num_workers):
            worker = threading.Thread(
                target=self._worker,
                args=(f"Worker-{i}",),
                daemon=True
            )
            worker.start()
            self.workers.append(worker)

    def _worker(self, name):
        """Worker thread"""
        while True:
            task, args = self.task_queue.get()
            try:
                print(f"{name} processing task")
                task(*args)
            except Exception as e:
                print(f"{name} error: {e}")
            finally:
                self.task_queue.task_done()

    def submit(self, task, *args):
        """Submit task to queue"""
        self.task_queue.put((task, args))

    def wait(self):
        """Wait for all tasks to complete"""
        self.task_queue.join()

# Usage
queue = BackgroundTaskQueue(num_workers=5)

for i in range(20):
    queue.submit(time.sleep, 0.5)

queue.wait()
print("All tasks completed")
```

---

## Best Practices

### ✅ Do's:

1. **Use async for I/O-bound** tasks
2. **Use multiprocessing for CPU-bound** tasks
3. **Use thread pools** instead of creating threads manually
4. **Set timeouts** on blocking operations
5. **Handle exceptions** in workers
6. **Use context managers** (with statements)
7. **Limit concurrent workers** to avoid overwhelming system
8. **Test with real workloads**

### ❌ Don'ts:

1. **Don't use threading for CPU-bound** tasks
2. **Don't share mutable state** without synchronization
3. **Don't create unlimited threads/processes**
4. **Don't forget to join** threads/processes
5. **Don't ignore the GIL** when choosing approach
6. **Don't use shared memory** unless necessary

---

## Interview Questions

### Q1: What is the GIL and how does it affect Python?

**Answer**: Global Interpreter Lock - only one thread executes Python bytecode at a time:

- Prevents race conditions in CPython internals
- Limits threading for CPU-bound tasks
- Doesn't affect I/O-bound tasks (I/O releases GIL)
- Use multiprocessing to bypass GIL for CPU work

### Q2: When to use threading vs multiprocessing?

**Answer**:

- **Threading**: I/O-bound (network, file), shared memory, lighter overhead
- **Multiprocessing**: CPU-bound, true parallelism, separate memory
- **Async**: I/O-bound, single-threaded concurrency, most efficient for I/O

### Q3: How do you share data between processes?

**Answer**:

- **Queue**: Thread and process safe, FIFO
- **Pipe**: Two-way communication
- **Value/Array**: Shared memory primitives
- **Manager**: Shared dicts, lists (slower, more flexible)

### Q4: What's the difference between Process and Thread?

**Answer**:

- **Process**: Separate memory, true parallelism, heavier, bypass GIL
- **Thread**: Shared memory, concurrent (not parallel with GIL), lighter

### Q5: How do you prevent race conditions?

**Answer**:

- Use **Lock** for mutual exclusion
- Use **RLock** for reentrant locks
- Use **Semaphore** for limited access
- Use **atomic operations** where possible
- Minimize shared state

---

## Summary

Concurrency approaches:

- **Threading**: I/O-bound, shared memory, GIL limited
- **Multiprocessing**: CPU-bound, true parallelism, separate memory
- **Async**: I/O-bound, single-threaded, most efficient for I/O
- **Thread Pool**: Managed threading
- **Process Pool**: Managed multiprocessing

Choose based on task type and GIL impact! 🧵
