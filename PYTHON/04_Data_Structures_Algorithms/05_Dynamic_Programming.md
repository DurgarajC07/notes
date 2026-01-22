# 🔄 Dynamic Programming Patterns

## Overview

Dynamic Programming (DP) is an optimization technique that solves complex problems by breaking them down into simpler subproblems and storing results to avoid redundant calculations.

---

## Core Concepts

### Two Key Properties

```python
"""
Dynamic Programming requires:

1. Overlapping Subproblems:
   - Problem can be broken into subproblems
   - Subproblems are reused multiple times
   - Without DP: Exponential time (recalculating)
   - With DP: Polynomial time (caching)

2. Optimal Substructure:
   - Optimal solution contains optimal solutions to subproblems
   - Can build solution from smaller solutions
   - Example: Shortest path uses shortest subpaths

When to use DP:
✅ Optimization problems (min/max)
✅ Counting problems (how many ways)
✅ Existence problems (is it possible)
✅ Recursive solution with repeated subproblems
"""
```

---

## DP Approaches

### 1. Memoization (Top-Down)

```python
from functools import lru_cache
from typing import Dict

# Classic example: Fibonacci
def fibonacci_naive(n: int) -> int:
    """
    Naive recursive - O(2^n) time

    Problem: Recalculates same values many times
    fib(5) calls fib(4) and fib(3)
    fib(4) calls fib(3) and fib(2)
    fib(3) calculated twice!
    """
    if n <= 1:
        return n

    return fibonacci_naive(n - 1) + fibonacci_naive(n - 2)

# Memoization with manual cache
def fibonacci_memo(n: int, memo: Dict[int, int] = None) -> int:
    """
    Memoization - O(n) time, O(n) space

    Cache results to avoid recalculation
    """
    if memo is None:
        memo = {}

    # Base cases
    if n <= 1:
        return n

    # Check cache
    if n in memo:
        return memo[n]

    # Calculate and cache
    memo[n] = fibonacci_memo(n - 1, memo) + fibonacci_memo(n - 2, memo)

    return memo[n]

# Using Python's lru_cache decorator
@lru_cache(maxsize=None)
def fibonacci_cached(n: int) -> int:
    """
    Using lru_cache - O(n) time, O(n) space

    Decorator handles caching automatically
    """
    if n <= 1:
        return n

    return fibonacci_cached(n - 1) + fibonacci_cached(n - 2)

# Performance comparison
import time

n = 35

start = time.time()
result = fibonacci_naive(n)
naive_time = time.time() - start
print(f"Naive: {result} in {naive_time:.4f}s")

start = time.time()
result = fibonacci_memo(n)
memo_time = time.time() - start
print(f"Memo: {result} in {memo_time:.6f}s")

start = time.time()
result = fibonacci_cached(n)
cached_time = time.time() - start
print(f"Cached: {result} in {cached_time:.6f}s")

# Naive: 9227465 in 3.2145s
# Memo: 9227465 in 0.000031s
# Cached: 9227465 in 0.000028s
```

### 2. Tabulation (Bottom-Up)

```python
def fibonacci_tabulation(n: int) -> int:
    """
    Tabulation - O(n) time, O(n) space

    Build solution from bottom up
    No recursion, iterative approach
    """
    if n <= 1:
        return n

    # Create table
    dp = [0] * (n + 1)

    # Base cases
    dp[0] = 0
    dp[1] = 1

    # Fill table bottom-up
    for i in range(2, n + 1):
        dp[i] = dp[i - 1] + dp[i - 2]

    return dp[n]

# Space-optimized version
def fibonacci_optimized(n: int) -> int:
    """
    Optimized - O(n) time, O(1) space

    Only keep last two values
    """
    if n <= 1:
        return n

    prev2 = 0
    prev1 = 1

    for _ in range(2, n + 1):
        current = prev1 + prev2
        prev2 = prev1
        prev1 = current

    return prev1
```

---

## Classic DP Problems

### 1. Climbing Stairs

```python
"""
Problem: Count ways to climb n stairs
Can climb 1 or 2 steps at a time

Example:
n = 3
Ways: [1,1,1], [1,2], [2,1]
Answer: 3
"""

def climb_stairs(n: int) -> int:
    """
    DP solution - O(n) time, O(1) space

    Logic:
    - To reach step i, can come from step i-1 or i-2
    - ways[i] = ways[i-1] + ways[i-2]
    - Same as Fibonacci!
    """
    if n <= 2:
        return n

    prev2 = 1  # 1 way to climb 1 stair
    prev1 = 2  # 2 ways to climb 2 stairs

    for _ in range(3, n + 1):
        current = prev1 + prev2
        prev2 = prev1
        prev1 = current

    return prev1

# Test
print(climb_stairs(3))  # 3
print(climb_stairs(5))  # 8
```

