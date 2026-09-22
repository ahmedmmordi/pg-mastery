# Subqueries — Problems

## 1. Employees Whose Manager Left the Company

[leetcode.com/problems/employees-whose-manager-left-the-company](https://leetcode.com/problems/employees-whose-manager-left-the-company/)

```sql
SELECT employee_id
FROM Employees
WHERE manager_id NOT IN (
    SELECT DISTINCT employee_id FROM Employees
    )
AND salary < 30000
ORDER BY employee_id;
```

`NOT IN` here is safe from the usual `NULL` trap because the subquery's
list (`SELECT DISTINCT employee_id`) can never contain a `NULL` —
`employee_id` is the primary key, so it's always populated. The condition
reads as "this employee's manager isn't anyone's `employee_id`" — i.e. the
manager no longer exists in the table — combined with the salary filter.

---

## 2. Exchange Seats

[leetcode.com/problems/exchange-seats](https://leetcode.com/problems/exchange-seats/)

```sql
SELECT id,
    CASE
        WHEN id % 2 = 1 THEN COALESCE(next_student, student)
        ELSE prev_student
    END AS student
FROM (
    SELECT
        id,
        student,
        (SELECT student FROM Seat s2 WHERE s2.id = s1.id + 1) AS next_student,
        (SELECT student FROM Seat s2 WHERE s2.id = s1.id - 1) AS prev_student
    FROM Seat s1
) s
ORDER BY id;
```

OR:

```sql
SELECT id,
    CASE
        WHEN id % 2 = 1 THEN COALESCE(LEAD(student) OVER (ORDER BY id), student)
        ELSE LAG(student) OVER (ORDER BY id)
    END AS student
FROM Seat;
```

The derived-table version uses two correlated scalar subqueries per row
(`id + 1` and `id - 1`) to pull in the neighboring seat's student, then
`CASE WHEN` swaps odd/even ids with their neighbor. `COALESCE(next_student,
student)` handles the last row when `id` is odd with no row after it (an
odd id out with nobody to swap with keeps their own seat). The window
function version does the same swap with `LEAD`/`LAG` instead of two
self-joins-in-disguise — same idea as the "Consecutive Numbers" problem
from the Advanced Select notes, just applied to a swap instead of a
match check.

---

## 3. Movie Rating

[leetcode.com/problems/movie-rating](https://leetcode.com/problems/movie-rating/)

```sql
WITH user_counts AS (
    SELECT user_id, COUNT(rating) AS count
    FROM MovieRating
    GROUP BY user_id
),
movie_avgs AS (
    SELECT movie_id, AVG(rating) AS avg
    FROM MovieRating
    WHERE EXTRACT(YEAR FROM created_at) = 2020 AND EXTRACT(MONTH FROM created_at) = 2
    GROUP BY movie_id
)

(SELECT (SELECT name FROM Users WHERE Users.user_id = user_counts.user_id) AS results
FROM user_counts
ORDER BY count DESC, results ASC
LIMIT 1
)

UNION ALL

(SELECT (SELECT title FROM Movies WHERE Movies.movie_id = movie_avgs.movie_id) AS results
FROM movie_avgs
ORDER BY avg DESC, results ASC
LIMIT 1
);
```

Two unrelated "top 1" questions stacked into one result via `UNION ALL`:
one CTE counts ratings per user, the other averages ratings per movie
(restricted to February 2020 via `EXTRACT`). Each half of the `UNION ALL`
independently sorts its own CTE (`ORDER BY count/avg DESC`, then the name
ascending as a tiebreaker for "lexicographically smallest on a tie") and
takes just the top row with `LIMIT 1`. The subquery in each `SELECT`
(`SELECT name FROM Users WHERE ...`) swaps the id for its human-readable
name — a scalar subquery, since it's guaranteed to match exactly one row
(a primary key lookup).

---

## 4. Restaurant Growth

[leetcode.com/problems/restaurant-growth](https://leetcode.com/problems/restaurant-growth/)

```sql
WITH days_window AS (
    SELECT DISTINCT c.visited_on
    FROM Customer c
    WHERE (
        SELECT COUNT(DISTINCT c2.visited_on)
        FROM Customer c2
        WHERE c2.visited_on BETWEEN c.visited_on - 6 AND c.visited_on
    ) = 7
)

SELECT d.visited_on,
(
    SELECT SUM(c.amount) FROM Customer c WHERE visited_on BETWEEN d.visited_on - 6 AND d.visited_on
) AS amount,
(
    SELECT ROUND(SUM(c.amount)::NUMERIC / 7, 2) FROM Customer c WHERE visited_on BETWEEN d.visited_on - 6 AND d.visited_on
) AS average_amount
FROM days_window d
GROUP BY d.visited_on
ORDER BY d.visited_on ASC;
```

The CTE `days_window` first filters down to only dates that actually have a
full 7-day trailing window of distinct visit dates behind them (so the
first 6 days of the whole dataset are correctly excluded — there aren't
enough prior days yet to form a valid 7-day window). For each qualifying
date, two correlated subqueries recompute the trailing 7-day sum and
average independently. This is the same "recompute a window per row"
family as a running total, just bounded to a fixed 7-day range instead of
"from the start to here" — the kind of thing a window function
(`SUM(amount) OVER (ORDER BY visited_on RANGE BETWEEN '6 days' PRECEDING
AND CURRENT ROW)`) could also express, more concisely.

---

## 5. Friend Requests II: Who Has the Most Friends

[leetcode.com/problems/friend-requests-ii-who-has-the-most-friends](https://leetcode.com/problems/friend-requests-ii-who-has-the-most-friends/)

```sql
WITH all_ids AS (
    SELECT requester_id AS id, COUNT(*) AS count
    FROM RequestAccepted
    GROUP BY requester_id

    UNION ALL

    SELECT accepter_id AS id, COUNT(*) AS count
    FROM RequestAccepted
    GROUP BY accepter_id
)

SELECT id, SUM(count) AS num
FROM all_ids
GROUP BY id
ORDER BY num DESC
LIMIT 1;
```

Each accepted friend request contributes a "friend" to both people
involved — the requester and the accepter — so both directions need
counting. The CTE stacks two separate `GROUP BY` counts (one from each
side of the relationship) with `UNION ALL`, deliberately without
deduplicating, since a person's `id` can legitimately show up in *both*
halves (having both sent and received accepted requests) and both
contributions need to survive to be summed. The outer query then
`GROUP BY id, SUM(count)` merges each person's two partial counts into
their true total friend count.

---

## 6. Investments in 2016

[leetcode.com/problems/investments-in-2016](https://leetcode.com/problems/investments-in-2016/)

```sql
WITH unique_cities AS (
    SELECT lat, lon
    FROM Insurance
    GROUP BY lat, lon
    HAVING COUNT(*) = 1
),
valid_tiv_2015 AS (
    SELECT tiv_2015
    FROM Insurance
    GROUP BY tiv_2015
    HAVING COUNT(*) > 1
)

SELECT ROUND(SUM(tiv_2016)::NUMERIC, 2) AS tiv_2016
FROM Insurance
WHERE tiv_2015 IN (SELECT tiv_2015 FROM valid_tiv_2015)
AND (lat, lon) IN (SELECT lat, lon FROM unique_cities);
```

Two independent conditions, each computed as its own CTE: `unique_cities`
finds `(lat, lon)` pairs that belong to exactly one policyholder (
`HAVING COUNT(*) = 1`, meaning no one else shares that exact location), and
`valid_tiv_2015` finds `tiv_2015` values shared by more than one
policyholder (`HAVING COUNT(*) > 1`). The outer query then requires
**both** conditions at once via two separate `IN` checks. The
`(lat, lon) IN (SELECT lat, lon FROM ...)` is a row-value `IN`, matching
both columns as a pair — same technique as the row-value `IN` used in the
"Product Price at a Given Date" problem in the Advanced Select notes.

---

## 7. Department Top Three Salaries

[leetcode.com/problems/department-top-three-salaries](https://leetcode.com/problems/department-top-three-salaries/)

```sql
SELECT d.name AS Department, e.name AS Employee, e.salary AS Salary
FROM (
    SELECT
        name, salary, departmentId, DENSE_RANK() OVER (PARTITION BY departmentId ORDER BY salary DESC) AS salary_rank
    FROM Employee
) e
LEFT JOIN Department d
ON e.departmentId = d.id
WHERE e.salary_rank <= 3;
```

`DENSE_RANK() OVER (PARTITION BY departmentId ORDER BY salary DESC)` ranks
each employee's salary *within their own department*, restarting the rank
at 1 for every new department — exactly the `PARTITION BY` behavior from
the Advanced Select notes. `DENSE_RANK` (not `ROW_NUMBER`) is the correct
choice here specifically because "top three salaries" means the top three
*distinct* salary values, including everyone tied at those values — if two
employees are tied for 2nd, both should appear, which `DENSE_RANK` allows
and `ROW_NUMBER` would not. The derived table computes the rank first,
then the outer query filters to `salary_rank <= 3` and joins in each
department's name.