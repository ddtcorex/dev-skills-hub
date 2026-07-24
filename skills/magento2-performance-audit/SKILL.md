---
name: magento2-performance-audit
description: |
  Performs comprehensive performance and health audit for Magento 2 projects. Use when:
  - "Audit performance", "check Core Web Vitals", "run Lighthouse"
  - "Check server configuration", "verify Redis/Varnish setup"
  - "Analyze database queries", "find N+1 query issues"
  - "Review indexer configuration", "check cron health"
  - "Debug cache flush", "why does full_page cache keep flushing", "trace FPC invalidation"
  - "Too many ajax requests", "customer data section reload storm", "crawler overloading server"

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

> If `full_page` cache is flushing far more often than page saves/deploys would explain, see **Cache Invalidation Efficiency Audit** below — Magento's own entity-save invalidation is narrowly scoped by design; unexplained broad/frequent flushes are almost always custom observer or plugin code.

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

## Cache Invalidation Efficiency Audit

Magento's default invalidation on entity save (product, category, CMS block/page, etc.) is already narrowly scoped — `getIdentities()` on the saved entity returns a small set of cache tags, and only pages/blocks carrying those tags get cleared. The problem this section targets is **custom code** (an observer, a plugin, a cron job, a "just flush everything to be safe" habit) that widens or duplicates that invalidation — clearing the entire `full_page` cache (or all cache types) on saves that only ever needed to touch a handful of tags. This is invisible in `cache:status` (caches are still "Enabled") and easy to miss without tracing what actually gets cleared and why.

### A. Built-in FPC (Redis or file cache backend, no Varnish)

Every Magento cache frontend (config, layout, block_html, full_page, etc.) is wrapped by `Magento\Framework\Cache\Frontend\Decorator\Logger` out of the box (see `vendor/magento/magento2-base/app/etc/di.xml`) — this is backend-agnostic (identical on Redis or file) and needs **no env.php change at all**. Every `clean()`/`remove()` call already logs a `cache_invalidate:` line at DEBUG level via `Magento\Framework\Cache\InvalidateLogger`, e.g.:

```
[...] main.DEBUG: cache_invalidate:  {"method":"GET","url":"http:/...","invalidateInfo":{"tags":["cat_p_123","FPC"],"mode":"matchingTag"}} []
[...] main.DEBUG: cache_invalidate:  {"method":"GET","url":"http:/...","invalidateInfo":{"tags":["FPC"],"mode":"matchingTag"}} []
```

**1. Confirm debug logging reaches `var/log/debug.log`.** Whether these DEBUG records are actually written depends on deployment config, not admin store config — `bin/magento config:set dev/debug/debug_logging 1` targets the wrong config store and will error (`Le chemin ... n'existe pas` / "path does not exist"). The real toggle is a deployment config value (`app/etc/env.php`), defaulting to **on** whenever the app is not in `production` mode:

```bash
govard sh -c "bin/magento deploy:mode:show"   # developer/default -> logging is on by default, nothing to change
# Only needed on production (verbose — logs ALL debug-level messages app-wide, not just cache;
# always pair with the matching --enable-debug-logging=0 once diagnosis is done):
govard sh -c "bin/magento setup:config:set --enable-debug-logging=1"
```

**2. Reproduce ONE isolated action, then grep for the invalidation trail:**

```bash
govard sh -c "> var/log/debug.log"
# ... perform the action (save one product, run one cron job, etc.) ...
govard sh -c "grep 'cache_invalidate:' var/log/debug.log"
```

Read the `tags`/`mode` in each `invalidateInfo` payload. A correctly-scoped save produces tags that pair the entity's own tag with the type tag (e.g. `["cat_p_123","FPC"]`) — Magento adds `"FPC"` as `full_page`'s type-scope tag automatically, it does not by itself mean "everything got cleared". The actual smoking gun is an entry whose tags are **only** the bare type tag (`["FPC"]` alone, with no accompanying entity tag) — that clears every single cached page for one action. Also watch for the *same* tag set logged more than once within a few seconds of one save — that can be a redundant custom observer, though first rule out an async reindex (schedule-mode indexer draining the changelog) re-triggering the same invalidation shortly after, which is expected behavior, not a bug.

