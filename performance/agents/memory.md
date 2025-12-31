---
description: "[Internal] Memory agent - use /performance-scan instead"
disable-model-invocation: true
allowed-tools: Read, Glob, Grep
---

# Memory Agent (PERF02)

Scan for performance issues related to:
- **Memory Leaks** - Unreleased references and resources
- **Large Object Allocations** - Unnecessary memory usage
- **Unbounded Caches** - Caches that grow without limits
- **Closure Memory Retention** - Variables captured unnecessarily
- **Event Listener Accumulation** - Listeners not cleaned up
- **Buffer Inefficiencies** - Suboptimal buffer handling
- **Missing Cleanup** - Resources not released on unmount/destroy

---

## 1. Memory Leaks

### 1.1 Event Listener Leaks

**Search:**
```
Grep: addEventListener|\.on\(|\.subscribe\(|eventEmitter|EventEmitter
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx, **/*.vue
```

**Check for:**
- [ ] Every addEventListener has corresponding removeEventListener
- [ ] Subscriptions are unsubscribed on cleanup
- [ ] Event emitter listeners are removed

**Leak Pattern (JavaScript/TypeScript):**
```typescript
// BAD: Listener never removed
class DataService {
  constructor() {
    window.addEventListener('resize', this.handleResize);
    document.addEventListener('scroll', this.handleScroll);
  }
  // Missing cleanup - listeners persist after instance is gone
}
```

**Fixed Pattern:**
```typescript
// GOOD: Proper cleanup
class DataService {
  constructor() {
    this.handleResize = this.handleResize.bind(this);
    this.handleScroll = this.handleScroll.bind(this);
    window.addEventListener('resize', this.handleResize);
    document.addEventListener('scroll', this.handleScroll);
  }

  destroy() {
    window.removeEventListener('resize', this.handleResize);
    document.removeEventListener('scroll', this.handleScroll);
  }
}
```

**Leak Pattern (React):**
```tsx
// BAD: useEffect without cleanup
function Component() {
  useEffect(() => {
    window.addEventListener('resize', handleResize);
    // Missing return cleanup function!
  }, []);
}
```

**Fixed Pattern:**
```tsx
// GOOD: Cleanup in useEffect
function Component() {
  useEffect(() => {
    const handleResize = () => { /* ... */ };
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);
}
```

### 1.2 Timer Leaks

**Search:**
```
Grep: setInterval|setTimeout|requestAnimationFrame
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx, **/*.vue
```

**Check for:**
- [ ] setInterval cleared with clearInterval
- [ ] setTimeout cleared on component unmount
- [ ] requestAnimationFrame cancelled

**Leak Pattern:**
```typescript
// BAD: Interval never cleared
class Poller {
  start() {
    setInterval(() => this.poll(), 5000);  // Runs forever
  }
}
```

**Fixed Pattern:**
```typescript
// GOOD: Interval tracked and cleared
class Poller {
  private intervalId: NodeJS.Timeout | null = null;

  start() {
    this.intervalId = setInterval(() => this.poll(), 5000);
  }

  stop() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
}
```

**React Pattern:**
```tsx
// GOOD: Timer cleanup in React
function Component() {
  useEffect(() => {
    const timerId = setInterval(() => updateData(), 5000);
    return () => clearInterval(timerId);
  }, []);
}
```

### 1.3 Subscription Leaks (RxJS/Observable)

**Search:**
```
Grep: \.subscribe\(|Subscription|takeUntil|unsubscribe
Glob: **/*.ts, **/*.tsx
```

**Check for:**
- [ ] All subscriptions stored and unsubscribed
- [ ] Using takeUntil pattern for cleanup
- [ ] Using async pipe in Angular templates

**Leak Pattern:**
```typescript
// BAD: Subscription not stored or cleaned
@Component({ /* ... */ })
export class MyComponent implements OnInit {
  ngOnInit() {
    this.dataService.getData().subscribe(data => {
      this.data = data;  // Subscription leaks
    });
  }
}
```

**Fixed Pattern:**
```typescript
// GOOD: Using takeUntil pattern
@Component({ /* ... */ })
export class MyComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.dataService.getData()
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => this.data = data);
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// BETTER: Using async pipe (no manual subscription)
@Component({
  template: `<div>{{ data$ | async }}</div>`
})
export class MyComponent {
  data$ = this.dataService.getData();
}
```

### 1.4 WebSocket/SSE Leaks

