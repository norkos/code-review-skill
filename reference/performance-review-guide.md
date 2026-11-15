# Performance Review Guide

Performance review guide covering frontend, backend, database, algorithmic complexity, and API performance.

## Table of Contents

- [Frontend Performance (Core Web Vitals)](#frontend-performance-core-web-vitals)
- [JavaScript Performance](#javascript-performance)
- [Memory Management](#memory-management)
- [Database Performance](#database-performance)
- [API Performance](#api-performance)
- [Algorithmic Complexity](#algorithmic-complexity)
- [Performance Review Checklist](#performance-review-checklist)

---

## Frontend Performance (Core Web Vitals)

### 2024 Core Metrics

| Metric | Full Name | Target | Meaning |
|------|------|--------|------|
| **LCP** | Largest Contentful Paint | ≤ 2.5s | Time to render the largest content element |
| **INP** | Interaction to Next Paint | ≤ 200ms | Interaction responsiveness (replaced FID in 2024) |
| **CLS** | Cumulative Layout Shift | ≤ 0.1 | Cumulative layout shift |
| **FCP** | First Contentful Paint | ≤ 1.8s | Time to first content render |
| **TBT** | Total Blocking Time | ≤ 200ms | Main-thread blocking time |

### LCP Optimization Checks

```javascript
// ❌ Lazy-loading the LCP image - delays critical content
<img src="hero.jpg" loading="lazy" />

// ✅ Load the LCP image immediately
<img src="hero.jpg" fetchpriority="high" />

// ❌ Unoptimized image format
<img src="hero.png" />  // PNG file is too large

// ✅ Modern image formats + responsive
<picture>
  <source srcset="hero.avif" type="image/avif" />
  <source srcset="hero.webp" type="image/webp" />
  <img src="hero.jpg" alt="Hero" />
</picture>
```

**Review points:**
- [ ] Does the LCP element set `fetchpriority="high"`?
- [ ] Are WebP/AVIF formats used?
- [ ] Is server-side rendering or static generation used?
- [ ] Is the CDN configured correctly?

### FCP Optimization Checks

```html
<!-- ❌ Render-blocking CSS -->
<link rel="stylesheet" href="all-styles.css" />

<!-- ✅ Inline critical CSS + load the rest asynchronously -->
<style>/* Critical above-the-fold styles */</style>
<link rel="preload" href="styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'" />

<!-- ❌ Render-blocking fonts -->
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2');
}

<!-- ✅ Optimized font display -->
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2');
  font-display: swap;  /* Use system font first, swap once loaded */
}
```

### INP Optimization Checks

```javascript
// ❌ Long task blocks the main thread
button.addEventListener('click', () => {
  // Synchronous operation taking 500ms
  processLargeData(data);
  updateUI();
});

// ✅ Break up long tasks
button.addEventListener('click', async () => {
  // Yield to the main thread
  await scheduler.yield?.() ?? new Promise(r => setTimeout(r, 0));

  // Process in batches
  for (const chunk of chunks) {
    processChunk(chunk);
    await scheduler.yield?.();
  }
  updateUI();
});

// ✅ Use a Web Worker for heavy computation
const worker = new Worker('heavy-computation.js');
worker.postMessage(data);
worker.onmessage = (e) => updateUI(e.data);
```

### CLS Optimization Checks

```css
/* ❌ Media without specified dimensions */
img { width: 100%; }

/* ✅ Reserve space */
img {
  width: 100%;
  aspect-ratio: 16 / 9;
}

/* ❌ Dynamically inserted content causes layout shift */
.ad-container { }

/* ✅ Reserve a fixed height */
.ad-container {
  min-height: 250px;
}
```

**CLS review checklist:**
- [ ] Do images/videos have width/height or aspect-ratio?
- [ ] Does font loading use `font-display: swap`?
- [ ] Is space reserved for dynamic content?
- [ ] Is inserting content above existing content avoided?

---

## JavaScript Performance

### Code Splitting & Lazy Loading

```javascript
// ❌ Load all code at once
import { HeavyChart } from './charts';
import { PDFExporter } from './pdf';
import { AdminPanel } from './admin';

// ✅ Load on demand
const HeavyChart = lazy(() => import('./charts'));
const PDFExporter = lazy(() => import('./pdf'));

// ✅ Route-level code splitting
const routes = [
  {
    path: '/dashboard',
    component: lazy(() => import('./pages/Dashboard')),
  },
  {
    path: '/admin',
    component: lazy(() => import('./pages/Admin')),
  },
];
```

### Bundle Size Optimization

```javascript
// ❌ Import the entire library
import _ from 'lodash';
import moment from 'moment';

// ✅ Import only what you need
import debounce from 'lodash/debounce';
import { format } from 'date-fns';

// ❌ Not tree-shakeable
export default {
  fn1() {},
  fn2() {},  // Unused but still bundled
};

// ✅ Named exports support tree shaking
export function fn1() {}
export function fn2() {}
```

**Bundle review checklist:**
- [ ] Is dynamic import() used for code splitting?
- [ ] Are large libraries imported selectively?
- [ ] Has the bundle size been analyzed? (webpack-bundle-analyzer)
- [ ] Are there unused dependencies?

### List Rendering Optimization

```javascript
// ❌ Rendering a large list
function List({ items }) {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );  // 10,000 items = 10,000 DOM nodes
}

// ✅ Virtualized list - render only visible items
import { FixedSizeList } from 'react-window';

function VirtualList({ items }) {
  return (
    <FixedSizeList
      height={400}
      itemCount={items.length}
      itemSize={35}
    >
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}
```

**Large data review points:**
- [ ] Do lists with more than 100 items use virtual scrolling?
- [ ] Do tables support pagination or virtualization?
- [ ] Is there unnecessary full re-rendering?

---

## Memory Management

### Common Memory Leaks

#### 1. Event Listeners Not Cleaned Up

```javascript
// ❌ Listener remains after the component unmounts
useEffect(() => {
  window.addEventListener('resize', handleResize);
}, []);

// ✅ Clean up the event listener
useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

#### 2. Timers Not Cleaned Up

```javascript
// ❌ Timer is never cleared
useEffect(() => {
  setInterval(fetchData, 5000);
}, []);

// ✅ Clear the timer
useEffect(() => {
  const timer = setInterval(fetchData, 5000);
  return () => clearInterval(timer);
}, []);
```

#### 3. Closure References

```javascript
// ❌ Closure holds a reference to a large object
function createHandler() {
  const largeData = new Array(1000000).fill('x');

  return function handler() {
    // largeData is referenced by the closure and cannot be garbage-collected
    console.log(largeData.length);
  };
}

// ✅ Keep only the data you need
function createHandler() {
  const largeData = new Array(1000000).fill('x');
  const length = largeData.length;  // Keep only the needed value

  return function handler() {
    console.log(length);
  };
}
```

#### 4. Subscriptions Not Cleaned Up

```javascript
// ❌ WebSocket/EventSource never closed
useEffect(() => {
  const ws = new WebSocket('wss://...');
  ws.onmessage = handleMessage;
}, []);

// ✅ Close the connection
useEffect(() => {
  const ws = new WebSocket('wss://...');
  ws.onmessage = handleMessage;
  return () => ws.close();
}, []);
```

### Memory Review Checklist

```markdown
- [ ] Does every useEffect have a cleanup function?
- [ ] Are event listeners removed when the component unmounts?
- [ ] Are timers cleared?
- [ ] Are WebSocket/SSE connections closed?
- [ ] Are large objects released promptly?
- [ ] Are global variables accumulating data?
```

### Detection Tools

| Tool | Purpose |
|------|------|
| Chrome DevTools Memory | Heap snapshot analysis |
| MemLab (Meta) | Automated memory leak detection |
| Performance Monitor | Real-time memory monitoring |

---

## Database Performance

### N+1 Query Problem

```python
# ❌ N+1 problem - 1 + N queries
users = User.objects.all()  # 1 query
for user in users:
    print(user.profile.bio)  # N queries (one per user)

# ✅ Eager loading - 2 queries
users = User.objects.select_related('profile').all()
for user in users:
    print(user.profile.bio)  # No extra queries

# ✅ Use prefetch_related for many-to-many relations
posts = Post.objects.prefetch_related('tags').all()
```

```javascript
// TypeORM example
// ❌ N+1 problem
const users = await userRepository.find();
for (const user of users) {
  const posts = await user.posts;  // Queries on every iteration
}

// ✅ Eager Loading
const users = await userRepository.find({
  relations: ['posts'],
});
```

### Index Optimization

```sql
-- ❌ Full table scan
SELECT * FROM orders WHERE status = 'pending';

-- ✅ Add an index
CREATE INDEX idx_orders_status ON orders(status);

-- ❌ Index not used: function applied to column
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- ✅ Range query can use the index
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- ❌ Index not used: LIKE with leading wildcard
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ Prefix match can use the index
SELECT * FROM products WHERE name LIKE 'phone%';
```

### Query Optimization

```sql
-- ❌ SELECT * fetches unneeded columns
SELECT * FROM users WHERE id = 1;

-- ✅ Select only the columns you need
SELECT id, name, email FROM users WHERE id = 1;

-- ❌ No LIMIT on a large table
SELECT * FROM logs WHERE type = 'error';

-- ✅ Paginated query
SELECT * FROM logs WHERE type = 'error' LIMIT 100 OFFSET 0;

-- ❌ Running queries in a loop
for id in user_ids:
    cursor.execute("SELECT * FROM users WHERE id = %s", (id,))

-- ✅ Batch query
cursor.execute("SELECT * FROM users WHERE id IN %s", (tuple(user_ids),))
```

### Database Review Checklist

```markdown
🔴 Must check:
- [ ] Are there N+1 queries?
- [ ] Are WHERE clause columns indexed?
- [ ] Is SELECT * avoided?
- [ ] Do queries on large tables have a LIMIT?

🟡 Should check:
- [ ] Was EXPLAIN used to analyze the query plan?
- [ ] Is the column order of composite indexes correct?
- [ ] Are there unused indexes?
- [ ] Is slow query logging monitored?
```

---

## API Performance

### Pagination

```javascript
// ❌ Returning all data
app.get('/users', async (req, res) => {
  const users = await User.findAll();  // May return 100,000 rows
  res.json(users);
});

// ✅ Pagination + a maximum page size
app.get('/users', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);  // Max 100
  const offset = (page - 1) * limit;

  const { rows, count } = await User.findAndCountAll({
    limit,
    offset,
    order: [['id', 'ASC']],
  });

  res.json({
    data: rows,
    pagination: {
      page,
      limit,
      total: count,
      totalPages: Math.ceil(count / limit),
    },
  });
});
```

### Caching Strategy

```javascript
// ✅ Redis caching example
async function getUser(id) {
  const cacheKey = `user:${id}`;

  // 1. Check the cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. Query the database
  const user = await db.users.findById(id);

  // 3. Write to the cache (with an expiration)
  await redis.setex(cacheKey, 3600, JSON.stringify(user));

  return user;
}

// ✅ HTTP cache headers
app.get('/static-data', (req, res) => {
  res.set({
    'Cache-Control': 'public, max-age=86400',  // 24 hours
    'ETag': 'abc123',
  });
  res.json(data);
});
```

### Response Compression

```javascript
// ✅ Enable Gzip/Brotli compression
const compression = require('compression');
app.use(compression());

// ✅ Return only the necessary fields
// Request: GET /users?fields=id,name,email
app.get('/users', async (req, res) => {
  const fields = req.query.fields?.split(',') || ['id', 'name'];
  const users = await User.findAll({
    attributes: fields,
  });
  res.json(users);
});
```

### Rate Limiting

```javascript
// ✅ Rate limiting
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 100,             // At most 100 requests
  message: { error: 'Too many requests, please try again later.' },
});

app.use('/api/', limiter);
```

### API Review Checklist

```markdown
- [ ] Do list endpoints have pagination?
- [ ] Is the maximum page size limited?
- [ ] Is hot data cached?
- [ ] Is response compression enabled?
- [ ] Is there rate limiting?
- [ ] Are only the necessary fields returned?
```

---

## Algorithmic Complexity

### Common Complexity Comparison

| Complexity | Name | 10 items | 1,000 items | 1 million items | Example |
|--------|------|-------|---------|----------|------|
| O(1) | Constant | 1 | 1 | 1 | Hash lookup |
| O(log n) | Logarithmic | 3 | 10 | 20 | Binary search |
| O(n) | Linear | 10 | 1000 | 1 million | Array traversal |
| O(n log n) | Linearithmic | 33 | 10000 | 20 million | Quicksort |
| O(n²) | Quadratic | 100 | 1 million | 1 trillion | Nested loops |
| O(2ⁿ) | Exponential | 1024 | ∞ | ∞ | Recursive Fibonacci |

### Spotting It in Code Review

```javascript
// ❌ O(n²) - nested loops
function findDuplicates(arr) {
  const duplicates = [];
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j]) {
        duplicates.push(arr[i]);
      }
    }
  }
  return duplicates;
}

