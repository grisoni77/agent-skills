---
name: server-deployment-lemp
description: Deploys PHP applications to a LEMP stack behind Akamai or Cloudflare. Use when configuring Nginx, PHP-FPM, or OPcache for production, when setting up zero-downtime deploys, or when coordinating cache purges with a CDN after a release.
---

# Server Deployment — LEMP

## Overview

Deploy PHP applications safely and predictably on a Linux + Nginx + MySQL + PHP-FPM stack, typically fronted by Akamai or Cloudflare. The goal is deployments that are atomic, reversible, and observable — no half-copied file trees, no manual SSH tinkering, no "it works on the dev server."

## When to Use

- Setting up a new LEMP host
- Configuring Nginx or PHP-FPM for production
- Wiring a CI pipeline to deploy to a real server
- Preparing a CDN (Akamai / Cloudflare) cache strategy
- Planning zero-downtime releases or rollbacks
- Hardening the server before go-live

## The Deploy Pipeline

```
CI build  →  rsync to release dir  →  run migrations  →  swap symlink
                                                           │
                                                           ▼
                                               reload php-fpm + nginx
                                                           │
                                                           ▼
                                                     purge CDN
                                                           │
                                                           ▼
                                              health check + monitor
```

## Server Layout

```
/var/www/app/
├── current          → symlink to releases/<timestamp>
├── releases/
│   ├── 20260401120000-a1b2c3/
│   ├── 20260402093000-d4e5f6/
│   └── 20260403153000-g7h8i9/   ← current points here
└── shared/
    ├── .env                       # secrets, 600 perms
    ├── var/
    │   └── log/
    └── uploads/                   # persistent user uploads
```

`public/` inside each release directory is symlinked to `/var/www/app/shared/uploads` for any user-writable content, so uploads survive a deploy.

## Nginx Configuration

```nginx
# /etc/nginx/sites-available/app.conf
server {
    listen 443 ssl http2;
    server_name app.example.com;

    root /var/www/app/current/public;
    index index.php;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options     "nosniff"                            always;
    add_header X-Frame-Options            "DENY"                               always;
    add_header Referrer-Policy            "strict-origin-when-cross-origin"   always;

    # Long-lived immutable assets — filename is content-hashed in build
    location ~* \.(?:css|js|woff2|png|jpg|jpeg|webp|svg|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # Front controller pattern
    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT   $realpath_root;
        internal;

        # Anonymous GET fastcgi cache — bypass for logged-in users
        fastcgi_cache          app_cache;
        fastcgi_cache_key      "$scheme$request_method$host$request_uri";
        fastcgi_cache_valid    200 10m;
        fastcgi_cache_bypass   $cookie_APP_SESSID;
        fastcgi_no_cache       $cookie_APP_SESSID;
        add_header X-Cache-Status $upstream_cache_status;
    }

    # Deny access to everything outside public/
    location ~ /\.(?!well-known).* { deny all; }
    location ~ /(vendor|src|config|tests)/ { deny all; }
}
```

Key points:

- `root` points at `current/public` — the symlink gets swapped on each release
- `$realpath_root` makes PHP-FPM resolve the real path, so OPcache invalidates cleanly on symlink swap
- The fastcgi cache bypasses sessions via a distinguishable cookie name

## PHP-FPM Pool

```ini
; /etc/php/8.2/fpm/pool.d/app.conf
[app]
user  = www-data
group = www-data
listen = /run/php/php8.2-fpm.sock
listen.owner = www-data
listen.group = www-data

pm = dynamic
pm.max_children      = 40
pm.start_servers     = 8
pm.min_spare_servers = 4
pm.max_spare_servers = 12
pm.max_requests      = 500      ; recycle workers to avoid slow memory leaks

request_terminate_timeout = 30s
catch_workers_output     = yes
php_admin_value[error_log] = /var/www/app/shared/var/log/fpm-error.log
```

Size `pm.max_children` from the average response memory: `(total RAM for PHP) / (avg per-request memory)`. Over-provisioning causes thrashing under load.

## OPcache Tuning

