# PostgreSQL `SELECT`

## 1. Basic syntax

```sql
SELECT [ALL | DISTINCT | DISTINCT ON (expr)] expressions
FROM tables
[WHERE conditions]
[GROUP BY expressions]
[HAVING condition]
[ORDER BY expression [ASC | DESC] [NULLS FIRST | NULLS LAST]]
[LIMIT number_rows]
[OFFSET offset_value]
[FETCH FIRST fetch_rows ROWS ONLY]
[FOR UPDATE | FOR SHARE];
```

- **expressions** — columns/fields to return (`*` = all columns).
- **tables** — at least one table in `FROM`.
- **conditions** — row filter in `WHERE`.

All clauses except `SELECT`/`FROM` are optional.

---

## 2. Select all fields

```sql
SELECT * FROM employees;
```

Avoid `SELECT *` on wide/large tables in real systems — it forces reading every
column, and can prevent index-only scans.

---

## 3. Select individual fields / alias

```sql
SELECT first_name, last_name
FROM employees
ORDER BY last_name ASC;
```

Alias with `AS` (optional keyword):

```sql
SELECT product_id AS id
FROM Orders;
```

---

## 4. Filtering with `WHERE`

```sql
SELECT title
FROM books
WHERE genre_id = 3;
```

### Combining conditions: `AND`, `OR`, `NOT`

```sql
SELECT product_name, price, stock
FROM Products
WHERE price <= 20 OR stock >= 500;
```

### NULL-safe filtering

`=`, `!=`, and most operators return `NULL` (not `TRUE`/`FALSE`) when compared
against `NULL` — so `manager_id != 5` silently **excludes** rows where
`manager_id IS NULL`. To keep them, be explicit:

```sql
SELECT employee_name
FROM Employees
WHERE manager_id IS NULL OR manager_id != 5;
```

### Self-column comparison (row-to-row, not row-to-constant)

```sql
SELECT DISTINCT customer_id AS id
FROM Orders
WHERE customer_id = shipped_to_id
ORDER BY id;
```

Comparing two columns of the *same row* can't use a plain single-column
index — Postgres has to scan.

---

## 5. `DISTINCT`

Removes duplicate rows from the result set.

```sql
SELECT DISTINCT department
FROM Employees
ORDER BY department;
```

`DISTINCT` is a **keyword**, not a function. `DISTINCT(col)` is not "distinct
applied to just `col`" — the parentheses are just grouping parentheses, not
function arguments, so `DISTINCT` still applies to the **whole row**. This:

```sql
SELECT DISTINCT(department), job_title FROM Employees;
```

means the same as:

```sql
SELECT DISTINCT department, job_title FROM Employees;
```

i.e. distinct `(department, job_title)` pairs — **not** "distinct department,
plus job_title tagging along," which is what the parentheses make it *look*
like. Write `DISTINCT col` (no parentheses) so this is clear at a glance.

If you actually want uniqueness on just one column while still returning
other columns, use `DISTINCT ON (expr)` (Postgres-specific) instead — it
keeps only the first row per group of `expr`, based on `ORDER BY`:

```sql
SELECT DISTINCT ON (department) department, job_title, salary
FROM Employees
ORDER BY department, salary DESC;
```

This keeps exactly one row per `department` — the highest-paid one, because
of `ORDER BY ... salary DESC` — while still returning `job_title` and
`salary` from that chosen row.

---

## 6. Pattern matching: `LIKE`

| Wildcard | Meaning                          |
|----------|-----------------------------------|
| `%`      | zero or more characters           |
| `_`      | exactly one character             |
| `^`      | start of pattern (regex, not LIKE)|
| `[a-z]`  | character range (regex, not LIKE) |
| `[^a-e]` | NOT in range (regex, not LIKE)    |

```sql
SELECT email FROM Users WHERE email LIKE '%@gmail.com';   -- ends with domain
SELECT sku FROM Inventory WHERE sku LIKE 'A_-___';        -- fixed-length pattern
```

For real regex ranges (`[a-z]`, `[^a-e]`) use `~` (POSIX regex) in Postgres:

```sql
SELECT last_name FROM Employees WHERE last_name ~ '^[M-Z]';
```

---

## 7. Sorting: `ORDER BY`

```sql
SELECT department, salary
FROM Employees
ORDER BY department ASC, salary DESC;
```

- Multiple columns → tie-breaking order matters (left to right).
- `NULLS FIRST` / `NULLS LAST` controls where NULLs sort (Postgres default:
  `NULLS LAST` for `ASC`, `NULLS FIRST` for `DESC`).

---

## 8. Aggregate functions

```sql
SELECT MIN(price), MAX(price), SUM(quantity), AVG(price), COUNT(*)
FROM OrderItems;
```

- Used alone → one summary row.
- Used with `GROUP BY` → one row per group.
- `HAVING` filters *groups* (after aggregation); `WHERE` filters *rows*
  (before aggregation).

```sql
SELECT category, AVG(price) AS avg_price
FROM Products
GROUP BY category
HAVING AVG(price) > 100;
```

---

## 9. Function calls in `WHERE` — sargability

```sql
SELECT customer_id FROM Reviews WHERE LENGTH(comment) > 200;
```

Wrapping a column in a function (`LENGTH`, `CHAR_LENGTH`, `LOWER`, etc.) makes
the predicate **non-sargable**: a plain index on `comment` can't be used,
because Postgres must compute the function for every row first.

**Fix (real-world, large tables):** an expression index.

```sql
CREATE INDEX idx_reviews_comment_length ON Reviews (CHAR_LENGTH(comment));
```

---

## 10. Concatenation

```sql
SELECT city || ', ' || country AS location
FROM Addresses;
```

`||` concatenates strings; `CONCAT()` and `CONCAT_WS()` are alternatives that
handle `NULL` more gracefully (`||` with a `NULL` operand yields `NULL`,
`CONCAT()` skips NULLs).

---

## 11. Calculations without `FROM`

```sql
SELECT 42 * 7 / 3;   -- pure expression, no table needed
```