# 🕸️ Graph Algorithms

## Overview

Graph algorithms for traversal, shortest paths, and connectivity problems - essential for interviews and real-world applications.

---

## Graph Representations

### Adjacency List vs Adjacency Matrix

```python
from collections import defaultdict, deque
from typing import List, Dict, Set, Tuple
import heapq

class Graph:
    """Graph using adjacency list"""

    def __init__(self, directed: bool = False):
        """
        Initialize graph

        Args:
            directed: True for directed graph, False for undirected
        """
        self.graph = defaultdict(list)
        self.directed = directed

    def add_edge(self, u: int, v: int, weight: int = 1):
        """
        Add edge to graph

        Time: O(1)
        Space: O(1)
        """
        self.graph[u].append((v, weight))

        if not self.directed:
            self.graph[v].append((u, weight))

    def get_neighbors(self, node: int) -> List[Tuple[int, int]]:
        """Get neighbors of node"""
        return self.graph[node]

# Adjacency Matrix representation
class GraphMatrix:
    """Graph using adjacency matrix"""

    def __init__(self, num_vertices: int, directed: bool = False):
        """
        Initialize graph with matrix

        Space: O(V²)
        """
        self.num_vertices = num_vertices
        self.directed = directed
        # Initialize matrix with infinity (no edge)
        self.matrix = [[float('inf')] * num_vertices
                      for _ in range(num_vertices)]

        # Distance from node to itself is 0
        for i in range(num_vertices):
            self.matrix[i][i] = 0

    def add_edge(self, u: int, v: int, weight: int = 1):
        """
        Add edge to graph

        Time: O(1)
        """
        self.matrix[u][v] = weight

        if not self.directed:
            self.matrix[v][u] = weight

    def has_edge(self, u: int, v: int) -> bool:
        """Check if edge exists"""
        return self.matrix[u][v] != float('inf')

"""
Comparison:
- Adjacency List: O(V + E) space, O(V) to check edge
- Adjacency Matrix: O(V²) space, O(1) to check edge

Use List when: Sparse graphs (E << V²)
Use Matrix when: Dense graphs (E ≈ V²), need fast edge lookup
"""
```

---

## Graph Traversal

### 1. Breadth-First Search (BFS)

```python
def bfs(graph: Graph, start: int) -> List[int]:
    """
    Breadth-First Search traversal

    Level-by-level exploration using queue.

    Time Complexity: O(V + E)
    Space Complexity: O(V) for visited set and queue

    Use Cases:
    - Shortest path in unweighted graph
    - Level-order traversal
    - Finding connected components
    """
    visited = set()
    queue = deque([start])
    visited.add(start)
    result = []

    while queue:
        node = queue.popleft()
        result.append(node)

        # Visit all unvisited neighbors
        for neighbor, _ in graph.get_neighbors(node):
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

    return result

def bfs_shortest_path(graph: Graph, start: int, target: int) -> List[int]:
    """
    Find shortest path using BFS

    Returns path from start to target, or empty if not reachable.

    Time: O(V + E)
    Space: O(V)
    """
    if start == target:
        return [start]

    visited = {start}
    queue = deque([(start, [start])])  # (node, path)

    while queue:
        node, path = queue.popleft()

        for neighbor, _ in graph.get_neighbors(node):
            if neighbor not in visited:
                new_path = path + [neighbor]

                if neighbor == target:
                    return new_path

                visited.add(neighbor)
                queue.append((neighbor, new_path))

    return []  # No path found

# BFS for level-order results
def bfs_by_level(graph: Graph, start: int) -> List[List[int]]:
    """
    BFS returning nodes grouped by level

    Level 0: [start]
    Level 1: [neighbors of start]
    Level 2: [neighbors of level 1], etc.
    """
    visited = {start}
    queue = deque([start])
    levels = []

    while queue:
        level_size = len(queue)
        current_level = []

        for _ in range(level_size):
            node = queue.popleft()
            current_level.append(node)

            for neighbor, _ in graph.get_neighbors(node):
                if neighbor not in visited:
                    visited.add(neighbor)
                    queue.append(neighbor)

        levels.append(current_level)

    return levels
```

### 2. Depth-First Search (DFS)

