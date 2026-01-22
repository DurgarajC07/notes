# 📁 File I/O and Serialization

## Overview

Understanding file operations and serialization formats (JSON, CSV, pickle, binary) is essential for data persistence and integration.

---

## Basic File Operations

### Reading and Writing Files

```python
# Basic file reading - BAD (no automatic close)
file = open('data.txt', 'r')
content = file.read()
file.close()

# Context manager - GOOD (automatic close)
with open('data.txt', 'r') as file:
    content = file.read()
# File automatically closed

# Different reading methods
with open('data.txt', 'r') as file:
    # Read entire file
    content = file.read()

    # Read line by line
    file.seek(0)  # Reset to beginning
    for line in file:
        print(line.strip())

    # Read all lines as list
    file.seek(0)
    lines = file.readlines()

    # Read specific number of characters
    file.seek(0)
    chunk = file.read(100)

# Writing files
with open('output.txt', 'w') as file:
    file.write("Hello, World!\n")
    file.write("Second line\n")

# Append to file
with open('output.txt', 'a') as file:
    file.write("Appended line\n")

# Write multiple lines
lines = ["Line 1\n", "Line 2\n", "Line 3\n"]
with open('output.txt', 'w') as file:
    file.writelines(lines)

# File modes
"""
'r'  - Read (default)
'w'  - Write (truncate)
'a'  - Append
'r+' - Read and write
'w+' - Write and read (truncate)
'a+' - Append and read
'x'  - Exclusive creation (fail if exists)
'b'  - Binary mode (rb, wb, etc.)
't'  - Text mode (default)
"""
```

---

## Working with Paths

### Path Manipulation

```python
from pathlib import Path
import os

# Pathlib (modern, recommended)
path = Path('data/files/document.txt')

# Path properties
print(path.name)        # 'document.txt'
print(path.stem)        # 'document'
print(path.suffix)      # '.txt'
print(path.parent)      # Path('data/files')
print(path.parts)       # ('data', 'files', 'document.txt')

# Check existence
print(path.exists())    # False
print(path.is_file())   # False
print(path.is_dir())    # False

# Create directories
path.parent.mkdir(parents=True, exist_ok=True)

# Join paths
base = Path('data')
file_path = base / 'files' / 'document.txt'

# Iterate directory
data_dir = Path('data')
for file in data_dir.iterdir():
    if file.is_file():
        print(file.name)

# Glob pattern matching
for txt_file in data_dir.glob('**/*.txt'):
    print(txt_file)

# Read/write with pathlib
path = Path('data.txt')
content = path.read_text()
path.write_text("New content")

# Get absolute path
abs_path = path.resolve()

# Legacy os.path (still common)
file_path = os.path.join('data', 'files', 'document.txt')
print(os.path.basename(file_path))  # 'document.txt'
print(os.path.dirname(file_path))   # 'data/files'
print(os.path.exists(file_path))    # False

# Get file info
if os.path.exists('data.txt'):
    stat = os.stat('data.txt')
    print(f"Size: {stat.st_size} bytes")
    print(f"Modified: {stat.st_mtime}")
```

---

## JSON Serialization

### Working with JSON

```python
import json

# Serialize to JSON
data = {
    "name": "Alice",
    "age": 30,
    "active": True,
    "scores": [95, 87, 91],
    "address": {
        "city": "NYC",
        "zip": "10001"
    }
}

# Write to file
with open('data.json', 'w') as file:
    json.dump(data, file, indent=2)

# Convert to string
json_string = json.dumps(data, indent=2)
print(json_string)

# Read from file
with open('data.json', 'r') as file:
    loaded_data = json.load(file)

# Parse from string
parsed_data = json.loads(json_string)

# Custom JSON encoder for complex types
from datetime import datetime
from decimal import Decimal

class CustomEncoder(json.JSONEncoder):
    """Custom JSON encoder"""

    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        elif isinstance(obj, Decimal):
            return float(obj)
        elif hasattr(obj, '__dict__'):
            return obj.__dict__
        return super().default(obj)

# Usage
data = {
    "timestamp": datetime.now(),
    "amount": Decimal("99.99")
}

json_string = json.dumps(data, cls=CustomEncoder)

# Custom decoder
def custom_decoder(dct):
    """Custom JSON decoder"""
    if 'timestamp' in dct:
        dct['timestamp'] = datetime.fromisoformat(dct['timestamp'])
    if 'amount' in dct:
        dct['amount'] = Decimal(dct['amount'])
    return dct

loaded = json.loads(json_string, object_hook=custom_decoder)

# Pydantic for JSON (better type safety)
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
    email: str

# Serialize
user = User(name="Alice", age=30, email="alice@example.com")
json_data = user.model_dump_json(indent=2)

# Deserialize
loaded_user = User.model_validate_json(json_data)
```

