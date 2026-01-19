# 🔍 Advanced Data Structures

## Overview

Beyond basic lists, dicts, and sets - understanding advanced data structures for optimal performance in specific scenarios.

---

## Trees

### Binary Search Tree

```python
from typing import Optional, Any
from dataclasses import dataclass

@dataclass
class TreeNode:
    """Binary tree node"""
    value: Any
    left: Optional['TreeNode'] = None
    right: Optional['TreeNode'] = None

class BinarySearchTree:
    """Binary Search Tree implementation"""

    def __init__(self):
        self.root: Optional[TreeNode] = None

    def insert(self, value: Any) -> None:
        """Insert value into BST"""
        if self.root is None:
            self.root = TreeNode(value)
        else:
            self._insert_recursive(self.root, value)

    def _insert_recursive(self, node: TreeNode, value: Any) -> None:
        """Recursive insert helper"""
        if value < node.value:
            if node.left is None:
                node.left = TreeNode(value)
            else:
                self._insert_recursive(node.left, value)
        else:
            if node.right is None:
                node.right = TreeNode(value)
            else:
                self._insert_recursive(node.right, value)

    def search(self, value: Any) -> bool:
        """Search for value in BST"""
        return self._search_recursive(self.root, value)

    def _search_recursive(self, node: Optional[TreeNode], value: Any) -> bool:
        """Recursive search helper"""
        if node is None:
            return False

        if value == node.value:
            return True
        elif value < node.value:
            return self._search_recursive(node.left, value)
        else:
            return self._search_recursive(node.right, value)

    def inorder_traversal(self) -> list:
        """In-order traversal (sorted)"""
        result = []
        self._inorder_recursive(self.root, result)
        return result

    def _inorder_recursive(self, node: Optional[TreeNode], result: list) -> None:
        """Recursive in-order helper"""
        if node:
            self._inorder_recursive(node.left, result)
            result.append(node.value)
            self._inorder_recursive(node.right, result)

# Usage
bst = BinarySearchTree()
for value in [50, 30, 70, 20, 40, 60, 80]:
    bst.insert(value)

print(bst.search(40))  # True
print(bst.inorder_traversal())  # [20, 30, 40, 50, 60, 70, 80]

# Time complexity:
# Average: O(log n) for insert, search, delete
# Worst: O(n) when unbalanced
```

### Trie (Prefix Tree)

```python
class TrieNode:
    """Trie node"""

    def __init__(self):
        self.children: dict = {}
        self.is_end_of_word: bool = False

class Trie:
    """Trie for efficient string operations"""

    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        """Insert word into trie"""
        node = self.root

        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]

        node.is_end_of_word = True

    def search(self, word: str) -> bool:
        """Search for exact word"""
        node = self._find_node(word)
        return node is not None and node.is_end_of_word

    def starts_with(self, prefix: str) -> bool:
        """Check if any word starts with prefix"""
        return self._find_node(prefix) is not None

    def _find_node(self, prefix: str) -> Optional[TrieNode]:
        """Find node for given prefix"""
        node = self.root

        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]

        return node

    def autocomplete(self, prefix: str) -> list:
        """Get all words with given prefix"""
        node = self._find_node(prefix)

        if node is None:
            return []

        results = []
        self._collect_words(node, prefix, results)
        return results

    def _collect_words(self, node: TrieNode, prefix: str, results: list) -> None:
        """Collect all words from node"""
        if node.is_end_of_word:
            results.append(prefix)

        for char, child_node in node.children.items():
            self._collect_words(child_node, prefix + char, results)

# Usage - Autocomplete system
trie = Trie()
words = ["apple", "application", "apply", "banana", "band"]

for word in words:
    trie.insert(word)

print(trie.search("apple"))  # True
print(trie.starts_with("app"))  # True
print(trie.autocomplete("app"))  # ['apple', 'application', 'apply']

# Time complexity: O(m) where m is word/prefix length
# Perfect for autocomplete, spell check, IP routing
```

---

## Heaps

### Priority Queue with Min Heap

