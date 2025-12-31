---
description: "[Internal] Frontend agent - use /performance-scan instead"
disable-model-invocation: true
allowed-tools: Read, Glob, Grep
---

# Frontend Agent (PERF04)

Scan for performance issues related to:
- **Render Blocking Resources** - CSS/JS blocking initial render
- **Large Bundle Sizes** - Oversized JavaScript bundles
- **Unused JavaScript/CSS** - Dead code in production
- **Unoptimized Images** - Missing optimization, wrong formats
- **Missing Lazy Loading** - Eager loading of off-screen content
- **Layout Thrashing** - Forced synchronous layouts
- **Web Vitals Issues** - LCP, FID/INP, CLS problems
- **Missing Code Splitting** - Monolithic bundles
- **Third-party Script Impact** - Blocking external scripts

---

## 1. Render Blocking Resources

### 1.1 Blocking CSS

**Search:**
```
Grep: <link.*stylesheet|<link.*rel=["']stylesheet|@import\s+url|@import\s+['"]
Glob: **/*.html, **/*.tsx, **/*.jsx, **/*.vue, **/index.html
```

**Check for:**
- [ ] Critical CSS inlined
- [ ] Non-critical CSS loaded async
- [ ] Media queries for conditional CSS

**Blocking Pattern:**
```html
<!-- BAD: All CSS blocks rendering -->
<head>
  <link rel="stylesheet" href="/styles/main.css">
  <link rel="stylesheet" href="/styles/components.css">
  <link rel="stylesheet" href="/styles/print.css">
</head>
```

**Optimized Pattern:**
```html
<!-- GOOD: Critical CSS inline, others async -->
<head>
  <style>
    /* Critical above-the-fold CSS inlined */
    .header { ... }
    .hero { ... }
  </style>

  <!-- Non-critical CSS loaded async -->
  <link rel="preload" href="/styles/main.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="/styles/main.css"></noscript>

  <!-- Print CSS with media query (doesn't block) -->
  <link rel="stylesheet" href="/styles/print.css" media="print">
</head>
```

### 1.2 Blocking JavaScript

**Search:**
```
Grep: <script\s+src=|<script.*src=(?!.*defer|async)
Glob: **/*.html, **/index.html
```

**Check for:**
- [ ] Scripts have defer or async
- [ ] Critical scripts at end of body
- [ ] Third-party scripts loaded async

**Blocking Pattern:**
```html
<!-- BAD: Blocking scripts -->
<head>
  <script src="/vendor/analytics.js"></script>
  <script src="/app/main.js"></script>
</head>
```

**Optimized Pattern:**
```html
<!-- GOOD: Non-blocking scripts -->
<head>
  <script src="/app/main.js" defer></script>
  <script src="/vendor/analytics.js" async></script>
</head>

<!-- OR: At end of body -->
<body>
  <!-- content -->
  <script src="/app/main.js"></script>
</body>
```

---

## 2. Large Bundle Sizes

### 2.1 Main Bundle Analysis

**Search:**
```
Glob: **/webpack.config.*, **/vite.config.*, **/rollup.config.*, **/next.config.*
Grep: splitChunks|manualChunks|codesplit|optimization
```

**Check for:**
- [ ] Bundle size < 250KB gzipped for main chunk
- [ ] Code splitting configured
- [ ] Vendor chunks separated
- [ ] Dynamic imports used

**Large Bundle Pattern (Webpack):**
```javascript
// BAD: No code splitting
module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'bundle.js'  // Single monolithic bundle
  }
};
```

**Optimized Pattern:**
```javascript
// GOOD: Code splitting configured
module.exports = {
  entry: './src/index.js',
  output: {
    filename: '[name].[contenthash].js',
    chunkFilename: '[name].[contenthash].chunk.js'
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        }
      }
    }
  }
};
```

### 2.2 Large Dependencies

**Search:**
```
Grep: import.*from ['"]moment|import.*from ['"]lodash['"$]|import.*from ['"]lodash/|require\(['"]moment|require\(['"]lodash
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for large imports:**
- [ ] moment.js -> dayjs or date-fns
- [ ] Full lodash -> lodash-es or individual imports
- [ ] Full rxjs -> individual operators
- [ ] Unused large libraries

**Large Import Pattern:**
```typescript
// BAD: Imports entire library
import _ from 'lodash';  // ~70KB
import moment from 'moment';  // ~66KB

const result = _.map(items, item => item.name);
const date = moment().format('YYYY-MM-DD');
```

**Optimized Pattern:**
```typescript
// GOOD: Individual imports
import map from 'lodash/map';  // ~2KB
import { format } from 'date-fns';  // Tree-shakeable

