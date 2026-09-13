# SELECT — Problems

## 1. Recyclable and Low Fat Products

[leetcode.com/problems/recyclable-and-low-fat-products](https://leetcode.com/problems/recyclable-and-low-fat-products/)

```sql
SELECT product_id
FROM Products
WHERE low_fats = 'Y' AND recyclable = 'Y';
```

---

## 2. Find Customer Referee

[leetcode.com/problems/find-customer-referee](https://leetcode.com/problems/find-customer-referee/)

```sql
SELECT name
FROM Customer
WHERE referee_id IS NULL OR referee_id != 2;
```

---

## 3. Big Countries

[leetcode.com/problems/big-countries](https://leetcode.com/problems/big-countries/)

```sql
SELECT name, population, area
FROM World
WHERE area >= 3000000 OR population >= 25000000;
```

---

## 4. Article Views I

[leetcode.com/problems/article-views-i](https://leetcode.com/problems/article-views-i/)

```sql
SELECT DISTINCT author_id AS id
FROM Views
WHERE author_id = viewer_id
ORDER BY id;
```

---

## 5. Invalid Tweets

[leetcode.com/problems/invalid-tweets](https://leetcode.com/problems/invalid-tweets/)

```sql
SELECT tweet_id
FROM Tweets
WHERE LENGTH(content) > 15;
```