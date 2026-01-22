# 🌳 Tree Algorithms

## Overview

Tree data structures and algorithms - binary trees, BST, heaps, and tree traversal patterns essential for interviews.

---

## Tree Basics

### Tree Terminology

```python
"""
Tree Structure:
        1          <- Root (level 0)
       / \
      2   3        <- Level 1
     / \   \
    4   5   6      <- Level 2 (leaves: 4, 5, 6)

Terms:
- Root: Top node (1)
- Parent: Node with children (1 is parent of 2, 3)
- Child: Node connected below (2, 3 are children of 1)
- Leaf: Node with no children (4, 5, 6)
- Siblings: Nodes with same parent (2 and 3)
- Depth: Distance from root (depth of 4 = 2)
- Height: Longest path to leaf (height of tree = 2)
- Subtree: Node and all descendants

Binary Tree:
- Each node has at most 2 children (left, right)
- Complete: All levels filled except last, filled left-to-right
- Perfect: All internal nodes have 2 children, all leaves at same level
- Balanced: Height difference of left and right subtrees ≤ 1
"""

from typing import Optional, List
from collections import deque

class TreeNode:
    """Binary tree node"""

    def __init__(self, val: int = 0, left: 'TreeNode' = None, right: 'TreeNode' = None):
        self.val = val
        self.left = left
        self.right = right

    def __repr__(self):
        return f"TreeNode({self.val})"
```

---

## Tree Traversal

### 1. Inorder Traversal (Left → Root → Right)

```python
def inorder_recursive(root: TreeNode) -> List[int]:
    """
    Inorder traversal (recursive)

    Order: Left subtree → Root → Right subtree
    For BST: Returns sorted values

    Time Complexity: O(n)
    Space Complexity: O(h) for recursion stack (h = height)
    """
    result = []

    def traverse(node):
        if not node:
            return

        traverse(node.left)   # Visit left
        result.append(node.val)  # Visit root
        traverse(node.right)  # Visit right

    traverse(root)
    return result

def inorder_iterative(root: TreeNode) -> List[int]:
    """
    Inorder traversal (iterative)

    Using explicit stack instead of recursion.
    Safer for deep trees (no stack overflow).

    Time: O(n)
    Space: O(h) for stack
    """
    result = []
    stack = []
    current = root

    while current or stack:
        # Go to leftmost node
        while current:
            stack.append(current)
            current = current.left

        # Visit node
        current = stack.pop()
        result.append(current.val)

        # Move to right subtree
        current = current.right

    return result

# Example:
#       1
#      / \
#     2   3
#    / \
#   4   5
# Inorder: [4, 2, 5, 1, 3]
```

### 2. Preorder Traversal (Root → Left → Right)

```python
def preorder_recursive(root: TreeNode) -> List[int]:
    """
    Preorder traversal (recursive)

    Order: Root → Left subtree → Right subtree
    Use case: Copy tree structure

    Time: O(n)
    Space: O(h)
    """
    result = []

    def traverse(node):
        if not node:
            return

        result.append(node.val)  # Visit root first
        traverse(node.left)
        traverse(node.right)

    traverse(root)
    return result

def preorder_iterative(root: TreeNode) -> List[int]:
    """
    Preorder traversal (iterative)

    Time: O(n)
    Space: O(h)
    """
    if not root:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)

        # Push right first (so left is processed first)
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)

    return result

# Example:
#       1
#      / \
#     2   3
#    / \
#   4   5
# Preorder: [1, 2, 4, 5, 3]
```

### 3. Postorder Traversal (Left → Right → Root)

```python
def postorder_recursive(root: TreeNode) -> List[int]:
    """
    Postorder traversal (recursive)

    Order: Left subtree → Right subtree → Root
    Use case: Delete tree (delete children before parent)

    Time: O(n)
    Space: O(h)
    """
    result = []

    def traverse(node):
        if not node:
            return

        traverse(node.left)
        traverse(node.right)
        result.append(node.val)  # Visit root last

    traverse(root)
    return result

def postorder_iterative(root: TreeNode) -> List[int]:
    """
    Postorder traversal (iterative)

    More complex - need to track visited nodes.

    Time: O(n)
    Space: O(h)
    """
    if not root:
        return []

    result = []
    stack = [root]
    visited = set()

    while stack:
        node = stack[-1]  # Peek

        # If leaf or both children visited
        if not node.left and not node.right or \
           (node.left in visited or not node.left) and \
           (node.right in visited or not node.right):
            stack.pop()
            result.append(node.val)
            visited.add(node)
        else:
            # Push children (right first so left is processed first)
            if node.right and node.right not in visited:
                stack.append(node.right)
            if node.left and node.left not in visited:
                stack.append(node.left)

    return result

# Example:
#       1
#      / \
#     2   3
#    / \
#   4   5
# Postorder: [4, 5, 2, 3, 1]
```