```ini
; /etc/php/8.2/fpm/conf.d/10-opcache.ini
opcache.enable                 = 1
opcache.enable_cli             = 0
opcache.memory_consumption     = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files  = 20000
opcache.validate_timestamps    = 0    ; bypass stat() per file — MUST reset on deploy
opcache.revalidate_freq        = 0
opcache.save_comments          = 1
opcache.jit                    = tracing
opcache.jit_buffer_size        = 128M
```

With `validate_timestamps=0`, PHP **never re-reads a file after it's cached**. This is a big throughput win but means you **must** reset OPcache after every release. Either `systemctl reload php8.2-fpm` (restarts workers cleanly) or hit an OPcache-reset endpoint gated to localhost.

## Atomic Deploy Script

```bash
#!/usr/bin/env bash
# bin/deploy.sh — runs on CI, over SSH
set -euo pipefail

HOST="$1"              # e.g. prod-web-01.example.com
BRANCH="${2:-main}"

RELEASE="$(date +%Y%m%d%H%M%S)-$(git rev-parse --short HEAD)"
TARGET="/var/www/app/releases/$RELEASE"

# 1. Ship the build
ssh "deploy@$HOST" "mkdir -p $TARGET"
rsync -az --delete \
      --exclude='.git/' --exclude='tests/' --exclude='node_modules/' \
      ./ "deploy@$HOST:$TARGET/"

# 2. Link shared writable dirs and .env
ssh "deploy@$HOST" bash -s <<EOF
  set -euo pipefail
  ln -sfn /var/www/app/shared/.env          $TARGET/.env
  ln -sfn /var/www/app/shared/uploads       $TARGET/public/uploads
  ln -sfn /var/www/app/shared/var/log       $TARGET/var/log

  # 3. Run migrations (idempotent)
  php $TARGET/bin/migrate.php up

  # 4. Atomic symlink swap
  ln -sfn $TARGET /var/www/app/current

  # 5. Reload PHP-FPM (resets OPcache) and Nginx
  sudo systemctl reload php8.2-fpm
  sudo nginx -s reload

  # 6. Keep the last 5 releases, prune older
  cd /var/www/app/releases && ls -1tr | head -n -5 | xargs -r rm -rf
EOF

# 7. Purge the CDN
./bin/purge-cdn.sh

# 8. Smoke test
curl -fsS https://app.example.com/healthz
```

The `ln -sfn` swap is POSIX-atomic: an in-flight request either sees the old release or the new one, never a half-copied tree.

## CDN Cache Coordination (Akamai / Cloudflare)

The fastest server-side rendering is useless if the CDN keeps serving the old page. Every production release must purge the edge.

### Cache-Control Strategy

```
Path                      Cache-Control                    CDN behavior
────────────────────────────────────────────────────────────────────────
/assets/*.{js,css,woff2}  public, max-age=31536000, immutable   cache forever (content-hashed filenames)
/uploads/<hash>.jpg       public, max-age=604800                cache 1 week
/                         public, max-age=0, s-maxage=600       edge caches for 10 min, browsers revalidate
/account, /api/*          private, no-store                     never cache
```

`s-maxage` lets the CDN cache aggressively while telling browsers not to.

### Bypass Cookies

Tell the CDN to skip the cache for authenticated users by detecting a session cookie. On **Cloudflare**, use a Page Rule or Cache Rule that bypasses cache when `cookie.APP_SESSID` exists. On **Akamai**, add a Cache Key + No-Store behavior on the session cookie.

### Purge on Release

```bash
# Cloudflare — purge everything (simple; prefer targeted purges for big sites)
curl -X POST "https://api.cloudflare.com/client/v4/zones/$CF_ZONE/purge_cache" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"purge_everything":true}'

# Akamai — Fast Purge API (authenticated via EdgeGrid)
akamai purge invalidate --network production https://app.example.com/
```

Targeted purges (by URL or tag) are better for high-traffic sites — purging everything causes a brief origin-load spike while the edge refills.

## Secrets and Environment Variables

```
/var/www/app/shared/.env         ← real secrets, mode 600, owner deploy:www-data
/var/www/app/shared/.env.example ← template, committed via initial provisioning
```

