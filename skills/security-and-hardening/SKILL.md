---
name: security-and-hardening
description: Hardens code against vulnerabilities. Use when handling user input, authentication, data storage, or external integrations. Use when building any feature that accepts untrusted data, manages user sessions, or interacts with third-party services.
---

# Security and Hardening

## Overview

Security-first development practices for PHP web applications. Treat every external input as hostile, every secret as sacred, and every authorization check as mandatory. Security isn't a phase — it's a constraint on every line of code that touches user data, authentication, or external systems.

## When to Use

- Building anything that accepts user input
- Implementing authentication or authorization
- Storing or transmitting sensitive data
- Integrating with external APIs or services
- Adding file uploads, webhooks, or callbacks
- Handling payment or PII data

## The Three-Tier Boundary System

### Always Do (No Exceptions)

- **Validate all external input** at the boundary (Slim route handlers, form handlers)
- **Parameterize all SQL queries** with PDO prepared statements — never concatenate user input
- **Escape output** — use Smarty's `|escape` modifier, `htmlspecialchars()` with `ENT_QUOTES`
- **Use HTTPS** for all external communication
- **Hash passwords** with `password_hash()` using `PASSWORD_BCRYPT` or `PASSWORD_ARGON2ID`
- **Set security headers** (CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy)
- **Set session cookie flags:** `HttpOnly`, `Secure`, `SameSite=Lax`
- **Run `composer audit`** before every release

### Ask First (Requires Human Approval)

- Adding new authentication flows or changing auth logic
- Storing new categories of sensitive data (PII, payment info)
- Adding new external service integrations
- Changing CORS configuration
- Adding file upload handlers
- Modifying rate limiting or throttling
- Granting elevated permissions or roles

### Never Do

- **Never commit secrets** to version control (`.env` must be gitignored)
- **Never log sensitive data** (passwords, session IDs, full card numbers)
- **Never trust client-side validation** as a security boundary
- **Never disable security headers** for convenience
- **Never `eval()` or `unserialize()` user input**
- **Never `include` / `require` paths built from user input**
- **Never store sessions in client-accessible storage** (localStorage for auth tokens)
- **Never expose stack traces** or internal error details to users

## OWASP Top 10 Prevention

### 1. Injection — SQL, Command, LDAP

```php
// BAD: SQL injection via string interpolation
$result = $pdo->query("SELECT * FROM users WHERE id = '{$_GET['id']}'");

// GOOD: Positional prepared statement
$stmt = $pdo->prepare('SELECT id, email, name FROM users WHERE id = ?');
$stmt->execute([$_GET['id']]);
$user = $stmt->fetch(PDO::FETCH_ASSOC);

// GOOD: Named parameters — clearer for multi-param queries
$stmt = $pdo->prepare(
    'SELECT * FROM tasks WHERE owner_id = :owner AND status = :status'
);
$stmt->execute([':owner' => $userId, ':status' => 'open']);

// BAD: shell command with user input
shell_exec("convert {$_POST['file']} out.png");

// GOOD: escape arguments explicitly
shell_exec('convert ' . escapeshellarg($safePath) . ' out.png');
```

> Identifiers (table/column names) **cannot** be parameterized. Validate
> them against an allowlist before interpolating.

### 2. Broken Authentication

```php
// Password hashing with password_hash — uses bcrypt by default
$hash = password_hash($plaintext, PASSWORD_BCRYPT, ['cost' => 12]);

// Verification is constant-time against timing attacks
if (password_verify($plaintext, $user['password_hash'])) {
    // On privilege change (login, role upgrade) — prevent session fixation
    session_regenerate_id(true);
    $_SESSION['user_id'] = $user['id'];
}

// Constant-time comparison for tokens (reset, API, HMAC)
if (hash_equals($expectedToken, $providedToken)) {
    // ...
}
```

**Session cookie configuration** — set before `session_start()`:

```php
session_set_cookie_params([
    'lifetime' => 0,
    'path'     => '/',
    'domain'   => '',
    'secure'   => true,        // HTTPS only
    'httponly' => true,        // not accessible to JS
    'samesite' => 'Lax',       // CSRF mitigation
]);
session_name('APP_SESSID');
session_start();
```

