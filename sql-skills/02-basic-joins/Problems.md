# Joins — Problems

## 1. Replace Employee ID With The Unique Identifier

[leetcode.com/problems/replace-employee-id-with-the-unique-identifier](https://leetcode.com/problems/replace-employee-id-with-the-unique-identifier/)

```sql
SELECT en.unique_id, e.name
FROM Employees e
LEFT JOIN EmployeeUNI en ON en.id = e.id;
```

**Why `LEFT JOIN`:** every employee must appear in the result, even
one with no matching row in `EmployeeUNI`. An `INNER JOIN` would drop those
employees entirely; `LEFT JOIN` keeps them and gives `unique_id = NULL`
instead.

---

## 2. Product Sales Analysis I

[leetcode.com/problems/product-sales-analysis-i](https://leetcode.com/problems/product-sales-analysis-i/)

```sql
SELECT p.product_name, s.year, s.price
FROM Sales s
LEFT JOIN Product p ON s.product_id = p.product_id;
```

**Why `LEFT JOIN`:** driving from `Sales` keeps every sale row and
attaches the matching product name. Since every sale is guaranteed to
reference a valid product here, `INNER JOIN` would give the same result —
`LEFT JOIN` is just the safer default when you're not 100% sure every
foreign key has a match.

---

## 3. Customer Who Visited but Did Not Make Any Transactions

[leetcode.com/problems/customer-who-visited-but-did-not-make-any-transactions](https://leetcode.com/problems/customer-who-visited-but-did-not-make-any-transactions/)

```sql
SELECT v.customer_id, COUNT(*) AS count_no_trans
FROM Visits v
LEFT JOIN Transactions t ON v.visit_id = t.visit_id
WHERE t.transaction_id IS NULL
GROUP BY v.customer_id;
```

**Why `LEFT JOIN` + `WHERE ... IS NULL`:** this is the classic **anti-join**
pattern. `LEFT JOIN` keeps every visit and fills `NULL` for any visit with
no transaction. Filtering `WHERE t.transaction_id IS NULL` then keeps only
the visits that had *no* match — i.e. visits with no transaction — which is
exactly what "did not make any transactions" means.

---

## 4. Rising Temperature

[leetcode.com/problems/rising-temperature](https://leetcode.com/problems/rising-temperature/)

```sql
SELECT w.id
FROM Weather w
WHERE w.temperature > (
  SELECT wIn.temperature
  FROM Weather wIn
  WHERE wIn.recordDate = w.recordDate - 1
);
```

OR:

```sql
SELECT w2.id
FROM Weather w1, Weather w2
WHERE (w2.recordDate - w1.recordDate = 1)
  AND (w2.temperature > w1.temperature);
```

**Why a self-comparison:** the task needs each row compared to *yesterday's*
row in the same table — there's no second table involved, just the same
table referenced twice (once as "today," once as "yesterday"). The first
version does it with a correlated subquery (`w.recordDate - 1` computed per
outer row); the second does it as an explicit **self join** using a comma
join with a date-difference condition instead of `ON`. Both express the
same idea — "match each row to the one exactly one day before it."

---

## 5. Average Time of Process per Machine

[leetcode.com/problems/average-time-of-process-per-machine](https://leetcode.com/problems/average-time-of-process-per-machine/)

```sql
SELECT a.machine_id,
       ROUND(AVG(b.timestamp - a.timestamp)::numeric, 3) AS processing_time
FROM Activity a
JOIN Activity b
  ON (a.machine_id = b.machine_id
      AND a.process_id = b.process_id
      AND a.activity_type = 'start'
      AND b.activity_type = 'end')
GROUP BY a.machine_id;
```

**Why a self join:** each process has two rows in the same table — a
`'start'` row and an `'end'` row — and they need to be paired up to compute
a duration. Joining `Activity` to itself on `(machine_id, process_id)`,
while pinning one side to `'start'` and the other to `'end'`, pairs each
process's two rows into one. `::numeric` is a cast needed before `ROUND()`,
since subtracting two `timestamp`/numeric values in Postgres can return a
type `ROUND()` doesn't accept directly.

---

## 6. Employee Bonus

[leetcode.com/problems/employee-bonus](https://leetcode.com/problems/employee-bonus/)

```sql
SELECT e.name, b.bonus
FROM Employee e
LEFT JOIN Bonus b ON e.empId = b.empId
WHERE (b.bonus IS NULL OR b.bonus < 1000);
```

**Why `LEFT JOIN` + `IS NULL OR`:** employees with no bonus row at all must
be included (that's the `LEFT JOIN`), and among employees that do have a
bonus, only those under 1000 should show. Since `bonus < 1000` alone would
silently drop the `NULL` rows (comparing `NULL` to a number returns `NULL`,
not `TRUE`), the `IS NULL OR` half is required to keep them — same NULL-safe
pattern used in the `SELECT` section's referee problem.

---

## 7. Students and Examinations

[leetcode.com/problems/students-and-examinations](https://leetcode.com/problems/students-and-examinations/)

```sql
SELECT s.student_id, s.student_name, sub.subject_name,
       COUNT(ex.subject_name) AS attended_exams
FROM Students s
CROSS JOIN Subjects sub
LEFT JOIN Examinations ex
  ON sub.subject_name = ex.subject_name AND s.student_id = ex.student_id
GROUP BY s.student_id, s.student_name, sub.subject_name
ORDER BY s.student_id, sub.subject_name;
```

**Why `CROSS JOIN` then `LEFT JOIN`:** the result needs a row for *every*
(student, subject) combination — including subjects a student never took an
exam for. `CROSS JOIN` generates that full combination first (every student
paired with every subject), then `LEFT JOIN` attaches matching exam
records where they exist, leaving `NULL` where they don't.
`COUNT(ex.subject_name)` counts only the non-`NULL` matches per group, so a
student-subject pair with no exam correctly comes out as `0`.

---

## 8. Managers with at Least 5 Direct Reports

[leetcode.com/problems/managers-with-at-least-5-direct-reports](https://leetcode.com/problems/managers-with-at-least-5-direct-reports/)

```sql
SELECT e1.name
FROM Employee e1
JOIN Employee e2 ON e1.id = e2.managerId
GROUP BY e1.id, e1.name
HAVING COUNT(e2.id) >= 5;
```

**Why a self join + `HAVING`:** each manager and their direct reports are
rows in the same table, linked by `managerId` pointing back to another row's
`id` — a self join is what connects a manager row (`e1`) to each of their
report rows (`e2`). `GROUP BY` collapses the joined rows to one per manager,
and `HAVING COUNT(e2.id) >= 5` filters those *groups* down to managers with
5+ reports — `HAVING` is required here (not `WHERE`) because the filter is
on an aggregate, and `WHERE` runs before aggregation.

---

## 9. Confirmation Rate

[leetcode.com/problems/confirmation-rate](https://leetcode.com/problems/confirmation-rate/)

```sql
SELECT s.user_id,
    ROUND(COALESCE(AVG((action = 'confirmed')::int), 0), 2) AS confirmation_rate
FROM Signups s
LEFT JOIN Confirmations c ON s.user_id = c.user_id
GROUP BY s.user_id;
```

**Why `LEFT JOIN` + `COALESCE`:** every signed-up user must appear, even one
with zero confirmation requests — that's the `LEFT JOIN`. `(action =
'confirmed')::int` turns each row into `1` (confirmed) or `0` (not
confirmed), so `AVG(...)` directly gives the confirmation rate for that
user. For users with no rows at all, `AVG()` over nothing returns `NULL`,
so `COALESCE(..., 0)` substitutes `0` instead of leaving the rate blank.