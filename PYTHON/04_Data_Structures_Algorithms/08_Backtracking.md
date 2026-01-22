# 🔄 Backtracking Algorithms

## Overview

Backtracking - a systematic way to explore all possible solutions by building candidates incrementally and abandoning them when they can't lead to a valid solution.

---

## What is Backtracking?

```python
"""
Backtracking = Depth-First Search + Pruning

Core Idea:
1. Build solution incrementally (one choice at a time)
2. Check if current partial solution is valid
3. If valid, continue building
4. If invalid, backtrack (undo last choice)
5. Try next possibility

Template:
def backtrack(state, choices):
    if is_solution(state):
        record(state)
        return

    for choice in choices:
        if is_valid(choice, state):
            make_choice(choice)
            backtrack(new_state, remaining_choices)
            undo_choice(choice)  # Backtrack!

Time Complexity: Often exponential O(2^n) or O(n!)
Space Complexity: O(depth) for recursion stack

When to Use:
✅ Generate all combinations/permutations
✅ Constraint satisfaction problems
✅ Puzzles (Sudoku, N-Queens)
✅ Path finding with constraints
✅ Subset generation

When NOT to Use:
❌ Can solve with greedy approach
❌ Dynamic programming applicable
❌ Simple iteration works
"""
```

---

## Classic Problems

### 1. Permutations

```python
from typing import List

def permutations(nums: List[int]) -> List[List[int]]:
    """
    Generate all permutations of array

    Example: [1, 2, 3] → [[1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,1,2], [3,2,1]]

    Time: O(n! × n) - n! permutations, n to copy each
    Space: O(n) for recursion stack

    Approach:
    - Fix first element, permute rest
    - Swap elements to generate permutations
    - Backtrack by swapping back
    """
    result = []

    def backtrack(start: int):
        # Base case: completed permutation
        if start == len(nums):
            result.append(nums[:])  # Copy current state
            return

        # Try each element as next in permutation
        for i in range(start, len(nums)):
            # Make choice: swap current element to position
            nums[start], nums[i] = nums[i], nums[start]

            # Recurse with next position
            backtrack(start + 1)

            # Undo choice: backtrack
            nums[start], nums[i] = nums[i], nums[start]

    backtrack(0)
    return result

# Example
print(permutations([1, 2, 3]))
# [[1, 2, 3], [1, 3, 2], [2, 1, 3], [2, 3, 1], [3, 1, 2], [3, 2, 1]]

def permutations_with_used(nums: List[int]) -> List[List[int]]:
    """
    Alternative: Track used elements

    Better when need to handle duplicates.
    """
    result = []
    used = [False] * len(nums)
    current = []

    def backtrack():
        if len(current) == len(nums):
            result.append(current[:])
            return

        for i in range(len(nums)):
            if used[i]:
                continue

            # Make choice
            current.append(nums[i])
            used[i] = True

            backtrack()

            # Undo choice
            current.pop()
            used[i] = False

    backtrack()
    return result
```

### 2. Combinations

```python
def combinations(n: int, k: int) -> List[List[int]]:
    """
    Generate all combinations of k numbers from 1 to n

    Example: n=4, k=2 → [[1,2], [1,3], [1,4], [2,3], [2,4], [3,4]]

    Time: O(C(n,k) × k) - C(n,k) combinations, k to copy each
    Space: O(k) for recursion and current combination

    Key: To avoid duplicates, only consider elements after current
    """
    result = []
    current = []

    def backtrack(start: int):
        # Base case: found combination of size k
        if len(current) == k:
            result.append(current[:])
            return

        # Try each number from start to n
        for i in range(start, n + 1):
            # Make choice
            current.append(i)

            # Recurse with next number (i+1 to avoid duplicates)
            backtrack(i + 1)

            # Undo choice
            current.pop()

    backtrack(1)
    return result

# Optimization: Early termination
def combinations_optimized(n: int, k: int) -> List[List[int]]:
    """Optimized with pruning"""
    result = []
    current = []

    def backtrack(start: int):
        if len(current) == k:
            result.append(current[:])
            return

        # Pruning: need k - len(current) more elements
        # At least need: start + (k - len(current)) - 1 <= n
        need = k - len(current)
        available = n - start + 1

        if available < need:
            return  # Not enough elements left

        for i in range(start, n + 1):
            current.append(i)
            backtrack(i + 1)
            current.pop()

    backtrack(1)
    return result
```

