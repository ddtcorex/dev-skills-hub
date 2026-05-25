---
name: magento2-performance-audit
description: |
  Performs comprehensive performance and health audit for Magento 2 projects. Use when:
  - "Audit performance", "check Core Web Vitals", "run Lighthouse"
  - "Check server configuration", "verify Redis/Varnish setup"
  - "Analyze database queries", "find N+1 query issues"
  - "Review indexer configuration", "check cron health"

  This skill runs automated checks against Adobe Commerce Best Practices. DEPENDENT on magento2-dev-core
  for code-level performance patterns.
compatibility: claude, codex, opencode, copilot
depends: [magento2-dev-core]
metadata:
  audience: developers
  workflow: magento
---

# Magento 2 Performance Audit

This skill performs a comprehensive audit of Magento 2 performance, infrastructure, and code-level patterns.

## Audit Categories

### 1. Infrastructure Configuration

| Check | Expected | Command |
|-------|----------|---------|
| Application Mode | `production` | `bin/magento deploy:mode:show` |
| PHP OPcache | >= 256MB | Check php.ini |
| Redis (Session) | Enabled | Check `app/etc/env.php` |
| Redis (Cache) | Enabled | Check `app/etc/env.php` |
| Varnish | Enabled for FPC | Check `.madeto.env` or nginx config |

### 2. Cache Configuration

```bash
# Verify all caches enabled
govard sh -c "bin/magento cache:status"

# Expected output
# Category          Status    Enabled
# config            1         1
# layout            1         1
# block_html        1         1
# full_page         1         1
```

### 3. Indexer Configuration

```bash
# Check indexer mode (Update by Schedule is CRITICAL for performance)
govard sh -c "bin/magento indexer:status"

# Switch to schedule mode
govard sh -c "bin/magento indexer:set-mode schedule"

# Reindex all
govard sh -c "bin/magento indexer:reindex"
```

| Indexer | Recommended Mode |
|---------|------------------|
| catalog_product_price | Update on Save |
| catalog_url_category | Update by Schedule |
| catalog_category_product | Update by Schedule |
| inventory | Update by Schedule |
| targetrule | Update by Schedule |

### 4. Async Operations

```bash
# Enable async email
govard sh -c "bin/magento config:set system/smtp/async_disabled 0"

# Enable async operations for bulk APIs
govard sh -c "bin/magento queue:consumers:list"
```

### 5. Asset Optimization

```bash
# JS Bundling (recommended for production)
govard sh -c "bin/magento config:set dev/js/enable_js_bundling 1"
govard sh -c "bin/magento config:set dev/js/minify_files 1"

# CSS Minification
govard sh -c "bin/magento config:set dev/css/minify_files 1"
```

## Core Web Vitals Audit

### Lighthouse CI

```bash
# Install Lighthouse CI
npm install -g @lhci/cli

# Run audit
lhci autorun --collect.url=https://your-store.test \
             --collect.numberOfRuns=3 \
             --assert.preset=desktop
```

### Core Web Vitals Thresholds

| Metric | Target | Warning | Critical |
|--------|--------|---------|---------|
| LCP (Largest Contentful Paint) | < 2.5s | 2.5-4s | > 4s |
| INP (Interaction to Next Paint) | < 200ms | 200-500ms | > 500ms |
| CLS (Cumulative Layout Shift) | < 0.1 | 0.1-0.25 | > 0.25 |
| FCP (First Contentful Paint) | < 1.8s | 1.8-3s | > 3s |
| TTFB (Time to First Byte) | < 800ms | 800-1800ms | > 1800ms |

### Manual Testing

Open Chrome DevTools > Lighthouse:
1. Select "Navigation" mode
2. Select "Mobile" and "Desktop"
3. Check all categories
4. Review opportunities

## Database Query Profiling

### Profiler Setup

```bash
# Enable profiler
govard sh -c "bin/magento dev:query-log:enable"

# Visit pages to capture queries
# View in var/debug/*.sql or in admin (if enabled)
```

### Common Query Issues

| Issue | Pattern | Impact |
|-------|---------|--------|
| N+1 Query | `foreach` with `->load()` inside | High |
| Full Collection Load | `count($collection)` | Medium |
| Missing Index | `WHERE unindexed_column` | High |
| Expensive Join | Multiple JOINs on large tables | Medium |

### Query Analysis Commands

