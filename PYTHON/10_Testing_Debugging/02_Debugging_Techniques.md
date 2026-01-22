# 🔍 Debugging Techniques and Tools

## Overview

Effective debugging skills are essential for quickly identifying and fixing issues. Understanding debugging tools and techniques saves time and improves code quality.

---

## Python Debugger (pdb)

### Interactive Debugging

```python
import pdb

# Set breakpoint in code
def calculate_total(items):
    """Calculate total with debugging"""
    total = 0

    for item in items:
        # Start debugger here
        pdb.set_trace()  # Python < 3.7
        # breakpoint()   # Python >= 3.7 (preferred)

        total += item['price'] * item['quantity']

    return total

# Common pdb commands:
"""
n (next)       - Execute next line
s (step)       - Step into function
c (continue)   - Continue execution
l (list)       - Show source code
p variable     - Print variable value
pp variable    - Pretty print
w (where)      - Show stack trace
u (up)         - Move up stack frame
d (down)       - Move down stack frame
b line_number  - Set breakpoint
cl             - Clear breakpoints
q (quit)       - Exit debugger
"""

# Conditional breakpoint
def process_users(users):
    """Debug specific user"""
    for user in users:
        # Break only for specific user
        if user['id'] == 123:
            breakpoint()

        process_user(user)

# Post-mortem debugging
def risky_function():
    """Debug after crash"""
    try:
        result = 10 / 0
    except Exception:
        import pdb
        pdb.post_mortem()  # Debug at exception point

# Context manager for debugging
from contextlib import contextmanager

@contextmanager
def debug_on_error():
    """Enter debugger on exception"""
    try:
        yield
    except Exception:
        import pdb
        pdb.post_mortem()
        raise

# Usage
with debug_on_error():
    # Code that might fail
    risky_operation()
```

### Advanced pdb Usage

```python
# pdb with IPython (ipdb) - enhanced debugger
import ipdb

def complex_function(data):
    """Debug with ipdb"""
    # Better than pdb:
    # - Tab completion
    # - Syntax highlighting
    # - Better output
    ipdb.set_trace()

    result = process(data)
    return result

# Remote debugging
import pdb
import sys

class RemotePdb(pdb.Pdb):
    """Remote debugger via telnet"""

    def __init__(self, host='127.0.0.1', port=4444):
        import socket

        # Create socket
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.sock.bind((host, port))
        self.sock.listen(1)

        print(f"Remote debugger listening on {host}:{port}")

        # Wait for connection
        (client, addr) = self.sock.accept()

        # Use socket for I/O
        handle = client.makefile('rw')
        pdb.Pdb.__init__(self, completekey='tab', stdin=handle, stdout=handle)

    def __del__(self):
        self.sock.close()

# Use in code
# RemotePdb().set_trace()
# Then: telnet localhost 4444
```

---

## Print Debugging

### Effective Print Statements

```python
import sys
import json
from pprint import pprint

# ❌ BAD: Simple print
def process_data(data):
    print(data)  # Not informative
    result = transform(data)
    print(result)
    return result

# ✅ GOOD: Informative prints
def process_data_better(data):
    """Process data with better debugging"""
    print(f"[DEBUG] Input data: {data}", file=sys.stderr)

    result = transform(data)

    print(f"[DEBUG] Result: {result}", file=sys.stderr)
    print(f"[DEBUG] Type: {type(result)}", file=sys.stderr)

    return result

# Pretty printing
data = {
    'users': [
        {'id': 1, 'name': 'Alice', 'scores': [95, 87, 91]},
        {'id': 2, 'name': 'Bob', 'scores': [88, 92, 85]}
    ]
}

# Regular print (hard to read)
print(data)

# Pretty print (formatted)
pprint(data)

# JSON formatting
print(json.dumps(data, indent=2))

# Debug print decorator
def debug_print(func):
    """Decorator to print function calls"""
    import functools

    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print(f"\n[CALL] {func.__name__}")
        print(f"  Args: {args}")
        print(f"  Kwargs: {kwargs}")

        result = func(*args, **kwargs)

        print(f"[RETURN] {func.__name__} -> {result}")

        return result

    return wrapper

@debug_print
def calculate(x, y):
    return x + y

# Stack trace printing
import traceback

def print_stack():
    """Print current stack trace"""
    print("Current stack:")
    traceback.print_stack()

# Get stack as string
def get_stack_info():
    """Get stack trace as string"""
    return ''.join(traceback.format_stack())
```

---

## Logging for Debugging

### Debug Logging Patterns

