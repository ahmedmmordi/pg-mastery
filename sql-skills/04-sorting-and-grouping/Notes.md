# Sorting and Grouping

## 1. `GROUP BY` basics

```sql
SELECT department, COUNT(*) AS headcount
FROM Employees
GROUP BY department;
```

Collapses rows sharing the same `department` value into one row per group,
with aggregates (`COUNT`, `SUM`, etc.) computed per group.

---

## 2. Grouping by multiple columns

```sql
SELECT department, job_title, AVG(salary) AS avg_salary
FROM Employees
GROUP BY department, job_title;
```

One row per **unique combination** of `department` and `job_title` — not
one row per department and a separate one per job title. Order of columns
in `GROUP BY` doesn't change the result, only how you might choose to
`ORDER BY` afterward.

---

## 3. `WHERE` vs `HAVING` — order of operations matters

```sql
SELECT department, AVG(salary) AS avg_salary
FROM Employees
WHERE hire_date >= '2020-01-01'   -- ① filters rows, BEFORE grouping
GROUP BY department               -- ② groups the surviving rows
HAVING AVG(salary) > 80000        -- ③ filters groups, AFTER aggregation
ORDER BY avg_salary DESC;         -- ④ sorts the final result
```

Actual execution order: `FROM` → `WHERE` → `GROUP BY` → `HAVING` →
`SELECT` → `ORDER BY` → `LIMIT`.

- Use `WHERE` for conditions on raw, ungrouped columns (cheaper — discards
  rows before the expensive aggregation step).
- Use `HAVING` for conditions on the aggregate result itself
  (`COUNT(*)`, `AVG(...)`, etc.) — these don't exist yet when `WHERE` runs.
- `SELECT department, AVG(salary) ... WHERE AVG(salary) > 80000` is invalid
  — the aggregate isn't computed until after `WHERE`.
- A condition that *could* be written either way should go in `WHERE` if
  it doesn't depend on an aggregate — it's more efficient, since it shrinks
  the data before grouping instead of after.

---

## 4. `ORDER BY` basics and multi-column sorting

```sql
SELECT department, salary
FROM Employees
ORDER BY department ASC, salary DESC;
```

Sorts by `department` first; within ties, sorts by `salary` descending.
Each column can have its own direction — `ASC` is the default if omitted.

---

## 5. Sorting by alias, position, or an expression not in `SELECT`

```sql
-- by column alias
SELECT department, AVG(salary) AS avg_salary
FROM Employees
GROUP BY department
ORDER BY avg_salary DESC;

-- by ordinal position (1-indexed) — works, but fragile if columns reorder
SELECT department, AVG(salary) AS avg_salary
FROM Employees
GROUP BY department
ORDER BY 2 DESC;

-- by an expression that isn't even selected
SELECT name, salary
FROM Employees
ORDER BY salary * 12 DESC;   -- sort by annual salary without selecting it
```

Sorting by alias is the clearest and safest of the three — position breaks
silently if the `SELECT` list changes, and repeating the full expression is
verbose.

---

## 6. `NULL`s in `ORDER BY`

```sql
SELECT name, manager_id
FROM Employees
ORDER BY manager_id ASC NULLS LAST;
```

PostgreSQL's default: `NULLS LAST` for `ASC`, `NULLS FIRST` for `DESC` —
opposite of what people often expect. Make it explicit with
`NULLS FIRST`/`NULLS LAST` if the ordering of `NULL`s actually matters for
the problem (e.g. employees with no manager should appear at the end of a
report).

---

## 7. Putting it together: `GROUP BY` + `HAVING` + `ORDER BY`

```sql
SELECT department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM Employees
WHERE status = 'active'
GROUP BY department
HAVING COUNT(*) >= 5
ORDER BY avg_salary DESC, department ASC;
```

Reads as: keep only active employees → group by department → keep only
departments with 5+ active employees → sort the surviving departments by
average salary (ties broken by department name).

---

## 8. Tricky pattern: "the row with MIN/MAX per group"

Getting an aggregate value (like `MIN(year)`) is easy — but getting the
*entire row* that has that value, per group, needs a bit more.

**Correlated subquery approach:**

```sql
SELECT o.product_id, o.order_year AS first_year, o.quantity
FROM Orders o
WHERE o.order_year = (
  SELECT MIN(o2.order_year)
  FROM Orders o2
  WHERE o2.product_id = o.product_id
);
```

For every outer row `o`, the subquery recomputes the minimum year *for that
same `product_id`* (that's what makes it "correlated" — it references the
outer query's row). The outer `WHERE` then keeps only rows that match their
own group's minimum — effectively "give me the first-year row per product."

This is different from just `SELECT product_id, MIN(order_year) FROM Orders
GROUP BY product_id`, which only returns the year, not the rest of that
row's columns (`quantity`, `price`, etc.) — `GROUP BY` alone can't return
non-aggregated, non-grouped columns.

---

## 9. Tricky pattern: filtering to "the single most extreme value" via nesting

```sql
SELECT MAX(num) AS num
FROM (
  SELECT num
  FROM MyNumbers
  GROUP BY num
  HAVING COUNT(*) = 1
) AS singles;
```

`HAVING COUNT(*) = 1` first narrows the groups down to values that appear
exactly once. Wrapping that in an outer `SELECT MAX(...)` then picks the
largest of those singles. This two-step shape — inner query does the
grouping/filtering, outer query does one final aggregate — comes up
whenever a problem needs "the biggest/smallest value that also satisfies
some group-level condition," since `HAVING` and a further aggregate on its
*result* can't both happen in one `GROUP BY` level.

An `ORDER BY ... DESC LIMIT 1` inside the subquery works too and is
sometimes clearer, but `MAX()` in the outer query is simpler when a plain
aggregate does the same job — no need to sort and cap at 1 just to find a
maximum.

---

## 10. `GROUP BY` with expressions

```sql
SELECT EXTRACT(YEAR FROM order_date) AS order_year, COUNT(*) AS order_count
FROM Orders
GROUP BY EXTRACT(YEAR FROM order_date);
```

`GROUP BY` isn't limited to raw columns — any expression can be a grouping
key, as long as the same expression (or its alias, in PostgreSQL) also
appears in `SELECT`. Common for bucketing dates into year/month, or
grouping by a computed category.