```bash
# Show slow queries (requires MySQL slow_query_log)
govard db query "SHOW FULL PROCESSLIST"

# Count queries on homepage (uncached)
govard sh -c "curl -s https://store.test/ | grep -o 'SELECT' | wc -l"

# Profile specific page
govard sh -c "bin/magento dev:profiler:enable && curl -s https://store.test/"
```

## Code-Level Performance Patterns

### N+1 Query Detection (From magento2-dev-core)

```php
// WRONG - N+1 query
foreach ($productIds as $id) {
    $product = $this->productFactory->create()->load($id); // Query per iteration
    $result[] = $product->getName();
}

// CORRECT - Batch load
$collection = $this->productCollection->create()
    ->addFieldToFilter('entity_id', ['in' => $productIds]);
foreach ($collection as $product) {
    $result[] = $product->getName();
}
```

### Collection Counting

```php
// WRONG - Loads all items
$count = count($this->collection->create()->getItems());

// CORRECT - Lightweight count query
$count = $this->collection->create()->getSize();
```

### Uncacheable Blocks

```xml
<!-- WRONG - Prevents FPC -->
<referenceBlock name="content" cacheable="false">
    <!-- This block will prevent full page caching -->
</referenceBlock>

<!-- CORRECT - Use esi:inline directive if needed -->
<referenceBlock name="dynamic.block" template="Magento_Cms::dynamic.phtml">
    <arguments>
        <argument name="cache_lifetime" xsi:type="number">3600</argument>
    </arguments>
</referenceBlock>
```

### Heavy Constructors

```php
// WRONG - Expensive operation in constructor
public function __construct(
    private readonly ExpensiveApiService $expensiveService
) {
    // This runs every time the class is instantiated
    $this->data = $this->expensiveService->fetchData();
}

// CORRECT - Lazy initialization via Proxy
public function __construct(
    private readonly ExpensiveApiServiceProxy $expensiveService
) {}

public function getData(): array
{
    if ($this->data === null) {
        $this->data = $this->expensiveService->fetchData();
    }
    return $this->data;
}
```

## Cron Health Check

```bash
# Verify cron is running
govard sh -c "crontab -l | grep magento"

# Check cron_schedule table
govard sh -c "bin/magento cron:install"

# Manual cron run for testing
govard sh -c "bin/magento cron:run --group=default"
```

| Cron Group | Schedule | Purpose |
|------------|----------|---------|
| default | Every minute | Low priority tasks |
| index | Every minute | Indexer updates |
| consumers | Every minute | Message queue |

## Security Probes

### Exposed Files Check

```bash
# Check for sensitive file exposure
curl -s https://store.test/app/etc/env.php | head -20
curl -s https://store.test/composer.json | head -20
curl -s https://store.test/.git/config 2>/dev/null
```

### Debug Headers

```bash
# Check for information leakage
curl -I https://store.test/ | grep -i "X-.*debug\|X-.*profile"

# Expected: No debug headers in production
```

## Audit Report Template

```markdown
# Performance Audit Report

## Infrastructure
- [ ] Application Mode: production
- [ ] PHP OPcache: >= 256MB
- [ ] Redis Session: configured
- [ ] Redis Cache: configured
- [ ] Varnish: enabled

## Cache Status
- [ ] All critical caches enabled
- [ ] FPC enabled and configured

## Indexer Configuration
- [ ] All indexers on "Update by Schedule"
- [ ] Cron running properly

## Database
- [ ] Query count < 80 on homepage
- [ ] Query count < 150 on category pages
- [ ] Query count < 150 on product pages
- [ ] No N+1 queries detected

## Core Web Vitals
| Metric | Value | Status |
|--------|-------|--------|
| LCP | X.Xs | PASS/FAIL |
| INP | X.Xms | PASS/FAIL |
| CLS | X.XXX | PASS/FAIL |

## Recommendations
1. ...
2. ...
3. ...
```

## Usage

**When invoked:**
1. Execute infrastructure checks (env.php, mode, cache status)
2. Run indexer status check
3. Perform database query profiling (if URL provided)
4. Run Lighthouse audit (if URL provided)
5. Scan for code-level performance patterns
6. Generate report with recommendations

**Trigger phrases:**
- "Audit performance"
- "Check configuration"
- "Run performance test"
- "Analyze database queries"
- "Core Web Vitals report"