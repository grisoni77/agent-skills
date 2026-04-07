---
name: documentation-and-adrs
description: Records decisions and documentation. Use when making architectural decisions, changing public APIs, shipping features, or when you need to record context that future engineers and agents will need to understand the codebase.
---

# Documentation and ADRs

## Overview

Document decisions, not just code. The most valuable documentation captures the *why* — the context, constraints, and trade-offs that led to a decision. Code shows *what* was built; documentation explains *why it was built this way* and *what alternatives were considered*. This context is essential for future humans and agents working in the codebase.

## When to Use

- Making a significant architectural decision
- Choosing between competing approaches
- Adding or changing a public API
- Shipping a feature that changes user-facing behavior
- Onboarding new team members (or agents) to the project
- When you find yourself explaining the same thing repeatedly

**When NOT to use:** Don't document obvious code. Don't add comments that restate what the code already says. Don't write docs for throwaway prototypes.

## Architecture Decision Records (ADRs)

ADRs capture the reasoning behind significant technical decisions. They're the highest-value documentation you can write.

### When to Write an ADR

- Choosing a framework, library, or major dependency
- Designing a data model or database schema
- Selecting an authentication strategy
- Deciding on an API architecture (REST vs. GraphQL vs. tRPC)
- Choosing between build tools, hosting platforms, or infrastructure
- Any decision that would be expensive to reverse

### ADR Template

Store ADRs in `docs/decisions/` with sequential numbering:

```markdown
# ADR-001: Use MySQL 8 for primary database

## Status
Accepted | Superseded by ADR-XXX | Deprecated

## Date
2026-01-15

## Context
We need a primary database for the task management application. Key requirements:
- Relational data model (users, tasks, teams with relationships)
- ACID transactions on InnoDB for task state changes
- Full-text search on task content via MySQL `FULLTEXT` indexes
- Runs inside our existing LEMP stack with minimal ops overhead

## Decision
Use MySQL 8 (InnoDB) with hand-written versioned SQL migrations and PDO
for data access from PHP.

## Alternatives Considered

### MongoDB
- Pros: Flexible schema, easy to start with
- Cons: Our data is inherently relational; would need to manage relationships manually
- Rejected: Relational data in a document store leads to complex joins or data duplication

### SQLite
- Pros: Zero configuration, embedded, fast for reads
- Cons: Limited concurrent write support, unsuitable for LEMP production use
- Rejected: Not suitable for a multi-user web application in production

### PostgreSQL
- Pros: Richer JSON and full-text tooling, strong ecosystem
- Cons: Adds a second DB engine to our ops stack; team already operates MySQL
- Rejected: MySQL 8 covers our feature requirements and fits existing ops

## Consequences
- Hand-rolled SQL migrations — explicit control, no ORM abstraction to fight
- InnoDB transactions gate every write path that mutates related rows
- MySQL `FULLTEXT` covers search without introducing Elasticsearch
- Team already operates MySQL in production — zero ops learning curve
```

### ADR Lifecycle

```
PROPOSED → ACCEPTED → (SUPERSEDED or DEPRECATED)
```

- **Don't delete old ADRs.** They capture historical context.
- When a decision changes, write a new ADR that references and supersedes the old one.

## Inline Documentation

### When to Comment

Comment the *why*, not the *what*:

```php
// BAD: Restates the code
// Increment counter by 1
$counter += 1;

// GOOD: Explains non-obvious intent
// Rate limit uses a sliding window — reset counter at window boundary,
// not on a fixed schedule, to prevent burst attacks at window edges
if ($now - $windowStart > self::WINDOW_SIZE_MS) {
    $counter = 0;
    $windowStart = $now;
}
```

### When NOT to Comment

```php
// Don't comment self-explanatory code
function calculateTotal(array $items): int
{
    return array_sum(array_map(fn(CartItem $i) => $i->price * $i->quantity, $items));
}

// Don't leave TODO comments for things you should just do now
// TODO: add error handling  ← Just add it

// Don't leave commented-out code
// $oldImplementation = fn() => ...;  ← Delete it, git has history
```