```python
import heapq
from typing import Any, List, Tuple
from dataclasses import dataclass, field

@dataclass(order=True)
class PriorityItem:
    """Item with priority"""
    priority: int
    data: Any = field(compare=False)

class PriorityQueue:
    """Priority queue using heap"""

    def __init__(self):
        self._heap: List[PriorityItem] = []

    def push(self, item: Any, priority: int) -> None:
        """Add item with priority"""
        heapq.heappush(self._heap, PriorityItem(priority, item))

    def pop(self) -> Any:
        """Remove and return highest priority item"""
        if self.is_empty():
            raise IndexError("Priority queue is empty")
        return heapq.heappop(self._heap).data

    def peek(self) -> Any:
        """View highest priority item without removing"""
        if self.is_empty():
            raise IndexError("Priority queue is empty")
        return self._heap[0].data

    def is_empty(self) -> bool:
        """Check if queue is empty"""
        return len(self._heap) == 0

    def size(self) -> int:
        """Get queue size"""
        return len(self._heap)

# Usage - Task scheduling
pq = PriorityQueue()

pq.push("Low priority task", priority=3)
pq.push("High priority task", priority=1)
pq.push("Medium priority task", priority=2)

while not pq.is_empty():
    task = pq.pop()
    print(f"Processing: {task}")

# Output (by priority):
# Processing: High priority task
# Processing: Medium priority task
# Processing: Low priority task

# Real-world use case: Dijkstra's algorithm
class Graph:
    """Graph with weighted edges"""

    def __init__(self):
        self.adjacency: dict = {}

    def add_edge(self, from_node: str, to_node: str, weight: int) -> None:
        """Add weighted edge"""
        if from_node not in self.adjacency:
            self.adjacency[from_node] = []
        self.adjacency[from_node].append((to_node, weight))

    def dijkstra(self, start: str) -> dict:
        """Find shortest paths using Dijkstra's algorithm"""
        distances = {start: 0}
        pq = PriorityQueue()
        pq.push(start, 0)

        while not pq.is_empty():
            current = pq.pop()
            current_distance = distances[current]

            if current not in self.adjacency:
                continue

            for neighbor, weight in self.adjacency[current]:
                distance = current_distance + weight

                if neighbor not in distances or distance < distances[neighbor]:
                    distances[neighbor] = distance
                    pq.push(neighbor, distance)

        return distances

# Usage
graph = Graph()
graph.add_edge("A", "B", 4)
graph.add_edge("A", "C", 2)
graph.add_edge("B", "D", 3)
graph.add_edge("C", "D", 1)
graph.add_edge("C", "B", 1)

distances = graph.dijkstra("A")
print(distances)  # {'A': 0, 'C': 2, 'B': 3, 'D': 3}
```

---

## Graphs

### Graph Representations

```python
from collections import defaultdict, deque
from typing import List, Set, Dict

class Graph:
    """Graph using adjacency list"""

    def __init__(self):
        self.graph: Dict[Any, List[Any]] = defaultdict(list)

    def add_edge(self, u: Any, v: Any, directed: bool = False) -> None:
        """Add edge between nodes"""
        self.graph[u].append(v)
        if not directed:
            self.graph[v].append(u)

    def bfs(self, start: Any) -> List[Any]:
        """Breadth-First Search"""
        visited: Set[Any] = set()
        queue = deque([start])
        result = []

        while queue:
            node = queue.popleft()

            if node in visited:
                continue

            visited.add(node)
            result.append(node)

            for neighbor in self.graph[node]:
                if neighbor not in visited:
                    queue.append(neighbor)

        return result

    def dfs(self, start: Any) -> List[Any]:
        """Depth-First Search"""
        visited: Set[Any] = set()
        result = []

        def dfs_recursive(node: Any) -> None:
            if node in visited:
                return

            visited.add(node)
            result.append(node)

            for neighbor in self.graph[node]:
                dfs_recursive(neighbor)

        dfs_recursive(start)
        return result

    def has_cycle(self) -> bool:
        """Detect cycle in directed graph"""
        visited: Set[Any] = set()
        rec_stack: Set[Any] = set()

        def has_cycle_util(node: Any) -> bool:
            visited.add(node)
            rec_stack.add(node)

            for neighbor in self.graph[node]:
                if neighbor not in visited:
                    if has_cycle_util(neighbor):
                        return True
                elif neighbor in rec_stack:
                    return True

            rec_stack.remove(node)
            return False

        for node in self.graph:
            if node not in visited:
                if has_cycle_util(node):
                    return True

        return False

    def topological_sort(self) -> List[Any]:
        """Topological sort for DAG"""
        visited: Set[Any] = set()
        stack = []

        def topological_sort_util(node: Any) -> None:
            visited.add(node)

            for neighbor in self.graph[node]:
                if neighbor not in visited:
                    topological_sort_util(neighbor)

            stack.append(node)

        for node in self.graph:
            if node not in visited:
                topological_sort_util(node)

        return stack[::-1]

# Usage - Dependency resolution
graph = Graph()
graph.add_edge("A", "B", directed=True)
graph.add_edge("A", "C", directed=True)
graph.add_edge("B", "D", directed=True)
graph.add_edge("C", "D", directed=True)

print("BFS:", graph.bfs("A"))  # ['A', 'B', 'C', 'D']
print("DFS:", graph.dfs("A"))  # ['A', 'B', 'D', 'C']
print("Topological:", graph.topological_sort())  # ['A', 'C', 'B', 'D']
print("Has cycle:", graph.has_cycle())  # False
```

