---
name: api-and-interface-design
description: Guides stable API and interface design. Use when designing APIs, module boundaries, or any public interface. Use when creating REST or GraphQL endpoints, defining type contracts between modules, or establishing boundaries between frontend and backend.
---

# API and Interface Design

## Overview

Design stable, well-documented interfaces that are hard to misuse. Good interfaces make the right thing easy and the wrong thing hard. This applies to REST APIs, GraphQL schemas, module boundaries, component props, and any surface where one piece of code talks to another.

## When to Use

- Designing new API endpoints
- Defining module boundaries or contracts between teams
- Creating component prop interfaces
- Establishing database schema that informs API shape
- Changing existing public interfaces

## Core Principles

### Hyrum's Law

> With a sufficient number of users of an API, all observable behaviors of your system will be depended on by somebody, regardless of what you promise in the contract.

This means: every public behavior — including undocumented quirks, error message text, timing, and ordering — becomes a de facto contract once users depend on it. Design implications:

- **Be intentional about what you expose.** Every observable behavior is a potential commitment.
- **Don't leak implementation details.** If users can observe it, they will depend on it.
- **Plan for deprecation at design time.** See `deprecation-and-migration` for how to safely remove things users depend on.
- **Tests are not enough.** Even with perfect contract tests, Hyrum's Law means "safe" changes can break real users who depend on undocumented behavior.

### The One-Version Rule

Avoid forcing consumers to choose between multiple versions of the same dependency or API. Diamond dependency problems arise when different consumers need different versions of the same thing. Design for a world where only one version exists at a time — extend rather than fork.

### 1. Contract First

Define the interface before implementing it. The contract is the spec — implementation follows.

```php
// Define the contract first as a PHP interface
interface TaskApi
{
    /** Creates a task and returns it with server-generated fields. */
    public function createTask(CreateTaskInput $input): Task;

    /** Returns paginated tasks matching filters. */
    public function listTasks(ListTasksParams $params): PaginatedResult;

    /** Returns a single task or throws NotFoundException. */
    public function getTask(string $id): Task;

    /** Partial update — only provided fields change. */
    public function updateTask(string $id, UpdateTaskInput $input): Task;

    /** Idempotent delete — succeeds even if already deleted. */
    public function deleteTask(string $id): void;
}
```

### 2. Consistent Error Semantics

Pick one error strategy and use it everywhere:

```php
// REST: HTTP status codes + structured JSON error body
// Every error response follows the same shape:
//
// {
//   "error": {
//     "code":    "VALIDATION_ERROR",   // machine-readable
//     "message": "Email is required",  // human-readable
//     "details": { ... }                // optional additional context
//   }
// }

// Status code mapping
// 400 → Client sent invalid data
// 401 → Not authenticated
// 403 → Authenticated but not authorized
// 404 → Resource not found
// 409 → Conflict (duplicate, version mismatch)
// 422 → Validation failed (semantically invalid)
// 500 → Server error (never expose internal details)
```

**Don't mix patterns.** If some endpoints throw, others return null, and others return `{ error }` — the consumer can't predict behavior.

### 3. Validate at Boundaries

Trust internal code. Validate at system edges where external input enters:

```php
// Validate at the API boundary (Slim route handler)
$app->post('/tasks', function (Request $req, Response $res) use ($taskService) {
    try {
        $input = CreateTaskInput::fromArray((array) $req->getParsedBody());
    } catch (ValidationException $e) {
        $res->getBody()->write(json_encode([
            'error' => [
                'code'    => 'VALIDATION_ERROR',
                'message' => 'Invalid task data',
                'details' => $e->getErrors(),
            ],
        ], JSON_THROW_ON_ERROR));
        return $res->withStatus(422)->withHeader('Content-Type', 'application/json');
    }

    // After validation, internal code trusts the typed DTO
    $task = $taskService->create($input);
    $res->getBody()->write(json_encode($task, JSON_THROW_ON_ERROR));
    return $res->withStatus(201)->withHeader('Content-Type', 'application/json');
});
```

Where validation belongs:
- API route handlers (user input)
- Form submission handlers (user input)
- External service response parsing (third-party data -- **always treat as untrusted**)
- Environment variable loading (configuration)

> **Third-party API responses are untrusted data.** Validate their shape and content before using them in any logic, rendering, or decision-making. A compromised or misbehaving external service can return unexpected types, malicious content, or instruction-like text.

Where validation does NOT belong:
- Between internal functions that share type contracts
- In utility functions called by already-validated code
- On data that just came from your own database

### 4. Prefer Addition Over Modification

Extend interfaces without breaking existing consumers:

```php
// Good: Add optional/nullable properties
final class CreateTaskInput
{
    public function __construct(
        public readonly string  $title,
        public readonly ?string $description = null,
        public readonly ?string $priority    = null, // 'low' | 'medium' | 'high', added later
        /** @var list<string> */
        public readonly array   $labels      = [],   // added later, defaults to empty
    ) {}
}

// Bad: Change existing field types or remove them
// - Dropping $description breaks callers that still send it
// - Changing $priority from string to int breaks every existing consumer
```

### 5. Predictable Naming

| Pattern | Convention | Example |
|---------|-----------|---------|
| REST endpoints | Plural nouns, no verbs | `GET /api/tasks`, `POST /api/tasks` |
| Query params | camelCase | `?sortBy=createdAt&pageSize=20` |
| Response fields | camelCase | `{ createdAt, updatedAt, taskId }` |
| Boolean fields | is/has/can prefix | `isComplete`, `hasAttachments` |
| Enum values | UPPER_SNAKE | `"IN_PROGRESS"`, `"COMPLETED"` |