```python
def dfs_recursive(graph: Graph, start: int, visited: Set[int] = None) -> List[int]:
    """
    Depth-First Search (recursive)

    Explores as far as possible along each branch before backtracking.

    Time Complexity: O(V + E)
    Space Complexity: O(V) for visited + O(V) for recursion stack
    """
    if visited is None:
        visited = set()

    visited.add(start)
    result = [start]

    for neighbor, _ in graph.get_neighbors(start):
        if neighbor not in visited:
            result.extend(dfs_recursive(graph, neighbor, visited))

    return result

def dfs_iterative(graph: Graph, start: int) -> List[int]:
    """
    Depth-First Search (iterative)

    Uses explicit stack instead of recursion.
    Safer for deep graphs (no stack overflow).

    Time: O(V + E)
    Space: O(V)
    """
    visited = set()
    stack = [start]
    result = []

    while stack:
        node = stack.pop()

        if node not in visited:
            visited.add(node)
            result.append(node)

            # Add neighbors to stack (reverse order for left-to-right)
            neighbors = graph.get_neighbors(node)
            for neighbor, _ in reversed(neighbors):
                if neighbor not in visited:
                    stack.append(neighbor)

    return result

def dfs_with_path(graph: Graph, start: int, target: int) -> List[int]:
    """
    Find path using DFS

    Returns a path (not necessarily shortest) or empty list.
    """
    def dfs_helper(node: int, path: List[int], visited: Set[int]) -> List[int]:
        if node == target:
            return path

        visited.add(node)

        for neighbor, _ in graph.get_neighbors(node):
            if neighbor not in visited:
                result = dfs_helper(neighbor, path + [neighbor], visited)
                if result:  # Found path
                    return result

        return []  # No path from this branch

    return dfs_helper(start, [start], set())
```

---

## Shortest Path Algorithms

### 1. Dijkstra's Algorithm

```python
def dijkstra(graph: Graph, start: int) -> Dict[int, int]:
    """
    Dijkstra's shortest path algorithm

    Finds shortest paths from start to all reachable nodes.

    Time Complexity: O((V + E) log V) with min-heap
    Space Complexity: O(V)

    Requirements:
    - Non-negative edge weights only

    Returns:
        Dict mapping node -> shortest distance from start
    """
    # Initialize distances: infinity for all except start
    distances = defaultdict(lambda: float('inf'))
    distances[start] = 0

    # Min-heap: (distance, node)
    heap = [(0, start)]
    visited = set()

    while heap:
        current_dist, node = heapq.heappop(heap)

        if node in visited:
            continue

        visited.add(node)

        # Check all neighbors
        for neighbor, weight in graph.get_neighbors(node):
            if neighbor in visited:
                continue

            # Calculate new distance
            new_dist = current_dist + weight

            # Update if shorter path found
            if new_dist < distances[neighbor]:
                distances[neighbor] = new_dist
                heapq.heappush(heap, (new_dist, neighbor))

    return dict(distances)

def dijkstra_with_path(graph: Graph, start: int, target: int) -> Tuple[int, List[int]]:
    """
    Dijkstra returning shortest distance and path

    Returns:
        (distance, path) or (float('inf'), []) if unreachable
    """
    distances = {start: 0}
    previous = {}  # Track path
    heap = [(0, start)]
    visited = set()

    while heap:
        current_dist, node = heapq.heappop(heap)

        if node in visited:
            continue

        if node == target:
            # Reconstruct path
            path = []
            current = target
            while current in previous:
                path.append(current)
                current = previous[current]
            path.append(start)
            return current_dist, list(reversed(path))

        visited.add(node)

        for neighbor, weight in graph.get_neighbors(node):
            if neighbor in visited:
                continue

            new_dist = current_dist + weight

            if neighbor not in distances or new_dist < distances[neighbor]:
                distances[neighbor] = new_dist
                previous[neighbor] = node
                heapq.heappush(heap, (new_dist, neighbor))

    return float('inf'), []  # Unreachable
```

### 2. Bellman-Ford Algorithm

```python
def bellman_ford(graph: Graph, start: int, num_vertices: int) -> Dict[int, int]:
    """
    Bellman-Ford shortest path algorithm

    Handles negative edge weights (unlike Dijkstra).
    Can detect negative cycles.

    Time Complexity: O(V * E)
    Space Complexity: O(V)

    Returns:
        Dict of distances, or None if negative cycle detected
    """
    # Initialize distances
    distances = {i: float('inf') for i in range(num_vertices)}
    distances[start] = 0

    # Relax edges V-1 times
    for _ in range(num_vertices - 1):
        updated = False

        for node in graph.graph:
            for neighbor, weight in graph.get_neighbors(node):
                if distances[node] + weight < distances[neighbor]:
                    distances[neighbor] = distances[node] + weight
                    updated = True

        if not updated:
            break  # Early termination if no updates

    # Check for negative cycles
    for node in graph.graph:
        for neighbor, weight in graph.get_neighbors(node):
            if distances[node] + weight < distances[neighbor]:
                return None  # Negative cycle detected

    return distances
```

