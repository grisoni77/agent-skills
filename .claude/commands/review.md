---
description: Conduct a five-axis code review — correctness, readability, architecture, security, performance
---

Invoke the agent-skills:code-review-and-quality skill.

Review the current changes (staged or recent commits) across all five axes:

1. **Correctness** — Does it match the spec? Edge cases handled? PHPUnit coverage adequate? `./vendor/bin/phpunit` green?
2. **Readability** — Clear names? Straightforward logic? PSR-12 compliant? `./vendor/bin/phpcs` clean?
3. **Architecture** — Thin controllers, fat services, I/O-only repositories? Matches agent-skills:php-backend-engineering? `./vendor/bin/phpstan analyse` clean?
4. **Security** — PDO prepared statements everywhere? Smarty `|escape` on every output? CSRF tokens present? `composer audit` clean? (Use agent-skills:security-and-hardening)
5. **Performance** — No N+1 queries? `EXPLAIN` checked on new queries? No unbounded ops? (Use agent-skills:performance-optimization and agent-skills:database-design-and-optimization)

Categorize findings as Critical, Important, or Suggestion.
Output a structured review with specific file:line references and fix recommendations.
