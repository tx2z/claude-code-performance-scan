---
description: "[Internal] Network agent - use /performance-scan instead"
disable-model-invocation: true
allowed-tools: Read, Glob, Grep
---

# Network Agent (PERF05)

Scan for performance issues related to:
- **Missing Compression** - Uncompressed responses
- **No Caching Headers** - Missing or incorrect cache configuration
- **Waterfall Requests** - Sequential requests that could be parallel
- **Large Payloads** - Over-fetching data, verbose responses
- **Missing CDN** - Static assets not on CDN
- **HTTP/1.1 vs HTTP/2** - Not leveraging multiplexing
- **Missing Preload/Prefetch** - No hints for critical resources
- **Too Many Requests** - Request sprawl, missing batching

---

## 1. Missing Compression

### 1.1 Server Response Compression

**Search:**
```
Grep: compression|gzip|brotli|deflate|Content-Encoding|app\.use\(compression|CompressionMiddleware
Glob: **/*.ts, **/*.js, **/server.ts, **/main.ts, **/app.ts, **/*.py, **/*.go
```

**Check for:**
- [ ] Compression middleware enabled
- [ ] Brotli preferred over gzip
- [ ] Minimum size threshold configured

**Missing Compression (Express):**
```typescript
// BAD: No compression
const app = express();
app.use(express.json());
// Missing compression middleware
```

**Optimized Pattern:**
```typescript
// GOOD: Compression enabled
import compression from 'compression';

const app = express();
app.use(compression({
  filter: (req, res) => {
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  },
  threshold: 1024,  // Only compress if > 1KB
  level: 6  // Compression level (1-9)
}));
```

**NestJS Pattern:**
```typescript
// GOOD: NestJS with compression
import * as compression from 'compression';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(compression());
  await app.listen(3000);
}
```

**Nginx Pattern:**
```nginx
# GOOD: Nginx compression
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_proxied any;
gzip_types text/plain text/css text/xml text/javascript
           application/javascript application/json
           application/xml image/svg+xml;
gzip_comp_level 6;

# Brotli (if module installed)
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css text/xml text/javascript
             application/javascript application/json;
```

### 1.2 Static Asset Compression

**Search:**
```
Glob: **/webpack.config.*, **/vite.config.*, **/*.config.js
Grep: CompressionPlugin|vite-plugin-compression|brotli|gzip
```

**Check for:**
- [ ] Pre-compressed static assets (.gz, .br)
- [ ] Build tool compression plugins

**Uncompressed Pattern:**
```javascript
// BAD: No static asset compression
module.exports = {
  output: {
    filename: '[name].[contenthash].js'
  }
  // No compression plugin
};
```

**Optimized Pattern:**
```javascript
// GOOD: Pre-compress static assets
const CompressionPlugin = require('compression-webpack-plugin');

module.exports = {
  plugins: [
    new CompressionPlugin({
      algorithm: 'gzip',
      test: /\.(js|css|html|svg)$/,
      threshold: 1024,
      minRatio: 0.8
    }),
    new CompressionPlugin({
      algorithm: 'brotliCompress',
      test: /\.(js|css|html|svg)$/,
      threshold: 1024,
      minRatio: 0.8,
      filename: '[path][base].br'
    })
  ]
};
```

---

## 2. Caching Headers

### 2.1 Static Asset Caching

**Search:**
```
Grep: Cache-Control|max-age|immutable|ETag|Last-Modified|setHeader.*cache
Glob: **/*.ts, **/*.js, **/nginx.conf, **/.htaccess, **/server.ts
```

**Check for:**
- [ ] Static assets (JS, CSS, images) have long cache
- [ ] Versioned/hashed filenames
- [ ] immutable flag for versioned assets

**Missing Caching (Express):**
```typescript
// BAD: No cache headers
app.use(express.static('public'));
```

**Optimized Pattern:**
```typescript
// GOOD: Proper cache headers
app.use('/static', express.static('public', {
  maxAge: '1y',  // 1 year for versioned assets
  immutable: true
}));

// For API responses
app.get('/api/config', (req, res) => {
  res.set('Cache-Control', 'public, max-age=3600');  // 1 hour
  res.json(config);
});
```