// ✅ O(n) - use a Set
function findDuplicates(arr) {
  const seen = new Set();
  const duplicates = new Set();
  for (const item of arr) {
    if (seen.has(item)) {
      duplicates.add(item);
    }
    seen.add(item);
  }
  return [...duplicates];
}
```

```javascript
// ❌ O(n²) - calls includes on every iteration
function removeDuplicates(arr) {
  const result = [];
  for (const item of arr) {
    if (!result.includes(item)) {  // includes is O(n)
      result.push(item);
    }
  }
  return result;
}

// ✅ O(n) - use a Set
function removeDuplicates(arr) {
  return [...new Set(arr)];
}
```

```javascript
// ❌ O(n) lookup - scans every time
const users = [{ id: 1, name: 'A' }, { id: 2, name: 'B' }, ...];

function getUser(id) {
  return users.find(u => u.id === id);  // O(n)
}

// ✅ O(1) lookup - use a Map
const userMap = new Map(users.map(u => [u.id, u]));

function getUser(id) {
  return userMap.get(id);  // O(1)
}
```

### Space Complexity Considerations

```javascript
// ⚠️ O(n) space - creates a new array
const doubled = arr.map(x => x * 2);

// ✅ O(1) space - modify in place (if allowed)
for (let i = 0; i < arr.length; i++) {
  arr[i] *= 2;
}