### 4. Level-Order Traversal (BFS)

```python
def level_order(root: TreeNode) -> List[List[int]]:
    """
    Level-order traversal (BFS)

    Visit nodes level by level, left to right.
    Returns list of lists (one per level).

    Time: O(n)
    Space: O(w) where w = max width of tree
    """
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        current_level = []

        for _ in range(level_size):
            node = queue.popleft()
            current_level.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(current_level)

    return result

# Example:
#       1
#      / \
#     2   3
#    / \
#   4   5
# Level-order: [[1], [2, 3], [4, 5]]

def zigzag_level_order(root: TreeNode) -> List[List[int]]:
    """
    Zigzag level-order traversal

    Level 0: left to right
    Level 1: right to left
    Level 2: left to right, etc.

    Time: O(n)
    Space: O(w)
    """
    if not root:
        return []

    result = []
    queue = deque([root])
    left_to_right = True

    while queue:
        level_size = len(queue)
        current_level = deque()

        for _ in range(level_size):
            node = queue.popleft()

            # Add to level based on direction
            if left_to_right:
                current_level.append(node.val)
            else:
                current_level.appendleft(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(list(current_level))
        left_to_right = not left_to_right

    return result

# Example:
#       1
#      / \
#     2   3
#    / \
#   4   5
# Zigzag: [[1], [3, 2], [4, 5]]
```

---

## Binary Search Tree (BST)

### BST Operations

```python
class BST:
    """Binary Search Tree"""

    def __init__(self):
        self.root = None

    def insert(self, val: int):
        """
        Insert value into BST

        Time: O(h) average O(log n), worst O(n) for skewed tree
        Space: O(h) for recursion
        """
        def insert_helper(node, val):
            if not node:
                return TreeNode(val)

            if val < node.val:
                node.left = insert_helper(node.left, val)
            else:
                node.right = insert_helper(node.right, val)

            return node

        self.root = insert_helper(self.root, val)

    def search(self, val: int) -> bool:
        """
        Search for value in BST

        Time: O(h)
        Space: O(1) iterative
        """
        current = self.root

        while current:
            if val == current.val:
                return True
            elif val < current.val:
                current = current.left
            else:
                current = current.right

        return False

    def delete(self, val: int):
        """
        Delete value from BST

        Cases:
        1. Node is leaf: Simply remove
        2. Node has one child: Replace with child
        3. Node has two children: Replace with inorder successor (min in right subtree)

        Time: O(h)
        Space: O(h) for recursion
        """
        def delete_helper(node, val):
            if not node:
                return None

            if val < node.val:
                node.left = delete_helper(node.left, val)
            elif val > node.val:
                node.right = delete_helper(node.right, val)
            else:
                # Found node to delete

                # Case 1: Leaf or one child
                if not node.left:
                    return node.right
                if not node.right:
                    return node.left

                # Case 2: Two children
                # Find inorder successor (min in right subtree)
                successor = self._find_min(node.right)
                node.val = successor.val
                node.right = delete_helper(node.right, successor.val)

            return node

        self.root = delete_helper(self.root, val)

    def _find_min(self, node: TreeNode) -> TreeNode:
        """Find minimum node in subtree"""
        while node.left:
            node = node.left
        return node

    def validate_bst(self, root: TreeNode) -> bool:
        """
        Validate if tree is valid BST

        For each node:
        - All left descendants < node.val
        - All right descendants > node.val

        Time: O(n)
        Space: O(h)
        """
        def is_valid(node, min_val, max_val):
            if not node:
                return True

            if node.val <= min_val or node.val >= max_val:
                return False

            return (is_valid(node.left, min_val, node.val) and
                    is_valid(node.right, node.val, max_val))

        return is_valid(root, float('-inf'), float('inf'))

# Example:
#       5
#      / \
#     3   7
#    / \ / \
#   2  4 6  8
# BST: Valid (left < root < right)
```

---

## Common Tree Problems

