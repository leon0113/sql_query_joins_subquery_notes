# SQL Interview Revision Notes – `phone.smartphones` & `phone.ipl`

> Every practice query from your sessions, grouped by topic. Cover the code and test yourself.
>
> **Result legend:** 🟡 All sample outputs below are **illustrative**. They show the *shape* of each result, and exact values depend on your real data.

---

## 0. Dataset Schemas

### `phone.smartphones`

| Column | Type | Column | Type |
|---|---|---|---|
| brand_name | text | num_rear_cameras | int |
| model | text | num_front_cameras | double |
| price | int | os | text |
| rating | double | primary_camera_rear | double |
| has_5g | text | primary_camera_front | double |
| has_nfc | text | extended_memory_available | int |
| has_ir_blaster | text | extended_upto | text |
| processor_brand | text | resolution_width | int |
| num_cores | double | resolution_height | int |
| processor_speed | double | battery_capacity | double |
| fast_charging_available | int | ram_capacity | double |
| fast_charging | text | internal_memory | double |
| screen_size | double | refresh_rate | int |

> ⚠️ `has_5g`, `has_nfc`, `has_ir_blaster` are **text** (`'True'`/`'False'`). `fast_charging_available` and `extended_memory_available` are **int** (`1`/`0`). Compare each with the right type.

### `phone.ipl` (ball-by-ball)

| Column | Type | Column | Type |
|---|---|---|---|
| ID | int | batsman_run | int |
| innings | int | extras_run | int |
| overs | int | total_run | int |
| ballnumber | int | non_boundary | int |
| batter | text | isWicketDelivery | int |
| bowler | text | player_out | text |
| non-striker | text | kind | text |
| extra_type | text | fields_involved | text |
| BattingTeam | text | | |

---

## 1. Order of Query Execution

SQL does **not** run in the order you write it.

| Step | Clause | What it does |
|---|---|---|
| 1 | `FROM` | Picks the table |
| 2 | `JOIN` | Combines tables |
| 3 | `WHERE` | Filters rows |
| 4 | `GROUP BY` | Groups the remaining rows |
| 5 | `HAVING` | Filters groups |
| 6 | `SELECT` | Chooses and computes columns |
| 7 | `DISTINCT` | Removes duplicate rows |
| 8 | `ORDER BY` | Sorts the result |

> 💡 An alias works in `ORDER BY` (runs after `SELECT`) but **not** in `WHERE` (runs before). MySQL also lets you use aliases in `GROUP BY`/`HAVING` as an extension. Standard SQL doesn't.
>
> 💡 `WHERE` filters **rows before** grouping. `HAVING` filters **groups after** `GROUP BY`.

---

## 2. SELECT Basics

### 2.1 Mathematical expression – PPI of all phones

`PPI = √(resolution_width² + resolution_height²) / screen_size`

```sql
SELECT model,
       SQRT(resolution_width * resolution_width
          + resolution_height * resolution_height) / screen_size AS "ppi"
FROM phone.smartphones;
```

🟡 **Sample output**

| model | ppi |
|---|---|
| OnePlus 11 5G | 525.9 |
| Samsung Galaxy S23 Ultra | 500.4 |
| Xiaomi Redmi Note 12 | 395.0 |

### 2.2 DISTINCT – all unique brands

```sql
SELECT DISTINCT(brand_name) AS 'All Brands'
FROM phone.smartphones;
```

> 💡 `DISTINCT` is **not** a function. The parentheses are cosmetic, and it applies to the whole row.

🟡 **Sample output**

| All Brands |
|---|
| apple |
| samsung |
| oneplus |
| xiaomi |

### 2.3 DISTINCT on combinations – unique brand + processor pairs

```sql
SELECT DISTINCT brand_name, processor_brand AS 'All Brands'
FROM phone.smartphones;
```

> ⚠️ The alias `'All Brands'` is attached **only** to `processor_brand`, which is misleading. Better: `SELECT DISTINCT brand_name, processor_brand FROM ...`

