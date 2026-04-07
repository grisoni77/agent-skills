---
name: php-backend-engineering
description: Guides modern PHP 8 backend engineering. Use when building or modifying PHP application code — controllers, services, repositories, middleware, sessions, error handling, or Composer setup. Use for both Slim-based and vanilla PHP front controllers.
---

# PHP Backend Engineering

## Overview

Write modern, maintainable PHP 8 code with clear separation of concerns. The goal is code that is testable, typed, and easy to change — not cleverness. This skill covers the patterns that keep a PHP codebase healthy whether you're on a lean Slim framework setup or a hand-rolled vanilla front controller.

## When to Use

- Creating a new controller, service, or repository
- Refactoring procedural code into classes
- Setting up Composer, autoloading, or namespaces
- Adding authentication, sessions, or middleware
- Deciding where a piece of logic belongs
- Configuring PSR-12 / static analysis tooling

## Core Principles

```
1. declare(strict_types=1)   → At the top of every PHP file
2. Separation of concerns    → Controllers thin, services fat, repositories I/O only
3. Typed everything          → Parameters, returns, properties, DTOs
4. PSR-4 autoloading         → No require_once spaghetti
5. PSR-12 formatting         → Enforced by phpcs in CI
6. Explicit errors           → Exceptions at boundaries, no silent failures
```

## Project Layout

```
project-root/
├── bin/                    # CLI entry points: migrate, queue-worker, seed
├── composer.json
├── config/
│   ├── app.php             # Main config array, env-driven
│   └── routes.php          # Slim route definitions (or vanilla dispatcher map)
├── public/                 # Webroot — only index.php + static assets
│   └── index.php           # Front controller
├── src/                    # PSR-4 "App\\" namespace, production code
│   ├── Http/
│   │   ├── Controllers/
│   │   └── Middleware/
│   ├── Domain/
│   │   ├── Task/
│   │   │   ├── Task.php
│   │   │   ├── TaskRepository.php      (interface)
│   │   │   ├── PdoTaskRepository.php   (implementation)
│   │   │   └── TaskService.php
│   │   └── User/...
│   └── Support/            # Small shared utilities
├── templates/              # Smarty .tpl files
├── tests/
│   ├── Unit/
│   └── Integration/
└── migrations/             # SQL migration files
```

**Rule:** only `public/` is exposed by Nginx. Everything else — `src`, `config`, `templates`, `vendor`, `.env` — lives outside the webroot.

## Composer and Autoloading

```json
{
    "name": "example/app",
    "require": {
        "php": "^8.2",
        "ext-pdo": "*",
        "ext-mbstring": "*",
        "slim/slim": "^4.12",
        "slim/psr7": "^1.6",
        "smarty/smarty": "^4.3",
        "monolog/monolog": "^3.5"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.5",
        "phpstan/phpstan": "^1.10",
        "squizlabs/php_codesniffer": "^3.8"
    },
    "autoload": {
        "psr-4": { "App\\": "src/" }
    },
    "autoload-dev": {
        "psr-4": { "App\\Tests\\": "tests/" }
    },
    "scripts": {
        "test":    "phpunit",
        "lint":    "phpcs --standard=PSR12 src/ tests/",
        "lint:fix": "phpcbf --standard=PSR12 src/ tests/",
        "analyse": "phpstan analyse --memory-limit=1G"
    }
}
```

Run `composer dump-autoload -o` in production for the optimized classmap.

## Thin Controllers, Fat Services, I/O-Only Repositories

The single most important rule in PHP backend code:

```
Controller   → parses HTTP input, calls one service method, shapes HTTP output
Service      → business rules, orchestration, transactions — the domain lives here
Repository   → SQL only, returns typed domain objects or arrays of them
```

### Slim Controller

```php
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Domain\Task\TaskService;
use App\Domain\Task\CreateTaskInput;
use App\Domain\Task\ValidationException;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;

final class TaskController
{
    public function __construct(private readonly TaskService $tasks) {}

    public function create(Request $req, Response $res): Response
    {
        try {
            $input = CreateTaskInput::fromArray((array) $req->getParsedBody());
        } catch (ValidationException $e) {
            return $this->json($res->withStatus(422), [
                'error' => ['code' => 'VALIDATION_ERROR', 'details' => $e->errors()],
            ]);
        }

        $task = $this->tasks->create($input, ownerId: (int) $req->getAttribute('user_id'));

        return $this->json($res->withStatus(201), $task);
    }

    private function json(Response $res, mixed $body): Response
    {
        $res->getBody()->write(json_encode($body, JSON_THROW_ON_ERROR));
        return $res->withHeader('Content-Type', 'application/json');
    }
}
```

### Service

