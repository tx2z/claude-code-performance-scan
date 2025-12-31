---
description: "[Internal] Report template - use /performance-scan instead"
disable-model-invocation: true
---

# Performance Scan Report

**Date:** {{DATE}}
**Scope:** {{SCOPE}}
**Project:** {{PROJECT_NAME}}
**Tech Stack:** {{TECH_STACK}}

---

## Performance Score

### Overall Score: {{OVERALL_SCORE}}/100

| Category | Score | Status |
|----------|-------|--------|
| Algorithm Complexity (PERF01) | {{PERF01_SCORE}}/100 | {{PERF01_STATUS}} |
| Memory Management (PERF02) | {{PERF02_SCORE}}/100 | {{PERF02_STATUS}} |
| Database Performance (PERF03) | {{PERF03_SCORE}}/100 | {{PERF03_STATUS}} |
| Frontend Performance (PERF04) | {{PERF04_SCORE}}/100 | {{PERF04_STATUS}} |
| Network Efficiency (PERF05) | {{PERF05_SCORE}}/100 | {{PERF05_STATUS}} |
| Async Operations (PERF06) | {{PERF06_SCORE}}/100 | {{PERF06_STATUS}} |
| Build/Bundle (PERF07) | {{PERF07_SCORE}}/100 | {{PERF07_STATUS}} |

### Score Breakdown

- **90-100**: Excellent - Minor optimizations possible
- **70-89**: Good - Some improvements recommended
- **50-69**: Fair - Noticeable performance issues
- **30-49**: Poor - Significant optimization needed
- **0-29**: Critical - Severe performance problems

---

## Executive Summary

| Severity | Count | Estimated Impact |
|----------|-------|------------------|
| Critical | {{CRITICAL_COUNT}} | {{CRITICAL_IMPACT}} |
| High | {{HIGH_COUNT}} | {{HIGH_IMPACT}} |
| Medium | {{MEDIUM_COUNT}} | {{MEDIUM_IMPACT}} |
| Low | {{LOW_COUNT}} | {{LOW_IMPACT}} |

**Total Findings:** {{TOTAL_COUNT}}

### Key Metrics Affected

| Metric | Current | Potential | Improvement |
|--------|---------|-----------|-------------|
| Page Load Time | {{CURRENT_LOAD}} | {{POTENTIAL_LOAD}} | {{LOAD_IMPROVEMENT}} |
| Bundle Size | {{CURRENT_BUNDLE}} | {{POTENTIAL_BUNDLE}} | {{BUNDLE_IMPROVEMENT}} |
| API Response Time | {{CURRENT_API}} | {{POTENTIAL_API}} | {{API_IMPROVEMENT}} |
| Memory Usage | {{CURRENT_MEMORY}} | {{POTENTIAL_MEMORY}} | {{MEMORY_IMPROVEMENT}} |

### Summary

{{EXECUTIVE_SUMMARY}}

---

## Critical Bottlenecks

> **Immediate Action Required:** These issues cause severe performance degradation and should be addressed urgently.