const result = map(items, item => item.name);
const date = format(new Date(), 'yyyy-MM-dd');
```

### 2.3 Import Size Check

**Search:**
```
Grep: import\s+\*\s+as|from\s+['"][^'"]+['"];?\s*$
```

**Check for:**
- [ ] Namespace imports avoided when possible
- [ ] Named imports for tree shaking
- [ ] No barrel file issues

---

## 3. Unoptimized Images

### 3.1 Image Format Check

**Search:**
```
Grep: <img|src=.*\.(png|jpg|jpeg|gif)|background.*url.*\.(png|jpg|jpeg)|Image.*src=
Glob: **/*.html, **/*.tsx, **/*.jsx, **/*.vue, **/*.css, **/*.scss
```

**Check for:**
- [ ] WebP/AVIF for photographs
- [ ] SVG for icons and logos
- [ ] Appropriate format for content type
- [ ] Next-gen formats with fallbacks

**Unoptimized Pattern:**
```html
<!-- BAD: Large PNG for photo -->
<img src="/images/hero-photo.png" alt="Hero">

<!-- BAD: Raster for icon -->
<img src="/icons/arrow.png" width="24" height="24">
```

**Optimized Pattern:**
```html
<!-- GOOD: WebP with fallback -->
<picture>
  <source srcset="/images/hero-photo.avif" type="image/avif">
  <source srcset="/images/hero-photo.webp" type="image/webp">
  <img src="/images/hero-photo.jpg" alt="Hero">
</picture>

<!-- GOOD: SVG for icon -->
<svg width="24" height="24"><use href="#icon-arrow" /></svg>

<!-- GOOD: Next.js Image optimization -->
<Image src="/images/hero.jpg" width={1200} height={600} alt="Hero" />
```

### 3.2 Missing Responsive Images

**Search:**
```
Grep: <img(?!.*srcset).*src=|<Image(?!.*sizes)
```

**Check for:**
- [ ] srcset for responsive images
- [ ] sizes attribute defined
- [ ] Art direction with picture element

**Non-responsive Pattern:**
```html
<!-- BAD: Same large image for all viewports -->
<img src="/images/hero-2000w.jpg" alt="Hero">
```

**Optimized Pattern:**
```html
<!-- GOOD: Responsive images -->
<img
  src="/images/hero-800w.jpg"
  srcset="
    /images/hero-400w.jpg 400w,
    /images/hero-800w.jpg 800w,
    /images/hero-1200w.jpg 1200w,
    /images/hero-2000w.jpg 2000w
  "
  sizes="(max-width: 600px) 100vw, (max-width: 1200px) 50vw, 800px"
  alt="Hero"
>
```

### 3.3 Missing Lazy Loading

**Search:**
```
Grep: <img(?!.*loading=).*src=|<Image(?!.*loading=|.*priority)
```

**Check for:**
- [ ] Below-fold images have loading="lazy"
- [ ] Above-fold images do NOT have lazy loading
- [ ] Intersection Observer for complex cases

**No Lazy Loading:**
```html
<!-- BAD: All images load immediately -->
<img src="/images/photo1.jpg" alt="Photo 1">
<img src="/images/photo2.jpg" alt="Photo 2">
<img src="/images/photo100.jpg" alt="Photo 100">
```

**Optimized Pattern:**
```html
<!-- GOOD: Above fold loads immediately -->
<img src="/images/hero.jpg" alt="Hero" fetchpriority="high">

<!-- GOOD: Below fold lazy loaded -->
<img src="/images/photo1.jpg" alt="Photo 1" loading="lazy">
<img src="/images/photo2.jpg" alt="Photo 2" loading="lazy">
```

---

## 4. React Performance Issues

### 4.1 Missing Memoization

**Search:**
```
Grep: function\s+\w+Component|const\s+\w+\s*=\s*\([^)]*\)\s*=>|export\s+(default\s+)?function
Glob: **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] React.memo on pure components
- [ ] useMemo for expensive computations
- [ ] useCallback for callback props

**Slow Pattern:**
```tsx
// BAD: Component re-renders on every parent render
function ExpensiveList({ items, onItemClick }) {
  return (
    <ul>
      {items.map(item => (
        <ExpensiveItem
          key={item.id}
          item={item}
          onClick={() => onItemClick(item.id)}  // New function each render
        />
      ))}
    </ul>
  );
}
```

