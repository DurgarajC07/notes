# Generators and Iterators - Memory-Efficient Iteration

## 📖 Concept Explanation

Generators and iterators are Python's way of handling sequences of data without loading everything into memory at once. They implement **lazy evaluation** - computing values on-demand rather than all at once.

### Iterators

An iterator is an object that implements:
- `__iter__()` - Returns the iterator object itself
- `__next__()` - Returns the next value or raises `StopIteration`

```python
class Counter:
    def __init__(self, start, end):
        self.current = start
        self.end = end
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.current >= self.end:
            raise StopIteration
        self.current += 1
        return self.current - 1

# Usage
for num in Counter(1, 5):
    print(num)  # 1, 2, 3, 4
```

### Generators

Generators are functions that use `yield` instead of `return`. They automatically implement the iterator protocol.

```python
def counter(start, end):
    """Generator function"""
    current = start
    while current < end:
        yield current
        current += 1

# Usage
for num in counter(1, 5):
    print(num)  # 1, 2, 3, 4
```

### Generator Expressions

Like list comprehensions but with parentheses, creating generators instead of lists.

```python
# List comprehension - creates entire list in memory
squares_list = [x**2 for x in range(1000000)]  # Uses ~8MB memory

# Generator expression - creates values on demand
squares_gen = (x**2 for x in range(1000000))   # Uses minimal memory

# Iterate through generator
for square in squares_gen:
    print(square)
    if square > 100:
        break  # No need to compute remaining values
```

## 🧠 Why It Matters in Real Projects

- **Memory Efficiency**: Process large datasets without loading into memory
- **Performance**: Compute values only when needed (lazy evaluation)
- **Streaming Data**: Handle infinite sequences or continuous data streams
- **Pipeline Processing**: Chain multiple operations efficiently
- **Database Queries**: Fetch and process rows one at a time
- **File Processing**: Read large files line by line
- **API Pagination**: Handle paginated API responses

## ⚙️ Internal Working

### How Generators Work Under the Hood

```python
import dis

def simple_generator():
    yield 1
    yield 2
    yield 3

dis.dis(simple_generator)
"""
Bytecode shows:
- GEN_START: Initialize generator
- YIELD_VALUE: Pause execution and return value
- RESUME: Continue execution from last yield point
"""

# Generator state machine
gen = simple_generator()
print(gen.gi_frame)       # Frame object
print(gen.gi_code)        # Code object
print(gen.gi_running)     # False (not currently executing)
```

### Generator States

```python
def state_example():
    print("Start")
    yield 1
    print("After first yield")
    yield 2
    print("After second yield")
    yield 3
    print("End")

gen = state_example()
print(gen.gi_frame)  # Frame object exists

next(gen)  # Prints "Start", yields 1
print(gen.gi_frame.f_lasti)  # Last instruction index

next(gen)  # Resumes from yield point
next(gen)
next(gen)  # Raises StopIteration

print(gen.gi_frame)  # None (generator exhausted)
```

### Memory Comparison

```python
import sys

# List - stores all values in memory
my_list = [x**2 for x in range(10000)]
print(f"List size: {sys.getsizeof(my_list)} bytes")  # ~87,616 bytes

# Generator - stores only the state
my_gen = (x**2 for x in range(10000))
print(f"Generator size: {sys.getsizeof(my_gen)} bytes")  # ~120 bytes
```

## ✅ Best Practices

### 1. Use Generators for Large Data Processing

```python
# BAD - Loads entire file into memory
def read_file_bad(filename):
    with open(filename) as f:
        return f.readlines()  # Returns list of all lines

lines = read_file_bad('large_file.txt')
for line in lines:
    process(line)

# GOOD - Processes one line at a time
def read_file_good(filename):
    with open(filename) as f:
        for line in f:  # File objects are iterators
            yield line.strip()

for line in read_file_good('large_file.txt'):
    process(line)
```

### 2. Generator Pipelines

```python
def read_logs(filename):
    """Stage 1: Read log file"""
    with open(filename) as f:
        for line in f:
            yield line

def parse_logs(lines):
    """Stage 2: Parse each line"""
    for line in lines:
        if 'ERROR' in line:
            yield line.strip()

def extract_timestamp(lines):
    """Stage 3: Extract timestamps"""
    for line in lines:
        timestamp = line.split()[0]
        yield timestamp

# Chain generators into pipeline
error_timestamps = extract_timestamp(
    parse_logs(
        read_logs('app.log')
    )
)

# Process one item at a time through entire pipeline
for timestamp in error_timestamps:
    print(timestamp)
```

