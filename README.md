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

### Basic Aggregate Functions
- Know: `COUNT`, `SUM`, `AVG`, `MIN/MAX`, `GROUP BY`
- Solve: LeetCode SQL 50 → **Basic Aggregate Functions**

### Sorting and Grouping
- Know: `GROUP BY` + `HAVING` (filters after aggregation, not `WHERE`), multi-column `ORDER BY`
- Solve: LeetCode SQL 50 → **Sorting and Grouping**

### Advanced Select and Joins
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
- Know: `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `NOT NULL`, when to normalize vs deliberately denormalize
- Why it matters: constraints are what actually keep production data correct, not application code

### UPSERT (ON CONFLICT)
- Know: `INSERT ... ON CONFLICT DO UPDATE` / `DO NOTHING`, `EXCLUDED` keyword
- Why it matters: extremely common in real backend code (idempotent writes, sync jobs)

### Views & Materialized Views
- Know: regular views (saved queries) vs materialized views (cached results, need `REFRESH`)
- Why it matters: materialized views tie directly into what you'll already know from Buffer Cache and Planner

### Functions & Triggers (PL/pgSQL)
- Know: writing a function in PL/pgSQL, trigger basics (`BEFORE`/`AFTER`, row vs statement level)
- Why it matters: some logic (audit columns, validation, cascading updates) belongs in the DB, not the app

### COPY
- Know: `COPY` for bulk insert/export vs row-by-row `INSERT`
- Why it matters: massive performance difference for bulk loads, common in real ingestion pipelines

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

## Postgres Extras
- JSONB: `@>`, `->`, `->>`, `#>`, GIN indexing on it
- Full-text search: `tsvector`/`tsquery`
- `LISTEN`/`NOTIFY` + `SKIP LOCKED`: Postgres as a queue
- `pg_stat_statements`: find your own slow queries

---