```python
import logging

# Configure debug logging
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - [%(filename)s:%(lineno)d] - %(message)s'
)

logger = logging.getLogger(__name__)

# Log function entry/exit
def process_order(order_id):
    """Process order with logging"""
    logger.debug(f"process_order called with order_id={order_id}")

    try:
        # Processing logic
        order = get_order(order_id)
        logger.debug(f"Order retrieved: {order}")

        result = validate_order(order)
        logger.debug(f"Validation result: {result}")

        logger.debug(f"process_order completed successfully")
        return result

    except Exception as e:
        logger.exception(f"process_order failed for order_id={order_id}")
        raise

# Log with context
class LogContext:
    """Context manager for debug logging"""

    def __init__(self, operation):
        self.operation = operation

    def __enter__(self):
        logger.debug(f"Starting: {self.operation}")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            logger.debug(f"Failed: {self.operation}", exc_info=True)
        else:
            logger.debug(f"Completed: {self.operation}")

# Usage
with LogContext("Process payment"):
    process_payment(order)

# Verbose debugging decorator
def verbose_debug(func):
    """Decorator for verbose debugging"""
    import functools
    import inspect

    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # Get function signature
        sig = inspect.signature(func)
        bound_args = sig.bind(*args, **kwargs)
        bound_args.apply_defaults()

        # Log call with arguments
        logger.debug(
            f"{func.__name__} called",
            extra={
                'function': func.__name__,
                'arguments': dict(bound_args.arguments)
            }
        )

        try:
            result = func(*args, **kwargs)

            # Log success
            logger.debug(
                f"{func.__name__} returned",
                extra={
                    'function': func.__name__,
                    'result': result
                }
            )

            return result

        except Exception as e:
            # Log failure
            logger.exception(
                f"{func.__name__} raised exception",
                extra={
                    'function': func.__name__,
                    'exception': str(e)
                }
            )
            raise

    return wrapper
```

---

## Memory Debugging

### Finding Memory Leaks

```python
import gc
import sys
import tracemalloc

# Track memory allocations
def debug_memory():
    """Debug memory usage"""
    # Start tracking
    tracemalloc.start()

    # Code to profile
    data = [i for i in range(1_000_000)]

    # Take snapshot
    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics('lineno')

    print("Top 10 memory allocations:")
    for stat in top_stats[:10]:
        print(stat)

    # Stop tracking
    tracemalloc.stop()

# Compare memory snapshots
def track_memory_growth():
    """Track memory growth"""
    tracemalloc.start()

    # Snapshot 1
    snapshot1 = tracemalloc.take_snapshot()

    # Code that might leak
    leaky_operation()

    # Snapshot 2
    snapshot2 = tracemalloc.take_snapshot()

    # Compare
    top_stats = snapshot2.compare_to(snapshot1, 'lineno')

    print("Top memory increases:")
    for stat in top_stats[:10]:
        print(stat)

# Find circular references
def find_circular_refs():
    """Find objects with circular references"""
    gc.collect()

    for obj in gc.garbage:
        print(f"Garbage: {type(obj)}")
        print(f"Referrers: {gc.get_referrers(obj)}")

# Track object counts
import sys
from collections import Counter

def count_objects():
    """Count objects by type"""
    counts = Counter()

    for obj in gc.get_objects():
        obj_type = type(obj).__name__
        counts[obj_type] += 1

    print("Top 10 object types:")
    for obj_type, count in counts.most_common(10):
        print(f"{obj_type}: {count}")

# Monitor memory usage
import psutil
import os

def monitor_memory():
    """Monitor process memory"""
    process = psutil.Process(os.getpid())

    mem_info = process.memory_info()
    print(f"RSS: {mem_info.rss / 1024 / 1024:.2f} MB")
    print(f"VMS: {mem_info.vms / 1024 / 1024:.2f} MB")

    # Memory percentage
    mem_percent = process.memory_percent()
    print(f"Memory %: {mem_percent:.2f}%")
```

---

## Performance Debugging

### Profiling and Timing

```python
import time
import cProfile
import pstats
from functools import wraps

# Simple timing decorator
def timeit(func):
    """Measure execution time"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start

        print(f"{func.__name__} took {elapsed:.4f} seconds")
        return result

    return wrapper

@timeit
def slow_function():
    time.sleep(1)
    return "Done"

# Profile function
def profile_function(func):
    """Profile function execution"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()

        result = func(*args, **kwargs)

        profiler.disable()
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(20)

        return result

    return wrapper

# Line-by-line timing
from line_profiler import LineProfiler

def profile_lines(func):
    """Profile line by line"""
    profiler = LineProfiler()
    profiler.add_function(func)

    profiler.enable()
    result = func()
    profiler.disable()

    profiler.print_stats()
    return result

# Compare implementations
def benchmark_implementations():
    """Compare different implementations"""
    import timeit

    # Implementation 1
    time1 = timeit.timeit(
        'sum([i for i in range(1000)])',
        number=10000
    )

    # Implementation 2
    time2 = timeit.timeit(
        'sum(i for i in range(1000))',
        number=10000
    )

    print(f"List comp: {time1:.4f}s")
    print(f"Generator: {time2:.4f}s")
    print(f"Faster: {'Generator' if time2 < time1 else 'List comp'}")
```

---

## Debugging Async Code

### Async Debugging Techniques