### 3. Use itertools for Common Patterns

```python
from itertools import islice, count, cycle, chain, groupby, accumulate

# Infinite counter
counter = count(start=1, step=2)
print(next(counter))  # 1
print(next(counter))  # 3

# Take first n items
first_10 = islice(counter, 10)
print(list(first_10))

# Cycle through values
colors = cycle(['red', 'green', 'blue'])
print([next(colors) for _ in range(7)])  # Repeats colors

# Chain multiple iterators
combined = chain([1, 2, 3], [4, 5, 6], [7, 8, 9])
print(list(combined))  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Group by key
data = [('A', 1), ('A', 2), ('B', 3), ('B', 4)]
for key, group in groupby(data, key=lambda x: x[0]):
    print(f"{key}: {list(group)}")

# Accumulate (running totals)
numbers = [1, 2, 3, 4, 5]
running_sum = accumulate(numbers)
print(list(running_sum))  # [1, 3, 6, 10, 15]
```

### 4. Generator Methods: send(), throw(), close()

```python
def echo_generator():
    """Generator that can receive values"""
    value = None
    while True:
        try:
            value = yield value
        except GeneratorExit:
            print("Generator closing")
            break
        except Exception as e:
            print(f"Exception received: {e}")

gen = echo_generator()
next(gen)  # Prime the generator

print(gen.send("Hello"))  # Send value into generator
print(gen.send("World"))

gen.throw(ValueError("Test error"))  # Inject exception
gen.close()  # Close generator
```

## ❌ Common Mistakes

### 1. Consuming Generator Multiple Times

```python
# WRONG - Generators are exhausted after one iteration
gen = (x**2 for x in range(5))
print(list(gen))  # [0, 1, 4, 9, 16]
print(list(gen))  # [] - Generator exhausted!

# CORRECT - Create new generator or use list
def make_gen():
    return (x**2 for x in range(5))

gen1 = make_gen()
gen2 = make_gen()
print(list(gen1))  # Works
print(list(gen2))  # Works

# Or convert to list (if memory allows)
data = list(make_gen())
print(data)  # Can use multiple times
print(data)
```

### 2. Forgetting to Iterate

```python
# WRONG - Returns generator object, not values
def get_numbers():
    for i in range(5):
        yield i

result = get_numbers()
print(result)  # <generator object> - NOT the values!

# CORRECT - Iterate through generator
for num in get_numbers():
    print(num)

# Or convert to list
numbers = list(get_numbers())
```

### 3. Modifying Sequence During Iteration

```python
# WRONG - Modifying list while iterating
numbers = [1, 2, 3, 4, 5]
for num in numbers:
    if num % 2 == 0:
        numbers.remove(num)  # Skips elements!

# CORRECT - Use list comprehension or create new list
numbers = [1, 2, 3, 4, 5]
numbers = [num for num in numbers if num % 2 != 0]
```

### 4. Returning vs Yielding

```python
# WRONG - Returns generator on first call, then returns None
def broken_generator(n):
    if n <= 0:
        return  # OK - stops generator
    for i in range(n):
        yield i
    return "Done"  # Generator can have return value (Python 3.3+)

# StopIteration will contain "Done" as value
gen = broken_generator(3)
try:
    while True:
        print(next(gen))
except StopIteration as e:
    print(f"Return value: {e.value}")  # "Done"
```

## 🔐 Security Considerations

### 1. Resource Cleanup with Generators

```python
class DatabaseConnection:
    """Properly close resources in generators"""
    
    def __init__(self, db_name):
        self.db_name = db_name
        self.connection = None
    
    def __enter__(self):
        print(f"Opening {self.db_name}")
        self.connection = f"Connection to {self.db_name}"
        return self
    
    def __exit__(self, *args):
        print(f"Closing {self.db_name}")
        self.connection = None

def fetch_users():
    """Generator that properly closes resources"""
    with DatabaseConnection("users.db") as db:
        for i in range(5):
            yield f"User {i} from {db.connection}"
    # Connection automatically closed after generator exhausts

# Even if generator is not fully consumed
gen = fetch_users()
print(next(gen))
print(next(gen))
# Generator garbage collected → __exit__ called
del gen
```

### 2. Handling Exceptions in Generators