### Document Known Gotchas

```php
/**
 * IMPORTANT: must be called before the Smarty view is rendered.
 * If called after the template is fetched, the theme variables are
 * missing from the compiled template cache and the page flashes
 * with the default palette on first paint.
 *
 * See ADR-003 for the full design rationale.
 */
public function initializeTheme(Theme $theme): void
{
    // ...
}
```

## API Documentation

For public APIs (REST, GraphQL, library interfaces):

### Inline with PHPDoc (Preferred for PHP)

```php
/**
 * Creates a new task.
 *
 * @param CreateTaskInput $input Task creation data (title required, description optional)
 * @return Task The created task with server-generated ID and timestamps
 * @throws ValidationException    If the title is empty or exceeds 200 characters
 * @throws AuthenticationException If the user is not authenticated
 *
 * @example
 *   $task = $taskService->createTask(new CreateTaskInput(title: 'Buy groceries'));
 *   echo $task->id; // "task_abc123"
 */
public function createTask(CreateTaskInput $input): Task
{
    // ...
}
```

### OpenAPI / Swagger for REST APIs

```yaml
paths:
  /api/tasks:
    post:
      summary: Create a task
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTaskInput'
      responses:
        '201':
          description: Task created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Task'
        '422':
          description: Validation error
```

## README Structure

Every project should have a README that covers:

```markdown
# Project Name

One-paragraph description of what this project does.

## Quick Start
1. Clone the repo
2. Install dependencies: `composer install`
3. Set up environment: `cp .env.example .env`
4. Apply migrations: `php bin/migrate.php up`
5. Run the dev server: `php -S localhost:8080 -t public/`

## Commands
| Command | Description |
|---------|-------------|
| `php -S localhost:8080 -t public/` | Start development server |
| `./vendor/bin/phpunit` | Run tests |
| `composer install --no-dev -o` | Production install |
| `./vendor/bin/phpcs` | Run linter (PSR-12) |
| `./vendor/bin/phpstan analyse` | Static analysis |

## Architecture
Brief overview of the project structure and key design decisions.
Link to ADRs for details.

## Contributing
How to contribute, coding standards, PR process.
```

## Changelog Maintenance

For shipped features:

```markdown
# Changelog

## [1.2.0] - 2025-01-20
### Added
- Task sharing: users can share tasks with team members (#123)
- Email notifications for task assignments (#124)

### Fixed
- Duplicate tasks appearing when rapidly clicking create button (#125)

### Changed
- Task list now loads 50 items per page (was 20) for better UX (#126)
```

## Documentation for Agents

Special consideration for AI agent context:

- **CLAUDE.md / rules files** — Document project conventions so agents follow them
- **Spec files** — Keep specs updated so agents build the right thing
- **ADRs** — Help agents understand why past decisions were made (prevents re-deciding)
- **Inline gotchas** — Prevent agents from falling into known traps

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The code is self-documenting" | Code shows what. It doesn't show why, what alternatives were rejected, or what constraints apply. |
| "We'll write docs when the API stabilizes" | APIs stabilize faster when you document them. The doc is the first test of the design. |
| "Nobody reads docs" | Agents do. Future engineers do. Your 3-months-later self does. |
| "ADRs are overhead" | A 10-minute ADR prevents a 2-hour debate about the same decision six months later. |
| "Comments get outdated" | Comments on *why* are stable. Comments on *what* get outdated — that's why you only write the former. |

## Red Flags

- Architectural decisions with no written rationale
- Public APIs with no documentation or types
- README that doesn't explain how to run the project
- Commented-out code instead of deletion
- TODO comments that have been there for weeks
- No ADRs in a project with significant architectural choices
- Documentation that restates the code instead of explaining intent

## Verification

After documenting:

- [ ] ADRs exist for all significant architectural decisions
- [ ] README covers quick start, commands, and architecture overview
- [ ] API functions have parameter and return type documentation
- [ ] Known gotchas are documented inline where they matter
- [ ] No commented-out code remains
- [ ] Rules files (CLAUDE.md etc.) are current and accurate
