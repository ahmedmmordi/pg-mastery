# pg-mastery

A hands-on roadmap for learning PostgreSQL deeply as a backend engineer, query skills plus how Postgres actually works underneath, and the production side of it.

---

## SQL Skill Phases

The [LeetCode SQL 50](https://leetcode.com/studyplan/top-sql-50/) study plan.

### [Select](sql-skills/01-select/)
- Know: projection, `WHERE`, `DISTINCT`, `LIMIT/OFFSET`, `ORDER BY`
- Solve: LeetCode SQL 50 → **Select** section

### [Basic Joins](sql-skills/02-basic-joins/)
- Know: `INNER`, `LEFT/RIGHT`, `FULL OUTER`, self-joins. Draw the Venn diagrams once.
- Solve: LeetCode SQL 50 → **Basic Joins**

### [Basic Aggregate Functions](sql-skills/03-basic-aggregate-functions/)
- Know: `COUNT`, `SUM`, `AVG`, `MIN/MAX`, `GROUP BY`
- Solve: LeetCode SQL 50 → **Basic Aggregate Functions**

### [Sorting and Grouping](sql-skills/04-sorting-and-grouping/)
- Know: `GROUP BY` + `HAVING` (filters after aggregation, not `WHERE`), multi-column `ORDER BY`
- Solve: LeetCode SQL 50 → **Sorting and Grouping**

### [Advanced Select and Joins](sql-skills/05-advanced-select-and-joins)
- Know: window functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG/LEAD`, `PARTITION BY`. Highest-leverage topic here.
- Solve: LeetCode SQL 50 → **Advanced Select and Joins**

### Subqueries
- Know: scalar, correlated, `EXISTS` vs `IN` (know which is faster and why), CTEs (`WITH`), recursive CTEs
- Solve: LeetCode SQL 50 → **Subqueries**

### Advanced String Functions / Regex / Clause
- Know: `LIKE`/`ILIKE`, `~`/`~*` regex, `SIMILAR TO`, `SPLIT_PART`, `CONCAT`, `TRIM`
- Solve: LeetCode SQL 50 → **Advanced String Functions / Regex / Clause**

---

## Schema & Production SQL

### Constraints & Schema Design
- Know: `PRIMARY KEY`, `FOREIGN KEY` (`ON DELETE`/`ON UPDATE` actions, `DEFERRABLE`), `UNIQUE`, `CHECK`, `NOT NULL`, `EXCLUDE`, generated and identity columns (`GENERATED ... AS IDENTITY` over `serial`), when to normalize vs deliberately denormalize
- Know: Postgres does **not** auto-index FK columns. Add those indexes yourself or deletes/joins on the parent get slow
- Why it matters: constraints are what actually keep production data correct, not application code
- Practice: design a small orders schema, then try to break it with bad inserts. Every bad row should be rejected by the DB.

### UPSERT (ON CONFLICT)
- Know: `INSERT ... ON CONFLICT DO UPDATE` / `DO NOTHING`, `EXCLUDED`, `RETURNING`, `MERGE` (PG15+)
- Know: the conflict target needs a unique index or constraint. Use `WHERE ... IS DISTINCT FROM EXCLUDED...` on `DO UPDATE` to skip no-op writes (less bloat, fewer WAL records)
- Know: why it's race-safe where `SELECT` then `INSERT` isn't
- Why it matters: extremely common in real backend code (idempotent writes, sync jobs)
- Practice: run the same batch twice. Row counts stay the same. Then run two sessions concurrently.

### Views & Materialized Views
- Know: regular views (saved query, no stored data) vs materialized views (stored physical result, stale until refreshed)
- Know: `REFRESH MATERIALIZED VIEW CONCURRENTLY` needs a unique index and avoids blocking readers. Nothing refreshes automatically, so you must schedule it (cron, `pg_cron`, or your app)
- Why it matters: it's the standard trade of freshness for read speed on heavy reports and dashboards
- Practice: build a slow aggregate, materialize it, compare `EXPLAIN ANALYZE`, then refresh concurrently while querying it.

### Functions & Triggers (PL/pgSQL)
- Know: writing a PL/pgSQL function, `BEFORE`/`AFTER`, row vs statement level, `OLD`/`NEW`, `TG_OP`, what a `BEFORE` trigger's return value does (`NEW` vs `NULL`), transition tables
- Know: triggers are hidden logic. Good for audit columns, `updated_at`, and strict validation. Risky for business logic (hard to test, invisible to app developers)
- Why it matters: some rules must hold no matter which service or script writes to the table
- Practice: add an audit trigger that writes to a history table, then update rows from two different clients.

### COPY
- Know: `COPY` vs row-by-row `INSERT`, server-side `COPY` vs psql `\copy`, `FORMAT csv`/`binary`, `COPY ... FROM STDIN` through drivers (Npgsql `BeginBinaryImport`, JDBC `CopyManager`)
- Know: `COPY` has no `ON CONFLICT`, so the standard pipeline is `COPY` into a staging table, then `INSERT ... ON CONFLICT` into the real table
- Why it matters: massive performance difference for bulk loads, common in real ingestion pipelines
- Practice: load 1M rows three ways (single inserts, batched inserts, `COPY`) and time each.

### Migrations & Schema Versioning
- Know: versioned, repeatable migrations with Flyway, Liquibase, or EF Migrations. Never edit an applied migration, always add a new one. Migrations live in git and run in CI/CD
- Why it matters: in a real team nobody runs DDL by hand on production. Schema changes are code-reviewed like everything else
- Practice: build your practice schema only through migrations, on a fresh database, from zero.

### Roles & Permissions
- Know: least-privilege roles (separate app, migration, and read-only roles), `GRANT`/`REVOKE`, default privileges. The app should not connect as superuser or table owner
- Why it matters: a bug or leaked credential in the app should not be able to drop tables
- Practice: create three roles and confirm each can do only its job.

---

## Internals Phases (how it actually works)

Reference: [The Internals of PostgreSQL](https://www.interdb.jp/pg/)

### Data Types
- Know: `NUMERIC` for money not `FLOAT`, `BIGINT`/UUID for PKs (UUIDv4 vs UUIDv7 index fragmentation), `TIMESTAMPTZ` always over `TIMESTAMP`, `JSONB` over `JSON`
- Why it matters: wrong type = silent bugs (money rounding) or slow indexes later

### Query Life Cycle
- Know: parser → rewriter (views/rules) → planner → executor
- Practice: `EXPLAIN ANALYZE` on one query, read the plan out loud

### Planner & Optimizer
- Know: cost model (`seq_page_cost`, `random_page_cost`), row estimates, why it picks nested loop vs hash join vs merge join
- Practice: `ANALYZE` a table, watch `pg_stats`, force a bad plan by letting stats go stale

### Indexes
- Know: B-tree (default), GIN (JSONB/full-text/arrays), GiST, BRIN, Hash. Composite index column order. Partial/expression/covering indexes.
- Practice: 1M-row table (`generate_series`), slow query, add index, compare `EXPLAIN ANALYZE` before/after

### Buffer Cache
- Know: `shared_buffers`, how pages move disk ↔ memory, why a "cold" query is slow and a repeated one is fast
- Practice: `EXPLAIN (ANALYZE, BUFFERS)`. Watch shared hit vs read counts

### Internal Architecture
- Know: `postmaster` forks a backend process per connection (not threads, why connections are expensive), background workers (`checkpointer`, `background writer`, `WAL writer`, `autovacuum`), shared memory (`shared_buffers`, WAL buffers, lock tables) vs per-backend local memory (`work_mem`, `maintenance_work_mem`), checkpoints
- Practice: `ps aux | grep postgres` while running queries in two `psql` sessions. Watch the separate backend processes appear

### MVCC
- Know: row versions (tuples), `xmin`/`xmax`, how readers never block writers
- Why it matters: explains almost everything below

### VACUUM
- Know: dead tuples from MVCC, `VACUUM` vs `VACUUM FULL` vs autovacuum, transaction ID wraparound (the outage-causer)
- Practice: `UPDATE` a row repeatedly, watch dead tuple count grow, run `VACUUM`, watch it shrink

### Transactions & ACID
- Know: tie each letter to a real Postgres mechanism: WAL (durability), MVCC (isolation/atomicity), constraints (consistency)

### Isolation
- Know: Read Committed (default), Repeatable Read, Serializable, which anomalies each one blocks (dirty read, non-repeatable read, phantom read)

### Locks & Concurrency
- Know: row-level locks, `FOR UPDATE`/`FOR SHARE`, deadlock detection, advisory locks
- Practice: two `psql` sessions, one locks a row, watch the other block

### WAL
- Know: write-ahead log, crash recovery, how it's the backbone of replication and durability

---

## Production Operations

### Safe (Zero-Downtime) Schema Changes
- Needs: Locks & Concurrency, MVCC
- Know: which `ALTER TABLE` operations take heavy locks (`ACCESS EXCLUSIVE`) and how a lock queue can stall the whole table
- Know: `CREATE INDEX CONCURRENTLY`, adding a column with a default, adding `NOT NULL`/FK safely (`NOT VALID`, then `VALIDATE CONSTRAINT`), and the expand-migrate-contract pattern for renames and type changes
- Know: set `lock_timeout` before DDL so a migration fails fast instead of blocking production
- Why it matters: this is where most production outages from schema work come from
- Practice: on a large table, run a migration in one session while another session runs a steady read/write load. Watch for blocking.

### Connection Pooling
- Needs: Internal Architecture
- Know: why connections are expensive (process per connection), PgBouncer pool modes (session vs transaction) and what breaks in transaction mode (session state, prepared statements)
- Why it matters: the app's pool size × instance count can exhaust `max_connections`
- Practice: run many clients with and without PgBouncer and watch backend processes and `pg_stat_activity`.

### Backup & Recovery
- Needs: WAL
- Know: logical (`pg_dump`/`pg_restore`) vs physical backups, WAL archiving and point-in-time recovery, and that an untested backup is not a backup
- Why it matters: "we can restore to 10:42 before the bad deploy" is a real production requirement
- Practice: take a base backup, make changes, then restore to a chosen point in time.

---

## Postgres Extras
- JSONB: `@>`, `->`, `->>`, `#>`, GIN indexing on it
- Full-text search: `tsvector`/`tsquery`
- `LISTEN`/`NOTIFY` + `SKIP LOCKED`: Postgres as a queue
- `pg_stat_statements`: find your own slow queries

---