```python
def safe_data_processor(data_source):
    """Handle exceptions within generator"""
    try:
        for item in data_source:
            try:
                # Process item
                if item < 0:
                    raise ValueError(f"Negative value: {item}")
                yield item * 2
            except ValueError as e:
                # Log error but continue processing
                print(f"Skipping invalid item: {e}")
                continue
    finally:
        # Cleanup code - always runs
        print("Cleanup complete")

data = [1, 2, -3, 4, -5, 6]
for result in safe_data_processor(data):
    print(result)
```

## 🚀 Performance Optimization

### 1. Generator vs List Performance

```python
import time
import sys

def measure_time(func):
    start = time.time()
    result = func()
    end = time.time()
    return result, end - start

# List approach
def list_approach():
    return sum([x**2 for x in range(1_000_000)])

# Generator approach
def generator_approach():
    return sum(x**2 for x in range(1_000_000))

result1, time1 = measure_time(list_approach)
result2, time2 = measure_time(generator_approach)

print(f"List: {time1:.4f}s")
print(f"Generator: {time2:.4f}s")
print(f"Generator is {time1/time2:.2f}x faster")
```

### 2. Memory-Efficient Data Processing

```python
def process_large_file(filename):
    """Process file without loading into memory"""
    def read_chunks(file_obj, chunk_size=8192):
        """Read file in chunks"""
        while True:
            chunk = file_obj.read(chunk_size)
            if not chunk:
                break
            yield chunk
    
    with open(filename, 'rb') as f:
        for chunk in read_chunks(f):
            # Process chunk
            process_chunk(chunk)

# CSV processing
import csv

def read_csv_generator(filename):
    """Read CSV row by row"""
    with open(filename) as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield row

# Process millions of rows without memory issues
for row in read_csv_generator('huge_data.csv'):
    if row['status'] == 'active':
        process_user(row)
```

### 3. Parallel Processing with Generators

```python
from multiprocessing import Pool
from itertools import islice

def process_item(item):
    """CPU-intensive processing"""
    return item ** 2

def batch_generator(iterable, batch_size):
    """Yield batches from iterable"""
    iterator = iter(iterable)
    while True:
        batch = list(islice(iterator, batch_size))
        if not batch:
            break
        yield batch

def parallel_process(data, workers=4):
    """Process data in parallel using generators"""
    with Pool(workers) as pool:
        for batch in batch_generator(data, batch_size=1000):
            results = pool.map(process_item, batch)
            for result in results:
                yield result

# Process large dataset in parallel
large_data = range(1_000_000)
for result in parallel_process(large_data):
    # Results available as they're computed
    pass
```

## 🧪 Code Examples

### 1. Fibonacci Generator

```python
def fibonacci():
    """Infinite Fibonacci sequence"""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# Get first 10 Fibonacci numbers
from itertools import islice
print(list(islice(fibonacci(), 10)))
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### 2. Tree Traversal Generator

```python
class Node:
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right

def inorder_traversal(node):
    """In-order tree traversal using generator"""
    if node:
        yield from inorder_traversal(node.left)
        yield node.value
        yield from inorder_traversal(node.right)

# Build tree
root = Node(1,
    Node(2, Node(4), Node(5)),
    Node(3, Node(6), Node(7))
)

# Traverse
for value in inorder_traversal(root):
    print(value)  # 4, 2, 5, 1, 6, 3, 7
```

### 3. Sliding Window Generator

```python
from collections import deque

def sliding_window(iterable, window_size):
    """Generate sliding windows over iterable"""
    iterator = iter(iterable)
    window = deque(maxlen=window_size)
    
    # Fill initial window
    for _ in range(window_size):
        try:
            window.append(next(iterator))
        except StopIteration:
            return
    
    yield list(window)
    
    # Slide window
    for item in iterator:
        window.append(item)
        yield list(window)

# Usage
data = [1, 2, 3, 4, 5, 6, 7, 8]
for window in sliding_window(data, 3):
    print(window)
# [1, 2, 3]
# [2, 3, 4]
# [3, 4, 5]
# ...
```

### 4. Generator-Based State Machine

```python
def state_machine():
    """Simple state machine using generator"""
    state = "START"
    
    while True:
        if state == "START":
            print("Starting...")
            command = yield "Waiting for input"
            state = "PROCESSING" if command == "start" else "START"
        
        elif state == "PROCESSING":
            print("Processing...")
            command = yield "Processing data"
            state = "COMPLETE" if command == "finish" else "PROCESSING"
        
        elif state == "COMPLETE":
            print("Complete!")
            yield "Done"
            break