**Optimized Pattern:**
```tsx
// GOOD: Memoized component and callback
const ExpensiveList = memo(function ExpensiveList({ items, onItemClick }) {
  const handleClick = useCallback((id) => {
    onItemClick(id);
  }, [onItemClick]);

  return (
    <ul>
      {items.map(item => (
        <MemoizedItem
          key={item.id}
          item={item}
          onClick={handleClick}
        />
      ))}
    </ul>
  );
});

const MemoizedItem = memo(ExpensiveItem);
```

### 4.2 Inline Object/Array Props

**Search:**
```
Grep: style=\{\{|\[\]}\s*}|=\{\{[^}]+\}\}(?=\s*[/>\)])
Glob: **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Inline style objects
- [ ] Inline object/array props
- [ ] Functions defined in render

**Slow Pattern:**
```tsx
// BAD: New objects/arrays on each render
function Component() {
  return (
    <ChildComponent
      style={{ marginTop: 20 }}  // New object
      options={['a', 'b', 'c']}   // New array
      config={{ enabled: true }} // New object
    />
  );
}
```

**Optimized Pattern:**
```tsx
// GOOD: Stable references
const styles = { marginTop: 20 };
const options = ['a', 'b', 'c'];
const config = { enabled: true };

function Component() {
  return (
    <ChildComponent
      style={styles}
      options={options}
      config={config}
    />
  );
}

// OR: useMemo for derived values
function Component({ enabled }) {
  const config = useMemo(() => ({ enabled }), [enabled]);
  return <ChildComponent config={config} />;
}
```

### 4.3 Key Prop Issues

**Search:**
```
Grep: key=\{index\}|key=\{i\}|\.map\([^)]*,\s*i\)|\.map\([^)]*,\s*index\)
Glob: **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Stable unique keys (not array index)
- [ ] Keys on list items

**Slow Pattern:**
```tsx
// BAD: Index as key causes unnecessary re-renders
{items.map((item, index) => (
  <ListItem key={index} item={item} />
))}
```

**Optimized Pattern:**
```tsx
// GOOD: Stable unique key
{items.map(item => (
  <ListItem key={item.id} item={item} />
))}
```

---

## 5. Vue Performance Issues

### 5.1 v-for Without Key

**Search:**
```
Grep: v-for=(?!.*:key)
Glob: **/*.vue
```

**Slow Pattern:**
```vue
<!-- BAD: No key -->
<li v-for="item in items">{{ item.name }}</li>
```

**Optimized Pattern:**
```vue
<!-- GOOD: Unique key -->
<li v-for="item in items" :key="item.id">{{ item.name }}</li>
```

### 5.2 v-if with v-for

**Search:**
```
Grep: v-for=.*v-if=|v-if=.*v-for=
Glob: **/*.vue
```

**Slow Pattern:**
```vue
<!-- BAD: v-if evaluated for each item -->
<li v-for="item in items" v-if="item.active" :key="item.id">
  {{ item.name }}
</li>
```

**Optimized Pattern:**
```vue
<!-- GOOD: Computed property for filtering -->
<li v-for="item in activeItems" :key="item.id">
  {{ item.name }}
</li>

<script>
computed: {
  activeItems() {
    return this.items.filter(item => item.active);
  }
}
</script>
```

---

## 6. Angular Performance Issues

### 6.1 Missing TrackBy

**Search:**
```
Grep: \*ngFor=(?!.*trackBy)
Glob: **/*.html, **/*.component.ts
```

**Slow Pattern:**
```html
<!-- BAD: No trackBy, entire list re-renders -->
<li *ngFor="let item of items">{{ item.name }}</li>
```

**Optimized Pattern:**
```html
<!-- GOOD: TrackBy for efficient updates -->
<li *ngFor="let item of items; trackBy: trackById">{{ item.name }}</li>

<!-- In component -->
trackById(index: number, item: Item): string {
  return item.id;
}
```

### 6.2 Missing OnPush

**Search:**
```
Grep: @Component\(\{(?![\s\S]*changeDetection)
Glob: **/*.component.ts
```

**Slow Pattern:**
```typescript
// BAD: Default change detection (checks on every cycle)
@Component({
  selector: 'app-list',
  templateUrl: './list.component.html'
})
export class ListComponent { }
```

**Optimized Pattern:**
```typescript
// GOOD: OnPush change detection
@Component({
  selector: 'app-list',
  templateUrl: './list.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ListComponent { }
```

---

## 7. Layout Thrashing

### 7.1 Forced Reflows