### 3. Subsets (Power Set)

```python
def subsets(nums: List[int]) -> List[List[int]]:
    """
    Generate all subsets (power set)

    Example: [1,2,3] → [[], [1], [2], [3], [1,2], [1,3], [2,3], [1,2,3]]

    Time: O(2^n × n) - 2^n subsets, n to copy each
    Space: O(n) for recursion

    Two choices for each element: include or exclude
    """
    result = []
    current = []

    def backtrack(start: int):
        # Every state is a valid subset
        result.append(current[:])

        # Try adding each remaining element
        for i in range(start, len(nums)):
            # Include nums[i]
            current.append(nums[i])
            backtrack(i + 1)
            current.pop()  # Exclude nums[i]

    backtrack(0)
    return result

# Iterative approach (also O(2^n))
def subsets_iterative(nums: List[int]) -> List[List[int]]:
    """
    Iterative approach

    For each element, add it to all existing subsets.
    """
    result = [[]]  # Start with empty subset

    for num in nums:
        # Add num to all existing subsets
        new_subsets = [subset + [num] for subset in result]
        result.extend(new_subsets)

    return result

# Bit manipulation approach
def subsets_bits(nums: List[int]) -> List[List[int]]:
    """
    Using bit manipulation

    Each subset corresponds to binary number:
    - Bit 0: include nums[0]?
    - Bit 1: include nums[1]?
    - etc.

    Time: O(2^n × n)
    """
    n = len(nums)
    result = []

    # Iterate through all possible subsets (0 to 2^n - 1)
    for mask in range(1 << n):  # 2^n
        subset = []

        for i in range(n):
            # Check if bit i is set
            if mask & (1 << i):
                subset.append(nums[i])

        result.append(subset)

    return result
```

### 4. N-Queens Problem

```python
def solve_n_queens(n: int) -> List[List[str]]:
    """
    Place n queens on n×n chessboard such that no two queens attack each other

    Constraints:
    - No two queens in same row
    - No two queens in same column
    - No two queens on same diagonal

    Time: O(n!)
    Space: O(n)

    Approach:
    - Place one queen per row
    - Track occupied columns and diagonals
    - Backtrack when no valid position
    """
    result = []
    board = [['.'] * n for _ in range(n)]

    # Track attacked positions
    cols = set()  # Columns with queens
    diag1 = set()  # Positive diagonals (row - col)
    diag2 = set()  # Negative diagonals (row + col)

    def is_safe(row: int, col: int) -> bool:
        """Check if position is safe"""
        return (col not in cols and
                row - col not in diag1 and
                row + col not in diag2)

    def backtrack(row: int):
        # Base case: all queens placed
        if row == n:
            # Convert board to required format
            solution = [''.join(row) for row in board]
            result.append(solution)
            return

        # Try placing queen in each column
        for col in range(n):
            if is_safe(row, col):
                # Make choice
                board[row][col] = 'Q'
                cols.add(col)
                diag1.add(row - col)
                diag2.add(row + col)

                # Recurse to next row
                backtrack(row + 1)

                # Undo choice
                board[row][col] = '.'
                cols.remove(col)
                diag1.remove(row - col)
                diag2.remove(row + col)

    backtrack(0)
    return result

# Example: 4-Queens
solutions = solve_n_queens(4)
for solution in solutions:
    for row in solution:
        print(row)
    print()

"""
Output (2 solutions):
.Q..
...Q
Q...
..Q.

..Q.
Q...
...Q
.Q..
"""
```

### 5. Sudoku Solver