```php
declare(strict_types=1);

namespace App\Domain\Task;

use App\Support\Clock;
use PDO;

final class TaskService
{
    public function __construct(
        private readonly TaskRepository $repo,
        private readonly PDO            $pdo,
        private readonly Clock          $clock,
    ) {}

    public function create(CreateTaskInput $input, int $ownerId): Task
    {
        $this->pdo->beginTransaction();
        try {
            $task = new Task(
                id:        0,
                ownerId:   $ownerId,
                title:     $input->title,
                status:    'pending',
                createdAt: $this->clock->now(),
            );
            $persisted = $this->repo->insert($task);
            $this->pdo->commit();
            return $persisted;
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}
```

### Repository

```php
declare(strict_types=1);

namespace App\Domain\Task;

interface TaskRepository
{
    public function insert(Task $task): Task;

    public function findById(int $id): ?Task;

    /** @return list<Task> */
    public function listForOwner(int $ownerId, int $limit, int $offset): array;
}

final class PdoTaskRepository implements TaskRepository
{
    public function __construct(private readonly \PDO $pdo) {}

    public function insert(Task $task): Task
    {
        $stmt = $this->pdo->prepare(
            'INSERT INTO tasks (owner_id, title, status, created_at)
             VALUES (:owner, :title, :status, :created)'
        );
        $stmt->execute([
            ':owner'   => $task->ownerId,
            ':title'   => $task->title,
            ':status'  => $task->status,
            ':created' => $task->createdAt->format('Y-m-d H:i:s.u'),
        ]);
        return new Task(
            id:        (int) $this->pdo->lastInsertId(),
            ownerId:   $task->ownerId,
            title:     $task->title,
            status:    $task->status,
            createdAt: $task->createdAt,
        );
    }

    public function findById(int $id): ?Task
    {
        $stmt = $this->pdo->prepare('SELECT * FROM tasks WHERE id = :id');
        $stmt->execute([':id' => $id]);
        $row = $stmt->fetch(\PDO::FETCH_ASSOC);
        return $row === false ? null : Task::fromRow($row);
    }

    /** @return list<Task> */
    public function listForOwner(int $ownerId, int $limit, int $offset): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT * FROM tasks WHERE owner_id = :o ORDER BY created_at DESC LIMIT :l OFFSET :off'
        );
        $stmt->bindValue(':o',   $ownerId, \PDO::PARAM_INT);
        $stmt->bindValue(':l',   $limit,   \PDO::PARAM_INT);
        $stmt->bindValue(':off', $offset,  \PDO::PARAM_INT);
        $stmt->execute();
        return array_map(Task::fromRow(...), $stmt->fetchAll(\PDO::FETCH_ASSOC));
    }
}
```

The service knows nothing about SQL. The repository knows nothing about HTTP. Each can be tested in isolation.

## Front Controller and Bootstrap

### Slim Front Controller

```php
// public/index.php
declare(strict_types=1);

use DI\Container;
use Slim\Factory\AppFactory;

require __DIR__ . '/../vendor/autoload.php';

$container = new Container();
(require __DIR__ . '/../config/container.php')($container);

AppFactory::setContainer($container);
$app = AppFactory::create();

$app->addBodyParsingMiddleware();
$app->addRoutingMiddleware();

(require __DIR__ . '/../config/middleware.php')($app);
(require __DIR__ . '/../config/routes.php')($app);

$errorMiddleware = $app->addErrorMiddleware(
    displayErrorDetails: false,
    logErrors:           true,
    logErrorDetails:     true,
);

$app->run();
```

### Vanilla Front Controller (when a framework is overkill)

```php
// public/index.php
declare(strict_types=1);

require __DIR__ . '/../vendor/autoload.php';

use App\Http\Router;

try {
    $router   = require __DIR__ . '/../config/routes.php';
    $response = $router->dispatch($_SERVER['REQUEST_METHOD'], parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH));
    $response->send();
} catch (\Throwable $e) {
    \App\Support\ErrorHandler::handle($e); // logs + 500 page
}
```

Both styles follow the same rule: the front controller wires dependencies and delegates immediately.

## Dependency Injection

Even without a DI container, inject dependencies through constructors. Never `new` a repository inside a service, never call a static singleton, never reach into `$_SESSION` from inside a repository.

```php
// GOOD — explicit, testable
final class TaskService
{
    public function __construct(private readonly TaskRepository $repo) {}
}

// BAD — hidden dependencies, untestable
final class TaskService
{
    public function create(): void
    {
        $repo = new PdoTaskRepository(Db::pdo()); // hidden global
        $user = $_SESSION['user_id'];             // hidden global
        // ...
    }
}
```

For legacy projects without a container, a tiny factory file is enough — see `legacy-code-refactoring`.

## Error Handling

### Exceptions at Boundaries

Throw typed exceptions from the domain. Catch them at the HTTP boundary and map to status codes.

