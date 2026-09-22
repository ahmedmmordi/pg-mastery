# Subqueries

A subquery is just a `SELECT` inside parentheses, used inside another statement. Each step below adds one idea, all on the same tiny dataset.

## 0. Setup

Working example used throughout — deliberately tiny.

```sql
CREATE TABLE departments (
  id int PRIMARY KEY,
  name text
);

CREATE TABLE employees (
  id int PRIMARY KEY,
  name text,
  salary numeric,
  dept_id int REFERENCES departments(id)
);

INSERT INTO departments VALUES
  (1,'Engineering'),
  (2,'Sales'),
  (3,'HR');

INSERT INTO employees VALUES
  (1,'Ali',    9000, 1),
  (2,'Sara',   7000, 1),
  (3,'Omar',   5000, 2),
  (4,'Mona',   6000, 2),
  (5,'Youssef',4000, NULL);
```

---

## 1. Scalar subquery (returns one value)

```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

The average is 6200, so the result is Ali and Sara. Read it as: the inner
query produces a single number, and the outer query uses it like a
constant.

**Common error**: if the inner query returns more than one row where a
single value is expected, PostgreSQL raises
`more than one row returned by a subquery used as an expression`.

```sql
-- ERROR: Engineering has 2 employees, so this returns 2 rows, not 1
WHERE salary = (SELECT salary FROM employees WHERE dept_id = 1)

-- Fix: say what you mean
WHERE salary = (SELECT MAX(salary) FROM employees WHERE dept_id = 1)
-- or use IN (section 2)
```

---

## 2. Multi-row subquery with `IN`

```sql
SELECT name
FROM employees
WHERE dept_id IN (SELECT id FROM departments WHERE name IN ('Engineering', 'Sales'));
```

The inner query returns a list (`1, 2`), and `IN` checks membership. The
result is Ali, Sara, Omar, Mona. Youssef is excluded because his `dept_id`
is `NULL` — `NULL IN (1, 2)` evaluates to `NULL` (unknown), not `TRUE`.

---

## 3. `ANY` and `ALL`

Compare a value against every row the subquery returns.

```sql
-- Earns more than EVERY Sales employee
SELECT name FROM employees
WHERE salary > ALL (SELECT salary FROM employees WHERE dept_id = 2);
-- Ali, Sara (must beat 5000 AND 6000)

-- Earns more than AT LEAST ONE Sales employee
SELECT name FROM employees
WHERE salary > ANY (SELECT salary FROM employees WHERE dept_id = 2);
-- Ali, Sara, Mona (only has to beat 5000)
```

Two equivalences worth memorizing: `= ANY (...)` is the same as
`IN (...)`, and `<> ALL (...)` is the same as `NOT IN (...)`.

---

## 4. `EXISTS` / `NOT EXISTS`

```sql
SELECT d.name
FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.id);
-- Engineering, Sales
```

`EXISTS` only asks "did this return any row?" — what you `SELECT` inside
doesn't matter (`SELECT 1` is the convention). Flip it to find departments
with no employees:

```sql
SELECT d.name
FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.id);
-- HR
```

### The `NOT IN` trap

The "obvious" version silently breaks:

```sql
SELECT name FROM departments
WHERE id NOT IN (SELECT dept_id FROM employees);
-- Returns ZERO rows!
```

The subquery list contains a `NULL` (Youssef's `dept_id`), so
`id NOT IN (1, 1, 2, 2, NULL)` can never be `TRUE` for any `id` — any
comparison against `NULL` is unknown, and `NOT IN` is really a chain of
`AND`-ed `!=` comparisons under the hood, so one unknown poisons the whole
chain. Use `NOT EXISTS`, or add `WHERE dept_id IS NOT NULL` inside the
subquery. **Prefer `NOT EXISTS` by default** — it doesn't have this trap
and it short-circuits at the first match.

---

## 5. Correlated subquery

So far, every inner query above could run on its own, standalone. A
**correlated** subquery references a column from the *outer* query, so it
gets re-evaluated for each outer row.

```sql
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (
  SELECT AVG(salary)
  FROM employees
  WHERE dept_id = e.dept_id      -- refers to the outer row
);
-- Ali (9000 > 8000), Mona (6000 > 5500)
```

Think of it as a loop: for Ali and Sara, compute the Engineering average;
for Omar and Mona, compute the Sales average. Youssef never qualifies,
because `dept_id = NULL` matches nothing.

The `EXISTS` example in section 4 was also correlated — it referenced
`d.id` from the outer query.

---

## 6. Subquery in `FROM` (derived table)

The subquery acts as a temporary table. PostgreSQL requires an alias for
it.

```sql
SELECT d.name, s.avg_salary
FROM departments d
JOIN (
  SELECT dept_id, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY dept_id
) AS s ON s.dept_id = d.id;
-- Engineering 8000, Sales 5500
```

Use this when you need to aggregate first, then join or filter on the
aggregated result — something you can't do in a single `WHERE`/`HAVING`
pass, since `HAVING` filters groups of the *same* query, not a
pre-computed result joined to something else.

---

## 7. Subquery in the `SELECT` list

Must return exactly one value per outer row (same scalar-subquery rule as
section 1, just placed differently).

```sql
SELECT d.name,
       (SELECT COUNT(*) FROM employees e WHERE e.dept_id = d.id) AS headcount