```python
def solve_sudoku(board: List[List[str]]) -> bool:
    """
    Solve Sudoku puzzle

    Rules:
    - Each row has digits 1-9 exactly once
    - Each column has digits 1-9 exactly once
    - Each 3×3 box has digits 1-9 exactly once

    Time: O(9^(empty cells)) - worst case
    Space: O(1) - modify in place

    Approach:
    - Find empty cell
    - Try digits 1-9
    - Check if valid
    - Recurse
    - Backtrack if no solution
    """
    def is_valid(row: int, col: int, num: str) -> bool:
        """Check if placing num at (row, col) is valid"""
        # Check row
        if num in board[row]:
            return False

        # Check column
        if any(board[r][col] == num for r in range(9)):
            return False

        # Check 3×3 box
        box_row, box_col = 3 * (row // 3), 3 * (col // 3)
        for r in range(box_row, box_row + 3):
            for c in range(box_col, box_col + 3):
                if board[r][c] == num:
                    return False

        return True

    def backtrack() -> bool:
        # Find next empty cell
        for row in range(9):
            for col in range(9):
                if board[row][col] == '.':
                    # Try each digit
                    for num in '123456789':
                        if is_valid(row, col, num):
                            # Make choice
                            board[row][col] = num

                            # Recurse
                            if backtrack():
                                return True

                            # Undo choice
                            board[row][col] = '.'

                    # No valid digit found
                    return False

        # No empty cell - solved!
        return True

    backtrack()
    return True

# Optimized: Track constraints
def solve_sudoku_optimized(board: List[List[str]]) -> bool:
    """
    Optimized with constraint tracking

    Track available digits for each row, column, box.
    """
    rows = [set('123456789') for _ in range(9)]
    cols = [set('123456789') for _ in range(9)]
    boxes = [set('123456789') for _ in range(9)]

    # Initialize constraints from existing digits
    for r in range(9):
        for c in range(9):
            if board[r][c] != '.':
                digit = board[r][c]
                rows[r].discard(digit)
                cols[c].discard(digit)
                boxes[(r // 3) * 3 + c // 3].discard(digit)

    def backtrack() -> bool:
        # Find empty cell with fewest possibilities (MRV heuristic)
        min_options = 10
        best_cell = None

        for r in range(9):
            for c in range(9):
                if board[r][c] == '.':
                    box_idx = (r // 3) * 3 + c // 3
                    # Available digits = intersection of constraints
                    available = rows[r] & cols[c] & boxes[box_idx]

                    if len(available) < min_options:
                        min_options = len(available)
                        best_cell = (r, c, available)

        if best_cell is None:
            return True  # Solved

        row, col, available = best_cell
        box_idx = (row // 3) * 3 + col // 3

        # Try each available digit
        for digit in available:
            # Make choice
            board[row][col] = digit
            rows[row].remove(digit)
            cols[col].remove(digit)
            boxes[box_idx].remove(digit)

            if backtrack():
                return True

            # Undo choice
            board[row][col] = '.'
            rows[row].add(digit)
            cols[col].add(digit)
            boxes[box_idx].add(digit)

        return False

    return backtrack()
```

### 6. Word Search

```python
def exist(board: List[List[str]], word: str) -> bool:
    """
    Find if word exists in board

    Rules:
    - Word can be constructed from letters of adjacent cells
    - Same cell can't be used twice
    - Adjacent: horizontally or vertically

    Time: O(M × N × 4^L) where M×N is board size, L is word length
    Space: O(L) for recursion

    Example:
    board = [
        ['A','B','C','E'],
        ['S','F','C','S'],
        ['A','D','E','E']
    ]
    word = "ABCCED" → True
    word = "SEE" → True
    word = "ABCB" → False
    """
    rows, cols = len(board), len(board[0])

    def backtrack(row: int, col: int, index: int) -> bool:
        # Base case: found entire word
        if index == len(word):
            return True

        # Out of bounds or wrong character
        if (row < 0 or row >= rows or col < 0 or col >= cols or
            board[row][col] != word[index]):
            return False

        # Mark cell as visited (modify in place)
        temp = board[row][col]
        board[row][col] = '#'

        # Try all 4 directions
        found = (backtrack(row + 1, col, index + 1) or
                 backtrack(row - 1, col, index + 1) or
                 backtrack(row, col + 1, index + 1) or
                 backtrack(row, col - 1, index + 1))

        # Restore cell (backtrack)
        board[row][col] = temp

        return found

    # Try starting from each cell
    for r in range(rows):
        for c in range(cols):
            if backtrack(r, c, 0):
                return True

    return False
```

---

## Optimization Techniques

### 1. Pruning

