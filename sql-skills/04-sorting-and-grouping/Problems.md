# Sorting and Grouping — Problems

## 1. Number of Unique Subjects Taught by Each Teacher

[leetcode.com/problems/number-of-unique-subjects-taught-by-each-teacher](https://leetcode.com/problems/number-of-unique-subjects-taught-by-each-teacher/)

```sql
SELECT teacher_id, COUNT(DISTINCT subject_id) AS cnt
FROM Teacher
GROUP BY teacher_id;
```

`COUNT(DISTINCT subject_id)` per teacher, rather than plain `COUNT(*)`,
because a teacher can have multiple rows for the *same* subject (taught
across different classes) — those shouldn't be double-counted.

---

## 2. User Activity for the Past 30 Days I

[leetcode.com/problems/user-activity-for-the-past-30-days-i](https://leetcode.com/problems/user-activity-for-the-past-30-days-i/)

```sql
SELECT DISTINCT activity_date AS day, COUNT(DISTINCT user_id) AS active_users
FROM Activity
WHERE activity_date BETWEEN '2019-07-27'::DATE - INTERVAL '29 DAYS' AND '2019-07-27'
GROUP BY day;
```

**Why `29` and not `30`:** the window is "30 days ending `2019-07-27`
*inclusively*" — meaning `2019-07-27` itself is the 30th day, already part
of the count. Going back 30 full days from `2019-07-27` would land on
`2019-06-27`, giving 31 days total once `2019-07-27` is included. Going
back only 29 days lands on `2019-06-28`, and `2019-06-28` through
`2019-07-27` inclusive is exactly 30 days. The end date always counts as
one of the days, so the "steps back" is always one less than the window
size.

**Why `BETWEEN`:** shorthand for
`activity_date >= '...' AND activity_date <= '2019-07-27'` — same as
discussed in the Aggregate section, purely a readability choice here.

**Why `::DATE`:** `'2019-07-27'` on its own is just a string literal;
casting it to `::DATE` tells PostgreSQL to treat it as an actual date value
so that subtracting an `INTERVAL` from it is valid date arithmetic, rather
than a string operation that would error.

**Why `INTERVAL '29 DAYS'`:** PostgreSQL's way of expressing a span of
time to add to or subtract from a date/timestamp. `date - INTEGER` also
works in Postgres (subtracting a plain number of days), but
`INTERVAL '29 DAYS'` is more explicit about intent and is the form that
generalizes to other units (`'2 MONTHS'`, `'1 YEAR'`, etc.) if the problem
needed a different unit.

---

## 3. Product Sales Analysis III

[leetcode.com/problems/product-sales-analysis-iii](https://leetcode.com/problems/product-sales-analysis-iii/)

```sql
SELECT product_id, year AS first_year, quantity, price
FROM Sales a
WHERE year = (SELECT MIN(year) FROM Sales b WHERE a.product_id = b.product_id);
```

The "row with the MIN value per group" pattern from the notes file: the
correlated subquery recomputes each product's minimum year, and the outer
`WHERE` keeps only the row(s) matching that minimum — giving the full row
(including `quantity` and `price`), not just the year a plain
`GROUP BY ... MIN(year)` would return.

---

## 4. Classes With At Least 5 Students

[leetcode.com/problems/classes-with-at-least-5-students](https://leetcode.com/problems/classes-with-at-least-5-students/)

```sql
SELECT class
FROM Courses a
WHERE (SELECT COUNT(*) FROM Courses b WHERE b.class = a.class) >= 5
GROUP BY class;
```

**Why this is bad and slow:** the subquery `(SELECT COUNT(*) FROM Courses b
WHERE b.class = a.class)` is **correlated** — it re-runs once *for every
row* of the outer `Courses a`, each time rescanning (or re-probing) the
whole table to count matching rows. For a table with `N` rows, that's
roughly `N` separate `COUNT` operations, then a `GROUP BY` on top to
deduplicate the classes that passed. It also mixes filtering-by-aggregate
into `WHERE` via a workaround, rather than using the tool built for exactly
this.

The much better version:

```sql
SELECT class
FROM Courses
GROUP BY class
HAVING (COUNT(*) >= 5);
```

This groups once, computes `COUNT(*)` once per group during that single
pass, and `HAVING` filters those group results directly — no repeated
per-row subquery, no need for `GROUP BY` afterward to clean up duplicates
(there are none to begin with, since grouping already happened first).
This is exactly the `WHERE`-vs-`HAVING` guidance from the notes file:
"count of rows per group" is a job for `GROUP BY` + `HAVING`, not a
correlated subquery bolted onto `WHERE`.

---

## 5. Find Followers Count

[leetcode.com/problems/find-followers-count](https://leetcode.com/problems/find-followers-count/)

```sql
SELECT user_id, COUNT(follower_id) AS followers_count
FROM Followers
GROUP BY user_id
ORDER BY user_id ASC;
```

Plain `GROUP BY` + `COUNT` + `ORDER BY` — no `HAVING` needed since the
problem doesn't ask to filter which users appear, only to count and sort
them.

---

## 6. Biggest Single Number

[leetcode.com/problems/biggest-single-number](https://leetcode.com/problems/biggest-single-number/)

```sql
SELECT MAX(num) AS num
FROM (
  SELECT num FROM MyNumbers GROUP BY num HAVING (COUNT(*) = 1)
);
```

The inner query returns the numbers that appear exactly once. The outer `MAX(num)` then selects the largest of those numbers.

---

## 7. Customers Who Bought All Products

[leetcode.com/problems/customers-who-bought-all-products](https://leetcode.com/problems/customers-who-bought-all-products/)

```sql
SELECT customer_id
FROM Customer
GROUP BY customer_id
HAVING COUNT(DISTINCT product_key) = (SELECT COUNT(*) FROM Product);
```

Groups purchases by customer, counts how many *distinct* products each
customer bought, and `HAVING` keeps only customers whose distinct-product
count equals the total number of products that exist — i.e. customers who
bought every product at least once. `COUNT(DISTINCT ...)` matters here
since a customer could have bought the same product more than once; without
`DISTINCT`, repeat purchases would inflate the count past the true number
of unique products. The scalar subquery `(SELECT COUNT(*) FROM Product)`
computes the total product count once, the same pattern as the earlier
percentage-style problems in the Aggregate section.