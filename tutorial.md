# PostgreSQL: Basic to Expert

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Installation & Setup](#2-installation--setup)
3. [Basic Concepts](#3-basic-concepts)
4. [Data Types](#4-data-types)
5. [DDL — Creating & Managing Tables](#5-ddl--creating--managing-tables)
6. [DML — CRUD Operations](#6-dml--crud-operations)
7. [Querying Data](#7-querying-data)
8. [Joins](#8-joins)
9. [Aggregation & Grouping](#9-aggregation--grouping)
10. [Indexes](#10-indexes)
11. [Constraints](#11-constraints)
12. [Transactions](#12-transactions)
13. [Views](#13-views)
14. [Functions & Stored Procedures](#14-functions--stored-procedures)
15. [Triggers](#15-triggers)
16. [Window Functions](#16-window-functions)
17. [CTEs (Common Table Expressions)](#17-ctes-common-table-expressions)
18. [JSON & JSONB](#18-json--jsonb)
19. [Full-Text Search](#19-full-text-search)
20. [Partitioning](#20-partitioning)
21. [Roles & Permissions](#21-roles--permissions)
22. [Performance Tuning](#22-performance-tuning)
23. [Replication & High Availability](#23-replication--high-availability)
24. [Extensions](#24-extensions)
25. [Expert Patterns](#25-expert-patterns)

---

## 1. Introduction

PostgreSQL (Postgres) — open-source object-relational database. ACID-compliant. Supports SQL standard + advanced features: JSON, full-text search, window functions, custom types.

**Key facts:**
- Default port: `5432`
- Config file: `postgresql.conf`
- Auth file: `pg_hba.conf`

---

## 2. Installation & Setup

### Windows / macOS
Download from https://www.postgresql.org/download/

### Ubuntu/Debian
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Connect via psql
```bash
# As system postgres user
sudo -u postgres psql

# With connection string
psql postgresql://username:password@localhost:5432/dbname

# With flags
psql -h localhost -p 5432 -U postgres -d mydb
```

### Useful psql meta-commands
```sql
\l              -- list databases
\c dbname       -- connect to database
\dt             -- list tables
\d tablename    -- describe table
\di             -- list indexes
\df             -- list functions
\du             -- list roles/users
\timing         -- toggle query timing
\e              -- open editor
\q              -- quit
\?              -- help for meta-commands
\h SELECT       -- help for SQL command
```

---

## 3. Basic Concepts

| Concept     | Description |
|-------------|-------------|
| Database    | Top-level container |
| Schema      | Namespace inside database (default: `public`) |
| Table       | Rows + columns |
| Row (tuple) | Single record |
| Column (attribute) | Field with a type |
| Primary Key | Unique row identifier |
| Foreign Key | Reference to another table's PK |

```sql
-- Create database
CREATE DATABASE shop;

-- Create schema
CREATE SCHEMA inventory;

-- Use schema-qualified name
SELECT * FROM inventory.products;
```

---

## 4. Data Types

### Numeric
```sql
SMALLINT          -- 2 bytes, -32768 to 32767
INTEGER / INT     -- 4 bytes
BIGINT            -- 8 bytes
DECIMAL(p, s)     -- exact numeric, e.g. DECIMAL(10,2)
NUMERIC(p, s)     -- same as DECIMAL
REAL              -- 4-byte float
DOUBLE PRECISION  -- 8-byte float
SERIAL            -- auto-increment integer (4 bytes)
BIGSERIAL         -- auto-increment bigint
```

### Text
```sql
CHAR(n)           -- fixed-length, padded
VARCHAR(n)        -- variable-length with limit
TEXT              -- unlimited length
```

### Date / Time
```sql
DATE              -- YYYY-MM-DD
TIME              -- HH:MI:SS
TIMESTAMP         -- date + time (no timezone)
TIMESTAMPTZ       -- date + time + timezone (preferred)
INTERVAL          -- time span, e.g. '3 days 2 hours'
```

### Boolean
```sql
BOOLEAN           -- TRUE / FALSE / NULL
```

### Special
```sql
UUID              -- universally unique identifier
BYTEA             -- binary data
ARRAY             -- e.g. INTEGER[], TEXT[]
JSON              -- stored as text
JSONB             -- stored as binary (indexable, preferred)
ENUM              -- user-defined set of values
```

---

## 5. DDL — Creating & Managing Tables

```sql
-- Basic table
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    username    VARCHAR(100) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Table with foreign key
CREATE TABLE posts (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title       TEXT NOT NULL,
    body        TEXT,
    published   BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Alter table
ALTER TABLE users ADD COLUMN bio TEXT;
ALTER TABLE users ALTER COLUMN username SET NOT NULL;
ALTER TABLE users RENAME COLUMN username TO handle;
ALTER TABLE users DROP COLUMN bio;

-- Drop table
DROP TABLE IF EXISTS posts;

-- Truncate (delete all rows, fast)
TRUNCATE TABLE posts;
TRUNCATE TABLE posts RESTART IDENTITY CASCADE;
```

---

## 6. DML — CRUD Operations

### INSERT
```sql
-- Single row
INSERT INTO users (email, username)
VALUES ('alice@example.com', 'alice');

-- Multiple rows
INSERT INTO users (email, username) VALUES
    ('bob@example.com', 'bob'),
    ('carol@example.com', 'carol');

-- Insert with RETURNING
INSERT INTO users (email, username)
VALUES ('dave@example.com', 'dave')
RETURNING id, created_at;

-- Upsert (insert or update on conflict)
INSERT INTO users (email, username)
VALUES ('alice@example.com', 'alice_v2')
ON CONFLICT (email)
DO UPDATE SET username = EXCLUDED.username;

-- Upsert — do nothing on conflict
INSERT INTO users (email, username)
VALUES ('alice@example.com', 'alice')
ON CONFLICT (email) DO NOTHING;
```

### SELECT
```sql
SELECT * FROM users;
SELECT id, email FROM users;
SELECT id, email AS mail FROM users;
SELECT DISTINCT username FROM users;
```

### UPDATE
```sql
UPDATE users
SET username = 'alice_updated'
WHERE email = 'alice@example.com';

-- Update with RETURNING
UPDATE users
SET username = 'alice_new'
WHERE id = 1
RETURNING *;

-- Update using join
UPDATE posts p
SET published = TRUE
FROM users u
WHERE p.user_id = u.id
  AND u.email = 'alice@example.com';
```

### DELETE
```sql
DELETE FROM users WHERE id = 5;

-- Delete with RETURNING
DELETE FROM users WHERE id = 5 RETURNING *;

-- Delete with subquery
DELETE FROM posts
WHERE user_id IN (
    SELECT id FROM users WHERE created_at < '2024-01-01'
);
```

---

## 7. Querying Data

### WHERE
```sql
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE email LIKE '%@example.com';
SELECT * FROM users WHERE email ILIKE '%alice%';   -- case-insensitive
SELECT * FROM users WHERE id IN (1, 2, 3);
SELECT * FROM users WHERE id BETWEEN 1 AND 10;
SELECT * FROM users WHERE bio IS NULL;
SELECT * FROM users WHERE bio IS NOT NULL;
```

### ORDER BY, LIMIT, OFFSET
```sql
SELECT * FROM users ORDER BY created_at DESC;
SELECT * FROM users ORDER BY username ASC NULLS LAST;
SELECT * FROM users LIMIT 10 OFFSET 20;   -- page 3 of 10
```

### Pattern Matching
```sql
-- LIKE / ILIKE
SELECT * FROM users WHERE username LIKE 'al%';    -- starts with 'al'
SELECT * FROM users WHERE username LIKE '%ice';   -- ends with 'ice'
SELECT * FROM users WHERE username LIKE '%lic%';  -- contains 'lic'

-- Regex
SELECT * FROM users WHERE username ~ '^al';        -- case-sensitive regex
SELECT * FROM users WHERE username ~* '^al';       -- case-insensitive regex
```

### Conditional Expressions
```sql
-- CASE
SELECT id, email,
    CASE
        WHEN created_at > NOW() - INTERVAL '7 days' THEN 'new'
        WHEN created_at > NOW() - INTERVAL '30 days' THEN 'recent'
        ELSE 'old'
    END AS user_age
FROM users;

-- COALESCE (first non-null)
SELECT COALESCE(bio, 'No bio') FROM users;

-- NULLIF (return null if equal)
SELECT NULLIF(score, 0) FROM results;
```

---

## 8. Joins

```sql
-- Setup
CREATE TABLE orders (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT REFERENCES users(id),
    total      NUMERIC(10,2),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- INNER JOIN — only matching rows
SELECT u.email, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN — all users, even without orders
SELECT u.email, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- RIGHT JOIN — all orders even if user missing
SELECT u.email, o.total
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- FULL OUTER JOIN — all rows from both sides
SELECT u.email, o.total
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- CROSS JOIN — cartesian product
SELECT u.username, p.title
FROM users u CROSS JOIN posts p;

-- SELF JOIN
SELECT a.username AS user1, b.username AS user2
FROM users a
JOIN users b ON a.id < b.id;  -- avoid duplicate pairs
```

---

## 9. Aggregation & Grouping

```sql
-- Aggregate functions
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT email) FROM users;
SELECT SUM(total), AVG(total), MIN(total), MAX(total) FROM orders;

-- GROUP BY
SELECT user_id, COUNT(*) AS order_count, SUM(total) AS total_spent
FROM orders
GROUP BY user_id;

-- HAVING (filter after grouping)
SELECT user_id, SUM(total) AS total_spent
FROM orders
GROUP BY user_id
HAVING SUM(total) > 500;

-- GROUP BY with JOIN
SELECT u.email, COUNT(o.id) AS orders, SUM(o.total) AS spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.email
ORDER BY spent DESC NULLS LAST;

-- GROUPING SETS — multiple GROUP BY in one query
SELECT user_id, DATE_TRUNC('month', created_at) AS month, SUM(total)
FROM orders
GROUP BY GROUPING SETS (
    (user_id, DATE_TRUNC('month', created_at)),
    (user_id),
    ()
);

-- ROLLUP — hierarchical subtotals
SELECT user_id, DATE_TRUNC('month', created_at), SUM(total)
FROM orders
GROUP BY ROLLUP (user_id, DATE_TRUNC('month', created_at));

-- CUBE — all combinations
SELECT user_id, DATE_TRUNC('month', created_at), SUM(total)
FROM orders
GROUP BY CUBE (user_id, DATE_TRUNC('month', created_at));
```

---

## 10. Indexes

```sql
-- B-Tree (default) — equality, range
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Unique index
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Composite index
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Partial index — only index subset of rows
CREATE INDEX idx_posts_published ON posts(created_at)
WHERE published = TRUE;

-- Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- GIN index — for JSONB, arrays, full-text
CREATE INDEX idx_products_tags ON products USING GIN(tags);

-- GiST index — for geometric, range types
CREATE INDEX idx_events_range ON events USING GIST(during);

-- BRIN index — for large sorted tables (e.g. time-series)
CREATE INDEX idx_logs_created ON logs USING BRIN(created_at);

-- List indexes
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'users';

-- Drop index
DROP INDEX IF EXISTS idx_users_email;

-- Rebuild index (no lock with CONCURRENTLY)
REINDEX INDEX CONCURRENTLY idx_users_email;
```

**Index strategy:**
- Index FK columns always
- Index columns used in WHERE, ORDER BY, JOIN ON
- Composite index: put equality columns first, range columns last
- Avoid over-indexing — writes slow down

---

## 11. Constraints

```sql
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    sku         VARCHAR(50) NOT NULL UNIQUE,
    name        TEXT NOT NULL,
    price       NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    stock       INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    category_id BIGINT REFERENCES categories(id) ON DELETE SET NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Named constraint (easier to drop/identify)
ALTER TABLE products
ADD CONSTRAINT chk_price_positive CHECK (price > 0);

-- Drop constraint
ALTER TABLE products DROP CONSTRAINT chk_price_positive;

-- Exclusion constraint (requires btree_gist extension)
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings
ADD CONSTRAINT no_overlap
EXCLUDE USING GIST (room_id WITH =, during WITH &&);
```

**ON DELETE options:**
| Option | Behavior |
|--------|----------|
| `CASCADE` | Delete child rows |
| `SET NULL` | Set FK to NULL |
| `SET DEFAULT` | Set FK to default value |
| `RESTRICT` | Prevent delete if children exist |
| `NO ACTION` | Same as RESTRICT (default) |

---

## 12. Transactions

```sql
-- Basic transaction
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Rollback on error
BEGIN;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
    -- Something went wrong
ROLLBACK;

-- Savepoints
BEGIN;
    INSERT INTO orders (user_id, total) VALUES (1, 99.99);
    SAVEPOINT sp1;
    INSERT INTO order_items (order_id, product_id) VALUES (currval('orders_id_seq'), 999);
    -- This failed, rollback to savepoint
    ROLLBACK TO SAVEPOINT sp1;
    -- Continue with corrected data
    INSERT INTO order_items (order_id, product_id) VALUES (currval('orders_id_seq'), 1);
COMMIT;

-- Isolation levels
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    -- No dirty reads, non-repeatable reads, or phantom reads
COMMIT;

-- READ COMMITTED (default)
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
COMMIT;

-- REPEATABLE READ
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
COMMIT;
```

**Isolation levels:**
| Level | Dirty Read | Non-repeatable | Phantom |
|-------|-----------|----------------|---------|
| READ UNCOMMITTED | possible | possible | possible |
| READ COMMITTED | no | possible | possible |
| REPEATABLE READ | no | no | no (in PG) |
| SERIALIZABLE | no | no | no |

---

## 13. Views

```sql
-- Simple view
CREATE VIEW user_order_summary AS
SELECT u.id, u.email, COUNT(o.id) AS order_count, SUM(o.total) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.email;

-- Query a view like a table
SELECT * FROM user_order_summary WHERE total_spent > 1000;

-- Replace view
CREATE OR REPLACE VIEW user_order_summary AS
SELECT u.id, u.email, u.username,
       COUNT(o.id) AS order_count,
       COALESCE(SUM(o.total), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.email, u.username;

-- Drop view
DROP VIEW IF EXISTS user_order_summary;

-- Materialized view — stores result physically
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT DATE_TRUNC('month', created_at) AS month, SUM(total) AS revenue
FROM orders
GROUP BY 1
ORDER BY 1;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW monthly_sales;

-- Refresh without locking reads
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales;
-- (requires unique index on materialized view)
CREATE UNIQUE INDEX ON monthly_sales(month);
```

---

## 14. Functions & Stored Procedures

### Functions
```sql
-- Simple function
CREATE OR REPLACE FUNCTION add_numbers(a INTEGER, b INTEGER)
RETURNS INTEGER AS $$
    SELECT a + b;
$$ LANGUAGE SQL;

SELECT add_numbers(3, 5);  -- returns 8

-- PL/pgSQL function with logic
CREATE OR REPLACE FUNCTION get_user_tier(user_id BIGINT)
RETURNS TEXT AS $$
DECLARE
    total_spent NUMERIC;
    tier TEXT;
BEGIN
    SELECT COALESCE(SUM(total), 0)
    INTO total_spent
    FROM orders
    WHERE orders.user_id = get_user_tier.user_id;

    IF total_spent >= 10000 THEN
        tier := 'platinum';
    ELSIF total_spent >= 1000 THEN
        tier := 'gold';
    ELSIF total_spent >= 100 THEN
        tier := 'silver';
    ELSE
        tier := 'bronze';
    END IF;

    RETURN tier;
END;
$$ LANGUAGE plpgsql;

SELECT email, get_user_tier(id) AS tier FROM users;

-- Function returning TABLE
CREATE OR REPLACE FUNCTION top_spenders(n INTEGER)
RETURNS TABLE(email TEXT, total_spent NUMERIC) AS $$
BEGIN
    RETURN QUERY
        SELECT u.email, COALESCE(SUM(o.total), 0) AS total_spent
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
        GROUP BY u.email
        ORDER BY total_spent DESC
        LIMIT n;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM top_spenders(5);
```

### Stored Procedures (PG 11+)
```sql
-- Procedures can use transactions (functions cannot)
CREATE OR REPLACE PROCEDURE transfer_funds(
    from_id BIGINT,
    to_id   BIGINT,
    amount  NUMERIC
)
LANGUAGE plpgsql AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
    COMMIT;
END;
$$;

CALL transfer_funds(1, 2, 500.00);
```

---

## 15. Triggers

```sql
-- Trigger function
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Add updated_at column
ALTER TABLE users ADD COLUMN updated_at TIMESTAMPTZ;

-- Create trigger
CREATE TRIGGER trg_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION update_updated_at();

-- Test
UPDATE users SET username = 'test' WHERE id = 1;
SELECT updated_at FROM users WHERE id = 1;

-- Audit log trigger
CREATE TABLE audit_log (
    id         BIGSERIAL PRIMARY KEY,
    table_name TEXT,
    operation  TEXT,
    old_data   JSONB,
    new_data   JSONB,
    changed_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION log_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, new_data)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        CASE WHEN TG_OP = 'DELETE' THEN row_to_json(OLD)::JSONB ELSE NULL END,
        CASE WHEN TG_OP != 'DELETE' THEN row_to_json(NEW)::JSONB ELSE NULL END
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION log_changes();

-- Drop trigger
DROP TRIGGER IF EXISTS trg_audit_users ON users;
```

---

## 16. Window Functions

Window functions operate over a "window" of rows related to current row — no GROUP BY needed.

```sql
-- ROW_NUMBER, RANK, DENSE_RANK
SELECT
    email,
    total_spent,
    ROW_NUMBER() OVER (ORDER BY total_spent DESC) AS row_num,
    RANK()       OVER (ORDER BY total_spent DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY total_spent DESC) AS dense_rank
FROM user_order_summary;

-- PARTITION BY — restart ranking per group
SELECT
    user_id,
    order_id,
    total,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS order_seq
FROM orders;

-- LAG / LEAD — access previous/next row
SELECT
    user_id,
    created_at,
    total,
    LAG(total)  OVER (PARTITION BY user_id ORDER BY created_at) AS prev_order,
    LEAD(total) OVER (PARTITION BY user_id ORDER BY created_at) AS next_order,
    total - LAG(total) OVER (PARTITION BY user_id ORDER BY created_at) AS diff
FROM orders;

-- Running total (SUM with frame)
SELECT
    created_at,
    total,
    SUM(total) OVER (ORDER BY created_at
                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;

-- Moving average (last 7 rows)
SELECT
    created_at,
    total,
    AVG(total) OVER (ORDER BY created_at
                     ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_7
FROM orders;

-- NTILE — divide into buckets
SELECT email, total_spent,
       NTILE(4) OVER (ORDER BY total_spent) AS quartile
FROM user_order_summary;

-- FIRST_VALUE / LAST_VALUE
SELECT
    user_id,
    total,
    FIRST_VALUE(total) OVER (PARTITION BY user_id ORDER BY created_at) AS first_order,
    LAST_VALUE(total)  OVER (PARTITION BY user_id ORDER BY created_at
                              ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) AS last_order
FROM orders;
```

---

## 17. CTEs (Common Table Expressions)

```sql
-- Basic CTE
WITH top_users AS (
    SELECT user_id, SUM(total) AS spent
    FROM orders
    GROUP BY user_id
    HAVING SUM(total) > 500
)
SELECT u.email, t.spent
FROM users u
JOIN top_users t ON u.id = t.user_id
ORDER BY t.spent DESC;

-- Multiple CTEs
WITH
monthly AS (
    SELECT DATE_TRUNC('month', created_at) AS month, SUM(total) AS revenue
    FROM orders
    GROUP BY 1
),
with_growth AS (
    SELECT
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
        ROUND((revenue - LAG(revenue) OVER (ORDER BY month))
              / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100, 2) AS growth_pct
    FROM monthly
)
SELECT * FROM with_growth ORDER BY month;

-- Recursive CTE — e.g. hierarchy / tree
CREATE TABLE categories (
    id        BIGSERIAL PRIMARY KEY,
    parent_id BIGINT REFERENCES categories(id),
    name      TEXT NOT NULL
);

INSERT INTO categories (name) VALUES ('Electronics');                          -- id=1
INSERT INTO categories (parent_id, name) VALUES (1, 'Phones');                -- id=2
INSERT INTO categories (parent_id, name) VALUES (1, 'Laptops');               -- id=3
INSERT INTO categories (parent_id, name) VALUES (2, 'Smartphones');           -- id=4
INSERT INTO categories (parent_id, name) VALUES (2, 'Feature Phones');        -- id=5

-- Recursive CTE to get full tree
WITH RECURSIVE category_tree AS (
    -- Base case: root nodes
    SELECT id, parent_id, name, 0 AS depth, name::TEXT AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Recursive case
    SELECT c.id, c.parent_id, c.name,
           ct.depth + 1,
           ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT depth, path FROM category_tree ORDER BY path;
```

---

## 18. JSON & JSONB

```sql
-- JSONB column
CREATE TABLE events (
    id      BIGSERIAL PRIMARY KEY,
    name    TEXT,
    payload JSONB
);

INSERT INTO events (name, payload) VALUES
('purchase', '{"user_id": 1, "items": [{"sku": "A1", "qty": 2}, {"sku": "B3", "qty": 1}], "total": 89.99}'),
('signup',   '{"user_id": 2, "referral": "google", "plan": "free"}');

-- Access operators
SELECT payload -> 'user_id'          FROM events;   -- returns JSON
SELECT payload ->> 'user_id'         FROM events;   -- returns TEXT
SELECT payload -> 'items' -> 0       FROM events;   -- first item
SELECT payload -> 'items' ->> 0      FROM events;   -- first item as text
SELECT payload #> '{items,0,sku}'    FROM events;   -- nested path
SELECT payload #>> '{items,0,sku}'   FROM events;   -- nested path as text

-- WHERE on JSONB
SELECT * FROM events WHERE payload ->> 'referral' = 'google';
SELECT * FROM events WHERE payload @> '{"plan": "free"}';   -- containment
SELECT * FROM events WHERE payload ? 'referral';             -- key exists
SELECT * FROM events WHERE payload ?| ARRAY['referral', 'plan'];  -- any key
SELECT * FROM events WHERE payload ?& ARRAY['user_id', 'plan'];   -- all keys

-- GIN index for JSONB
CREATE INDEX idx_events_payload ON events USING GIN(payload);

-- Modify JSONB
UPDATE events
SET payload = payload || '{"processed": true}'::JSONB
WHERE name = 'purchase';

UPDATE events
SET payload = payload - 'referral'   -- remove key
WHERE name = 'signup';

-- jsonb_set — update nested value
UPDATE events
SET payload = jsonb_set(payload, '{plan}', '"pro"')
WHERE name = 'signup';

-- Expand array to rows
SELECT id, jsonb_array_elements(payload -> 'items') AS item
FROM events WHERE name = 'purchase';

-- jsonb_each — expand object to key/value pairs
SELECT key, value FROM events, jsonb_each(payload)
WHERE id = 1;

-- Aggregate to JSONB
SELECT jsonb_agg(email) FROM users;
SELECT jsonb_object_agg(username, email) FROM users;

-- Build JSONB from row
SELECT row_to_json(u)::JSONB FROM users u;
SELECT to_jsonb(u) FROM users u;
```

---

## 19. Full-Text Search

```sql
-- tsvector and tsquery
SELECT to_tsvector('english', 'PostgreSQL is a powerful open-source database');
-- Result: 'databas':8 'open-sourc':6 'postgresql':1 'powerful':4

SELECT to_tsquery('english', 'postgres & database');
SELECT plainto_tsquery('english', 'postgres database');  -- phrase
SELECT websearch_to_tsquery('english', 'postgres OR mysql database');

-- Match
SELECT 'PostgreSQL is great'::tsvector @@ 'postgres'::tsquery;

-- Full-text on a table
ALTER TABLE posts ADD COLUMN tsv TSVECTOR;

UPDATE posts
SET tsv = to_tsvector('english', COALESCE(title,'') || ' ' || COALESCE(body,''));

CREATE INDEX idx_posts_tsv ON posts USING GIN(tsv);

-- Trigger to keep tsv updated
CREATE OR REPLACE FUNCTION posts_tsv_trigger()
RETURNS TRIGGER AS $$
BEGIN
    NEW.tsv = to_tsvector('english',
        COALESCE(NEW.title,'') || ' ' || COALESCE(NEW.body,''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_posts_tsv
BEFORE INSERT OR UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION posts_tsv_trigger();

-- Search
SELECT title, ts_rank(tsv, query) AS rank
FROM posts, plainto_tsquery('english', 'postgresql database') AS query
WHERE tsv @@ query
ORDER BY rank DESC;

-- Highlight matches
SELECT ts_headline('english', body,
                   plainto_tsquery('english', 'database'),
                   'MaxWords=30, MinWords=10')
FROM posts
WHERE tsv @@ plainto_tsquery('english', 'database');
```

---

## 20. Partitioning

Partitioning splits large tables into smaller physical pieces while keeping one logical table.

```sql
-- Range partitioning by month
CREATE TABLE orders_partitioned (
    id         BIGSERIAL,
    user_id    BIGINT,
    total      NUMERIC(10,2),
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE orders_2024_01 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Default partition catches anything else
CREATE TABLE orders_default PARTITION OF orders_partitioned DEFAULT;

-- List partitioning
CREATE TABLE users_by_region (
    id     BIGSERIAL,
    name   TEXT,
    region TEXT NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE users_us PARTITION OF users_by_region FOR VALUES IN ('US', 'CA');
CREATE TABLE users_eu PARTITION OF users_by_region FOR VALUES IN ('DE', 'FR', 'GB');

-- Hash partitioning (distribute evenly)
CREATE TABLE logs_hash (
    id      BIGSERIAL,
    message TEXT,
    user_id BIGINT NOT NULL
) PARTITION BY HASH (user_id);

CREATE TABLE logs_hash_0 PARTITION OF logs_hash FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE logs_hash_1 PARTITION OF logs_hash FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE logs_hash_2 PARTITION OF logs_hash FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE logs_hash_3 PARTITION OF logs_hash FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Inspect partitions
SELECT inhrelid::regclass AS partition, inhparent::regclass AS parent
FROM pg_inherits
WHERE inhparent = 'orders_partitioned'::regclass;
```

---

## 21. Roles & Permissions

```sql
-- Create role/user
CREATE ROLE readonly;
CREATE ROLE app_user LOGIN PASSWORD 'securepass';
CREATE USER admin_user WITH LOGIN PASSWORD 'adminpass' SUPERUSER;

-- Grant privileges
GRANT CONNECT ON DATABASE shop TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- Default privileges (for future objects)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;

-- Readonly role
GRANT CONNECT ON DATABASE shop TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- Role inheritance
GRANT readonly TO app_user;

-- Revoke
REVOKE INSERT, UPDATE, DELETE ON orders FROM app_user;

-- Row Level Security (RLS)
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- Policy: users see only their own posts
CREATE POLICY user_posts_policy ON posts
    USING (user_id = current_setting('app.current_user_id')::BIGINT);

-- Policy with insert check
CREATE POLICY insert_own_posts ON posts
    FOR INSERT
    WITH CHECK (user_id = current_setting('app.current_user_id')::BIGINT);

-- Set context variable (in application)
SET app.current_user_id = '42';
SELECT * FROM posts;  -- only user 42's posts

-- List privileges
\dp tablename
SELECT * FROM information_schema.role_table_grants WHERE table_name = 'users';
```

---

## 22. Performance Tuning

### EXPLAIN & EXPLAIN ANALYZE
```sql
-- Show query plan
EXPLAIN SELECT * FROM orders WHERE user_id = 1;

-- Show actual execution stats
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM orders WHERE user_id = 1;

-- Key plan nodes:
-- Seq Scan      — full table scan (bad on large tables)
-- Index Scan    — uses index
-- Index Only Scan — reads only index (fastest)
-- Nested Loop   — join method for small rows
-- Hash Join     — join method for larger sets
-- Merge Join    — join method for sorted inputs
```

### Query Optimization Tips
```sql
-- Avoid SELECT *
SELECT id, email FROM users;  -- only needed columns

-- Avoid functions on indexed columns in WHERE
-- BAD:
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- GOOD (with expression index):
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- Use LIMIT when you need only a few rows
SELECT * FROM orders ORDER BY created_at DESC LIMIT 1;

-- Use EXISTS instead of COUNT for presence check
-- BAD:
SELECT COUNT(*) > 0 FROM orders WHERE user_id = 1;
-- GOOD:
SELECT EXISTS(SELECT 1 FROM orders WHERE user_id = 1);

-- Avoid DISTINCT if GROUP BY works
SELECT user_id FROM orders GROUP BY user_id;

-- Use covering index for index-only scans
CREATE INDEX idx_orders_covering ON orders(user_id, created_at, total);
```

### Important Config Parameters
```sql
-- View current settings
SHOW shared_buffers;       -- cache size (set to 25% RAM)
SHOW work_mem;             -- per-sort/hash memory (256MB for analytics)
SHOW max_connections;      -- limit connections, use pgBouncer instead
SHOW effective_cache_size; -- hint to planner (50-75% RAM)
SHOW checkpoint_completion_target;
SHOW wal_buffers;

-- Check table/index bloat
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Autovacuum stats
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- VACUUM and ANALYZE
VACUUM ANALYZE users;
VACUUM FULL orders;  -- reclaims disk space (locks table)
```

### pg_stat_statements (enable in postgresql.conf)
```sql
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
CREATE EXTENSION pg_stat_statements;

-- Top slow queries
SELECT query, calls, total_exec_time/calls AS avg_ms,
       rows/calls AS avg_rows
FROM pg_stat_statements
ORDER BY avg_ms DESC
LIMIT 20;
```

---

## 23. Replication & High Availability

### Streaming Replication (overview)

**Primary** `postgresql.conf`:
```ini
wal_level = replica
max_wal_senders = 5
hot_standby = on
```

**Primary** `pg_hba.conf`:
```
host replication replicator 192.168.1.0/24 md5
```

**Standby setup:**
```bash
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/data -P -R
# -R creates standby.signal and recovery config automatically
```

### Logical Replication
```sql
-- On publisher
CREATE PUBLICATION my_pub FOR TABLE users, orders;
-- Or all tables:
CREATE PUBLICATION all_tables FOR ALL TABLES;

-- On subscriber
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=primary dbname=shop user=replicator password=pass'
    PUBLICATION my_pub;

-- Status
SELECT * FROM pg_stat_replication;     -- on primary
SELECT * FROM pg_stat_subscription;    -- on subscriber
```

---

## 24. Extensions

```sql
-- List available extensions
SELECT * FROM pg_available_extensions ORDER BY name;

-- Commonly used extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";      -- UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";       -- crypto functions
CREATE EXTENSION IF NOT EXISTS "hstore";         -- key-value pairs
CREATE EXTENSION IF NOT EXISTS "pg_trgm";        -- trigram similarity
CREATE EXTENSION IF NOT EXISTS "btree_gist";     -- GiST on scalar types
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements"; -- query stats
CREATE EXTENSION IF NOT EXISTS "postgis";        -- spatial / GIS data
CREATE EXTENSION IF NOT EXISTS "timescaledb";    -- time-series

-- UUID examples
CREATE TABLE sessions (
    id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,  -- pg 13+ (pgcrypto)
    -- or:
    -- id      UUID DEFAULT uuid_generate_v4() PRIMARY KEY, -- uuid-ossp
    user_id    BIGINT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- pg_trgm — fuzzy string matching
CREATE INDEX idx_users_username_trgm ON users USING GIN(username gin_trgm_ops);

SELECT username, similarity(username, 'alce') AS sim
FROM users
WHERE username % 'alce'   -- similarity threshold (default 0.3)
ORDER BY sim DESC;
```

---

## 25. Expert Patterns

### Advisory Locks — application-level locking
```sql
-- Session-level lock (released on disconnect)
SELECT pg_try_advisory_lock(12345);   -- returns bool
SELECT pg_advisory_lock(12345);       -- blocks until acquired
SELECT pg_advisory_unlock(12345);

-- Transaction-level lock
SELECT pg_try_advisory_xact_lock(12345);  -- auto-released at COMMIT/ROLLBACK
```

### SKIP LOCKED — queue processing
```sql
-- Worker picks next available job (no blocking on locked rows)
BEGIN;
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- Process job, then:
UPDATE jobs SET status = 'done' WHERE id = <picked_id>;
COMMIT;
```

### Generated Columns
```sql
CREATE TABLE rectangles (
    width  NUMERIC NOT NULL,
    height NUMERIC NOT NULL,
    area   NUMERIC GENERATED ALWAYS AS (width * height) STORED
);
```

### Table Inheritance
```sql
CREATE TABLE vehicles (id BIGSERIAL, make TEXT, model TEXT);
CREATE TABLE cars   (doors INTEGER) INHERITS (vehicles);
CREATE TABLE trucks (payload_tons NUMERIC) INHERITS (vehicles);

-- Query all vehicles including subclasses
SELECT * FROM vehicles;
-- Query only base table
SELECT * FROM ONLY vehicles;
```

### COPY — bulk import/export (fastest method)
```sql
-- Export to CSV
COPY users TO '/tmp/users.csv' WITH (FORMAT csv, HEADER true, DELIMITER ',');

-- Import from CSV
COPY users (email, username) FROM '/tmp/users.csv'
WITH (FORMAT csv, HEADER true, DELIMITER ',');

-- From/to stdin (useful in scripts)
\copy users TO 'users.csv' CSV HEADER   -- psql client-side
```

### Useful System Catalog Queries
```sql
-- Table sizes
SELECT relname, pg_size_pretty(pg_relation_size(oid)) AS table_size
FROM pg_class WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace
ORDER BY pg_relation_size(oid) DESC;

-- Active queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Kill a query
SELECT pg_cancel_backend(pid);    -- graceful cancel
SELECT pg_terminate_backend(pid); -- force terminate

-- Locks
SELECT pid, locktype, relation::regclass, mode, granted
FROM pg_locks l JOIN pg_stat_activity a USING (pid)
WHERE NOT granted;

-- Table dependencies
SELECT dependent_ns.nspname AS dep_schema,
       dependent_view.relname AS dep_view,
       source_ns.nspname AS source_schema,
       source_table.relname AS source_table
FROM pg_depend JOIN pg_rewrite ON pg_depend.objid = pg_rewrite.oid
JOIN pg_class AS dependent_view ON pg_rewrite.ev_class = dependent_view.oid
JOIN pg_class AS source_table ON pg_depend.refobjid = source_table.oid
JOIN pg_namespace dependent_ns ON dependent_ns.oid = dependent_view.relnamespace
JOIN pg_namespace source_ns ON source_ns.oid = source_table.relnamespace
WHERE NOT (dependent_view.oid = pg_depend.refobjid)
  AND source_table.relname = 'users';
```

### Connection Pooling — PgBouncer config
```ini
; pgbouncer.ini
[databases]
shop = host=127.0.0.1 port=5432 dbname=shop

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
pool_mode = transaction       ; transaction pooling (most efficient)
max_client_conn = 1000
default_pool_size = 20
```

---

## Quick Reference Cheat Sheet

```sql
-- DDL
CREATE TABLE t (...);   ALTER TABLE t ADD COLUMN c TYPE;   DROP TABLE t;

-- DML
INSERT INTO t VALUES (...) RETURNING id;
UPDATE t SET col = val WHERE ...;
DELETE FROM t WHERE ...;
SELECT col FROM t WHERE ... ORDER BY ... LIMIT n OFFSET m;

-- Joins: INNER, LEFT, RIGHT, FULL, CROSS, SELF
-- Aggregates: COUNT, SUM, AVG, MIN, MAX + GROUP BY + HAVING

-- Indexes: B-Tree, GIN, GiST, BRIN, Hash
-- Transactions: BEGIN / COMMIT / ROLLBACK / SAVEPOINT

-- Window: ROW_NUMBER, RANK, LAG, LEAD, SUM OVER, PARTITION BY
-- CTE: WITH name AS (...) SELECT ...
-- Recursive CTE: WITH RECURSIVE name AS (base UNION ALL recursive)

-- JSONB operators: -> ->> #> #>> @> ? ?| ?&
-- FTS: to_tsvector, to_tsquery, @@, ts_rank, ts_headline

-- Maintenance
VACUUM ANALYZE tablename;
REINDEX INDEX CONCURRENTLY idx_name;
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

---

*PostgreSQL version: 16.x | Last updated: 2026-05*
