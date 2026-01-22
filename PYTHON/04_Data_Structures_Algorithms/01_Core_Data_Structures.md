# 📊 Data Structures and Algorithms in Python

## Overview

Understanding core data structures and their time/space complexity is crucial for writing efficient Python code and succeeding in technical interviews.

---

## Time and Space Complexity

### Big O Notation

```python
# O(1) - Constant Time
def get_first_element(arr):
    """Always takes same time regardless of input size"""
    return arr[0] if arr else None

# O(log n) - Logarithmic Time
def binary_search(arr, target):
    """Divides search space in half each iteration"""
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = (left + right) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1

# O(n) - Linear Time
def find_max(arr):
    """Must check every element"""
    max_val = arr[0]
    for num in arr:
        if num > max_val:
            max_val = num
    return max_val

# O(n log n) - Linearithmic Time
def merge_sort(arr):
    """Efficient sorting algorithm"""
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])

    return merge(left, right)

# O(n²) - Quadratic Time
def bubble_sort(arr):
    """Nested loops over array"""
    n = len(arr)
    for i in range(n):
        for j in range(0, n - i - 1):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
    return arr

# O(2^n) - Exponential Time
def fibonacci_recursive(n):
    """Exponential - very slow for large n"""
    if n <= 1:
        return n
    return fibonacci_recursive(n - 1) + fibonacci_recursive(n - 2)

# O(n!) - Factorial Time
def permutations(arr):
    """Generate all permutations"""
    if len(arr) <= 1:
        return [arr]

    result = []
    for i in range(len(arr)):
        rest = arr[:i] + arr[i+1:]
        for p in permutations(rest):
            result.append([arr[i]] + p)

    return result
```

### Complexity Comparison Table

| Complexity | Name         | Example                   |
| ---------- | ------------ | ------------------------- |
| O(1)       | Constant     | Array access, dict lookup |
| O(log n)   | Logarithmic  | Binary search             |
| O(n)       | Linear       | Array scan, linear search |
| O(n log n) | Linearithmic | Merge sort, quicksort     |
| O(n²)      | Quadratic    | Bubble sort, nested loops |
| O(2^n)     | Exponential  | Recursive fibonacci       |
| O(n!)      | Factorial    | Permutations              |

---

## Arrays and Lists

### Python List Operations

```python
# List operations complexity
operations = {
    "append()": "O(1) amortized",
    "pop()": "O(1)",
    "pop(i)": "O(n)",
    "insert(i, x)": "O(n)",
    "remove(x)": "O(n)",
    "index(x)": "O(n)",
    "sort()": "O(n log n)",
    "reverse()": "O(n)",
    "extend()": "O(k) where k is length of extension"
}

# Common list patterns
class ListProblems:
    """Common list problems"""

    @staticmethod
    def two_sum(nums: list[int], target: int) -> list[int]:
        """Find two numbers that add up to target

        Time: O(n), Space: O(n)
        """
        seen = {}
        for i, num in enumerate(nums):
            complement = target - num
            if complement in seen:
                return [seen[complement], i]
            seen[num] = i
        return []

    @staticmethod
    def remove_duplicates(nums: list[int]) -> int:
        """Remove duplicates from sorted array in-place

        Time: O(n), Space: O(1)
        """
        if not nums:
            return 0

        write_idx = 1
        for i in range(1, len(nums)):
            if nums[i] != nums[i - 1]:
                nums[write_idx] = nums[i]
                write_idx += 1

        return write_idx

    @staticmethod
    def max_subarray_sum(nums: list[int]) -> int:
        """Kadane's algorithm - maximum subarray sum

        Time: O(n), Space: O(1)
        """
        max_sum = current_sum = nums[0]

        for num in nums[1:]:
            current_sum = max(num, current_sum + num)
            max_sum = max(max_sum, current_sum)

        return max_sum

    @staticmethod
    def sliding_window_max(nums: list[int], k: int) -> list[int]:
        """Maximum in sliding window of size k

        Time: O(n), Space: O(k)
        """
        from collections import deque

        result = []
        window = deque()  # Store indices

        for i, num in enumerate(nums):
            # Remove elements outside window
            while window and window[0] <= i - k:
                window.popleft()

            # Remove smaller elements (not useful)
            while window and nums[window[-1]] < num:
                window.pop()

            window.append(i)

            # Add to result once window is full
            if i >= k - 1:
                result.append(nums[window[0]])

        return result
```

