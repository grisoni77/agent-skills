---
name: ci-cd-and-automation
description: Automates CI/CD pipeline setup. Use when setting up or modifying build and deployment pipelines. Use when you need to automate quality gates, configure test runners in CI, or establish deployment strategies.
---

# CI/CD and Automation

## Overview

Automate quality gates so that no change reaches production without passing tests, lint, static analysis, and security audit. CI/CD is the enforcement mechanism for every other skill — it catches what humans and agents miss, and it does so consistently on every single change.

**Shift Left:** Catch problems as early in the pipeline as possible. A bug caught in `phpcs` costs minutes; the same bug caught in production costs hours. Move checks upstream — static analysis before tests, tests before staging, staging before production.

**Faster is Safer:** Smaller batches and more frequent releases reduce risk, not increase it. A deployment with 3 changes is easier to debug than one with 30. Frequent releases build confidence in the release process itself.

## When to Use

- Setting up a new project's CI pipeline
- Adding or modifying automated checks
- Configuring deployment pipelines
- When a change should trigger automated verification
- Debugging CI failures

## The Quality Gate Pipeline

Every change goes through these gates before merge:

```
Pull Request Opened
    │
    ▼
┌─────────────────────┐
│   LINT (phpcs)       │  PSR-12
│   ↓ pass             │
│   STATIC (phpstan)   │  level 6+ / psalm
│   ↓ pass             │
│   UNIT TESTS         │  PHPUnit / Pest
│   ↓ pass             │
│   INTEGRATION        │  Slim routes + MySQL test DB
│   ↓ pass             │
│   BUILD              │  composer install --no-dev -o
│   ↓ pass             │
│   SECURITY AUDIT     │  composer audit
│   ↓ pass             │
│   E2E (optional)     │  DevTools MCP / Playwright
└─────────────────────┘
    │
    ▼
  Ready for review
```

**No gate can be skipped.** If `phpcs` fails, fix the style — don't suppress the rule. If a test fails, fix the code — don't skip the test.

## Bitbucket Pipelines Configuration

### Basic CI Pipeline

```yaml
# bitbucket-pipelines.yml
image: php:8.2-cli

definitions:
  caches:
    composer: ~/.composer/cache
  steps:
    - step: &quality
        name: Quality gates
        caches:
          - composer
        script:
          - apt-get update && apt-get install -y git unzip libzip-dev libicu-dev
          - docker-php-ext-install zip intl pdo_mysql
          - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
          - composer install --no-interaction --prefer-dist
          - ./vendor/bin/phpcs --standard=PSR12 src/
          - ./vendor/bin/phpstan analyse --memory-limit=1G
          - ./vendor/bin/phpunit --coverage-text
          - composer audit

pipelines:
  pull-requests:
    '**':
      - step: *quality
  branches:
    main:
      - step: *quality
```

### With Database Integration Tests

Bitbucket Pipelines supports service containers for MySQL:

```yaml
definitions:
  services:
    mysql:
      image: mysql:8.0
      variables:
        MYSQL_DATABASE: testdb
        MYSQL_ROOT_PASSWORD: $CI_DB_PASSWORD   # from repository secrets
        MYSQL_USER: ci_user
        MYSQL_PASSWORD: $CI_DB_PASSWORD

pipelines:
  pull-requests:
    '**':
      - step:
          name: Integration tests
          image: php:8.2-cli
          services:
            - mysql
          caches:
            - composer
          script:
            - docker-php-ext-install pdo_mysql
            - composer install --no-interaction --prefer-dist
            - php bin/migrate.php up           # run versioned SQL migrations
            - ./vendor/bin/phpunit --testsuite=integration
          variables:
            DB_HOST: 127.0.0.1
            DB_NAME: testdb
            DB_USER: ci_user
```

> **Note:** Store `CI_DB_PASSWORD` and any other secret in the Bitbucket
> repository variables UI, never in `bitbucket-pipelines.yml`.

### E2E Tests

```yaml
- step:
    name: End-to-end
    image: mcr.microsoft.com/playwright:v1.45.0-jammy
    script:
      - apt-get update && apt-get install -y php-cli php-mysql
      - php -S 127.0.0.1:8080 -t public &
      - sleep 2
      - npx playwright test
    artifacts:
      - playwright-report/**
```

