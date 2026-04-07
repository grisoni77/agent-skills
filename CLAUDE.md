# agent-skills

This is the agent-skills project — a collection of production-grade engineering skills for AI coding agents.

## Project Structure

```
skills/       → Core skills (SKILL.md per directory)
agents/       → Reusable agent personas (code-reviewer, test-engineer, security-auditor)
hooks/        → Session lifecycle hooks
.claude/commands/ → Slash commands (/spec, /plan, /build, /test, /review, /code-simplify, /ship)
references/   → Supplementary checklists (testing, performance, security, accessibility)
docs/         → Setup guides for different tools
```

## Target Stack

The skills in this pack are written for and tested against the following
stack. Examples in every SKILL.md speak this stack's idioms:

- **Language / runtime:** PHP 8.2+, `declare(strict_types=1)` by default
- **Framework:** Slim 4 (PSR-15) or a vanilla front controller
- **Database:** MySQL 8 via PDO; versioned SQL migrations
- **Templating:** Smarty 4
- **Frontend:** vanilla JS modules, jQuery on legacy pages, Vue 3 SFC islands
- **Dependency manager:** Composer (PSR-4 autoloading)
- **Tests:** PHPUnit (primary), Pest (optional), Jest/Vitest for JS-only modules
- **Quality gates:** `phpcs` (PSR-12), `phpstan`, `composer audit`
- **CI:** Bitbucket Pipelines
- **Runtime / deploy:** LEMP (Nginx + PHP-FPM + MySQL) behind Akamai / Cloudflare

## Skills by Phase

**Define:** spec-driven-development
**Plan:** planning-and-task-breakdown
**Build:** incremental-implementation, test-driven-development, context-engineering, frontend-ui-engineering, api-and-interface-design, php-backend-engineering, database-design-and-optimization
**Verify:** browser-testing-with-devtools, debugging-and-error-recovery
**Review:** code-review-and-quality, code-simplification, security-and-hardening, performance-optimization, legacy-code-refactoring
**Ship:** git-workflow-and-versioning, ci-cd-and-automation, deprecation-and-migration, documentation-and-adrs, shipping-and-launch, server-deployment-lemp

## Conventions

- Every skill lives in `skills/<name>/SKILL.md`
- YAML frontmatter with `name` and `description` fields
- Description starts with what the skill does (third person), followed by trigger conditions ("Use when...")
- Every skill has: Overview, When to Use, Process, Common Rationalizations, Red Flags, Verification
- References are in `references/`, not inside skill directories
- Supporting files only created when content exceeds 100 lines

## Commands

This repo is documentation only — there is no PHP runtime here. Validate by
checking that every `skills/*/SKILL.md` has valid YAML frontmatter with
`name` + `description`, ≤ 500 lines, and the six required sections.

When agents apply these skills in a **consuming PHP project**, the canonical
commands are:

- `composer install --no-interaction` — install dependencies
- `./vendor/bin/phpunit` — run the test suite
- `./vendor/bin/phpcs --standard=PSR12 src/ tests/` — lint
- `./vendor/bin/phpstan analyse --memory-limit=1G` — static analysis
- `composer audit` — security audit

## Boundaries

- Always: Follow the skill-anatomy.md format for new skills
- Never: Add skills that are vague advice instead of actionable processes
- Never: Duplicate content between skills — reference other skills instead
