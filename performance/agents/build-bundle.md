---
description: "[Internal] Build/bundle agent - use /performance-scan instead"
disable-model-invocation: true
allowed-tools: Read, Glob, Grep
---

# Build/Bundle Agent (PERF07)

Scan for performance issues related to:
- **Large Dependencies** - Oversized npm packages
- **Duplicate Dependencies** - Same package multiple versions
- **Tree Shaking Issues** - Dead code not eliminated
- **Missing Minification** - Unminified production code
- **Unoptimized Assets** - Images, fonts not optimized
- **Source Maps in Production** - Debugging code shipped
- **Development Code in Prod** - Debug/dev code not stripped

---

## 1. Large Dependencies

### 1.1 Known Large Packages

**Search:**
```
Grep: "moment"|"lodash"|"@material-ui/core"|"@mui/material"|"antd"|"rxjs"|"@angular/|"three"|"pdfjs"|"xlsx"|"socket.io-client"
Glob: **/package.json
```

**Check for heavy packages and lighter alternatives:**

| Heavy Package | Size (approx) | Alternative | Size (approx) |
|---------------|---------------|-------------|---------------|
| moment | 66KB+ | date-fns, dayjs | 2-12KB |
| lodash (full) | 70KB+ | lodash-es (tree-shakeable) | varies |
| @mui/material (full) | 500KB+ | Individual imports | varies |
| antd (full) | 1MB+ | Individual imports | varies |
| rxjs (full) | 150KB+ | Individual operators | varies |
| socket.io-client | 50KB+ | ws (native) | 0KB |
| axios | 13KB | fetch (native) | 0KB |

**Large Import Pattern:**
```typescript
// BAD: Importing entire library
import moment from 'moment';
import _ from 'lodash';
import { Button, TextField, Dialog, Table, Menu } from '@mui/material';
```

**Optimized Pattern:**
```typescript
// GOOD: Tree-shakeable imports
import { format, parseISO } from 'date-fns';  // Only what's needed
import map from 'lodash/map';  // Individual functions
import Button from '@mui/material/Button';  // Direct imports

// BETTER: Native alternatives
const date = new Intl.DateTimeFormat('en-US').format(new Date());
const items = array.map(x => x.name);  // Native array methods
await fetch('/api/data');  // Native fetch instead of axios
```

### 1.2 Bundle Analysis

**Search:**
```
Glob: **/webpack.config.*, **/vite.config.*, **/next.config.*
Grep: BundleAnalyzerPlugin|rollup-plugin-visualizer|source-map-explorer|analyze
```

**Check for:**
- [ ] Bundle analyzer configured
- [ ] Regular bundle size monitoring
- [ ] Size budgets defined

**Missing Analysis Pattern:**
```javascript
// BAD: No bundle analysis
module.exports = {
  // ... config without analyzer
};
```

**Optimized Pattern:**
```javascript
// GOOD: Bundle analyzer for development
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  plugins: [
    process.env.ANALYZE && new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      reportFilename: 'bundle-report.html'
    })
  ].filter(Boolean)
};

// GOOD: Size limits
module.exports = {
  performance: {
    maxEntrypointSize: 250000,  // 250KB
    maxAssetSize: 250000,
    hints: 'error'
  }
};
```

### 1.3 Dependency Size Check

**Search:**
```
Glob: **/package.json, **/package-lock.json, **/yarn.lock, **/pnpm-lock.yaml
```

**Check package.json for:**
- [ ] Dependencies needed for production only in dependencies
- [ ] Build tools in devDependencies
- [ ] No test libraries in dependencies

**Mixed Dependencies:**
```json
{
  "dependencies": {
    "react": "^18.2.0",
    "jest": "^29.0.0",           // BAD: Should be devDependency
    "@types/node": "^20.0.0",   // BAD: Should be devDependency
    "typescript": "^5.0.0"      // BAD: Should be devDependency
  }
}
```