🟡 **Sample output**

| brand_name | All Brands |
|---|---|
| apple | bionic |
| samsung | snapdragon |
| samsung | exynos |
| xiaomi | dimensity |

---

## 3. Filtering Rows with WHERE

### 3.1 All Samsung phones

```sql
SELECT * FROM phone.smartphones
WHERE brand_name = 'samsung';
```

🟡 **Sample output** *(first columns only)*

| brand_name | model | price | rating |
|---|---|---|---|
| samsung | Samsung Galaxy S23 Ultra | 124999 | 89 |
| samsung | Samsung Galaxy A54 | 38999 | 82 |
| samsung | Samsung Galaxy M14 | 13490 | 74 |

### 3.2 Price > 50000

```sql
SELECT * FROM phone.smartphones
WHERE price > 50000;
```

🟡 **Sample output**

| brand_name | model | price |
|---|---|---|
| apple | Apple iPhone 14 Pro Max | 129900 |
| samsung | Samsung Galaxy S23 Ultra | 124999 |
| oneplus | OnePlus 11 5G | 56999 |

### 3.3 BETWEEN – price 10000 to 20000

```sql
SELECT * FROM phone.smartphones
WHERE price BETWEEN 10000 AND 20000;
```

> 💡 `BETWEEN` is **inclusive** on both ends.

🟡 **Sample output**

| brand_name | model | price |
|---|---|---|
| samsung | Samsung Galaxy M14 | 13490 |
| xiaomi | Xiaomi Redmi Note 12 | 16999 |
| realme | Realme Narzo 60 | 17999 |

### 3.4 AND – price < 25000 and rating > 80

```sql
SELECT * FROM phone.smartphones
WHERE price < 25000 AND rating > 80;
```

🟡 **Sample output**

| brand_name | model | price | rating |
|---|---|---|---|
| xiaomi | Xiaomi Redmi Note 12 Pro | 23999 | 83 |
| realme | Realme 11 Pro | 22999 | 81 |

### 3.5 Samsung phones with RAM > 8 GB

```sql
SELECT * FROM phone.smartphones
WHERE brand_name = 'samsung' AND ram_capacity > 8;
```

🟡 **Sample output**

| brand_name | model | ram_capacity |
|---|---|---|
| samsung | Samsung Galaxy S23 Ultra | 12 |
| samsung | Samsung Galaxy Z Fold 4 | 12 |

### 3.6 Samsung phones with a Snapdragon processor

```sql
SELECT * FROM phone.smartphones
WHERE brand_name = 'samsung' AND processor_brand = 'snapdragon';
```

🟡 **Sample output**

| brand_name | model | processor_brand |
|---|---|---|
| samsung | Samsung Galaxy S23 Ultra | snapdragon |
| samsung | Samsung Galaxy A34 | snapdragon |

### 3.7 Brands that have phones with price > 50000

```sql
SELECT DISTINCT(brand_name) FROM phone.smartphones
WHERE price > 50000;
```

🟡 **Sample output**

| brand_name |
|---|
| apple |
| samsung |
| oneplus |

### 3.8 IN – Snapdragon or Bionic processor

```sql
SELECT * FROM phone.smartphones
WHERE processor_brand IN ("snapdragon", "bionic");
```

🟡 **Sample output**

| model | processor_brand |
|---|---|
| Apple iPhone 14 | bionic |
| OnePlus 11 5G | snapdragon |
| Samsung Galaxy A34 | snapdragon |

### 3.9 NOT IN – without Snapdragon or Bionic

```sql
SELECT * FROM phone.smartphones
WHERE processor_brand NOT IN ("snapdragon", "bionic");
```

> ⚠️ Rows where `processor_brand` is `NULL` are **excluded** by `NOT IN`, because a `NULL` comparison is neither true nor false.

🟡 **Sample output**

| model | processor_brand |
|---|---|
| Xiaomi Redmi Note 12 | dimensity |
| Samsung Galaxy S22 | exynos |
| Google Pixel 7 | tensor |