---

## Hash Tables (Dictionaries)

### Dictionary Operations

```python
# Dictionary complexity - all O(1) average case
operations = {
    "access": "O(1) average",
    "insert": "O(1) average",
    "delete": "O(1) average",
    "search": "O(1) average"
}

class HashTableProblems:
    """Common hash table problems"""

    @staticmethod
    def group_anagrams(strs: list[str]) -> list[list[str]]:
        """Group anagrams together

        Time: O(n * k log k), Space: O(n * k)
        where n is number of strings, k is max length
        """
        from collections import defaultdict

        groups = defaultdict(list)

        for s in strs:
            # Sort string as key
            key = ''.join(sorted(s))
            groups[key].append(s)

        return list(groups.values())

    @staticmethod
    def top_k_frequent(nums: list[int], k: int) -> list[int]:
        """Find k most frequent elements

        Time: O(n log k), Space: O(n)
        """
        from collections import Counter
        import heapq

        # Count frequencies
        counts = Counter(nums)

        # Use heap to find top k
        return heapq.nlargest(k, counts.keys(), key=counts.get)

    @staticmethod
    def longest_consecutive(nums: list[int]) -> int:
        """Longest consecutive sequence

        Time: O(n), Space: O(n)
        """
        num_set = set(nums)
        max_length = 0

        for num in num_set:
            # Only start counting from sequence start
            if num - 1 not in num_set:
                current = num
                length = 1

                while current + 1 in num_set:
                    current += 1
                    length += 1

                max_length = max(max_length, length)

        return max_length

    @staticmethod
    def subarray_sum_k(nums: list[int], k: int) -> int:
        """Count subarrays with sum equal to k

        Time: O(n), Space: O(n)
        """
        from collections import defaultdict

        prefix_sums = defaultdict(int)
        prefix_sums[0] = 1  # Empty subarray

        count = 0
        current_sum = 0

        for num in nums:
            current_sum += num

            # If (current_sum - k) exists, found valid subarray
            if current_sum - k in prefix_sums:
                count += prefix_sums[current_sum - k]

            prefix_sums[current_sum] += 1

        return count
```

---

## Stacks and Queues

### Stack Implementation

```python
class Stack:
    """Stack using list"""

    def __init__(self):
        self.items = []

    def push(self, item):
        """Add item to top - O(1)"""
        self.items.append(item)

    def pop(self):
        """Remove and return top item - O(1)"""
        if self.is_empty():
            raise IndexError("Pop from empty stack")
        return self.items.pop()

    def peek(self):
        """View top item without removing - O(1)"""
        if self.is_empty():
            raise IndexError("Peek empty stack")
        return self.items[-1]

    def is_empty(self):
        """Check if stack is empty - O(1)"""
        return len(self.items) == 0

    def size(self):
        """Get stack size - O(1)"""
        return len(self.items)

class StackProblems:
    """Common stack problems"""

    @staticmethod
    def valid_parentheses(s: str) -> bool:
        """Check if parentheses are balanced

        Time: O(n), Space: O(n)
        """
        stack = []
        pairs = {'(': ')', '[': ']', '{': '}'}

        for char in s:
            if char in pairs:
                stack.append(char)
            elif not stack or pairs[stack.pop()] != char:
                return False

        return len(stack) == 0

    @staticmethod
    def daily_temperatures(temperatures: list[int]) -> list[int]:
        """Days until warmer temperature

        Time: O(n), Space: O(n)
        """
        result = [0] * len(temperatures)
        stack = []  # Store indices

        for i, temp in enumerate(temperatures):
            while stack and temperatures[stack[-1]] < temp:
                prev_idx = stack.pop()
                result[prev_idx] = i - prev_idx
            stack.append(i)

        return result

    @staticmethod
    def evaluate_rpn(tokens: list[str]) -> int:
        """Evaluate Reverse Polish Notation

        Time: O(n), Space: O(n)
        """
        stack = []
        operators = {
            '+': lambda a, b: a + b,
            '-': lambda a, b: a - b,
            '*': lambda a, b: a * b,
            '/': lambda a, b: int(a / b)
        }

        for token in tokens:
            if token in operators:
                b = stack.pop()
                a = stack.pop()
                result = operators[token](a, b)
                stack.append(result)
            else:
                stack.append(int(token))

        return stack[0]
```

