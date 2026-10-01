# SQL Subqueries – Revision Notes

> **Datasets used**
> - `movies` (IMDb-style): `name, rating, genre, year, released, score, votes, director, writer, star, country, budget, gross, company, runtime`
> - `zomato` schema: `users, restaurants, food, menu, orders, order_details, delivery_partner`
>
> **Result legend**
> - 🟢 **Zomato results are exact** (computed from the provided CSVs).
> - 🟡 **Movies results are illustrative** (they show the *shape* of the output; exact values depend on your full 1000-row table).

---

## 0. Quick Theory

A **subquery** is a query nested inside another query (`SELECT`, `FROM`, `WHERE`, `HAVING`, `INSERT`, `UPDATE`, `DELETE`).

### Classification

| Type | Depends on outer query? | Returns | Typical operator |
|---|---|---|---|
| **Independent – Scalar** | No | 1 value (1 row × 1 col) | `=`, `>`, `<` |
| **Independent – Row (column) subquery** | No | 1 column, many rows | `IN`, `NOT IN`, `ANY`, `ALL` |
| **Independent – Table** | No | many cols, many rows | `(a, b) IN (...)`, `FROM (...)` |
| **Correlated** | **Yes** (references outer alias) | Re-evaluated per outer row | `=`, `>`, `<` |

### Rules of thumb
1. **Independent** subquery runs **once**, and its result is handed to the outer query.
2. **Correlated** subquery runs **once per outer row**, because it references a column from the outer query (e.g. `m1.genre`).
3. Scalar subquery must return **exactly one value**, otherwise you get an error.
4. `NOT IN` + `NULL` in the subquery result = **empty result** (a classic trap). Use `NOT EXISTS` or filter `IS NOT NULL`.
5. MySQL doesn't allow `LIMIT` directly inside an `IN (...)` subquery, so wrap it in a **CTE** or derived table.
6. MySQL doesn't allow `UPDATE`/`DELETE` on a table while selecting from the *same* table in a subquery (Error 1093) unless wrapped in a derived table.

---

## 1. Independent Subquery – Scalar Subquery

The inner query returns a **single value**, which the outer query compares against.

### 1.1 Movie with the highest profit

```sql
SELECT * FROM sql_cx_live.movies
WHERE (gross - budget) = (SELECT MAX(gross - budget) FROM sql_cx_live.movies);
```

**Logic:** Inner query finds max profit → outer query returns the movie(s) matching it.

🟡 **Sample output**

| name | year | score | budget | gross | profit* |
|---|---|---|---|---|---|
| Avatar | 2009 | 7.8 | 237000000 | 2847246203 | 2610246203 |

\*`profit` is shown for clarity; it isn't a column in the table.

---

### 1.2 Count of movies with rating above the average rating

```sql
SELECT COUNT(*) FROM sql_cx_live.movies
WHERE score > (SELECT AVG(score) FROM sql_cx_live.movies);
```

🟡 **Sample output**

| COUNT(*) |
|---|
| 507 |

---

### 1.3 Highest-rated movie among movies with votes above the average votes

> ⚠️ **Fix:** the original note had `WHERE (SELECT AVG(votes) FROM movies)`. That has no comparison, so it's always "truthy" and filters nothing. It also doesn't restrict the outer query. Corrected below.

```sql
SELECT * FROM movies
WHERE score = (
    SELECT MAX(score) FROM movies
    WHERE votes > (SELECT AVG(votes) FROM movies)
)
AND votes > (SELECT AVG(votes) FROM movies);
```

**Logic (nested scalars):**
1. Innermost: `AVG(votes)` → one number.
2. Middle: max score among movies above that vote average.
3. Outer: fetch the movie(s) with that score (and the same vote condition).

🟡 **Sample output**

| name | genre | year | score | votes |
|---|---|---|---|---|
| The Shawshank Redemption | Drama | 1994 | 9.3 | 2400000 |

---

### 1.4 Highest-rated movie of the year 2000

```sql
SELECT * FROM movies
WHERE year = 2000 AND score = (
    SELECT MAX(score) FROM movies
    WHERE year = 2000
);
```

🟡 **Sample output**

| name | genre | year | score | director |
|---|---|---|---|---|
| Gladiator | Action | 2000 | 8.5 | Ridley Scott |

