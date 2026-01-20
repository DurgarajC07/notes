# 🔄 Sorting and Searching Algorithms

## Overview

Understanding fundamental sorting and searching algorithms, their time/space complexity, and when to use each in production Python code.

---

## Sorting Algorithms

### Bubble Sort

```python
from typing import List

def bubble_sort(arr: List[int]) -> List[int]:
    """
    Bubble sort - Simple but inefficient

    Time: O(n²) - worst and average
    Space: O(1) - in-place sorting

    Use: Educational purposes, nearly sorted data
    """
    n = len(arr)

    for i in range(n):
        swapped = False

        for j in range(0, n - i - 1):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
                swapped = True

        # Optimization: stop if no swaps (already sorted)
        if not swapped:
            break

    return arr

# Example
arr = [64, 34, 25, 12, 22, 11, 90]
print(bubble_sort(arr))  # [11, 12, 22, 25, 34, 64, 90]

# Time complexity:
# Best: O(n) when already sorted
# Average: O(n²)
# Worst: O(n²)
```

### Quick Sort

```python
def quick_sort(arr: List[int]) -> List[int]:
    """
    Quick sort - Divide and conquer

    Time: O(n log n) average, O(n²) worst
    Space: O(log n) for recursion stack

    Use: General purpose, good cache performance
    """
    if len(arr) <= 1:
        return arr

    # Choose pivot (middle element for better average case)
    pivot = arr[len(arr) // 2]

    # Partition into three parts
    left = [x for x in arr if x < pivot]
    middle = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]

    # Recursively sort and combine
    return quick_sort(left) + middle + quick_sort(right)

# In-place version (more memory efficient)
def quick_sort_inplace(arr: List[int], low: int = 0, high: int = None) -> List[int]:
    """Quick sort with in-place partitioning"""
    if high is None:
        high = len(arr) - 1

    if low < high:
        # Partition and get pivot index
        pi = partition(arr, low, high)

        # Sort left and right of pivot
        quick_sort_inplace(arr, low, pi - 1)
        quick_sort_inplace(arr, pi + 1, high)

    return arr

def partition(arr: List[int], low: int, high: int) -> int:
    """Partition array around pivot"""
    pivot = arr[high]
    i = low - 1

    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1

# Example
arr = [10, 7, 8, 9, 1, 5]
print(quick_sort(arr))  # [1, 5, 7, 8, 9, 10]
```

### Merge Sort

```python
def merge_sort(arr: List[int]) -> List[int]:
    """
    Merge sort - Stable divide and conquer

    Time: O(n log n) - always
    Space: O(n) - needs auxiliary space

    Use: When stability matters, linked lists
    """
    if len(arr) <= 1:
        return arr

    # Divide
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])

    # Conquer (merge)
    return merge(left, right)

def merge(left: List[int], right: List[int]) -> List[int]:
    """Merge two sorted arrays"""
    result = []
    i = j = 0

    # Merge while both have elements
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1

    # Add remaining elements
    result.extend(left[i:])
    result.extend(right[j:])

    return result

# Example
arr = [38, 27, 43, 3, 9, 82, 10]
print(merge_sort(arr))  # [3, 9, 10, 27, 38, 43, 82]
```

### Heap Sort

```python
def heap_sort(arr: List[int]) -> List[int]:
    """
    Heap sort - Using binary heap

    Time: O(n log n) - always
    Space: O(1) - in-place

    Use: When O(1) space needed, guaranteed O(n log n)
    """
    n = len(arr)

    # Build max heap
    for i in range(n // 2 - 1, -1, -1):
        heapify(arr, n, i)

    # Extract elements from heap
    for i in range(n - 1, 0, -1):
        # Move current root to end
        arr[0], arr[i] = arr[i], arr[0]

        # Heapify reduced heap
        heapify(arr, i, 0)

    return arr

def heapify(arr: List[int], n: int, i: int) -> None:
    """Heapify subtree rooted at index i"""
    largest = i
    left = 2 * i + 1
    right = 2 * i + 2

    # Left child larger than root
    if left < n and arr[left] > arr[largest]:
        largest = left

    # Right child larger than largest so far
    if right < n and arr[right] > arr[largest]:
        largest = right

    # Largest is not root
    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)

# Example
arr = [12, 11, 13, 5, 6, 7]
print(heap_sort(arr))  # [5, 6, 7, 11, 12, 13]
```