**Search:**
```
Grep: offsetWidth|offsetHeight|clientWidth|clientHeight|scrollTop|scrollLeft|getComputedStyle|getBoundingClientRect
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] Layout reads/writes interleaved
- [ ] Measurements in loops
- [ ] Style changes followed by reads

**Thrashing Pattern:**
```javascript
// BAD: Interleaved reads and writes
for (const el of elements) {
  const width = el.offsetWidth;  // Read (forces layout)
  el.style.width = width + 10 + 'px';  // Write
  // Next read forces another layout!
}
```

**Optimized Pattern:**
```javascript
// GOOD: Batch reads, then batch writes
const widths = elements.map(el => el.offsetWidth);  // All reads

elements.forEach((el, i) => {
  el.style.width = widths[i] + 10 + 'px';  // All writes
});

// OR: Use requestAnimationFrame
function updateLayout() {
  const width = element.offsetWidth;  // Read

  requestAnimationFrame(() => {
    element.style.width = width + 10 + 'px';  // Write in next frame
  });
}
```

### 7.2 Animation Performance

**Search:**
```
Grep: @keyframes|animation:|transition:|transform:|will-change
Glob: **/*.css, **/*.scss, **/*.less
```

**Check for:**
- [ ] Animations use transform/opacity
- [ ] will-change used sparingly
- [ ] No layout properties animated

**Slow Pattern:**
```css
/* BAD: Animating layout properties */
.animate {
  animation: slide 0.3s;
}
@keyframes slide {
  from { left: 0; width: 100px; }
  to { left: 100px; width: 200px; }
}
```

**Optimized Pattern:**
```css
/* GOOD: Using transform (GPU accelerated) */
.animate {
  animation: slide 0.3s;
  will-change: transform;  /* Use sparingly */
}
@keyframes slide {
  from { transform: translateX(0) scaleX(1); }
  to { transform: translateX(100px) scaleX(2); }
}
```

---

## 8. Web Vitals Issues

### 8.1 LCP (Largest Contentful Paint)

**Search:**
```
Grep: <img.*hero|<video.*hero|<h1|loading=["']lazy["'].*hero
Glob: **/*.html, **/*.tsx, **/*.jsx, **/*.vue
```

**Check for:**
- [ ] Hero image preloaded
- [ ] LCP element not lazy loaded
- [ ] Critical fonts preloaded
- [ ] Server-side rendering for above-fold content

**LCP Issues Pattern:**
```html
<!-- BAD: Hero lazy loaded -->
<img class="hero" src="/hero.jpg" loading="lazy" alt="Hero">

<!-- BAD: No preload for hero -->
<head>
  <!-- Missing preload -->
</head>
```

**Optimized Pattern:**
```html
<!-- GOOD: Hero preloaded and prioritized -->
<head>
  <link rel="preload" as="image" href="/hero.webp" imagesrcset="..." imagesizes="...">
  <link rel="preload" as="font" href="/fonts/main.woff2" crossorigin>
</head>
<body>
  <img class="hero" src="/hero.jpg" fetchpriority="high" alt="Hero">
</body>
```

### 8.2 CLS (Cumulative Layout Shift)

**Search:**
```
Grep: <img(?!.*width=.*height=)|<video(?!.*width=.*height=)|<iframe(?!.*width=.*height=)
Glob: **/*.html, **/*.tsx, **/*.jsx, **/*.vue
```

**Check for:**
- [ ] Images have width/height or aspect-ratio
- [ ] Fonts have font-display: swap with fallback
- [ ] Dynamic content has reserved space
- [ ] Ads/embeds have dimensions

**CLS Issues Pattern:**
```html
<!-- BAD: No dimensions, causes layout shift -->
<img src="/photo.jpg" alt="Photo">

<!-- BAD: No space reserved for dynamic content -->
<div id="dynamic-banner"></div>
```

**Optimized Pattern:**
```html
<!-- GOOD: Dimensions prevent layout shift -->
<img src="/photo.jpg" alt="Photo" width="800" height="600">

<!-- OR: Aspect ratio -->
<img src="/photo.jpg" alt="Photo" style="aspect-ratio: 4/3; width: 100%;">

<!-- GOOD: Reserved space for dynamic content -->
<div id="dynamic-banner" style="min-height: 250px;"></div>
```

### 8.3 FID/INP (First Input Delay / Interaction to Next Paint)

**Search:**
```
Grep: addEventListener\(['"]click|onClick=|@click=|\.on\(['"]click
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx, **/*.vue
```

**Check for:**
- [ ] Long tasks broken up
- [ ] Heavy computation offloaded to Web Workers
- [ ] Event handlers are lightweight

**FID/INP Issues Pattern:**
```javascript
// BAD: Long synchronous task on click
button.addEventListener('click', () => {
  for (let i = 0; i < 1000000; i++) {
    // Expensive computation
  }
  updateUI();
});
```

**Optimized Pattern:**
```javascript
// GOOD: Async processing or Web Worker
button.addEventListener('click', async () => {
  // Show loading state immediately
  showLoading();

  // Defer heavy work
  await new Promise(resolve => setTimeout(resolve, 0));
  const result = await computeInWorker(data);

  updateUI(result);
});

// OR: Use requestIdleCallback
button.addEventListener('click', () => {
  showLoading();
  requestIdleCallback(() => {
    heavyComputation();
    updateUI();
  });
});
```

---

## 9. Third-Party Script Impact

### 9.1 Blocking Third-Party Scripts

**Search:**
```
Grep: <script.*src=["'][^"']*(?:analytics|tracking|widget|chat|ads|pixel)
Glob: **/*.html, **/index.html
```

**Check for:**
- [ ] Third-party scripts loaded async
- [ ] Non-critical scripts deferred
- [ ] Resource hints for third-party origins

**Blocking Pattern:**
```html
<!-- BAD: Blocking third-party scripts -->
<script src="https://analytics.example.com/script.js"></script>
<script src="https://widget.example.com/chat.js"></script>
```

**Optimized Pattern:**
```html
<!-- GOOD: Async loading with resource hints -->
<head>
  <link rel="preconnect" href="https://analytics.example.com">
  <link rel="dns-prefetch" href="https://widget.example.com">
</head>
<body>
  <script src="https://analytics.example.com/script.js" async></script>
  <script>
    // Lazy load chat widget after page load
    window.addEventListener('load', () => {
      const script = document.createElement('script');
      script.src = 'https://widget.example.com/chat.js';
      document.body.appendChild(script);
    });
  </script>
</body>
```

---

## 10. Missing Code Splitting

### 10.1 Route-Based Splitting

**Search:**
```
Grep: React\.lazy|import\(\)|loadChildren|component:\s*\(\)\s*=>|defineAsyncComponent
Glob: **/*.tsx, **/*.jsx, **/*.ts, **/*.vue
```

**Check for:**
- [ ] Route components lazy loaded
- [ ] Large libraries dynamically imported
- [ ] Feature modules code split

**No Splitting Pattern:**
```tsx
// BAD: All routes in main bundle
import Home from './pages/Home';
import Dashboard from './pages/Dashboard';
import Settings from './pages/Settings';
import Admin from './pages/Admin';

const routes = [
  { path: '/', component: Home },
  { path: '/dashboard', component: Dashboard },
  { path: '/settings', component: Settings },
  { path: '/admin', component: Admin },
];
```

**Optimized Pattern:**
```tsx
// GOOD: Lazy loaded routes
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const Admin = lazy(() => import('./pages/Admin'));

const routes = [
  { path: '/', element: <Suspense fallback={<Loading />}><Home /></Suspense> },
  { path: '/dashboard', element: <Suspense fallback={<Loading />}><Dashboard /></Suspense> },
  // ...
];
```

**Next.js Pattern:**
```tsx
// GOOD: Dynamic imports in Next.js
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <Loading />,
  ssr: false  // If not needed for SSR
});
```

---

## 11. Impact Classification

### Critical
- Main bundle > 500KB (uncompressed)
- No code splitting configured
- Render-blocking resources in head
- Hero image lazy loaded (LCP killer)
- Layout properties animated

### High
- Large library imports (moment, lodash full)
- Missing image optimization
- No responsive images for hero
- Missing React.memo on list items
- v-for without key in Vue

### Medium
- Inline style objects in React
- Missing lazy loading for below-fold images
- No preconnect for third-party origins
- Missing trackBy in Angular
- Images without dimensions (CLS)

### Low
- Minor bundle optimizations
- Optional memoization opportunities
- Style/animation preferences

---

## 12. Output Format

For each finding:

```
FINDING: [PERF04] Title
IMPACT: Critical|High|Medium|Low
EFFORT: Low|Medium|High
FILE: path/to/file.tsx:lineNumber
DESCRIPTION: What the frontend performance issue is and user impact
SLOW_CODE:
```tsx
// Current implementation
```
OPTIMIZED_CODE:
```tsx
// Recommended optimized implementation
```
ESTIMATED_GAIN: "Reduces LCP by ~2s, bundle size by 50KB"
BENCHMARK: "Measure with: Lighthouse, WebPageTest, or Chrome DevTools Performance tab"
```