FROM departments d;
-- Engineering 2, Sales 2, HR 0
```

Notice HR appears with `0`. It's handy, but on big tables a
`LEFT JOIN ... GROUP BY` is often easier for the planner to optimize — a
correlated scalar subquery in `SELECT` re-runs once per outer row, while a
`LEFT JOIN` + `GROUP BY` computes the same result in one pass.

---

## 8. Nested subqueries — three levels deep

Subqueries can nest inside subqueries. This gets hard to read fast, which
is exactly the problem CTEs (next section) solve.

**"Which departments have an above-average department-average salary?"**
— note the two different averages involved: each department's own average,
and the average *of those averages*.

```sql
SELECT dept_id, dept_avg
FROM (                                          -- level 1: dept averages
  SELECT dept_id, AVG(salary) AS dept_avg
  FROM employees
  GROUP BY dept_id
) dept_avgs
WHERE dept_avg > (                              -- level 2: compare against...
  SELECT AVG(dept_avg)                          -- level 3: ...the average of
  FROM (                                        --           those averages
    SELECT dept_id, AVG(salary) AS dept_avg
    FROM employees
    GROUP BY dept_id
  ) all_dept_avgs
);
```

Three levels: the outer query filters `dept_avgs` (level 1, a derived
table), against a scalar subquery (level 2) that itself wraps another
derived table (level 3) — and that level-3 query is a near-duplicate of
level 1, computed twice. This duplication is the exact pain point a CTE
removes.

---

## 9. CTEs (`WITH` clause)

A **Common Table Expression** names a subquery once, up front, so it can
be referenced (even multiple times) later in the query — like a derived
table, but written before the main query instead of nested inline.

```sql
WITH dept_avgs AS (
  SELECT dept_id, AVG(salary) AS dept_avg
  FROM employees
  GROUP BY dept_id
)
SELECT dept_id, dept_avg
FROM dept_avgs
WHERE dept_avg > (SELECT AVG(dept_avg) FROM dept_avgs);
```

Same result as the three-level nested version in section 8, but
`dept_avgs` is computed once and referenced twice — no duplicated logic,
and it reads top-to-bottom instead of inside-out. This is the main reason
to reach for a CTE: **readability and avoiding repetition**, not raw
performance — PostgreSQL (16+) usually inlines a non-recursive CTE much
like a regular subquery, so don't expect a CTE alone to make a query
faster.

### Multiple CTEs

```sql
WITH dept_avgs AS (
  SELECT dept_id, AVG(salary) AS dept_avg FROM employees GROUP BY dept_id
),
high_earners AS (
  SELECT * FROM employees WHERE salary > 6000
)
SELECT h.name, d.dept_avg
FROM high_earners h
JOIN dept_avgs d ON h.dept_id = d.dept_id;
```

CTEs can reference earlier CTEs in the same `WITH`, and the main query can
join/filter across several of them — useful for breaking a complex query
into named, readable steps.

---

## 10. Recursive CTEs

For data with a self-referencing hierarchy (org charts, category trees,
bill-of-materials) or for generating a sequence, a plain CTE isn't enough
— it can't reference *itself*. `WITH RECURSIVE` can.

```sql
WITH RECURSIVE subordinates AS (
  -- base case: the starting row(s), no recursion yet
  SELECT employee_id, name, manager_id, 1 AS level
  FROM Employees
  WHERE employee_id = 101

  UNION ALL

  -- recursive case: join the CTE to the base table, one level deeper each time
  SELECT e.employee_id, e.name, e.manager_id, s.level + 1
  FROM Employees e
  JOIN subordinates s ON e.manager_id = s.employee_id
)
SELECT * FROM subordinates;
```

How it runs: the **base case** seeds the CTE with the starting row (manager
`101`). The **recursive case** then joins `Employees` back to the CTE's
*current* contents — on the first pass that finds `101`'s direct reports,
on the second pass it finds their reports, and so on. Each pass's new rows
become the input for the next pass. It stops automatically once a pass
produces zero new rows (e.g. the bottom of the org chart is reached).

**`UNION ALL`, not `UNION`, is the norm** here — recursive traversal of a
tree naturally produces no duplicate rows, so `UNION`'s extra dedup pass is
usually wasted work. (If the underlying data isn't a strict tree — e.g. it
has cycles — `UNION ALL` alone can loop forever; that's a case where either
`UNION` or explicit cycle-tracking is actually required.)

**Simpler example — generating a sequence of numbers:**

```sql
WITH RECURSIVE counter(n) AS (
  SELECT 1                              -- base case
  UNION ALL
  SELECT n + 1 FROM counter WHERE n < 10  -- recursive case + stopping condition
)
SELECT n FROM counter;
-- 1, 2, 3, ..., 10
```

The `WHERE n < 10` in the recursive branch is what makes it terminate —
without a stopping condition somewhere in the recursive case, the query
runs indefinitely (until it hits a resource limit and errors). Always make
sure the recursive branch has some condition that will eventually become
false for every row.

**A step up — Fibonacci sequence:**

```sql
WITH RECURSIVE fib(n, a, b) AS (
  SELECT 1, 0, 1                          -- base case: 1st term, a=0, b=1
  UNION ALL
  SELECT n + 1, b, a + b FROM fib WHERE n < 10  -- recursive case + stop
)
SELECT n, a AS fibonacci FROM fib;
-- 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

The counter example only needed to remember the previous value (`n`) to
compute the next one. Fibonacci needs the previous **two** values, but a
recursive CTE only ever sees the row produced by the last iteration, not
the whole history before it — so both values have to be carried forward
explicitly as columns. `a` is "the current Fibonacci number," `b` is "the
next one to be produced"; each iteration shifts them forward
(`a, a + b` becomes the next row's `b, a + b`... concretely, the next `a`
is the old `b`, and the next `b` is `old a + old b`). This "carry extra
state alongside the loop variable" trick generalizes to most recursive
CTEs that need more than just "the previous row's single value" — e.g.
running totals with more than one accumulator, or path-tracking in a graph
traversal.