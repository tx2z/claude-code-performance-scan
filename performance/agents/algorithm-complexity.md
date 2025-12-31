---
description: "[Internal] Algorithm complexity agent - use /performance-scan instead"
disable-model-invocation: true
allowed-tools: Read, Glob, Grep
---

# Algorithm Complexity Agent (PERF01)

Scan for performance issues related to:
- **Time Complexity** - O(n^2) or worse algorithms
- **Unnecessary Iterations** - Redundant loops and traversals
- **Missing Memoization** - Repeated expensive calculations
- **Inefficient Data Structures** - Wrong data structure for the use case
- **Sorting/Searching Inefficiencies** - Suboptimal algorithms
- **Missing Early Returns** - Unnecessary continued processing

---

## 1. Nested Loop Detection (O(n^2) or worse)

### 1.1 Direct Nested Loops

**Search:**
```
Grep: for\s*\(.*\)\s*\{[^}]*for\s*\(|\.forEach\([^)]*\)[^}]*\.forEach\(|\.map\([^)]*\)[^}]*\.map\(
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx, **/*.py, **/*.go, **/*.java, **/*.php, **/*.cs
```

**Check for:**
- [ ] Nested loops over the same or related data sets
- [ ] Inner loop that could be replaced with Set/Map lookup
- [ ] Quadratic comparisons that could be optimized

**Slow Pattern (JavaScript/TypeScript):**
```typescript
// BAD: O(n^2) - nested array includes check
function findCommon(arr1: number[], arr2: number[]): number[] {
  const result = [];
  for (const item of arr1) {
    if (arr2.includes(item)) {  // O(n) inside O(n) loop
      result.push(item);
    }
  }
  return result;
}
```

**Optimized Pattern:**
```typescript
// GOOD: O(n) - using Set for O(1) lookup
function findCommon(arr1: number[], arr2: number[]): number[] {
  const set2 = new Set(arr2);  // O(n) once
  return arr1.filter(item => set2.has(item));  // O(n) with O(1) lookups
}
```

**Slow Pattern (Python):**
```python
# BAD: O(n^2) - nested list check
def find_duplicates(items):
    duplicates = []
    for i, item in enumerate(items):
        if item in items[i+1:]:  # O(n) inside O(n) loop
            duplicates.append(item)
    return duplicates
```

**Optimized Pattern:**
```python
# GOOD: O(n) - using Counter
from collections import Counter
def find_duplicates(items):
    counts = Counter(items)
    return [item for item, count in counts.items() if count > 1]
```

### 1.2 Array Methods in Loops

**Search:**
```
Grep: \.includes\(|\.indexOf\(|\.find\(|\.findIndex\(|\.some\(|\.every\(
```

**Check for array methods inside:**
- [ ] for/for-of/forEach loops
- [ ] map/filter/reduce callbacks
- [ ] while loops

**Slow Pattern:**
```typescript
// BAD: O(n^2) - includes inside filter
const uniqueItems = items.filter((item, index) =>
  items.indexOf(item) === index
);
```

**Optimized Pattern:**
```typescript
// GOOD: O(n) - using Set
const uniqueItems = [...new Set(items)];
```

### 1.3 Multiple Array Traversals

**Search:**
```
Grep: \.filter\(.*\)\.map\(|\.map\(.*\)\.filter\(|\.filter\(.*\)\.filter\(
```

**Check for:**
- [ ] Chained array methods that could be combined
- [ ] Multiple passes over the same data

**Slow Pattern:**
```typescript
// BAD: Three traversals
const result = items
  .filter(x => x.active)
  .filter(x => x.age > 18)
  .map(x => x.name);
```

**Optimized Pattern:**
```typescript
// GOOD: Single traversal with reduce
const result = items.reduce((acc, x) => {
  if (x.active && x.age > 18) {
    acc.push(x.name);
  }
  return acc;
}, []);

// OR: Combine filters
const result = items
  .filter(x => x.active && x.age > 18)
  .map(x => x.name);
```

---

## 2. Missing Memoization

### 2.1 Recursive Functions Without Caching

**Search:**
```
Grep: function\s+\w+\([^)]*\)[^{]*\{[^}]*\1\s*\(|const\s+(\w+)\s*=.*=>[^}]*\1\s*\(
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Fibonacci-like recursion without memoization
- [ ] Tree/graph traversals with repeated subproblems
- [ ] Dynamic programming problems without caching

**Slow Pattern:**
```typescript
// BAD: O(2^n) - no memoization
function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