**3. Cron/CLI-triggered invalidations, or entries whose `url`/`method` don't identify the caller** (a CLI-run action logs a generic `"method":"GET","url":"http:/"` with no useful context): add a temporary diagnostic plugin that logs a stack trace whenever `clean()` is called broadly — this is the only way to get a file:line for something triggered outside an HTTP request.

```php
<?php
// app/code/Vendor/DevTools/Plugin/TraceCacheClean.php — TEMPORARY, remove after diagnosis
declare(strict_types=1);

namespace Vendor\DevTools\Plugin;

use Magento\Framework\App\Cache;
use Psr\Log\LoggerInterface;

class TraceCacheClean
{
    public function __construct(private readonly LoggerInterface $logger)
    {
    }

    public function beforeClean(Cache $subject, $mode = \Zend_Cache::CLEANING_MODE_ALL, array $tags = []): array
    {
        // Log unconditionally — only run this during ONE isolated reproduction step, so volume stays manageable
        $this->logger->debug(
            sprintf("CACHE CLEAN mode=%s tags=%s\n%s", $mode, implode(',', $tags), (new \Exception())->getTraceAsString())
        );
        return [$mode, $tags];
    }
}
```

```xml
<!-- app/code/Vendor/DevTools/etc/di.xml — TEMPORARY -->
<type name="Magento\Framework\App\Cache">
    <plugin name="devtools_trace_cache_clean" type="Vendor\DevTools\Plugin\TraceCacheClean"/>
</type>
```

The logged trace's file:line points directly at the observer/plugin/cron job issuing the clean. **Remove this plugin (and, if it was changed on production, revert `--enable-debug-logging`) as soon as diagnosis is done** — same rule as the query-log/profiler tools elsewhere in this skill: this is a diagnostic state, not something to leave running.

### B. Varnish-fronted FPC

Varnish invalidation happens via HTTP BAN requests carrying an `X-Magento-Tags-Pattern` header — Magento's `debug.log` doesn't see this directly, so trace it on the Varnish side:

```bash
# Watch BAN requests live as you reproduce an action
varnishlog -g request -q 'ReqMethod eq "BAN"'

# Inspect currently active bans — a fast-growing list, or any pattern that is
# just ".*" (matches everything), is the same "flush-all" anti-pattern as mode=all above
varnishadm ban.list
```

A `.*` pattern (or a pattern far broader than the tags of the entity actually saved) means the whole cache was purged for one change. To trace which PHP code issued that specific BAN, use the same temporary diagnostic plugin from branch A — Varnish purges still originate from the same `CacheInterface`/`clean_cache_by_tags` call path in Magento before the BAN request goes out.

### Interpreting results

| Signal | Pattern | Severity |
|--------|---------|----------|
| Full/blanket flush | debug.log entry whose tags are **only** the bare type tag (e.g. `["FPC"]` alone, no entity tag alongside it), or `X-Magento-Tags-Pattern: .*` in varnishlog/ban.list | High |
| Over-broad tag scope | Tag list clears far more than the entity actually changed (e.g. clearing every `cat_p_*` tag for a single product save) | Medium-High |
| Flush tied to non-rendering fields | Invalidation fires on saving a field never used in any cacheable block's `getIdentities()`/cache tags | Medium |
| Untraceable / scheduled flush | Repeated clean/BAN entries with no corresponding admin/API save nearby — often a cron job or deploy script calling `cache:flush` on a timer "just in case" | High |
| Duplicate flush per save | The same tag set logged more than once within seconds of one single save — rule out an async reindex re-invalidating the same tag shortly after (expected) before calling it a redundant custom observer (a bug) | Low-Medium |

> Flush frequency should scale with save/import volume, not run on a fixed schedule unrelated to actual content changes. If `cache:flush`/`cache:clean` shows up in a cron job or deploy script "just to be safe," that's a scheduled full flush independent of whether anything relevant even changed — treat it the same as a bare-type-tag finding above.

## Client-Side AJAX Request Load Audit

