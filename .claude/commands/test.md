---
description: Run TDD workflow — write failing tests, implement, verify. For bugs, use the Prove-It pattern.
---

Invoke the agent-skills:test-driven-development skill.

For new features:
1. Write PHPUnit tests that describe the expected behavior (they should FAIL)
2. Implement the code to make them pass
3. Refactor while keeping tests green

For bug fixes (Prove-It pattern):
1. Write a PHPUnit test that reproduces the bug (must FAIL)
2. Confirm the test fails via `./vendor/bin/phpunit --filter <name>`
3. Implement the fix
4. Confirm the test passes
5. Run `./vendor/bin/phpunit` for the full suite to catch regressions

For Smarty pages, jQuery widgets, or Vue islands, also invoke
agent-skills:browser-testing-with-devtools to verify with Chrome DevTools MCP.