**Search:**
```
Grep: WebSocket|EventSource|socket\.io|new WebSocket
Glob: **/*.ts, **/*.js
```

**Check for:**
- [ ] WebSocket closed on cleanup
- [ ] EventSource closed on cleanup
- [ ] Socket.io disconnected properly

**Leak Pattern:**
```typescript
// BAD: WebSocket never closed
function connectToServer() {
  const ws = new WebSocket('wss://api.example.com');
  ws.onmessage = handleMessage;
  // Component unmounts, WebSocket stays open
}
```

**Fixed Pattern:**
```typescript
// GOOD: WebSocket with cleanup
function useWebSocket(url: string) {
  useEffect(() => {
    const ws = new WebSocket(url);
    ws.onmessage = handleMessage;

    return () => {
      ws.close();
    };
  }, [url]);
}
```

---

## 2. Unbounded Caches

### 2.1 Object/Map Caches Without Limits

**Search:**
```
Grep: cache\[|cache\.set\(|Map\(\)|new Map|WeakMap|LRU|lru-cache
Glob: **/*.ts, **/*.js
```

**Check for:**
- [ ] Cache has maximum size limit
- [ ] Old entries evicted (LRU, TTL)
- [ ] Cache cleared periodically or on conditions

**Leak Pattern:**
```typescript
// BAD: Unbounded cache - grows forever
const cache = new Map<string, Data>();

function fetchData(key: string): Data {
  if (!cache.has(key)) {
    cache.set(key, loadData(key));  // Never evicts!
  }
  return cache.get(key)!;
}
```

**Fixed Pattern:**
```typescript
// GOOD: LRU cache with limit
import LRU from 'lru-cache';

const cache = new LRU<string, Data>({
  max: 500,  // Maximum 500 entries
  ttl: 1000 * 60 * 5,  // 5 minute TTL
});

function fetchData(key: string): Data {
  if (!cache.has(key)) {
    cache.set(key, loadData(key));
  }
  return cache.get(key)!;
}
```

**Simple LRU Implementation:**
```typescript
// GOOD: Simple bounded cache
class BoundedCache<K, V> {
  private cache = new Map<K, V>();
  private maxSize: number;

  constructor(maxSize: number) {
    this.maxSize = maxSize;
  }

  set(key: K, value: V) {
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }

  get(key: K): V | undefined {
    return this.cache.get(key);
  }
}
```

### 2.2 Memoization Without Bounds

**Search:**
```
Grep: memoize|memo\(|useMemo|React\.memo|computed
```

**Check for:**
- [ ] Memoization with unique keys limited
- [ ] Cache cleared when appropriate

**Leak Pattern:**
```typescript
// BAD: Memoize with unlimited unique inputs
const memoizedFetch = memoize((userId: string) => fetchUser(userId));
// If called with millions of different userIds, cache grows unbounded
```

**Fixed Pattern:**
```typescript
// GOOD: Memoize with bounded cache
import memoize from 'lodash/memoize';

const memoizedFetch = memoize(
  (userId: string) => fetchUser(userId),
  { maxSize: 100 }  // If using lodash-memoize-cache
);

// OR: Use a Map with manual eviction
```

### 2.3 Array Accumulation

**Search:**
```
Grep: \.push\(|\.unshift\(|\.concat\(|\[\.\.\.\w+,
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Arrays trimmed to max length
- [ ] Old entries removed as new added
- [ ] Logs/history arrays bounded

**Leak Pattern:**
```typescript
// BAD: Log array grows unbounded
const logs: LogEntry[] = [];

function addLog(entry: LogEntry) {
  logs.push(entry);  // Never removes old entries
}
```

**Fixed Pattern:**
```typescript
// GOOD: Bounded log array
const MAX_LOGS = 1000;
const logs: LogEntry[] = [];

function addLog(entry: LogEntry) {
  logs.push(entry);
  if (logs.length > MAX_LOGS) {
    logs.shift();  // Remove oldest
  }
}

// BETTER: Ring buffer for O(1) operations
class RingBuffer<T> {
  private buffer: (T | undefined)[];
  private head = 0;
  private size = 0;

  constructor(private capacity: number) {
    this.buffer = new Array(capacity);
  }