### Parallel Steps for Speed

```yaml
pipelines:
  pull-requests:
    '**':
      - parallel:
          - step:
              name: Lint
              script:
                - composer install --no-interaction --prefer-dist
                - ./vendor/bin/phpcs --standard=PSR12 src/
          - step:
              name: Static analysis
              script:
                - composer install --no-interaction --prefer-dist
                - ./vendor/bin/phpstan analyse --memory-limit=1G
          - step:
              name: Unit tests
              script:
                - composer install --no-interaction --prefer-dist
                - ./vendor/bin/phpunit --testsuite=unit
```

## Feeding CI Failures Back to Agents

The power of CI with AI agents is the feedback loop. When CI fails:

```
CI fails
    │
    ▼
Copy the failure output
    │
    ▼
Feed it to the agent:
"The Bitbucket Pipeline failed with this error:
[paste specific error]
Fix the issue and verify locally with ./vendor/bin/phpunit
and ./vendor/bin/phpstan analyse before pushing."
    │
    ▼
Agent fixes → pushes → pipeline runs again
```

**Key patterns:**

```
phpcs failure   → Agent runs `./vendor/bin/phpcbf src/` and commits the fix
phpstan failure → Agent reads the error location and fixes the type
Test failure    → Agent follows debugging-and-error-recovery skill
Build error     → Agent checks composer.json and extensions
```

## Deployment Strategies

### Staging Deploy on merge to main

```yaml
pipelines:
  branches:
    main:
      - step: *quality
      - step:
          name: Deploy to staging
          deployment: staging
          script:
            - apt-get update && apt-get install -y rsync openssh-client
            - composer install --no-dev --optimize-autoloader --no-interaction
            - RELEASE="releases/$(date +%Y%m%d%H%M%S)-$BITBUCKET_COMMIT"
            - ssh $DEPLOY_USER@$STAGING_HOST "mkdir -p /var/www/app/$RELEASE"
            - rsync -az --delete --exclude='.git' --exclude='tests' ./ $DEPLOY_USER@$STAGING_HOST:/var/www/app/$RELEASE/
            - ssh $DEPLOY_USER@$STAGING_HOST "ln -sfn /var/www/app/$RELEASE /var/www/app/current && sudo systemctl reload php8.2-fpm && sudo nginx -s reload"
            # Purge the Akamai/Cloudflare cache for the release
            - curl -X POST "https://api.cloudflare.com/client/v4/zones/$CF_ZONE/purge_cache" -H "Authorization: Bearer $CF_API_TOKEN" -H "Content-Type: application/json" --data '{"purge_everything":true}'
```

Key properties:

- **Atomic symlink swap** — the `current` symlink is updated last, so requests either hit the old release or the new one, never a half-copied tree
- **Release directories kept** — roll back by swapping the symlink to a previous release
- **CDN purge** — Akamai / Cloudflare cached pages must be invalidated after the swap

### Production Deploy (manual promotion)

```yaml
      - step:
          name: Deploy to production
          deployment: production
          trigger: manual             # click-to-deploy from main
          script:
            - # same rsync + symlink + purge, targeting $PROD_HOST
```

### Feature Flags

Feature flags decouple deployment from release. Deploy incomplete or risky features behind flags so you can:

- **Ship code without enabling it** — merge to main early, enable when ready
- **Roll back without redeploying** — disable the flag instead of reverting
- **Canary new features** — enable for 1% of users, then 10%, then 100%
- **Run A/B tests**

```php
// config/features.php — committed, env-overridable
return [
    'task_sharing'       => (bool) (getenv('FEATURE_TASK_SHARING') ?: false),
    'new_checkout_flow'  => (bool) (getenv('FEATURE_NEW_CHECKOUT') ?: false),
];

// Usage
if ($this->features->isEnabled('new_checkout_flow', userId: $userId)) {
    return $this->renderNewCheckout();
}
return $this->renderLegacyCheckout();
```

**Flag lifecycle:** create → enable for testing → canary → full rollout → remove flag and dead code. Flags that live forever become technical debt — set a cleanup date at creation.

### Staged Rollouts

