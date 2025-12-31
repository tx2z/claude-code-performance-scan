---
description: "[Internal] Async operations agent - use /performance-scan instead"
disable-model-invocation: true
allowed-tools: Read, Glob, Grep
---

# Async Operations Agent (PERF06)

Scan for performance issues related to:
- **Sequential Async** - Serial operations that could be parallel
- **Missing Promise.all** - Inefficient promise handling
- **Callback Patterns** - Legacy callbacks instead of async/await
- **Missing Request Batching** - Individual requests that could be batched
- **Polling vs WebSockets** - Inefficient real-time patterns
- **Missing Debounce/Throttle** - Excessive event handling
- **Race Conditions** - Async timing issues

---

## 1. Sequential Async That Could Be Parallel

### 1.1 Independent Await Statements

**Search:**
```
Grep: await\s+\w+;?\s*\n\s*await|await\s+.*;\s*await|await\s+this\.\w+\([^)]*\);\s*await
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Multiple awaits on independent operations
- [ ] Sequential API calls that don't depend on each other
- [ ] Database queries that could be parallel

**Sequential Pattern:**
```typescript
// BAD: Sequential execution (slow)
async function loadDashboardData(userId: string) {
  const user = await this.userService.findById(userId);
  const orders = await this.orderService.findByUserId(userId);
  const notifications = await this.notificationService.findByUserId(userId);
  const analytics = await this.analyticsService.getForUser(userId);

  return { user, orders, notifications, analytics };
}
// Time: A + B + C + D = ~4 seconds if each takes 1s
```

**Optimized Pattern:**
```typescript
// GOOD: Parallel execution
async function loadDashboardData(userId: string) {
  const [user, orders, notifications, analytics] = await Promise.all([
    this.userService.findById(userId),
    this.orderService.findByUserId(userId),
    this.notificationService.findByUserId(userId),
    this.analyticsService.getForUser(userId)
  ]);

  return { user, orders, notifications, analytics };
}
// Time: max(A, B, C, D) = ~1 second if each takes 1s

// GOOD: With error handling
async function loadDashboardData(userId: string) {
  const results = await Promise.allSettled([
    this.userService.findById(userId),
    this.orderService.findByUserId(userId),
    this.notificationService.findByUserId(userId),
    this.analyticsService.getForUser(userId)
  ]);

  const [user, orders, notifications, analytics] = results.map((r, i) =>
    r.status === 'fulfilled' ? r.value : null
  );

  return { user, orders, notifications, analytics };
}
```

### 1.2 Loop Awaits

**Search:**
```
Grep: for\s*\([^)]*\)\s*\{[^}]*await|\.forEach\([^)]*async|for\s+await\s+\(
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] await inside for/forEach loops
- [ ] Map operations that could use Promise.all
- [ ] Serial processing when parallel is safe

**Sequential Loop Pattern:**
```typescript
// BAD: Serial processing
async function processUsers(userIds: string[]) {
  const results = [];
  for (const id of userIds) {
    const result = await processUser(id);  // One at a time!
    results.push(result);
  }
  return results;
}
// 100 users x 100ms each = 10 seconds
```

**Optimized Pattern:**
```typescript
// GOOD: Parallel processing
async function processUsers(userIds: string[]) {
  return Promise.all(userIds.map(id => processUser(id)));
}
// 100 users = ~100ms total (parallel)

// GOOD: With concurrency limit
import pLimit from 'p-limit';

async function processUsers(userIds: string[]) {
  const limit = pLimit(10);  // Max 10 concurrent
  return Promise.all(
    userIds.map(id => limit(() => processUser(id)))
  );
}
```

### 1.3 Chained Then Callbacks

**Search:**
```
Grep: \.then\([^)]+\)\.then\(|\.then\([^)]+\)\s*\.then\(
Glob: **/*.ts, **/*.js
```

**Check for:**
- [ ] Promise chains that could be parallel
- [ ] Independent operations in chain

**Chained Pattern:**
```typescript
// BAD: Sequential promise chain
function loadData() {
  return fetchUser()
    .then(user => fetchOrders(user.id))
    .then(orders => fetchItems(orders))
    .then(items => { /* ... */ });
}
```

