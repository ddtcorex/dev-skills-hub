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

## Related Skills

**REQUIRED BACKGROUND:** Load `magento2-dev-core` first — code-level fixes for N+1 queries and heavy constructors follow the patterns it defines.

Part of the QA trio with `magento2-linter` and `magento2-security-scan`. Async/queue findings often point back to `magento2-backend-dev`.

## Audit Categories

### 1. Infrastructure Configuration

| Check | Expected | Command |
|-------|----------|---------|
| Application Mode | `production` | `bin/magento deploy:mode:show` |
| PHP OPcache | >= 256MB | Check php.ini |
| Redis (Session) | Enabled | Check `app/etc/env.php` |
| Redis (Cache) | Enabled | Check `app/etc/env.php` |
| Varnish | Running + terminating HTTPS/HTTP in front of the app | Check for a running Varnish container/process and its VCL config; the exact config file is project-specific (e.g. `nginx.conf`, a Docker/orchestrator config file, or a cloud provider's Varnish config) — there is no universal filename to grep for. |

> On a **local dev environment** it's normal and expected to have no Redis/Varnish at all (file-based session/cache, FPC often disabled) — don't flag this as a bug unless the target is staging/production. Confirm which environment you're actually auditing before treating any of the above as a problem.

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
# Check indexer mode (Update by Schedule is CRITICAL for performance on larger catalogs)
govard sh -c "bin/magento indexer:status"

# Switch a specific indexer to schedule mode (don't blanket-apply to all — see table below)
govard sh -c "bin/magento indexer:set-mode schedule <indexer_code>"

# Reindex all
govard sh -c "bin/magento indexer:reindex"
```

| Indexer | Recommended Mode |
|---------|------------------|
| catalog_product_price | Update on Save for small catalogs / frequent price changes; Update by Schedule for large catalogs (thousands+ SKUs) where synchronous reindex-on-save would slow down admin saves and imports. Don't apply one rule blindly — check catalog size and how prices are updated (manual saves vs. bulk import) first. |
| catalog_url_category | Update by Schedule |
| catalog_category_product | Update by Schedule |
| inventory | Update by Schedule |
| targetrule | Update by Schedule |

**Also check that cron is actually running and draining the changelog** — schedule-mode indexers are only as fresh as the last successful cron run. Check `crontab -l` for a magento entry, and query `cron_schedule` for recent `success` rows (`SELECT MAX(executed_at) FROM cron_schedule WHERE status='success'`) — an idle cron combined with schedule-mode indexers silently produces stale prices/URLs/inventory with no error anywhere.

### 4. Async Operations (message queue consumers)

Bulk APIs, async email sending, and async operations in Magento all run through message queue consumers — they don't do anything unless the consumers are actually running as processes (via cron or a supervisor), not just configured.

```bash
# List available consumers (does NOT start/enable them — just enumerates what's defined)
govard sh -c "bin/magento queue:consumers:list"

# Check whether consumers are actually running as processes
govard sh -c "ps aux | grep 'queue:consumers:start'"

# Start a specific consumer manually (for testing — production should run these via cron/supervisor)
govard sh -c "bin/magento queue:consumers:start <consumer_name> --max-messages=100"
```

If no `queue:consumers:start` processes are running and there's no cron/supervisor job launching them, bulk operations and async email will queue up in `queue_message` tables and never actually process — check for this rather than assuming a config flag turns "async" on.

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

### DB Query Log Setup

```bash
# Enable full query logging with call stacks (see caveat below on log size)
govard sh -c "bin/magento dev:query-log:enable --include-all-queries=true --include-call-stack=true --query-time-threshold=0"

# Visit the page(s) to capture queries — output goes to var/debug/db.log (plain text, NOT *.sql)
# Format per entry: "## QUERY" header, then "SQL: ...", "AFF: <rows>", "TIME: <seconds>", then (if
# --include-call-stack=true) a full PHP call stack — use the stack to trace a repeated/slow query
# back to the exact file:line that issued it.

# Count queries for one page load: clear the log, hit the page once, count entries
govard sh -c "> var/debug/db.log"
curl -sk -o /dev/null https://store.test/
govard sh -c "grep -c '^## QUERY' var/debug/db.log"

# ALWAYS disable when done — this is expensive and grows fast (a single page load with
# --include-call-stack=true can produce several MB of log; on a bigger page ~10+ MB is normal)
govard sh -c "bin/magento dev:query-log:disable"
```

### Common Query Issues

| Issue | Pattern | Impact |
|-------|---------|--------|
| N+1 Query | `foreach` with `->load()` inside, or the same normalized query shape (ignore literal values) appearing dozens of times in `var/debug/db.log` for one page load | High |
| Full Collection Load | `count($collection)` | Medium |
| Missing Index | `WHERE unindexed_column` | High |
| Expensive Join | Multiple JOINs on large tables | Medium |

### Query Analysis Commands

```bash
# Show slow queries (requires MySQL slow_query_log)
govard db query "SHOW FULL PROCESSLIST"
```

> **Local dev DBs are small and fast** — the absolute query *time* on a local box will often look fine (tens of milliseconds total) even when the query *count* is far over budget. Raw count is what matters here: on production, the same N+1 pattern pays a real network round-trip per query (even ~0.3–1ms same-datacenter) against much larger tables, so a high count on a fast local DB is still a real finding, not a false positive — don't dismiss it just because the local timing looks fine.

### HTML Profiler (per-request timing breakdown)

```bash
# Enable the code profiler with HTML output
govard sh -c "bin/magento dev:profiler:enable html"

# IMPORTANT: the profiler only activates if the request's Accept header contains "text/html" —
# a bare `curl -s` without this header will produce NO profiler output at all (this is checked
# in app/bootstrap.php). Always include it:
curl -sk -H "Accept: text/html" -o page.html https://store.test/

# The profiler table is appended near the end of the HTML response body (a
# `<table border="1">...</table>` with columns: Timer Id, Time, Avg, Cnt, Emalloc, RealMem).
# Timer Id values use "->" as a nesting separator and are also embedded in each cell's
# `title="..."` attribute — if parsing programmatically, match on `<td title="[^"]*">(.*?)</td>`,
# not a naive `<td[^>]*>`, since the nesting arrows inside the attribute value will break a
# naive parser that treats any ">" as the tag's end.

# Disable when done
govard sh -c "bin/magento dev:profiler:disable"
```

## Per-Page-Type Audit (homepage, product, category)

A single-page spot check isn't representative — different page types have very different bottleneck shapes (a CMS-heavy homepage vs. a layout-heavy product page vs. a grid-heavy category page). Audit at least one of each of these three page types, using both the HTML profiler and the query log together, with `full_page`, `block_html`, and `layout` caches **disabled** so you're measuring true cache-miss cost (the worst case every real cache-miss/deploy/flush pays) rather than a warm-cache request that tells you almost nothing.

### 0. Pick genuinely representative pages first

Before measuring anything, verify the specific URLs you're about to test aren't degenerate cases — this is the single easiest way to get a misleading audit:

```bash
# Category: confirm it actually has products assigned (an empty category renders no grid,
# no pagination, no real layered-nav facets, and will understate real page cost)
govard db query "SELECT COUNT(*) FROM catalog_category_product WHERE category_id=<id>"

# Product: confirm it's assigned to a website (unassigned products 404 / aren't routable)
govard db query "SELECT * FROM catalog_product_website WHERE product_id=<id>"

# Product: also watch for url_rewrite entries that 301/302 redirect elsewhere (including,
# in some data sets, out to a live production domain) — follow redirects manually first,
# don't blindly -L through them into a request against someone's production site
curl -sk -o /dev/null -w "%{http_code} -> %{redirect_url}\n" https://store.test/<product-url>.html
```

Pick a category with a normal/median product count (not the largest root category, not an edge case), and a product that resolves 200 directly.

### 1. Set up the uncached measurement environment

```bash
govard sh -c "bin/magento dev:profiler:enable html"
govard sh -c "bin/magento dev:query-log:enable --include-all-queries=true --include-call-stack=true --query-time-threshold=0"
govard sh -c "bin/magento cache:disable full_page block_html layout"
govard sh -c "bin/magento cache:flush"

# One throwaway request first (discard its output/log) — this lets config/eav/compiled_config
# caches rebuild after the flush so that one-time rebuild cost doesn't contaminate the
# per-page numbers you're about to capture
curl -sk -H "Accept: text/html" -o /dev/null https://store.test/
govard sh -c "> var/debug/db.log"
```

### 2. Capture each page type separately

For each of the 3 URLs, clear the query log, fetch with the profiler-required `Accept` header, save the HTML (for the profiler table) and a copy of the query log, then clear the log again before the next page:

```bash
for name in home product category; do
  curl -sk -H "Accept: text/html" -o "${name}.html" "https://store.test/<url-for-$name>"
  cp var/debug/db.log "${name}.db.log"
  govard sh -c "> var/debug/db.log"
done
```

Analyze each page's `*.html` for its profiler table (slowest TEMPLATE/BLOCK rows, and total in-app time via the root `magento` timer) and each `*.db.log` for total query count, repeated/duplicate query shapes (candidate N+1s), and — using the call stack in each entry — the exact file:line responsible for the worst offenders.

### 3. Always restore state afterward

```bash
govard sh -c "bin/magento cache:enable full_page block_html layout"
govard sh -c "bin/magento cache:flush"
govard sh -c "bin/magento dev:profiler:disable"
govard sh -c "bin/magento dev:query-log:disable"
rm -f var/debug/db.log
```

Don't leave a target environment with caches disabled and full query logging on — this is a diagnostic state, not a normal running state, and matters especially if the target is shared with other developers or is staging rather than a disposable local box.

### Interpreting results correctly

- **Query count vs. query time are different signals.** A small/fast local DB can show trivial total query *time* (tens of ms) even when the query *count* is 3–5x over budget — don't dismiss a high count just because local timing looks fine; the count is what will hurt on production's real network round-trips and larger tables.
- **Not every slow section benefits from the caches you just disabled.** `layout` cache only skips re-parsing/merging layout XML — it does NOT skip instantiating the PHP block objects for every declared block (that happens fresh on every request regardless of cache, since live objects can't be cached across requests). If a page's time is dominated by layout *generation* rather than block *rendering*, re-enabling `layout`/`block_html` cache won't fix it — the real lever is reducing how many blocks/modules contribute to that page's layout.
- **A query shape repeated with near-identical counts across all 3 page types** (not just one) is a strong signal it comes from a globally-rendered block (header/footer/cart-drawer widget), not something page-specific — prioritize fixing that over a page-specific N+1, since it's paid on every single page view site-wide.

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

### Debug Headers & Version Disclosure

```bash
# Check for information leakage — debug/profiler headers AND server/version disclosure
curl -I https://store.test/ | grep -iE "x-.*debug|x-.*profile|^server:|x-powered-by"

# Expected: no debug headers, and Server/X-Powered-By should not reveal exact
# nginx/PHP versions in production (server_tokens off; expose_php = Off)
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
- [ ] Query count < 80 on homepage (uncached — see Per-Page-Type Audit)
- [ ] Query count < 150 on category pages (uncached, on a page with a representative/median product count — not an empty category)
- [ ] Query count < 150 on product pages (uncached, on a product that resolves 200 — not one missing a website assignment)
- [ ] No N+1 queries detected (same query shape repeated many times in one page's `var/debug/db.log`)
- [ ] Note: on a small/fast local DB, absolute query time can look fine even when count is over budget — flag on count, not just time

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

## Workflow

**When invoked:**
1. Execute infrastructure checks (env.php, mode, cache status) — first confirm whether the target is local dev, staging, or production, since expectations differ (see note under Infrastructure Configuration)
2. Run indexer status check, and verify cron is actually running/draining `cron_schedule`
3. Run the Per-Page-Type Audit (homepage, product, category) with `full_page`/`block_html`/`layout` caches disabled — verify each test page is representative first (§0), then capture profiler + query log together (§1–2), then restore state (§3)
4. Run Lighthouse / Core Web Vitals audit (if URL provided)
5. Scan for code-level performance patterns
6. Generate report with recommendations, prioritizing any finding that repeats across all 3 page types (site-wide impact) over page-specific ones