### 3. Floyd-Warshall Algorithm

```python
def floyd_warshall(matrix: List[List[int]]) -> List[List[int]]:
    """
    Floyd-Warshall all-pairs shortest path

    Finds shortest paths between all pairs of vertices.

    Time Complexity: O(V³)
    Space Complexity: O(V²)

    Args:
        matrix: Adjacency matrix (use float('inf') for no edge)

    Returns:
        Matrix of shortest distances
    """
    n = len(matrix)
    # Make a copy to avoid modifying input
    dist = [row[:] for row in matrix]

    # For each intermediate vertex k
    for k in range(n):
        # For each source vertex i
        for i in range(n):
            # For each destination vertex j
            for j in range(n):
                # Update if path through k is shorter
                if dist[i][k] + dist[k][j] < dist[i][j]:
                    dist[i][j] = dist[i][k] + dist[k][j]

    return dist

"""
When to use which algorithm:

Dijkstra:
✅ Single source shortest path
✅ Non-negative weights
✅ Sparse or dense graphs
✅ O((V + E) log V) - fast

Bellman-Ford:
✅ Single source shortest path
✅ Negative weights allowed
✅ Detect negative cycles
✅ O(V * E) - slower

Floyd-Warshall:
✅ All-pairs shortest paths
✅ Negative weights (no negative cycles)
✅ Dense graphs
✅ O(V³) - expensive but finds all paths
"""
```

---

## Topological Sort

```python
def topological_sort_dfs(graph: Graph, num_vertices: int) -> List[int]:
    """
    Topological sort using DFS

    Orders vertices such that for edge u→v, u appears before v.
    Only works on Directed Acyclic Graphs (DAGs).

    Time Complexity: O(V + E)
    Space Complexity: O(V)

    Use Cases:
    - Task scheduling with dependencies
    - Build systems
    - Course prerequisites
    """
    visited = set()
    stack = []

    def dfs(node: int):
        visited.add(node)

        for neighbor, _ in graph.get_neighbors(node):
            if neighbor not in visited:
                dfs(neighbor)

        # Add to stack after visiting all descendants
        stack.append(node)

    # Visit all nodes
    for node in range(num_vertices):
        if node not in visited:
            dfs(node)

    # Reverse to get topological order
    return stack[::-1]

def topological_sort_kahn(graph: Graph, num_vertices: int) -> List[int]:
    """
    Topological sort using Kahn's algorithm (BFS-based)

    Repeatedly removes vertices with no incoming edges.
    Can detect cycles (result length < num_vertices).

    Time: O(V + E)
    Space: O(V)
    """
    # Calculate in-degree for each vertex
    in_degree = [0] * num_vertices

    for node in graph.graph:
        for neighbor, _ in graph.get_neighbors(node):
            in_degree[neighbor] += 1

    # Queue of vertices with no incoming edges
    queue = deque([i for i in range(num_vertices) if in_degree[i] == 0])
    result = []

    while queue:
        node = queue.popleft()
        result.append(node)

        # Remove edges from this node
        for neighbor, _ in graph.get_neighbors(node):
            in_degree[neighbor] -= 1

            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    # Check for cycle
    if len(result) != num_vertices:
        return []  # Cycle detected

    return result
```

---

## Real-World Applications

### 1. Social Network (Find Shortest Connection)

```python
class SocialNetwork:
    """Model social network as graph"""

    def __init__(self):
        self.graph = Graph(directed=False)
        self.users = {}

    def add_user(self, user_id: int, name: str):
        """Add user to network"""
        self.users[user_id] = name

    def add_friendship(self, user1: int, user2: int):
        """Add friendship (bidirectional)"""
        self.graph.add_edge(user1, user2)

    def degrees_of_separation(self, user1: int, user2: int) -> int:
        """
        Find degrees of separation between two users

        Uses BFS to find shortest path length.
        """
        if user1 == user2:
            return 0

        visited = {user1}
        queue = deque([(user1, 0)])

        while queue:
            user, degree = queue.popleft()

            for neighbor, _ in self.graph.get_neighbors(user):
                if neighbor == user2:
                    return degree + 1

                if neighbor not in visited:
                    visited.add(neighbor)
                    queue.append((neighbor, degree + 1))

        return -1  # Not connected

    def mutual_friends(self, user1: int, user2: int) -> List[int]:
        """Find mutual friends"""
        friends1 = {n for n, _ in self.graph.get_neighbors(user1)}
        friends2 = {n for n, _ in self.graph.get_neighbors(user2)}

        return list(friends1 & friends2)  # Intersection

# Usage
network = SocialNetwork()
network.add_user(1, "Alice")
network.add_user(2, "Bob")
network.add_user(3, "Charlie")

network.add_friendship(1, 2)
network.add_friendship(2, 3)

degrees = network.degrees_of_separation(1, 3)  # 2 (through Bob)
```

