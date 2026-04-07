---
name: database-design-and-optimization
description: Designs and optimizes MySQL schemas, queries, and migrations. Use when creating new tables, changing schema, debugging slow queries, designing indexes, or setting up migrations and backups for a MySQL-backed PHP application.
---

# Database Design and Optimization

## Overview

Design MySQL schemas that are correct by default and fast under load. The database is the single hardest thing to change later — bad choices compound across every feature that touches the data. Normalize first, denormalize only with measurements, index for the queries you actually run, and treat every schema change as a reviewable migration.

## When to Use

- Designing a new table or relationship
- Adding indexes or changing existing ones
- Debugging a slow query or an N+1 pattern
- Writing or reviewing a migration
- Planning backups and recovery
- Interpreting `EXPLAIN` output

## Core Process

```
1. MODEL      → Entities, relationships, cardinality
2. NORMALIZE  → 3NF by default; denormalize only with evidence
3. TYPE       → Pick the smallest correct column types
4. INDEX      → For the queries you actually run (read the EXPLAIN)
5. MIGRATE    → Versioned SQL files, up + down
6. MEASURE    → Slow query log + EXPLAIN after every schema change
```

## Schema Design

### Data Types — Pick the Smallest Correct One

| Use case | Good | Avoid |
|---|---|---|
| Primary keys | `BIGINT UNSIGNED AUTO_INCREMENT` or `BINARY(16)` for UUIDs | `VARCHAR(36)` for UUIDs (3× the storage, slower joins) |
| Booleans | `TINYINT(1) NOT NULL DEFAULT 0` | `ENUM('y','n')`, `VARCHAR(5)` |
| Enums | `ENUM(...)` for rarely changing sets, lookup table otherwise | `VARCHAR` with ad-hoc values |
| Timestamps | `DATETIME(6)` (UTC, microseconds) or `TIMESTAMP` (TZ-aware) | `VARCHAR`, `INT` epoch |
| Money | `DECIMAL(12,2)` | `FLOAT`, `DOUBLE` |
| Text | `VARCHAR(n)` sized to the real max, `TEXT` for unbounded | `TEXT` everywhere by default |
| JSON blobs | `JSON` only when the structure is truly variable | `JSON` as a dumping ground for fields that should be columns |

### NOT NULL by Default

`NULL` has a meaning: "the value is unknown or not applicable." Don't use it as "empty." Every column should be `NOT NULL` unless it is semantically nullable.

```sql
CREATE TABLE tasks (
    id           BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT,
    owner_id     BIGINT UNSIGNED    NOT NULL,
    title        VARCHAR(200)       NOT NULL,
    description  TEXT               NULL,                 -- genuinely optional
    status       ENUM('pending','in_progress','done','cancelled') NOT NULL DEFAULT 'pending',
    priority     TINYINT UNSIGNED   NOT NULL DEFAULT 2,   -- 1=low, 2=med, 3=high
    due_at       DATETIME           NULL,                 -- may be unset
    created_at   DATETIME(6)        NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at   DATETIME(6)        NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id),
    CONSTRAINT fk_tasks_owner
        FOREIGN KEY (owner_id) REFERENCES users (id) ON DELETE CASCADE,
    KEY idx_tasks_owner_status_due (owner_id, status, due_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### Character Sets

Always `utf8mb4`. Never `utf8` (which is a MySQL-specific 3-byte subset that cannot store emoji or many Asian characters).

### Foreign Keys

Declare them even if the ORM does not require them. They prevent orphan rows, document the relationship, and let MySQL enforce cascade rules you'd otherwise have to write in application code.

## Normalization (and When to Break It)

Default to **3NF**:

1. **1NF:** atomic values — no comma-separated lists in a column
2. **2NF:** every non-key column depends on the whole primary key
3. **3NF:** no transitive dependencies (no `user_city` alongside `user_id` and `city_id`)

**Denormalize only with evidence**: if a read pattern is demonstrably slow even with correct indexes, consider a counter column, a materialized summary table, or duplicated data maintained by a trigger or application code. Document the reason in a comment and in an ADR.

## Index Strategy

### The Golden Rule

**Index for the queries you actually run.** An index that no query uses is pure cost — extra writes, extra disk, slower backups.

### Composite Index Column Order

For a multi-column index, order columns by how they appear in `WHERE` / `ORDER BY`:

```
WHERE col_A = ?             AND col_B = ? ORDER BY col_C
                    ↓
