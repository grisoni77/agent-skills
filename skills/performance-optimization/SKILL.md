---
name: performance-optimization
description: Optimizes application performance. Use when performance requirements exist, when you suspect performance regressions, or when Core Web Vitals or load times need improvement. Use when profiling reveals bottlenecks that need fixing.
---

# Performance Optimization

## Overview

Measure before optimizing. Performance work without measurement is guessing — and guessing leads to premature optimization that adds complexity without improving what matters. Profile first, identify the actual bottleneck, fix it, measure again. Optimize only what measurements prove matters.

## When to Use

- Performance requirements exist in the spec (load time budgets, response time SLAs)
- Users or monitoring report slow behavior
- Core Web Vitals scores are below thresholds
- You suspect a change introduced a regression
- Building features that handle large datasets or high traffic

**When NOT to use:** Don't optimize before you have evidence of a problem. Premature optimization adds complexity that costs more than the performance it gains.

## Core Web Vitals Targets

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## The Optimization Workflow

```
1. MEASURE  → Establish baseline with real data
2. IDENTIFY → Find the actual bottleneck (not assumed)
3. FIX      → Address the specific bottleneck
4. VERIFY   → Measure again, confirm improvement
5. GUARD    → Add monitoring or tests to prevent regression
```

### Step 1: Measure

**Frontend:**
```bash
# Lighthouse in Chrome DevTools (or CI)
# Chrome DevTools → Performance tab → Record
# Chrome DevTools MCP → Performance trace
```

```html
<!-- web-vitals library loaded from the CDN and wired to vanilla JS -->
<script type="module">
  import { onLCP, onINP, onCLS } from 'https://unpkg.com/web-vitals?module';
  onLCP(console.log);
  onINP(console.log);
  onCLS(console.log);
</script>
```

**Backend:**
```bash
# MySQL slow query log — slowest offenders across the app
sudo mysqldumpslow -s t -t 20 /var/log/mysql/slow.log

# Xdebug profiler → Cachegrind dumps openable in KCachegrind/qcachegrind
php -d xdebug.mode=profile -d xdebug.output_dir=/tmp bin/run.php
```

```php
// Simple timing in PHP (temporary, remove after diagnosis)
$start = hrtime(true);
$rows  = $pdo->query('SELECT ...')->fetchAll();
$ms    = (hrtime(true) - $start) / 1_000_000;
error_log(sprintf('db-query: %.2f ms', $ms));
```

### Where to Start Measuring

Use the symptom to decide what to measure first:

```
What is slow?
├── First page load
│   ├── Large JS/CSS bundle? --> Measure asset size, check code splitting
│   ├── Slow server response?  --> Measure TTFB, check PHP-FPM, MySQL, OPcache
│   └── Render-blocking resources? --> Check network waterfall for CSS/JS blocking
├── Interaction feels sluggish
│   ├── UI freezes on click?   --> Profile main thread, look for long tasks (>50ms)
│   ├── Form input lag?        --> Check jQuery handlers, event delegation
│   └── Animation jank?        --> Check layout thrashing, forced reflows
├── Page after navigation
│   ├── Data loading?          --> Measure fetch/XHR response times, check for waterfalls
│   └── Template rendering?    --> Smarty compile cache warm? template_c writable?
└── Backend / API
    ├── Single endpoint slow?  --> EXPLAIN the queries, check indexes, OPcache status
    ├── All endpoints slow?    --> Check PHP-FPM pool sizing, MySQL connections, server CPU
    └── Intermittent slowness? --> Check InnoDB lock contention, Cloudflare cache hit ratio, external deps
```

### Step 2: Identify the Bottleneck

Common bottlenecks by category:

**Frontend:**

| Symptom | Likely Cause | Investigation |
|---------|-------------|---------------|
| Slow LCP | Large images, render-blocking resources, slow server | Check network waterfall, image sizes |
| High CLS | Images without dimensions, late-loading content, font shifts | Check layout shift attribution |
| Poor INP | Heavy JavaScript on main thread, large DOM updates | Check long tasks in Performance trace |
| Slow initial load | Large bundle, many network requests | Check bundle size, code splitting |

