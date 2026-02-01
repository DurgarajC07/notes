# Frontend Coding Challenges

## Table of Contents

- [Introduction](#introduction)
- [Data Structure Challenges](#data-structure-challenges)
- [Algorithm Problems](#algorithm-problems)
- [DOM Manipulation Challenges](#dom-manipulation-challenges)
- [Async Programming Challenges](#async-programming-challenges)
- [System Design Coding](#system-design-coding)
- [React-Specific Challenges](#react-specific-challenges)
- [Performance Optimization Problems](#performance-optimization-problems)
- [TypeScript Advanced Patterns](#typescript-advanced-patterns)
- [Testing Challenges](#testing-challenges)
- [Key Takeaways](#key-takeaways)

## Introduction

Frontend coding challenges test your problem-solving abilities, data structure knowledge, and practical implementation skills. This guide covers common patterns, optimal solutions, and TypeScript implementations.

### Challenge Categories

```typescript
interface CodingChallenge {
  difficulty: "easy" | "medium" | "hard";
  category: "algorithm" | "data-structure" | "dom" | "async" | "system-design";
  timeComplexity: string;
  spaceComplexity: string;
  tags: string[];
}

const challengeApproach = {
  understand: "Clarify requirements and edge cases",
  plan: "Discuss approach and complexity",
  implement: "Write clean, tested code",
  optimize: "Improve time/space complexity",
  test: "Validate with edge cases",
};
```

## Data Structure Challenges

### 1. Implement a LRU Cache

```typescript
/**
 * LRU Cache Implementation
 * Time: O(1) for get/put
 * Space: O(capacity)
 */
class LRUCache<K, V> {
  private capacity: number;
  private cache: Map<K, V>;

  constructor(capacity: number) {
    if (capacity <= 0) {
      throw new Error("Capacity must be positive");
    }
    this.capacity = capacity;
    this.cache = new Map();
  }

  get(key: K): V | undefined {
    if (!this.cache.has(key)) {
      return undefined;
    }

    // Move to end (most recently used)
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  put(key: K, value: V): void {
    // Remove if exists to update position
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }

    // Add to end
    this.cache.set(key, value);

    // Remove oldest if over capacity
    if (this.cache.size > this.capacity) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
  }

  has(key: K): boolean {
    return this.cache.has(key);
  }

  clear(): void {
    this.cache.clear();
  }

  get size(): number {
    return this.cache.size;
  }
}

// Usage Example
const cache = new LRUCache<string, number>(3);
cache.put("a", 1);
cache.put("b", 2);
cache.put("c", 3);
console.log(cache.get("a")); // 1
cache.put("d", 4); // Evicts 'b'
console.log(cache.get("b")); // undefined

// Test Suite
describe("LRUCache", () => {
  it("should evict least recently used item", () => {
    const cache = new LRUCache<string, number>(2);
    cache.put("a", 1);
    cache.put("b", 2);
    cache.put("c", 3);
    expect(cache.has("a")).toBe(false);
    expect(cache.has("b")).toBe(true);
    expect(cache.has("c")).toBe(true);
  });

  it("should update position on get", () => {
    const cache = new LRUCache<string, number>(2);
    cache.put("a", 1);
    cache.put("b", 2);
    cache.get("a"); // Makes 'a' most recent
    cache.put("c", 3); // Should evict 'b'
    expect(cache.has("b")).toBe(false);
  });
});
```

### 2. Implement a Trie (Prefix Tree)

```typescript
class TrieNode {
  children: Map<string, TrieNode>;
  isEndOfWord: boolean;

  constructor() {
    this.children = new Map();
    this.isEndOfWord = false;
  }
}

class Trie {
  private root: TrieNode;

  constructor() {
    this.root = new TrieNode();
  }

  /**
   * Insert word into trie
   * Time: O(m) where m is word length
   */
  insert(word: string): void {
    let node = this.root;

    for (const char of word) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }
      node = node.children.get(char)!;
    }

    node.isEndOfWord = true;
  }

  /**
   * Search for exact word
   * Time: O(m)
   */
  search(word: string): boolean {
    const node = this.findNode(word);
    return node !== null && node.isEndOfWord;
  }

  /**
   * Check if prefix exists
   * Time: O(m)
   */
  startsWith(prefix: string): boolean {
    return this.findNode(prefix) !== null;
  }

  /**
   * Find all words with given prefix
   * Time: O(n) where n is total characters in results
   */
  findWordsWithPrefix(prefix: string): string[] {
    const results: string[] = [];
    const node = this.findNode(prefix);

    if (!node) return results;

    this.dfs(node, prefix, results);
    return results;
  }

  /**
   * Delete word from trie
   * Time: O(m)
   */
  delete(word: string): boolean {
    return this.deleteHelper(this.root, word, 0);
  }

  private findNode(prefix: string): TrieNode | null {
    let node = this.root;

    for (const char of prefix) {
      if (!node.children.has(char)) {
        return null;
      }
      node = node.children.get(char)!;
    }

    return node;
  }

  private dfs(node: TrieNode, currentWord: string, results: string[]): void {
    if (node.isEndOfWord) {
      results.push(currentWord);
    }

    for (const [char, childNode] of node.children) {
      this.dfs(childNode, currentWord + char, results);
    }
  }

  private deleteHelper(node: TrieNode, word: string, index: number): boolean {
    if (index === word.length) {
      if (!node.isEndOfWord) return false;
      node.isEndOfWord = false;
      return node.children.size === 0;
    }

    const char = word[index];
    const childNode = node.children.get(char);

    if (!childNode) return false;

    const shouldDeleteChild = this.deleteHelper(childNode, word, index + 1);

    if (shouldDeleteChild) {
      node.children.delete(char);
      return node.children.size === 0 && !node.isEndOfWord;
    }

    return false;
  }
}

// Autocomplete Implementation
class Autocomplete {
  private trie: Trie;

  constructor(words: string[]) {
    this.trie = new Trie();
    words.forEach((word) => this.trie.insert(word.toLowerCase()));
  }

  suggest(prefix: string, limit: number = 10): string[] {
    const suggestions = this.trie.findWordsWithPrefix(prefix.toLowerCase());
    return suggestions.slice(0, limit);
  }
}

// Usage
const autocomplete = new Autocomplete([
  "apple",
  "application",
  "apply",
  "banana",
  "band",
  "can",
]);
console.log(autocomplete.suggest("app")); // ['apple', 'application', 'apply']
```

### 3. Implement a Min/Max Heap

```typescript
class MinHeap<T> {
  private heap: T[];
  private compare: (a: T, b: T) => number;

  constructor(
    compare: (a: T, b: T) => number = (a, b) => Number(a) - Number(b),
  ) {
    this.heap = [];
    this.compare = compare;
  }

  /**
   * Insert element - O(log n)
   */
  insert(value: T): void {
    this.heap.push(value);
    this.heapifyUp(this.heap.length - 1);
  }

  /**
   * Extract minimum - O(log n)
   */
  extractMin(): T | undefined {
    if (this.heap.length === 0) return undefined;
    if (this.heap.length === 1) return this.heap.pop();

    const min = this.heap[0];
    this.heap[0] = this.heap.pop()!;
    this.heapifyDown(0);

    return min;
  }

  /**
   * Peek at minimum - O(1)
   */
  peek(): T | undefined {
    return this.heap[0];
  }

  get size(): number {
    return this.heap.length;
  }

  private heapifyUp(index: number): void {
    while (index > 0) {
      const parentIndex = Math.floor((index - 1) / 2);

      if (this.compare(this.heap[index], this.heap[parentIndex]) >= 0) {
        break;
      }

      this.swap(index, parentIndex);
      index = parentIndex;
    }
  }

  private heapifyDown(index: number): void {
    while (true) {
      let smallest = index;
      const leftChild = 2 * index + 1;
      const rightChild = 2 * index + 2;

      if (
        leftChild < this.heap.length &&
        this.compare(this.heap[leftChild], this.heap[smallest]) < 0
      ) {
        smallest = leftChild;
      }

      if (
        rightChild < this.heap.length &&
        this.compare(this.heap[rightChild], this.heap[smallest]) < 0
      ) {
        smallest = rightChild;
      }

      if (smallest === index) break;

      this.swap(index, smallest);
      index = smallest;
    }
  }

  private swap(i: number, j: number): void {
    [this.heap[i], this.heap[j]] = [this.heap[j], this.heap[i]];
  }
}

// Priority Queue Implementation
interface Task {
  id: string;
  priority: number;
  task: () => void;
}

class PriorityQueue {
  private heap: MinHeap<Task>;

  constructor() {
    this.heap = new MinHeap<Task>((a, b) => a.priority - b.priority);
  }

  enqueue(task: Task): void {
    this.heap.insert(task);
  }

  dequeue(): Task | undefined {
    return this.heap.extractMin();
  }

  peek(): Task | undefined {
    return this.heap.peek();
  }

  get size(): number {
    return this.heap.size;
  }
}

// Usage Example: Task Scheduler
const taskQueue = new PriorityQueue();

taskQueue.enqueue({
  id: "1",
  priority: 5,
  task: () => console.log("Low priority"),
});
taskQueue.enqueue({
  id: "2",
  priority: 1,
  task: () => console.log("High priority"),
});
taskQueue.enqueue({
  id: "3",
  priority: 3,
  task: () => console.log("Medium priority"),
});

while (taskQueue.size > 0) {
  const task = taskQueue.dequeue();
  task?.task();
}
// Output: High priority, Medium priority, Low priority
```

## Algorithm Problems

### 1. Debounce and Throttle

```typescript
/**
 * Debounce: Delay execution until after wait period
 * Use case: Search input, window resize
 */
function debounce<T extends (...args: any[]) => any>(
  func: T,
  wait: number,
): (...args: Parameters<T>) => void {
  let timeoutId: ReturnType<typeof setTimeout> | null = null;

  return function (this: any, ...args: Parameters<T>) {
    const context = this;

    if (timeoutId !== null) {
      clearTimeout(timeoutId);
    }

    timeoutId = setTimeout(() => {
      func.apply(context, args);
      timeoutId = null;
    }, wait);
  };
}

/**
 * Throttle: Execute at most once per wait period
 * Use case: Scroll events, mouse movement
 */
function throttle<T extends (...args: any[]) => any>(
  func: T,
  wait: number,
): (...args: Parameters<T>) => void {
  let inThrottle = false;
  let lastArgs: Parameters<T> | null = null;

  return function (this: any, ...args: Parameters<T>) {
    const context = this;

    if (!inThrottle) {
      func.apply(context, args);
      inThrottle = true;

      setTimeout(() => {
        inThrottle = false;
        if (lastArgs !== null) {
          func.apply(context, lastArgs);
          lastArgs = null;
        }
      }, wait);
    } else {
      lastArgs = args;
    }
  };
}

// Advanced: Debounce with leading/trailing options
interface DebounceOptions {
  leading?: boolean;
  trailing?: boolean;
  maxWait?: number;
}

function advancedDebounce<T extends (...args: any[]) => any>(
  func: T,
  wait: number,
  options: DebounceOptions = {},
): (...args: Parameters<T>) => void {
  const { leading = false, trailing = true, maxWait } = options;

  let timeoutId: ReturnType<typeof setTimeout> | null = null;
  let lastCallTime = 0;
  let lastInvokeTime = 0;

  return function (this: any, ...args: Parameters<T>) {
    const context = this;
    const now = Date.now();
    const timeSinceLastCall = now - lastCallTime;

    lastCallTime = now;

    const shouldInvoke = () => {
      if (leading && timeSinceLastCall >= wait) return true;
      if (maxWait && now - lastInvokeTime >= maxWait) return true;
      return false;
    };

    if (shouldInvoke()) {
      lastInvokeTime = now;
      func.apply(context, args);
    }

    if (timeoutId !== null) {
      clearTimeout(timeoutId);
    }

    if (trailing) {
      timeoutId = setTimeout(() => {
        lastInvokeTime = Date.now();
        func.apply(context, args);
        timeoutId = null;
      }, wait);
    }
  };
}

// Usage Examples
const searchAPI = debounce((query: string) => {
  console.log("Searching for:", query);
}, 500);

const handleScroll = throttle(() => {
  console.log("Scroll position:", window.scrollY);
}, 100);

window.addEventListener("scroll", handleScroll);
```

### 2. Deep Clone Object

```typescript
/**
 * Deep clone with circular reference handling
 */
function deepClone<T>(obj: T, hash = new WeakMap()): T {
  // Handle primitives and null
  if (obj === null || typeof obj !== "object") {
    return obj;
  }

  // Handle circular references
  if (hash.has(obj)) {
    return hash.get(obj);
  }

  // Handle Date
  if (obj instanceof Date) {
    return new Date(obj.getTime()) as any;
  }

  // Handle RegExp
  if (obj instanceof RegExp) {
    return new RegExp(obj.source, obj.flags) as any;
  }

  // Handle Map
  if (obj instanceof Map) {
    const clonedMap = new Map();
    hash.set(obj, clonedMap);
    obj.forEach((value, key) => {
      clonedMap.set(deepClone(key, hash), deepClone(value, hash));
    });
    return clonedMap as any;
  }

  // Handle Set
  if (obj instanceof Set) {
    const clonedSet = new Set();
    hash.set(obj, clonedSet);
    obj.forEach((value) => {
      clonedSet.add(deepClone(value, hash));
    });
    return clonedSet as any;
  }

  // Handle Array
  if (Array.isArray(obj)) {
    const clonedArr: any[] = [];
    hash.set(obj, clonedArr);
    obj.forEach((item, index) => {
      clonedArr[index] = deepClone(item, hash);
    });
    return clonedArr as any;
  }

  // Handle Object
  const clonedObj = Object.create(Object.getPrototypeOf(obj));
  hash.set(obj, clonedObj);

  // Clone own properties (including non-enumerable)
  Object.getOwnPropertyNames(obj).forEach((key) => {
    const descriptor = Object.getOwnPropertyDescriptor(obj, key);
    if (descriptor) {
      Object.defineProperty(clonedObj, key, {
        ...descriptor,
        value: deepClone((obj as any)[key], hash),
      });
    }
  });

  // Clone symbol properties
  Object.getOwnPropertySymbols(obj).forEach((sym) => {
    const descriptor = Object.getOwnPropertyDescriptor(obj, sym);
    if (descriptor) {
      Object.defineProperty(clonedObj, sym, {
        ...descriptor,
        value: deepClone((obj as any)[sym], hash),
      });
    }
  });

  return clonedObj;
}

// Test Cases
const testObject = {
  string: "hello",
  number: 42,
  bool: true,
  null: null,
  undefined: undefined,
  date: new Date(),
  regex: /test/gi,
  array: [1, 2, { nested: true }],
  map: new Map([["key", "value"]]),
  set: new Set([1, 2, 3]),
  nested: {
    deep: {
      value: "deeply nested",
    },
  },
};

// Add circular reference
testObject.nested.deep["circular"] = testObject;

const cloned = deepClone(testObject);
console.log(cloned.nested.deep["circular"] === cloned); // true (maintains circular reference)
```

### 3. Flatten Nested Array

```typescript
/**
 * Flatten array to specified depth
 */
function flatten<T>(arr: any[], depth: number = 1): T[] {
  if (depth === 0) return arr;

  return arr.reduce((acc, val) => {
    if (Array.isArray(val)) {
      acc.push(...flatten(val, depth - 1));
    } else {
      acc.push(val);
    }
    return acc;
  }, []);
}

/**
 * Flatten array completely (infinite depth)
 */
function flattenDeep<T>(arr: any[]): T[] {
  return arr.reduce((acc, val) => {
    if (Array.isArray(val)) {
      acc.push(...flattenDeep(val));
    } else {
      acc.push(val);
    }
    return acc;
  }, []);
}

// Alternative implementations
const flattenIterative = <T>(arr: any[]): T[] => {
  const stack = [...arr];
  const result: T[] = [];

  while (stack.length > 0) {
    const item = stack.pop();
    if (Array.isArray(item)) {
      stack.push(...item);
    } else {
      result.unshift(item);
    }
  }

  return result;
};

// Usage
const nested = [1, [2, [3, [4, 5]]]];
console.log(flatten(nested, 1)); // [1, 2, [3, [4, 5]]]
console.log(flatten(nested, 2)); // [1, 2, 3, [4, 5]]
console.log(flattenDeep(nested)); // [1, 2, 3, 4, 5]
```

### 4. Implement Promise.all and Promise.race

```typescript
/**
 * Custom Promise.all implementation
 */
function promiseAll<T>(promises: Promise<T>[]): Promise<T[]> {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      resolve([]);
      return;
    }

    const results: T[] = new Array(promises.length);
    let completedCount = 0;

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then((value) => {
          results[index] = value;
          completedCount++;

          if (completedCount === promises.length) {
            resolve(results);
          }
        })
        .catch(reject);
    });
  });
}

/**
 * Custom Promise.race implementation
 */
function promiseRace<T>(promises: Promise<T>[]): Promise<T> {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      return;
    }

    promises.forEach((promise) => {
      Promise.resolve(promise).then(resolve).catch(reject);
    });
  });
}

/**
 * Custom Promise.allSettled implementation
 */
type PromiseSettledResult<T> =
  | { status: "fulfilled"; value: T }
  | { status: "rejected"; reason: any };

function promiseAllSettled<T>(
  promises: Promise<T>[],
): Promise<PromiseSettledResult<T>[]> {
  return new Promise((resolve) => {
    if (promises.length === 0) {
      resolve([]);
      return;
    }

    const results: PromiseSettledResult<T>[] = new Array(promises.length);
    let completedCount = 0;

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then((value) => {
          results[index] = { status: "fulfilled", value };
        })
        .catch((reason) => {
          results[index] = { status: "rejected", reason };
        })
        .finally(() => {
          completedCount++;
          if (completedCount === promises.length) {
            resolve(results);
          }
        });
    });
  });
}

/**
 * Custom Promise.any implementation
 */
function promiseAny<T>(promises: Promise<T>[]): Promise<T> {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      reject(new AggregateError([], "All promises were rejected"));
      return;
    }

    const errors: any[] = [];
    let rejectedCount = 0;

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(resolve)
        .catch((error) => {
          errors[index] = error;
          rejectedCount++;

          if (rejectedCount === promises.length) {
            reject(new AggregateError(errors, "All promises were rejected"));
          }
        });
    });
  });
}

// Usage Examples
const p1 = Promise.resolve(1);
const p2 = new Promise((resolve) => setTimeout(() => resolve(2), 100));
const p3 = Promise.resolve(3);

promiseAll([p1, p2, p3]).then(console.log); // [1, 2, 3]
promiseRace([p1, p2, p3]).then(console.log); // 1
```

## DOM Manipulation Challenges

### 1. Implement Virtual Scrolling

```typescript
/**
 * Virtual scrolling for large lists
 * Only renders visible items
 */
class VirtualScroller {
  private container: HTMLElement;
  private itemHeight: number;
  private visibleCount: number;
  private totalCount: number;
  private renderItem: (index: number) => HTMLElement;
  private scrollTop: number = 0;
  private startIndex: number = 0;

  constructor(options: {
    container: HTMLElement;
    itemHeight: number;
    totalCount: number;
    renderItem: (index: number) => HTMLElement;
  }) {
    this.container = options.container;
    this.itemHeight = options.itemHeight;
    this.totalCount = options.totalCount;
    this.renderItem = options.renderItem;

    // Calculate visible items
    this.visibleCount =
      Math.ceil(this.container.clientHeight / this.itemHeight) + 1;

    this.initialize();
  }

  private initialize(): void {
    // Create viewport
    const viewport = document.createElement("div");
    viewport.style.height = `${this.totalCount * this.itemHeight}px`;
    viewport.style.position = "relative";

    // Create content container
    const content = document.createElement("div");
    content.style.position = "absolute";
    content.style.top = "0";
    content.style.left = "0";
    content.style.right = "0";

    viewport.appendChild(content);
    this.container.appendChild(viewport);

    // Setup scroll listener
    this.container.addEventListener("scroll", () => {
      this.scrollTop = this.container.scrollTop;
      this.render();
    });

    this.render();
  }

  private render(): void {
    this.startIndex = Math.floor(this.scrollTop / this.itemHeight);
    const endIndex = Math.min(
      this.startIndex + this.visibleCount,
      this.totalCount,
    );

    const content = this.container.querySelector("div > div") as HTMLElement;
    content.innerHTML = "";
    content.style.transform = `translateY(${this.startIndex * this.itemHeight}px)`;

    for (let i = this.startIndex; i < endIndex; i++) {
      const item = this.renderItem(i);
      item.style.height = `${this.itemHeight}px`;
      content.appendChild(item);
    }
  }

  update(totalCount: number): void {
    this.totalCount = totalCount;
    const viewport = this.container.querySelector("div") as HTMLElement;
    viewport.style.height = `${this.totalCount * this.itemHeight}px`;
    this.render();
  }
}

// Usage
const container = document.getElementById("scroll-container")!;
const scroller = new VirtualScroller({
  container,
  itemHeight: 50,
  totalCount: 10000,
  renderItem: (index) => {
    const div = document.createElement("div");
    div.textContent = `Item ${index}`;
    div.className = "list-item";
    return div;
  },
});
```

### 2. Implement Drag and Drop

```typescript
/**
 * Drag and Drop Implementation
 */
class DragDropManager {
  private draggedElement: HTMLElement | null = null;
  private placeholder: HTMLElement;
  private sourceContainer: HTMLElement | null = null;

  constructor(private containers: HTMLElement[]) {
    this.placeholder = this.createPlaceholder();
    this.initialize();
  }

  private createPlaceholder(): HTMLElement {
    const placeholder = document.createElement("div");
    placeholder.className = "drag-placeholder";
    placeholder.style.height = "50px";
    placeholder.style.border = "2px dashed #ccc";
    placeholder.style.margin = "5px 0";
    return placeholder;
  }

  private initialize(): void {
    this.containers.forEach((container) => {
      const items = container.querySelectorAll('[draggable="true"]');

      items.forEach((item) => {
        item.addEventListener("dragstart", this.handleDragStart.bind(this));
        item.addEventListener("dragend", this.handleDragEnd.bind(this));
      });

      container.addEventListener("dragover", this.handleDragOver.bind(this));
      container.addEventListener("drop", this.handleDrop.bind(this));
      container.addEventListener("dragleave", this.handleDragLeave.bind(this));
    });
  }

  private handleDragStart(e: DragEvent): void {
    this.draggedElement = e.target as HTMLElement;
    this.sourceContainer = this.draggedElement.parentElement;

    this.draggedElement.style.opacity = "0.5";
    e.dataTransfer!.effectAllowed = "move";
    e.dataTransfer!.setData("text/html", this.draggedElement.innerHTML);
  }

  private handleDragEnd(e: DragEvent): void {
    if (this.draggedElement) {
      this.draggedElement.style.opacity = "1";
    }

    // Remove placeholder
    if (this.placeholder.parentNode) {
      this.placeholder.parentNode.removeChild(this.placeholder);
    }

    this.draggedElement = null;
    this.sourceContainer = null;
  }

  private handleDragOver(e: DragEvent): void {
    e.preventDefault();
    e.dataTransfer!.dropEffect = "move";

    const container = e.currentTarget as HTMLElement;
    const afterElement = this.getDragAfterElement(container, e.clientY);

    if (afterElement) {
      container.insertBefore(this.placeholder, afterElement);
    } else {
      container.appendChild(this.placeholder);
    }
  }

  private handleDrop(e: DragEvent): void {
    e.preventDefault();

    if (!this.draggedElement) return;

    const container = e.currentTarget as HTMLElement;

    // Insert dragged element at placeholder position
    if (this.placeholder.parentNode === container) {
      container.insertBefore(this.draggedElement, this.placeholder);
    }

    // Remove placeholder
    if (this.placeholder.parentNode) {
      this.placeholder.parentNode.removeChild(this.placeholder);
    }

    // Dispatch custom event
    const event = new CustomEvent("itemMoved", {
      detail: {
        element: this.draggedElement,
        from: this.sourceContainer,
        to: container,
      },
    });
    container.dispatchEvent(event);
  }

  private handleDragLeave(e: DragEvent): void {
    const container = e.currentTarget as HTMLElement;
    if (e.target === container && this.placeholder.parentNode === container) {
      container.removeChild(this.placeholder);
    }
  }

  private getDragAfterElement(
    container: HTMLElement,
    y: number,
  ): HTMLElement | null {
    const draggableElements = [
      ...container.querySelectorAll('[draggable="true"]:not(.dragging)'),
    ] as HTMLElement[];

    return draggableElements.reduce<{
      offset: number;
      element: HTMLElement | null;
    }>(
      (closest, child) => {
        const box = child.getBoundingClientRect();
        const offset = y - box.top - box.height / 2;

        if (offset < 0 && offset > closest.offset) {
          return { offset, element: child };
        } else {
          return closest;
        }
      },
      { offset: Number.NEGATIVE_INFINITY, element: null },
    ).element;
  }
}

// Usage
const lists = document.querySelectorAll(
  ".sortable-list",
) as NodeListOf<HTMLElement>;
const dragDrop = new DragDropManager(Array.from(lists));

// Listen for move events
lists.forEach((list) => {
  list.addEventListener("itemMoved", ((e: CustomEvent) => {
    console.log("Item moved:", e.detail);
  }) as EventListener);
});
```

### 3. Implement Infinite Scroll

```typescript
/**
 * Infinite Scroll Implementation
 */
class InfiniteScroll {
  private container: HTMLElement;
  private threshold: number;
  private loading: boolean = false;
  private hasMore: boolean = true;
  private page: number = 1;
  private observer: IntersectionObserver | null = null;
  private sentinel: HTMLElement;

  constructor(options: {
    container: HTMLElement;
    threshold?: number;
    loadMore: (page: number) => Promise<{ items: any[]; hasMore: boolean }>;
    renderItem: (item: any) => HTMLElement;
  }) {
    this.container = options.container;
    this.threshold = options.threshold || 200;
    this.sentinel = this.createSentinel();

    this.initialize(options.loadMore, options.renderItem);
  }

  private createSentinel(): HTMLElement {
    const sentinel = document.createElement("div");
    sentinel.className = "infinite-scroll-sentinel";
    sentinel.style.height = "1px";
    return sentinel;
  }

  private initialize(
    loadMore: (page: number) => Promise<{ items: any[]; hasMore: boolean }>,
    renderItem: (item: any) => HTMLElement,
  ): void {
    // Append sentinel
    this.container.appendChild(this.sentinel);

    // Setup Intersection Observer
    this.observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting && !this.loading && this.hasMore) {
            this.loadMoreItems(loadMore, renderItem);
          }
        });
      },
      {
        root: null,
        rootMargin: `${this.threshold}px`,
        threshold: 0,
      },
    );

    this.observer.observe(this.sentinel);

    // Initial load
    this.loadMoreItems(loadMore, renderItem);
  }

  private async loadMoreItems(
    loadMore: (page: number) => Promise<{ items: any[]; hasMore: boolean }>,
    renderItem: (item: any) => HTMLElement,
  ): Promise<void> {
    if (this.loading || !this.hasMore) return;

    this.loading = true;
    this.showLoader();

    try {
      const result = await loadMore(this.page);

      result.items.forEach((item) => {
        const element = renderItem(item);
        this.container.insertBefore(element, this.sentinel);
      });

      this.hasMore = result.hasMore;
      this.page++;
    } catch (error) {
      console.error("Failed to load more items:", error);
    } finally {
      this.loading = false;
      this.hideLoader();
    }
  }

  private showLoader(): void {
    const loader = document.createElement("div");
    loader.className = "infinite-scroll-loader";
    loader.textContent = "Loading...";
    this.container.insertBefore(loader, this.sentinel);
  }

  private hideLoader(): void {
    const loader = this.container.querySelector(".infinite-scroll-loader");
    if (loader) {
      loader.remove();
    }
  }

  destroy(): void {
    if (this.observer) {
      this.observer.disconnect();
    }
    if (this.sentinel.parentNode) {
      this.sentinel.parentNode.removeChild(this.sentinel);
    }
  }
}

// Usage
const infiniteScroll = new InfiniteScroll({
  container: document.getElementById("posts-container")!,
  threshold: 300,
  loadMore: async (page) => {
    const response = await fetch(`/api/posts?page=${page}`);
    const data = await response.json();
    return {
      items: data.posts,
      hasMore: data.hasMore,
    };
  },
  renderItem: (post) => {
    const article = document.createElement("article");
    article.innerHTML = `
      <h2>${post.title}</h2>
      <p>${post.excerpt}</p>
    `;
    return article;
  },
});
```

## Async Programming Challenges

### 1. Retry with Exponential Backoff

```typescript
/**
 * Retry failed operations with exponential backoff
 */
interface RetryOptions {
  maxRetries: number;
  initialDelay: number;
  maxDelay: number;
  backoffFactor: number;
  onRetry?: (error: Error, attempt: number) => void;
}

async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: RetryOptions,
): Promise<T> {
  const { maxRetries, initialDelay, maxDelay, backoffFactor, onRetry } =
    options;

  let lastError: Error;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      if (attempt === maxRetries) {
        throw new Error(
          `Failed after ${maxRetries + 1} attempts: ${lastError.message}`,
        );
      }

      const delay = Math.min(
        initialDelay * Math.pow(backoffFactor, attempt),
        maxDelay,
      );

      if (onRetry) {
        onRetry(lastError, attempt + 1);
      }

      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError!;
}

// Usage Example
async function fetchWithRetry(url: string) {
  return retryWithBackoff(
    () =>
      fetch(url).then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      }),
    {
      maxRetries: 3,
      initialDelay: 1000,
      maxDelay: 10000,
      backoffFactor: 2,
      onRetry: (error, attempt) => {
        console.log(`Retry attempt ${attempt}: ${error.message}`);
      },
    },
  );
}
```

### 2. Rate Limiter

```typescript
/**
 * Rate limiter for API calls
 */
class RateLimiter {
  private queue: Array<() => void> = [];
  private running: number = 0;

  constructor(
    private maxConcurrent: number,
    private minTime: number = 0,
  ) {}

  async schedule<T>(fn: () => Promise<T>): Promise<T> {
    while (this.running >= this.maxConcurrent) {
      await new Promise((resolve) => this.queue.push(resolve as () => void));
    }

    this.running++;

    try {
      const start = Date.now();
      const result = await fn();
      const elapsed = Date.now() - start;

      if (this.minTime > 0 && elapsed < this.minTime) {
        await new Promise((resolve) =>
          setTimeout(resolve, this.minTime - elapsed),
        );
      }

      return result;
    } finally {
      this.running--;
      const resolve = this.queue.shift();
      if (resolve) resolve();
    }
  }
}

// Token Bucket Rate Limiter
class TokenBucket {
  private tokens: number;
  private lastRefill: number;

  constructor(
    private capacity: number,
    private refillRate: number,
  ) {
    this.tokens = capacity;
    this.lastRefill = Date.now();
  }

  async acquire(tokens: number = 1): Promise<void> {
    await this.refill();

    while (this.tokens < tokens) {
      const waitTime = ((tokens - this.tokens) / this.refillRate) * 1000;
      await new Promise((resolve) => setTimeout(resolve, waitTime));
      await this.refill();
    }

    this.tokens -= tokens;
  }

  private async refill(): Promise<void> {
    const now = Date.now();
    const timePassed = (now - this.lastRefill) / 1000;
    const tokensToAdd = timePassed * this.refillRate;

    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }
}

// Usage
const limiter = new RateLimiter(5, 1000); // 5 concurrent, min 1s between calls

const promises = Array.from({ length: 20 }, (_, i) =>
  limiter.schedule(() => fetch(`/api/data/${i}`).then((r) => r.json())),
);

const results = await Promise.all(promises);
```

## React-Specific Challenges

### 1. Implement useDebounce Hook

```typescript
import { useState, useEffect } from 'react';

/**
 * useDebounce Hook
 */
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
}

/**
 * useThrottle Hook
 */
function useThrottle<T>(value: T, interval: number): T {
  const [throttledValue, setThrottledValue] = useState<T>(value);
  const [lastUpdated, setLastUpdated] = useState<number>(Date.now());

  useEffect(() => {
    const now = Date.now();
    const timeSinceLastUpdate = now - lastUpdated;

    if (timeSinceLastUpdate >= interval) {
      setThrottledValue(value);
      setLastUpdated(now);
    } else {
      const timeoutId = setTimeout(() => {
        setThrottledValue(value);
        setLastUpdated(Date.now());
      }, interval - timeSinceLastUpdate);

      return () => clearTimeout(timeoutId);
    }
  }, [value, interval, lastUpdated]);

  return throttledValue;
}

// Usage in Component
function SearchComponent() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 500);

  useEffect(() => {
    if (debouncedSearchTerm) {
      // Perform search
      console.log('Searching for:', debouncedSearchTerm);
    }
  }, [debouncedSearchTerm]);

  return (
    <input
      type="text"
      value={searchTerm}
      onChange={(e) => setSearchTerm(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

### 2. Implement Custom useState with Undo/Redo

```typescript
import { useState, useCallback, useRef } from 'react';

interface UseUndoableStateReturn<T> {
  state: T;
  setState: (newState: T | ((prev: T) => T)) => void;
  undo: () => void;
  redo: () => void;
  canUndo: boolean;
  canRedo: boolean;
  reset: () => void;
}

function useUndoableState<T>(
  initialState: T
): UseUndoableStateReturn<T> {
  const [state, setInternalState] = useState<T>(initialState);
  const history = useRef<T[]>([initialState]);
  const currentIndex = useRef<number>(0);

  const setState = useCallback((newState: T | ((prev: T) => T)) => {
    setInternalState(currentState => {
      const nextState =
        typeof newState === 'function'
          ? (newState as (prev: T) => T)(currentState)
          : newState;

      // Remove future states when making a new change
      history.current = history.current.slice(0, currentIndex.current + 1);

      // Add new state
      history.current.push(nextState);
      currentIndex.current++;

      return nextState;
    });
  }, []);

  const undo = useCallback(() => {
    if (currentIndex.current > 0) {
      currentIndex.current--;
      setInternalState(history.current[currentIndex.current]);
    }
  }, []);

  const redo = useCallback(() => {
    if (currentIndex.current < history.current.length - 1) {
      currentIndex.current++;
      setInternalState(history.current[currentIndex.current]);
    }
  }, []);

  const reset = useCallback(() => {
    history.current = [initialState];
    currentIndex.current = 0;
    setInternalState(initialState);
  }, [initialState]);

  return {
    state,
    setState,
    undo,
    redo,
    canUndo: currentIndex.current > 0,
    canRedo: currentIndex.current < history.current.length - 1,
    reset
  };
}

// Usage
function DrawingApp() {
  const {
    state: drawings,
    setState: setDrawings,
    undo,
    redo,
    canUndo,
    canRedo
  } = useUndoableState<string[]>([]);

  const addDrawing = (drawing: string) => {
    setDrawings(prev => [...prev, drawing]);
  };

  return (
    <div>
      <button onClick={undo} disabled={!canUndo}>Undo</button>
      <button onClick={redo} disabled={!canRedo}>Redo</button>
      <button onClick={() => addDrawing('Line')}>Add Line</button>
      {/* Render drawings */}
    </div>
  );
}
```

## Performance Optimization Problems

### 1. Implement Memoization

```typescript
/**
 * Generic memoization function
 */
function memoize<T extends (...args: any[]) => any>(
  fn: T,
  getKey?: (...args: Parameters<T>) => string,
): T {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: Parameters<T>): ReturnType<T> => {
    const key = getKey ? getKey(...args) : JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key)!;
    }

    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// Memoization with expiry
function memoizeWithExpiry<T extends (...args: any[]) => any>(
  fn: T,
  ttl: number,
): T {
  const cache = new Map<string, { value: ReturnType<T>; timestamp: number }>();

  return ((...args: Parameters<T>): ReturnType<T> => {
    const key = JSON.stringify(args);
    const cached = cache.get(key);

    if (cached && Date.now() - cached.timestamp < ttl) {
      return cached.value;
    }

    const result = fn(...args);
    cache.set(key, { value: result, timestamp: Date.now() });
    return result;
  }) as T;
}

// Fibonacci with memoization
const fibonacci = memoize((n: number): number => {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
});

console.log(fibonacci(40)); // Fast with memoization
```

### 2. Batch DOM Updates

```typescript
/**
 * Batch DOM updates to prevent layout thrashing
 */
class DOMBatcher {
  private reads: Array<() => void> = [];
  private writes: Array<() => void> = [];
  private scheduled: boolean = false;

  read(fn: () => void): void {
    this.reads.push(fn);
    this.schedule();
  }

  write(fn: () => void): void {
    this.writes.push(fn);
    this.schedule();
  }

  private schedule(): void {
    if (this.scheduled) return;

    this.scheduled = true;
    requestAnimationFrame(() => {
      this.flush();
    });
  }

  private flush(): void {
    // Execute all reads first
    const reads = this.reads.splice(0);
    reads.forEach((fn) => fn());

    // Then execute all writes
    const writes = this.writes.splice(0);
    writes.forEach((fn) => fn());

    this.scheduled = false;

    // If new operations were queued, schedule again
    if (this.reads.length > 0 || this.writes.length > 0) {
      this.schedule();
    }
  }
}

// Usage
const batcher = new DOMBatcher();
const elements = document.querySelectorAll(".box");

elements.forEach((el) => {
  batcher.read(() => {
    const height = el.clientHeight;
    batcher.write(() => {
      el.style.height = `${height * 2}px`;
    });
  });
});
```

## TypeScript Advanced Patterns

### 1. Type-Safe Event Emitter

```typescript
/**
 * Type-safe event emitter
 */
type EventMap = Record<string, any>;

type EventKey<T extends EventMap> = string & keyof T;
type EventReceiver<T> = (params: T) => void;

class TypedEventEmitter<T extends EventMap> {
  private listeners: {
    [K in keyof T]?: Array<EventReceiver<T[K]>>;
  } = {};

  on<K extends EventKey<T>>(event: K, fn: EventReceiver<T[K]>): () => void {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(fn);

    // Return unsubscribe function
    return () => this.off(event, fn);
  }

  off<K extends EventKey<T>>(event: K, fn: EventReceiver<T[K]>): void {
    const listeners = this.listeners[event];
    if (!listeners) return;

    const index = listeners.indexOf(fn);
    if (index > -1) {
      listeners.splice(index, 1);
    }
  }

  emit<K extends EventKey<T>>(event: K, params: T[K]): void {
    const listeners = this.listeners[event];
    if (!listeners) return;

    listeners.forEach((fn) => fn(params));
  }

  once<K extends EventKey<T>>(event: K, fn: EventReceiver<T[K]>): void {
    const onceFn: EventReceiver<T[K]> = (params) => {
      fn(params);
      this.off(event, onceFn);
    };
    this.on(event, onceFn);
  }
}

// Usage with type safety
interface AppEvents {
  userLoggedIn: { userId: string; timestamp: Date };
  userLoggedOut: { userId: string };
  dataUpdated: { collection: string; id: string };
}

const emitter = new TypedEventEmitter<AppEvents>();

emitter.on("userLoggedIn", (data) => {
  // data is typed as { userId: string; timestamp: Date }
  console.log(`User ${data.userId} logged in at ${data.timestamp}`);
});

emitter.emit("userLoggedIn", {
  userId: "123",
  timestamp: new Date(),
});
```

### 2. Builder Pattern with Type Safety

```typescript
/**
 * Type-safe builder pattern
 */
type OptionalKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? K : never;
}[keyof T];

type RequiredKeys<T> = Exclude<keyof T, OptionalKeys<T>>;

type Builder<T, Built extends Partial<T> = {}> = {
  [K in keyof T]: (value: T[K]) => Builder<T, Built & { [P in K]: T[K] }>;
} & (RequiredKeys<T> extends keyof Built ? { build(): T } : {});

function createBuilder<T>(): Builder<T> {
  const built: Partial<T> = {};

  const builder = new Proxy({} as Builder<T>, {
    get(_, prop: string) {
      if (prop === "build") {
        return () => built as T;
      }

      return (value: any) => {
        built[prop as keyof T] = value;
        return builder;
      };
    },
  });

  return builder;
}

// Usage
interface User {
  id: string;
  name: string;
  email: string;
  age?: number;
  role?: "admin" | "user";
}

const userBuilder = createBuilder<User>();

const user = userBuilder
  .id("123")
  .name("John Doe")
  .email("john@example.com")
  .age(30)
  .build(); // TypeScript ensures all required fields are set

// This would be a TypeScript error:
// const invalidUser = userBuilder.id('123').build(); // Missing required fields
```

## Testing Challenges

### 1. Implement Test Runner

```typescript
/**
 * Simple test runner implementation
 */
interface TestCase {
  name: string;
  fn: () => void | Promise<void>;
}

interface TestSuite {
  name: string;
  tests: TestCase[];
  beforeEach?: () => void | Promise<void>;
  afterEach?: () => void | Promise<void>;
}

class TestRunner {
  private suites: TestSuite[] = [];

  describe(name: string, fn: () => void): void {
    const suite: TestSuite = { name, tests: [] };
    this.suites.push(suite);

    // Set current suite context
    const previousDescribe = (globalThis as any).__currentSuite;
    (globalThis as any).__currentSuite = suite;

    fn();

    (globalThis as any).__currentSuite = previousDescribe;
  }

  it(name: string, fn: () => void | Promise<void>): void {
    const currentSuite = (globalThis as any).__currentSuite as TestSuite;
    if (!currentSuite) {
      throw new Error("it() must be called within describe()");
    }

    currentSuite.tests.push({ name, fn });
  }

  beforeEach(fn: () => void | Promise<void>): void {
    const currentSuite = (globalThis as any).__currentSuite as TestSuite;
    if (!currentSuite) {
      throw new Error("beforeEach() must be called within describe()");
    }

    currentSuite.beforeEach = fn;
  }

  afterEach(fn: () => void | Promise<void>): void {
    const currentSuite = (globalThis as any).__currentSuite as TestSuite;
    if (!currentSuite) {
      throw new Error("afterEach() must be called within describe()");
    }

    currentSuite.afterEach = fn;
  }

  async run(): Promise<void> {
    let totalTests = 0;
    let passedTests = 0;
    let failedTests = 0;

    for (const suite of this.suites) {
      console.log(`\n${suite.name}`);

      for (const test of suite.tests) {
        totalTests++;

        try {
          if (suite.beforeEach) {
            await suite.beforeEach();
          }

          await test.fn();

          if (suite.afterEach) {
            await suite.afterEach();
          }

          passedTests++;
          console.log(`  ✓ ${test.name}`);
        } catch (error) {
          failedTests++;
          console.log(`  ✗ ${test.name}`);
          console.error(`    ${error}`);
        }
      }
    }

    console.log(
      `\nTotal: ${totalTests}, Passed: ${passedTests}, Failed: ${failedTests}`,
    );
  }
}

// Assertion library
class Expect<T> {
  constructor(private actual: T) {}

  toBe(expected: T): void {
    if (this.actual !== expected) {
      throw new Error(
        `Expected ${JSON.stringify(expected)}, got ${JSON.stringify(this.actual)}`,
      );
    }
  }

  toEqual(expected: T): void {
    if (JSON.stringify(this.actual) !== JSON.stringify(expected)) {
      throw new Error(
        `Expected ${JSON.stringify(expected)}, got ${JSON.stringify(this.actual)}`,
      );
    }
  }

  toBeTruthy(): void {
    if (!this.actual) {
      throw new Error(`Expected truthy value, got ${this.actual}`);
    }
  }

  toBeFalsy(): void {
    if (this.actual) {
      throw new Error(`Expected falsy value, got ${this.actual}`);
    }
  }
}

function expect<T>(actual: T): Expect<T> {
  return new Expect(actual);
}

// Usage
const runner = new TestRunner();

runner.describe("Calculator", () => {
  let calculator: { add: (a: number, b: number) => number };

  runner.beforeEach(() => {
    calculator = { add: (a, b) => a + b };
  });

  runner.it("should add two numbers", () => {
    const result = calculator.add(2, 3);
    expect(result).toBe(5);
  });

  runner.it("should handle negative numbers", () => {
    const result = calculator.add(-2, 3);
    expect(result).toBe(1);
  });
});

runner.run();
```

## Key Takeaways

1. **Problem-Solving Approach**: Always clarify requirements, discuss time/space complexity, and validate edge cases before implementing

2. **Data Structures**: Master fundamental structures (LRU Cache, Trie, Heap) as they're commonly asked in interviews and useful in real applications

3. **Algorithm Patterns**: Recognize common patterns (sliding window, two pointers, dynamic programming) to solve problems efficiently

4. **TypeScript Mastery**: Leverage TypeScript's type system for safer, more maintainable code - use generics, utility types, and conditional types

5. **Async Programming**: Understand Promises, async/await, and implement patterns like retry logic, rate limiting, and concurrent request management

6. **DOM Performance**: Minimize reflows/repaints by batching DOM operations, implementing virtual scrolling, and using IntersectionObserver

7. **React Patterns**: Create reusable custom hooks, implement advanced patterns like undo/redo, and understand render optimization

8. **Testing Skills**: Write testable code, understand testing patterns, and implement testing utilities from scratch

9. **Code Quality**: Write clean, documented code with proper error handling, edge case coverage, and meaningful variable names

10. **Communication**: Explain your thought process clearly, discuss trade-offs, and ask clarifying questions - technical communication is as important as coding skills

---

**Practice Resources**:

- LeetCode: Algorithm and data structure problems
- Frontend Mentor: Real-world frontend challenges
- CodeWars: Programming puzzles
- HackerRank: Technical assessments
- Daily coding challenges: Practice consistently

**Interview Tips**:

- Think out loud and explain your reasoning
- Start with brute force, then optimize
- Test your code with examples
- Discuss trade-offs and alternatives
- Ask about performance requirements