**Optimized Pattern:**
```typescript
// GOOD: Async/await with parallelism
async function loadData() {
  const user = await fetchUser();
  const [orders, preferences] = await Promise.all([
    fetchOrders(user.id),
    fetchPreferences(user.id)  // Independent of orders
  ]);
  return { user, orders, preferences };
}
```

---

## 2. Promise.all Best Practices

### 2.1 Missing Promise.all

**Search:**
```
Grep: Promise\.all|Promise\.allSettled|Promise\.race|Promise\.any
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

Cross-reference with async patterns to find missing usage.

### 2.2 Promise.all with Dynamic Array

**Search:**
```
Grep: Promise\.all\(\s*\[|\.map\([^)]*=>\s*\{[^}]*await
```

**Check for:**
- [ ] Promise.all with .map for collections
- [ ] Proper error handling with allSettled

**Missing Promise.all Pattern:**
```typescript
// BAD: Processing array sequentially
const results = [];
for (const item of items) {
  results.push(await process(item));
}

// BAD: Creating promises without awaiting
const promises = items.map(item => process(item));  // Runs in parallel but not awaited!
```

**Optimized Pattern:**
```typescript
// GOOD: Promise.all with map
const results = await Promise.all(
  items.map(item => process(item))
);

// GOOD: With error handling
const results = await Promise.allSettled(
  items.map(item => process(item))
);
const successful = results
  .filter((r): r is PromiseFulfilledResult<T> => r.status === 'fulfilled')
  .map(r => r.value);
```

### 2.3 Promise.race for Timeout

**Search:**
```
Grep: Promise\.race|timeout|AbortController|signal
```

**Check for:**
- [ ] Long-running operations have timeouts
- [ ] Race condition with timeout promise
- [ ] AbortController for cancellation

**No Timeout Pattern:**
```typescript
// BAD: No timeout, could hang forever
async function fetchData() {
  return fetch('/api/data').then(r => r.json());
}
```

**Optimized Pattern:**
```typescript
// GOOD: With timeout
async function fetchWithTimeout(url: string, ms: number) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), ms);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return response.json();
  } finally {
    clearTimeout(timeoutId);
  }
}

// ALTERNATIVE: Promise.race
async function fetchWithTimeout(url: string, ms: number) {
  return Promise.race([
    fetch(url).then(r => r.json()),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), ms)
    )
  ]);
}
```

---

## 3. Callback to Async/Await Migration

### 3.1 Callback Patterns

**Search:**
```
Grep: callback\(|cb\(|function\s*\([^)]*,\s*(callback|cb|done|next)\)|err,\s*data
Glob: **/*.ts, **/*.js
```

**Check for:**
- [ ] Legacy callback APIs
- [ ] Pyramid of doom patterns
- [ ] Mixed callback/promise code

**Callback Pattern:**
```javascript
// BAD: Callback hell
function processOrder(orderId, callback) {
  getOrder(orderId, (err, order) => {
    if (err) return callback(err);
    getUser(order.userId, (err, user) => {
      if (err) return callback(err);
      processPayment(order, user, (err, result) => {
        if (err) return callback(err);
        callback(null, result);
      });
    });
  });
}
```

**Optimized Pattern:**
```typescript
// GOOD: Promisified and async/await
import { promisify } from 'util';

const getOrderAsync = promisify(getOrder);
const getUserAsync = promisify(getUser);
const processPaymentAsync = promisify(processPayment);

async function processOrder(orderId: string) {
  const order = await getOrderAsync(orderId);
  const user = await getUserAsync(order.userId);
  return processPaymentAsync(order, user);
}
```

### 3.2 Event Emitter Patterns

**Search:**
```
Grep: \.on\(['"]|\.once\(['"]|EventEmitter|\.emit\(['"]
Glob: **/*.ts, **/*.js
```

**Check for:**
- [ ] Event-based flows that could be promisified
- [ ] Missing error handling on events
- [ ] Once listeners for single events

**Event Pattern:**
```javascript
// BAD: Complex event handling
function processStream(stream) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    stream.on('data', chunk => chunks.push(chunk));
    stream.on('end', () => resolve(Buffer.concat(chunks)));
    stream.on('error', reject);
  });
}
```

**Optimized Pattern:**
```typescript
// GOOD: Using async iterators
async function processStream(stream: ReadableStream) {
  const chunks: Buffer[] = [];
  for await (const chunk of stream) {
    chunks.push(chunk);
  }
  return Buffer.concat(chunks);
}