sm = state_machine()
print(next(sm))  # Prime generator
print(sm.send("start"))
print(sm.send("finish"))
```

### 5. Cooperative Multitasking with Generators

```python
def task(name, steps):
    """Simulates a task with multiple steps"""
    for step in range(steps):
        print(f"{name}: Step {step + 1}/{steps}")
        yield  # Yield control

def scheduler(tasks):
    """Schedule multiple tasks cooperatively"""
    while tasks:
        for task in list(tasks):
            try:
                next(task)
            except StopIteration:
                tasks.remove(task)

# Run multiple tasks concurrently
tasks = [
    task("Task A", 3),
    task("Task B", 5),
    task("Task C", 2)
]

scheduler(tasks)
"""Output:
Task A: Step 1/3
Task B: Step 1/5
Task C: Step 1/2
Task A: Step 2/3
Task B: Step 2/5
Task C: Step 2/2
Task A: Step 3/3
Task B: Step 3/5
Task B: Step 4/5
Task B: Step 5/5
"""
```

## 🏗️ Real-World Use Cases

### 1. Database Pagination

```python
def fetch_users_paginated(page_size=100):
    """Fetch users from database in pages"""
    offset = 0
    while True:
        # Query database
        users = User.objects.all()[offset:offset + page_size]
        
        if not users:
            break
        
        for user in users:
            yield user
        
        offset += page_size

# Process millions of users without memory issues
for user in fetch_users_paginated():
    send_email(user)
```

### 2. API Response Streaming

```python
import requests
import json

def stream_api_responses(url):
    """Stream large API responses"""
    response = requests.get(url, stream=True)
    
    for line in response.iter_lines():
        if line:
            data = json.loads(line)
            yield data

# Process API data as it arrives
for item in stream_api_responses('https://api.example.com/stream'):
    process_realtime_data(item)
```

### 3. Log File Analysis

```python
def analyze_logs(log_files):
    """Analyze multiple log files efficiently"""
    for log_file in log_files:
        with open(log_file) as f:
            for line in f:
                if 'ERROR' in line or 'CRITICAL' in line:
                    yield {
                        'file': log_file,
                        'line': line.strip(),
                        'timestamp': extract_timestamp(line)
                    }

# Analyze logs from multiple files
error_logs = analyze_logs(['app1.log', 'app2.log', 'app3.log'])
for error in error_logs:
    alert_ops_team(error)
```

### 4. Data ETL Pipeline

```python
def extract_from_csv(filename):
    """Extract data from CSV"""
    import csv
    with open(filename) as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield row

def transform_data(records):
    """Transform records"""
    for record in records:
        # Data cleaning
        record['email'] = record['email'].lower().strip()
        record['age'] = int(record['age'])
        
        # Data validation
        if '@' not in record['email']:
            continue
        
        yield record

def load_to_database(records, batch_size=1000):
    """Load records to database in batches"""
    batch = []
    for record in records:
        batch.append(record)
        
        if len(batch) >= batch_size:
            User.objects.bulk_create([
                User(**record) for record in batch
            ])
            batch.clear()
            yield f"Loaded {batch_size} records"
    
    # Load remaining records
    if batch:
        User.objects.bulk_create([
            User(**record) for record in batch
        ])
        yield f"Loaded {len(batch)} records"

# ETL Pipeline
pipeline = load_to_database(
    transform_data(
        extract_from_csv('users.csv')
    )
)

for status in pipeline:
    print(status)
```

### 5. Web Scraping with Generators

```python
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin, urlparse

def crawl_website(start_url, max_pages=100):
    """Crawl website and yield pages"""
    visited = set()
    to_visit = [start_url]
    
    while to_visit and len(visited) < max_pages:
        url = to_visit.pop(0)
        
        if url in visited:
            continue
        
        try:
            response = requests.get(url, timeout=5)
            visited.add(url)
            
            soup = BeautifulSoup(response.content, 'html.parser')
            
            # Find new links
            for link in soup.find_all('a', href=True):
                absolute_url = urljoin(url, link['href'])
                if urlparse(absolute_url).netloc == urlparse(start_url).netloc:
                    to_visit.append(absolute_url)
            
            yield {
                'url': url,
                'title': soup.title.string if soup.title else None,
                'content': soup.get_text()
            }
        
        except Exception as e:
            print(f"Error crawling {url}: {e}")