---

## 2. Independent Subquery – Row Subquery (one column, many rows)

Inner query returns a **list of values** → use `IN` / `NOT IN`.

### 2.1 Users who never ordered

```sql
SELECT * FROM zomato.users
WHERE user_id NOT IN (SELECT DISTINCT(user_id) FROM zomato.orders);
```

> 💡 If `orders.user_id` could ever be `NULL`, `NOT IN` returns nothing. Safer alternative:
> ```sql
> SELECT u.* FROM zomato.users u
> WHERE NOT EXISTS (SELECT 1 FROM zomato.orders o WHERE o.user_id = u.user_id);
> ```

🟢 **Output (exact)**

| user_id | name | email | password |
|---|---|---|---|
| 6 | Anupama | anupama@gmail.com | 46rdw2 |
| 7 | Rishabh | rishabh@gmail.com | 4sw123 |

---

### 2.2 All movies made by the top 3 directors (by total gross)

A CTE is required because MySQL doesn't support `LIMIT` inside an `IN` subquery directly.

```sql
WITH top_dir AS (
    SELECT director FROM movies
    GROUP BY director
    ORDER BY SUM(gross) DESC
    LIMIT 3
)
SELECT * FROM movies
WHERE director IN (SELECT * FROM top_dir);
```

🟡 **Sample output**

| name | year | director | gross |
|---|---|---|---|
| Titanic | 1997 | James Cameron | 2201647264 |
| Avatar | 2009 | James Cameron | 2847246203 |
| Jurassic Park | 1993 | Steven Spielberg | 1029939903 |
| The Avengers | 2012 | Joss Whedon | 1518815515 |

---

### 2.3 All movies of actors whose filmography average rating > 8.5 (votes cutoff 25,000)

```sql
SELECT * FROM movies
WHERE star IN (
    SELECT star FROM movies
    WHERE votes > 25000
    GROUP BY star
    HAVING AVG(score) > 8.5
)
AND votes > 25000;
```

**Logic:** Inner query lists qualifying stars → outer query returns all their (well-voted) movies.

🟡 **Sample output**

| name | star | score | votes |
|---|---|---|---|
| Movie A | Actor X | 8.7 | 310000 |
| Movie B | Actor X | 8.6 | 120000 |
| Movie C | Actor Y | 8.9 | 540000 |

---

## 3. Independent Subquery – Table Subquery (many columns, many rows)

Compare **multiple columns at once** using a tuple: `(col1, col2) IN (SELECT col1, col2 ...)`.

### 3.1 Most profitable movie of each year

```sql
SELECT * FROM movies
WHERE (year, gross - budget) IN (
    SELECT year, MAX(gross - budget)
    FROM movies
    GROUP BY year
);
```

🟡 **Sample output**

| name | year | budget | gross |
|---|---|---|---|
| Star Wars: Episode V | 1980 | 18000000 | 538375067 |
| E.T. the Extra-Terrestrial | 1982 | 10500000 | 792910554 |
| Return of the Jedi | 1983 | 32500000 | 572700000 |

---

### 3.2 Highest-rated movie of each genre (votes cutoff 25,000)

```sql
SELECT * FROM movies
WHERE (genre, score) IN (
    SELECT genre, MAX(score)
    FROM movies
    WHERE votes > 25000
    GROUP BY genre
)
AND votes > 25000;
```

🟡 **Sample output**

| name | genre | score | votes |
|---|---|---|---|
| The Shawshank Redemption | Drama | 9.3 | 2400000 |
| The Dark Knight | Action | 9.0 | 2300000 |
| Spirited Away | Animation | 8.6 | 710000 |

---

### 3.3 Highest-grossing movies of the top 5 star/director combos (by total gross)

```sql
WITH top_start_dir AS (
    SELECT star, director FROM movies
    GROUP BY star, director
    ORDER BY SUM(gross) DESC
    LIMIT 5
)
SELECT name, star, director, gross FROM movies
WHERE (star, director) IN (SELECT * FROM top_start_dir);
```

> 💡 This returns **all** movies of those 5 combos. To get only the single highest-grossing movie per combo, you'd need an additional `MAX(gross)` condition or a window function.

🟡 **Sample output**