A page can pass every check above — `full_page` cache enabled, invalidation narrowly scoped — and still overload the backend, because `full_page` only caches the initial HTML response. Customer data (private content), GraphQL calls, and custom AJAX endpoints are session/customer-scoped, so they bypass FPC entirely and hit PHP-FPM/the database on **every single page view**, cached HTML or not. This matters more than it used to: modern crawlers (SEO bots, AI scrapers, headless-Chrome-based tools) execute JS the same way a real browser does, so every page they crawl re-fires the same AJAX calls a human visitor would — a site can look fully cached in every metric above and still fall over under crawl volume, because the part that's actually uncached is invisible to an HTML/query-count audit.

### 1. Capture the AJAX footprint of a representative page

Do this for **each of the 3 page types** from the Per-Page-Type Audit (homepage, product, category) — same rationale: the AJAX footprint differs by page type just as much as the query/profiler footprint does, and a homepage-only check can miss the worst offender entirely (a real audit found the homepage firing one same-origin call, while the product page on the same site fired six distinct same-origin endpoints — the two page types are not interchangeable samples).

Open Chrome DevTools > Network, filter to Fetch/XHR, and load the page **in incognito / with site data cleared** — this simulates what an anonymous visitor (or crawler) sees, not a footprint inflated or deflated by your own logged-in dev session.

> **Watch specifically for `/customer/section/load` with an empty `sections` parameter** (`?sections=`). Magento has a long-standing quirk where an empty `sections` filter returns **every** registered section rather than none — including the full cart, checkout eligibility, and the complete worldwide country/region directory used by address forms. What looks like the cheapest, most trivial call on the page can silently be the single most expensive uncached response it makes. Check the actual response body size/content, not just the request URL, before dismissing it.

- Count same-origin (your Magento domain) requests separately from third-party ones — only same-origin requests add load to your server; a Facebook/Google pixel call goes straight to their servers and isn't a Magento capacity concern (unless the project has a custom server-side tracking proxy — e.g. a controller relaying Conversion API/Measurement Protocol events — in which case treat that controller like any other custom AJAX endpoint below). In practice, on a real storefront this list is often dominated by third-party marketing tags (chat widgets, popups, review widgets, ad pixels) — don't let their volume distract from the (usually much smaller) same-origin count, which is the one that matters here.
- For each same-origin XHR/fetch, check its response headers — anything `Cache-Control: private`/`no-store` (Magento surfaces this as `X-Magento-Cache-Debug: MISS` / `X-Magento-Cache-Control: ..., no-store` on its own responses) is a call that hits the backend fresh every time, unlike the FPC-served HTML.
- Watch for the *same* same-origin endpoint firing more than once per page load, especially if each call sets a **new** `Set-Cookie: PHPSESSID=...` — that means each occurrence is opening its own PHP session server-side, doubling (or worse) the real backend cost of a single page view.

Common first-party culprits to look for: `/customer/section/load` (private content / Customer Data), `/graphql` (if any theme/PWA component fetches client-side), custom wishlist/compare/stock-check/price AJAX endpoints, any custom analytics/tracking proxy controller, and — easy to miss — **third-party marketing/personalization integrations (Connectif, Klaviyo, Nosto, etc.) that ship their own customer/cart-context controller** to sync cart/login state into their widget. These sit entirely outside `sections.xml`/Magento's private-content system, so auditing §2 below won't catch them — only the network capture in this step will.

### 2. Audit `sections.xml` for overly broad Customer Data invalidation

`sections.xml` declares which Customer Data sections must be invalidated (forcing a `/customer/section/load` reload) after specific controller actions. A wildcard or over-broad rule forces **every** unrelated action to reload the **full** section list, not just what actually changed — the most common cause of a `/customer/section/load` reload storm.

```xml
<!-- WRONG - "*" invalidates ALL sections on ANY controller action -->
<action name="*">
    <section name="cart"/>
    <section name="customer"/>
    <section name="wishlist"/>
    <section name="compare-products"/>
    <section name="review"/>
</action>

<!-- CORRECT - scope invalidation to only the sections that action actually affects -->
<action name="checkout_cart_add">
    <section name="cart"/>
</action>
<action name="wishlist_index_add">
    <section name="wishlist"/>
</action>
```