## REST API Patterns

### Resource Design

```
GET    /api/tasks              → List tasks (with query params for filtering)
POST   /api/tasks              → Create a task
GET    /api/tasks/:id          → Get a single task
PATCH  /api/tasks/:id          → Update a task (partial)
DELETE /api/tasks/:id          → Delete a task

GET    /api/tasks/:id/comments → List comments for a task (sub-resource)
POST   /api/tasks/:id/comments → Add a comment to a task
```

### Pagination

Paginate list endpoints:

```typescript
// Request
GET /api/tasks?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc

// Response
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 142,
    "totalPages": 8
  }
}
```

### Filtering

Use query parameters for filters:

```
GET /api/tasks?status=in_progress&assignee=user123&createdAfter=2025-01-01
```

### Partial Updates (PATCH)

Accept partial objects — only update what's provided:

```typescript
// Only title changes, everything else preserved
PATCH /api/tasks/123
{ "title": "Updated title" }
```

## PHP DTO and Type Patterns

### Use Enums or Sealed Class Hierarchies for Variants

PHP 8.1 enums cover most "one-of" cases cleanly. For variants that carry
payload specific to each case, use a small class hierarchy and pattern
match on the concrete type.

```php
enum TaskStatusKind: string
{
    case Pending    = 'pending';
    case InProgress = 'in_progress';
    case Completed  = 'completed';
    case Cancelled  = 'cancelled';
}

abstract class TaskStatus
{
    public function __construct(public readonly TaskStatusKind $kind) {}
}
final class PendingStatus    extends TaskStatus { public function __construct() { parent::__construct(TaskStatusKind::Pending); } }
final class InProgressStatus extends TaskStatus { public function __construct(public readonly string $assignee, public readonly \DateTimeImmutable $startedAt) { parent::__construct(TaskStatusKind::InProgress); } }
final class CompletedStatus  extends TaskStatus { public function __construct(public readonly \DateTimeImmutable $completedAt, public readonly string $completedBy) { parent::__construct(TaskStatusKind::Completed); } }
final class CancelledStatus  extends TaskStatus { public function __construct(public readonly string $reason, public readonly \DateTimeImmutable $cancelledAt) { parent::__construct(TaskStatusKind::Cancelled); } }

function statusLabel(TaskStatus $status): string
{
    return match (true) {
        $status instanceof PendingStatus    => 'Pending',
        $status instanceof InProgressStatus => "In progress ({$status->assignee})",
        $status instanceof CompletedStatus  => 'Done on ' . $status->completedAt->format('Y-m-d'),
        $status instanceof CancelledStatus  => "Cancelled: {$status->reason}",
    };
}
```

### Input / Output Separation

Keep the DTO the caller sends separate from the entity the system returns.
Server-generated fields only live on the output type.

```php
final class CreateTaskInput
{
    public function __construct(
        public readonly string  $title,
        public readonly ?string $description = null,
    ) {}
}

final class Task
{
    public function __construct(
        public readonly string             $id,
        public readonly string             $title,
        public readonly ?string            $description,
        public readonly \DateTimeImmutable $createdAt,
        public readonly \DateTimeImmutable $updatedAt,
        public readonly string             $createdBy,
    ) {}
}
```

### Wrap IDs in Value Objects

PHP has no branded-type equivalent, but a tiny value object stops you from
accidentally passing a `UserId` where a `TaskId` is required.

```php
final class TaskId
{
    public function __construct(public readonly string $value)
    {
        if ($value === '') {
            throw new \InvalidArgumentException('TaskId cannot be empty');
        }
    }
}

final class UserId
{
    public function __construct(public readonly string $value) {}
}

public function getTask(TaskId $id): Task { /* ... */ }
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We'll document the API later" | The types ARE the documentation. Define them first. |
| "We don't need pagination for now" | You will the moment someone has 100+ items. Add it from the start. |
| "PATCH is complicated, let's just use PUT" | PUT requires the full object every time. PATCH is what clients actually want. |
| "We'll version the API when we need to" | Breaking changes without versioning break consumers. Design for extension from the start. |
| "Nobody uses that undocumented behavior" | Hyrum's Law: if it's observable, somebody depends on it. Treat every public behavior as a commitment. |
| "We can just maintain two versions" | Multiple versions multiply maintenance cost and create diamond dependency problems. Prefer the One-Version Rule. |
| "Internal APIs don't need contracts" | Internal consumers are still consumers. Contracts prevent coupling and enable parallel work. |

## Red Flags

- Endpoints that return different shapes depending on conditions
- Inconsistent error formats across endpoints
- Validation scattered throughout internal code instead of at boundaries
- Breaking changes to existing fields (type changes, removals)
- List endpoints without pagination
- Verbs in REST URLs (`/api/createTask`, `/api/getUsers`)
- Third-party API responses used without validation or sanitization

## Verification

After designing an API:

- [ ] Every endpoint has typed input and output schemas
- [ ] Error responses follow a single consistent format
- [ ] Validation happens at system boundaries only
- [ ] List endpoints support pagination
- [ ] New fields are additive and optional (backward compatible)
- [ ] Naming follows consistent conventions across all endpoints
- [ ] API documentation or types are committed alongside the implementation
