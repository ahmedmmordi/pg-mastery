# Advanced Select and Joins

## 1. `CASE WHEN` as a general conditional expression

Not just for conditional aggregation (covered in the Aggregate notes) —
`CASE WHEN` is a full expression that can appear anywhere a value can:
`SELECT`, `WHERE`, `ORDER BY`, even inside another `CASE WHEN`.

```sql
SELECT name, salary,
       CASE
         WHEN salary >= 100000 THEN 'high'
         WHEN salary >= 50000  THEN 'mid'
         ELSE 'low'
       END AS salary_band
FROM Employees;
```

- Conditions are checked **top to bottom**; the first match wins, the rest
  are skipped — order conditions from most to least specific.
- Omitting `ELSE` means non-matching rows get `NULL`, not an error.
- Multi-condition classification (like validating a shape, checking a
  business rule) is a straightforward `CASE WHEN` with `AND`/`OR` inside
  each branch:

```sql
SELECT a, b, c,
       CASE
         WHEN a + b > c AND a + c > b AND b + c > a THEN 'Yes'
         ELSE 'No'
       END AS is_valid
FROM Triangles;
```

---

## 2. `UNION` vs `UNION ALL`

```sql
SELECT city FROM Customers
UNION
SELECT city FROM Suppliers;
-- removes duplicate rows across both result sets (does an implicit dedup pass)

SELECT city FROM Customers
UNION ALL
SELECT city FROM Suppliers;
-- keeps every row from both, including duplicates — faster, no dedup step
```

Use `UNION ALL` by default unless duplicates genuinely need removing —
`UNION`'s dedup pass isn't free (it's effectively a sort/hash over the
combined result).

**Tricky pattern: guaranteeing every category appears, even with zero
rows.** A plain `GROUP BY` only outputs groups that exist in the data — if
a category has no matching rows at all, it's silently missing from the
result, not shown with a `0`.

```sql
-- Wrong: 'no_activity' users never appear if zero rows match that bucket
SELECT
  CASE WHEN days_since_login <= 7 THEN 'active'
       WHEN days_since_login <= 30 THEN 'lapsing'
       ELSE 'inactive' END AS bucket,
  COUNT(*) AS user_count
FROM Users
GROUP BY bucket;

-- Fix: build a "spine" of all possible categories, LEFT JOIN counts onto it
SELECT b.bucket, COUNT(u.user_id) AS user_count
FROM (
  SELECT 'active' AS bucket
  UNION ALL SELECT 'lapsing'
  UNION ALL SELECT 'inactive'
) b
LEFT JOIN Users u
  ON (b.bucket = 'active'   AND u.days_since_login <= 7)
  OR (b.bucket = 'lapsing'  AND u.days_since_login BETWEEN 8 AND 30)
  OR (b.bucket = 'inactive' AND u.days_since_login > 30)
GROUP BY b.bucket;
```

The `UNION ALL` of literal rows manufactures a fixed set of categories that
always exist, independent of the data. `LEFT JOIN`-ing the real data onto
that spine (instead of `GROUP BY`-ing the data directly) guarantees every
category shows up, with `0` via `COUNT(u.user_id)` (which ignores the
`NULL`s a `LEFT JOIN` produces for unmatched categories) if nothing matched.

---

## 3. Window functions vs. self-joins/correlated subqueries

Two families of SQL solve "compare this row to other rows" problems: older
self-join/correlated-subquery patterns, and window functions
(`OVER (...)`), which are usually cleaner and often faster.

### Running totals

```sql
-- Self-join / correlated subquery way (older, works everywhere):
SELECT a.id, a.amount,
       (SELECT SUM(b.amount) FROM Ledger b WHERE b.id <= a.id) AS running_total
FROM Ledger a;

-- Window function way (cleaner, one pass):
SELECT id, amount,
       SUM(amount) OVER (ORDER BY id) AS running_total
FROM Ledger;
```

`SUM(amount) OVER (ORDER BY id)` computes, for each row, the sum of
`amount` across all rows from the start up through the current row (the
default window frame for an ordered window is "start to current row") —
exactly a running total, without a self-join or repeated subquery per row.

**Tricky use**: picking "the last row before a cumulative total crosses a
threshold" (e.g. "how many people fit on the bus before total weight
exceeds capacity") combines a running total with a filter:

```sql
SELECT id
FROM (
  SELECT id, SUM(weight) OVER (ORDER BY arrival_time) AS running_weight
  FROM Passengers
) t
WHERE running_weight <= 1000
ORDER BY running_weight DESC
LIMIT 1;
```

### Row-to-row comparison: `LAG`/`LEAD`

```sql
SELECT id, value,
       LAG(value) OVER (ORDER BY id) AS prev_value,
       LEAD(value) OVER (ORDER BY id) AS next_value
FROM Readings;
```

`LAG(col)` pulls the value from the **previous** row (by the given
`ORDER BY`); `LEAD(col)` pulls it from the **next** row. Useful for
"compare to yesterday," gap detection, or finding sequences.