// GOOD: Using events.once
import { once } from 'events';

async function waitForReady(emitter: EventEmitter) {
  await once(emitter, 'ready');
}
```

---

## 4. Request Batching

### 4.1 Individual API Calls

**Search:**
```
Grep: fetch\(|axios\.|http\.|\.get\(|\.post\(
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Multiple similar requests that could be batched
- [ ] Loop with individual API calls
- [ ] DataLoader pattern opportunities

**Unbatched Pattern:**
```typescript
// BAD: N+1 API calls
async function getUsersWithPosts(userIds: string[]) {
  return Promise.all(userIds.map(async id => {
    const user = await fetch(`/api/users/${id}`).then(r => r.json());
    const posts = await fetch(`/api/users/${id}/posts`).then(r => r.json());
    return { ...user, posts };
  }));
}
// 10 users = 20 requests
```

**Optimized Pattern:**
```typescript
// GOOD: Batch API endpoints
async function getUsersWithPosts(userIds: string[]) {
  const [users, allPosts] = await Promise.all([
    fetch('/api/users/batch', {
      method: 'POST',
      body: JSON.stringify({ ids: userIds })
    }).then(r => r.json()),
    fetch('/api/posts/batch', {
      method: 'POST',
      body: JSON.stringify({ userIds })
    }).then(r => r.json())
  ]);

  return users.map(user => ({
    ...user,
    posts: allPosts.filter(p => p.userId === user.id)
  }));
}
// 10 users = 2 requests
```

### 4.2 DataLoader Pattern

**Search:**
```
Grep: DataLoader|dataloader|batchLoadFn|loader\.load\(
Glob: **/*.ts, **/*.js
```

**Check for:**
- [ ] GraphQL resolvers using DataLoader
- [ ] Batched database queries
- [ ] Request coalescing

**No DataLoader Pattern:**
```typescript
// BAD: Individual loads in resolver
@ResolveField()
async author(@Parent() post: Post) {
  return this.userService.findById(post.authorId);  // N+1!
}
```

**Optimized Pattern:**
```typescript
// GOOD: DataLoader for batching
const userLoader = new DataLoader(async (ids: string[]) => {
  const users = await this.userService.findByIds(ids);
  const userMap = new Map(users.map(u => [u.id, u]));
  return ids.map(id => userMap.get(id) || null);
});

@ResolveField()
async author(@Parent() post: Post, @Context() ctx) {
  return ctx.loaders.user.load(post.authorId);  // Batched!
}
```

---

## 5. Polling vs Real-time

### 5.1 Polling Detection

**Search:**
```
Grep: setInterval|setTimeout.*setTimeout|poll|polling|refetch|refetchInterval
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Short polling intervals (< 5 seconds)
- [ ] Polling for real-time data needs
- [ ] Missing WebSocket/SSE for live updates

**Polling Pattern:**
```typescript
// BAD: Aggressive polling
function startPolling() {
  setInterval(async () => {
    const data = await fetch('/api/notifications').then(r => r.json());
    updateUI(data);
  }, 1000);  // Every second!
}
```

**Optimized Pattern:**
```typescript
// GOOD: WebSocket for real-time
function connectRealtime() {
  const ws = new WebSocket('wss://api.example.com/ws');

  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    updateUI(data);
  };

  return () => ws.close();
}

// GOOD: Server-Sent Events for one-way
function connectSSE() {
  const eventSource = new EventSource('/api/notifications/stream');

  eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    updateUI(data);
  };

  return () => eventSource.close();
}

// ACCEPTABLE: Long polling with exponential backoff
async function longPoll() {
  let delay = 1000;
  while (true) {
    try {
      const data = await fetch('/api/notifications/long-poll').then(r => r.json());
      updateUI(data);
      delay = 1000;  // Reset on success
    } catch (error) {
      delay = Math.min(delay * 2, 30000);  // Exponential backoff
    }
    await sleep(delay);
  }
}
```

### 5.2 React Query/SWR Refetch

**Search:**
```
Grep: refetchInterval|refetchOnWindowFocus|staleTime|cacheTime
Glob: **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Appropriate refetch intervals
- [ ] Stale time configuration
- [ ] Window focus refetch disabled when not needed

**Aggressive Refetch:**
```tsx
// BAD: Too frequent refetching
const { data } = useQuery('data', fetchData, {
  refetchInterval: 1000,  // Every second
  refetchOnWindowFocus: true,
  staleTime: 0
});
```

**Optimized Pattern:**
```tsx
// GOOD: Appropriate intervals and stale time
const { data } = useQuery('data', fetchData, {
  refetchInterval: 30000,  // Every 30 seconds
  staleTime: 10000,  // Consider fresh for 10 seconds
  refetchOnWindowFocus: false  // Only if needed
});

// GOOD: Using mutation invalidation instead of polling
const mutation = useMutation(updateData, {
  onSuccess: () => {
    queryClient.invalidateQueries('data');
  }
});
```

---

## 6. Debounce and Throttle

### 6.1 Missing Debounce

**Search:**
```
Grep: debounce|onInput|onChange|onKeyUp|onKeyDown|onScroll|onResize|search.*input|filter.*input
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx, **/*.vue
```

**Check for:**
- [ ] Search inputs debounced
- [ ] Form validation debounced
- [ ] API calls on input changes

**No Debounce Pattern:**
```tsx
// BAD: API call on every keystroke
function SearchComponent() {
  const [query, setQuery] = useState('');

  const handleChange = async (e: ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
    const results = await search(e.target.value);  // Called on every keystroke!
    setResults(results);
  };

  return <input value={query} onChange={handleChange} />;
}
```

**Optimized Pattern:**
```tsx
// GOOD: Debounced search
import { useDebouncedCallback } from 'use-debounce';

function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const debouncedSearch = useDebouncedCallback(async (value: string) => {
    const results = await search(value);
    setResults(results);
  }, 300);

  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
    debouncedSearch(e.target.value);
  };

  return <input value={query} onChange={handleChange} />;
}

// ALTERNATIVE: Using lodash
import debounce from 'lodash/debounce';

const debouncedSearch = useCallback(
  debounce(async (value: string) => {
    const results = await search(value);
    setResults(results);
  }, 300),
  []
);
```

### 6.2 Missing Throttle

**Search:**
```
Grep: throttle|onScroll|onMouseMove|onDrag|onResize|requestAnimationFrame
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Scroll handlers throttled
- [ ] Resize handlers throttled
- [ ] Mouse/touch move handlers throttled

**No Throttle Pattern:**
```typescript
// BAD: Expensive calculation on every scroll
window.addEventListener('scroll', () => {
  const scrollPercent = window.scrollY / document.body.scrollHeight;
  updateProgressBar(scrollPercent);  // Called 100+ times per second
});
```

**Optimized Pattern:**
```typescript
// GOOD: Throttled scroll handler
import throttle from 'lodash/throttle';

const handleScroll = throttle(() => {
  const scrollPercent = window.scrollY / document.body.scrollHeight;
  updateProgressBar(scrollPercent);
}, 100);  // Max 10 times per second

window.addEventListener('scroll', handleScroll);

// BETTER: requestAnimationFrame for visual updates
let ticking = false;
window.addEventListener('scroll', () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      const scrollPercent = window.scrollY / document.body.scrollHeight;
      updateProgressBar(scrollPercent);
      ticking = false;
    });
    ticking = true;
  }
});
```

---

## 7. Race Condition Prevention

### 7.1 Request Race Conditions

**Search:**
```
Grep: AbortController|signal|cancel|cancelled|isAborted|isCancelled
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Previous requests cancelled on new request
- [ ] Stale response handling
- [ ] Request deduplication