```bash
# Find wildcard/broad invalidation rules across custom and third-party modules
grep -rn '<action name="\*"' app/code vendor/*/module-*/etc/frontend/sections.xml 2>/dev/null
```

> Magento core itself ships one wildcard rule (`vendor/magento/module-theme/etc/frontend/sections.xml`, `<action name="*"><section name="messages"/></action>`) — that one is expected and cheap (a single lightweight section on every action). It's not the anti-pattern; the anti-pattern is a wildcard rule reloading several/heavy sections. When grepping, check what's actually declared inside each match before flagging it — don't flag on the wildcard alone.

Also check custom JS for widgets that force their own reload instead of relying on the invalidation-rule-driven one — this duplicates whatever `sections.xml` already triggers for the same user action:

```javascript
// WRONG - forces a reload of everything on every click, on top of whatever
// the sections.xml invalidation rule for this action already reloads
$('.some-widget').on('click', function () {
    customerData.reload(['cart', 'customer', 'wishlist'], true);
});

// CORRECT - let sections.xml own invalidation for rule-covered sections;
// only call reload() explicitly for a section with no matching action rule,
// and scope it to just that section
customerData.reload(['wishlist-widget-count'], false);
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

### Inefficient Cache Invalidation

```php
// WRONG - blanket flush on every save, regardless of what actually changed
class FlushFullPageCache implements ObserverInterface
{
    public function execute(Observer $observer): void
    {
        $this->cacheTypeList->cleanType(\Magento\PageCache\Model\Cache\Type::TYPE_IDENTIFIER); // clears ALL of full_page
    }
}

// WRONG - "just in case" full flush from a cron job or deploy script,
// unconditional and unrelated to whether anything relevant changed
// bin/magento cache:flush

// CORRECT - targeted invalidation scoped to the entity that actually changed
public function execute(Observer $observer): void
{
    $product = $observer->getEvent()->getProduct();
    $this->cache->clean(
        \Zend_Cache::CLEANING_MODE_MATCHING_TAG,
        [\Magento\Catalog\Model\Product::CACHE_TAG . '_' . $product->getId()]
    );
}

// EVEN BETTER - don't add a parallel custom flush at all; make sure the
// affected block/data actually participates in the entity's native getIdentities()
// cache tags, so Magento's own save-time invalidation already covers it
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

## Cache Invalidation Efficiency
- [ ] No unexplained full/`mode=all` flushes outside deploy/indexer/explicit admin-flush windows
- [ ] Custom observers/plugins use targeted tag invalidation, not blanket `clean()`/`cache:flush`
- [ ] (Varnish only) `varnishadm ban.list` shows no overly broad patterns (e.g. `.*`) originating from custom code
- [ ] Flush frequency is proportional to actual entity save/import volume, not constant/scheduled

## Client-Side AJAX Load
- [ ] Same-origin (Magento) AJAX/XHR count on a fresh/anonymous page load noted as baseline
- [ ] No wildcard (`<action name="*">`) Customer Data invalidation rules in `sections.xml`
- [ ] No redundant `customerData.reload()` calls duplicating invalidation-rule-driven reloads
- [ ] Uncacheable (session/customer-scoped) AJAX endpoints identified — these are what crawler/bot JS execution multiplies under load, independent of FPC hit rate

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
4. Trace cache invalidation efficiency — enable temporary logging (debug.log for built-in Redis/file FPC, varnishlog/ban.list for Varnish), reproduce one isolated save/action, and flag any custom code causing broad/frequent flushes beyond Magento's default targeted invalidation
5. Capture the AJAX footprint of a fresh/anonymous page load (Network tab) and audit `sections.xml` for overly broad Customer Data invalidation — these uncacheable requests are what crawler/bot JS execution multiplies regardless of FPC hit rate
6. Run Lighthouse / Core Web Vitals audit (if URL provided)
7. Scan for code-level performance patterns
8. Generate report with recommendations, prioritizing any finding that repeats across all 3 page types (site-wide impact) over page-specific ones