### 2. Route Planning (GPS Navigation)

```python
class RouteMap:
    """Route planning with weighted graph"""

    def __init__(self):
        self.graph = Graph(directed=False)
        self.locations = {}

    def add_location(self, loc_id: int, name: str):
        """Add location"""
        self.locations[loc_id] = name

    def add_road(self, loc1: int, loc2: int, distance: int):
        """Add road with distance"""
        self.graph.add_edge(loc1, loc2, distance)

    def find_shortest_route(
        self,
        start: int,
        end: int
    ) -> Tuple[int, List[str]]:
        """
        Find shortest route using Dijkstra

        Returns:
            (total_distance, [location names])
        """
        distance, path = dijkstra_with_path(self.graph, start, end)

        if not path:
            return float('inf'), []

        # Convert IDs to names
        route = [self.locations[loc_id] for loc_id in path]

        return distance, route

# Usage
road_map = RouteMap()
road_map.add_location(0, "Home")
road_map.add_location(1, "School")
road_map.add_location(2, "Work")
road_map.add_location(3, "Store")

road_map.add_road(0, 1, 5)   # Home -> School: 5 km
road_map.add_road(1, 2, 10)  # School -> Work: 10 km
road_map.add_road(0, 3, 3)   # Home -> Store: 3 km
road_map.add_road(3, 2, 8)   # Store -> Work: 8 km

distance, route = road_map.find_shortest_route(0, 2)
# distance = 11, route = ['Home', 'Store', 'Work']
```

---

## Best Practices

### ✅ Do's:

1. **Choose right representation** - List for sparse, matrix for dense
2. **Use BFS for shortest path** - Unweighted graphs
3. **Use Dijkstra for weighted** - Non-negative weights
4. **Use Bellman-Ford for negative** - Or cycle detection
5. **Consider space** - DFS iterative for deep graphs
6. **Handle disconnected graphs** - Visit all components

### ❌ Don'ts:

1. **Don't use recursion for large graphs** - Stack overflow risk
2. **Don't use Dijkstra with negative weights** - Wrong results
3. **Don't forget visited set** - Infinite loops
4. **Don't modify graph during traversal** - Undefined behavior

---

## Interview Questions

### Q1: BFS vs DFS - when to use?

**Answer**:

- **BFS**: Shortest path, level-order, closer nodes first
- **DFS**: Topological sort, cycle detection, exploring all paths
- **BFS queue**: O(V) space worst case
- **DFS stack**: O(V) space, O(h) for tree (height)
  Choose based on problem requirements.

### Q2: Why Dijkstra doesn't work with negative weights?

**Answer**: Greedy choice assumption violated:

- Assumes shortest path to visited node won't improve
- Negative weight can make revisiting beneficial
- Example: A→B (5), A→C (1), C→B (-10) = A→C→B (-9) shorter
  Use Bellman-Ford for negative weights instead.

### Q3: How to detect cycle in graph?

**Answer**:

- **Undirected**: DFS tracking parent, if visited neighbor ≠ parent → cycle
- **Directed**: DFS with three colors (white/gray/black), gray neighbor → cycle
- **Or**: Topological sort, if result length < num_vertices → cycle
  Detection: O(V + E) time.

### Q4: Adjacency list vs matrix?

**Answer**:

- **List**: O(V + E) space, O(degree) neighbor lookup, sparse graphs
- **Matrix**: O(V²) space, O(1) edge check, dense graphs
- **List better**: E << V², most real graphs sparse
- **Matrix better**: Need fast edge checks, dense graph
  Trade-off between space and query speed.

### Q5: What is topological sort used for?

**Answer**: Ordering with dependencies:

- **Task scheduling**: Execute tasks respecting dependencies
- **Build systems**: Compile order for modules
- **Course planning**: Prerequisites ordering
- **Requirements**: DAG only (no cycles)
- **Algorithms**: DFS (reverse post-order) or Kahn's (BFS)
  Essential for dependency resolution.

---

## Summary

Graph algorithms essentials:

- **Traversal**: BFS (queue, shortest path), DFS (stack, explore deep)
- **Shortest path**: Dijkstra (non-negative), Bellman-Ford (negative), Floyd-Warshall (all-pairs)
- **Topological sort**: DFS or Kahn's for DAGs
- **Representation**: Adjacency list (sparse), matrix (dense)
- **Applications**: Social networks, routing, scheduling

Master graphs for system design! 🕸️