  push(item: T) {
    this.buffer[this.head] = item;
    this.head = (this.head + 1) % this.capacity;
    this.size = Math.min(this.size + 1, this.capacity);
  }
}
```

---

## 3. Closure Memory Retention

### 3.1 Large Objects in Closures

**Search:**
```
Grep: =>\s*\{|function\s*\(|async\s*\(
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Large objects captured in closures unnecessarily
- [ ] References to DOM elements in callbacks
- [ ] Entire parent scope captured when only part needed

**Leak Pattern:**
```typescript
// BAD: Large data captured in closure
function createHandler(largeData: BigObject[]) {
  return () => {
    console.log('Handler called');
    // largeData is captured even though not used
  };
}
```

**Fixed Pattern:**
```typescript
// GOOD: Only capture what's needed
function createHandler(largeData: BigObject[]) {
  const dataLength = largeData.length;  // Extract needed value
  return () => {
    console.log(`Handler called with ${dataLength} items`);
  };
}
```

### 3.2 Callback References in Modules

**Search:**
```
Grep: module\.exports|export\s+(const|function|class)
```

**Check for:**
- [ ] Module-level variables holding closures
- [ ] Callbacks stored in module scope

**Leak Pattern:**
```typescript
// BAD: Module-level array of closures holding references
const callbacks: (() => void)[] = [];

export function registerCallback(cb: () => void) {
  callbacks.push(cb);  // Callbacks may hold references to large objects
}
```

**Fixed Pattern:**
```typescript
// GOOD: Use WeakRef or cleanup mechanism
const callbacks = new Set<() => void>();

export function registerCallback(cb: () => void) {
  callbacks.add(cb);
  return () => callbacks.delete(cb);  // Return cleanup function
}
```

---

## 4. Large Object Allocations

### 4.1 Unnecessary Object Cloning

**Search:**
```
Grep: JSON\.parse\(JSON\.stringify|\.\.\.(\w+)|Object\.assign|structuredClone|\_.cloneDeep|cloneDeep
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Deep cloning where shallow would suffice
- [ ] Cloning in loops
- [ ] Cloning large objects unnecessarily

**Slow Pattern:**
```typescript
// BAD: Deep clone for shallow update
function updateUser(user: User, updates: Partial<User>) {
  const cloned = JSON.parse(JSON.stringify(user));  // Expensive!
  return { ...cloned, ...updates };
}
```

**Optimized Pattern:**
```typescript
// GOOD: Shallow spread for flat objects
function updateUser(user: User, updates: Partial<User>) {
  return { ...user, ...updates };
}

// GOOD: Targeted deep update with immer
import produce from 'immer';
function updateUser(user: User, updates: Partial<User>) {
  return produce(user, draft => {
    Object.assign(draft, updates);
  });
}
```

### 4.2 Large Array Operations

**Search:**
```
Grep: \.map\(|\.filter\(|\.reduce\(|\.flatMap\(|\.flat\(
```

**Check for:**
- [ ] Intermediate arrays for large datasets
- [ ] Chained operations creating multiple arrays
- [ ] Processing entire array when subset needed

**Slow Pattern:**
```typescript
// BAD: Creates 3 intermediate arrays
const result = hugeArray
  .map(x => transform(x))
  .filter(x => x.valid)
  .map(x => x.value);
```

**Optimized Pattern:**
```typescript
// GOOD: Single pass with reduce
const result = hugeArray.reduce((acc, x) => {
  const transformed = transform(x);
  if (transformed.valid) {
    acc.push(transformed.value);
  }
  return acc;
}, []);

// BETTER: Generator for lazy evaluation
function* processItems(items) {
  for (const item of items) {
    const transformed = transform(item);
    if (transformed.valid) {
      yield transformed.value;
    }
  }
}
```

### 4.3 String Concatenation in Loops

**Search:**
```
Grep: \+=\s*['"`]|\+=\s*\w+\s*\+
Glob: **/*.ts, **/*.js, **/*.py, **/*.java, **/*.go
```

**Check for:**
- [ ] String concatenation in loops
- [ ] Building large strings incrementally

**Slow Pattern:**
```typescript
// BAD: O(n^2) string concatenation
let result = '';
for (const item of items) {
  result += item.toString() + ',';
}
```

**Optimized Pattern:**
```typescript
// GOOD: Join array
const result = items.map(item => item.toString()).join(',');

// OR: Use array and join
const parts: string[] = [];
for (const item of items) {
  parts.push(item.toString());
}
const result = parts.join(',');
```

---

## 5. Buffer Inefficiencies

### 5.1 Buffer Allocation in Loops

**Search:**
```
Grep: Buffer\.alloc|Buffer\.from|new ArrayBuffer|new Uint8Array
Glob: **/*.ts, **/*.js
```

**Check for:**
- [ ] Buffers pre-allocated when size known
- [ ] Buffer reuse where possible
- [ ] Streaming instead of buffering large files

**Slow Pattern:**
```typescript
// BAD: Allocating buffers in loop
function processChunks(chunks: Buffer[]): Buffer {
  let result = Buffer.alloc(0);
  for (const chunk of chunks) {
    result = Buffer.concat([result, chunk]);  // O(n^2)!
  }
  return result;
}
```

**Optimized Pattern:**
```typescript
// GOOD: Pre-calculate size and allocate once
function processChunks(chunks: Buffer[]): Buffer {
  const totalLength = chunks.reduce((sum, chunk) => sum + chunk.length, 0);
  const result = Buffer.alloc(totalLength);
  let offset = 0;
  for (const chunk of chunks) {
    chunk.copy(result, offset);
    offset += chunk.length;
  }
  return result;
}

// OR: Use Buffer.concat directly
function processChunks(chunks: Buffer[]): Buffer {
  return Buffer.concat(chunks);  // Efficient internal implementation
}
```

### 5.2 File Buffering

**Search:**
```
Grep: readFileSync|readFile|createReadStream|createWriteStream
Glob: **/*.ts, **/*.js
```

**Check for:**
- [ ] Large files use streaming
- [ ] Memory-mapped files for very large files
- [ ] Chunked processing for large data

**Slow Pattern:**
```typescript
// BAD: Loading entire large file into memory
const data = fs.readFileSync('huge-file.json', 'utf-8');
const parsed = JSON.parse(data);  // All in memory
```

**Optimized Pattern:**
```typescript
// GOOD: Stream processing
import { createReadStream } from 'fs';
import { createInterface } from 'readline';

const fileStream = createReadStream('huge-file.txt');
const rl = createInterface({ input: fileStream });

for await (const line of rl) {
  processLine(line);  // Process line by line
}

// FOR JSON: Use streaming JSON parser
import { parser } from 'stream-json';
import { streamArray } from 'stream-json/streamers/StreamArray';

createReadStream('huge-array.json')
  .pipe(parser())
  .pipe(streamArray())
  .on('data', ({ value }) => processItem(value));
```

---

## 6. React-Specific Memory Issues

### 6.1 State Holding Large Objects

**Search:**
```
Grep: useState\(|useReducer\(|createContext
Glob: **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Large objects in state that could be derived
- [ ] Storing derived data in state
- [ ] Keeping unnecessary history

**Slow Pattern:**
```tsx
// BAD: Storing derived data
function Component({ items }) {
  const [filteredItems, setFilteredItems] = useState([]);
  const [sortedItems, setSortedItems] = useState([]);

  useEffect(() => {
    setFilteredItems(items.filter(/* ... */));
  }, [items]);

  useEffect(() => {
    setSortedItems(filteredItems.sort(/* ... */));
  }, [filteredItems]);
}
```

**Optimized Pattern:**
```tsx
// GOOD: Derive from props/state with useMemo
function Component({ items }) {
  const filteredItems = useMemo(
    () => items.filter(/* ... */),
    [items]
  );

  const sortedItems = useMemo(
    () => [...filteredItems].sort(/* ... */),
    [filteredItems]
  );
}
```

### 6.2 Context Provider Memory

**Search:**
```
Grep: createContext|Provider\s+value|useContext
Glob: **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Context value stability (not recreated on each render)
- [ ] Large objects in context

**Slow Pattern:**
```tsx
// BAD: New object on every render
function AppProvider({ children }) {
  const [user, setUser] = useState(null);

  return (
    <AppContext.Provider value={{ user, setUser }}>
      {children}
    </AppContext.Provider>
  );
}
```

**Optimized Pattern:**
```tsx
// GOOD: Memoized context value
function AppProvider({ children }) {
  const [user, setUser] = useState(null);

  const value = useMemo(
    () => ({ user, setUser }),
    [user]  // setUser is stable
  );

  return (
    <AppContext.Provider value={value}>
      {children}
    </AppContext.Provider>
  );
}
```

---

## 7. Python Memory Issues

### 7.1 Generator vs List

**Search:**
```
Grep: list\(|\.append\(|\[.*for.*in.*\]|yield
Glob: **/*.py
```

**Check for:**
- [ ] Generators for large data processing
- [ ] Generator expressions vs list comprehensions
- [ ] Itertools for memory-efficient operations

**Slow Pattern:**
```python
# BAD: Creates full list in memory
def process_large_file(filename):
    lines = open(filename).readlines()  # All in memory
    results = [process(line) for line in lines]  # Another full list
    return results
```

**Optimized Pattern:**
```python
# GOOD: Generator-based processing
def process_large_file(filename):
    with open(filename) as f:
        for line in f:  # Lazy iteration
            yield process(line)

# Usage
for result in process_large_file('huge.txt'):
    handle(result)
```

### 7.2 Django QuerySet Memory

**Search:**
```
Grep: \.all\(\)|\.filter\(|\.objects\.|iterator\(\)|\.values\(|\.only\(
Glob: **/*.py
```

**Check for:**
- [ ] Using iterator() for large querysets
- [ ] Selecting only needed fields
- [ ] Chunked processing for bulk operations

**Slow Pattern:**
```python
# BAD: Loads all records into memory
def export_users():
    users = User.objects.all()  # All in memory
    for user in users:
        export(user)
```

**Optimized Pattern:**
```python
# GOOD: Using iterator for memory efficiency
def export_users():
    users = User.objects.only('id', 'email').iterator()
    for user in users:
        export(user)

# BETTER: Chunked processing
from django.core.paginator import Paginator

def export_users():
    paginator = Paginator(User.objects.only('id', 'email'), 1000)
    for page_num in range(1, paginator.num_pages + 1):
        for user in paginator.page(page_num):
            export(user)
```

---

## 8. Go Memory Issues

### 8.1 Slice Capacity

**Search:**
```
Grep: make\(\[\]|append\(|cap\(|len\(
Glob: **/*.go
```

**Check for:**
- [ ] Pre-allocating slices when size known
- [ ] Avoiding slice header leaks
- [ ] Proper reslicing

**Slow Pattern:**
```go
// BAD: Multiple reallocations
func processItems(count int) []Result {
    var results []Result
    for i := 0; i < count; i++ {
        results = append(results, process(i))
    }
    return results
}
```

**Optimized Pattern:**
```go
// GOOD: Pre-allocated slice
func processItems(count int) []Result {
    results := make([]Result, 0, count)
    for i := 0; i < count; i++ {
        results = append(results, process(i))
    }
    return results
}
```

### 8.2 String Building

**Search:**
```
Grep: strings\.Builder|\+=.*string|fmt\.Sprintf.*loop
Glob: **/*.go
```

**Slow Pattern:**
```go
// BAD: String concatenation
func buildMessage(items []string) string {
    result := ""
    for _, item := range items {
        result += item + ","
    }
    return result
}
```

**Optimized Pattern:**
```go
// GOOD: strings.Builder
func buildMessage(items []string) string {
    var builder strings.Builder
    builder.Grow(estimatedSize)  // Pre-allocate if known
    for _, item := range items {
        builder.WriteString(item)
        builder.WriteString(",")
    }
    return builder.String()
}

// OR: strings.Join
func buildMessage(items []string) string {
    return strings.Join(items, ",")
}
```

---

## 9. Impact Classification

### Critical
- Event listeners never removed (confirmed leak)
- setInterval/setTimeout without cleanup
- Unbounded cache growing with user data
- WebSocket/SSE connections not closed
- Large file read entirely into memory

### High
- Subscription leaks in Angular/RxJS
- Missing cleanup in React useEffect
- Large objects in closures
- Buffer allocation in loops

### Medium
- Unbounded memoization cache
- Unnecessary deep cloning
- Array accumulation without limits
- Intermediate arrays in chains

### Low
- Minor closure captures
- Suboptimal buffer handling
- Non-critical string concatenation

---

## 10. Output Format

For each finding:

```
FINDING: [PERF02] Title
IMPACT: Critical|High|Medium|Low
EFFORT: Low|Medium|High
FILE: path/to/file.ts:lineNumber
DESCRIPTION: What the memory issue is and potential impact
SLOW_CODE:
```typescript
// Current implementation with memory issue
```
OPTIMIZED_CODE:
```typescript
// Fixed implementation
```
ESTIMATED_GAIN: "Prevents memory leak of ~50MB per hour of usage"
BENCHMARK: "Monitor with: process.memoryUsage() or Chrome DevTools Memory tab"
```