### Queue Implementation

```python
from collections import deque

class Queue:
    """Queue using deque"""

    def __init__(self):
        self.items = deque()

    def enqueue(self, item):
        """Add item to rear - O(1)"""
        self.items.append(item)

    def dequeue(self):
        """Remove and return front item - O(1)"""
        if self.is_empty():
            raise IndexError("Dequeue from empty queue")
        return self.items.popleft()

    def peek(self):
        """View front item - O(1)"""
        if self.is_empty():
            raise IndexError("Peek empty queue")
        return self.items[0]

    def is_empty(self):
        """Check if queue is empty - O(1)"""
        return len(self.items) == 0

    def size(self):
        """Get queue size - O(1)"""
        return len(self.items)
```

---

## Trees

### Binary Tree

```python
class TreeNode:
    """Binary tree node"""

    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

class BinaryTreeProblems:
    """Common binary tree problems"""

    @staticmethod
    def inorder_traversal(root: TreeNode) -> list[int]:
        """Inorder: Left, Root, Right

        Time: O(n), Space: O(h) where h is height
        """
        result = []

        def inorder(node):
            if not node:
                return
            inorder(node.left)
            result.append(node.val)
            inorder(node.right)

        inorder(root)
        return result

    @staticmethod
    def level_order_traversal(root: TreeNode) -> list[list[int]]:
        """Level-order traversal (BFS)

        Time: O(n), Space: O(w) where w is max width
        """
        if not root:
            return []

        result = []
        queue = deque([root])

        while queue:
            level_size = len(queue)
            level = []

            for _ in range(level_size):
                node = queue.popleft()
                level.append(node.val)

                if node.left:
                    queue.append(node.left)
                if node.right:
                    queue.append(node.right)

            result.append(level)

        return result

    @staticmethod
    def max_depth(root: TreeNode) -> int:
        """Maximum depth of tree

        Time: O(n), Space: O(h)
        """
        if not root:
            return 0

        left_depth = BinaryTreeProblems.max_depth(root.left)
        right_depth = BinaryTreeProblems.max_depth(root.right)

        return 1 + max(left_depth, right_depth)

    @staticmethod
    def is_valid_bst(root: TreeNode) -> bool:
        """Check if tree is valid BST

        Time: O(n), Space: O(h)
        """
        def validate(node, min_val, max_val):
            if not node:
                return True

            if not (min_val < node.val < max_val):
                return False

            return (validate(node.left, min_val, node.val) and
                    validate(node.right, node.val, max_val))

        return validate(root, float('-inf'), float('inf'))

    @staticmethod
    def lowest_common_ancestor(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
        """Find LCA of two nodes

        Time: O(n), Space: O(h)
        """
        if not root or root == p or root == q:
            return root

        left = BinaryTreeProblems.lowest_common_ancestor(root.left, p, q)
        right = BinaryTreeProblems.lowest_common_ancestor(root.right, p, q)

        if left and right:
            return root

        return left if left else right
```