// ⚠️ Excessive recursion depth may overflow the stack
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);  // O(n) stack space
}

// ✅ Iterative version, O(1) space
function factorial(n) {
  let result = 1;
  for (let i = 2; i <= n; i++) {
    result *= i;
  }
  return result;
}
```

### Complexity Review Comments

```markdown
💡 "This nested loop is O(n²); it will have performance issues with large data sets"
🔴 "Using Array.includes() inside a loop here makes it O(n²) overall; consider using a Set"
🟡 "This recursion depth may overflow the stack; consider an iterative or tail-recursive version"
```

---

## Performance Review Checklist

### 🔴 Must Check (Blocking)

**Frontend:**
- [ ] Is the LCP image lazy-loaded? (it should not be)
- [ ] Is `transition: all` used?
- [ ] Are width/height/top/left animated?
- [ ] Are lists with >100 items virtualized?

**Backend:**
- [ ] Are there N+1 queries?
- [ ] Do list endpoints have pagination?
- [ ] Is SELECT * used on large tables?

**General:**
- [ ] Are there O(n²) or worse nested loops?
- [ ] Are useEffect hooks/event listeners cleaned up?

### 🟡 Should Check (Important)

**Frontend:**
- [ ] Is code splitting used?
- [ ] Are large libraries imported selectively?
- [ ] Do images use WebP/AVIF?
- [ ] Are there unused dependencies?

**Backend:**
- [ ] Is hot data cached?
- [ ] Are WHERE columns indexed?
- [ ] Is there slow query monitoring?

**API:**
- [ ] Is response compression enabled?
- [ ] Is there rate limiting?
- [ ] Are only the necessary fields returned?

### 🟢 Optimization Suggestions (Nice to Have)

- [ ] Has the bundle size been analyzed?
- [ ] Is a CDN used?
- [ ] Is there performance monitoring?
- [ ] Have performance benchmarks been run?

---

## Performance Thresholds

### Frontend Metrics

| Metric | Good | Needs Improvement | Poor |
|------|-----|--------|-----|
| LCP | ≤ 2.5s | 2.5-4s | > 4s |
| INP | ≤ 200ms | 200-500ms | > 500ms |
| CLS | ≤ 0.1 | 0.1-0.25 | > 0.25 |
| FCP | ≤ 1.8s | 1.8-3s | > 3s |
| Bundle Size (JS) | < 200KB | 200-500KB | > 500KB |

### Backend Metrics

| Metric | Good | Needs Improvement | Poor |
|------|-----|--------|-----|
| API response time | < 100ms | 100-500ms | > 500ms |
| Database query | < 50ms | 50-200ms | > 200ms |
| Page load | < 3s | 3-5s | > 5s |

---

## Recommended Tools

### Frontend Performance

| Tool | Purpose |
|------|------|
| [Lighthouse](https://developer.chrome.com/docs/lighthouse/) | Core Web Vitals testing |
| [WebPageTest](https://www.webpagetest.org/) | Detailed performance analysis |
| [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer) | Bundle analysis |
| [Chrome DevTools Performance](https://developer.chrome.com/docs/devtools/performance/) | Runtime performance analysis |

### Memory Detection

| Tool | Purpose |
|------|------|
| [MemLab](https://github.com/facebookincubator/memlab) | Automated memory leak detection |
| Chrome Memory Tab | Heap snapshot analysis |

### Backend Performance

| Tool | Purpose |
|------|------|
| EXPLAIN | Database query plan analysis |
| [pganalyze](https://pganalyze.com/) | PostgreSQL performance monitoring |
| [New Relic](https://newrelic.com/) / [Datadog](https://www.datadoghq.com/) | APM monitoring |

---

## Low-Level Efficiency Anti-Patterns

Code-level efficiency mistakes, independent of architecture-level performance issues. Complements the resource management and concurrency defects already covered in [common-bugs-checklist.md](common-bugs-checklist.md).

### Unnecessary Repeated Work

- [ ] Is the same function/query called repeatedly within the same request/render?
- [ ] Are files/configs re-read inside a loop (loop-invariant)?
- [ ] Can computed results be cached or passed downstream?

```typescript
// ❌ Loop-invariant work executed repeatedly inside the loop
for (const path of paths) {
  const config = JSON.parse(fs.readFileSync("config.json", "utf-8"));
  processFile(path, config);
}