**Optimized Pattern:**
```json
{
  "dependencies": {
    "react": "^18.2.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## 2. Duplicate Dependencies

### 2.1 Multiple Versions

**Search:**
```
Glob: **/package-lock.json, **/yarn.lock, **/pnpm-lock.yaml
```

**Check for multiple versions of:**
- [ ] Core libraries (react, lodash, etc.)
- [ ] Transitive dependencies
- [ ] Peer dependency conflicts

**Duplicate Pattern (package-lock.json):**
```json
{
  "packages": {
    "node_modules/lodash": { "version": "4.17.21" },
    "node_modules/some-lib/node_modules/lodash": { "version": "4.17.15" }
  }
}
```

**Resolution:**
```json
// package.json - force single version
{
  "resolutions": {
    "lodash": "4.17.21"
  },
  // OR for npm
  "overrides": {
    "lodash": "4.17.21"
  }
}
```

### 2.2 Deduplication Check

**Run npm/pnpm/yarn dedupe:**
```bash
# Check for duplicates
npm ls lodash
npm dedupe --dry-run

# pnpm
pnpm dedupe --check

# yarn
yarn dedupe --check
```

---

## 3. Tree Shaking Issues

### 3.1 CommonJS Imports

**Search:**
```
Grep: require\(['"]|module\.exports|exports\.|import\s+.*\s+from\s+['"][^'"]+(?<!\.mjs)['"]
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] ESM imports for tree shaking
- [ ] No require() in frontend code
- [ ] Package.json has "sideEffects": false

**Non-tree-shakeable Pattern:**
```typescript
// BAD: CommonJS (not tree-shakeable)
const _ = require('lodash');
const { map, filter } = require('lodash');

// BAD: Full namespace import
import * as _ from 'lodash';
```

**Optimized Pattern:**
```typescript
// GOOD: Named ESM imports (tree-shakeable)
import { map, filter } from 'lodash-es';

// GOOD: Direct file imports
import map from 'lodash/map';
```

### 3.2 Side Effects Configuration

**Search:**
```
Glob: **/package.json
Grep: sideEffects
```

**Check for:**
- [ ] "sideEffects": false for pure libraries
- [ ] CSS files listed in sideEffects

**Missing sideEffects:**
```json
{
  "name": "my-library",
  "main": "dist/index.js"
  // Missing sideEffects declaration
}
```

**Optimized Pattern:**
```json
{
  "name": "my-library",
  "main": "dist/index.js",
  "module": "dist/index.esm.js",
  "sideEffects": false
}

// OR with exceptions
{
  "sideEffects": [
    "*.css",
    "*.scss"
  ]
}
```

### 3.3 Barrel Files

**Search:**
```
Glob: **/index.ts, **/index.js
Grep: export\s+\*\s+from|export\s+\{[^}]+\}\s+from
```

**Check for:**
- [ ] Large barrel files causing import of unused code
- [ ] Re-exports that defeat tree shaking

**Barrel Problem:**
```typescript
// src/components/index.ts - BAD: Barrel file
export * from './Button';
export * from './Table';  // Heavy component
export * from './Chart';  // Heavy component
export * from './Calendar'; // Heavy component

// Usage - imports everything even if only Button needed
import { Button } from '@/components';
```

**Optimized Pattern:**
```typescript
// GOOD: Direct imports
import { Button } from '@/components/Button';

// OR: Smaller barrel files per category
import { Button } from '@/components/forms';
import { Table } from '@/components/data';
```

---

## 4. Minification

### 4.1 Production Minification

**Search:**
```
Glob: **/webpack.config.*, **/vite.config.*, **/rollup.config.*
Grep: TerserPlugin|esbuild|swc|minify|uglify|minimize
```

**Check for:**
- [ ] Minification enabled in production
- [ ] Modern minifier (esbuild, swc, terser)
- [ ] CSS minification

**Missing Minification:**
```javascript
// BAD: No minification config
module.exports = {
  mode: 'production',
  optimization: {
    minimize: false  // BAD!
  }
};
```

**Optimized Pattern:**
```javascript
// GOOD: Terser with modern config
const TerserPlugin = require('terser-webpack-plugin');

module.exports = {
  mode: 'production',
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,  // Remove console.log
            drop_debugger: true,
            pure_funcs: ['console.log', 'console.info']
          },
          mangle: true,
          format: {
            comments: false
          }
        },
        extractComments: false
      })
    ]
  }
};

// GOOD: Vite (uses esbuild by default)
// vite.config.ts
export default defineConfig({
  build: {
    minify: 'esbuild',  // or 'terser' for more control
    target: 'es2020'
  }
});
```

### 4.2 CSS Minification

**Search:**
```
Grep: CssMinimizerPlugin|cssnano|postcss|purgecss|MiniCssExtractPlugin
Glob: **/webpack.config.*, **/postcss.config.*
```

**Missing CSS Minification:**
```javascript
// BAD: No CSS optimization
module.exports = {
  // No CSS minimizer
};
```

**Optimized Pattern:**
```javascript
// GOOD: CSS minification
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

module.exports = {
  optimization: {
    minimizer: [
      '...',  // Keep JS minimizer
      new CssMinimizerPlugin()
    ]
  }
};

// postcss.config.js with cssnano
module.exports = {
  plugins: [
    require('cssnano')({
      preset: 'default'
    })
  ]
};
```

---

## 5. Unused Code Removal

### 5.1 PurgeCSS / Unused CSS

**Search:**
```
Grep: purgecss|@fullhuman/postcss-purgecss|uncss|content:.*glob
Glob: **/postcss.config.*, **/tailwind.config.*
```

**Check for:**
- [ ] Unused CSS removal configured
- [ ] Content paths correctly specified
- [ ] Safelist for dynamic classes

**Missing PurgeCSS:**
```javascript
// BAD: No CSS purging (Tailwind ships full CSS)
module.exports = {
  content: [],  // Empty or missing
  theme: { /* ... */ }
};
```

**Optimized Pattern:**
```javascript
// GOOD: Tailwind with content paths
module.exports = {
  content: [
    './src/**/*.{js,jsx,ts,tsx}',
    './public/index.html'
  ],
  safelist: [
    'bg-red-500',  // Dynamic classes
    /^bg-/  // Regex patterns
  ]
};

// GOOD: PurgeCSS with PostCSS
const purgecss = require('@fullhuman/postcss-purgecss');

module.exports = {
  plugins: [
    purgecss({
      content: ['./src/**/*.{js,jsx,ts,tsx}', './public/**/*.html'],
      defaultExtractor: content => content.match(/[\w-/:]+(?<!:)/g) || []
    })
  ]
};
```

### 5.2 Dead Code Detection

**Search:**
```
Grep: PURE|#__PURE__|sideEffects|unused|dead
```

**Check for:**
- [ ] /*#__PURE__*/ annotations on pure function calls
- [ ] Unused exports detected

---

## 6. Source Maps

### 6.1 Source Maps in Production

**Search:**
```
Grep: devtool.*source-map|sourceMap.*true|sourcemap|\.map$
Glob: **/webpack.config.*, **/vite.config.*, **/tsconfig.json, **/dist/**/*.map
```

**Check for:**
- [ ] Full source maps not shipped to production
- [ ] Hidden source maps for error tracking
- [ ] Source maps only accessible internally

**Exposed Source Maps:**
```javascript
// BAD: Full source maps in production
module.exports = {
  mode: 'production',
  devtool: 'source-map'  // Creates accessible .map files
};
```

**Optimized Pattern:**
```javascript
// GOOD: Hidden source maps (for error tracking)
module.exports = {
  mode: 'production',
  devtool: 'hidden-source-map'  // .map files not referenced in bundle
};

// GOOD: No source maps in production
module.exports = {
  mode: 'production',
  devtool: false
};

// GOOD: Different config per environment
module.exports = (env, argv) => ({
  devtool: argv.mode === 'development' ? 'eval-source-map' : 'hidden-source-map'
});
```

### 6.2 Source Map Upload

**Check for error tracking setup:**
- [ ] Sentry/Datadog source map upload
- [ ] Source maps not publicly accessible
- [ ] Upload on deploy, remove from server

---

## 7. Development Code in Production

### 7.1 Console Statements

**Search:**
```
Grep: console\.(log|debug|info|warn|trace)|debugger
Glob: **/*.ts, **/*.js, **/*.tsx, **/*.jsx
```

**Check for:**
- [ ] console.log removed in production
- [ ] Debugger statements removed
- [ ] Development-only code stripped

**Development Code Pattern:**
```typescript
// BAD: Debug code in production
function processOrder(order: Order) {
  console.log('Processing order:', order);  // Debug log
  debugger;  // Debugger statement!

  if (process.env.NODE_ENV === 'development') {
    console.log('Development mode');
  }

  // ... implementation
}
```

**Optimized Pattern:**
```typescript
// GOOD: Stripped in production build (Terser config)
// webpack.config.js or terser config
{
  terserOptions: {
    compress: {
      drop_console: true,
      drop_debugger: true
    }
  }
}

// GOOD: Conditional logging
const logger = {
  debug: process.env.NODE_ENV === 'development'
    ? console.log.bind(console)
    : () => {}
};

logger.debug('Processing order:', order);
```

### 7.2 Development Dependencies in Bundle

**Search:**
```
Grep: __DEV__|process\.env\.NODE_ENV|import.*devtools|DevTools
Glob: **/*.ts, **/*.tsx, **/*.js, **/*.jsx
```

**Check for:**
- [ ] React DevTools not in production
- [ ] Development tools tree-shaken
- [ ] __DEV__ checks present

**Dev Tools in Production:**
```typescript
// BAD: DevTools always imported
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

function App() {
  return (
    <>
      <MainApp />
      <ReactQueryDevtools />  {/* Ships in production! */}
    </>
  );
}
```

**Optimized Pattern:**
```typescript
// GOOD: Conditional import
import { lazy, Suspense } from 'react';

const ReactQueryDevtools = lazy(() =>
  import('@tanstack/react-query-devtools').then(mod => ({
    default: mod.ReactQueryDevtools
  }))
);

function App() {
  return (
    <>
      <MainApp />
      {process.env.NODE_ENV === 'development' && (
        <Suspense fallback={null}>
          <ReactQueryDevtools />
        </Suspense>
      )}
    </>
  );
}

// BETTER: Don't import at all in production
// Webpack DefinePlugin replaces process.env.NODE_ENV
// Tree shaking removes the conditional branch
```

---

## 8. Asset Optimization

### 8.1 Image Optimization

**Search:**
```
Glob: **/*.{png,jpg,jpeg,gif,svg,webp}
Grep: ImageMinimizerPlugin|imagemin|sharp|svgo
```

**Check for:**
- [ ] Images optimized at build time
- [ ] SVG optimization (SVGO)
- [ ] Next-gen formats generated

**Missing Image Optimization:**
```javascript
// BAD: No image optimization
module.exports = {
  module: {
    rules: [
      {
        test: /\.(png|jpg|gif)$/,
        type: 'asset/resource'  // No optimization
      }
    ]
  }
};
```

**Optimized Pattern:**
```javascript
// GOOD: With image optimization
const ImageMinimizerPlugin = require('image-minimizer-webpack-plugin');

module.exports = {
  optimization: {
    minimizer: [
      new ImageMinimizerPlugin({
        minimizer: {
          implementation: ImageMinimizerPlugin.sharpMinify,
          options: {
            encodeOptions: {
              jpeg: { quality: 80 },
              webp: { quality: 80 },
              png: { compressionLevel: 9 }
            }
          }
        },
        generator: [
          {
            preset: 'webp',
            implementation: ImageMinimizerPlugin.sharpGenerate,
            options: {
              encodeOptions: { webp: { quality: 80 } }
            }
          }
        ]
      })
    ]
  }
};
```

### 8.2 Font Optimization

**Search:**
```
Glob: **/*.{woff,woff2,ttf,eot,otf}
Grep: font-display|preload.*font|@font-face
```

**Check for:**
- [ ] WOFF2 format used
- [ ] Font subsetting
- [ ] font-display: swap

**Unoptimized Fonts:**
```css
/* BAD: Full font file, blocking */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.ttf');
}
```

**Optimized Pattern:**
```css
/* GOOD: Optimized font loading */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2'),
       url('/fonts/custom.woff') format('woff');
  font-display: swap;
  unicode-range: U+0000-00FF;  /* Latin subset */
}
```

---

## 9. Code Splitting Configuration

### 9.1 Vendor Chunk

**Search:**
```
Grep: splitChunks|manualChunks|vendor|commons
Glob: **/webpack.config.*, **/vite.config.*, **/rollup.config.*
```

**Check for:**
- [ ] Vendor chunk separated
- [ ] Common chunks extracted
- [ ] Async chunks configured

**No Code Splitting:**
```javascript
// BAD: Single bundle
module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'bundle.js'
  }
};
```

**Optimized Pattern:**
```javascript
// GOOD: Proper splitting
module.exports = {
  entry: './src/index.js',
  output: {
    filename: '[name].[contenthash].js',
    chunkFilename: '[name].[contenthash].chunk.js'
  },
  optimization: {
    runtimeChunk: 'single',
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: 25,
      minSize: 20000,
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name(module) {
            const packageName = module.context.match(
              /[\\/]node_modules[\\/](.*?)([\\/]|$)/
            )[1];
            return `vendor.${packageName.replace('@', '')}`;
          }
        }
      }
    }
  }
};
```

### 9.2 Dynamic Imports

**Search:**
```
Grep: import\(|React\.lazy|defineAsyncComponent|loadChildren
Glob: **/*.ts, **/*.tsx, **/*.js, **/*.jsx, **/*.vue
```

**Check for:**
- [ ] Routes lazy loaded
- [ ] Heavy components dynamically imported
- [ ] Named chunks for debugging

**Static Imports:**
```typescript
// BAD: Static imports for all routes
import Home from './pages/Home';
import Dashboard from './pages/Dashboard';
import Admin from './pages/Admin';
import Analytics from './pages/Analytics';
```

**Optimized Pattern:**
```typescript
// GOOD: Dynamic imports with chunk names
const Home = lazy(() => import(/* webpackChunkName: "home" */ './pages/Home'));
const Dashboard = lazy(() => import(/* webpackChunkName: "dashboard" */ './pages/Dashboard'));
const Admin = lazy(() => import(/* webpackChunkName: "admin" */ './pages/Admin'));
const Analytics = lazy(() => import(/* webpackChunkName: "analytics" */ './pages/Analytics'));
```

---

## 10. Modern JavaScript

### 10.1 Browser Targets

**Search:**
```
Glob: **/.browserslistrc, **/package.json, **/tsconfig.json, **/babel.config.*
Grep: browserslist|targets|target.*es|browsers
```

**Check for:**
- [ ] Modern browser targets
- [ ] No excessive transpilation
- [ ] Differential serving (modern/legacy)

**Old Browser Targets:**
```
# .browserslistrc - BAD: Too broad
> 0.5%
last 2 versions
IE 11
```

**Optimized Pattern:**
```
# .browserslistrc - GOOD: Modern targets
> 0.5%
last 2 versions
not dead
not IE 11
```

```javascript
// GOOD: Differential serving
module.exports = {
  output: {
    filename: '[name].[contenthash].js'
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              ['@babel/preset-env', {
                targets: { esmodules: true },  // Modern only
                modules: false
              }]
            ]
          }
        }
      }
    ]
  }
};
```

---

## 11. Impact Classification

### Critical
- No minification in production
- Source maps publicly accessible
- Development dependencies in bundle (>100KB)
- No code splitting (bundle >500KB)

### High
- Large dependencies without alternatives
- Duplicate major dependencies
- CommonJS blocking tree shaking
- No CSS purging (Tailwind full: ~3MB)

### Medium
- Missing image optimization
- Full library imports
- Development console.logs in production
- Suboptimal browser targets

### Low
- Minor bundle optimizations
- Font optimization opportunities
- Build configuration tweaks

---

## 12. Output Format

For each finding:

```
FINDING: [PERF07] Title
IMPACT: Critical|High|Medium|Low
EFFORT: Low|Medium|High
FILE: path/to/file.ts:lineNumber OR package.json
DESCRIPTION: What the build/bundle issue is and impact
SLOW_CODE:
```typescript
// Current implementation or configuration
```
OPTIMIZED_CODE:
```typescript
// Recommended optimized implementation
```
ESTIMATED_GAIN: "Reduces bundle size by 200KB, improves load time by 1s"
BENCHMARK: "Measure with: webpack-bundle-analyzer, source-map-explorer, or Lighthouse"
```