**Backend:**

| Symptom | Likely Cause | Investigation |
|---------|-------------|---------------|
| Slow responses | N+1 queries, missing indexes, unoptimized SQL | `EXPLAIN` the queries, read the slow query log |
| Memory growth | Large result sets loaded into arrays, Smarty template cache bloat | `memory_get_peak_usage()`, Xdebug profiling |
| CPU spikes | Synchronous heavy computation, regex backtracking | Xdebug profiler + Cachegrind |
| High latency | Missing OPcache, cold template compile, no HTTP cache | Check OPcache status, Nginx `fastcgi_cache`, Cloudflare/Akamai hit ratio |

### Step 3: Fix Common Anti-Patterns

#### N+1 Queries (Backend)

```php
// BAD: N+1 — one query per task for the owner
$tasks = $pdo->query('SELECT id, title, owner_id FROM tasks')->fetchAll();
foreach ($tasks as &$task) {
    $stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
    $stmt->execute([$task['owner_id']]);
    $task['owner'] = $stmt->fetch();
}

// GOOD: Single query with JOIN
$sql = <<<SQL
    SELECT t.id, t.title, u.id AS owner_id, u.name AS owner_name
    FROM tasks t
    INNER JOIN users u ON u.id = t.owner_id
SQL;
$tasks = $pdo->query($sql)->fetchAll(PDO::FETCH_ASSOC);

// GOOD (alternative): one follow-up query with IN() — still O(1) roundtrips
$tasks    = $pdo->query('SELECT id, title, owner_id FROM tasks')->fetchAll();
$ownerIds = array_column($tasks, 'owner_id');
$in       = implode(',', array_fill(0, count($ownerIds), '?'));
$stmt     = $pdo->prepare("SELECT id, name FROM users WHERE id IN ($in)");
$stmt->execute($ownerIds);
$owners   = array_column($stmt->fetchAll(), null, 'id');
foreach ($tasks as &$task) {
    $task['owner'] = $owners[$task['owner_id']] ?? null;
}
```

See `database-design-and-optimization` for index strategy and `EXPLAIN` workflows.

#### Unbounded Data Fetching

```php
// BAD: fetching every row
$allTasks = $pdo->query('SELECT * FROM tasks')->fetchAll();

// GOOD: OFFSET-based pagination (simple, fine for small offsets)
$stmt = $pdo->prepare(
    'SELECT id, title, created_at
       FROM tasks
       ORDER BY created_at DESC
       LIMIT :limit OFFSET :offset'
);
$stmt->bindValue(':limit',  20,                       PDO::PARAM_INT);
$stmt->bindValue(':offset', ($page - 1) * 20,         PDO::PARAM_INT);
$stmt->execute();
$tasks = $stmt->fetchAll(PDO::FETCH_ASSOC);

// BETTER: keyset pagination — stable O(log n) even for deep pages
$stmt = $pdo->prepare(
    'SELECT id, title, created_at
       FROM tasks
       WHERE created_at < :cursor
       ORDER BY created_at DESC
       LIMIT 20'
);
$stmt->execute([':cursor' => $cursor]);
```

#### Missing Image Optimization (Frontend)

```html
<!-- BAD: No dimensions, no lazy loading, no responsive sizes -->
<img src="/hero.jpg" />

<!-- GOOD: Responsive, lazy-loaded, properly sized -->
<img
  src="/hero.jpg"
  srcset="/hero-400.webp 400w, /hero-800.webp 800w, /hero-1200.webp 1200w"
  sizes="(max-width: 768px) 100vw, 50vw"
  width="1200"
  height="600"
  loading="lazy"
  alt="Hero image description"
/>
```

#### Expensive Template Work (Smarty / jQuery)

```smarty
{* BAD: N+1 style lookup inside a template loop *}
{foreach $tasks as $task}
  {assign var=owner value=UserService::findById($task.owner_id)}
  <li>{$task.title} — {$owner.name|escape}</li>
{/foreach}

{* GOOD: pre-join owner in PHP, assign a flat list to the template *}
{foreach $tasks as $task}
  <li>{$task.title|escape} — {$task.owner_name|escape}</li>
{/foreach}
```