### Tim Sort (Python's Built-in)

```python
"""
Python's sorted() and list.sort() use Timsort:

- Hybrid of merge sort and insertion sort
- Time: O(n log n) worst case, O(n) best case
- Space: O(n)
- Stable (preserves order of equal elements)
- Optimized for real-world data

When to use built-in sort:
- Almost always! It's highly optimized
- Handles partially sorted data well
- Written in C for performance
"""

# Basic usage
arr = [3, 1, 4, 1, 5, 9, 2, 6]

# sorted() - returns new list
sorted_arr = sorted(arr)
print(sorted_arr)  # [1, 1, 2, 3, 4, 5, 6, 9]

# list.sort() - in-place
arr.sort()
print(arr)  # [1, 1, 2, 3, 4, 5, 6, 9]

# Reverse order
arr.sort(reverse=True)
print(arr)  # [9, 6, 5, 4, 3, 2, 1, 1]

# Custom key function
users = [
    {'name': 'Alice', 'age': 30},
    {'name': 'Bob', 'age': 25},
    {'name': 'Charlie', 'age': 35}
]

# Sort by age
sorted_users = sorted(users, key=lambda x: x['age'])
print([u['name'] for u in sorted_users])  # ['Bob', 'Alice', 'Charlie']

# Sort by multiple keys
data = [(1, 'b'), (2, 'a'), (1, 'a')]
sorted_data = sorted(data, key=lambda x: (x[0], x[1]))
print(sorted_data)  # [(1, 'a'), (1, 'b'), (2, 'a')]

# Using itemgetter for performance
from operator import itemgetter

sorted_users = sorted(users, key=itemgetter('age'))
```

---

## Searching Algorithms

### Linear Search

```python
def linear_search(arr: List[int], target: int) -> int:
    """
    Linear search - Check each element

    Time: O(n)
    Space: O(1)

    Use: Unsorted data, small datasets
    """
    for i, value in enumerate(arr):
        if value == target:
            return i

    return -1  # Not found

# Example
arr = [2, 3, 4, 10, 40]
print(linear_search(arr, 10))  # 3
print(linear_search(arr, 5))   # -1
```

### Binary Search

```python
def binary_search(arr: List[int], target: int) -> int:
    """
    Binary search - Divide and conquer on sorted array

    Time: O(log n)
    Space: O(1)

    Requirements: Array must be sorted
    Use: Searching in sorted data
    """
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2  # Avoid overflow

        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1  # Not found

# Recursive version
def binary_search_recursive(
    arr: List[int],
    target: int,
    left: int = 0,
    right: int = None
) -> int:
    """Binary search - recursive implementation"""
    if right is None:
        right = len(arr) - 1

    if left > right:
        return -1

    mid = left + (right - left) // 2

    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        return binary_search_recursive(arr, target, mid + 1, right)
    else:
        return binary_search_recursive(arr, target, left, mid - 1)

# Example
arr = [2, 3, 4, 10, 40]
print(binary_search(arr, 10))  # 3
print(binary_search(arr, 5))   # -1

# Python's bisect module
import bisect

# Find insertion point
arr = [1, 3, 4, 4, 6, 8]
pos = bisect.bisect_left(arr, 4)   # 2 (leftmost position)
pos = bisect.bisect_right(arr, 4)  # 4 (rightmost position)

# Insert and maintain sorted order
bisect.insort(arr, 5)
print(arr)  # [1, 3, 4, 4, 5, 6, 8]
```

### Binary Search Variations