### 2. Coin Change (Minimum Coins)

```python
"""
Problem: Minimum coins needed to make amount
Given coins: [1, 2, 5], amount: 11
Answer: 3 (5 + 5 + 1)
"""

def coin_change_min(coins: list[int], amount: int) -> int:
    """
    DP solution - O(amount * len(coins)) time

    dp[i] = minimum coins to make amount i
    """
    # Initialize with infinity
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0  # 0 coins for amount 0

    # For each amount
    for i in range(1, amount + 1):
        # Try each coin
        for coin in coins:
            if coin <= i:
                # Take min of current and using this coin
                dp[i] = min(dp[i], dp[i - coin] + 1)

    return dp[amount] if dp[amount] != float('inf') else -1

# Test
coins = [1, 2, 5]
print(coin_change_min(coins, 11))  # 3 (5+5+1)
print(coin_change_min(coins, 3))   # 2 (2+1)
print(coin_change_min(coins, 7))   # 2 (5+2)
```

### 3. Coin Change (Count Ways)

```python
"""
Problem: Count ways to make amount
Given coins: [1, 2, 5], amount: 5
Answer: 4 ways ([5], [2,2,1], [2,1,1,1], [1,1,1,1,1])
"""

def coin_change_ways(coins: list[int], amount: int) -> int:
    """
    DP solution - O(amount * len(coins)) time

    dp[i] = number of ways to make amount i
    """
    dp = [0] * (amount + 1)
    dp[0] = 1  # 1 way to make 0 (use no coins)

    # For each coin
    for coin in coins:
        # Update all amounts that can use this coin
        for i in range(coin, amount + 1):
            dp[i] += dp[i - coin]

    return dp[amount]

# Test
coins = [1, 2, 5]
print(coin_change_ways(coins, 5))   # 4
print(coin_change_ways(coins, 10))  # 10
```

### 4. Longest Common Subsequence (LCS)

```python
"""
Problem: Find longest subsequence common to both strings
text1 = "abcde", text2 = "ace"
Answer: 3 ("ace")
"""

def longest_common_subsequence(text1: str, text2: str) -> int:
    """
    DP solution - O(m*n) time, O(m*n) space

    dp[i][j] = LCS length of text1[0:i] and text2[0:j]
    """
    m, n = len(text1), len(text2)

    # Create 2D table
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # Fill table
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                # Characters match - extend LCS
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                # Take max of excluding one character
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    return dp[m][n]

# Reconstruct the actual subsequence
def lcs_string(text1: str, text2: str) -> str:
    """Return the actual LCS string"""
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # Fill table
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    # Reconstruct string
    result = []
    i, j = m, n

    while i > 0 and j > 0:
        if text1[i - 1] == text2[j - 1]:
            result.append(text1[i - 1])
            i -= 1
            j -= 1
        elif dp[i - 1][j] > dp[i][j - 1]:
            i -= 1
        else:
            j -= 1

    return ''.join(reversed(result))

# Test
print(longest_common_subsequence("abcde", "ace"))  # 3
print(lcs_string("abcde", "ace"))  # "ace"
```

### 5. 0/1 Knapsack Problem

```python
"""
Problem: Maximize value without exceeding capacity
Items: [(weight, value), ...]
Capacity: W
Can take each item 0 or 1 time
"""

def knapsack(weights: list[int], values: list[int], capacity: int) -> int:
    """
    DP solution - O(n*W) time, O(n*W) space

    dp[i][w] = max value using items 0..i with capacity w
    """
    n = len(weights)

    # Create 2D table
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    # Fill table
    for i in range(1, n + 1):
        for w in range(capacity + 1):
            # Don't take item i-1
            dp[i][w] = dp[i - 1][w]

            # Try taking item i-1
            if weights[i - 1] <= w:
                take_value = dp[i - 1][w - weights[i - 1]] + values[i - 1]
                dp[i][w] = max(dp[i][w], take_value)

    return dp[n][capacity]

# Space-optimized version
def knapsack_optimized(weights: list[int], values: list[int], capacity: int) -> int:
    """
    Optimized - O(n*W) time, O(W) space

    Only need previous row
    """
    n = len(weights)
    dp = [0] * (capacity + 1)

    for i in range(n):
        # Iterate backwards to avoid using updated values
        for w in range(capacity, weights[i] - 1, -1):
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

    return dp[capacity]

# Test
weights = [1, 3, 4, 5]
values = [1, 4, 5, 7]
capacity = 7
print(knapsack(weights, values, capacity))  # 9 (items 1,2: 4+5)
```