```python
def combination_sum(candidates: List[int], target: int) -> List[List[int]]:
    """
    Find all combinations that sum to target

    Can use same number multiple times.

    Pruning: Stop early if sum exceeds target.
    """
    result = []
    current = []

    # Sort to enable pruning
    candidates.sort()

    def backtrack(start: int, remaining: int):
        if remaining == 0:
            result.append(current[:])
            return

        for i in range(start, len(candidates)):
            # Pruning: if current number > remaining, all next will be too
            if candidates[i] > remaining:
                break  # Stop, no need to continue

            current.append(candidates[i])
            backtrack(i, remaining - candidates[i])  # Can reuse same element
            current.pop()

    backtrack(0, target)
    return result
```

### 2. Avoiding Duplicates

```python
def permutations_unique(nums: List[int]) -> List[List[int]]:
    """
    Generate unique permutations (input may have duplicates)

    Example: [1,1,2] → [[1,1,2], [1,2,1], [2,1,1]]

    Key: Sort and skip duplicates at same recursion level.
    """
    result = []
    nums.sort()  # Sort to group duplicates
    used = [False] * len(nums)
    current = []

    def backtrack():
        if len(current) == len(nums):
            result.append(current[:])
            return

        for i in range(len(nums)):
            # Skip if used
            if used[i]:
                continue

            # Skip duplicate: if same as previous and previous not used
            # This ensures we only use duplicate after using previous occurrence
            if i > 0 and nums[i] == nums[i-1] and not used[i-1]:
                continue

            current.append(nums[i])
            used[i] = True

            backtrack()

            current.pop()
            used[i] = False

    backtrack()
    return result
```

---

## Best Practices

### ✅ Do's:

1. **Draw recursion tree** - Visualize the search
2. **Identify base case** - When to stop
3. **Prune early** - Skip invalid branches
4. **Sort when needed** - Enables pruning
5. **Track visited** - Avoid cycles
6. **Consider iterative** - For simple cases

### ❌ Don'ts:

1. **Don't forget to backtrack** - Undo choices
2. **Don't modify global state** - Use parameters
3. **Don't copy unnecessarily** - Copy only when recording solution
4. **Don't ignore optimization** - Pruning crucial

---

## Interview Questions

### Q1: What's difference between backtracking and DFS?

**Answer**:

- **Backtracking**: DFS + undo choices when not leading to solution
- **DFS**: Explore all paths, mark visited
- **Backtracking**: Build solution incrementally, undo when invalid
- **Key**: Backtracking explicitly backtracks (undo state)
- **Example**: N-Queens backtracks when queen can't be placed
  Backtracking is DFS with explicit state management.

### Q2: How to optimize backtracking?

**Answer**:

- **Pruning**: Stop early if can't lead to solution
- **Sorting**: Enable early termination
- **Constraint propagation**: Track what's possible
- **MRV heuristic**: Choose variable with fewest options first
- **Example**: Sudoku - try cell with fewest possibilities
  Pruning can reduce exponential to manageable.

### Q3: Backtracking vs dynamic programming?

**Answer**:

- **Backtracking**: Try all possibilities, undo when wrong
- **DP**: Solve subproblems once, reuse solutions
- **Backtracking**: When need actual solutions (not just count)
- **DP**: When overlapping subproblems, optimal substructure
- **Example**: N-Queens use backtracking, coin change use DP
  Use DP when can memoize, backtracking when building solutions.

### Q4: How to avoid duplicates in backtracking?

**Answer**:

- **Sort input**: Group duplicates together
- **Skip at same level**: If same as previous and previous not used
- **Track used**: Boolean array to mark used elements
- **Set for results**: If duplicates still appear
- **Example**: [1,1,2] permutations, skip second 1 if first 1 not used
  Sorting + skip duplicates at recursion level.

### Q5: When is backtracking too slow?

**Answer**:

- **Exponential time**: O(2^n) or O(n!) grows fast
- **No pruning possible**: Must explore all paths
- **Large input**: n > 20-25 often too slow
- **Alternative**: Greedy, DP, heuristics, approximation
- **Acceptable**: When need all solutions, small input
  Know limits - backtracking not always practical.

---

## Summary

Backtracking essentials:

- **Pattern**: Build incrementally, backtrack when invalid
- **Template**: Choose → recurse → undo
- **Problems**: Permutations, combinations, N-Queens, Sudoku
- **Optimization**: Pruning, sorting, constraint tracking
- **Time**: Often exponential O(2^n) or O(n!)
- **Key**: Undo choices (backtrack) when can't continue

Systematic exploration of solution space! 🔄