### 3. Cross-Site Scripting (XSS)

```smarty
{* BAD: raw output *}
<div>{$userInput}</div>

{* GOOD: Smarty auto-escaping *}
<div>{$userInput|escape:'html'}</div>

{* For attributes *}
<a href="{$url|escape:'url'}" title="{$title|escape:'htmlall'}">...</a>
```

```php
// PHP output outside Smarty
echo htmlspecialchars($userInput, ENT_QUOTES | ENT_HTML5, 'UTF-8');
```

> **Never** inject user data into inline `<script>` blocks or `onclick=`
> handlers, even after escaping. If JS needs server data, serialize it to
> a JSON script tag and read it from JS.

### 4. Broken Access Control

```php
// Always check authorization, not just authentication
$app->patch('/api/tasks/{id}', function (Request $req, Response $res, array $args) use ($taskService) {
    $userId = $_SESSION['user_id'] ?? null;
    if ($userId === null) {
        return $res->withStatus(401);
    }

    $task = $taskService->findById($args['id']);
    if ($task === null) {
        return $res->withStatus(404);
    }

    // The authenticated user must own this resource
    if ($task->ownerId !== $userId) {
        $res->getBody()->write(json_encode([
            'error' => ['code' => 'FORBIDDEN', 'message' => 'Not authorized'],
        ], JSON_THROW_ON_ERROR));
        return $res->withStatus(403)->withHeader('Content-Type', 'application/json');
    }

    $updated = $taskService->update($args['id'], (array) $req->getParsedBody());
    $res->getBody()->write(json_encode($updated, JSON_THROW_ON_ERROR));
    return $res->withHeader('Content-Type', 'application/json');
});
```

### 5. Security Misconfiguration — Headers, CORS

```php
// Send security headers early (front controller or Slim middleware)
header("Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:");
header('Strict-Transport-Security: max-age=31536000; includeSubDomains');
header('X-Content-Type-Options: nosniff');
header('X-Frame-Options: DENY');
header('Referrer-Policy: strict-origin-when-cross-origin');
header('Permissions-Policy: geolocation=(), camera=(), microphone=()');

// CORS — restrict to known origins, never wildcard with credentials
$allowed = ['https://app.example.com', 'https://admin.example.com'];
$origin  = $_SERVER['HTTP_ORIGIN'] ?? '';
if (in_array($origin, $allowed, true)) {
    header("Access-Control-Allow-Origin: $origin");
    header('Access-Control-Allow-Credentials: true');
    header('Vary: Origin');
}
```

Production `php.ini`:

```ini
expose_php            = Off
display_errors        = Off
display_startup_errors = Off
log_errors            = On
error_reporting       = E_ALL
session.use_strict_mode = 1
session.cookie_httponly = 1
session.cookie_secure   = 1
session.cookie_samesite = "Lax"
```

### 6. Sensitive Data Exposure

```php
// Never return sensitive fields — build explicit output DTOs
final class PublicUser
{
    public function __construct(
        public readonly string $id,
        public readonly string $email,
        public readonly string $name,
    ) {}

    public static function fromRow(array $row): self
    {
        return new self($row['id'], $row['email'], $row['name']);
        // password_hash, reset_token, etc. intentionally dropped
    }
}

// Secrets via environment only — never in code
$stripeKey = getenv('STRIPE_API_KEY') ?: throw new \RuntimeException('STRIPE_API_KEY not set');
```

## Input Validation at the Boundary

```php
final class CreateTaskInput
{
    public function __construct(
        public readonly string  $title,
        public readonly ?string $description,
        public readonly string  $priority,
    ) {}

    /** @param array<string,mixed> $data */
    public static function fromArray(array $data): self
    {
        $errors = [];

        $title = trim((string) ($data['title'] ?? ''));
        if ($title === '' || mb_strlen($title) > 200) {
            $errors['title'] = 'Title is required and must be 1-200 characters';
        }

        $description = isset($data['description']) ? (string) $data['description'] : null;
        if ($description !== null && mb_strlen($description) > 2000) {
            $errors['description'] = 'Description too long';
        }

        $priority = (string) ($data['priority'] ?? 'medium');
        if (!in_array($priority, ['low', 'medium', 'high'], true)) {
            $errors['priority'] = 'Invalid priority';
        }

        if ($errors) {
            throw new ValidationException($errors);
        }
        return new self($title, $description, $priority);
    }
}
```