### 6. Longest Increasing Subsequence (LIS)

```python
"""
Problem: Find longest strictly increasing subsequence
nums = [10,9,2,5,3,7,101,18]
Answer: 4 ([2,3,7,101])
"""

def longest_increasing_subsequence(nums: list[int]) -> int:
    """
    DP solution - O(n²) time, O(n) space

    dp[i] = length of LIS ending at index i
    """
    if not nums:
        return 0

    n = len(nums)
    dp = [1] * n  # Each element is LIS of length 1

    for i in range(1, n):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)

    return max(dp)

# Optimized with binary search - O(n log n)
import bisect

def lis_optimized(nums: list[int]) -> int:
    """
    Binary search solution - O(n log n) time

    Maintain array of smallest tail values
    """
    tails = []

    for num in nums:
        # Binary search for insertion position
        pos = bisect.bisect_left(tails, num)

        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num

    return len(tails)

# Test
nums = [10, 9, 2, 5, 3, 7, 101, 18]
print(longest_increasing_subsequence(nums))  # 4
print(lis_optimized(nums))  # 4
```

---

## Real-World Applications

### E-commerce: Product Recommendations

```python
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class Product:
    """Product model"""
    id: int
    name: str
    price: float
    category: str
    rating: float

def max_value_products(
    products: List[Product],
    budget: float
) -> List[Product]:
    """
    Select products to maximize total rating within budget

    This is a knapsack problem:
    - Items: products
    - Weights: prices
    - Values: ratings
    - Capacity: budget
    """
    n = len(products)
    budget_cents = int(budget * 100)  # Avoid float precision issues

    # DP table
    dp = [[0] * (budget_cents + 1) for _ in range(n + 1)]

    # Fill table
    for i in range(1, n + 1):
        product = products[i - 1]
        price_cents = int(product.price * 100)

        for b in range(budget_cents + 1):
            # Don't take product
            dp[i][b] = dp[i - 1][b]

            # Try taking product
            if price_cents <= b:
                take_value = dp[i - 1][b - price_cents] + product.rating
                dp[i][b] = max(dp[i][b], take_value)

    # Reconstruct selected products
    selected = []
    b = budget_cents

    for i in range(n, 0, -1):
        if dp[i][b] != dp[i - 1][b]:
            selected.append(products[i - 1])
            b -= int(products[i - 1].price * 100)

    return selected

# Example
products = [
    Product(1, "Laptop", 999.99, "Electronics", 4.5),
    Product(2, "Mouse", 29.99, "Electronics", 4.2),
    Product(3, "Keyboard", 79.99, "Electronics", 4.3),
    Product(4, "Monitor", 299.99, "Electronics", 4.6),
]

selected = max_value_products(products, budget=400)
print(f"Selected products: {[p.name for p in selected]}")
print(f"Total rating: {sum(p.rating for p in selected):.1f}")
```

### Text Processing: Edit Distance

```python
def edit_distance(word1: str, word2: str) -> int:
    """
    Minimum edits to convert word1 to word2

    Operations: insert, delete, replace

    Use case: Spell checking, autocorrect, DNA sequencing
    """
    m, n = len(word1), len(word2)

    # dp[i][j] = min edits to convert word1[0:i] to word2[0:j]
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # Base cases
    for i in range(m + 1):
        dp[i][0] = i  # Delete all characters

    for j in range(n + 1):
        dp[0][j] = j  # Insert all characters

    # Fill table
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i - 1] == word2[j - 1]:
                # Characters match - no edit needed
                dp[i][j] = dp[i - 1][j - 1]
            else:
                # Try all operations, take minimum
                dp[i][j] = 1 + min(
                    dp[i - 1][j],      # Delete from word1
                    dp[i][j - 1],      # Insert to word1
                    dp[i - 1][j - 1]   # Replace in word1
                )

    return dp[m][n]

# Use for spell checking
def suggest_correction(word: str, dictionary: List[str], max_distance: int = 2) -> List[str]:
    """Suggest corrections based on edit distance"""
    suggestions = []

    for dict_word in dictionary:
        distance = edit_distance(word, dict_word)

        if distance <= max_distance:
            suggestions.append((dict_word, distance))

    # Sort by distance
    suggestions.sort(key=lambda x: x[1])

    return [word for word, _ in suggestions]

# Example
dictionary = ["hello", "world", "help", "hero", "hold"]
print(suggest_correction("helo", dictionary))  # ['hello', 'help', 'hero']
```

