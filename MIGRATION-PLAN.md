# Migration Plan — agent-skills → PHP / MySQL / LEMP stack

This document maps every asset in the upstream `agent-skills` repository
(originally written for a TypeScript / Node / React / Vercel stack) to the
target stack used in production:

- **Backend:** PHP 8.x (Slim or vanilla PHP depending on the project)
- **Database:** MySQL 8.x (InnoDB)
- **Frontend:** vanilla JavaScript, jQuery, some Vue.js islands
- **Templating:** Smarty + server-rendered HTML
- **Server:** LEMP (Nginx, Linux) behind Akamai / Cloudflare
- **Tooling:** Composer, npm (assets only), Git
- **Testing:** PHPUnit / Pest, occasional Jest for JS
- **CI/CD:** Bitbucket Pipelines + git hooks

The plan is split into four execution phases. **Phase 1 stops here** and
waits for explicit user approval of this document before any SKILL.md is
touched.

---

## 1. Skill inventory and compatibility matrix

Legend: ✅ usable as-is · 🔧 adapt · ❌ rewrite · ➕ create new

| # | Skill | Class | Notes |
|---|---|---|---|
| 1 | `idea-refine` | ✅ | Stack-agnostic; no code examples tied to a language. |
| 2 | `spec-driven-development` | 🔧 | Replace the single `npm test` example with `./vendor/bin/phpunit`; keep the Commands section list-shaped. |
| 3 | `planning-and-task-breakdown` | 🔧 | Swap minor `npm` / Go references; examples become PHP feature slices. |
| 4 | `incremental-implementation` | ❌ | Rewrite slice examples around Composer + PHPUnit; replace "build green" checks with `composer install` + `./vendor/bin/phpunit`; drop `tsc`. |
| 5 | `test-driven-development` | ❌ | Port all test snippets from Jest / Vitest to PHPUnit (default) and Pest (alt). Keep the Prove-It pattern. Add a short JS sidebar for Jest when touching front-end code. |
| 6 | `context-engineering` | ❌ | Rewrite project-layout and rules-file examples for a PHP / Composer / Smarty repo. Keep the hierarchy and trust-level principles. |
| 7 | `frontend-ui-engineering` | ❌ | Replace React / Tailwind / Zustand with: Smarty templates, vanilla JS modules, jQuery patterns, Vue SFC islands. Keep design-system and WCAG 2.1 AA content. |
| 8 | `api-and-interface-design` | 🔧 | Convert TS / Go signatures to PHP 8 typed method signatures + Slim route handlers + JSON response contracts. Keep Hyrum's Law and error-semantics sections. |
| 9 | `browser-testing-with-devtools` | 🔧 | Keep Chrome DevTools MCP workflow. Reframe the examples around server-rendered Smarty pages and jQuery event handlers instead of React component state. |
| 10 | `debugging-and-error-recovery` | 🔧 | Replace Node / V8 stack-trace examples with PHP stack traces, `error_log`, Xdebug step-through, and MySQL error codes. Triage framework stays. |
| 11 | `code-review-and-quality` | 🔧 | Rewrite code examples in PHP; add PSR-12 and `phpcs` references. Five-axis framework stays verbatim. |
| 12 | `code-simplification` | 🔧 | Port example snippets to PHP; Rule of 500 / Chesterton stay. |
| 13 | `security-and-hardening` | ❌ | OWASP and three-tier boundary model stay. Replace `npm audit` with `composer audit`. Add PHP-specific sub-sections: prepared statements (PDO/mysqli), file upload validation, session fixation, CSRF tokens, `filter_var`, `htmlspecialchars` output encoding. |
| 14 | `performance-optimization` | ❌ | Keep Core Web Vitals. Add server-side sections: MySQL `EXPLAIN`, slow query log, OPcache tuning, Nginx fastcgi_cache, Akamai / Cloudflare cache headers. Replace Node profiler examples with Xdebug profiler and `mysqldumpslow`. |
| 15 | `git-workflow-and-versioning` | 🔧 | Drop `npm version` reference. Trunk-based content and atomic-commit guidance stay. |
| 16 | `ci-cd-and-automation` | ❌ | Full rewrite of the pipeline YAML: GitHub Actions → Bitbucket Pipelines. Replace `npm ci` + Prisma migrate with `composer install --no-dev -o` + manual SQL migrations. Keep Shift-Left and quality-gate principles. |
| 17 | `deprecation-and-migration` | ✅ | Fully stack-agnostic. |
| 18 | `documentation-and-adrs` | 🔧 | Replace the one TypeScript example with a PHP class doc comment. ADR format is unchanged. |
| 19 | `shipping-and-launch` | ❌ | Rewrite pre-launch checklist around LEMP + rsync/symlink deploys + Bitbucket pipeline promotion + Akamai purge. Feature-flag lifecycle stays. |
| 20 | `using-agent-skills` | ✅ | Meta-skill. Only its skill inventory list needs to be updated once new skills are added in Phase 3. |
| 21 | `database-design-and-optimization` | ➕ | **New.** MySQL 8 schema design, indexes, `EXPLAIN`, N+1, manual migrations, backups. |
| 22 | `php-backend-engineering` | ➕ | **New.** PHP 8 MVC, Service/Repository pattern, Composer + PSR-4, PSR-12, error handling, sessions, auth, middleware. |
| 23 | `server-deployment-lemp` | ➕ | **New.** LEMP pre-deploy checklist behind Akamai/Cloudflare. |
| 24 | `legacy-code-refactoring` | ➕ | **New.** Characterization tests, strangler fig, procedural → OOP, gradual typing and namespacing. |