Use `filter_var()` for standard formats:

```php
$email = filter_var($input, FILTER_VALIDATE_EMAIL);
$int   = filter_var($input, FILTER_VALIDATE_INT, [
    'options' => ['min_range' => 1, 'max_range' => 1_000_000],
]);
$url   = filter_var($input, FILTER_VALIDATE_URL, FILTER_FLAG_SCHEME_REQUIRED);
```

## File Upload Safety

```php
final class UploadValidator
{
    private const ALLOWED_MIME = ['image/jpeg', 'image/png', 'image/webp'];
    private const ALLOWED_EXT  = ['jpg', 'jpeg', 'png', 'webp'];
    private const MAX_SIZE     = 5 * 1024 * 1024;

    public function validate(array $file): void
    {
        if (($file['error'] ?? UPLOAD_ERR_NO_FILE) !== UPLOAD_ERR_OK) {
            throw new ValidationException(['file' => 'Upload failed']);
        }
        if ($file['size'] > self::MAX_SIZE) {
            throw new ValidationException(['file' => 'File too large (max 5 MB)']);
        }

        // Sniff real MIME from bytes — do NOT trust the client-sent type
        $finfo = new \finfo(FILEINFO_MIME_TYPE);
        $real  = $finfo->file($file['tmp_name']);
        if (!in_array($real, self::ALLOWED_MIME, true)) {
            throw new ValidationException(['file' => 'File type not allowed']);
        }

        $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
        if (!in_array($ext, self::ALLOWED_EXT, true)) {
            throw new ValidationException(['file' => 'File extension not allowed']);
        }
    }
}
```

**Storage:** write uploads to a directory **outside the webroot**, serve them
through a PHP handler that enforces authorization, and generate a random
filename (`bin2hex(random_bytes(16))`) so the original name cannot influence
the filesystem.

## CSRF Protection

```php
// Generate a per-session token
if (empty($_SESSION['csrf'])) {
    $_SESSION['csrf'] = bin2hex(random_bytes(32));
}

// Render it into every form
?>
<form method="post" action="/tasks">
  <input type="hidden" name="csrf" value="<?= htmlspecialchars($_SESSION['csrf'], ENT_QUOTES) ?>">
  <!-- ... -->
</form>
<?php

// Verify on every state-changing request
if (!hash_equals($_SESSION['csrf'] ?? '', (string) ($_POST['csrf'] ?? ''))) {
    http_response_code(419);
    exit('CSRF token mismatch');
}
```

## Triaging `composer audit` Results

```
composer audit reports a vulnerability
├── Severity: critical or high
│   ├── Reachable in your code path? YES → update or replace immediately
│   └── Dev-only dep (phpunit plugin etc.)? → fix soon, not a blocker
├── Severity: medium
│   └── Fix next release cycle
└── Severity: low
    └── Track and fix during regular updates
```

When you defer a fix, document the reason and set a review date.

## Rate Limiting

Rate-limit auth and password-reset endpoints. A minimal PHP approach uses a
dedicated Redis / MySQL counter table keyed by `ip + route` with a sliding
window. For pure-PHP installs, APCu works for per-server counters:

```php
$key     = "ratelimit:login:{$_SERVER['REMOTE_ADDR']}";
$current = (int) apcu_fetch($key);
if ($current >= 10) {
    http_response_code(429);
    header('Retry-After: 900');
    exit('Too many attempts');
}
apcu_store($key, $current + 1, 900); // 15-minute window
```

## PHP-Specific Pitfalls

