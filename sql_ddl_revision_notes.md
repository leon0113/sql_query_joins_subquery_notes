## 12. DDL – Data Definition Language (MySQL)

> DDL commands define or change the **structure** of the database (tables, columns, constraints), not the data inside.
> 🟡 Sample outputs are **illustrative**.

| Category | Commands | Works on |
|---|---|---|
| **DDL** | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME` | Structure |
| **DML** | `INSERT`, `UPDATE`, `DELETE` | Data (rows) |
| **DQL** | `SELECT` | Reading data |

> ⚠️ In MySQL, every DDL statement **auto-commits**. You cannot `ROLLBACK` a `DROP`, `TRUNCATE`, or `ALTER`.

---

## 12.1 Constraints at a Glance

| Constraint | Purpose | Example |
|---|---|---|
| `NOT NULL` | Value is mandatory | `name VARCHAR(50) NOT NULL` |
| `UNIQUE` | No duplicates (NULLs allowed) | `email VARCHAR(100) UNIQUE` |
| `PRIMARY KEY` | Unique + NOT NULL, one per table | `user_id INT PRIMARY KEY` |
| `FOREIGN KEY` | Links to a parent table's key | `FOREIGN KEY (user_id) REFERENCES users(user_id)` |
| `CHECK` | Value must satisfy a condition | `CHECK (age >= 18)` |
| `DEFAULT` | Value used when none is given | `status VARCHAR(10) DEFAULT 'active'` |
| `AUTO_INCREMENT` | Auto-generates sequential numbers | `user_id INT AUTO_INCREMENT` |

> 💡 **PRIMARY KEY = UNIQUE + NOT NULL.** A table can have only **one** primary key (it may span several columns) but **many** UNIQUE constraints.

---

## 12.2 CREATE – Database & Table

### Create a database

```sql
CREATE DATABASE IF NOT EXISTS phone;
USE phone;
```

### Create a table with column-level constraints

```sql
CREATE TABLE phone.users (
    user_id    INT           AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(50)   NOT NULL,
    email      VARCHAR(100)  NOT NULL UNIQUE,
    password   VARCHAR(100)  NOT NULL,
    age        INT           CHECK (age >= 18),
    status     VARCHAR(10)   DEFAULT 'active',
    created_at TIMESTAMP     DEFAULT CURRENT_TIMESTAMP
);
```

### Create a table with table-level (named) constraints

Naming constraints makes them easy to drop or modify later.

```sql
CREATE TABLE phone.orders (
    order_id   INT AUTO_INCREMENT,
    user_id    INT           NOT NULL,
    model      VARCHAR(100)  NOT NULL,
    amount     INT           NOT NULL,
    order_date DATE          DEFAULT (CURRENT_DATE),

    CONSTRAINT pk_orders     PRIMARY KEY (order_id),
    CONSTRAINT chk_amount    CHECK (amount > 0),
    CONSTRAINT fk_orders_user
        FOREIGN KEY (user_id) REFERENCES phone.users(user_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

### Composite primary key

```sql
CREATE TABLE phone.wishlist (
    user_id INT,
    model   VARCHAR(100),
    PRIMARY KEY (user_id, model)
);
```

### Foreign key actions

| Option | What happens to child rows when the parent row is deleted/updated |
|---|---|
| `CASCADE` | Child rows are deleted/updated too |
| `SET NULL` | Child column becomes `NULL` (column must allow NULL) |
| `RESTRICT` / `NO ACTION` | The parent change is **blocked** (default) |
| `SET DEFAULT` | Not supported by InnoDB |

### Inspect the structure

```sql
DESCRIBE phone.users;           -- columns, types, keys
SHOW CREATE TABLE phone.users;  -- full DDL including constraint names
SHOW TABLES;
```

🟡 **Sample output of `DESCRIBE`**

| Field | Type | Null | Key | Default | Extra |
|---|---|---|---|---|---|
| user_id | int | NO | PRI | NULL | auto_increment |
| name | varchar(50) | NO | | NULL | |
| email | varchar(100) | NO | UNI | NULL | |
| age | int | YES | | NULL | |
| status | varchar(10) | YES | | active | |

---

## 12.3 ALTER TABLE – Add

### Add a column

```sql
ALTER TABLE phone.users
ADD COLUMN phone_no VARCHAR(15);
```

### Add a column with constraints, in a specific position

```sql
ALTER TABLE phone.users
ADD COLUMN country VARCHAR(30) NOT NULL DEFAULT 'BD' AFTER email;
-- or FIRST to place it as the first column
```

### Add multiple columns

```sql
ALTER TABLE phone.users
ADD COLUMN city VARCHAR(30),
ADD COLUMN dob  DATE;
```

### Add constraints to an existing table

```sql
-- PRIMARY KEY
ALTER TABLE phone.smartphones
ADD COLUMN phone_id INT AUTO_INCREMENT PRIMARY KEY FIRST;

-- UNIQUE
ALTER TABLE phone.users
ADD CONSTRAINT uq_users_phone UNIQUE (phone_no);

-- CHECK  (enforced from MySQL 8.0.16+; older versions parse but ignore it)
ALTER TABLE phone.smartphones
ADD CONSTRAINT chk_price  CHECK (price > 0),
ADD CONSTRAINT chk_rating CHECK (rating BETWEEN 0 AND 100);

-- DEFAULT
ALTER TABLE phone.users
ALTER COLUMN status SET DEFAULT 'inactive';

-- NOT NULL  (done through MODIFY, see 12.4)

-- FOREIGN KEY
ALTER TABLE phone.orders
ADD CONSTRAINT fk_orders_user
FOREIGN KEY (user_id) REFERENCES phone.users(user_id)
ON DELETE CASCADE;
```

> ⚠️ Adding a constraint **validates existing rows**. `ADD CHECK (price > 0)` fails if any row already has `price <= 0`. Adding `UNIQUE` fails if duplicates exist. Clean the data first.

---

## 12.4 ALTER TABLE – Modify

### Change a column's data type or size

```sql
ALTER TABLE phone.users
MODIFY COLUMN name VARCHAR(100);
```

### Add / remove NOT NULL

```sql
-- Make NOT NULL (fails if the column already contains NULLs)
ALTER TABLE phone.users
MODIFY COLUMN phone_no VARCHAR(15) NOT NULL;

-- Make nullable again
ALTER TABLE phone.users
MODIFY COLUMN phone_no VARCHAR(15) NULL;
```

> ⚠️ **`MODIFY` redefines the entire column.** Anything you don't repeat (`NOT NULL`, `DEFAULT`, `AUTO_INCREMENT`) is **lost**.
> Wrong: `MODIFY COLUMN status VARCHAR(20)` silently drops `DEFAULT 'active'`.
> Right: `MODIFY COLUMN status VARCHAR(20) DEFAULT 'active'`.

### Change a column's name (and optionally its definition)

```sql
-- MySQL 8.0+
ALTER TABLE phone.users
RENAME COLUMN phone_no TO mobile_no;

-- Works in all versions (must restate the full definition)
ALTER TABLE phone.users
CHANGE COLUMN mobile_no phone_no VARCHAR(20) NOT NULL;
```

| Command | Rename? | Change type/constraints? |
|---|---|---|
| `RENAME COLUMN` | ✅ | ❌ |
| `MODIFY COLUMN` | ❌ | ✅ |
| `CHANGE COLUMN` | ✅ | ✅ |

### Change or reset a default

```sql
ALTER TABLE phone.users ALTER COLUMN status SET DEFAULT 'pending';
ALTER TABLE phone.users ALTER COLUMN status DROP DEFAULT;
```

### Modify a constraint (drop and re-create)

MySQL cannot edit most constraints in place. The pattern is **drop, then add**, in one statement:

```sql
-- Change CHECK rule
ALTER TABLE phone.smartphones
DROP CHECK chk_price,
ADD CONSTRAINT chk_price CHECK (price BETWEEN 1000 AND 500000);

-- Change FOREIGN KEY action (CASCADE -> SET NULL)
ALTER TABLE phone.orders
DROP FOREIGN KEY fk_orders_user,
ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES phone.users(user_id)
    ON DELETE SET NULL;
```

> 💡 `SET NULL` requires `user_id` to be nullable, so remove `NOT NULL` first.

Temporarily switch a CHECK off without deleting it (8.0.16+):

```sql
ALTER TABLE phone.smartphones ALTER CHECK chk_price NOT ENFORCED;
ALTER TABLE phone.smartphones ALTER CHECK chk_price ENFORCED;
```

### Rename a table

```sql
ALTER TABLE phone.users RENAME TO phone.customers;
-- or
RENAME TABLE phone.customers TO phone.users;
```

---

## 12.5 ALTER TABLE – Delete (Drop)

### Drop a column

```sql
ALTER TABLE phone.users DROP COLUMN city;

-- Drop several at once
ALTER TABLE phone.users DROP COLUMN dob, DROP COLUMN country;
```

### Drop constraints

| Constraint | Syntax |
|---|---|
| PRIMARY KEY | `ALTER TABLE t DROP PRIMARY KEY;` |
| UNIQUE | `ALTER TABLE t DROP INDEX uq_users_phone;` |
| FOREIGN KEY | `ALTER TABLE t DROP FOREIGN KEY fk_orders_user;` |
| CHECK | `ALTER TABLE t DROP CHECK chk_price;` *(8.0.16+; use `DROP CONSTRAINT` in 8.0.19+)* |
| DEFAULT | `ALTER TABLE t ALTER COLUMN status DROP DEFAULT;` |
| NOT NULL | `ALTER TABLE t MODIFY COLUMN col TYPE NULL;` |

```sql
ALTER TABLE phone.users  DROP INDEX uq_users_phone;
ALTER TABLE phone.orders DROP FOREIGN KEY fk_orders_user;
ALTER TABLE phone.smartphones DROP CHECK chk_rating;
```

> ⚠️ **Dropping a PRIMARY KEY on an `AUTO_INCREMENT` column fails.** Remove `AUTO_INCREMENT` first with `MODIFY`.
> ⚠️ Dropping a foreign key does **not** drop its backing index. Use `DROP INDEX fk_orders_user` afterwards if you want it gone.
> 💡 Forgot the constraint name? Run `SHOW CREATE TABLE table_name;`.

🟡 **Sample output** (any successful ALTER)

```
Query OK, 0 rows affected (0.06 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

---

## 12.6 DROP, TRUNCATE, DELETE – Know the Difference

```sql
DROP TABLE IF EXISTS phone.wishlist;       -- removes data + structure
TRUNCATE TABLE phone.orders;               -- removes all rows, keeps structure
DELETE FROM phone.orders WHERE amount < 500; -- removes selected rows
DROP DATABASE IF EXISTS phone_backup;      -- removes the whole database
```

| Feature | `DELETE` | `TRUNCATE` | `DROP` |
|---|---|---|---|
| Category | DML | DDL | DDL |
| Removes | Chosen rows | All rows | Table + data + structure |
| `WHERE` clause | ✅ | ❌ | ❌ |
| Rollback | ✅ (inside a transaction) | ❌ | ❌ |
| Resets `AUTO_INCREMENT` | ❌ | ✅ | n/a |
| Fires triggers | ✅ | ❌ | ❌ |
| Speed | Slow (row by row) | Fast | Fast |

> ⚠️ `TRUNCATE` is blocked on a table that is **referenced by a foreign key**. Drop the FK, or delete child rows first. `DROP TABLE` on a parent fails for the same reason.

---

## 12.7 Putting It Together – Full Example Flow

```sql
-- 1. Create
CREATE TABLE phone.reviews (
    review_id INT AUTO_INCREMENT PRIMARY KEY,
    model     VARCHAR(100) NOT NULL,
    stars     INT
);

-- 2. Add a column and constraints
ALTER TABLE phone.reviews ADD COLUMN reviewer VARCHAR(50);
ALTER TABLE phone.reviews ADD CONSTRAINT chk_stars CHECK (stars BETWEEN 1 AND 5);

-- 3. Modify
ALTER TABLE phone.reviews MODIFY COLUMN reviewer VARCHAR(100) NOT NULL DEFAULT 'anonymous';
ALTER TABLE phone.reviews RENAME COLUMN stars TO rating_stars;

-- 4. Delete a constraint and a column
ALTER TABLE phone.reviews DROP CHECK chk_stars;
ALTER TABLE phone.reviews DROP COLUMN reviewer;

-- 5. Remove the table
DROP TABLE phone.reviews;
```

---

## 12.8 DDL Interview Q&A

| Question | Short answer |
|---|---|
| PRIMARY KEY vs UNIQUE? | PK: one per table, no NULLs. UNIQUE: many per table, NULLs allowed (MySQL allows multiple NULLs). |
| Can a table have no primary key? | Yes, but it is bad practice. |
| What is a foreign key? | A column that references a primary/unique key in another table to enforce referential integrity. |
| DELETE vs TRUNCATE vs DROP? | See the table in 12.6. |
| Why did `MODIFY` remove my default? | `MODIFY` replaces the whole column definition, so restate everything you want to keep. |
| How do you change a constraint? | Drop it and add it again, ideally in one `ALTER TABLE`. |
| Does DDL support rollback? | No. MySQL auto-commits every DDL statement. |
| Can you add NOT NULL to a column with NULLs? | No. Update the NULLs first, e.g. `UPDATE t SET col = 0 WHERE col IS NULL;`. |
| What is `ON DELETE CASCADE`? | Deleting a parent row automatically deletes its child rows. |

### DDL common-mistakes checklist
- [ ] Using `MODIFY` and forgetting to repeat `NOT NULL`/`DEFAULT`/`AUTO_INCREMENT`.
- [ ] Adding `UNIQUE`/`CHECK`/`NOT NULL` before cleaning existing bad data.
- [ ] Not naming constraints, which makes dropping them painful.
- [ ] Using `DROP INDEX` for a foreign key (use `DROP FOREIGN KEY`) or `DROP CONSTRAINT` on old MySQL versions.
- [ ] Running `TRUNCATE`/`DROP` and expecting a rollback.
- [ ] Assuming `CHECK` works on MySQL older than 8.0.16 (it is silently ignored).
- [ ] Dropping a parent table before its child tables.