---

## Union-Find (Disjoint Set)

### Efficient Set Operations

```python
class UnionFind:
    """Union-Find with path compression and union by rank"""

    def __init__(self, n: int):
        """Initialize n disjoint sets"""
        self.parent = list(range(n))
        self.rank = [0] * n
        self.count = n  # Number of disjoint sets

    def find(self, x: int) -> int:
        """Find root with path compression"""
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x: int, y: int) -> bool:
        """Union two sets by rank"""
        root_x = self.find(x)
        root_y = self.find(y)

        if root_x == root_y:
            return False  # Already in same set

        # Union by rank
        if self.rank[root_x] < self.rank[root_y]:
            self.parent[root_x] = root_y
        elif self.rank[root_x] > self.rank[root_y]:
            self.parent[root_y] = root_x
        else:
            self.parent[root_y] = root_x
            self.rank[root_x] += 1

        self.count -= 1
        return True

    def connected(self, x: int, y: int) -> bool:
        """Check if x and y are in same set"""
        return self.find(x) == self.find(y)

# Usage - Network connectivity
uf = UnionFind(5)  # 5 nodes

# Connect nodes
uf.union(0, 1)
uf.union(1, 2)
uf.union(3, 4)

print(uf.connected(0, 2))  # True (0-1-2 connected)
print(uf.connected(0, 3))  # False (different components)
print(uf.count)  # 2 (two components: {0,1,2} and {3,4})

# Kruskal's MST algorithm
class Edge:
    """Graph edge"""

    def __init__(self, u: int, v: int, weight: int):
        self.u = u
        self.v = v
        self.weight = weight

def kruskal_mst(n: int, edges: List[Edge]) -> List[Edge]:
    """Find Minimum Spanning Tree using Kruskal's algorithm"""
    # Sort edges by weight
    edges.sort(key=lambda e: e.weight)

    uf = UnionFind(n)
    mst = []

    for edge in edges:
        # Add edge if it doesn't create cycle
        if uf.union(edge.u, edge.v):
            mst.append(edge)

            # Stop when we have n-1 edges
            if len(mst) == n - 1:
                break

    return mst

# Time complexity: O(α(n)) amortized for find/union
# where α is inverse Ackermann function (practically constant)
```

---

## Segment Tree

### Range Query Data Structure