```
PR merged to main
    │
    ▼
  Staging deployment (auto)
    │ Manual smoke test
    ▼
  Production deployment (manual trigger)
    │
    ▼
  Monitor for errors (15-minute window)
    │
    ├── Errors detected → swap symlink back to previous release
    └── Clean → done
```

### Rollback Plan

Every deployment must be reversible. With the release-directory + symlink
scheme, rollback is one SSH command:

```bash
ssh deploy@prod "ln -sfn /var/www/app/releases/<previous> /var/www/app/current && sudo systemctl reload php8.2-fpm"
# then purge the CDN again
```

For DB migrations, keep an explicit down-SQL file alongside every up-SQL
file. Additive migrations (new table, new nullable column) are safe to leave
in place even after an application rollback — see `database-design-and-optimization`.

## Environment Management

```
.env.example       → committed (template for developers)
.env                → NOT committed, lives outside webroot
.env.test           → committed, no real secrets
CI secrets          → Bitbucket repository variables (masked)
Production secrets  → server-side .env outside webroot, 600 perms
```

CI should never have production secrets. Use separate secrets for CI testing.

## Automation Beyond CI

### Dependency Updates

Use Bitbucket's scheduled pipelines or a Renovate bot to run
`composer outdated` weekly and open PRs for minor/patch updates:

```yaml
pipelines:
  custom:
    weekly-deps:
      - step:
          name: Check for outdated dependencies
          script:
            - composer outdated --direct --strict
            - composer audit
```

Schedule it via the Bitbucket "Schedules" UI against the `main` branch.

### Build Cop Role

Designate someone responsible for keeping the main branch green. When the
pipeline breaks, the Build Cop's job is to fix or revert — not the person
whose change caused the break. This prevents broken builds from accumulating
while everyone assumes someone else will fix them.

### PR Checks

- **Required reviewers:** at least one approval before merge
- **Required builds:** Bitbucket "Require successful build" branch permission on `main`
- **Branch permissions:** no direct pushes or force-pushes to `main`
- **Auto-merge:** if all checks pass and approved

## CI Optimization

When the pipeline exceeds 10 minutes, apply these strategies in order of impact:

```
Slow pipeline?
├── Cache composer and docker layers
│   └── definitions.caches.composer, plus custom caches for vendor/
├── Run steps in parallel
│   └── parallel block for lint / phpstan / phpunit
├── Skip unrelated jobs
│   └── Use condition: changesets for path-filtered steps
├── Shard the test suite
│   └── Split PHPUnit by testsuite or directory across parallel steps
├── Trim slow tests
│   └── Move slow integration tests to a nightly scheduled pipeline
└── Use larger runners
    └── size: 2x for CPU-heavy static analysis
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "CI is too slow" | Optimize the pipeline, don't skip it. A 5-minute pipeline prevents hours of debugging. |
| "This change is trivial, skip CI" | Trivial changes break builds. CI is fast for trivial changes anyway. |
| "The test is flaky, just re-run" | Flaky tests mask real bugs and waste everyone's time. Fix the flakiness. |
| "We'll add CI later" | Projects without CI accumulate broken states. Set it up on day one. |
| "Manual testing is enough" | Manual testing doesn't scale and isn't repeatable. Automate what you can. |
| "Deploys via FTP are fine" | FTP deploys are non-atomic and unreviewable. Use rsync + symlink from CI. |

## Red Flags

- No CI pipeline in the project
- CI failures ignored or silenced
- Tests disabled in CI to make the pipeline pass
- Production deploys without staging verification
- No rollback mechanism (no previous release directory kept)
- Secrets stored in `bitbucket-pipelines.yml` (not in repository variables)
- Long pipeline times with no optimization effort
- Deploys that run database migrations without a rollback SQL file

## Verification

After setting up or modifying CI:

- [ ] All quality gates present: `phpcs`, `phpstan`, `phpunit`, `composer audit`
- [ ] Pipeline runs on every PR and push to `main`
- [ ] Failures block merge (branch permission configured)
- [ ] CI results feed back into the development loop
- [ ] Secrets are stored in Bitbucket repository variables, not in code
- [ ] Deployment is atomic (rsync + symlink) and reversible
- [ ] CDN purge step runs after every production deploy
- [ ] Pipeline runs in under 10 minutes for the critical path