# Crawl website
for page in crawl_website('https://example.com', max_pages=50):
    index_page(page)
```

## ❓ Interview Questions

### Q1: What's the difference between a generator and a regular function?

**Answer**:
- **Regular function**: Uses `return`, executes completely, returns once
- **Generator function**: Uses `yield`, can pause/resume, returns multiple times
- Generators maintain state between calls
- Generators are memory-efficient (lazy evaluation)

### Q2: How do generators save memory?

**Answer**: Generators don't store all values in memory. They compute and yield one value at a time, maintaining only the generator state (local variables and execution position). A list of 1 million items might use 8MB+, while a generator uses ~200 bytes regardless of sequence length.

### Q3: What's the difference between `yield` and `return`?

**Answer**:
```python
def with_return():
    return 1
    return 2  # Never executed

def with_yield():
    yield 1
    yield 2  # Both execute

print(with_return())  # 1
print(list(with_yield()))  # [1, 2]
```

### Q4: What is `yield from` and when would you use it?

**Answer**: `yield from` delegates to another generator, yielding all its values:

```python
def inner_gen():
    yield 1
    yield 2

def outer_gen():
    yield from inner_gen()  # Delegates to inner_gen
    yield 3

list(outer_gen())  # [1, 2, 3]

# Equivalent to:
def outer_gen_manual():
    for value in inner_gen():
        yield value
    yield 3
```

### Q5: Can you send values into a generator? How?

**Answer**: Yes, using `.send()` method:

```python
def accumulator():
    total = 0
    while True:
        value = yield total
        if value is not None:
            total += value

acc = accumulator()
next(acc)  # Prime the generator (get to first yield)
print(acc.send(10))  # 10
print(acc.send(5))   # 15
print(acc.send(3))   # 18
```

### Q6: What happens when a generator is exhausted?

**Answer**: It raises `StopIteration` exception:

```python
gen = (x for x in range(2))
print(next(gen))  # 0
print(next(gen))  # 1
print(next(gen))  # StopIteration exception
```

### Q7: How would you implement `range()` as a generator?

**Answer**:
```python
def my_range(start, stop=None, step=1):
    if stop is None:
        start, stop = 0, start
    
    current = start
    while (step > 0 and current < stop) or (step < 0 and current > stop):
        yield current
        current += step

list(my_range(5))  # [0, 1, 2, 3, 4]
list(my_range(2, 10, 2))  # [2, 4, 6, 8]
```

### Q8: What's the difference between a generator expression and a list comprehension?

**Answer**:
```python
# List comprehension - creates entire list in memory
squares_list = [x**2 for x in range(1000)]  # ~8KB

# Generator expression - computes on demand
squares_gen = (x**2 for x in range(1000))   # ~200 bytes

# List can be reused multiple times
print(sum(squares_list))
print(sum(squares_list))  # Works

# Generator exhausts after one use
print(sum(squares_gen))
print(sum(squares_gen))  # Returns 0 (exhausted)
```

## 🧩 Design Patterns

### Iterator Pattern

```python
class BookCollection:
    """Iterable collection of books"""
    
    def __init__(self):
        self._books = []
    
    def add_book(self, book):
        self._books.append(book)
    
    def __iter__(self):
        """Return iterator for books"""
        return iter(self._books)
    
    def reverse_iterator(self):
        """Custom iterator for reverse traversal"""
        for book in reversed(self._books):
            yield book
    
    def filter_iterator(self, condition):
        """Custom iterator with filter"""
        for book in self._books:
            if condition(book):
                yield book

# Usage
collection = BookCollection()
collection.add_book({"title": "Python Tricks", "year": 2017})
collection.add_book({"title": "Fluent Python", "year": 2015})

# Standard iteration
for book in collection:
    print(book['title'])

# Reverse iteration
for book in collection.reverse_iterator():
    print(book['title'])

# Filtered iteration
for book in collection.filter_iterator(lambda b: b['year'] >= 2017):
    print(book['title'])
```

---

## 📚 Summary

Generators and iterators are essential for:
- Memory-efficient data processing
- Lazy evaluation and on-demand computation
- Streaming and pipeline architectures
- Handling large datasets and infinite sequences

**Key Takeaways**:
1. Use generators for large datasets
2. Generator expressions for simple cases
3. `yield from` for generator delegation
4. Generators are single-use (create new or convert to list)
5. Leverage `itertools` for common patterns
6. Generators enable functional, composable pipelines
