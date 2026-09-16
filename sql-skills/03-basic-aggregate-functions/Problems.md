# Aggregate Functions — Problems

## 1. Not Boring Movies

[leetcode.com/problems/not-boring-movies](https://leetcode.com/problems/not-boring-movies/)

```sql
SELECT *
FROM Cinema
WHERE (id % 2 = 1 AND description != 'boring')
ORDER BY rating DESC;
```

Straightforward row filter, no aggregation needed — `id % 2 = 1` picks odd
ids, combined with excluding `'boring'` descriptions, then sorted.

---

## 2. Average Selling Price

[leetcode.com/problems/average-selling-price](https://leetcode.com/problems/average-selling-price/)

```sql
SELECT p.product_id,
       COALESCE(ROUND(SUM(p.price * u.units)::NUMERIC / SUM(u.units), 2), 0) AS average_price
FROM Prices p
LEFT JOIN UnitsSold u
  ON p.product_id = u.product_id AND u.purchase_date BETWEEN p.start_date AND p.end_date
GROUP BY p.product_id;
```

**Why `::NUMERIC`:** `SUM(p.price * u.units)` and `SUM(u.units)` can end up
as types PostgreSQL's `/` operator handles as integer or imprecise
floating-point division depending on the input types. Casting the numerator
to `::NUMERIC` forces the division to use exact decimal arithmetic instead
of truncating or losing precision — this also satisfies `ROUND(x, 2)`,
which needs a `numeric` argument for the two-argument form.

**Why `COALESCE`:** a product can exist in `Prices` with zero matching sales
(no rows in `UnitsSold` during its price window). Because of the
`LEFT JOIN`, that product still appears, but `SUM(u.units)` over zero
matching rows is `NULL`, so the whole division becomes `NULL`. `COALESCE`
substitutes `0` so the average price shows as `0` instead of blank.

**Why `BETWEEN` instead of `<=`/`>=`:** they're equivalent —
`u.purchase_date BETWEEN p.start_date AND p.end_date` is just shorthand for
`u.purchase_date >= p.start_date AND u.purchase_date <= p.end_date`.
`BETWEEN` is chosen here purely for readability; there's no performance or
correctness difference between the two forms in Postgres.

**Why the date range condition is in `ON`, not `WHERE`:** this is a
`LEFT JOIN`, and `ON` vs `WHERE` behaves differently for outer joins.
Putting `u.purchase_date BETWEEN ...` in `ON` means it's part of *what
counts as a match* — a `Prices` row with no sales in its date range still
appears (with `u.*` as `NULL`), because the `LEFT JOIN` still keeps the
left-side row even if this extra condition fails to match anything. If the
same condition were moved to `WHERE` instead, it would run *after* the
join, filtering out any row where `u.purchase_date` is `NULL` (since
`NULL BETWEEN ...` is `NULL`, not `TRUE`) — which would silently turn this
back into something behaving like an `INNER JOIN`, dropping products with
no sales entirely. That's the general rule: extra conditions on the
*nullable* (right) side of a `LEFT JOIN` belong in `ON` if you want
unmatched left rows to still show up.

---

## 3. Project Employees I

[leetcode.com/problems/project-employees-i](https://leetcode.com/problems/project-employees-i/)

```sql
SELECT p.project_id, COALESCE(ROUND(AVG(e.experience_years), 2), 0) AS average_years
FROM Project p
LEFT JOIN Employee e ON p.employee_id = e.employee_id
GROUP BY p.project_id;
```

Same `LEFT JOIN` + `COALESCE` pattern as #2, at a smaller scale: keep every
project even one with no matched employee row, and default a `NULL` average
to `0`.

---

## 4. Percentage of Users Attended a Contest

[leetcode.com/problems/percentage-of-users-attended-a-contest](https://leetcode.com/problems/percentage-of-users-attended-a-contest/)

```sql
SELECT r.contest_id,
       ROUND((COUNT(*)::NUMERIC * 100 / (SELECT COUNT(*) FROM Users)), 2) AS percentage
FROM Register r
GROUP BY r.contest_id
ORDER BY percentage DESC, r.contest_id ASC;
```

**Why no `COALESCE`:** `COALESCE` is needed when an aggregate can return
`NULL` because a group matched zero rows. Here, every group comes from
`GROUP BY r.contest_id` on `Register` itself — a contest only appears in
the result if it has at least one row in `Register`, so `COUNT(*)` for that
group is never `0` or `NULL` by construction. There's no missing-row case
to guard against.

**Why no join:** the denominator (`total users`) doesn't need to be
attached row-by-row to `Register` — it's the same single number for every
group. A scalar subquery `(SELECT COUNT(*) FROM Users)` computes it once
and reuses it in every row's calculation, which is simpler and avoids
inflating `Register`'s row count the way a `JOIN Users` would (a join would
multiply rows and break the `COUNT(*)` in the numerator).

---

## 5. Queries Quality and Percentage

[leetcode.com/problems/queries-quality-and-percentage](https://leetcode.com/problems/queries-quality-and-percentage/)

```sql
SELECT query_name,
       ROUND(AVG(rating::NUMERIC / position), 2) AS quality,
       ROUND(COUNT(*) FILTER (WHERE rating < 3)::NUMERIC * 100 / COUNT(*), 2) AS poor_query_percentage
FROM Queries
GROUP BY query_name;
```

**Why `FILTER` and not `WHERE`:** `WHERE rating < 3` would remove every row
that doesn't have a poor rating *before* grouping — which would corrupt
both the `quality` column (computed from *all* ratings, not just poor ones)
and the denominator of `poor_query_percentage` (`COUNT(*)` would then only
count poor rows, making the percentage always 100%). `FILTER` instead scopes
the condition to *just that one aggregate call* — `COUNT(*) FILTER (WHERE
rating < 3)` counts only poor-rated rows, while every other aggregate in
the same `SELECT` (like the plain `COUNT(*)` in the denominator, and
`AVG(...)` for `quality`) still sees the full, unfiltered group.

---

## 6. Monthly Transactions I

[leetcode.com/problems/monthly-transactions-i](https://leetcode.com/problems/monthly-transactions-i/)

```sql
SELECT TO_CHAR(trans_date, 'YYYY-MM') AS month, country,
       COUNT(*) AS trans_count,
       COUNT(CASE WHEN state = 'approved' THEN 1 END) AS approved_count,
       SUM(amount) trans_total_amount,
       SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END) AS approved_total_amount
FROM Transactions
GROUP BY month, country;
```

Same conditional-aggregation idea as `FILTER` above, written with the more
portable `CASE WHEN` form: `COUNT(CASE WHEN ... THEN 1 END)` only counts
matching rows because the implicit `ELSE NULL` makes non-matching rows
invisible to `COUNT`; `SUM(CASE WHEN ... THEN amount ELSE 0 END)` sums only
matching rows because non-matching ones contribute `0` instead of `NULL`.

---

## 7. Immediate Food Delivery II

[leetcode.com/problems/immediate-food-delivery-ii](https://leetcode.com/problems/immediate-food-delivery-ii/)

```sql
SELECT ROUND(COUNT(CASE WHEN order_date = customer_pref_delivery_date THEN 1 END)::NUMERIC * 100 / COUNT(*), 2) AS immediate_percentage
FROM Delivery d
INNER JOIN (
  SELECT customer_id, MIN(order_date) AS first_order_date
  FROM Delivery
  GROUP BY customer_id
) first_orders
  ON d.customer_id = first_orders.customer_id AND d.order_date = first_orders.first_order_date;
```

OR:

```sql
SELECT ROUND(COUNT(*) FILTER(WHERE order_date = customer_pref_delivery_date)::NUMERIC * 100 / COUNT(*), 2) AS immediate_percentage
FROM Delivery d
INNER JOIN (
  SELECT customer_id, MIN(order_date) AS first_order_date
  FROM Delivery
  GROUP BY customer_id
) first_orders
  ON d.customer_id = first_orders.customer_id AND d.order_date = first_orders.first_order_date;
```

**Why another table (a subquery here, specifically a *derived table*):**
the problem needs "each customer's *first* order," but that's not a fact
that exists on any single row of `Delivery` — it has to be computed by
aggregating all of a customer's rows together (`MIN(order_date)`) first. A
subquery inside `FROM`/`JOIN` (wrapped in parens and aliased —
`first_orders` here) runs first, produces a small temporary result table
of `(customer_id, first_order_date)` pairs, and the outer query then joins
against *that* like it would against any real table. This is different
from a scalar subquery (like `(SELECT COUNT(*) FROM Users)` in problem #4),
which returns a single value, not a table — you can't join to a single
value, only reference it.

**Why this join type (`INNER JOIN`):** the goal is to keep only the
`Delivery` rows that *are* a customer's first order — a plain match/filter,
not "keep everything from one side." `INNER JOIN` on
`(customer_id, order_date) = (customer_id, first_order_date)` does exactly
that: it narrows `Delivery` down to one row per customer (their earliest
order), discarding the rest. No `LEFT JOIN` is needed here because there's
nothing to preserve beyond the match — every customer has exactly one first
order, so every customer survives the `INNER JOIN` once.

---

## 8. Game Play Analysis IV

[leetcode.com/problems/game-play-analysis-iv](https://leetcode.com/problems/game-play-analysis-iv/)

```sql
SELECT ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(DISTINCT(player_id)) FROM Activity), 2) AS fraction
FROM Activity a
INNER JOIN (
  SELECT player_id, MIN(event_date) AS first_login
  FROM Activity
  GROUP BY player_id
) first_activities
  ON a.player_id = first_activities.player_id AND a.event_date = first_activities.first_login + 1;
```

**Why another table / subquery:** same reasoning as #7 — "each player's
first login date" is an aggregate fact (`MIN(event_date)` per player), not
something readable off a single row. The derived table
`first_activities` computes that once per player, then the outer query
joins `Activity` against it to find, for each player, whether a row exists
exactly one day after their first login (`a.event_date =
first_activities.first_login + 1`).

**Why this join type (`INNER JOIN`):** the numerator only needs to count
players who *did* log in again the next day — rows that don't satisfy the
date condition simply shouldn't be counted at all, which is exactly what
`INNER JOIN` does: only rows where both the `player_id` match *and* the
`+1 day` condition match survive. `COUNT(*)` on the joined result then
directly gives "number of players who returned the day after their first
login," which is the numerator; the total player count for the denominator
comes from a separate scalar subquery, same idea as problem #4.