---

## CSV Files

### CSV Operations

```python
import csv
from io import StringIO

# Writing CSV
data = [
    ["Name", "Age", "City"],
    ["Alice", "30", "NYC"],
    ["Bob", "25", "LA"],
    ["Charlie", "35", "Chicago"]
]

# Basic CSV writing
with open('data.csv', 'w', newline='') as file:
    writer = csv.writer(file)
    writer.writerows(data)

# Dict writer (better for structured data)
users = [
    {"name": "Alice", "age": 30, "city": "NYC"},
    {"name": "Bob", "age": 25, "city": "LA"}
]

with open('users.csv', 'w', newline='') as file:
    fieldnames = ['name', 'age', 'city']
    writer = csv.DictWriter(file, fieldnames=fieldnames)

    writer.writeheader()
    writer.writerows(users)

# Reading CSV
with open('data.csv', 'r') as file:
    reader = csv.reader(file)
    header = next(reader)  # Skip header

    for row in reader:
        print(f"{row[0]}: {row[1]} years, {row[2]}")

# Dict reader
with open('users.csv', 'r') as file:
    reader = csv.DictReader(file)

    for row in reader:
        print(f"{row['name']}: {row['age']} years")

# CSV with custom delimiter
with open('data.tsv', 'w', newline='') as file:
    writer = csv.writer(file, delimiter='\t')
    writer.writerows(data)

# Handle large CSV files (streaming)
def process_large_csv(filename):
    """Process CSV line by line"""
    with open(filename, 'r') as file:
        reader = csv.DictReader(file)

        for row in reader:
            # Process one row at a time
            yield row

# Usage
for user in process_large_csv('large_users.csv'):
    process_user(user)
```

---

## Pickle Serialization

### Python Object Serialization

```python
import pickle

# Serialize Python objects
data = {
    "numbers": [1, 2, 3],
    "nested": {"key": "value"},
    "set": {1, 2, 3},
    "tuple": (1, 2, 3)
}

# Write pickle
with open('data.pkl', 'wb') as file:
    pickle.dump(data, file)

# Read pickle
with open('data.pkl', 'rb') as file:
    loaded = pickle.load(file)

# Pickle multiple objects
with open('multi.pkl', 'wb') as file:
    pickle.dump(obj1, file)
    pickle.dump(obj2, file)
    pickle.dump(obj3, file)

with open('multi.pkl', 'rb') as file:
    obj1 = pickle.load(file)
    obj2 = pickle.load(file)
    obj3 = pickle.load(file)

# Pickle custom classes
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        return f"User({self.name}, {self.age})"

user = User("Alice", 30)

# Serialize
with open('user.pkl', 'wb') as file:
    pickle.dump(user, file)

# Deserialize
with open('user.pkl', 'rb') as file:
    loaded_user = pickle.load(file)

# ⚠️ Security warning
"""
NEVER unpickle data from untrusted sources!
Pickle can execute arbitrary code during deserialization.
Use JSON for untrusted data.
"""

# Pickle protocol versions
with open('data.pkl', 'wb') as file:
    # Protocol 5 (Python 3.8+) for large data
    pickle.dump(data, file, protocol=pickle.HIGHEST_PROTOCOL)
```

---

## Binary Files

### Binary File Operations