| name | star | director | gross |
|---|---|---|---|
| Avatar | Sam Worthington | James Cameron | 2847246203 |
| Titanic | Leonardo DiCaprio | James Cameron | 2201647264 |
| Iron Man 2 | Robert Downey Jr. | Jon Favreau | 623933331 |

---

## 4. Correlated Subquery

The inner query **references a column of the outer query**, so it runs **once per outer row**.

### 4.1 Movies rated higher than the average of their own genre

```sql
SELECT * FROM movies m1
WHERE score > (
    SELECT AVG(score) FROM movies m2
    WHERE m2.genre = m1.genre
);
```

**Logic:** For each movie `m1`, compute the average score of movies with the same genre, then compare.

🟡 **Sample output**

| name | genre | score |
|---|---|---|
| The Shining | Drama | 8.4 |
| Raging Bull | Biography | 8.2 |
| Alien | Horror | 8.4 |

---

### 4.2 Favorite food of each customer

```sql
WITH fav_food AS (
    SELECT t2.user_id, t4.f_name, COUNT(*) AS 'frequency'
    FROM zomato.users t1
    JOIN zomato.orders t2        ON t1.user_id = t2.user_id
    JOIN zomato.order_details t3 ON t2.order_id = t3.order_id
    JOIN zomato.food t4          ON t3.f_id = t4.f_id
    GROUP BY t2.user_id, t4.f_name
)
SELECT * FROM fav_food f1
WHERE frequency = (
    SELECT MAX(frequency) FROM fav_food f2
    WHERE f2.user_id = f1.user_id
);
```

**Logic:** The CTE counts how often each user ordered each food. The correlated subquery finds each user's max count, and the outer query keeps only the rows matching it. **Ties are kept** (user 4 below).

🟢 **Output (exact, first 5 of 6 rows)**

| user_id | f_name | frequency |
|---|---|---|
| 1 | Choco Lava cake | 5 |
| 2 | Choco Lava cake | 3 |
| 3 | Chicken Wings | 3 |
| 4 | Schezwan Noodles | 3 |
| 4 | Veg Manchurian | 3 |

*(6th row: user 5 → Choco Lava cake, 5)*

---

## 5. Subqueries in Different Clauses

### 5.1 Inside `SELECT` – % of total votes per movie

```sql
SELECT name,
       (votes / (SELECT SUM(votes) FROM movies)) * 100 AS 'votes_per'
FROM movies
ORDER BY votes_per DESC;
```

> 💡 In practice this is simpler with a window function: `votes / SUM(votes) OVER ()`. The subquery version is for practice.

🟡 **Sample output**

| name | votes_per |
|---|---|
| The Dark Knight | 0.4182 |
| Inception | 0.3711 |
| Fight Club | 0.3590 |

---

### 5.2 Inside `SELECT` (correlated) – movie score vs genre average

```sql
SELECT name, genre, score,
       (SELECT ROUND(AVG(score), 1)
        FROM movies m2
        WHERE m2.genre = m1.genre) AS 'genre_avg_rating'
FROM movies m1;
```

🟡 **Sample output**

| name | genre | score | genre_avg_rating |
|---|---|---|---|
| The Shining | Drama | 8.4 | 6.9 |
| Blue Lagoon | Adventure | 5.8 | 6.4 |
| Star Wars: Episode V | Action | 8.7 | 6.3 |

---

### 5.3 Inside `FROM` (derived table) – average rating of every restaurant

> 📝 This is a **derived table** (inline view), not correlated. It's an independent subquery used as a table.

```sql
SELECT r_name, avg_rating FROM (
    SELECT r_id, AVG(restaurant_rating) AS 'avg_rating'
    FROM zomato.orders
    GROUP BY r_id
) t1
JOIN zomato.restaurants t2
  ON t1.r_id = t2.r_id;
```

> `AVG()` ignores `NULL` ratings, which is why some orders without a rating don't drag the average down.

🟢 **Output (exact)**

| r_name | avg_rating |
|---|---|
| dominos | 1.6667 |
| kfc | 2.2000 |
| box8 | 4.6667 |
| Dosa Plaza | 3.6667 |
| China Town | 3.6667 |

---

### 5.4 Inside `HAVING` – genres with average score above the overall average