// ✅ Hoist it out of the loop
const config = JSON.parse(fs.readFileSync("config.json", "utf-8"));
for (const path of paths) processFile(path, config);
```

### Missed Concurrency Opportunities

- [ ] Are independent async operations `await`ed sequentially?
- [ ] Could they run concurrently with `Promise.all` / `asyncio.gather` / `tokio::join!`?

```typescript
// ❌ Sequential await
const a = await fetchA();
const b = await fetchB();

// ✅ Concurrent
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

### Hot Path Bloat

- [ ] Does module-level / import-time code perform heavy operations (file I/O, network, large object construction)?
- [ ] Does the per-request path contain initialization that could be deferred?
- [ ] Does startup code block the first request?

### Unbounded Data Structures

> For resource lifecycle defects (unclosed connections, listeners not removed, timers not cleared), see [common-bugs-checklist.md → Resource Management](common-bugs-checklist.md#resource-management). This section focuses on *capacity bounds*.

- [ ] Do global dicts / lists / caches have a `max-size` or TTL?
- [ ] Do accumulating data structures (queues, logs, metrics buffers) have an upper bound?
- [ ] Are per-request objects held by persistent references that prevent GC?

```python
# ❌ Unbounded cache
_cache: dict[str, Any] = {}

# ✅ Bounded LRU
from functools import lru_cache

@lru_cache(maxsize=256)
def get_cached(key: str) -> Any:
    return expensive_computation(key)
```

---

## References

- [Core Web Vitals - web.dev](https://web.dev/articles/vitals)
- [Optimizing Core Web Vitals - Vercel](https://vercel.com/guides/optimizing-core-web-vitals-in-2024)
- [MemLab - Meta Engineering](https://engineering.fb.com/2022/09/12/open-source/memlab/)
- [Big O Cheat Sheet](https://www.bigocheatsheet.com/)
- [N+1 Query Problem - Stack Overflow](https://stackoverflow.com/questions/97197/what-is-the-n1-selects-problem-in-orm-object-relational-mapping)
- [API Performance Optimization](https://algorithmsin60days.com/blog/optimizing-api-performance/)