```python
def find_first_occurrence(arr: List[int], target: int) -> int:
    """Find first occurrence of target"""
    left, right = 0, len(arr) - 1
    result = -1

    while left <= right:
        mid = left + (right - left) // 2

        if arr[mid] == target:
            result = mid
            right = mid - 1  # Continue searching left
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return result

def find_last_occurrence(arr: List[int], target: int) -> int:
    """Find last occurrence of target"""
    left, right = 0, len(arr) - 1
    result = -1

    while left <= right:
        mid = left + (right - left) // 2

        if arr[mid] == target:
            result = mid
            left = mid + 1  # Continue searching right
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return result

def count_occurrences(arr: List[int], target: int) -> int:
    """Count occurrences of target"""
    first = find_first_occurrence(arr, target)

    if first == -1:
        return 0

    last = find_last_occurrence(arr, target)
    return last - first + 1

# Example
arr = [1, 2, 2, 2, 3, 4, 5]
print(find_first_occurrence(arr, 2))  # 1
print(find_last_occurrence(arr, 2))   # 3
print(count_occurrences(arr, 2))      # 3
```

### Search in Rotated Sorted Array

```python
def search_rotated(arr: List[int], target: int) -> int:
    """
    Search in rotated sorted array
    Example: [4, 5, 6, 7, 0, 1, 2]

    Time: O(log n)
    """
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if arr[mid] == target:
            return mid

        # Left half is sorted
        if arr[left] <= arr[mid]:
            if arr[left] <= target < arr[mid]:
                right = mid - 1
            else:
                left = mid + 1

        # Right half is sorted
        else:
            if arr[mid] < target <= arr[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1

# Example
arr = [4, 5, 6, 7, 0, 1, 2]
print(search_rotated(arr, 0))  # 4
print(search_rotated(arr, 3))  # -1
```

---

## Real-World Applications

### Sorting Users by Multiple Criteria

```python
from dataclasses import dataclass
from typing import List
from datetime import datetime

@dataclass
class User:
    """User model"""
    id: int
    name: str
    email: str
    created_at: datetime
    is_active: bool
    score: int

def sort_users(users: List[User]) -> List[User]:
    """Sort users by multiple criteria"""
    # Sort by: active first, then by score desc, then by name
    return sorted(
        users,
        key=lambda u: (not u.is_active, -u.score, u.name)
    )

# Example
users = [
    User(1, "Alice", "alice@example.com", datetime.now(), True, 100),
    User(2, "Bob", "bob@example.com", datetime.now(), False, 150),
    User(3, "Charlie", "charlie@example.com", datetime.now(), True, 120),
]

sorted_users = sort_users(users)
for user in sorted_users:
    print(f"{user.name}: active={user.is_active}, score={user.score}")
```

### Efficient Database Query Results Processing

```python
from typing import List, Dict, Any

def process_large_dataset(records: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
    """
    Process large dataset efficiently

    Use case: Sort 1M+ database records
    Strategy: Use key extraction for performance
    """
    # Bad: Lambda creates overhead
    # sorted_records = sorted(records, key=lambda r: r['created_at'])

    # Good: itemgetter is faster
    from operator import itemgetter

    sorted_records = sorted(records, key=itemgetter('created_at'))

    return sorted_records

def get_top_n_users(users: List[User], n: int = 10) -> List[User]:
    """
    Get top N users by score

    Use case: Leaderboard, trending users
    Strategy: heapq.nlargest for partial sorting
    """
    import heapq

    # More efficient than sorting entire list
    return heapq.nlargest(n, users, key=lambda u: u.score)
```

### Autocomplete with Binary Search

```python
class Autocomplete:
    """Autocomplete using binary search"""

    def __init__(self, words: List[str]):
        """Initialize with sorted word list"""
        self.words = sorted(words)

    def search(self, prefix: str) -> List[str]:
        """Find all words with given prefix"""
        # Find starting position
        start = bisect.bisect_left(self.words, prefix)

        # Collect words with prefix
        results = []
        for i in range(start, len(self.words)):
            if self.words[i].startswith(prefix):
                results.append(self.words[i])
            else:
                break

        return results

# Example
words = ["apple", "application", "apply", "banana", "band", "bank"]
autocomplete = Autocomplete(words)

print(autocomplete.search("app"))  # ['apple', 'application', 'apply']
print(autocomplete.search("ban"))  # ['banana', 'band', 'bank']
```

---

## Algorithm Comparison

### When to Use Each Sort