```python
class SegmentTree:
    """Segment tree for range queries"""

    def __init__(self, arr: List[int]):
        """Build segment tree"""
        self.n = len(arr)
        self.tree = [0] * (4 * self.n)
        self._build(arr, 0, 0, self.n - 1)

    def _build(self, arr: List[int], node: int, start: int, end: int) -> None:
        """Build tree recursively"""
        if start == end:
            self.tree[node] = arr[start]
            return

        mid = (start + end) // 2
        left_child = 2 * node + 1
        right_child = 2 * node + 2

        self._build(arr, left_child, start, mid)
        self._build(arr, right_child, mid + 1, end)

        self.tree[node] = self.tree[left_child] + self.tree[right_child]

    def query(self, left: int, right: int) -> int:
        """Query sum in range [left, right]"""
        return self._query(0, 0, self.n - 1, left, right)

    def _query(
        self,
        node: int,
        start: int,
        end: int,
        left: int,
        right: int
    ) -> int:
        """Query helper"""
        # No overlap
        if right < start or left > end:
            return 0

        # Complete overlap
        if left <= start and end <= right:
            return self.tree[node]

        # Partial overlap
        mid = (start + end) // 2
        left_sum = self._query(2 * node + 1, start, mid, left, right)
        right_sum = self._query(2 * node + 2, mid + 1, end, left, right)

        return left_sum + right_sum

    def update(self, index: int, value: int) -> None:
        """Update element at index"""
        self._update(0, 0, self.n - 1, index, value)

    def _update(
        self,
        node: int,
        start: int,
        end: int,
        index: int,
        value: int
    ) -> None:
        """Update helper"""
        if start == end:
            self.tree[node] = value
            return

        mid = (start + end) // 2

        if index <= mid:
            self._update(2 * node + 1, start, mid, index, value)
        else:
            self._update(2 * node + 2, mid + 1, end, index, value)

        self.tree[node] = (
            self.tree[2 * node + 1] + self.tree[2 * node + 2]
        )

# Usage
arr = [1, 3, 5, 7, 9, 11]
seg_tree = SegmentTree(arr)

print(seg_tree.query(1, 3))  # Sum of arr[1:4] = 3+5+7 = 15

seg_tree.update(1, 10)  # Update arr[1] = 10
print(seg_tree.query(1, 3))  # Sum = 10+5+7 = 22

# Time complexity:
# Build: O(n)
# Query: O(log n)
# Update: O(log n)
# Perfect for range sum/min/max queries with updates
```

---

## Best Practices

### ✅ Do's:

1. **Choose right structure** for problem
2. **Know time complexities** of operations
3. **Use heapq** for priority queues
4. **Use collections** module (deque, defaultdict)
5. **Consider space-time** trade-offs
6. **Test edge cases** (empty, single element)
7. **Document complexity** in comments

### ❌ Don'ts:

1. **Don't use list** as queue (O(n) pop(0))
2. **Don't implement** what's in standard library
3. **Don't optimize** prematurely
4. **Don't forget** to handle cycles in graphs
5. **Don't ignore** memory constraints

---

## Interview Questions

### Q1: When to use Trie vs Hash Table?

**Answer**:

- **Trie**: Prefix searches, autocomplete, spell check
- **Hash Table**: Exact key lookups
- **Trie advantages**: Prefix operations, sorted order
- **Trie disadvantages**: More memory, slower single lookup

### Q2: Explain Union-Find applications.

**Answer**:

- **Network connectivity**: Check if nodes connected
- **Kruskal's MST**: Find minimum spanning tree
- **Image processing**: Connected components
- **Social networks**: Friend groups
  Nearly O(1) operations with optimizations.

### Q3: When to use Segment Tree?

**Answer**: Range queries with updates:

- Range sum/min/max queries
- Array updates
- O(log n) query and update
- Alternative: Fenwick tree (simpler, less memory)

### Q4: BFS vs DFS - when to use each?

**Answer**:

- **BFS**: Shortest path, level-order, close neighbors
- **DFS**: Detect cycles, topological sort, path finding
- **BFS**: O(V+E) time, O(V) space (queue)
- **DFS**: O(V+E) time, O(V) space (recursion/stack)

### Q5: How to detect cycle in graph?

**Answer**:

- **Directed**: DFS with recursion stack
- **Undirected**: DFS tracking parent
- **Alternative**: Union-Find
  Check if visiting already visited node in current path.

---

## Summary

Advanced structures:

- **Trees**: BST for sorted data, Trie for strings
- **Heaps**: Priority queue O(log n) operations
- **Graphs**: BFS/DFS for traversal, topological sort
- **Union-Find**: Nearly O(1) for connectivity
- **Segment Tree**: O(log n) range queries
- **Choose wisely**: Based on operations needed

Master these for interviews! 🔍