Totals: 2 ✅ · 9 🔧 · 9 ❌ · 4 ➕ = **24 skills** after migration.

---

## 2. New skills (Phase 3 scope)

Each new SKILL.md must follow `docs/skill-anatomy.md` exactly:
YAML frontmatter (`name`, `description` starting with what-it-does + "Use
when…", ≤ 1024 chars), then Overview, When to Use, Core Process,
Techniques/Patterns, Common Rationalizations (table), Red Flags, Verification.
Each file ≤ 500 lines; overflow goes into sibling `.md` supporting files in
the same skill directory.

### 2.1 `skills/database-design-and-optimization/SKILL.md`
- MySQL 8 schema design: normalization vs denormalization, data-type
  selection, charset / collation, InnoDB specifics.
- Index strategy: single-column, composite, covering, prefix; when not to
  index; cardinality considerations.
- Query optimization: `EXPLAIN` / `EXPLAIN ANALYZE`, slow query log,
  `performance_schema`, query rewriting.
- N+1 detection and fixes: batch loading, joins, identity map.
- Manual migration workflow: versioned SQL files, up/down scripts, the
  "no transactional DDL" caveat, online schema changes.
- Backup & recovery: `mysqldump`, binary log, point-in-time restore.
- **Rationalizations:** "I'll optimize later" · "We don't need an index
  yet" · "The database is small".

### 2.2 `skills/php-backend-engineering/SKILL.md`
- MVC & separation of concerns in PHP 8; folder layout for Slim and for
  vanilla/legacy projects.
- Service / Repository pattern; DI container basics (Slim / PHP-DI) and
  the simplified vanilla variant.
- Composer + PSR-4 autoloading; `require` fallback pattern for legacy
  codebases.
- Structured error handling and logging (Monolog preferred, `error_log`
  fallback); exception hierarchy.
- Sessions, authentication (password_hash / password_verify), middleware
  (PSR-15 via Slim or a hand-rolled chain).
- PSR-12 enforcement with `phpcs`; static analysis with `phpstan`.
- **Rationalizations:** "It's just a quick script" · "Namespaces
  complicate everything" · "No service layer needed for this".

### 2.3 `skills/server-deployment-lemp/SKILL.md`
- Pre-deploy checklist for LEMP: Nginx site config, PHP-FPM pool sizing,
  OPcache, filesystem permissions, `open_basedir`, `disable_functions`.
- Cache-layer interaction with Akamai / Cloudflare: `Cache-Control`,
  `Surrogate-Control`, cache-busting strategies, purge on release, bypass
  cookies, staging host headers.
- Secrets and env handling (`.env` outside webroot, server env vars,
  `php-fpm` env).
- Zero-downtime deploy: rsync + atomic symlink swap, PHP-FPM reload, cache
  warm-up.
- **Rationalizations:** "It's a small site, no need to harden" · "I'll
  secure the server later".

### 2.4 `skills/legacy-code-refactoring/SKILL.md`
- Strategy map for legacy PHP: assess, isolate, test, refactor.
- Characterization tests with PHPUnit against untested procedural code;
  golden-master technique.
- Strangler fig pattern for incremental migration of routes / includes
  behind a new front controller.
- Gradual introduction of namespaces, autoloading, type hints, and
  `declare(strict_types=1)`.
- Safe seam creation; dependency injection as a refactoring tool.
- **Rationalizations:** "It works, don't touch it" · "Let's rewrite from
  scratch" · "No time for tests on old code".

---

## 3. Configuration changes (Phase 4 scope)

### 3.1 Root configuration files

- `CLAUDE.md`
  - Rewrite **Project Structure** to describe the post-migration layout.
  - Replace **Commands** section: `composer install`, `composer test`,
    `./vendor/bin/phpunit`, `./vendor/bin/pest`, `phpcs`, `phpstan`,
    `eslint` (JS only).
  - Add the 4 new skills to the **Skills by Phase** map.
  - Update **Boundaries** to mention PHP-specific anti-patterns (SQL via
    string interpolation, `==` vs `===`, untyped signatures).

- `AGENTS.md`
  - Add a **Stack context** section: PHP 8 / MySQL 8 / vanilla JS + jQuery
    + Vue / Smarty / LEMP / Composer / PHPUnit / Bitbucket Pipelines.
  - Keep the skill-creation conventions (naming, skill-anatomy reference).

### 3.2 Slash commands (`.claude/commands/*.md`)

- `/spec`, `/plan`: no structural change, update example commands from
  `npm test` → `./vendor/bin/phpunit`.
- `/build`: reference `php-backend-engineering` and
  `database-design-and-optimization` alongside TDD / incremental-impl.
- `/test`: point to PHPUnit / Pest; keep browser-testing reference.
- `/review`: add a bullet pointing at `legacy-code-refactoring` when
  touching legacy modules.
- `/code-simplify`: unchanged logic, just PHP-flavoured examples.
- `/ship`: reference `server-deployment-lemp`.

### 3.3 Agents (`agents/*.md`)

- `code-reviewer.md` — Add PHP-sensitive checks:
  - SQL built via string interpolation / missing prepared statements
  - Loose comparison (`==`) where strict (`===`) is required
  - Missing type hints / return types
  - PSR-12 violations
  - Uncaught exceptions and silent `@` error suppression
  - Output encoding (`htmlspecialchars`) in Smarty escape modifiers

- `security-auditor.md` — Add PHP-specific checks:
  - Prepared statements (PDO / mysqli) everywhere user input touches SQL
  - File upload validation (MIME sniff, extension allowlist, storage
    outside webroot)
  - Session fixation / regeneration on privilege change
  - CSRF tokens on state-changing requests
  - Legacy `register_globals`, `magic_quotes`, `extract($_POST)`
  - `password_hash` / `password_verify` usage
  - Secrets in repo / `.env` outside webroot

- `test-engineer.md` — Add PHP testing strategies:
  - PHPUnit and Pest structure and conventions
  - Database testing with transactions + rollback per test
  - Test doubles for repositories and external services
  - Fixture management and seeding

### 3.4 References (`references/*.md`)

- `testing-patterns.md` — Add PHPUnit / Pest patterns, DB test isolation,
  and a short jQuery / Vue testing note. Keep Playwright section.
- `security-checklist.md` — Add PHP items and replace `npm audit` with
  `composer audit`.
- `performance-checklist.md` — Add MySQL `EXPLAIN`, OPcache, Nginx cache,
  Akamai/Cloudflare headers.
- `accessibility-checklist.md` — **unchanged** (WCAG is stack-agnostic).

---

## 4. Adaptation cheatsheet (applied during Phase 2)

Apply these swaps consistently to every 🔧 and ❌ skill:

| Upstream token | Replacement |
|---|---|
| `npm test` | `./vendor/bin/phpunit` (or `./vendor/bin/pest`) |
| `npm run build` | `composer install --no-dev -o` (or project-specific build) |
| `npm ci` | `composer install --no-dev --prefer-dist` |
| `tsc` / TypeScript type checks | *removed* |
| `eslint` (for PHP files) | `phpcs` + `phpstan` |
| `eslint` (for JS files) | kept as-is |
| `npm audit` | `composer audit` |
| `package.json` | `composer.json` |
| GitHub Actions YAML | Bitbucket Pipelines YAML |
| `React` / `useState` / `Zustand` | Smarty templates + vanilla JS + jQuery + Vue SFC islands |
| Prisma migrations | Versioned SQL migration files |
| Vercel deploy | rsync + symlink swap on LEMP + Akamai purge |

---

## 5. Commit strategy

One commit per phase, all on `claude/setup-agent-skills-PxXhq`:

1. `docs: add MIGRATION-PLAN for PHP/MySQL stack adaptation` *(Phase 1)*
2. `refactor(skills): adapt existing skills to PHP/MySQL stack` *(Phase 2)*
3. `feat(skills): add PHP/MySQL/LEMP/legacy skills` *(Phase 3)*
4. `chore(config): update CLAUDE.md, agents and commands for PHP stack` *(Phase 4)*

Final push: `git push -u origin claude/setup-agent-skills-PxXhq`. No PR is
opened unless the user explicitly asks for one.

---

## 6. Verification checklist (run at the end of Phase 4)

- [ ] Every `skills/*/SKILL.md` (all 24) has valid YAML frontmatter with
      `name` + `description`, and `name` matches its directory.
- [ ] Every SKILL.md contains the six required sections (Overview, When to
      Use, Core Process, Techniques/Patterns, Common Rationalizations,
      Red Flags, Verification).
- [ ] No remaining `npm test`, `tsc`, `next build`, `vercel`, `React`,
      `useState`, `Prisma` tokens in adapted skills (grep).
- [ ] The four new skill directories exist under `skills/` with correct
      kebab-case names.
- [ ] `CLAUDE.md` phase map lists all 24 skills.
- [ ] `git status` is clean at each commit boundary.
- [ ] Branch `claude/setup-agent-skills-PxXhq` pushed to origin.