```javascript
// BAD: jQuery DOM churn inside a loop (re-renders per iteration)
items.forEach(function (item) {
  $('#list').append('<li>' + item.name + '</li>');
});

// GOOD: build the fragment once, insert once
var html = items.map(function (i) {
  return '<li>' + $('<div>').text(i.name).html() + '</li>';
}).join('');
$('#list').append(html);
```

#### OPcache and Object Caches (Server-side)

```ini
; php.ini — production tuning for PHP 8
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0   ; bump the version on deploy instead
opcache.jit=tracing
opcache.jit_buffer_size=128M
```

```php
// In-process memoization for config that is expensive to load
final class ConfigCache
{
    private static ?AppConfig $cached   = null;
    private static int        $expiryTs = 0;
    private const TTL_SECONDS           = 300;

    public static function get(PDO $pdo): AppConfig
    {
        if (self::$cached !== null && time() < self::$expiryTs) {
            return self::$cached;
        }
        self::$cached   = AppConfig::loadFromDb($pdo);
        self::$expiryTs = time() + self::TTL_SECONDS;
        return self::$cached;
    }
}
```

#### HTTP Caching — Nginx + Akamai / Cloudflare

```nginx
# Nginx: long-lived immutable assets (filename is content-hashed)
location ~* \.(?:js|css|woff2|png|jpg|webp)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# Fastcgi cache for anonymous GET responses (bypass for logged-in users)
fastcgi_cache_bypass $cookie_PHPSESSID;
fastcgi_no_cache     $cookie_PHPSESSID;
fastcgi_cache_valid  200 10m;
```

```php
// Send explicit Cache-Control from PHP for cacheable endpoints
header('Cache-Control: public, max-age=300, s-maxage=600');
header('Vary: Accept-Encoding');
```

See `server-deployment-lemp` for the full Akamai / Cloudflare purge strategy
and cache-bypass cookie configuration.

## Performance Budget

Set budgets and enforce them:

```
JavaScript bundle:         < 200KB gzipped (initial load)
CSS:                       < 50KB  gzipped
Images:                    < 200KB per image (above the fold)
Fonts:                     < 100KB total
Server TTFB (cached):      < 100ms
Server TTFB (uncached):    < 400ms (p95)
MySQL slow query threshold: 200ms
Lighthouse Performance:    ≥ 90
```

**Enforce in CI:**
```bash
# Lighthouse CI (runs against a preview URL in Bitbucket Pipelines)
npx lhci autorun

# Fail the build if any query in the slow query log crossed the threshold
./bin/assert-no-slow-queries.sh
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We'll optimize later" | Performance debt compounds. Fix obvious anti-patterns now, defer micro-optimizations. |
| "It's fast on my machine" | Your machine isn't the user's. Profile on representative hardware and networks. |
| "This optimization is obvious" | If you didn't measure, you don't know. Profile first. |
| "Users won't notice 100ms" | Research shows 100ms delays impact conversion rates. Users notice more than you think. |
| "The framework handles performance" | Frameworks prevent some issues but can't fix N+1 queries or oversized bundles. |

## Red Flags

- Optimization without profiling data to justify it
- N+1 query patterns in data fetching
- List endpoints without pagination
- Images without dimensions, lazy loading, or responsive sizes
- Bundle size growing without review
- No performance monitoring in production
- OPcache disabled in production, or `validate_timestamps=1` on a busy app
- `SELECT *` everywhere with no index awareness

## Verification

After any performance-related change:

- [ ] Before and after measurements exist (specific numbers)
- [ ] The specific bottleneck is identified and addressed
- [ ] Core Web Vitals are within "Good" thresholds
- [ ] Bundle size hasn't increased significantly
- [ ] No N+1 queries in new data fetching code
- [ ] Performance budget passes in CI (if configured)
- [ ] Existing tests still pass (optimization didn't break behavior)