**Nginx Pattern:**
```nginx
# GOOD: Nginx caching
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}

location /api/ {
  add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

### 2.2 API Response Caching

**Search:**
```
Grep: @CacheControl|@Cacheable|res\.set\(['"]Cache-Control|cache-control|s-maxage|stale-while-revalidate
```

**Check for:**
- [ ] Read endpoints have appropriate cache headers
- [ ] CDN cache with s-maxage
- [ ] stale-while-revalidate for better UX

**No API Caching:**
```typescript
// BAD: No caching for read endpoint
@Get('products')
async getProducts() {
  return this.productService.findAll();  // No cache headers
}
```

**Optimized Pattern:**
```typescript
// GOOD: Cache headers for read endpoint
@Get('products')
@Header('Cache-Control', 'public, max-age=60, s-maxage=300, stale-while-revalidate=86400')
async getProducts() {
  return this.productService.findAll();
}

// OR: Using interceptor
@UseInterceptors(CacheInterceptor)
@CacheTTL(300)
@Get('products')
async getProducts() {
  return this.productService.findAll();
}
```

---

## 3. Request Optimization

### 3.1 Waterfall Requests

**Search:**
```
Grep: await.*await|\.then\(.*\.then\(|fetch\(.*\n.*fetch\(|axios\.[get|post].*\n.*axios\.
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Sequential requests that could be parallel
- [ ] Dependent data fetched in proper order
- [ ] Request chaining minimized

**Waterfall Pattern:**
```typescript
// BAD: Sequential requests
async function loadDashboard() {
  const user = await fetchUser();
  const orders = await fetchOrders();
  const notifications = await fetchNotifications();
  const analytics = await fetchAnalytics();

  return { user, orders, notifications, analytics };
}
// Total time: sum of all request times
```

**Optimized Pattern:**
```typescript
// GOOD: Parallel requests
async function loadDashboard() {
  const [user, orders, notifications, analytics] = await Promise.all([
    fetchUser(),
    fetchOrders(),
    fetchNotifications(),
    fetchAnalytics()
  ]);

  return { user, orders, notifications, analytics };
}
// Total time: max of all request times
```

**Dependent Requests Pattern:**
```typescript
// GOOD: Parallel where possible, sequential where needed
async function loadDashboard() {
  // User needed first for other requests
  const user = await fetchUser();

  // These can be parallel, but need user.id
  const [orders, notifications] = await Promise.all([
    fetchOrders(user.id),
    fetchNotifications(user.id)
  ]);

  return { user, orders, notifications };
}
```

### 3.2 Too Many Requests

**Search:**
```
Grep: fetch\(|axios\.|http\.get|HttpClient|\.subscribe\(
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Multiple similar requests batched
- [ ] GraphQL or BFF for aggregation
- [ ] Request deduplication

**Too Many Requests Pattern:**
```typescript
// BAD: Separate request for each item
async function loadProductDetails(productIds: string[]) {
  return Promise.all(
    productIds.map(id => fetch(`/api/products/${id}`))
  );  // 100 products = 100 requests!
}
```

**Optimized Pattern:**
```typescript
// GOOD: Batch request
async function loadProductDetails(productIds: string[]) {
  const response = await fetch('/api/products/batch', {
    method: 'POST',
    body: JSON.stringify({ ids: productIds })
  });
  return response.json();  // 1 request for all products
}

// ALTERNATIVE: GraphQL
const { data } = await client.query({
  query: GET_PRODUCTS,
  variables: { ids: productIds }
});
```

### 3.3 Request Deduplication

**Search:**
```
Grep: SWR|React Query|TanStack|useQuery|useSWR|Apollo
Glob: **/*.ts, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Duplicate requests prevented
- [ ] Same data requested once
- [ ] Client-side caching of requests

**Duplicate Requests Pattern:**
```tsx
// BAD: Each component fetches separately
function Header() {
  const [user, setUser] = useState(null);
  useEffect(() => { fetchUser().then(setUser); }, []);
  return <div>{user?.name}</div>;
}

function Sidebar() {
  const [user, setUser] = useState(null);
  useEffect(() => { fetchUser().then(setUser); }, []);  // Duplicate!
  return <div>{user?.email}</div>;
}
```

**Optimized Pattern:**
```tsx
// GOOD: Using React Query / SWR
function Header() {
  const { data: user } = useQuery('user', fetchUser);
  return <div>{user?.name}</div>;
}

function Sidebar() {
  const { data: user } = useQuery('user', fetchUser);  // Deduped!
  return <div>{user?.email}</div>;
}

// OR: Context/global state
function App() {
  const [user, setUser] = useState(null);
  useEffect(() => { fetchUser().then(setUser); }, []);

  return (
    <UserContext.Provider value={user}>
      <Header />
      <Sidebar />
    </UserContext.Provider>
  );
}
```

---

## 4. Payload Optimization

### 4.1 Over-fetching Data

**Search:**
```
Grep: /api/.*\?|\.find\(\)|\.findMany\(\)|SELECT\s+\*
Glob: **/*.ts, **/*.js, **/api/*.ts
```

**Check for:**
- [ ] Only required fields requested
- [ ] Pagination for lists
- [ ] Sparse fieldsets (API design)

**Over-fetch Pattern:**
```typescript
// BAD: Fetches all product data for list view
app.get('/api/products', async (req, res) => {
  const products = await productRepo.find();  // All fields
  res.json(products);
});

// Response includes descriptions, full images, metadata... (100KB+)
```

**Optimized Pattern:**
```typescript
// GOOD: Select only needed fields
app.get('/api/products', async (req, res) => {
  const products = await productRepo.find({
    select: ['id', 'name', 'price', 'thumbnailUrl']
  });
  res.json(products);
});

// OR: GraphQL for client-specified fields
// OR: Sparse fieldsets parameter
app.get('/api/products', async (req, res) => {
  const fields = req.query.fields?.split(',') || ['id', 'name', 'price'];
  const products = await productRepo.find({ select: fields });
  res.json(products);
});
```

### 4.2 JSON Response Size

**Search:**
```
Grep: res\.json|Response\(|JsonResponse|JSON\.stringify
```

**Check for:**
- [ ] Minimal response structure
- [ ] No redundant nested objects
- [ ] Appropriate precision for numbers

**Large Response Pattern:**
```json
{
  "data": {
    "products": [{
      "id": "123",
      "name": "Product",
      "price": 19.99000000000001,
      "createdAt": "2024-01-01T00:00:00.000Z",
      "updatedAt": "2024-01-01T00:00:00.000Z",
      "metadata": { "key": "value" },
      "description": "Very long description...",
      "images": ["url1", "url2", "url3", "url4", "url5"]
    }]
  },
  "meta": { "total": 1, "page": 1, "perPage": 20 },
  "links": { "self": "...", "next": null, "prev": null }
}
```

**Optimized Response Pattern:**
```json
{
  "products": [{
    "id": "123",
    "name": "Product",
    "price": 19.99,
    "thumb": "url1"
  }],
  "total": 1
}
```

---

## 5. Resource Hints

### 5.1 Missing Preload

**Search:**
```
Grep: <link.*rel=["']preload|rel=["']preconnect|rel=["']dns-prefetch|rel=["']prefetch
Glob: **/*.html, **/index.html, **/_document.tsx, **/_document.js
```

**Check for:**
- [ ] Critical fonts preloaded
- [ ] Above-fold images preloaded
- [ ] Third-party origins preconnected

**Missing Hints Pattern:**
```html
<!-- BAD: No resource hints -->
<head>
  <link rel="stylesheet" href="/styles.css">
</head>
```

**Optimized Pattern:**
```html
<!-- GOOD: Proper resource hints -->
<head>
  <!-- Preconnect to third-party origins -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://api.example.com">
  <link rel="dns-prefetch" href="https://analytics.example.com">

  <!-- Preload critical resources -->
  <link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>
  <link rel="preload" href="/hero.webp" as="image">
  <link rel="preload" href="/critical.css" as="style">

  <!-- Prefetch next page resources -->
  <link rel="prefetch" href="/dashboard.js">

  <link rel="stylesheet" href="/styles.css">
</head>
```

### 5.2 Module Preload

**Search:**
```
Grep: <script.*type=["']module|modulepreload
Glob: **/*.html, **/index.html
```

**Check for:**
- [ ] ES modules have modulepreload
- [ ] Critical module chunks preloaded

**Missing Modulepreload:**
```html
<!-- BAD: No modulepreload for critical modules -->
<script type="module" src="/app.js"></script>
```

**Optimized Pattern:**
```html
<!-- GOOD: Modulepreload for critical path -->
<link rel="modulepreload" href="/app.js">
<link rel="modulepreload" href="/vendor.js">
<script type="module" src="/app.js"></script>
```

---

## 6. CDN Usage

### 6.1 Static Assets on CDN

**Search:**
```
Grep: cloudfront|cdn\.|akamai|fastly|cloudflare|bunny|jsdelivr|unpkg
Glob: **/*.ts, **/*.js, **/*.html, **/config*.ts
```

**Check for:**
- [ ] Static assets served from CDN
- [ ] CDN configured for compression
- [ ] Edge caching enabled

**No CDN Pattern:**
```typescript
// BAD: Serving static assets from origin
app.use('/static', express.static('public'));
// All requests hit origin server
```

**Optimized Pattern:**
```typescript
// GOOD: CDN URL for production assets
const CDN_URL = process.env.CDN_URL || '';

function assetUrl(path: string): string {
  return `${CDN_URL}${path}`;
}

// In templates/components
<img src={assetUrl('/images/hero.jpg')} alt="Hero" />
<link rel="stylesheet" href={assetUrl('/styles/main.css')} />
```

### 6.2 API Edge Caching

**Search:**
```
Grep: edge|cdn-cache|surrogate-key|purge|vary
```

**Check for:**
- [ ] Cacheable API responses use CDN
- [ ] Proper Vary headers for CDN
- [ ] Cache invalidation strategy

---

## 7. HTTP/2 Optimization

### 7.1 Multiplexing Utilization

**Search:**
```
Grep: http2|HTTP2|h2|ALPN
Glob: **/nginx.conf, **/server.ts, **/*.go
```

**Check for:**
- [ ] HTTP/2 enabled on server
- [ ] Multiple small resources (not bundled for H1)
- [ ] Server Push configured (if beneficial)

**HTTP/1.1 Pattern:**
```nginx
# Limited parallelism
server {
  listen 443 ssl;
  # ...
}
```

**Optimized Pattern:**
```nginx
# GOOD: HTTP/2 enabled
server {
  listen 443 ssl http2;
  # Multiplexing allows many parallel requests
}
```

### 7.2 Remove HTTP/1.1 Optimizations

With HTTP/2, some H1 optimizations are counter-productive:

**Check for removal of:**
- [ ] Domain sharding (not needed, hurts H2)
- [ ] Sprite sheets (individual files better with H2)
- [ ] Over-bundling (smaller chunks load faster with H2)

---

## 8. Image Optimization

### 8.1 Image Formats

**Search:**
```
Glob: **/*.html, **/*.tsx, **/*.jsx, **/*.vue
Grep: \.(png|jpg|jpeg|gif)["'\s>]
```

**Check for:**
- [ ] WebP with JPEG/PNG fallback
- [ ] AVIF for supported browsers
- [ ] SVG for icons and logos

### 8.2 Image CDN

**Search:**
```
Grep: cloudinary|imgix|imagekit|next/image|Image.*loader
```

**Check for:**
- [ ] Image CDN for dynamic resizing
- [ ] Automatic format conversion
- [ ] Responsive image generation

**Manual Image Pattern:**
```html
<!-- BAD: Serving original images -->
<img src="/uploads/photo.jpg" alt="Photo">
```

**Optimized Pattern:**
```html
<!-- GOOD: Image CDN with transformations -->
<img
  src="https://images.example.com/photo.jpg?w=800&f=auto&q=80"
  srcset="
    https://images.example.com/photo.jpg?w=400&f=auto 400w,
    https://images.example.com/photo.jpg?w=800&f=auto 800w,
    https://images.example.com/photo.jpg?w=1200&f=auto 1200w
  "
  alt="Photo"
>
```

---

## 9. GraphQL Optimization

### 9.1 Over-fetching Prevention

**Search:**
```
Grep: gql`|graphql`|useQuery|useMutation|@Query
Glob: **/*.ts, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Only required fields in queries
- [ ] Fragments for reusable selections
- [ ] Proper query depth

**Over-fetch Pattern:**
```graphql
# BAD: Fetching more than needed
query GetUser {
  user {
    id
    name
    email
    profile {
      bio
      avatar
      socialLinks {
        twitter
        github
        linkedin
      }
    }
    posts {
      id
      title
      content
      comments { ... }
    }
  }
}
```

**Optimized Pattern:**
```graphql
# GOOD: Only needed fields
query GetUserCard {
  user {
    id
    name
    profile {
      avatar
    }
  }
}
```

### 9.2 Query Batching

**Search:**
```
Grep: BatchHttpLink|batchMax|batch:|dataloader
```

**Check for:**
- [ ] Apollo batching configured
- [ ] DataLoader for N+1 prevention
- [ ] Persisted queries for caching

---

## 10. Impact Classification

### Critical
- No compression (gzip/brotli)
- No cache headers on static assets
- Sequential requests for independent data
- No CDN for global audience

### High
- Missing resource hints (preconnect, preload)
- Large API payloads (over-fetching)
- Too many requests (>20 on page load)
- No API response caching

### Medium
- Missing stale-while-revalidate
- No modulepreload for ES modules
- HTTP/1.1 optimizations with HTTP/2
- Missing image optimization

### Low
- Optional prefetch hints
- Minor payload optimizations
- GraphQL query refinements

---

## 11. Output Format

For each finding:

```
FINDING: [PERF05] Title
IMPACT: Critical|High|Medium|Low
EFFORT: Low|Medium|High
FILE: path/to/file.ts:lineNumber
DESCRIPTION: What the network issue is and user impact
SLOW_CODE:
```typescript
// Current implementation
```
OPTIMIZED_CODE:
```typescript
// Recommended optimized implementation
```
ESTIMATED_GAIN: "Reduces page load by 500ms, saves 100KB bandwidth per request"
BENCHMARK: "Measure with: Network tab waterfall, WebPageTest, or curl timing"
```