**Race Condition Pattern:**
```tsx
// BAD: Race condition in search
function SearchComponent() {
  const [results, setResults] = useState([]);

  const handleSearch = async (query: string) => {
    const results = await search(query);
    setResults(results);  // May overwrite newer results!
  };

  return <input onChange={(e) => handleSearch(e.target.value)} />;
}
// Fast type "abc" -> 3 requests, last one to complete "wins" (might be "a")
```

**Optimized Pattern:**
```tsx
// GOOD: Cancel previous requests
function SearchComponent() {
  const [results, setResults] = useState([]);
  const abortControllerRef = useRef<AbortController | null>(null);

  const handleSearch = async (query: string) => {
    // Cancel previous request
    abortControllerRef.current?.abort();
    abortControllerRef.current = new AbortController();

    try {
      const results = await search(query, {
        signal: abortControllerRef.current.signal
      });
      setResults(results);
    } catch (err) {
      if (err.name !== 'AbortError') throw err;
    }
  };

  return <input onChange={(e) => handleSearch(e.target.value)} />;
}

// ALTERNATIVE: Request ID tracking
let currentRequestId = 0;

async function handleSearch(query: string) {
  const requestId = ++currentRequestId;
  const results = await search(query);

  // Only update if this is still the latest request
  if (requestId === currentRequestId) {
    setResults(results);
  }
}
```