---

## 4. Modifying Data – UPDATE & DELETE

> ⚠️ **Always** add a `WHERE` clause to `UPDATE` and `DELETE`. Without it, every row is affected.

### 4.1 UPDATE – rename processor brand "mediatek" → "dimensity"

```sql
UPDATE phone.smartphones
SET processor_brand = 'dimensity'
WHERE processor_brand = 'mediatek';
```

🟡 **Sample output**

```
Query OK, 38 rows affected  (Rows matched: 38  Changed: 38  Warnings: 0)
```

### 4.2 UPDATE multiple columns at once

```sql
UPDATE phone.users
SET email = 'leon@yahoo.com', password = '89595'
WHERE name = 'Leon';
```

🟡 **Sample output**

```
Query OK, 1 row affected  (Rows matched: 1  Changed: 1  Warnings: 0)
```

### 4.3 DELETE – phones with price > 200,000

```sql
DELETE FROM phone.smartphones
WHERE price > 200000;
```

🟡 **Sample output**

```
Query OK, 3 rows affected
```

---

## 5. Aggregate & Scalar Functions

> 💡 Aggregates collapse many rows into **one value** and ignore `NULL`s (except `COUNT(*)`).

### 5.1 Minimum price

```sql
SELECT MIN(price) FROM phone.smartphones;
```

🟡 **Sample output**

| MIN(price) |
|---|
| 3499 |

### 5.2 Average internal memory (refresh rate ≥ 120, front camera ≥ 20 MP)

```sql
SELECT AVG(internal_memory) FROM phone.smartphones
WHERE refresh_rate >= 120 AND primary_camera_front >= 20;
```

🟡 **Sample output**

| AVG(internal_memory) |
|---|
| 180.4 |

### 5.3 Total price of all Apple phones

```sql
SELECT SUM(price) FROM phone.smartphones
WHERE brand_name = 'apple';
```

🟡 **Sample output**

| SUM(price) |
|---|
| 2145700 |

### 5.4 Number of Apple phones

```sql
SELECT COUNT(*) FROM phone.smartphones
WHERE brand_name = 'apple';
```

🟡 **Sample output**

| COUNT(*) |
|---|
| 21 |

### 5.5 Number of unique brands

```sql
SELECT COUNT(DISTINCT(brand_name)) FROM phone.smartphones;
```

🟡 **Sample output**

| COUNT(DISTINCT(brand_name)) |
|---|
| 46 |

### 5.6 Standard deviation of screen size

```sql
SELECT STD(screen_size) FROM phone.smartphones;
```

🟡 **Sample output**

| STD(screen_size) |
|---|
| 0.3472916 |

### 5.7 Scalar function `ROUND` – std. deviation to 2 decimals

```sql
SELECT ROUND(STD(screen_size), 2) FROM phone.smartphones;
```

🟡 **Sample output**

| ROUND(STD(screen_size), 2) |
|---|
| 0.35 |

---

## 6. Sorting & Limiting (`ORDER BY`, `LIMIT`)

### 6.1 Top 5 Samsung phones with the biggest screen

```sql
SELECT model, price, screen_size
FROM phone.smartphones
WHERE brand_name = 'samsung'
ORDER BY screen_size DESC
LIMIT 5;
```

🟡 **Sample output**

| model | price | screen_size |
|---|---|---|
| Samsung Galaxy S23 Ultra | 124999 | 6.8 |
| Samsung Galaxy A73 | 41999 | 6.7 |
| Samsung Galaxy M54 | 27999 | 6.7 |

### 6.2 Sort by total cameras (front + rear), descending

```sql
SELECT brand_name, model,
       num_rear_cameras + num_front_cameras AS 'Total_cameras'
FROM phone.smartphones
ORDER BY Total_cameras DESC;
```

🟡 **Sample output**

| brand_name | model | Total_cameras |
|---|---|---|
| vivo | Vivo X90 Pro | 6 |
| oppo | Oppo Find X5 | 5 |
| samsung | Samsung Galaxy S23 Ultra | 5 |