### 1. Maximum Depth

```python
def max_depth(root: TreeNode) -> int:
    """
    Find maximum depth of tree

    Depth = number of nodes along longest path from root to leaf.

    Time: O(n)
    Space: O(h)
    """
    if not root:
        return 0

    left_depth = max_depth(root.left)
    right_depth = max_depth(root.right)

    return 1 + max(left_depth, right_depth)

def max_depth_iterative(root: TreeNode) -> int:
    """Maximum depth (iterative BFS)"""
    if not root:
        return 0

    depth = 0
    queue = deque([root])

    while queue:
        depth += 1
        level_size = len(queue)

        for _ in range(level_size):
            node = queue.popleft()

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

    return depth
```

### 2. Balanced Binary Tree

```python
def is_balanced(root: TreeNode) -> bool:
    """
    Check if tree is balanced

    Balanced: For every node, height difference between
    left and right subtrees ≤ 1.

    Time: O(n)
    Space: O(h)
    """
    def check_height(node):
        if not node:
            return 0

        left_height = check_height(node.left)
        if left_height == -1:
            return -1  # Left subtree not balanced

        right_height = check_height(node.right)
        if right_height == -1:
            return -1  # Right subtree not balanced

        # Check current node
        if abs(left_height - right_height) > 1:
            return -1  # Current node not balanced

        return 1 + max(left_height, right_height)

    return check_height(root) != -1
```

### 3. Lowest Common Ancestor (LCA)

```python
def lowest_common_ancestor(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
    """
    Find lowest common ancestor of two nodes

    LCA: Deepest node that has both p and q as descendants.

    Time: O(n)
    Space: O(h)
    """
    if not root or root == p or root == q:
        return root

    # Search in left and right subtrees
    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)

    # If both found in different subtrees, current node is LCA
    if left and right:
        return root

    # Return non-null side
    return left if left else right

def lca_bst(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
    """
    LCA for Binary Search Tree (optimized)

    Use BST property: left < root < right

    Time: O(h)
    Space: O(1) iterative
    """
    current = root

    while current:
        # Both in left subtree
        if p.val < current.val and q.val < current.val:
            current = current.left
        # Both in right subtree
        elif p.val > current.val and q.val > current.val:
            current = current.right
        else:
            # Split point - this is LCA
            return current

    return None
```

### 4. Serialize and Deserialize

```python
def serialize(root: TreeNode) -> str:
    """
    Serialize tree to string

    Use preorder traversal with 'null' for None nodes.

    Example:
        1
       / \
      2   3
         / \
        4   5

    Output: "1,2,null,null,3,4,null,null,5,null,null"

    Time: O(n)
    Space: O(n)
    """
    def serialize_helper(node):
        if not node:
            return 'null'

        return f"{node.val},{serialize_helper(node.left)},{serialize_helper(node.right)}"

    return serialize_helper(root)

def deserialize(data: str) -> TreeNode:
    """
    Deserialize string to tree

    Time: O(n)
    Space: O(n)
    """
    def deserialize_helper(values):
        val = next(values)

        if val == 'null':
            return None

        node = TreeNode(int(val))
        node.left = deserialize_helper(values)
        node.right = deserialize_helper(values)

        return node

    values = iter(data.split(','))
    return deserialize_helper(values)
```

### 5. Path Sum

```python
def has_path_sum(root: TreeNode, target_sum: int) -> bool:
    """
    Check if tree has root-to-leaf path with sum = target_sum

    Time: O(n)
    Space: O(h)
    """
    if not root:
        return False

    # Leaf node - check sum
    if not root.left and not root.right:
        return root.val == target_sum

    # Recurse with remaining sum
    remaining = target_sum - root.val
    return (has_path_sum(root.left, remaining) or
            has_path_sum(root.right, remaining))

def path_sum_all(root: TreeNode, target_sum: int) -> List[List[int]]:
    """
    Find all root-to-leaf paths with sum = target_sum

    Time: O(n²) - copying paths
    Space: O(n)
    """
    result = []

    def dfs(node, remaining, path):
        if not node:
            return

        path.append(node.val)

        # Leaf node
        if not node.left and not node.right:
            if node.val == remaining:
                result.append(path[:])  # Copy path
        else:
            dfs(node.left, remaining - node.val, path)
            dfs(node.right, remaining - node.val, path)

        path.pop()  # Backtrack

    dfs(root, target_sum, [])
    return result
```