```python
# Write binary data
data = bytes([0x48, 0x65, 0x6C, 0x6C, 0x6F])  # "Hello"

with open('data.bin', 'wb') as file:
    file.write(data)

# Read binary data
with open('data.bin', 'rb') as file:
    content = file.read()
    print(content)  # b'Hello'

# Read images
with open('image.jpg', 'rb') as file:
    image_data = file.read()

# Copy binary file
def copy_binary_file(source, destination):
    """Copy file in chunks"""
    chunk_size = 4096  # 4KB chunks

    with open(source, 'rb') as src:
        with open(destination, 'wb') as dst:
            while True:
                chunk = src.read(chunk_size)
                if not chunk:
                    break
                dst.write(chunk)

# Struct for binary data
import struct

# Pack data to bytes
data = struct.pack('i', 42)  # Integer
data = struct.pack('f', 3.14)  # Float
data = struct.pack('if', 42, 3.14)  # Int and float

# Unpack from bytes
value = struct.unpack('i', data)[0]

# Read structured binary file
def read_binary_records(filename):
    """Read fixed-size binary records"""
    record_format = 'i10sf'  # int, 10-char string, float
    record_size = struct.calcsize(record_format)

    with open(filename, 'rb') as file:
        while True:
            record_data = file.read(record_size)
            if not record_data:
                break

            id, name, score = struct.unpack(record_format, record_data)
            name = name.decode('utf-8').strip('\x00')

            yield {"id": id, "name": name, "score": score}
```

---

## Streaming Large Files

### Memory-Efficient File Processing

```python
# Process large file line by line
def process_large_file(filename):
    """Memory-efficient line processing"""
    with open(filename, 'r') as file:
        for line in file:
            # Process one line at a time
            yield line.strip()

# Count lines without loading entire file
def count_lines(filename):
    """Count lines efficiently"""
    count = 0
    with open(filename, 'r') as file:
        for _ in file:
            count += 1
    return count

# Read file in chunks
def read_in_chunks(filename, chunk_size=1024):
    """Read file in chunks"""
    with open(filename, 'r') as file:
        while True:
            chunk = file.read(chunk_size)
            if not chunk:
                break
            yield chunk

# Process CSV in batches
def process_csv_in_batches(filename, batch_size=1000):
    """Process CSV in batches"""
    import csv

    batch = []
    with open(filename, 'r') as file:
        reader = csv.DictReader(file)

        for row in reader:
            batch.append(row)

            if len(batch) >= batch_size:
                yield batch
                batch = []

        # Yield remaining
        if batch:
            yield batch

# Usage
for batch in process_csv_in_batches('large_data.csv'):
    # Process batch (e.g., insert to database)
    db.bulk_insert(batch)
```

---

## Best Practices

### ✅ Do's:

1. **Use context managers** (with statement)
2. **Use pathlib** for path operations
3. **Stream large files** don't load all
4. **Handle encoding** explicitly (utf-8)
5. **Use JSON** for config and APIs
6. **Use CSV** for tabular data
7. **Close files** properly
8. **Check file existence** before operations

### ❌ Don'ts:

1. **Don't forget to close** files
2. **Don't use pickle** for untrusted data
3. **Don't load huge files** into memory
4. **Don't ignore encoding** errors
5. **Don't hardcode paths** (use os.path.join)
6. **Don't pickle for long-term** storage

---

## Interview Questions

### Q1: What's the difference between 'r', 'w', and 'a' modes?

**Answer**:

- **'r'**: Read only (error if not exists)
- **'w'**: Write (creates/truncates file)
- **'a'**: Append (creates if not exists, writes at end)

### Q2: Why use context managers for files?

**Answer**:

- **Automatic cleanup**: File closed even if exception
- **Cleaner code**: No explicit close()
- **Resource management**: Prevents file descriptor leaks
- **Exception safe**: Guaranteed cleanup

### Q3: When to use JSON vs Pickle?

**Answer**:

- **JSON**: Human-readable, cross-language, untrusted data, config
- **Pickle**: Python-specific, complex objects, trusted data only
  Never pickle untrusted data (security risk).

### Q4: How to process very large files efficiently?

**Answer**:

- **Stream line-by-line**: Don't load entire file
- **Read in chunks**: Fixed-size buffers
- **Use generators**: Lazy evaluation
- **Process in batches**: For database inserts
- **Memory profiling**: Monitor usage

### Q5: Explain pathlib advantages over os.path.

**Answer**:

- **Object-oriented**: Path objects vs strings
- **Operator overloading**: Use `/` to join paths
- **Built-in methods**: read_text(), write_text()
- **More intuitive**: .name, .stem, .suffix properties
- **Cross-platform**: Handles OS differences

---

## Summary

File I/O essentials:

- **Context managers**: Automatic cleanup
- **Pathlib**: Modern path handling
- **JSON**: Human-readable serialization
- **CSV**: Tabular data
- **Pickle**: Python objects (trusted only)
- **Streaming**: Memory-efficient processing
- **Binary**: Structured data with struct

Handle files safely and efficiently! 📁