Never commit `.env`. Never pass secrets as CLI arguments (they show up in `ps`). Use a provisioning tool (Ansible, Salt, or an encrypted secrets store like Vault / 1Password CLI) to write `.env` onto the host.

## SSL / TLS

Use Let's Encrypt for origins without a CDN, or the CDN's origin certificate otherwise:

```bash
certbot --nginx -d app.example.com --redirect --hsts
```

Renewal is automated via `certbot.timer`. Pin the cipher suite and enable OCSP stapling in the Nginx `ssl_*` directives.

## Health Checks and Monitoring

Expose a cheap `/healthz` endpoint that:

1. Confirms PHP-FPM can execute code (returns 200)
2. Optionally pings MySQL with a trivial `SELECT 1`
3. Is **excluded from the fastcgi cache** so it reflects real state

```php
// public/healthz.php equivalent route
$app->get('/healthz', function ($req, $res) use ($pdo) {
    try {
        $pdo->query('SELECT 1');
        $res->getBody()->write('ok');
        return $res->withStatus(200);
    } catch (\Throwable $e) {
        return $res->withStatus(503);
    }
});
```

Point uptime monitoring (Pingdom, UptimeRobot, Datadog) at `/healthz` and alert on three consecutive failures.

## Pre-Launch Checklist

```
Infrastructure
- [ ] .env outside webroot, 600 perms, no secrets in repo
- [ ] Nginx denies /vendor, /src, /config, /tests
- [ ] Directory listing off, expose_php off
- [ ] TLS valid, HSTS enabled, A+ on SSL Labs
- [ ] Security headers served on every response
- [ ] OPcache enabled, validate_timestamps=0, reload on deploy
- [ ] PHP-FPM pool sized for RAM, pm.max_requests > 0
- [ ] MySQL slow query log enabled
- [ ] fail2ban (or equivalent) on sshd and nginx auth endpoints

Deploy
- [ ] bin/deploy.sh is idempotent and runs from CI
- [ ] rsync + symlink swap is atomic
- [ ] Last 5 releases kept on disk for instant rollback
- [ ] Migrations run inside the deploy; down-SQL exists for each
- [ ] CDN purge runs after every production release

Observability
- [ ] /healthz returns 200 and is excluded from cache
- [ ] Uptime monitoring alerts on 3 consecutive failures
- [ ] Error logs (PHP, Nginx, slow query) shipped to a central viewer
- [ ] Deploy hook posts to the team chat channel
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "It's a small site, simple FTP is fine" | FTP deploys are non-atomic and unreviewable. Use rsync + symlink from day one. |
| "I'll secure the server later" | "Later" is after the first compromise. Harden before go-live. |
| "The CDN handles everything" | The CDN caches what you tell it to. Misconfigured headers serve stale or private data. |
| "OPcache makes deploys complicated" | `systemctl reload php-fpm` is one line. The throughput win is worth it. |
| "One server is enough" | One server is a single point of failure. Even if you run one, automate the deploy so a replacement is 10 minutes away. |
| "We'll tune PHP-FPM if it gets slow" | Under-sized pools cause 502s on traffic spikes. Size them during provisioning. |

## Red Flags

- Webroot pointing at the repo root instead of `public/`
- `.env` inside the webroot or committed to the repo
- Missing `validate_timestamps=0` on production OPcache
- No release directories — deploys overwrite `current` in place
- No `down.sql` for migrations
- CDN caching authenticated pages (check `X-Cache-Status` with a logged-in cookie)
- `display_errors=On` in production
- sshd password authentication enabled
- `sudo` without an allowlist — deploy user can restart anything

## Verification

After provisioning or deploying:

- [ ] `curl -I https://app.example.com/` shows security headers
- [ ] `curl -I https://cdn.example.com/assets/app.abc123.js` shows 1-year cache
- [ ] Logging in, then requesting a normally-cached page, returns `X-Cache-Status: BYPASS`
- [ ] `php -i | grep opcache.validate_timestamps` shows `0`
- [ ] `systemctl status php8.2-fpm nginx` both active
- [ ] `/healthz` returns 200 and is not cached
- [ ] Rollback was tested (symlink swap to previous release works)
- [ ] Previous 5 releases are present under `releases/`