---

## Real-World Applications

### File System

```python
class FileNode:
    """File system node (directory or file)"""

    def __init__(self, name: str, is_file: bool = False, size: int = 0):
        self.name = name
        self.is_file = is_file
        self.size = size  # For files
        self.children = {}  # name -> FileNode

    def add_child(self, child: 'FileNode'):
        """Add file or directory"""
        self.children[child.name] = child

    def get_size(self) -> int:
        """Get total size (recursive for directories)"""
        if self.is_file:
            return self.size

        return sum(child.get_size() for child in self.children.values())

    def find(self, name: str) -> Optional['FileNode']:
        """Find file or directory by name (DFS)"""
        if self.name == name:
            return self

        for child in self.children.values():
            found = child.find(name)
            if found:
                return found

        return None

    def list_files(self, extension: str = None) -> List[str]:
        """List all files (optionally filtered by extension)"""
        files = []

        if self.is_file:
            if extension is None or self.name.endswith(extension):
                files.append(self.name)
        else:
            for child in self.children.values():
                files.extend(child.list_files(extension))

        return files

# Usage
root = FileNode("/", is_file=False)
home = FileNode("home", is_file=False)
user = FileNode("user", is_file=False)
doc1 = FileNode("doc1.txt", is_file=True, size=100)
doc2 = FileNode("doc2.pdf", is_file=True, size=200)

root.add_child(home)
home.add_child(user)
user.add_child(doc1)
user.add_child(doc2)

total_size = root.get_size()  # 300
txt_files = root.list_files('.txt')  # ['doc1.txt']
```

---

## Best Practices

### ✅ Do's:

1. **Consider iterative** - For deep trees (avoid stack overflow)
2. **Use BFS for levels** - Level-order problems
3. **Use DFS for paths** - Root-to-leaf paths
4. **Validate input** - Check for null root
5. **Consider BST property** - Optimize with sorted order
6. **Use recursion carefully** - Beautiful but can overflow

### ❌ Don'ts:

1. **Don't forget base cases** - Null checks crucial
2. **Don't modify during traversal** - Unless intended
3. **Don't assume balanced** - Could be skewed (O(n) height)
4. **Don't forget to return** - In recursive calls

---

## Interview Questions

### Q1: Inorder vs preorder vs postorder?

**Answer**:

- **Inorder** (L→Root→R): BST gives sorted, evaluate expressions
- **Preorder** (Root→L→R): Copy tree, prefix expressions
- **Postorder** (L→R→Root): Delete tree, postfix expressions
- **Level-order** (BFS): Level by level, shortest path
  Choose based on problem requirements.

### Q2: Recursive vs iterative traversal?

**Answer**:

- **Recursive**: Cleaner code, easier to write
- **Iterative**: No stack overflow, explicit control
- **Space**: Both O(h), but recursion uses call stack
- **When**: Recursive for simple, iterative for deep trees
  Iterative safer for production.

### Q3: How to check if BST is valid?

**Answer**:

- **Wrong**: Just check node.left < node < node.right
- **Correct**: Check entire left subtree < node < entire right subtree
- **Solution**: Pass min/max bounds down recursion
- **Time**: O(n) - visit each node once
- **Common mistake**: Only checking immediate children

### Q4: What's tree height vs depth?

**Answer**:

- **Height**: Longest path from node to leaf (bottom-up)
- **Depth**: Distance from root to node (top-down)
- **Tree height**: Height of root
- **Node depth**: Distance from root to that node
- **Leaf height**: 0 (no children)
  Root has depth 0, leaves have height 0.

### Q5: When to use BST vs heap?

**Answer**:

- **BST**: Search O(log n), inorder gives sorted, range queries
- **Heap**: Fast min/max O(1), insert/delete O(log n), priority queue
- **BST better**: Need sorted order, search specific values
- **Heap better**: Only need min/max, priority queue
- **Trade-off**: BST more flexible, heap faster for min/max
  Choose based on access pattern.

---

## Summary

Tree algorithms essentials:

- **Traversals**: Inorder (sorted BST), preorder (copy), postorder (delete), level-order (BFS)
- **BST**: Insert/search/delete O(h), validate with bounds
- **Common problems**: Depth, balanced, LCA, serialize, path sum
- **Recursion**: Clean but can overflow, use iterative for deep trees
- **Applications**: File systems, expression trees, decision trees

Master trees for technical interviews! 🌳