**Tricky pattern: N consecutive rows with the same value.**

```sql
-- Self-join way: match row, row+1, row+2 all having the same value
SELECT DISTINCT a.value
FROM Logs a
JOIN Logs b ON b.id = a.id + 1 AND b.value = a.value
JOIN Logs c ON c.id = a.id + 2 AND c.value = a.value;

-- Window function way: compare each row to the 2 before it
SELECT DISTINCT value
FROM (
  SELECT value,
         LAG(value, 1) OVER (ORDER BY id) AS prev1,
         LAG(value, 2) OVER (ORDER BY id) AS prev2
  FROM Logs
) t
WHERE value = prev1 AND value = prev2;
```

The self-join version relies on `id` being contiguous (no gaps) for
`id + 1`/`id + 2` to correctly mean "the next row" — if `id`s can have
gaps, this silently breaks. `LAG(value, 2)` always means "two rows back in
the given order," regardless of whether the underlying `id`s are
contiguous — more robust, and it's a single scan instead of two self-joins.

---

## 4. Ranking functions and `PARTITION BY`

Arguably the highest-leverage window function topic — these three come up
constantly, both in interview-style SQL and in real reporting queries.

### `PARTITION BY` — resetting a window per group

Everything in section 3 computed `OVER (ORDER BY ...)` across the **whole**
result set. `PARTITION BY` restarts the window for each group, the same
way `GROUP BY` would — except rows aren't collapsed, so every original row
still appears.

```sql
SELECT department, name, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary
FROM Employees;
```

Every employee's row is kept, but `dept_avg_salary` is computed only from
rows in that *same* department — like a `GROUP BY` average, but without
losing per-employee detail. `PARTITION BY` and `ORDER BY` combine freely:
`OVER (PARTITION BY department ORDER BY salary DESC)` resets a running
computation at the start of each department, ordered within it.

### `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` — three ways to number rows

```sql
SELECT name, department, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
       RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk
FROM Employees;
```

| salary | `ROW_NUMBER()` | `RANK()` | `DENSE_RANK()` |
|---|---|---|---|
| 100 | 1 | 1 | 1 |
| 90  | 2 | 2 | 2 |
| 90  | 3 | 2 | 2 |
| 80  | 4 | 4 | 3 |

- **`ROW_NUMBER()`** — always `1, 2, 3, 4, ...`, strictly unique per row,
  even for exact ties (ties are broken arbitrarily by whatever order the
  engine encounters them in beyond the `ORDER BY` — if the `ORDER BY`
  column itself has duplicates, two runs can number the tied rows
  differently unless the `ORDER BY` is made fully unique, e.g. by adding
  `id` as a tiebreaker).
- **`RANK()`** — ties share the same rank, but the **next** rank skips
  ahead by the number of tied rows (`1, 2, 2, 4` — no `3`).
- **`DENSE_RANK()`** — ties share the same rank, and the next rank is
  always just `+1` with no gap (`1, 2, 2, 3`).

**Tricky, very common pattern: "the Nth highest value per group."**

```sql
-- 2nd highest salary in each department
SELECT department, name, salary
FROM (
  SELECT department, name, salary,
         DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
  FROM Employees
) t
WHERE rnk = 2;
```

`DENSE_RANK()` is usually the right choice here over `ROW_NUMBER()`: if two
employees are tied for the highest salary, `ROW_NUMBER()` would arbitrarily
call one of them "1st" and the other "2nd," incorrectly excluding a true
top earner from a "top 1" query. `DENSE_RANK()` correctly gives both of
them rank `1`, and the "2nd highest" then means the next *distinct* salary
value, not just the next row. `RANK()` would also give both `1`, but a
"2nd highest" query with `RANK()` would jump straight to whatever is
numbered `3` (skipping `2` entirely, since two rows tied at `1`) — meaning
`RANK() = 2` might not exist for that department at all if there's a tie
at the top. `ROW_NUMBER()` is the right pick instead when the goal is
genuinely "one row per group, no matter what" (e.g. deduplication — see
below), where ties don't need special handling.

**Tricky pattern: deduplication — keep only the first row per group.**

```sql
SELECT *
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at ASC) AS rn
  FROM Users
) t
WHERE rn = 1;
```

`ROW_NUMBER()` is the right tool here, not `RANK()`/`DENSE_RANK()` —
deduplication needs exactly one winner per group even if the `ORDER BY`
column ties, and `ROW_NUMBER()` is the only one of the three guaranteed to
produce a single `1` per partition.

---

## 5. "Latest value as of a given date" — a very common correlated pattern

```sql
SELECT p.product_id, p.new_price
FROM PriceChanges p
WHERE p.change_date = (
  SELECT MAX(p2.change_date)
  FROM PriceChanges p2
  WHERE p2.product_id = p.product_id AND p2.change_date <= '2019-08-16'
);
```

Same family as the earlier "row with MIN/MAX per group" pattern, but
bounded by a cutoff date: for each product, find its most recent price
change *on or before* the target date, not just the overall latest.