### 7.2 State Race Conditions

**Search:**
```
Grep: setState|setResults|setData|useEffect.*async|useState.*async
Glob: **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Setting state after component unmount
- [ ] Cleanup in useEffect
- [ ] Mounted check for async operations

**Race Condition Pattern:**
```tsx
// BAD: Setting state after unmount
function DataComponent({ id }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetchData(id).then(setData);  // Component may be unmounted!
  }, [id]);

  return <div>{data?.name}</div>;
}
```

**Optimized Pattern:**
```tsx
// GOOD: Cleanup and mounted check
function DataComponent({ id }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    let isMounted = true;

    fetchData(id).then(result => {
      if (isMounted) {
        setData(result);
      }
    });

    return () => {
      isMounted = false;
    };
  }, [id]);

  return <div>{data?.name}</div>;
}

// BETTER: Using AbortController
function DataComponent({ id }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    fetchData(id, { signal: controller.signal })
      .then(setData)
      .catch(err => {
        if (err.name !== 'AbortError') console.error(err);
      });

    return () => controller.abort();
  }, [id]);

  return <div>{data?.name}</div>;
}
```

---

## 8. Concurrent Operations

### 8.1 Semaphore/Concurrency Limit

**Search:**
```
Grep: p-limit|pLimit|Semaphore|concurrency|maxConcurrent|poolSize
Glob: **/*.ts, **/*.js
```

**Check for:**
- [ ] Bulk operations have concurrency limits
- [ ] External API calls rate limited
- [ ] Database connections limited

**Unlimited Concurrency Pattern:**
```typescript
// BAD: Could overwhelm resources
async function processAll(items: Item[]) {
  return Promise.all(items.map(item => processItem(item)));
}
// 10000 items = 10000 concurrent operations!
```

**Optimized Pattern:**
```typescript
// GOOD: Limited concurrency
import pLimit from 'p-limit';

const limit = pLimit(10);  // Max 10 concurrent

async function processAll(items: Item[]) {
  return Promise.all(
    items.map(item => limit(() => processItem(item)))
  );
}

// ALTERNATIVE: Batch processing
async function processInBatches(items: Item[], batchSize = 10) {
  const results = [];
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await Promise.all(
      batch.map(item => processItem(item))
    );
    results.push(...batchResults);
  }
  return results;
}
```

---

## 9. Impact Classification

### Critical
- Sequential awaits for independent operations (3+ operations)
- Polling every second for non-critical data
- Missing debounce on search/filter with API calls
- Loop await on large collections

### High
- Missing Promise.all for 2+ independent operations
- No timeout on external API calls
- Missing request cancellation in search
- setInterval without clear for real-time needs

### Medium
- Callback patterns that could be promisified
- Missing throttle on scroll/resize handlers
- Short polling interval (< 10 seconds)
- No concurrency limit on bulk operations

### Low
- Minor callback to async/await refactors
- Optional debounce/throttle opportunities
- Race condition edge cases

---

## 10. Output Format

For each finding:

```
FINDING: [PERF06] Title
IMPACT: Critical|High|Medium|Low
EFFORT: Low|Medium|High
FILE: path/to/file.ts:lineNumber
DESCRIPTION: What the async issue is and performance impact
SLOW_CODE:
```typescript
// Current implementation
```
OPTIMIZED_CODE:
```typescript
// Recommended optimized implementation
```
ESTIMATED_GAIN: "Reduces load time from 4s to 1s by parallelizing 4 independent requests"
BENCHMARK: "Measure with: console.time() or performance.mark() and measure()"
```