INDEX (col_A, col_B, col_C)          ← leftmost prefix rule
```

The leftmost prefix rule means `INDEX (a, b, c)` serves queries filtering on
`a`, on `a AND b`, and on `a AND b AND c` — but **not** a query filtering on
only `b`.

### Covering Indexes

If an index contains every column the query reads, MySQL answers from the index alone and never touches the table. This is a huge win for hot list endpoints:

```sql
-- Query
SELECT id, title, status FROM tasks WHERE owner_id = ? AND status = ? ORDER BY created_at DESC LIMIT 20;

-- Covering index
KEY idx_tasks_owner_status_created (owner_id, status, created_at, title)
```

### Prefix Indexes on Long Strings

For a `VARCHAR(255)` email column, index only the first 32 characters — it is almost always selective enough and it dramatically reduces index size:

```sql
KEY idx_users_email_prefix (email(32))
```

### When NOT to Add an Index

- Columns with very low cardinality (boolean flag with 99% one value)
- Tables written heavily and read rarely (every index is a write tax)
- "Just in case" — prove it with an `EXPLAIN` that shows a full table scan first

## Query Optimization

### `EXPLAIN` Workflow

Run `EXPLAIN` on every non-trivial query you write and every query surfaced by the slow query log. The columns to watch:

| Column | What it means | What you want |
|---|---|---|
| `type` | Join type | `const`, `eq_ref`, `ref`, `range` — avoid `ALL` (full scan) |
| `key` | Index used | Not `NULL`, and the one you designed for |
| `rows` | Rows examined | As small as possible |
| `Extra` | Hints | Good: `Using index`. Bad: `Using filesort`, `Using temporary` (on hot paths) |

```sql
EXPLAIN SELECT id, title
FROM tasks
WHERE owner_id = 42 AND status = 'pending'
ORDER BY due_at
LIMIT 20;
```

If `type = ALL` or `rows` is huge, add or fix an index and re-run. If `Extra` says `Using filesort` on a high-traffic query, align the `ORDER BY` with the index column order.

### Slow Query Log

Enable in `my.cnf`:

```ini
slow_query_log             = 1
slow_query_log_file        = /var/log/mysql/slow.log
long_query_time            = 0.200
log_queries_not_using_indexes = 1
```

Summarize offenders with `mysqldumpslow -s t -t 20 /var/log/mysql/slow.log`.

### N+1 Query Detection and Fixes

The **N+1 problem**: fetch N rows, then issue one follow-up query per row. See `performance-optimization` for the canonical anti-pattern. The three fixes:

1. **JOIN** — one query returns parent + child columns together
2. **IN-list batch** — fetch all children with `WHERE id IN (?, ?, ?, …)`, then stitch in PHP
3. **Identity map** — cache already-fetched rows per request to avoid re-querying the same row

In a PHP request, prefer the identity map + batch pattern: it keeps the domain code simple and avoids wide joins when only a few child rows are needed.

### Keyset Pagination for Deep Pages

`LIMIT 20 OFFSET 100000` is `O(offset)` — MySQL scans and discards 100 000 rows. Use keyset pagination instead:

```sql
-- First page
SELECT id, title, created_at FROM tasks
WHERE owner_id = ?
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page (cursor = last row of previous page)
SELECT id, title, created_at FROM tasks
WHERE owner_id = ?
  AND (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

## Migrations

### Versioned SQL Files

Keep migrations in plain SQL, one file per change, with an explicit `up` and `down`:

```
migrations/
  20260401_001_create_tasks.up.sql
  20260401_001_create_tasks.down.sql
  20260405_002_add_tasks_due_at_index.up.sql
  20260405_002_add_tasks_due_at_index.down.sql
```

A minimal runner (`bin/migrate.php`) records applied filenames in a
`schema_migrations` table and applies pending files in order inside a
transaction where MySQL allows it.

### Transactional DDL Limits

**MySQL DDL is not transactional**: `CREATE TABLE`, `ALTER TABLE`, `DROP` each auto-commit. A migration that runs DDL and then fails on a subsequent DML leaves the database in a partial state. Rules:

- Split schema changes into small single-DDL migrations
- Never combine destructive DDL with data movement in one file — do them in separate deploys
- For large `ALTER TABLE`s, use `pt-online-schema-change` or `gh-ost` to avoid locking

### Safe Migration Patterns

**Additive and backward-compatible** changes are always safe:

- Add a new nullable column, backfill later
- Add a new table
- Add a new index

**Backward-incompatible** changes need a multi-step deploy:

```
1. Add new column / table alongside the old
2. Deploy app code that writes BOTH old and new
3. Backfill the new column from the old
4. Deploy app code that reads the new and ignores the old
5. In a later release, drop the old column
```

This is the same strangler-fig pattern used for code — see `legacy-code-refactoring`.

### Migration File Hygiene

- **Never edit a migration that has been applied in any environment.** Create a new one that undoes and reapplies.
- Every `up.sql` has a matching `down.sql`. If a change is genuinely irreversible (data deletion), make the `down.sql` an explicit `-- irreversible` comment and document the blast radius.
- Run migrations in CI against a fresh DB to catch syntax errors before production.

## Backups and Recovery

### Daily Logical Backups

```bash
mysqldump \
    --single-transaction \
    --quick \
    --routines \
    --triggers \
    --set-gtid-purged=OFF \
    --databases app_production \
  | gzip > /backups/app_$(date +%F).sql.gz
```

`--single-transaction` gives a consistent snapshot of InnoDB tables without locking writers.

### Binary Logs for Point-in-Time Recovery

Enable binlogs in `my.cnf` (`log_bin = mysql-bin`, `binlog_format = ROW`,
`expire_logs_days = 14`). Combined with a nightly dump, you can restore to any
second by replaying binlogs from the dump timestamp forward:

```bash
# Restore the latest dump
gunzip < /backups/app_2026-04-01.sql.gz | mysql app_production

# Replay binlogs up to just before the bad event
mysqlbinlog --stop-datetime="2026-04-01 14:32:00" /var/log/mysql/mysql-bin.* | mysql app_production
```

### Test Restores Quarterly

A backup you have never restored is a hope, not a recovery plan. Restore the latest dump to a scratch database every quarter and run a smoke suite against it.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll add the index later when it's slow" | By then users are already complaining. Add the index when you add the query. |
| "The table is small, normalization is overkill" | Small tables become big tables. Denormalization is a one-way door. |
| "JSON columns are flexible" | Flexible until you need to index, query, or constrain them — then they're a nightmare. |
| "We'll migrate to a different DB later" | You won't. Design for the DB you use today. |
| "Foreign keys slow writes" | The cost is tiny and dwarfed by the bugs they prevent. |
| "We don't need `down` migrations" | Until a release goes wrong at 2 AM and you have no way back. |
| "Backups are the ops team's problem" | Backups you never test aren't backups. Restore quarterly. |

## Red Flags

- `SELECT *` on every query path (no awareness of columns vs. indexes)
- `VARCHAR(255)` on every string column regardless of actual max length
- Tables with no `PRIMARY KEY` or no `created_at`
- Schema changes deployed without a migration file
- Migrations with destructive DDL and no `down.sql`
- `NULL` used as a sentinel for "empty string" or "zero"
- `LIMIT … OFFSET …` used for deep pagination on hot endpoints
- No slow query log enabled in production
- `utf8` (3-byte) instead of `utf8mb4`

## Verification

After any schema or query change:

- [ ] New tables have primary key, `created_at`, `updated_at`, explicit charset
- [ ] Every `WHERE` / `ORDER BY` combination in hot paths is covered by an index
- [ ] `EXPLAIN` shows `type` better than `ALL` on new queries
- [ ] Migration has both `up.sql` and `down.sql`
- [ ] Migration applied cleanly against a fresh DB in CI
- [ ] Slow query log shows no new entries above the threshold
- [ ] Backup script still completes and a recent restore test succeeded