---

## Graphs

### Graph Representation

```python
class Graph:
    """Graph using adjacency list"""

    def __init__(self):
        self.graph = defaultdict(list)

    def add_edge(self, u, v, bidirectional=True):
        """Add edge between u and v"""
        self.graph[u].append(v)
        if bidirectional:
            self.graph[v].append(u)

    def dfs(self, start):
        """Depth-First Search

        Time: O(V + E), Space: O(V)
        """
        visited = set()
        result = []

        def dfs_helper(node):
            visited.add(node)
            result.append(node)

            for neighbor in self.graph[node]:
                if neighbor not in visited:
                    dfs_helper(neighbor)

        dfs_helper(start)
        return result

    def bfs(self, start):
        """Breadth-First Search

        Time: O(V + E), Space: O(V)
        """
        visited = {start}
        queue = deque([start])
        result = []

        while queue:
            node = queue.popleft()
            result.append(node)

            for neighbor in self.graph[node]:
                if neighbor not in visited:
                    visited.add(neighbor)
                    queue.append(neighbor)

        return result

    def has_cycle(self) -> bool:
        """Detect cycle in undirected graph

        Time: O(V + E), Space: O(V)
        """
        visited = set()

        def dfs(node, parent):
            visited.add(node)

            for neighbor in self.graph[node]:
                if neighbor not in visited:
                    if dfs(neighbor, node):
                        return True
                elif neighbor != parent:
                    return True

            return False

        for node in self.graph:
            if node not in visited:
                if dfs(node, None):
                    return True

        return False
```

---

## Best Practices

### ✅ Do's:

1. **Understand time/space complexity** of operations
2. **Choose right data structure** for the problem
3. **Consider trade-offs** (time vs space)
4. **Use built-in data structures** when possible
5. **Test edge cases** (empty, single element, large input)
6. **Explain your approach** before coding

### ❌ Don'ts:

1. **Don't use nested loops** unless necessary
2. **Don't ignore complexity** analysis
3. **Don't reinvent the wheel** - use collections module
4. **Don't forget to handle** edge cases
5. **Don't optimize prematurely** - correctness first

---

## Interview Questions

### Q1: What's the difference between list and deque?

**Answer**:

- **List**: Dynamic array, O(1) append/pop, O(n) insert/pop(0)
- **Deque**: Double-ended queue, O(1) for both ends
  Use deque for queue operations, list for stack/array operations.

### Q2: When to use dict vs set?

**Answer**:

- **Dict**: Need key-value mapping
- **Set**: Only need membership testing, uniqueness
  Both have O(1) lookup, but set uses less memory.

### Q3: Explain tree traversal methods.

**Answer**:

- **Inorder** (Left, Root, Right): Gives sorted order for BST
- **Preorder** (Root, Left, Right): Copy tree, prefix expression
- **Postorder** (Left, Right, Root): Delete tree, postfix expression
- **Level-order** (BFS): Level-by-level processing

### Q4: What is a hash collision and how is it resolved?

**Answer**: When two keys hash to same index. Resolution:

- **Chaining**: Store multiple items at same index (linked list)
- **Open addressing**: Find next available slot
  Python dict uses open addressing with random probing.

### Q5: What's the difference between BFS and DFS?

**Answer**:

- **BFS**: Uses queue, explores level-by-level, finds shortest path
- **DFS**: Uses stack/recursion, explores depth-first, less memory for sparse graphs
  Choose based on problem requirements.

---

## Summary

Key data structures:

- **List**: O(1) append, O(n) insert
- **Dict/Set**: O(1) average lookup/insert/delete
- **Deque**: O(1) both ends
- **Heap**: O(log n) insert/delete-min
- **Tree**: O(log n) balanced BST operations
- **Graph**: O(V + E) traversal

Master these for interviews! 📊