```python
import asyncio
import logging

logger = logging.getLogger(__name__)

# Debug async functions
async def debug_async():
    """Debug async code"""
    logger.debug("Async function started")

    # Breakpoint in async code
    import pdb; pdb.set_trace()

    await asyncio.sleep(1)

    logger.debug("Async function completed")

# Track pending tasks
def print_pending_tasks():
    """Print all pending async tasks"""
    tasks = asyncio.all_tasks()

    print(f"Pending tasks: {len(tasks)}")
    for task in tasks:
        print(f"  - {task.get_name()}: {task}")

        # Get task stack
        if not task.done():
            task.print_stack()

# Timeout debugging
async def debug_timeout():
    """Debug timeout issues"""
    try:
        await asyncio.wait_for(
            slow_async_operation(),
            timeout=5.0
        )
    except asyncio.TimeoutError:
        logger.error("Operation timed out")
        print_pending_tasks()
        raise

# Debug event loop
def debug_event_loop():
    """Debug event loop status"""
    loop = asyncio.get_event_loop()

    print(f"Loop running: {loop.is_running()}")
    print(f"Loop closed: {loop.is_closed()}")

    # Enable debug mode
    loop.set_debug(True)

    # Slow callback warning
    loop.slow_callback_duration = 0.1  # Warn if > 100ms

# Trace coroutine execution
import sys

def trace_coroutines(frame, event, arg):
    """Trace coroutine execution"""
    if event == 'call':
        code = frame.f_code
        if code.co_flags & 0x100:  # CO_COROUTINE
            print(f"Coroutine called: {code.co_name}")

    return trace_coroutines

# Enable tracing
sys.settrace(trace_coroutines)
```

---

## Debugging Tools

### IDE Debuggers

```python
"""
Visual Studio Code:
1. Set breakpoints (click left of line number)
2. F5 to start debugging
3. F10 - Step over
4. F11 - Step into
5. Shift+F11 - Step out
6. View variables in left panel
7. Add watch expressions
8. Conditional breakpoints (right-click breakpoint)

PyCharm:
1. Ctrl+F8 - Toggle breakpoint
2. Shift+F9 - Start debugging
3. F8 - Step over
4. F7 - Step into
5. Shift+F8 - Step out
6. Evaluate expression (Alt+F8)
7. Inline debugging
8. Remote debugging support
"""

# Debug configuration (launch.json for VS Code)
debug_config = {
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Current File",
            "type": "python",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal",
            "justMyCode": False  # Step into libraries
        },
        {
            "name": "Python: FastAPI",
            "type": "python",
            "request": "launch",
            "module": "uvicorn",
            "args": [
                "main:app",
                "--reload"
            ],
            "jinja": True
        }
    ]
}
```

---

## Best Practices

### ✅ Do's:

1. **Use debugger** over print statements
2. **Write tests** before debugging
3. **Reproduce issue** consistently
4. **Isolate problem** with binary search
5. **Check assumptions** with assertions
6. **Use logging** for production
7. **Read error messages** carefully
8. **Check recent changes** first
9. **Use version control** to bisect
10. **Document solutions** for future

### ❌ Don'ts:

1. **Don't debug without** understanding code
2. **Don't make random** changes
3. **Don't skip error** messages
4. **Don't debug** without reproducing
5. **Don't leave debug** code in production
6. **Don't ignore warnings**
7. **Don't debug tired** (take breaks)

---

## Interview Questions

### Q1: Explain common pdb commands.

**Answer**:

- **n (next)**: Execute next line
- **s (step)**: Step into function
- **c (continue)**: Continue to next breakpoint
- **p variable**: Print variable
- **l (list)**: Show source code
- **w (where)**: Stack trace
- **b line**: Set breakpoint

### Q2: How do you debug memory leaks?

**Answer**:

- **tracemalloc**: Track allocations
- **objgraph**: Find reference cycles
- **gc.get_referrers()**: Who references object
- **Compare snapshots**: Before/after operation
- Look for: Global caches, circular refs, event handlers

### Q3: What's the difference between pdb and print debugging?

**Answer**:

- **pdb**: Interactive, inspect state, modify variables
- **print**: Simple, non-interactive, requires rerun
- **When pdb**: Complex logic, unknown state
- **When print**: Quick checks, logging

### Q4: How do you debug async code?

**Answer**:

- Use **breakpoint()** in async functions
- **asyncio.all_tasks()**: List pending tasks
- **loop.set_debug(True)**: Enable warnings
- **Print stack**: task.print_stack()
- Track timeouts with wait_for

### Q5: Explain post-mortem debugging.

**Answer**: Debug after crash:

- **pdb.post_mortem()**: Start debugger at exception
- Inspect state when error occurred
- No need to reproduce
- Useful for production crashes (save traceback)

---

## Summary

Debugging essentials:

- **pdb**: Interactive Python debugger
- **Print debugging**: Strategic print statements
- **Logging**: Production debugging
- **Memory**: tracemalloc, gc, objgraph
- **Performance**: cProfile, timeit
- **Async**: asyncio debugging tools
- **IDE debuggers**: Visual Studio Code, PyCharm

Debug systematically, not randomly! 🔍
