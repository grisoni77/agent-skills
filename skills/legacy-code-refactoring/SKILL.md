---
name: legacy-code-refactoring
description: Refactors legacy PHP codebases incrementally. Use when working on old procedural PHP, untyped / untested code, or tangled include-chains. Use when modernizing toward namespaces, Composer, typed classes, and testability without rewriting from scratch.
---

# Legacy Code Refactoring

## Overview

Modernize legacy PHP codebases **without rewriting them from scratch**. Rewrites almost always fail — they underestimate the undocumented behavior baked into the existing code and leave the business with a half-working new system and a still-running old one. Instead, use characterization tests and the strangler-fig pattern to replace the legacy code piece by piece while it keeps serving production.

## When to Use

- Working on pre-PSR PHP (no namespaces, no Composer, no types)
- A codebase with `require_once` chains and global variables
- Untested code that has to keep running
- Procedural scripts that need new features
- Gradual migration toward PSR-4 / PSR-12 / typed properties
- Any change where "just rewrite it" is being proposed

## Core Principles

```
1. UNDERSTAND BEFORE CHANGING  → Chesterton's fence, git blame, read the caller
2. PIN BEHAVIOR WITH TESTS      → Characterization tests before refactoring
3. STRANGLER FIG                → New code alongside old, route traffic gradually
4. ADDITIVE FIRST               → Add the new path before removing the old
5. SMALL SAFE STEPS             → Each commit passes tests in isolation
```

Never combine "understand," "test," and "rewrite" into one commit. Each is its own PR.

## Characterization Tests