**Tricky part**: a product that has **no** price change on or before the
target date needs a default value (often `10`, or whatever the problem
specifies) — the query above would simply omit that product entirely,
since the subquery returns no rows to match against. Handling that usually
means a `LEFT JOIN` against a spine of all products (see section 2) with
`COALESCE` to the default, rather than a bare correlated subquery.

This same "one row per group" goal can also be reached with `ROW_NUMBER()`
from section 4 instead of a correlated subquery:

```sql
SELECT product_id, new_price
FROM (
  SELECT product_id, new_price,
         ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY change_date DESC) AS rn
  FROM PriceChanges
  WHERE change_date <= '2019-08-16'
) t
WHERE rn = 1;
```

Same result, one scan instead of a correlated subquery re-run per row —
generally the faster option on larger tables.

---

## 6. Understanding `ON` conditions

### Equi-joins vs. non-equi joins

Most joins use `=` (an **equi-join**), but `ON` accepts *any* boolean
condition — `>`, `<`, `>=`, `<=`, `!=`, `BETWEEN`, even `OR`.

```sql
-- Non-equi join: pair each employee with every employee who earns less
SELECT a.name AS higher_earner, b.name AS lower_earner
FROM Employees a
JOIN Employees b ON a.salary > b.salary;
```

Non-equi joins can't use a hash join (hashing only works for `=`), so
PostgreSQL falls back to a nested loop — potentially expensive on large
tables, since there's no efficient index structure for "match everything
greater than."

### Range overlaps and time-based joins

Two of the most common non-equi patterns:

```sql
-- "Does this event's timestamp fall inside this record's validity window?"
SELECT s.sale_id, p.price
FROM Sales s
JOIN Prices p
  ON s.product_id = p.product_id
  AND s.sale_date BETWEEN p.start_date AND p.end_date;

-- "Do these two date ranges overlap at all?" (classic interval-overlap test)
SELECT a.booking_id, b.booking_id
FROM Bookings a
JOIN Bookings b
  ON a.room_id = b.room_id
  AND a.booking_id != b.booking_id
  AND a.start_date <= b.end_date
  AND a.end_date >= b.start_date;
```

The overlap condition `a.start <= b.end AND a.end >= b.start` is the
standard interval-overlap test — it's easy to get backwards (people often
try `a.start <= b.start AND a.end >= b.end`, which only catches one range
fully containing the other, not partial overlaps).

### Multiple `ON` conditions

Combine with `AND` for "these columns must all match" (e.g. matching a
`'start'` row to an `'end'` row for the same machine and process, as in the
Activity self-join from the Joins notes):

```sql
SELECT a.machine_id, b.timestamp - a.timestamp AS duration
FROM Activity a
JOIN Activity b
  ON a.machine_id = b.machine_id
  AND a.process_id = b.process_id
  AND a.activity_type = 'start'
  AND b.activity_type = 'end';
```

Every `AND`-ed condition narrows what counts as a match. Mixing `OR` into
`ON` is valid but changes the semantics significantly — it widens what
counts as a match rather than narrowing it, so it's easy to accidentally
produce far more matched pairs than intended.

### Correlated subqueries in `ON`

`ON` can contain a subquery that references the tables being joined —
same correlated-subquery mechanics as in `WHERE`, just placed in `ON`
instead (this is ordinary SQL, not the `LATERAL` join feature, which is a
different — more powerful — mechanism for subqueries that need to appear
in `FROM`).

```sql
SELECT o.order_id, c.customer_id
FROM Orders o
JOIN Customers c
  ON o.customer_id = c.customer_id
  AND EXISTS (
    SELECT 1 FROM VipList v WHERE v.customer_id = c.customer_id
  );
```

This restricts the join itself to only VIP customers — different from
putting the same `EXISTS` in `WHERE`, which (for an `INNER JOIN`) would
give an identical result, but would behave differently if this were a
`LEFT JOIN` (same `ON`-vs-`WHERE` distinction as the Aggregate notes'
`BETWEEN` example: conditions in `ON` don't remove already-unmatched left
rows; the same condition in `WHERE` would).

### `BETWEEN`, `IN`, and `EXISTS` — where each fits

| | Typical use | Tricky note |
|---|---|---|
| `BETWEEN a AND b` | Range check on a single column | Inclusive on both ends; just shorthand for `>= a AND <= b` |
| `IN (list or subquery)` | Membership test against a fixed list or a subquery's result | `NOT IN` breaks silently if the list/subquery can contain `NULL` (see the Joins notes) |
| `EXISTS (correlated subquery)` | "Does at least one matching row exist?" | Stops at the first match (short-circuits); doesn't care about the subquery's actual column values, only whether any row comes back — `SELECT 1` inside is idiomatic since the selected value is irrelevant |

All three can appear inside `ON` as well as `WHERE` — the difference
between putting a condition in one place vs. the other only matters for
outer joins, as covered above.