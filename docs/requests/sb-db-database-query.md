---
id: request-sb-db-database-query-plugin-postgresql-mysql-sqlite-redis
title: "Request: sb-db — database query plugin (PostgreSQL / MySQL / SQLite / Redis)"
status: request
version: 26.608.1506
tags: [ sb-core, database, sql, redis, request ]
---

# Request: sb-db — database query plugin

**Origin.** Self-audit + comparative scan (awesome-claude-code-toolkit's
recommended MCP set). Every project session that touches data ends up
in `psql` / `sqlite3` / `redis-cli` through Bash. That's a class of
shell-escape we can collapse.

## What

Single Go plugin that exposes structured query + schema introspection
across the four ubiquitous data stores.

## Tools

### SQL (Postgres / MySQL / SQLite)

| Tool | Purpose |
|---|---|
| `db_query(conn, sql, args?, dialect=auto, timeout_ms=5000)` | Execute SQL; returns `{columns, rows, row_count, duration_ms}`. SELECT-style returns rows; INSERT/UPDATE/DELETE returns `{rows_affected, last_insert_id?}`. |
| `db_schema(conn, dialect=auto)` | List tables + view + sequences + index summary across all schemas. |
| `db_describe(conn, table, dialect=auto)` | Per-table columns: `{name, type, nullable, default, is_pk, is_unique, foreign_key?}`. |
| `db_explain(conn, sql, args?, dialect=auto)` | EXPLAIN / EXPLAIN ANALYZE wrapper. Returns plan as text + structured (Postgres JSON plan; MySQL TRADITIONAL; SQLite EXPLAIN QUERY PLAN). |
| `db_indexes_list(conn, table?, dialect=auto)` | Indexes on one table or all. |

`conn` accepts:
- DSN string (`postgres://user:pass@host:5432/db`, `mysql://...`, `file:./local.db`, …)
- Named alias from `~/.systembridge/db_conns.toml` (`primary`, `cache`).

`dialect=auto` sniffs from DSN prefix; explicit override for ambiguous cases.

### Redis

| Tool | Purpose |
|---|---|
| `redis_get(conn, key)` / `redis_set(conn, key, value, ttl_ms?)` | KV ops. Auto-decode JSON if `decode=true`. |
| `redis_keys(conn, pattern, limit=200)` | SCAN-based. Caps at `limit`. |
| `redis_type(conn, key)` | `string` / `hash` / `list` / `set` / `zset` / `stream`. |
| `redis_hget(conn, key, field)` / `redis_hgetall(conn, key)` | Hash ops. |
| `redis_info(conn, section?)` | `INFO` parsed. |
| `redis_del(conn, key)` | Delete. Idempotent. |

## Why

Routine reach-for-shell patterns we'd replace:

- "How many users in the dev DB?" → one `db_query` call vs `psql -c ...`.
- "Schema of orders table?" → `db_describe`, no inspection of stale dumps.
- "Why's this query slow?" → `db_explain` returns structured plan.
- "Stale key in Redis?" → `redis_get` / `redis_keys`, no `redis-cli`.

Structured output = AI decides next step without parsing CLI text.

## Implementation route

Go plugin `cmd/sb-db/`:

- PostgreSQL: `github.com/jackc/pgx/v5/stdlib` — already vetted dep family.
- MySQL: `github.com/go-sql-driver/mysql`.
- SQLite: `modernc.org/sqlite` (pure-Go, no cgo — keeps cross-compile).
- Redis: `github.com/redis/go-redis/v9`.

Common interface: `Driver { Query, Exec, Schema, Describe, Explain }`.
Dispatch by dialect; connection caching with idle eviction (5 min).

Conn aliases (`~/.systembridge/db_conns.toml`) reuse the existing
secrets store pattern — never put raw passwords in tool args.

## Safety

- `read_only=true` flag on `db_query` refuses anything that isn't
  SELECT/SHOW/DESCRIBE/EXPLAIN (parse the first non-comment token).
- Write ops gated by `write_databases` permission.
- Per-query `timeout_ms` enforced via context. Default 5s.
- Row cap: 1000 rows per `db_query` response; `rows_truncated` flag if hit.
- Audit log captures connection alias (not raw DSN), tool, SQL hash,
  duration, row_count.

## Acceptance

- `db_query("postgres://localhost/dev", "SELECT count(*) FROM users")`
  returns `{columns:["count"], rows:[[42]], row_count:1}`.
- `db_schema("primary")` returns the full table list with row counts.
- `redis_keys("cache", "session:*", 50)` returns up to 50 keys.
- `db_query` with a destructive SQL refuses unless write_databases
  granted.
- Named alias resolves through secrets store; DSN with embedded
  password never appears in audit log.

## Non-goals (defer)

- Stored procedure execution.
- Transactions across tool calls (each call is its own short-lived txn).
- Streaming large result sets (1000-row cap is plenty for AI inspection).
- MongoDB / DynamoDB / Cassandra — long-tail.
