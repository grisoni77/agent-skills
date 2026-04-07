---
description: Implement the next task incrementally — build, test, verify, commit
---

Invoke the agent-skills:incremental-implementation skill alongside
agent-skills:test-driven-development. For backend work, also load
agent-skills:php-backend-engineering and, when touching the database,
agent-skills:database-design-and-optimization.

Pick the next pending task from the plan. For each task:

1. Read the task's acceptance criteria
2. Load relevant context (existing code, patterns, types)
3. Write a failing PHPUnit test for the expected behavior (RED)
4. Implement the minimum code to pass the test (GREEN)
5. Run `./vendor/bin/phpunit` to check for regressions
6. Run `./vendor/bin/phpcs` and `./vendor/bin/phpstan analyse` to verify quality gates
7. Commit with a descriptive message
8. Mark the task complete and move to the next one

If any step fails, follow the agent-skills:debugging-and-error-recovery skill.