**Optimized Pattern:**
```typescript
// GOOD: O(n) - with memoization
function fibonacci(n: number, memo: Map<number, number> = new Map()): number {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n)!;

  const result = fibonacci(n - 1, memo) + fibonacci(n - 2, memo);
  memo.set(n, result);
  return result;
}

// OR: Using a memoization decorator/HOF
const memoizedFibonacci = memoize((n: number): number => {
  if (n <= 1) return n;
  return memoizedFibonacci(n - 1) + memoizedFibonacci(n - 2);
});
```

### 2.2 Expensive Computations in Render/Hot Paths

**Search (React):**
```
Grep: useMemo|useCallback|React\.memo
Glob: **/*.tsx, **/*.jsx
```

**Check for missing memoization:**
- [ ] Expensive calculations in render functions
- [ ] Object/array creation in render
- [ ] Functions passed as props without useCallback

**Slow Pattern (React):**
```tsx
// BAD: Recalculates on every render
function ExpensiveComponent({ items }) {
  const sortedItems = items.sort((a, b) => a.value - b.value);  // Mutates and recalcs
  const total = items.reduce((sum, item) => sum + item.value, 0);

  return <List items={sortedItems} total={total} />;
}
```

**Optimized Pattern:**
```tsx
// GOOD: Memoized calculations
function ExpensiveComponent({ items }) {
  const sortedItems = useMemo(
    () => [...items].sort((a, b) => a.value - b.value),
    [items]
  );

  const total = useMemo(
    () => items.reduce((sum, item) => sum + item.value, 0),
    [items]
  );

  return <List items={sortedItems} total={total} />;
}
```

### 2.3 Vue Computed Properties

**Search (Vue):**
```
Grep: computed\(|computed:|@Computed
Glob: **/*.vue, **/*.ts
```

**Check for:**
- [ ] Methods that should be computed properties
- [ ] Template expressions with calculations

---

## 3. Inefficient Data Structure Usage

### 3.1 Array for Frequent Lookups

**Search:**
```
Grep: \.find\(.*===|\.find\(.*==|\.includes\(|\.indexOf\(|\.some\(.*===
```

**Check if array is used for:**
- [ ] Frequent existence checks - should use Set
- [ ] Key-value lookups - should use Map/Object
- [ ] Deduplication - should use Set

**Slow Pattern:**
```typescript
// BAD: O(n) lookup every time
const userIds = [1, 2, 3, 4, 5];
function isUserAllowed(id: number): boolean {
  return userIds.includes(id);  // O(n)
}
```

**Optimized Pattern:**
```typescript
// GOOD: O(1) lookup
const userIds = new Set([1, 2, 3, 4, 5]);
function isUserAllowed(id: number): boolean {
  return userIds.has(id);  // O(1)
}
```

### 3.2 Object vs Map for Dynamic Keys

**Search:**
```
Grep: \[\w+\]\s*=|\[.*\]\s*=|delete\s+\w+\[
```

**Check for:**
- [ ] Frequent additions/deletions - Map is faster
- [ ] Non-string keys needed - Map required
- [ ] Iteration order matters - Map preserves insertion order

**Slow Pattern:**
```typescript
// BAD: Object with frequent dynamic operations
const cache: { [key: string]: Data } = {};

function addToCache(key: string, data: Data) {
  cache[key] = data;
}

function removeFromCache(key: string) {
  delete cache[key];  // Slow delete operation
}
```

**Optimized Pattern:**
```typescript
// GOOD: Map for dynamic key-value storage
const cache = new Map<string, Data>();

function addToCache(key: string, data: Data) {
  cache.set(key, data);
}

function removeFromCache(key: string) {
  cache.delete(key);  // O(1) delete
}
```

### 3.3 Array as Queue/Stack

**Search:**
```
Grep: \.shift\(\)|\.unshift\(
```

**Check for:**
- [ ] shift() in loops - O(n) per operation
- [ ] unshift() with large arrays

**Slow Pattern:**
```typescript
// BAD: O(n^2) - shift is O(n)
const queue: number[] = [];
while (queue.length > 0) {
  const item = queue.shift();  // O(n) - reindexes entire array
  process(item);
}
```

**Optimized Pattern:**
```typescript
// GOOD: Use index pointer or dedicated queue
class Queue<T> {
  private items: T[] = [];
  private head = 0;

  enqueue(item: T) { this.items.push(item); }
  dequeue(): T | undefined {
    if (this.head >= this.items.length) return undefined;
    return this.items[this.head++];
  }
}

// OR: Process in reverse
while (queue.length > 0) {
  const item = queue.pop();  // O(1)
  process(item);
}
```

