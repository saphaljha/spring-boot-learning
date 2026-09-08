 DDL and DML operations
 Constraints, primary keys, foreign keys, and indexes
 Joins (INNER, LEFT, RIGHT, FULL), Subqueries, Aggregate functions, Group By
 Database normalization (1NF to 3NF) vs. denormalization
 ACID transactions and isolation levels (READ COMMITTED, REPEATABLE READ, SERIALIZABLE)
----------------------------

# Relational Database Design & SQL for Enterprise Java

A comprehensive deep dive into database design, normalization, complex SQL operations, ACID transactions, and indexing strategies for backend Java/Spring Boot services.

---

## 1. Relational Schema Design: DDL, DML & Constraints

Relational schema design balances integrity, query performance, and domain boundaries.

```sql
-- DDL: Defining Table Structure with Constraints
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(150) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    order_number VARCHAR(64) NOT NULL UNIQUE,
    total_amount NUMERIC(12, 2) NOT NULL CHECK (total_amount >= 0),
    status VARCHAR(32) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    sku VARCHAR(64) NOT NULL,
    unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price > 0),
    quantity INT NOT NULL CHECK (quantity > 0),
    CONSTRAINT fk_items_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);
```

### Core Constraints
* `PRIMARY KEY`: Guarantees unique entity identity and automatically creates a clustered or unique B-tree index.
* `FOREIGN KEY`: Enforces referential integrity across entity boundaries (`RESTRICT`, `CASCADE`, `SET NULL`).
* `UNIQUE`: Enforces column-level or composite-level non-duplication.
* `CHECK`: Guarantees domain constraints directly at the database engine level (e.g., `quantity > 0`).

---

## 2. SQL Joins, Aggregations & Grouping

Understanding join semantics and group reductions is essential for writing efficient JPQL, native queries, and analytics reports.

```
       Table A (Left)               Table B (Right)
     +-----------------+          +-----------------+
     | id | name       |          | id | a_id | val |
     +-----------------+          +-----------------+
              \                            /
               \      INNER JOIN          /
                +------------------------+
                | Matches only in BOTH   |
                +------------------------+
              /                                         /         LEFT JOIN                 +-----------------------+               +-----------------------+
     | All rows from Table A |               | All rows from Table B |
     | + matching Table B    |               | + matching Table A    |
     +-----------------------+               +-----------------------+
```

### Analytical Query Example

```sql
-- Aggregate order totals, counts, and average order values grouped by active customer
SELECT 
    c.id AS customer_id,
    c.full_name,
    COUNT(o.id) AS total_orders,
    COALESCE(SUM(o.total_amount), 0.00) AS lifetime_value,
    ROUND(AVG(o.total_amount), 2) AS average_order_value
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.status != 'CANCELLED'
WHERE c.status = 'ACTIVE'
GROUP BY c.id, c.full_name
HAVING COUNT(o.id) >= 1
ORDER BY lifetime_value DESC;
```

---

## 3. Database Normalization (1NF to 3NF) vs. Denormalization

| Form | Rule | Target Problem Solved |
| :--- | :--- | :--- |
| **1NF (First Normal Form)** | Atomic values (no arrays/delimited values in columns) and a unique primary key. | Eliminates repeating groups and unstructured columns. |
| **2NF (Second Normal Form)** | Must be in 1NF; all non-key columns must be fully functionally dependent on the whole primary key. | Eliminates partial dependencies in composite keys. |
| **3NF (Third Normal Form)** | Must be in 2NF; non-key columns must not depend on other non-key columns (no transitive dependencies). | Eliminates transitive dependencies and update anomalies. |

### When to Denormalize
* **Read Heavy Systems:** Pre-computing summary values (e.g., storing `total_amount` directly in `orders` rather than summing `order_items` on every dashboard render).
* **Historical Immutability:** Storing a snapshot of the shipping address inside the `orders` table so subsequent customer address changes don't alter past receipts.

---

## 4. ACID Transactions & Isolation Levels

Transactions guarantee atomic state transitions via ACID properties:
* **Atomicity:** All statements commit together or roll back completely.
* **Consistency:** The database transitions from one valid state to another, satisfying all constraints.
* **Isolation:** Concurrent transactions execute without cross-talk or race anomalies.
* **Durability:** Committed transactions persist even through hardware crashes.

### Concurrency Anomalies & Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
| :--- | :---: | :---: | :---: |
| **READ UNCOMMITTED** | Possible | Possible | Possible |
| **READ COMMITTED** *(Postgres default)* | Prevented | Possible | Possible |
| **REPEATABLE READ** *(MySQL default)* | Prevented | Prevented | Possible (or prevented via MVCC) |
| **SERIALIZABLE** | Prevented | Prevented | Prevented |

* **Dirty Read:** Reading uncommitted mutations from another transaction that later rolls back.
* **Non-Repeatable Read:** Re-reading the same row within a transaction and seeing modified column values from another committed transaction.
* **Phantom Read:** Re-running a range query (`WHERE amount > 50`) and discovering newly inserted rows committed by another transaction.

---

## 5. Indexing Mechanics & Query Optimization

Indexes are data structures (most commonly **B-Trees**) that allow the database engine to locate target rows in $O(\log n)$ disk reads instead of scanning the full table ($O(n)$).

### Index Strategies

```sql
-- 1. Single Column Index: Speeds up direct lookups
CREATE INDEX idx_customers_email ON customers(email);

-- 2. Composite (Multi-Column) Index: Follows the Leftmost Prefix Rule
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);

-- 3. Partial (Filtered) Index: Saves memory and index maintenance overhead
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'PENDING';
```

### The Leftmost Prefix Rule for Composite Indexes
An index on `(customer_id, status)` can be used for queries filtering by:
* `WHERE customer_id = ?` (Supported)
* `WHERE customer_id = ? AND status = ?` (Supported)
* `WHERE status = ?` (**Not supported**; misses leftmost column)

### Inspecting Query Plans

Always inspect execution plans before deploying queries into production:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE customer_id = 42 AND status = 'PENDING';
```
Look for `Index Scan` / `Bitmap Index Scan` instead of costly `Seq Scan` (Sequential Table Scan).

---

## 6. Key Takeaways for Magento / PHP Developers

1. **EAV vs. Normalized Relational Models:** Magento models dynamic attributes via Entity-Attribute-Value (EAV) tables (`catalog_product_entity_varchar`, `catalog_product_entity_int`), which requires numerous complex SQL joins for a single product load. In modern Spring Boot microservices, schemas are designed normalized, clean, and explicit—or leverage native `jsonb` column indexing for dynamic fields.
2. **Schema Control:** In Magento, database mutations were controlled via declarative XML (`db_schema.xml`) or setup patches. In the Java ecosystem, schema versioning is managed explicitly through versioned SQL scripts with tools like **Flyway** or **Liquibase**.
