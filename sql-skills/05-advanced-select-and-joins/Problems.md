# Advanced Select and Joins — Problems

## 1. The Number of Employees Which Report to Each Employee

[leetcode.com/problems/the-number-of-employees-which-report-to-each-employee](https://leetcode.com/problems/the-number-of-employees-which-report-to-each-employee/)

```sql
SELECT manager.employee_id, manager.name, COUNT(*) AS reports_count, ROUND(AVG(reporter.age)) AS average_age
FROM Employees manager
INNER JOIN Employees reporter ON manager.employee_id = reporter.reports_to
GROUP BY manager.employee_id, manager.name
ORDER BY manager.employee_id;
```

`INNER JOIN` here naturally excludes employees with zero reports — an
employee only appears as `manager` if at least one row in `reporter` points
to them via `reports_to`, so there's no need to filter separately for
"has at least one report." A `LEFT JOIN` would incorrectly include
employees with no reports at all (with `NULL` `reporter` columns), which
isn't asked for.

---

## 2. Primary Department for Each Employee

[leetcode.com/problems/primary-department-for-each-employee](https://leetcode.com/problems/primary-department-for-each-employee/)

```sql
SELECT employee_id, department_id
FROM Employee
WHERE primary_flag = 'Y'
   OR employee_id IN (
     SELECT employee_id FROM Employee GROUP BY employee_id HAVING COUNT(department_id) = 1
   );
```

Two different cases need combining: employees explicitly flagged
`primary_flag = 'Y'`, **and** employees who only belong to one department
at all (so that department is "primary" by default, even without the flag
set). The subquery finds employee_ids with exactly one department row via
`GROUP BY` + `HAVING COUNT(...) = 1`; the outer `WHERE ... OR employee_id
IN (...)` combines both cases into one filter, avoiding the need for a
`UNION` of two separate `SELECT`s.

---

## 3. Triangle Judgement

[leetcode.com/problems/triangle-judgement](https://leetcode.com/problems/triangle-judgement/)

```sql
SELECT x, y, z,
    CASE WHEN x + y > z AND x + z > y AND y + z > x THEN 'Yes'
    ELSE 'No' END AS triangle
FROM Triangle;
```

Pure row-by-row classification, no aggregation — `CASE WHEN` checks the
triangle inequality (the sum of any two sides must exceed the third) and
labels each row directly.

---

## 4. Consecutive Numbers

[leetcode.com/problems/consecutive-numbers](https://leetcode.com/problems/consecutive-numbers/)

```sql
SELECT num AS ConsecutiveNums
FROM (
    SELECT num,
        LAG(num) OVER (ORDER BY id) AS prev_value,
        LEAD(num) OVER (ORDER BY id) AS next_value
    FROM Logs
) x
WHERE x.num = x.prev_value AND x.num = x.next_value
GROUP BY num;
```

`LAG`/`LEAD` pull the previous/next row's `num` (ordered by `id`) onto the
same row, so a single `WHERE` can check "does this row match both its
neighbor before and after" — the definition of being the *middle* of three
consecutive equal values. `GROUP BY num` at the end removes duplicate
values if the same number has more than one qualifying run in the data
(the problem asks for each qualifying value once, not once per
occurrence).

---

## 5. Product Price at a Given Date

[leetcode.com/problems/product-price-at-a-given-date](https://leetcode.com/problems/product-price-at-a-given-date/)

```sql
SELECT p.product_id, COALESCE(x.new_price, 10) AS price
FROM (SELECT DISTINCT product_id FROM Products) p
LEFT JOIN (
    SELECT product_id, new_price
    FROM Products
    WHERE (product_id, change_date) IN (
        SELECT product_id, MAX(change_date)
        FROM Products
        WHERE change_date <= '2019-08-16'::DATE
        GROUP BY product_id
    )
) x
ON p.product_id = x.product_id;
```

Three pieces working together: the inner `(product_id, MAX(change_date))`
subquery finds each product's latest price-change date on or before the
cutoff; wrapping that in `(product_id, change_date) IN (...)` — a
**row-value `IN`** — matches whole `(product_id, change_date)` pairs at
once instead of matching each column separately (avoiding an extra self
join). The outer `p` subquery lists every distinct product regardless of
whether it has a qualifying price change; `LEFT JOIN`-ing the price lookup
onto that full product list, then `COALESCE(x.new_price, 10)`, gives
products with no price change before the cutoff the default price of `10`
instead of being dropped from the result.

---

## 6. Last Person to Fit in the Bus

[leetcode.com/problems/last-person-to-fit-in-the-bus](https://leetcode.com/problems/last-person-to-fit-in-the-bus/)

```sql
SELECT person_name
FROM (
    SELECT person_name, SUM(weight) OVER (ORDER BY turn) AS total_weight
    FROM Queue
) x
WHERE total_weight <= 1000
ORDER BY total_weight DESC
LIMIT 1;
```

`SUM(weight) OVER (ORDER BY turn)` computes a running total of weight in
boarding order. Filtering to `total_weight <= 1000` keeps every person who
still fits within the limit at their point in line; sorting that filtered
set descending by `total_weight` and taking the top row picks the person
whose running total is the largest while still being ≤ 1000 — i.e. the
last person who fits before the limit would be exceeded.

---

## 7. Count Salary Categories

[leetcode.com/problems/count-salary-categories](https://leetcode.com/problems/count-salary-categories/)

```sql
SELECT 'Low Salary' AS category, COUNT(*) AS accounts_count FROM Accounts WHERE income < 20000
UNION ALL
SELECT 'Average Salary' AS category, COUNT(*) FROM Accounts WHERE income BETWEEN 20000 AND 50000
UNION ALL
SELECT 'High Salary' AS category, COUNT(*) FROM Accounts WHERE income > 50000;
```

The "guarantee every category appears" pattern from the notes file, in its
simplest form: three separate `SELECT`s, each hard-coding one category
label and counting only the rows for that income range, stacked with
`UNION ALL`. Since each branch always produces exactly one row (`COUNT(*)`
never returns no rows — it returns `0` on an empty match, per the
Aggregate notes), all three categories are guaranteed to appear even if one
of them has zero matching accounts. `UNION ALL` (not `UNION`) is correct
here since the three category labels are already distinct by construction
— there's nothing to deduplicate, so plain `UNION`'s extra dedup pass would
be wasted work.