```php
namespace App\Domain\Task;

class TaskNotFoundException extends \RuntimeException {}
class ValidationException extends \RuntimeException
{
    /** @param array<string,string> $errors */
    public function __construct(private readonly array $errors) { parent::__construct('Validation failed'); }
    /** @return array<string,string> */
    public function errors(): array { return $this->errors; }
}
```

Map them centrally in Slim's error middleware, not in every controller.

### Logging with Monolog

```php
use Monolog\Handler\StreamHandler;
use Monolog\Logger;

$logger = new Logger('app');
$logger->pushHandler(new StreamHandler(__DIR__ . '/../var/log/app.log', Logger::INFO));

// Usage
$logger->error('Could not send welcome email', [
    'exception' => $e,
    'user_id'   => $userId,
]);
```

For projects that can't depend on Monolog, a thin wrapper around `error_log()` is acceptable — the key rule is that the application code calls `$logger->error(...)`, not `error_log()` directly. The implementation is swappable.

## Sessions, Authentication, Middleware

See `security-and-hardening` for the cookie flag rules. In PHP code:

```php
// App\Http\Middleware\AuthMiddleware (PSR-15)
final class AuthMiddleware implements MiddlewareInterface
{
    public function process(Request $req, RequestHandler $handler): Response
    {
        if (empty($_SESSION['user_id'])) {
            $res = new \Slim\Psr7\Response(401);
            $res->getBody()->write(json_encode(['error' => ['code' => 'UNAUTHENTICATED']]));
            return $res->withHeader('Content-Type', 'application/json');
        }
        $req = $req->withAttribute('user_id', (int) $_SESSION['user_id']);
        return $handler->handle($req);
    }
}
```

Privilege changes (login, role upgrade) must call `session_regenerate_id(true)` to prevent session fixation.

## Tooling

### PSR-12 with `phpcs`

```xml
<!-- phpcs.xml -->
<?xml version="1.0"?>
<ruleset name="App">
    <file>src</file>
    <file>tests</file>
    <rule ref="PSR12"/>
    <arg name="colors"/>
    <arg name="parallel" value="8"/>
</ruleset>
```

Run `./vendor/bin/phpcs` in CI; `phpcbf` autofixes locally.

### Static Analysis with `phpstan`

```neon
# phpstan.neon
parameters:
    level: 6
    paths:
        - src
        - tests
    excludePaths:
        - src/Legacy/*
```

Raise the level gradually on legacy projects — see `legacy-code-refactoring`.

## PHP-Specific Gotchas

| Pitfall | Fix |
|---|---|
| Loose comparison (`==`) | Always use `===` / `!==` |
| `array_key_exists` vs `isset` | `isset` is false for `null`; use `array_key_exists` when you care |
| `foreach` by reference leaks | `unset($ref)` after the loop |
| Float math (`0.1 + 0.2 !== 0.3`) | `bcmath` or integer cents for money |
| `DateTime` is mutable | Always use `\DateTimeImmutable` |
| String-to-int coercion in `switch` | Prefer `match(true)` with `===` |
| `@` error suppression | Never — handle the error path explicitly |

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "It's just a quick script" | Quick scripts become 5-year-old production code. Start with `declare(strict_types=1)` and a class. |
| "Namespaces complicate things" | PSR-4 autoloading is the single biggest quality upgrade for a PHP codebase. |
| "No service layer needed here" | Controllers without a service layer become untestable procedural blobs. Add the seam. |
| "I'll type it later" | Types prevent real bugs. Add them as you write, not as a later refactor. |
| "Exceptions are slow" | They are not slow on the non-exceptional path, which is the path that matters. |
| "The legacy code doesn't use Composer" | Introduce Composer alongside the legacy loader — see `legacy-code-refactoring`. |

## Red Flags

- `require_once` chains at the top of files that should be PSR-4 autoloaded
- Business logic inside controllers (no service layer)
- SQL strings inside controllers or services (no repository layer)
- Static singletons for database, session, or config access
- `$_SESSION` / `$_POST` / `$_GET` accessed from inside services
- Mixed `mysqli_*` and `PDO` in the same codebase
- Missing `declare(strict_types=1)` in new files
- `DateTime` (mutable) used for timestamps
- `@`-suppressed function calls
- `phpcs` and `phpstan` not wired to CI

## Verification

After writing or changing backend code:

- [ ] `declare(strict_types=1);` at the top of every new PHP file
- [ ] `./vendor/bin/phpcs` clean
- [ ] `./vendor/bin/phpstan analyse` clean at the project's level
- [ ] `./vendor/bin/phpunit` passes, including new service + repository tests
- [ ] Controllers hold no business logic
- [ ] Services hold no SQL
- [ ] Dependencies injected through constructors, not fetched from globals
- [ ] All SQL goes through PDO prepared statements