| Pitfall | Problem | Fix |
|---|---|---|
| `==` / `!=` | Loose comparison: `'0e1' == '0e2'` is `true` | Always use `===` / `!==` |
| `@` error suppression | Hides real errors and makes debugging impossible | Handle errors explicitly, check return values |
| `extract($_POST)` | Lets attackers define arbitrary variables | Never extract user input into the symbol table |
| `unserialize($_COOKIE[...])` | Trivial RCE via object injection | Use `json_decode()`; if you must, use `allowed_classes: []` |
| `include $_GET['page']` | Local/remote file inclusion | Allowlist route names, never interpolate user input into paths |
| `mysqli_real_escape_string` | Easy to forget, context-sensitive | Use PDO prepared statements everywhere |
| `md5()` / `sha1()` for passwords | Fast hashes are trivially cracked | `password_hash()` only |
| Trusting `$_SERVER['HTTP_X_FORWARDED_FOR']` | Client-controlled | Only trust behind a known reverse proxy, validate the proxy chain |

## Secrets Management

```
.env files:
  ├── .env.example  → committed (placeholder values only)
  ├── .env          → NOT committed, lives outside webroot
  └── .env.local    → NOT committed, developer overrides

.gitignore must include:
  .env
  .env.local
  .env.*.local
  *.pem
  *.key
```

```bash
# Pre-commit sanity check
git diff --cached | grep -i "password\|secret\|api_key\|token"
```

## Security Review Checklist

```markdown
### Authentication
- [ ] Passwords hashed with password_hash (bcrypt cost ≥ 12 or Argon2id)
- [ ] Session cookies HttpOnly + Secure + SameSite=Lax
- [ ] session_regenerate_id(true) on login and privilege change
- [ ] Login and password reset endpoints rate-limited
- [ ] Password reset tokens have short expiry

### Authorization
- [ ] Every endpoint checks user permissions, not just authentication
- [ ] Users can only access their own resources (verify ownerId)
- [ ] Admin actions verify admin role

### Input
- [ ] All user input validated at the boundary (DTOs / filter_var)
- [ ] All SQL queries use PDO prepared statements
- [ ] Output escaped in Smarty with |escape, or htmlspecialchars in PHP
- [ ] CSRF tokens on every state-changing form
- [ ] File uploads validated by MIME sniff, size, and extension allowlist

### Data
- [ ] No secrets in code or version control
- [ ] password_hash / reset_token / session_id excluded from API responses
- [ ] PII encrypted at rest where applicable

### Infrastructure
- [ ] Security headers present (CSP, HSTS, X-Content-Type-Options, etc.)
- [ ] CORS restricted to known origins (no wildcard with credentials)
- [ ] composer audit clean
- [ ] display_errors=Off in production php.ini
- [ ] Error responses don't expose stack traces or SQL text
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This is an internal tool, security doesn't matter" | Internal tools get compromised. Attackers target the weakest link. |
| "We'll add security later" | Security retrofitting is 10× harder than building it in. Add it now. |
| "No one would try to exploit this" | Automated scanners will find it. Security by obscurity is not security. |
| "The framework handles security" | Frameworks provide tools, not guarantees. You still need to use them correctly. |
| "prepared statements are slower" | The difference is measured in microseconds and is dwarfed by the network round trip. |
| "It's just a prototype" | Prototypes become production. Security habits from day one. |

## Red Flags

- User input concatenated into SQL, shell commands, or HTML
- Secrets in source code or commit history
- Endpoints without authentication or authorization checks
- Missing CSRF tokens on state-changing forms
- CORS wildcard (`*`) with credentials
- No rate limiting on login / password reset
- `==` comparisons on user-controlled data
- `unserialize()` or `include` on user input
- Stack traces or SQL errors exposed to users
- `composer audit` ignored in CI

## Verification

After implementing security-relevant code:

- [ ] `composer audit` shows no critical or high vulnerabilities
- [ ] No secrets in source code or git history
- [ ] All user input validated at system boundaries
- [ ] All SQL queries go through PDO prepared statements (grep for raw interpolation)
- [ ] AuthN and authZ checked on every protected endpoint
- [ ] Security headers present in response (verify with browser DevTools)
- [ ] Error responses don't expose internal details
- [ ] Rate limiting active on auth endpoints