A **characterization test** (a term from Michael Feathers' *Working Effectively with Legacy Code*) pins down what the code currently does, not what it should do. The goal is a safety net, not a specification.

### Process

```
1. Pick a public entry point (function, endpoint, class method)
2. Call it with realistic inputs
3. Capture the output exactly — strings, arrays, HTML, DB side effects
4. Write a PHPUnit test that asserts on that captured output
5. Run it → GREEN → you can now refactor
```

### Example: legacy function

```php
// legacy/inc/tasks.php — procedural, global $db, no types
function build_task_list_html($user_id, $status = 'all') {
    global $db;
    $where = "owner_id = " . intval($user_id);
    if ($status !== 'all') {
        $where .= " AND status = '" . mysqli_real_escape_string($db, $status) . "'";
    }
    $res  = mysqli_query($db, "SELECT id, title, status FROM tasks WHERE $where ORDER BY id DESC");
    $html = '<ul>';
    while ($row = mysqli_fetch_assoc($res)) {
        $html .= '<li class="' . htmlspecialchars($row['status']) . '">' . htmlspecialchars($row['title']) . '</li>';
    }
    return $html . '</ul>';
}
```

The characterization test captures current behavior, including its quirks:

```php
final class BuildTaskListHtmlTest extends TestCase
{
    protected function setUp(): void
    {
        require_once __DIR__ . '/../../legacy/inc/tasks.php';
        $GLOBALS['db'] = TestDatabase::mysqli();
        TestDatabase::seedTasks([
            ['id' => 1, 'owner_id' => 42, 'title' => 'Buy milk', 'status' => 'pending'],
            ['id' => 2, 'owner_id' => 42, 'title' => 'Call mom', 'status' => 'done'],
            ['id' => 3, 'owner_id' => 99, 'title' => 'Other',    'status' => 'pending'],
        ]);
    }

    public function test_renders_html_list_for_user(): void
    {
        $html = build_task_list_html(42);
        self::assertSame(
            '<ul><li class="done">Call mom</li><li class="pending">Buy milk</li></ul>',
            $html,
        );
    }

    public function test_filters_by_status(): void
    {
        self::assertSame(
            '<ul><li class="pending">Buy milk</li></ul>',
            build_task_list_html(42, 'pending'),
        );
    }
}
```

These tests **are not beautiful**. They pin the current output including any HTML quirks — that is the point. Once they are green, you can change the implementation with confidence.

### Capturing Output for Large Functions

For functions with sprawling output, generate an approval test: run the legacy function once, write its output to a fixture file, then have the test compare against the file. Review the fixture by eye once — after that, any diff is a real behavior change.

## The Strangler Fig Pattern

Named after the strangler fig tree, which grows around a host tree until the host is gone. In code, you build the new implementation alongside the old, route a small share of traffic to the new path, and expand until the old path is unused — then delete it.

### Phases

```
1. WRAP    → Put a seam around the legacy entry point
2. DOUBLE  → New implementation runs in parallel, compared against the old
3. SHIFT   → Flag-controlled traffic split moves to the new path
4. REMOVE  → Once the flag is 100%, delete the old code
```

### Step 1 — Wrap

Introduce a thin façade so callers stop calling the legacy function directly.

```php
// src/Domain/Task/TaskListRenderer.php  (new, PSR-4, typed)
declare(strict_types=1);

namespace App\Domain\Task;

final class TaskListRenderer
{
    public function __construct(private readonly FeatureFlags $flags) {}

    public function render(int $ownerId, string $status = 'all'): string
    {
        if ($this->flags->isEnabled('new_task_list_renderer', $ownerId)) {
            return $this->renderNew($ownerId, $status);
        }
        return build_task_list_html($ownerId, $status); // legacy bridge
    }

    private function renderNew(int $ownerId, string $status): string { /* new impl */ }
}
```

Replace `build_task_list_html(...)` calls across the app with `$renderer->render(...)`. This is **mechanical** — do it in one pass, tests must stay green.

### Step 2 — Double (Shadow Mode)

Run the new implementation alongside the old, compare results, log mismatches:

```php
public function render(int $ownerId, string $status = 'all'): string
{
    $old = build_task_list_html($ownerId, $status);

    try {
        $new = $this->renderNew($ownerId, $status);
        if ($new !== $old) {
            $this->logger->warning('task_list renderer drift', [
                'owner' => $ownerId, 'status' => $status,
                'new_len' => strlen($new), 'old_len' => strlen($old),
            ]);
        }
    } catch (\Throwable $e) {
        $this->logger->error('new renderer failed', ['exception' => $e]);
    }

    return $old; // still returning old while we observe
}
```

Let this run for a day or two. Every drift warning is a missed edge case in the characterization tests — add a test for each and fix the new implementation.

### Step 3 — Shift

When drift is zero, flip the flag to 10% → 50% → 100% (see `shipping-and-launch` for rollout thresholds). Monitor error rates and user reports at each step.

### Step 4 — Remove

Once the flag has been at 100% for a full release cycle:

1. Delete the old function
2. Delete the legacy `require_once` that included it
3. Delete the feature flag
4. Delete the shadow-mode comparison code
5. Delete the bridge in the renderer

The removal commit should be 100% deletions. If it is not, you still had callers depending on the old path.

## Procedural → Object-Oriented

Legacy PHP often has hundreds of free functions in `inc/` files. Convert them incrementally:

### Step 1 — Move to a class, one function at a time

```php
// Before: legacy/inc/users.php
function get_user_by_id($id) { /* ... */ }
function get_user_by_email($e) { /* ... */ }
function create_user($data) { /* ... */ }

// After: src/Domain/User/UserRepository.php
namespace App\Domain\User;

final class UserRepository
{
    public function __construct(private readonly \PDO $pdo) {}
    public function findById(int $id): ?User { /* ... */ }
    public function findByEmail(string $email): ?User { /* ... */ }
    public function create(CreateUserInput $input): User { /* ... */ }
}
```

Keep the legacy function as a one-line bridge during migration:

```php
// legacy/inc/users.php
function get_user_by_id($id) {
    return \App\Container::get(\App\Domain\User\UserRepository::class)->findById((int) $id);
}
```

Callers keep working while you migrate them one by one. When the last caller is gone, delete the bridge.

### Step 2 — Introduce Types Gradually

PHP's `declare(strict_types=1)` is per-file, so you can adopt it file by file. Order of operations:

```
1. Add declare(strict_types=1); to new files only
2. When touching an old file, add return types to its functions
3. Then parameter types
4. Then property types (PHP 7.4+)
5. Finally strict_types at the top of the file
```

PHPStan with a baseline lets you lock in the current error count and only fail on new violations — gradual tightening without blocking daily work:

```bash
./vendor/bin/phpstan analyse --generate-baseline
```

Raise the PHPStan level for the **new** directory (`src/`) independently from the legacy directory (`legacy/`) using per-path configuration.

## Introducing Composer to a Non-Composer Project

```
1. composer init — minimal composer.json
2. composer require — install a tiny dep just to prove loading works
3. Require the autoloader from the front controller BEFORE the legacy bootstrap
4. Autoload "App\\" from src/ (PSR-4)
5. New code goes under src/; old code stays where it is
```

```php
// public/index.php
require __DIR__ . '/../vendor/autoload.php';   // new: Composer autoload
require __DIR__ . '/../legacy/bootstrap.php';  // existing legacy include chain
```

Now you can start writing namespaced classes in `src/` that coexist with the legacy includes. The legacy bridges from the strangler-fig pattern resolve through Composer's autoloader.

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Refactoring without tests first | Write characterization tests first. Always. |
| Rewriting a module in one PR | Ship a wrap/shadow/shift/remove sequence instead. |
| Deleting `require_once` too early | Leave the legacy includes in place until the bridge is gone. |
| Changing behavior and refactoring in the same commit | Split into two commits: "pin behavior" then "refactor." |
| Global `$db` inside new classes | Inject `\PDO` into the constructor; add a bridge for legacy callers. |
| Touching code outside the task scope | Note it, don't fix it — see `incremental-implementation`. |
| Adding strict_types to a file with untyped callers | Expect implicit string→int coercions to break; fix at the boundary. |

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "It works, don't touch it" | Working but untested code will be impossible to fix when it finally breaks. |
| "Let's just rewrite it" | Rewrites almost always fail. Strangle, don't rewrite. |
| "No time for tests on old code" | Every minute spent on characterization tests pays back 10× on the first regression. |
| "Tests are impossible because of globals" | Wrap the globals. That wrapping IS the first refactor. |
| "The new code should be perfect" | The new code must be *better*, not *perfect*. Incremental wins. |
| "We'll delete the old code later" | Later never comes unless you scheduled it with a flag cleanup date. |

## Red Flags

- Refactoring PRs without accompanying tests
- "Big-bang" rewrite branches that live longer than a week
- Tests deleted because "they tested old behavior"
- `declare(strict_types=1)` added to a file whose callers are untyped
- New classes that reach for `$GLOBALS` or `$_SESSION` directly
- Feature flags with no owner and no cleanup date
- Strangler-fig shadow comparisons never promoted past 0%

## Verification

After a legacy refactoring pass:

- [ ] Characterization tests existed **before** any production code changed
- [ ] `./vendor/bin/phpunit` passes, including the characterization suite
- [ ] `./vendor/bin/phpstan analyse` level is not regressed (baseline honored)
- [ ] The old code path and the new one produce identical output in shadow mode
- [ ] Feature flag has an owner and a cleanup date
- [ ] Removed legacy code has zero remaining callers (`grep` is clean)
- [ ] The diff for the removal commit is pure deletions