---

## DP Problem-Solving Strategy

### Step-by-Step Approach

```python
"""
How to solve DP problems:

1. Identify DP problem:
   ✓ Optimization (min/max)
   ✓ Counting (how many ways)
   ✓ Recursive with overlapping subproblems

2. Define state:
   - What information do you need?
   - dp[i] represents what?
   - Example: dp[i] = max profit up to day i

3. Find recurrence relation:
   - How to compute dp[i] from smaller subproblems?
   - Example: dp[i] = max(dp[i-1], dp[i-2] + nums[i])

4. Base cases:
   - What are the simplest cases?
   - Example: dp[0] = nums[0], dp[1] = max(nums[0], nums[1])

5. Compute order:
   - Memoization: Top-down, recursive
   - Tabulation: Bottom-up, iterative

6. Optimize space:
   - Do you need entire table?
   - Can you use rolling array?
   - Example: Only need last 2 values
"""
```

---

## Best Practices

### ✅ Do's:

1. **Start with recursion** - Define recursive solution first
2. **Add memoization** - Cache results to avoid recalculation
3. **Convert to tabulation** - Bottom-up for better performance
4. **Optimize space** - Use O(1) space when possible
5. **Test edge cases** - Empty input, single element
6. **Document state** - What dp[i] represents
7. **Draw table** - Visualize for debugging

### ❌ Don'ts:

1. **Don't jump** to DP immediately - Try simpler approaches first
2. **Don't forget** base cases
3. **Don't use** wrong dimensions for table
4. **Don't ignore** space optimization opportunities
5. **Don't over-complicate** - Start simple

---

## Interview Questions

### Q1: How to identify DP problem?

**Answer**: Look for these patterns:

- **Optimization**: Find min/max value
- **Counting**: Count number of ways
- **Decision**: Yes/no, possible/impossible
- **Overlapping subproblems**: Same calculation repeated
- **Keywords**: "maximize", "minimize", "count ways", "longest"
  Classic signs: optimal substructure + overlapping subproblems

### Q2: Memoization vs Tabulation?

**Answer**:

- **Memoization**: Top-down, recursive, cache results
- **Tabulation**: Bottom-up, iterative, fill table
- **Memoization pros**: Easier to code, only computes needed
- **Tabulation pros**: Better space, no recursion overhead
  Use memoization for initial solution, optimize to tabulation.

### Q3: How to optimize DP space?

**Answer**:

- **Observation**: Often only need previous row/few values
- **Technique**: Use 1D array instead of 2D
- **Example**: Fibonacci only needs last 2 values
- **Rolling array**: Reuse same array with modulo
- **Trade-off**: Can't reconstruct solution path
  Analyze dependencies between states.

### Q4: Common DP patterns?

**Answer**:

- **Linear DP**: dp[i] depends on dp[i-1], dp[i-2]...
- **2D DP**: dp[i][j] for two sequences/dimensions
- **Knapsack**: Subset selection with constraints
- **Subsequence**: LCS, LIS
- **Path problems**: Min/max path sum
  Recognize pattern, apply template.

### Q5: When not to use DP?

**Answer**:

- **No overlapping**: Divide and conquer better
- **Too many states**: Exponential space
- **Greedy works**: Simpler and faster
- **Requirements change**: Hard to modify DP
- **Example**: Binary search (no overlapping subproblems)
  Use simplest solution that works.

---

## Summary

Dynamic Programming essentials:

- **Two properties**: Overlapping subproblems, optimal substructure
- **Two approaches**: Memoization (top-down), tabulation (bottom-up)
- **Patterns**: Knapsack, LCS, LIS, coin change
- **Strategy**: Recursive → Memoize → Tabulate → Optimize space
- **Applications**: Optimization, counting, decision problems
- **Practice**: LeetCode DP tag, common patterns

Master DP for interviews! 🔄