{{#CRITICAL_FINDINGS}}

### CRIT-{{INDEX}}: [{{PERF_ID}}] {{TITLE}}

| Attribute | Value |
|-----------|-------|
| **Impact** | Critical |
| **Effort** | {{EFFORT}} |
| **Category** | {{CATEGORY_NAME}} |
| **File** | `{{FILE_PATH}}` |
| **Line** | {{LINE_NUMBER}} |

**Description:**
{{DESCRIPTION}}

**Performance Impact:**
{{IMPACT_DESCRIPTION}}

**Current Implementation:**
```{{LANGUAGE}}
{{SLOW_CODE}}
```

**Optimized Implementation:**
```{{LANGUAGE}}
{{OPTIMIZED_CODE}}
```

**Estimated Gain:**
{{ESTIMATED_GAIN}}

**How to Benchmark:**
{{BENCHMARK}}

---

{{/CRITICAL_FINDINGS}}

## Quick Wins

> **Low Effort, High Impact:** These optimizations can be implemented quickly with significant returns.

{{#QUICK_WINS}}

### QW-{{INDEX}}: [{{PERF_ID}}] {{TITLE}}

| Attribute | Value |
|-----------|-------|
| **Impact** | {{IMPACT_LEVEL}} |
| **Effort** | Low |
| **Category** | {{CATEGORY_NAME}} |
| **File** | `{{FILE_PATH}}:{{LINE_NUMBER}}` |

**Description:**
{{DESCRIPTION}}

**Current Implementation:**
```{{LANGUAGE}}
{{SLOW_CODE}}
```

**Optimized Implementation:**
```{{LANGUAGE}}
{{OPTIMIZED_CODE}}
```

**Estimated Gain:** {{ESTIMATED_GAIN}}

---

{{/QUICK_WINS}}

## High Impact Findings

> **Priority:** Address these findings in the current sprint for noticeable improvements.

{{#HIGH_FINDINGS}}

### HIGH-{{INDEX}}: [{{PERF_ID}}] {{TITLE}}

| Attribute | Value |
|-----------|-------|
| **Impact** | High |
| **Effort** | {{EFFORT}} |
| **Category** | {{CATEGORY_NAME}} |
| **File** | `{{FILE_PATH}}` |
| **Line** | {{LINE_NUMBER}} |

**Description:**
{{DESCRIPTION}}

**Performance Impact:**
{{IMPACT_DESCRIPTION}}

**Current Implementation:**
```{{LANGUAGE}}
{{SLOW_CODE}}
```

**Optimized Implementation:**
```{{LANGUAGE}}
{{OPTIMIZED_CODE}}
```

**Estimated Gain:** {{ESTIMATED_GAIN}}

---

{{/HIGH_FINDINGS}}

## Medium Priority Findings

> **Timeline:** Plan to address within 30 days.

{{#MEDIUM_FINDINGS}}

### MED-{{INDEX}}: [{{PERF_ID}}] {{TITLE}}

- **Category:** {{CATEGORY_NAME}}
- **File:** `{{FILE_PATH}}:{{LINE_NUMBER}}`
- **Effort:** {{EFFORT}}
- **Description:** {{DESCRIPTION}}
- **Estimated Gain:** {{ESTIMATED_GAIN}}

{{/MEDIUM_FINDINGS}}

---

## Low Priority Findings

> **Timeline:** Address when convenient or during related work.

{{#LOW_FINDINGS}}

- **[{{PERF_ID}}]** {{TITLE}} - `{{FILE_PATH}}:{{LINE_NUMBER}}`
  - {{DESCRIPTION}}
  - Estimated Gain: {{ESTIMATED_GAIN}}

{{/LOW_FINDINGS}}

---

## Before/After Comparisons

### Algorithm Improvements

| Pattern | Before | After | Improvement |
|---------|--------|-------|-------------|
{{#ALGORITHM_COMPARISONS}}
| {{PATTERN_NAME}} | {{BEFORE_COMPLEXITY}} | {{AFTER_COMPLEXITY}} | {{IMPROVEMENT}} |
{{/ALGORITHM_COMPARISONS}}

### Bundle Size Impact

| Change | Size Reduction | Load Time Impact |
|--------|----------------|------------------|
{{#BUNDLE_COMPARISONS}}
| {{CHANGE_NAME}} | {{SIZE_REDUCTION}} | {{LOAD_IMPACT}} |
{{/BUNDLE_COMPARISONS}}

### Database Query Improvements

| Query Type | Before | After | Reduction |
|------------|--------|-------|-----------|
{{#DATABASE_COMPARISONS}}
| {{QUERY_TYPE}} | {{BEFORE_QUERIES}} | {{AFTER_QUERIES}} | {{REDUCTION}} |
{{/DATABASE_COMPARISONS}}

---

## Benchmark Suggestions

### Frontend Benchmarks

1. **Lighthouse Audit**
   ```bash
   npx lighthouse {{URL}} --output=html --output-path=./lighthouse-report.html
   ```

2. **Bundle Analysis**
   ```bash
   npm run build -- --analyze
   # OR
   npx source-map-explorer dist/**/*.js
   ```

3. **Web Vitals Measurement**
   ```javascript
   import { onCLS, onFID, onLCP } from 'web-vitals';

   onCLS(console.log);
   onFID(console.log);
   onLCP(console.log);
   ```

### Backend Benchmarks

1. **API Load Testing**
   ```bash
   # Using autocannon
   npx autocannon -c 100 -d 30 {{API_URL}}

   # Using wrk
   wrk -t12 -c400 -d30s {{API_URL}}
   ```

2. **Database Query Profiling**
   ```sql
   -- PostgreSQL
   EXPLAIN ANALYZE {{QUERY}};

   -- MySQL
   EXPLAIN {{QUERY}};
   SET profiling = 1;
   {{QUERY}};
   SHOW PROFILES;
   ```

3. **Memory Profiling**
   ```javascript
   // Node.js
   const used = process.memoryUsage();
   console.log(`heapUsed: ${Math.round(used.heapUsed / 1024 / 1024 * 100) / 100} MB`);

   // With --inspect flag for Chrome DevTools
   node --inspect server.js
   ```

### Suggested Tools

| Purpose | Tool | Command |
|---------|------|---------|
| Bundle Size | webpack-bundle-analyzer | `npm run build:analyze` |
| Bundle Size | source-map-explorer | `npx source-map-explorer dist/*.js` |
| Web Vitals | web-vitals | Built-in library |
| Load Testing | autocannon | `npx autocannon URL` |
| Load Testing | k6 | `k6 run script.js` |
| Memory | clinic.js | `npx clinic doctor -- node server.js` |
| Profiling | 0x | `npx 0x server.js` |
| Database | EXPLAIN ANALYZE | SQL query prefix |

---

## Monitoring Recommendations

### Key Metrics to Track

1. **Web Vitals**
   - Largest Contentful Paint (LCP): < 2.5s
   - First Input Delay (FID): < 100ms
   - Cumulative Layout Shift (CLS): < 0.1
   - Interaction to Next Paint (INP): < 200ms

2. **Backend Metrics**
   - API response time (p50, p95, p99)
   - Database query time
   - Memory usage
   - CPU utilization
   - Error rates

3. **Infrastructure**
   - Request throughput
   - Connection pool usage
   - Cache hit rates
   - CDN hit rates

### Recommended Tools

| Category | Tool | Purpose |
|----------|------|---------|
| APM | Datadog, New Relic | Full-stack monitoring |
| RUM | SpeedCurve, Calibre | Real User Monitoring |
| Synthetic | WebPageTest, Lighthouse CI | Automated testing |
| Profiling | Clinic.js, 0x | Node.js profiling |
| Tracing | Jaeger, Zipkin | Distributed tracing |
| Logging | Grafana, Kibana | Log analysis |

### Alerting Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| API p95 latency | > 500ms | > 2s |
| Error rate | > 1% | > 5% |
| Memory usage | > 70% | > 90% |
| CPU usage | > 70% | > 90% |
| LCP | > 2.5s | > 4s |

---

## Recommendations Summary

### Immediate Actions (Critical)

1. {{IMMEDIATE_1}}
2. {{IMMEDIATE_2}}
3. {{IMMEDIATE_3}}

### This Sprint (High Priority)

1. {{SPRINT_1}}
2. {{SPRINT_2}}
3. {{SPRINT_3}}

### Short-Term (30 Days)

1. {{SHORTTERM_1}}
2. {{SHORTTERM_2}}
3. {{SHORTTERM_3}}

### Long-Term (Roadmap)

1. {{LONGTERM_1}}
2. {{LONGTERM_2}}
3. {{LONGTERM_3}}

---

## Appendix

### A. Performance Categories Reference

| ID | Name | Focus Areas |
|----|------|-------------|
| PERF01 | Algorithm Complexity | Time complexity, loops, memoization |
| PERF02 | Memory | Leaks, caches, closures, cleanup |
| PERF03 | Database | N+1, indexes, queries, pooling |
| PERF04 | Frontend | Rendering, bundles, Web Vitals |
| PERF05 | Network | Compression, caching, batching |
| PERF06 | Async Operations | Parallelism, debounce, batching |
| PERF07 | Build/Bundle | Dependencies, tree shaking, minification |

### B. Impact Levels

| Level | Description | User Impact |
|-------|-------------|-------------|
| Critical | Severe degradation | Unusable/very slow |
| High | Significant slowdown | Noticeable lag |
| Medium | Measurable impact | Slightly slower |
| Low | Minor optimization | Marginal improvement |

### C. Effort Levels

| Level | Time Estimate | Scope |
|-------|---------------|-------|
| Low | < 1 hour | Simple code change |
| Medium | 1-4 hours | Moderate refactoring |
| High | > 4 hours | Significant redesign |

### D. Scan Metadata

- **Scanner:** Claude Code Performance Scanner
- **Version:** 1.0.0
- **Scan Duration:** {{DURATION}}
- **Files Analyzed:** {{FILES_COUNT}}

### E. Files Scanned

**Source Code ({{SOURCE_FILES_COUNT}} files):**
- Controllers, Services, Components, Utilities

**Configuration ({{CONFIG_FILES_COUNT}} files):**
- package.json, webpack/vite config, tsconfig

**Static Assets ({{ASSET_FILES_COUNT}} files):**
- Images, fonts, stylesheets

---

**Report Generated by Claude Code Performance Scanner**

*For questions or to re-run specific categories, use `/performance-scan [scope]`*
