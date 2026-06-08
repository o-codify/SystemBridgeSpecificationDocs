---
id: plugin-db
title: "Plugin: db"
status: stable
version: 26.608.1925
tags: [ plugin, db, postgres, mysql, sqlite, redis ]
---

# Plugin: `db`

SQL (PostgreSQL / MySQL / SQLite) + Redis query and introspection. AI
doesn't have to shell out to `psql` / `sqlite3` / `redis-cli`.

## Connection resolution

```mermaid
flowchart TD
  call[db.query / schema / redis_*] --> conn{conn arg shape?}
  conn -- contains : --> raw[treat as DSN]
  conn -- bare name --> alias[lookup in\n~/.systembridge/db_conns.toml]
  alias --> aliasOk{found?}
  aliasOk -- no --> err[validation_failed:\nunknown alias]
  aliasOk -- yes --> raw
  raw --> detect{prefix?}
  detect -- postgres:// --> pg[(pgx driver)]
  detect -- mysql:// --> my[(go-sql-driver)]
  detect -- sqlite: / file: --> lite[(modernc.org/sqlite)]
  detect -- redis:// / rediss:// --> red[(go-redis client)]
  detect -- ends with .db/.sqlite --> lite
  pg --> cached[sqlCache by dialect+DSN]
  my --> cached
  lite --> cached
  red --> redisCache[redisCache by DSN]
  cached --> exec[ctx-bounded Query/Exec\n5s default timeout]
  redisCache --> exec
  exec --> rows[scan rows\n1000-row cap]
  rows --> trunc{rows_truncated?}
  trunc -- yes --> respTrunc[response with\nrows_truncated: true]
  trunc -- no --> resp[response]
```

DSN with embedded password is **never** logged or returned — only the
alias name appears in the audit log. `db.conns_list` redacts the
password via `dsnPwRe`.

## Connection strings

Either a raw DSN:

```
postgres://user:pass@host:5432/db
mysql://user:pass@tcp(host:3306)/db
sqlite:./local.db     (or file:./local.db)
redis://host:6379/0
```

…or a **named alias** from `~/.systembridge/db_conns.toml`:

```toml
[primary]
dsn = "postgres://user:pass@host:5432/db"

[cache]
dsn = "redis://localhost:6379/0"
```

Used as: `db.query(conn="primary", sql="SELECT 1")`. The raw DSN
(with password) **never appears in the audit log** — only the alias.

## SQL tools

| Tool | Purpose |
|---|---|
| `db.query(conn, sql, dialect?, read_only?, timeout_ms?)` | Execute SQL. Returns `{columns, rows, row_count, rows_truncated, duration_ms}` for SELECT. Returns `{rows_affected, last_insert_id?}` for writes. **Risk: medium.** |
| `db.schema(conn, dialect?)` | List tables + views + sequences. |
| `db.describe(conn, table, dialect?)` | Per-table columns: `{name, type, nullable, default, is_pk}`. |
| `db.explain(conn, sql, analyze?, dialect?)` | EXPLAIN / EXPLAIN ANALYZE wrapper. |
| `db.indexes_list(conn, table?, dialect?)` | Indexes on one table or every table. |
| `db.conns_list()` | List configured aliases (DSN redacted). |

## Redis tools

| Tool | Purpose |
|---|---|
| `db.redis_get(conn, key, decode?)` | GET with optional JSON decode. |
| `db.redis_set(conn, key, value, ttl_ms?)` | SET. **Risk: medium.** |
| `db.redis_keys(conn, pattern?, limit?)` | SCAN-based with cap (default 200, max 5000). |
| `db.redis_type(conn, key)` | Returns string/hash/list/set/zset/stream/none. |
| `db.redis_hgetall(conn, key)` | HGETALL. |
| `db.redis_info(conn, section?)` | Parsed INFO. |
| `db.redis_del(conn, keys)` | DEL. **Risk: high.** Idempotent. |

## Safety

- Reads unrestricted. Writes gated by `write_databases` permission.
- `read_only=true` on `db.query` refuses anything that isn't SELECT /
  SHOW / DESCRIBE / EXPLAIN even if the permission is granted.
- Row cap: 1000 per `db.query`; `rows_truncated` flag if hit.
- Per-query timeout (default 5s, max 60s).
- Audit log records the alias + SQL hash + duration + row_count — never the raw DSN.

## Implementation notes

- PostgreSQL: `github.com/jackc/pgx/v5/stdlib` via `database/sql`.
- MySQL: `github.com/go-sql-driver/mysql`.
- SQLite: `modernc.org/sqlite` — pure-Go, no cgo (clean cross-compile).
- Redis: `github.com/redis/go-redis/v9`.
- Connection pool: cached per `(dialect, DSN)` pair; idle eviction TBD.

## Cross-references

- [Plugin: files](files.md) — for `.sql` script reading
- [Architecture: risk labels](../architecture-risk-labels.md)
