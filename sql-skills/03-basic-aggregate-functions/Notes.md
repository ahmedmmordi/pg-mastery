# Aggregate Functions

## 1. The core five

```sql
SELECT COUNT(*), COUNT(salary), SUM(salary), AVG(salary), MIN(salary), MAX(salary)
FROM Employees;
```

- `COUNT(*)` — counts **rows**, including rows where every column is `NULL`.
- `COUNT(col)` — counts only rows where `col` is **not** `NULL`.
- `SUM`, `AVG`, `MIN`, `MAX` — all **ignore `NULL`s** in the column they're
  applied to; they don't error, don't count them, and don't treat them as 0.
- With no `GROUP BY`, an aggregate collapses the *entire table* into one row.
- With `GROUP BY`, one aggregate result per group.

```sql
SELECT department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM Employees
GROUP BY department;
```

---

## 2. `WHERE` vs `HAVING`

```sql
SELECT department, AVG(salary) AS avg_salary
FROM Employees
WHERE hire_date >= '2020-01-01'   -- filters ROWS, before grouping
GROUP BY department
HAVING AVG(salary) > 80000;       -- filters GROUPS, after aggregating
```

You cannot write `WHERE AVG(salary) > 80000` — aggregates don't exist yet
when `WHERE` runs. That's what `HAVING` is for.

---

## 3. Every non-aggregated column must be in `GROUP BY`

```sql
-- Wrong: department appears in SELECT but not GROUP BY (or an aggregate)
SELECT department, name, AVG(salary) FROM Employees GROUP BY department;
```

PostgreSQL rejects this — for any column you `SELECT` outside of an
aggregate function, either it's in `GROUP BY`, or the query doesn't make
sense (which `name` would it show for a group of many rows?).

---

## 4. `COUNT(DISTINCT col)`

```sql
SELECT COUNT(DISTINCT department) AS num_departments
FROM Employees;
```

Counts unique non-`NULL` values only — duplicates collapse to one, and
`NULL`s still don't count.

---

## 5. Conditional aggregation — the tricky, high-value pattern

Counting/summing only rows that meet a condition, without a separate query
per condition.

**Pattern A — `CASE WHEN` inside the aggregate (works everywhere):**

```sql
SELECT
  COUNT(CASE WHEN status = 'completed' THEN 1 END) AS completed_count,
  COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled_count
FROM Orders;
```

This works because `CASE WHEN ... THEN 1 END` returns `NULL` (implicit
`ELSE NULL`) for non-matching rows, and `COUNT()` ignores `NULL`s — so it
only counts the rows where the condition was true. This is the same trick
as `SUM(CASE WHEN ... THEN amount ELSE 0 END)` for conditional totals.

**Pattern B — `FILTER` (PostgreSQL-specific, cleaner):**

```sql
SELECT
  COUNT(*) FILTER (WHERE status = 'completed') AS completed_count,
  COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled_count
FROM Orders;
```

Same result as Pattern A, but reads more clearly. Not standard SQL though —
`CASE WHEN` is the portable choice if the query needs to run outside
Postgres.

---

## 6. Turning a boolean condition into a rate — a very common trap

```sql
SELECT user_id,
       ROUND(AVG((action = 'confirmed')::int), 2) AS confirmation_rate
FROM Confirmations
GROUP BY user_id;
```

`(action = 'confirmed')` is a boolean; `::int` casts `TRUE`/`FALSE` to
`1`/`0`. `AVG()` of a column of 1s and 0s is exactly the proportion of rows
where the condition held — a fast, common way to compute a rate without a
subquery or `CASE`.

**Tricky part**: `AVG()` divides by the count of **non-NULL** rows, not
the count of all rows in the group. If some rows have `action IS NULL`,
they're silently excluded from both the numerator and denominator — which
can be exactly what you want, or exactly the bug you didn't notice.

---

## 7. Division: integer truncation and divide-by-zero

```sql
-- Wrong on integer columns in some engines: integer division truncates.
-- PostgreSQL: int / int = int (floors toward zero), so this can silently
-- lose precision:
SELECT SUM(completed) / SUM(total) FROM Stats;

-- Fix: cast at least one side to numeric/float before dividing
SELECT ROUND(SUM(completed)::numeric / SUM(total), 2) FROM Stats;
```

```sql
-- Divide-by-zero: SUM(total) could be 0 for some group, which errors
-- (Postgres) or returns NULL/Infinity depending on the engine.
SELECT ROUND(SUM(completed)::numeric / NULLIF(SUM(total), 0), 2) FROM Stats;
```

`NULLIF(a, b)` returns `NULL` if `a = b`, otherwise `a`. Here it turns a `0`
denominator into `NULL`, so the division returns `NULL` instead of
erroring — then wrap the whole thing in `COALESCE(..., 0)` if you want `0`
displayed instead of `NULL`.

---

## 8. `COALESCE` for groups with no matching rows

```sql
SELECT s.user_id,
       ROUND(COALESCE(AVG((c.action = 'confirmed')::int), 0), 2) AS rate
FROM Signups s
LEFT JOIN Confirmations c ON s.user_id = c.user_id
GROUP BY s.user_id;
```

If a user has zero rows in `Confirmations` (only `NULL`s from the
`LEFT JOIN`), `AVG()` over zero non-`NULL` values returns `NULL`, not `0`.
`COALESCE(..., 0)` substitutes `0` only in that case, so users with no
activity show a `0%` rate instead of a blank one.

---

## 9. Rounding gotcha: casting before `ROUND()`

```sql
-- Can fail in Postgres: ROUND() doesn't have a two-argument overload
-- for double precision, only for numeric
SELECT ROUND(AVG(salary), 2) FROM Employees;          -- fine, AVG already numeric-ish
SELECT ROUND(SUM(x - y), 2) FROM Events;               -- fine if x,y are numeric/int
SELECT ROUND((b.timestamp - a.timestamp)::numeric, 3)  -- explicit cast needed
FROM Activity a JOIN Activity b ON ...;
```

When the value being rounded comes from subtracting two `timestamp`s or
other non-`numeric` types, PostgreSQL may produce a type `ROUND(x, n)`
doesn't accept with a fixed decimal count — cast to `::numeric` first.

---

## 10. Aggregates that aren't COUNT/SUM/AVG/MIN/MAX (PostgreSQL extras)

```sql
-- Concatenate group values into one string
SELECT department, STRING_AGG(name, ', ' ORDER BY name) AS employees
FROM Employees
GROUP BY department;

-- Collect group values into an array
SELECT department, ARRAY_AGG(name ORDER BY name) AS employees
FROM Employees
GROUP BY department;
```

`STRING_AGG`/`ARRAY_AGG` are genuinely useful for "list all X per group"
problems — the kind of thing that looks like it needs a loop but is a
single aggregate call.

---

## 11. Aggregate on an empty result set — no rows at all

```sql
SELECT COUNT(*), SUM(salary), AVG(salary) FROM Employees WHERE department = 'ghost';
```

- `COUNT(*)` on zero matching rows → `0` (never `NULL`).
- `SUM`, `AVG`, `MIN`, `MAX` on zero matching rows → `NULL` (not `0`).

This asymmetry is a frequent source of bugs: code that assumes "no data
means 0" breaks silently on `SUM`/`AVG`/`MIN`/`MAX`, but works fine on
`COUNT`. Wrap the non-`COUNT` aggregates in `COALESCE(..., 0)` if a numeric
default is expected downstream.