---

## 4. Sorting Inefficiencies

### 4.1 Repeated Sorting

**Search:**
```
Grep: \.sort\(
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Sorting in loops
- [ ] Sorting already sorted data
- [ ] Sorting before each access

**Slow Pattern:**
```typescript
// BAD: Sorting repeatedly
function getTopN(items: Item[], n: number): Item[] {
  return items.sort((a, b) => b.score - a.score).slice(0, n);
}
// Called multiple times with same items - sorts every time
```

**Optimized Pattern:**
```typescript
// GOOD: Sort once, cache result
class ItemRanking {
  private sorted: Item[] | null = null;

  constructor(private items: Item[]) {}

  getTopN(n: number): Item[] {
    if (!this.sorted) {
      this.sorted = [...this.items].sort((a, b) => b.score - a.score);
    }
    return this.sorted.slice(0, n);
  }

  invalidate() { this.sorted = null; }
}
```

### 4.2 Sort for Min/Max

**Search:**
```
Grep: \.sort\(.*\)\[0\]|\.sort\(.*\)\.slice\(0,\s*1\)|\.sort\(.*\)\.at\(0\)
```

**Slow Pattern:**
```typescript
// BAD: O(n log n) for single element
const max = items.sort((a, b) => b.value - a.value)[0];
```

**Optimized Pattern:**
```typescript
// GOOD: O(n) using reduce
const max = items.reduce((max, item) =>
  item.value > max.value ? item : max
);