```python
"""
Sorting Algorithm Selection Guide:

1. Python's built-in sort (Timsort):
   - Use: 99% of the time
   - Why: Highly optimized, stable, handles real data well
   - Example: sorted(arr) or arr.sort()

2. Quick Sort:
   - Use: When implementing from scratch, good cache locality
   - Avoid: Already have sorted(), unstable

3. Merge Sort:
   - Use: When stability required, external sorting
   - Avoid: Extra space overhead

4. Heap Sort:
   - Use: When O(1) space needed, priority queue
   - Avoid: Poor cache performance, unstable

5. Counting Sort / Radix Sort:
   - Use: Integer sorting with limited range
   - Example: Sorting ages (0-150), IDs with known range
"""

# Counting sort for limited range
def counting_sort(arr: List[int], max_val: int) -> List[int]:
    """
    Counting sort - for integers in limited range

    Time: O(n + k) where k is range
    Space: O(k)

    Use: Sorting ages, small integers
    """
    count = [0] * (max_val + 1)

    # Count occurrences
    for num in arr:
        count[num] += 1

    # Reconstruct sorted array
    result = []
    for num, cnt in enumerate(count):
        result.extend([num] * cnt)

    return result

# Example
ages = [25, 30, 22, 25, 28, 30, 22]
sorted_ages = counting_sort(ages, max_val=150)
print(sorted_ages)  # [22, 22, 25, 25, 28, 30, 30]
```

---

## Best Practices

### ✅ Do's:

1. **Use built-in sort** - It's highly optimized
2. **Use bisect** for binary search
3. **Use heapq.nlargest/nsmallest** for partial sorting
4. **Use key parameter** instead of comparators
5. **Use operator.itemgetter** for better performance
6. **Ensure data is sorted** before binary search
7. **Consider stability** when sorting
8. **Profile before** optimizing sort algorithm

### ❌ Don'ts:

1. **Don't implement** sorting from scratch unless needed
2. **Don't use** bubble sort in production
3. **Don't forget** to handle edge cases (empty, single element)
4. **Don't use** O(n²) sorts for large data
5. **Don't binary search** on unsorted data
6. **Don't ignore** stability requirements

---

## Interview Questions

### Q1: What's the difference between stable and unstable sorts?

**Answer**: Stable sorts preserve relative order of equal elements:

- **Stable**: Merge sort, Timsort, insertion sort
- **Unstable**: Quick sort, heap sort
- **Matters when**: Sorting objects with multiple fields
- **Example**: Sort users by score, then by name - need stability

### Q2: Why is Python's sort so fast?

**Answer**: Timsort combines best of merge and insertion:

- **Adaptive**: O(n) on nearly sorted data
- **Stable**: Preserves order
- **Smart**: Detects runs, galloping mode
- **Optimized**: Written in C
- **Real-world**: Handles actual data patterns well

### Q3: When would you use quick sort over merge sort?

**Answer**:

- **Quick sort**: Better cache locality, in-place (O(1) space)
- **Merge sort**: Guaranteed O(n log n), stable
- **Quick sort better**: Memory constrained, cache matters
- **Merge sort better**: Need stability, worst case guarantees
  In Python, just use sorted() - it's Timsort.

### Q4: How to find kth largest element efficiently?

**Answer**:

- **Option 1**: Sort and take arr[-k] - O(n log n)
- **Option 2**: heapq.nlargest(k, arr)[-1] - O(n log k)
- **Option 3**: Quick select - O(n) average
- **Best**: Use heapq for top-k problems

### Q5: How does binary search handle duplicates?

**Answer**:

- **bisect_left**: Leftmost position for insertion
- **bisect_right**: Rightmost position
- **First occurrence**: Use bisect_left, verify found
- **Last occurrence**: Use bisect_right - 1
- **Count**: last - first + 1

---

## Summary

Sorting and searching essentials:

- **Use built-in sort** - Timsort is excellent
- **Binary search** - O(log n) on sorted data
- **Quick sort** - Average O(n log n), in-place
- **Merge sort** - Stable, guaranteed O(n log n)
- **Heap sort** - O(1) space, priority queue
- **bisect module** - Binary search utilities
- **heapq** - Partial sorting, top-k problems

Master the fundamentals! 🔄