### 6.3 Sort by PPI, descending

```sql
SELECT model,
       ROUND(SQRT(resolution_width * resolution_width
                + resolution_height * resolution_height) / screen_size, 2) AS "ppi"
FROM phone.smartphones
ORDER BY ppi DESC;
```

🟡 **Sample output**

| model | ppi |
|---|---|
| Sony Xperia 1 IV | 643.28 |
| OnePlus 11 5G | 525.93 |
| Samsung Galaxy S23 Ultra | 500.40 |

### 6.4 Phone with the 2nd largest battery

```sql
SELECT brand_name, model, battery_capacity
FROM phone.smartphones
ORDER BY battery_capacity DESC
LIMIT 1, 1;
```

> 💡 `LIMIT x, y` = **skip x rows, return y rows**. `LIMIT 10, 2` skips 10 and returns the 11th and 12th.
> ⚠️ With duplicate values, "2nd row" ≠ "2nd largest value". See the tie-breaker below.

🟡 **Sample output**

| brand_name | model | battery_capacity |
|---|---|---|
| oukitel | Oukitel WP21 | 9800 |

### 6.5 4th largest battery with a tie-breaker

```sql
SELECT brand_name, model, battery_capacity
FROM phone.smartphones
ORDER BY battery_capacity DESC, model ASC
LIMIT 3, 1;
```

> 💡 **Tie-breaker explained**
> - **Problem:** when many rows share a value (e.g., several phones with 7000 mAh), SQL doesn't guarantee which comes first.
> - **Solution:** `, model ASC` sorts tied rows by model name (A → Z).
> - **Benefit:** the query returns the **same row every time**.

🟡 **Sample output**

| brand_name | model | battery_capacity |
|---|---|---|
| doogee | Doogee S98 | 6000 |

### 6.6 Worst-rated Apple phone

```sql
SELECT brand_name, model, rating
FROM phone.smartphones
WHERE brand_name = 'apple'
ORDER BY rating ASC
LIMIT 1;
```

> 💡 In MySQL, `NULL`s sort **first** in `ASC` order. Add `AND rating IS NOT NULL` if ratings can be missing.

🟡 **Sample output**

| brand_name | model | rating |
|---|---|---|
| apple | Apple iPhone SE 2020 | 65 |

### 6.7 Sort by brand (A → Z), then price (low → high)

```sql
SELECT brand_name, model, price
FROM phone.smartphones
ORDER BY brand_name ASC, price ASC;
```

> 💡 The first column is sorted first, and the next column only breaks ties.

🟡 **Sample output**

| brand_name | model | price |
|---|---|---|
| apple | Apple iPhone SE 2020 | 29999 |
| apple | Apple iPhone 13 | 61999 |
| asus | Asus ROG Phone 6 | 71999 |

### 6.8 Costliest phone

```sql
SELECT brand_name, model, price
FROM phone.smartphones
ORDER BY price DESC
LIMIT 1;
```

🟡 **Sample output**

| brand_name | model | price |
|---|---|---|
| samsung | Samsung Galaxy Z Fold 4 | 154999 |

---

## 7. GROUP BY