// OR: Using Math.max with map
const maxValue = Math.max(...items.map(i => i.value));
```

### 4.3 Custom Sort Without Locale Comparison

**Search:**
```
Grep: \.sort\(.*localeCompare|\.sort\(\(a,\s*b\)\s*=>\s*a\s*[<>]|Collator
```

**Check for string sorting issues:**
- [ ] Case-sensitive when should be insensitive
- [ ] Missing locale-aware comparison

---

## 5. Search Inefficiencies

### 5.1 Linear Search on Sorted Data

**Search:**
```
Grep: \.find\(|\.indexOf\(|\.includes\(
```

**Check if data could be:**
- [ ] Pre-sorted for binary search
- [ ] Indexed in a Map/Set
- [ ] Organized in a search tree

**Slow Pattern:**
```typescript
// BAD: O(n) search on sorted array
const sortedIds = [1, 5, 10, 15, 20, 25, 30];  // Sorted!
function exists(id: number): boolean {
  return sortedIds.includes(id);  // O(n) - ignores sorted property
}
```

**Optimized Pattern:**
```typescript
// GOOD: O(log n) binary search
function binarySearch(sortedArr: number[], target: number): boolean {
  let left = 0, right = sortedArr.length - 1;
  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (sortedArr[mid] === target) return true;
    if (sortedArr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return false;
}

// OR: Use Set for O(1) if memory allows
const idSet = new Set(sortedIds);
```

### 5.2 String Search Patterns

**Search:**
```
Grep: \.search\(|\.match\(|\.indexOf\(.*\.indexOf|new RegExp\(
```

**Check for:**
- [ ] Compiling regex in loops
- [ ] Multiple indexOf calls that could use one regex
- [ ] Case-insensitive search without optimization

**Slow Pattern:**
```typescript
// BAD: Compiling regex in loop
items.filter(item => {
  const regex = new RegExp(searchTerm, 'i');  // Compiled each iteration
  return regex.test(item.name);
});
```

**Optimized Pattern:**
```typescript
// GOOD: Compile regex once
const regex = new RegExp(searchTerm, 'i');
items.filter(item => regex.test(item.name));

// OR: Use toLowerCase for simple case-insensitive
const lowerSearch = searchTerm.toLowerCase();
items.filter(item => item.name.toLowerCase().includes(lowerSearch));
```

---

## 6. Missing Early Returns

### 6.1 Continue Processing After Result Found

**Search:**
```
Grep: for\s*\(|\.forEach\(|while\s*\(
```

**Check for:**
- [ ] Loops that could break early
- [ ] forEach that should be some/every/find

**Slow Pattern:**
```typescript
// BAD: Continues after finding match
function hasAdmin(users: User[]): boolean {
  let found = false;
  users.forEach(user => {
    if (user.role === 'admin') {
      found = true;  // Continues iterating!
    }
  });
  return found;
}
```

**Optimized Pattern:**
```typescript
// GOOD: Stops at first match
function hasAdmin(users: User[]): boolean {
  return users.some(user => user.role === 'admin');
}

// OR: Using for-of with return
function hasAdmin(users: User[]): boolean {
  for (const user of users) {
    if (user.role === 'admin') return true;
  }
  return false;
}
```

### 6.2 Guard Clauses

**Search:**
```
Grep: if\s*\([^)]+\)\s*\{[^}]+return|throw
```

**Check for:**
- [ ] Nested conditionals that could be guard clauses
- [ ] Late validation that wastes cycles

**Slow Pattern:**
```typescript
// BAD: Late validation
function processData(data: Data | null) {
  const result = [];
  for (const item of data?.items || []) {
    if (item.active) {
      // ... expensive processing
    }
  }
  if (!data) {
    throw new Error('No data');  // Too late!
  }
  return result;
}
```

**Optimized Pattern:**
```typescript
// GOOD: Early return
function processData(data: Data | null) {
  if (!data) {
    throw new Error('No data');
  }

  return data.items
    .filter(item => item.active)
    .map(item => /* processing */);
}
```

---

## 7. Tech Stack Specific Patterns

### 7.1 React Performance

**Search:**
```
Grep: useState|useEffect|useContext|React\.memo|useMemo|useCallback
Glob: **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Missing React.memo on frequently re-rendered components
- [ ] Inline object/array props causing re-renders
- [ ] Missing dependency arrays in hooks

### 7.2 Vue Performance

**Search:**
```
Grep: v-for|v-if|computed|watch|@Watch
Glob: **/*.vue
```

**Check for:**
- [ ] v-for without :key
- [ ] v-if with v-for on same element
- [ ] Methods that should be computed

### 7.3 Angular Performance

**Search:**
```
Grep: \*ngFor|trackBy|ChangeDetectionStrategy|OnPush
Glob: **/*.ts, **/*.html
```

**Check for:**
- [ ] Missing trackBy in *ngFor
- [ ] Missing OnPush change detection
- [ ] Pure pipes for transformations

### 7.4 Python Performance

**Search:**
```
Grep: for\s+\w+\s+in|list\(|\.append\(|range\(len\(
Glob: **/*.py
```

**Check for:**
- [ ] List comprehension vs loop append
- [ ] Generator vs list for large data
- [ ] range(len()) instead of enumerate

**Slow Pattern:**
```python
# BAD: Loop with append
result = []
for item in items:
    if item.active:
        result.append(item.name)
```

**Optimized Pattern:**
```python
# GOOD: List comprehension
result = [item.name for item in items if item.active]

# BETTER for large data: Generator
result = (item.name for item in items if item.active)
```

### 7.5 Go Performance

**Search:**
```
Grep: for\s+.*range|append\(|make\(
Glob: **/*.go
```

**Check for:**
- [ ] Pre-allocating slices when size is known
- [ ] Using maps for lookups
- [ ] Avoiding unnecessary allocations

### 7.6 Java/Spring Performance

**Search:**
```
Grep: for\s*\(|\.stream\(\)|\.forEach\(|ArrayList|HashMap
Glob: **/*.java
```

**Check for:**
- [ ] Stream vs loop for simple operations
- [ ] Pre-sizing collections
- [ ] StringBuilder vs string concatenation

---

## 8. Impact Classification

### Critical
- O(n^2) or worse in hot paths
- Exponential recursion without memoization
- Unbounded loops processing user data
- Blocking operations in event loops

### High
- Nested array methods with O(n) lookups
- Missing memoization in React render
- Sorting for single element retrieval
- Linear search on large sorted datasets

### Medium
- Multiple array traversals that could be combined
- Array used where Set/Map would be better
- Regex compiled in loops
- Missing early returns

### Low
- Suboptimal but not critical patterns
- Minor data structure choices
- Style preferences with minimal impact

---

## 9. Output Format

For each finding:

```
FINDING: [PERF01] Title
IMPACT: Critical|High|Medium|Low
EFFORT: Low|Medium|High
FILE: path/to/file.ts:lineNumber
DESCRIPTION: What the performance issue is and why it matters
SLOW_CODE:
```typescript
// Current implementation
```
OPTIMIZED_CODE:
```typescript
// Recommended optimized implementation
```
ESTIMATED_GAIN: "Reduces from O(n^2) to O(n), ~10x faster for 1000 items"
BENCHMARK: "Measure with: console.time('operation'); operation(); console.timeEnd('operation');"
```