> 📝 Again an **independent scalar subquery**, placed in `HAVING` because it filters on an aggregate.

```sql
SELECT genre, AVG(score) AS 'avg_genre'
FROM movies
GROUP BY genre
HAVING avg_genre > (SELECT AVG(score) FROM movies);
```

🟡 **Sample output**

| genre | avg_genre |
|---|---|
| Biography | 7.1 |
| Animation | 6.9 |
| Drama | 6.8 |

---

## 6. Subqueries with DML (`INSERT`, `UPDATE`, `DELETE`)

### 6.1 `INSERT ... SELECT` – populate `loyal_users` (customers with more than 3 orders)

```sql
INSERT INTO loyal_users (user_id, name)
SELECT t1.user_id, t2.name
FROM zomato.orders t1
JOIN zomato.users t2 ON t1.user_id = t2.user_id
GROUP BY t1.user_id, t2.name
HAVING COUNT(*) > 3;
```

🟢 **Rows inserted (exact): 5 rows** (every customer who placed an order has exactly 5 orders)

| user_id | name |
|---|---|
| 1 | Nitish |
| 2 | Khushboo |
| 3 | Vartika |
| 4 | Ankit |
| 5 | Neha |

---

### 6.2 `UPDATE` with a correlated subquery – give 10% app money

```sql
UPDATE zomato.loyal_users
SET money = (
    SELECT SUM(amount) * 0.1
    FROM zomato.orders
    WHERE orders.user_id = loyal_users.user_id
);
```

**Logic:** For each row in `loyal_users`, the subquery sums that user's order amounts and takes 10%.

🟢 **`loyal_users` after update (exact)**

| user_id | name | money |
|---|---|---|
| 1 | Nitish | 166.5 |
| 2 | Khushboo | 267.0 |
| 3 | Vartika | 132.0 |
| 4 | Ankit | 180.0 |
| 5 | Neha | 303.5 |

---

### 6.3 `DELETE` – remove users who never ordered

> ⚠️ **Fix:** the original note selected from `zomato.users` inside a `DELETE FROM zomato.users`, which triggers MySQL **Error 1093** ("You can't specify target table for update in FROM clause"). The extra nesting was also unnecessary, and the statement was missing its semicolon.

```sql
DELETE FROM zomato.users
WHERE user_id NOT IN (SELECT DISTINCT user_id FROM zomato.orders);
```

🟢 **Rows deleted (exact): 2**

| user_id | name |
|---|---|
| 6 | Anupama |
| 7 | Rishabh |

---

## 7. Cheat Sheet

| Goal | Pattern |
|---|---|
| Compare to one computed value | `WHERE col > (SELECT AVG(col) FROM t)` |
| Match any value in a list | `WHERE col IN (SELECT col FROM ...)` |
| Exclude a list | `WHERE col NOT IN (...)` *(beware NULLs → prefer `NOT EXISTS`)* |
| Match multiple columns | `WHERE (a, b) IN (SELECT a, b ...)` |
| Compare to a group-level value per row | Correlated: `WHERE col > (SELECT AVG(col) FROM t t2 WHERE t2.g = t1.g)` |
| Add a computed column | Subquery in `SELECT` list |
| Treat result as a table | Derived table in `FROM (...) alias` |
| Filter groups against a global value | Subquery in `HAVING` |
| Bulk-load from a query | `INSERT INTO t (...) SELECT ...` |
| Row-wise update from another table | `UPDATE t SET col = (SELECT ... WHERE ...)` |
| Top-N inside `IN` (MySQL) | Wrap with a CTE / derived table |

### Common mistakes checklist
- [ ] Scalar subquery returns more than one row → error.
- [ ] Forgot the comparison operator before `(SELECT ...)`.
- [ ] `NOT IN` with a `NULL` in the list → empty result.
- [ ] Correlated subquery without table aliases → ambiguous column names.
- [ ] `UPDATE`/`DELETE` selecting from the same table → Error 1093.
- [ ] `LIMIT` directly inside `IN (...)` → unsupported in MySQL.

### Independent vs Correlated – one-liner
- **Independent:** inner query can be run on its own → executes once.
- **Correlated:** inner query can't run on its own (needs the outer alias) → executes per outer row.
