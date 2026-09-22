# Advanced String Functions / Regex — Problems

## 1. Fix Names in a Table

[leetcode.com/problems/fix-names-in-a-table](https://leetcode.com/problems/fix-names-in-a-table/)

```sql
SELECT user_id, UPPER(SUBSTRING(name, 1, 1)) || LOWER(SUBSTRING(name FROM 2)) AS name
FROM Users
ORDER BY user_id;
```

OR:

```sql
SELECT user_id, CONCAT(UPPER(SUBSTRING(name, 1, 1)), LOWER(SUBSTRING(name FROM 2))) AS name
FROM Users
ORDER BY user_id;
```

`SUBSTRING(name, 1, 1)` grabs just the first character, `UPPER()`-ed;
`SUBSTRING(name FROM 2)` grabs everything from the second character
onward, `LOWER()`-ed — combined, that's a manual "capitalize first letter,
lowercase the rest." `||` and `CONCAT()` are interchangeable here since
neither operand can be `NULL` (both are derived from `name`, which the
problem guarantees is present) — the `NULL`-poisoning difference between
them from the notes file doesn't come into play in this particular case.

---

## 2. Patients With a Condition

[leetcode.com/problems/patients-with-a-condition](https://leetcode.com/problems/patients-with-a-condition/)

```sql
SELECT *
FROM Patients
WHERE conditions LIKE 'DIAB1%' OR conditions LIKE '% DIAB1%';
```

OR:

```sql
SELECT * FROM Patients WHERE conditions ~ '(^| )DIAB1';
```

`conditions` holds a space-separated list of codes, and the target code
`DIAB1` needs to be matched as a **whole code**, not just a substring
match anywhere (which would wrongly also match something like `DIAB100`).
The `LIKE` version checks two cases explicitly: the string starts with
`DIAB1` directly, or `DIAB1` appears right after a space somewhere in the
middle/end. The regex version expresses the same "start of string, or
right after a space" idea in one pattern: `(^|space)DIAB1` — `^` anchors to
the string's start, `|` is alternation ("or"), so the match point must be
either the very beginning or immediately after a space.

---

## 3. Delete Duplicate Emails

[leetcode.com/problems/delete-duplicate-emails](https://leetcode.com/problems/delete-duplicate-emails/)

```sql
DELETE FROM Person a
USING Person b
WHERE a.email = b.email AND a.id > b.id;
```

`USING` (PostgreSQL's extension to `DELETE`) lets a `DELETE` reference a
second copy of the table to compare against — effectively a self join
inside a `DELETE`. For every pair of rows sharing the same `email`, this
keeps the one with the smaller `id` and deletes every row whose `id` is
larger than some other row with the same email — leaving exactly one
(the earliest-inserted) row per email.

---

## 4. Second Highest Salary

[leetcode.com/problems/second-highest-salary](https://leetcode.com/problems/second-highest-salary/)

```sql
SELECT (
    SELECT DISTINCT salary
    FROM Employee
    ORDER BY salary DESC
    LIMIT 1 OFFSET 1
) AS SecondHighestSalary;
```

`DISTINCT` first collapses duplicate salary values so ties at the top
don't distort "second highest" into just "second row." `ORDER BY salary
DESC` puts the highest first, and `OFFSET 1` skips it, then `LIMIT 1`
takes the next one — the second-highest *distinct* value. Wrapping the
whole thing in an outer `SELECT (...)` (a scalar subquery with nothing
selected from) is what makes a missing second-highest salary (fewer than
2 distinct salaries exist) come back as `NULL` instead of returning zero
rows — this trick guarantees exactly one output row no matter what the
inner query finds.

---

## 5. Group Sold Products By The Date

[leetcode.com/problems/group-sold-products-by-the-date](https://leetcode.com/problems/group-sold-products-by-the-date/)

```sql
SELECT sell_date, COUNT(DISTINCT product) AS num_sold,
       STRING_AGG(DISTINCT product, ',' ORDER BY product) AS products
FROM Activities
GROUP BY sell_date
ORDER BY sell_date;
```

`COUNT(DISTINCT product)` counts unique products sold per day, and
`STRING_AGG(DISTINCT product, ',' ORDER BY product)` builds the
comma-separated product list — the `STRING_AGG` from the Aggregate notes'
section on lesser-known aggregates, here combined with `DISTINCT` (so a
product sold multiple times in a day appears once in the list) and
`ORDER BY product` (so the list comes out alphabetically sorted, not in
whatever order rows happen to be scanned).

---

## 6. List the Products Ordered in a Period

[leetcode.com/problems/list-the-products-ordered-in-a-period](https://leetcode.com/problems/list-the-products-ordered-in-a-period/)

```sql
WITH all_in_date AS (
    SELECT product_id, SUM(unit) AS unit
    FROM Orders
    WHERE EXTRACT(YEAR FROM order_date) = 2020 AND EXTRACT(MONTH FROM order_date) = 2
    GROUP BY product_id
    HAVING SUM(unit) >= 100
)

SELECT p.product_name, d.unit
FROM Products p
JOIN all_in_date d ON d.product_id = p.product_id;
```

OR directly:

```sql
SELECT p.product_name, SUM(o.unit) AS unit
FROM Products p
JOIN Orders o USING(product_id)
WHERE EXTRACT(YEAR FROM o.order_date) = 2020 AND EXTRACT(MONTH FROM o.order_date) = 2
GROUP BY p.product_name
HAVING SUM(o.unit) >= 100;
```

Both versions filter to February 2020 orders, sum units per product, and
keep only products totaling 100+ units — the CTE version computes that
total first and joins the product names on afterward; the direct version
joins first and does the `GROUP BY`/`HAVING` on the joined result. `USING
(product_id)` in the second version is shorthand for
`ON o.product_id = p.product_id` (from the Joins notes) — usable here
since both tables share the exact column name `product_id`. Both are
correct; the CTE version is arguably easier to read since it separates
"compute the qualifying totals" from "attach the product names" into two
distinct steps.

---

## 7. Find Users With Valid E-Mails

[leetcode.com/problems/find-users-with-valid-e-mails](https://leetcode.com/problems/find-users-with-valid-e-mails/)

```sql
SELECT *
FROM Users
WHERE mail ~ '^[A-Za-z][A-Za-z0-9_.-]*@leetcode\.com$';
```

A single anchored regex validates the whole email in one pass: `^`/`$`
anchor to the full string (so nothing extra is allowed before or after);
`[A-Za-z]` requires the very first character to be a letter (no email can
start with a digit or symbol here); `[A-Za-z0-9_.-]*` allows zero or more
letters/digits/underscore/dot/hyphen for the rest of the local part; and
`@leetcode\.com` requires the exact domain, with `\.` escaping the dot so
it means a literal period rather than regex's "any character" wildcard.
Without anchoring (`^`/`$`), this pattern would behave like `LIKE
'%...%'` and match the required shape *anywhere* inside a longer string —
same anchoring trap called out in the notes file's `~` section.