> 💡 Every non-aggregated column in `SELECT` must appear in `GROUP BY` (MySQL's `ONLY_FULL_GROUP_BY` mode enforces this).

### 7.1 Per brand: count, avg price, rating, screen size, battery

```sql
SELECT brand_name,
       COUNT(model)                    AS "No of Model",
       ROUND(AVG(price), 2)            AS "AVG Price",
       ROUND(AVG(rating), 2)           AS "AVG Rating",
       ROUND(AVG(screen_size), 2)      AS "AVG Screen Size",
       ROUND(AVG(battery_capacity), 2) AS "AVG Battery capacity"
FROM phone.smartphones
GROUP BY brand_name
ORDER BY brand_name ASC;
```

🟡 **Sample output**

| brand_name | No of Model | AVG Price | AVG Rating | AVG Screen Size | AVG Battery capacity |
|---|---|---|---|---|---|
| apple | 21 | 76842.19 | 80.50 | 6.10 | 3300.00 |
| asus | 12 | 41420.50 | 83.20 | 6.45 | 5100.00 |
| samsung | 85 | 36050.31 | 79.10 | 6.50 | 4900.20 |

### 7.2 Group by NFC availability – avg price & rating

```sql
SELECT has_nfc, AVG(price) AS 'AVG Price', AVG(rating) AS 'AVG Rating'
FROM phone.smartphones
GROUP BY has_nfc;
```

🟡 **Sample output**

| has_nfc | AVG Price | AVG Rating |
|---|---|---|
| False | 17450.3 | 73.8 |
| True | 52100.6 | 83.4 |

### 7.3 Group by extended memory availability – avg price & rating

```sql
SELECT extended_memory_available, AVG(price) AS 'AVG Price', AVG(rating) AS 'AVG Rating'
FROM phone.smartphones
GROUP BY extended_memory_available;
```

🟡 **Sample output**

| extended_memory_available | AVG Price | AVG Rating |
|---|---|---|
| 0 | 48200.1 | 82.6 |
| 1 | 19870.4 | 74.9 |

### 7.4 Group by brand + processor – models & avg rear camera

```sql
SELECT brand_name, processor_brand,
       COUNT(*)                 AS "No of phones",
       AVG(primary_camera_rear) AS "AVG Camera res"
FROM phone.smartphones
GROUP BY brand_name, processor_brand;
```

🟡 **Sample output**

| brand_name | processor_brand | No of phones | AVG Camera res |
|---|---|---|---|
| apple | bionic | 21 | 12.0 |
| samsung | snapdragon | 44 | 52.4 |
| samsung | exynos | 30 | 50.8 |

### 7.5 Top 5 costliest brands

```sql
SELECT brand_name, COUNT(*) AS "No_phones", AVG(price) AS "AVG_price"
FROM phone.smartphones
GROUP BY brand_name
ORDER BY AVG_price DESC
LIMIT 5;
```

🟡 **Sample output**

| brand_name | No_phones | AVG_price |
|---|---|---|
| royal enfield | 1 | 124999 |
| apple | 21 | 76842 |
| sony | 3 | 71833 |

> 💡 Brands with a single phone can top this list. Pair it with `HAVING COUNT(*) > N` (see Section 8).

### 7.6 Brands making the smallest-screen smartphones

```sql
SELECT brand_name, AVG(screen_size) AS "AVG_SCREEN"
FROM phone.smartphones
GROUP BY brand_name
ORDER BY AVG_SCREEN ASC
LIMIT 5;
```

🟡 **Sample output**

| brand_name | AVG_SCREEN |
|---|---|
| sony | 5.50 |
| apple | 6.10 |
| jio | 6.30 |

### 7.7 Average price: 5G vs non-5G

```sql
SELECT has_5g, AVG(price) AS "AVG_price"
FROM phone.smartphones
GROUP BY has_5g
ORDER BY AVG_price ASC
LIMIT 5;
```

> 💡 `LIMIT 5` is redundant because only two groups exist.

🟡 **Sample output**

| has_5g | AVG_price |
|---|---|
| False | 14200.8 |
| True | 41350.2 |

### 7.8 Brand with the most models having both NFC and IR blaster

```sql
SELECT brand_name, COUNT(*) AS 'count'
FROM phone.smartphones
WHERE has_nfc = 'True' AND has_ir_blaster = 'True'
GROUP BY brand_name
ORDER BY count DESC;
```

🟡 **Sample output**

| brand_name | count |
|---|---|
| xiaomi | 18 |
| oneplus | 11 |
| samsung | 5 |

### 7.9 Samsung phones – avg price for NFC vs non-NFC

```sql
SELECT brand_name, COUNT(*) AS 'count', has_nfc, AVG(price)
FROM phone.smartphones
WHERE brand_name = 'samsung'
GROUP BY has_nfc;
```

> ⚠️ Your original question said **"5G enabled"**, but this query doesn't filter on `has_5g`. To match the question, add `AND has_5g = 'True'` to the `WHERE` clause.

🟡 **Sample output**

| brand_name | count | has_nfc | AVG(price) |
|---|---|---|---|
| samsung | 52 | False | 21800.5 |
| samsung | 33 | True | 62400.9 |

---

## 8. HAVING Clause

> 💡 `WHERE` = filter **rows** (before grouping). `HAVING` = filter **groups** (after grouping), so it can use `COUNT()`, `AVG()`, etc.

### 8.1 Avg rating of brands with more than 20 phones

```sql
SELECT brand_name, COUNT(*) AS 'count', ROUND(AVG(rating)) AS 'Avg_rating'
FROM phone.smartphones
GROUP BY brand_name
HAVING count > 20
ORDER BY Avg_rating DESC;
```

🟡 **Sample output**

| brand_name | count | Avg_rating |
|---|---|---|
| oneplus | 25 | 84 |
| apple | 21 | 81 |
| samsung | 85 | 79 |

### 8.2 Top 3 brands by avg RAM (refresh rate > 90, fast charging, > 10 phones)

```sql
SELECT brand_name, COUNT(*) AS "count", AVG(ram_capacity) AS 'AVG_Ram'
FROM phone.smartphones
WHERE refresh_rate > 90 AND fast_charging_available = 1
GROUP BY brand_name
HAVING count > 10
ORDER BY AVG_Ram DESC
LIMIT 3;
```

**Logic:** `WHERE` (rows) → `GROUP BY` → `HAVING` (groups) → `ORDER BY` → `LIMIT`.

🟡 **Sample output**

| brand_name | count | AVG_Ram |
|---|---|---|
| oneplus | 19 | 10.6 |
| iqoo | 14 | 9.4 |
| xiaomi | 33 | 7.9 |

### 8.3 Avg price of 5G brands with avg rating > 70 and > 10 phones

```sql
SELECT brand_name, COUNT(*) AS "count",
       AVG(rating) AS 'AVG_rating', AVG(price) AS 'AVG_price'
FROM phone.smartphones
WHERE has_5g = 'True'
GROUP BY brand_name
HAVING count > 10 AND AVG_rating > 70
ORDER BY AVG_price DESC;
```

🟡 **Sample output**

| brand_name | count | AVG_rating | AVG_price |
|---|---|---|---|
| apple | 18 | 80.9 | 79200.0 |
| samsung | 41 | 80.2 | 54300.5 |
| oneplus | 22 | 84.1 | 38900.0 |

---

## 9. Practice with the IPL Dataset (`phone.ipl`)

### 9.1 Top 5 batsmen by total runs

```sql
SELECT batter, SUM(total_run) AS "runs"
FROM phone.ipl
GROUP BY batter
ORDER BY runs DESC
LIMIT 5;
```

> ⚠️ `total_run` = `batsman_run + extras_run`, so extras (wides, byes) get credited to the batter. To count **only the batter's runs**, use `SUM(batsman_run)`. Section 9.3 already does this.

🟡 **Sample output**

| batter | runs |
|---|---|
| V Kohli | 6900 |
| S Dhawan | 6500 |
| DA Warner | 6200 |
| RG Sharma | 6100 |
| SK Raina | 5600 |

### 9.2 2nd highest six-hitter

```sql
SELECT batter, COUNT(batsman_run) AS "sixes"
FROM phone.ipl
WHERE batsman_run = 6
GROUP BY batter
ORDER BY sixes DESC
LIMIT 5;
```

> ⚠️ This lists the **top 5** six-hitters. To return only the 2nd, replace `LIMIT 5` with `LIMIT 1, 1`.
> 💡 Optional: add `AND non_boundary = 0` to exclude sixes that were not counted as boundaries.

🟡 **Sample output** *(top 5 version)*

| batter | sixes |
|---|---|
| CH Gayle | 359 |
| RG Sharma | 280 |
| V Kohli | 270 |
| AB de Villiers | 251 |
| MS Dhoni | 239 |

### 9.3 Top 5 batsmen by strike rate (minimum 1000 balls)

```sql
SELECT batter,
       SUM(batsman_run)   AS "total_runs",
       COUNT(batsman_run) AS 'Bowls_played',
       ROUND(SUM(batsman_run) / COUNT(batsman_run) * 100, 2) AS "strick_rate"
FROM phone.ipl
GROUP BY batter
HAVING Bowls_played > 1000
ORDER BY strick_rate DESC
LIMIT 5;
```

> 💡 **Strike rate = (total runs ÷ balls faced) × 100.**
> ⚠️ Cosmetic: aliases `Bowls_played` and `strick_rate` are misspelled. Consider `Balls_played` and `strike_rate`.
> ⚠️ Accuracy: `COUNT(batsman_run)` counts every delivery, including wides, which are not "balls faced". For exact figures, add `WHERE extra_type <> 'wides'` (check the actual value in your data).

🟡 **Sample output**

| batter | total_runs | Bowls_played | strick_rate |
|---|---|---|---|
| AD Russell | 2100 | 1180 | 177.97 |
| HH Pandya | 1900 | 1130 | 168.14 |
| RR Pant | 2800 | 1750 | 160.00 |
| KA Pollard | 3300 | 2100 | 157.14 |
| SA Yadav | 2500 | 1650 | 151.52 |

---

## 10. Not Solved Yet

| Question | Status |
|---|---|
| Virat Kohli's performance against all IPL teams | ❌ The table has `BattingTeam` but no **bowling/opposition team** column. Needs a join with a matches table, or a self-join on `ID` to derive the opponent. |
| Top 10 batsmen with centuries | ⚠️ No query written yet. A suggested attempt is below. |

### 💡 Suggested attempt – centuries (untested)

A century = 100+ runs by one batter in **one match** (`ID` identifies the match).

```sql
SELECT batter, COUNT(*) AS centuries
FROM (
    SELECT ID, batter, SUM(batsman_run) AS runs
    FROM phone.ipl
    GROUP BY ID, batter
    HAVING runs >= 100
) t
GROUP BY batter
ORDER BY centuries DESC
LIMIT 10;
```

🟡 **Sample output**

| batter | centuries |
|---|---|
| V Kohli | 8 |
| CH Gayle | 6 |
| DA Warner | 4 |

---

## 11. Cheat Sheet

| Goal | Pattern |
|---|---|
| Unique values | `SELECT DISTINCT col ...` |
| Range filter (inclusive) | `col BETWEEN a AND b` |
| Match a list / exclude a list | `col IN (...)` / `col NOT IN (...)` |
| Top-N | `ORDER BY col DESC LIMIT N` |
| Nth highest row | `ORDER BY col DESC LIMIT N-1, 1` |
| Stable ordering with ties | `ORDER BY col DESC, other_col ASC` |
| Count unique | `COUNT(DISTINCT col)` |
| Aggregate per category | `GROUP BY category` |
| Filter on an aggregate | `HAVING COUNT(*) > N` |
| Filter rows before grouping | `WHERE ... GROUP BY ...` |
| Derived column | `SELECT a + b AS total` |
| Per-match totals then roll-up | Subquery in `FROM` + outer `GROUP BY` |

### Common mistakes checklist
- [ ] Using `WHERE` for an aggregate condition (use `HAVING`).
- [ ] Forgetting `WHERE` on `UPDATE`/`DELETE`.
- [ ] Comparing a text column (`'True'`) with a number, or an int column with text.
- [ ] `LIMIT 1, 1` returning the wrong row when values tie (add a tie-breaker).
- [ ] `NOT IN` silently dropping `NULL` rows.
- [ ] Selecting a non-aggregated column that isn't in `GROUP BY`.
- [ ] Ranking brands with only one phone (add `HAVING COUNT(*) > N`).
- [ ] Counting wides as balls faced in strike rate.
