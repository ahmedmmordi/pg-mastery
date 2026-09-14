# SQL Joins

## 1. Join types (logical result)

![Join types overview](https://cdn0.devart.com/views/content/products/dbforge/postgresql/studio/images/types-of-postgresql-joins.png)

*Source: [dbForge Studio for PostgreSQL — Types of PostgreSQL Joins](https://www.devart.com/dbforge/postgresql/studio/postgresql-joins.html)*

Only five of these are actual SQL `JOIN` keywords: `INNER`, `LEFT`, `RIGHT`,
`FULL OUTER`, and `CROSS`. The rest are patterns, not new syntax:

- **Self join** isn't a keyword — it's just an ordinary `JOIN` where both
  sides happen to be the same table (aliased twice).
- **Semi join** and **anti join** aren't `JOIN` syntax at all — there's no
  `SEMI JOIN`/`ANTI JOIN` keyword in SQL. They're `WHERE IN`/`EXISTS`
  (semi) and `WHERE NOT IN`/`NOT EXISTS` (anti) filters that behave like a
  join without actually being one — no duplicated rows, no columns pulled
  from the other table.

### Inner Join
Only rows with a match on both sides. Commutative: `A JOIN B` = `B JOIN A`.

```sql
SELECT c.c_name, s.s_name
FROM customers c
JOIN stores s ON c.city = s.city;
```

### Left / Right Outer Join
Keep all rows from one side; unmatched columns from the other side become
`NULL`. `A LEFT JOIN B` = `B RIGHT JOIN A` (swap the table order to swap the
join type), but `LEFT JOIN` on its own is not symmetric.

```sql
SELECT c.c_name, s.s_name
FROM customers c
LEFT JOIN stores s ON c.city = s.city;
```

### Full Outer Join
Keep unmatched rows from both sides, `NULL`-padded either way. Symmetric,
like inner join.

```sql
SELECT c.c_name, s.s_name
FROM customers c
FULL OUTER JOIN stores s ON c.city = s.city;
```

### Cross Join
Cartesian product — every row of A paired with every row of B, no condition.

```sql
SELECT c.c_name, s.s_name
FROM customers c
CROSS JOIN stores s;
```

### Self Join
A table joined to itself, usually to compare rows within the same table
(e.g. employees to their managers). Requires aliases to distinguish the two
copies.

```sql
SELECT e.name AS employee, m.name AS manager
FROM Employees e
JOIN Employees m ON e.manager_id = m.id;
```

### Semi Join
Rows from one table that **have** a match in the other — but unlike a real
`JOIN`, it doesn't duplicate rows or pull columns from the other table.
Written with `IN` or `EXISTS`, not an actual `JOIN` keyword.

```sql
SELECT *
FROM customers c
WHERE c.city IN (SELECT s.city FROM stores s);

SELECT *
FROM customers c
WHERE EXISTS (SELECT 1 FROM stores s WHERE s.city = c.city);
```

### Anti Join
Rows from one table that **do not** have a match — `NOT IN` or `NOT EXISTS`.

```sql
SELECT *
FROM customers c
WHERE c.city NOT IN (SELECT s.city FROM stores s);

SELECT *
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM stores s WHERE s.city = c.city);
```

**`NOT IN` NULL trap**: if the subquery can return a `NULL`, `NOT IN`
silently returns zero rows — `x NOT IN (1, NULL)` evaluates to `UNKNOWN`, not
`TRUE`, for every `x`. `NOT EXISTS` doesn't have this problem, so prefer it
when you're not sure the subquery's column is `NOT NULL`.

**Simply put — why `EXISTS` is preferred over `IN`:**

- `IN` builds the *entire list* of values from the subquery first, then
  checks if your value is in that list.
- `EXISTS` just asks *"is there at least one match?"* and stops looking the
  moment it finds one.

Example: `stores.city` has 1 million rows, but only 3 unique cities.

```sql
-- IN: computes the full list from stores, THEN checks membership
WHERE c.city IN (SELECT s.city FROM stores s)

-- EXISTS: for each customer, stops at the first matching store row
WHERE EXISTS (SELECT 1 FROM stores s WHERE s.city = c.city)
```

Same result either way — `EXISTS` is just the safer default (no `NULL`
trap) and often the faster one on large tables.

---

## 2. Join condition syntax

```sql
-- ON: general condition, any columns/names
... JOIN b ON a.id = b.id

-- USING: shorthand when both sides use the same column name;
-- also collapses the column into one (no a.id / b.id duplicate)
... JOIN b USING (id)

-- NATURAL JOIN: auto-matches on all identically-named columns.
-- Avoid in real queries — silently breaks if the schema gains a new
-- shared column name later.
... NATURAL JOIN b
```

---

## 3. Join algorithms (physical execution — PostgreSQL)

### Terminology
- **Relation** — a table, or the result of any plan node (a subquery, an
  index scan, another join). "Scanning a relation" doesn't necessarily mean
  a sequential scan on a table.
- **Outer relation** — the relation on top of the join in the plan tree
  (driving side). **Inner relation** — the one below it.
- **Join key** — the columns compared in the join condition
  (`a.col1 = b.col2`).

### Nested Loop Join
For each row in the outer relation, scan the inner relation for matches.

- **Cost**: `N_outer + (N_outer × N_inner)` — put the smaller input on the
  outer side.
- **Indexes**: an index on the outer relation doesn't help (it's scanned
  sequentially either way); an index on the inner relation's join key turns
  this into an **Index Nested Loop Join** — look up matches directly instead
  of scanning all of the inner relation.
- **Best when**: the outer relation is small. Typical for OLTP workloads on
  a normalized schema. It's also the only strategy available when the join
  condition has no `=` operator (e.g. range joins), so it's the fallback of
  last resort.

### Hash Join
Build a hash table on the inner relation (keyed by the `=` join columns),
then scan the outer relation and probe the hash for each row.

- **Indexes**: none help — both relations are scanned sequentially.
- **Best when**: neither relation is small, but the hash table for the
  smaller one fits in `work_mem`. If it doesn't fit, PostgreSQL has to batch
  the hash to disk, which hurts performance — the planner usually switches
  to a merge join instead.
- **Requires** at least one `=` join condition.

### Merge Join
Sort both relations on the join key, then walk them in lockstep matching
rows.

- **Indexes**: an index on the join key of both relations can avoid an
  explicit sort — but an explicit sort is often cheaper unless an
  index-only scan is possible.
- **Best when**: both relations are too large for a hash join to fit in
  `work_mem`. The go-to strategy for joining very large tables.
- **Requires** at least one `=` join condition (like hash join).

### Summary

| | Nested Loop | Hash Join | Merge Join |
|---|---|---|---|
| Algorithm | For each outer row, scan inner | Build hash from inner, probe with outer | Sort both, merge |
| Helpful indexes | Join key on inner relation | None | Join key on both relations |
| Good when | Outer relation is small | Hash fits in `work_mem` | Both relations are large |

---

## 4. Diagnosing and tuning join strategy

```sql
EXPLAIN (COSTS OFF)
SELECT * FROM a JOIN b USING (id);
```

Wrong strategy usually traces back to a bad row-count estimate, not the join
itself:
- Underestimated rows → planner picks nested loop by mistake → scans the
  inner relation far more than expected.
- Overestimated rows → planner picks hash/merge by mistake → scans both
  relations fully, which can be worse than an indexed nested loop.

Levers to try, in order of how targeted they are:
1. `ANALYZE` the tables (optionally raise `default_statistics_target`) so
   row-count estimates improve.
2. Increase `work_mem` so a hash table fits without spilling to disk.
3. Tune `random_page_cost`, `effective_cache_size`,
   `effective_io_concurrency` so index scans are priced correctly.
4. Add covering indexes (`INCLUDE`) to enable index-only scans, speeding up
   nested loop and merge joins — keep the table vacuumed for this to help.

For experimentation only, individual strategies can be discouraged (not
fully disabled) per session:

```sql
SET enable_nestloop = off;
SET enable_hashjoin = off;
SET enable_mergejoin = off;
```