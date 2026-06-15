# The Complete Guide to Database Design

*From first principles to production-grade systems.*

---

## Table of Contents

1. [How to Think About Database Design](#1-how-to-think-about-database-design)
2. [Conceptual Design: Entities, Attributes, Relationships](#2-conceptual-design-entities-attributes-relationships)
3. [From Real-World Problem to Schema](#3-from-real-world-problem-to-schema)
4. [Keys: Primary, Foreign, Composite, Surrogate](#4-keys-primary-foreign-composite-surrogate)
5. [Normalization (1NF → BCNF)](#5-normalization-1nf--bcnf)
6. [Constraints & Data Integrity](#6-constraints--data-integrity)
7. [Indexing](#7-indexing)
8. [Denormalization & Trade-offs](#8-denormalization--trade-offs)
9. [Query Optimization](#9-query-optimization)
10. [Transactions & ACID](#10-transactions--acid)
11. [Concurrency Control](#11-concurrency-control)
12. [Scaling Strategies](#12-scaling-strategies)
13. [SQL vs NoSQL: Choosing Wisely](#13-sql-vs-nosql-choosing-wisely)
14. [Schema Design Patterns for Real Systems](#14-schema-design-patterns-for-real-systems)
15. [Security, Maintainability, Performance](#15-security-maintainability-performance)
16. [Common Mistakes to Avoid](#16-common-mistakes-to-avoid)
17. [A Decision-Making Cheat Sheet](#17-a-decision-making-cheat-sheet)

---

## 1. How to Think About Database Design

Database design happens in three layers, and confusing them is the #1 source of bad schemas:

| Layer | Question it answers | Output |
|-------|--------------------|--------|
| **Conceptual** | What things exist in this domain and how do they relate? | Entity-Relationship (ER) model |
| **Logical** | How do those things map to tables, columns, keys? | Normalized relational schema |
| **Physical** | How is this stored, indexed, partitioned for performance? | Indexes, partitions, storage engine config |

**Golden rule:** Design conceptually first, *independent of any database product*. Only descend to physical concerns once the logical model is sound. Premature optimization at the physical layer (e.g., denormalizing before you have a query problem) is the most common rookie mistake.

---

## 2. Conceptual Design: Entities, Attributes, Relationships

### Identifying Entities
An **entity** is a thing the business cares about and needs to *remember*. A practical heuristic: scan the problem description and underline the **nouns**.

> "A *customer* places *orders*. Each order contains *products*. Products belong to *categories*."

Candidate entities: Customer, Order, Product, Category.

Filter the nouns:
- Keep nouns that have their own identity and attributes (`Customer`, `Order`).
- Discard nouns that are merely attributes of something else (`color` is an attribute of `Product`, not an entity).
- Watch for nouns that are actually *relationships* (`order line` connects Order and Product).

### Identifying Attributes
Attributes describe an entity. For each entity, ask "what do I need to know about it?"

- **Simple vs composite:** `address` may decompose into `street`, `city`, `postal_code`.
- **Single vs multi-valued:** a customer with multiple phone numbers signals a separate table.
- **Stored vs derived:** `age` (derived from `birth_date`) should usually *not* be stored — compute it.

### Identifying Relationships
Relationships are the **verbs** connecting entities ("places", "contains", "belongs to"). Each has a **cardinality**:

- **One-to-One (1:1)** — a User has one UserProfile. Often foldable into a single table.
- **One-to-Many (1:N)** — a Customer has many Orders. The most common type; implemented with a foreign key on the "many" side.
- **Many-to-Many (M:N)** — an Order contains many Products; a Product appears in many Orders. *Requires a junction (associative) table.*

### Example ER Sketch

```
Customer ──< places >── Order ──< contains >── OrderItem >── refers to ── Product
   1            N          1           N            N              1
                                            (OrderItem is the junction
                                             resolving Order ⋈ Product)
```

`OrderItem` exists because the M:N relationship between Order and Product carries its own data (quantity, unit price at time of sale).

---

## 3. From Real-World Problem to Schema

A repeatable 7-step process:

1. **Gather requirements.** List the queries and reports the system must support. *Design for how data is read, not just how it's stored.*
2. **Identify entities** (nouns with identity).
3. **Identify attributes** for each entity; mark candidate keys.
4. **Identify relationships** and their cardinalities.
5. **Resolve M:N** relationships into junction tables.
6. **Normalize** to remove redundancy (Section 5).
7. **Add physical design** — keys, indexes, constraints — and selectively denormalize where measured performance demands it.

### Worked Example: A Library

Requirements: track books, copies, members, and loans.

```sql
CREATE TABLE member (
    member_id    BIGINT PRIMARY KEY,
    full_name    VARCHAR(120) NOT NULL,
    email        VARCHAR(255) NOT NULL UNIQUE,
    joined_at    TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE book (
    book_id      BIGINT PRIMARY KEY,
    isbn         CHAR(13) NOT NULL UNIQUE,
    title        VARCHAR(255) NOT NULL,
    author       VARCHAR(255) NOT NULL
);

-- One physical copy of a book; a book has many copies (1:N)
CREATE TABLE copy (
    copy_id      BIGINT PRIMARY KEY,
    book_id      BIGINT NOT NULL REFERENCES book(book_id),
    shelf_code   VARCHAR(20) NOT NULL
);

-- A loan connects a member and a copy (M:N over time, resolved here)
CREATE TABLE loan (
    loan_id      BIGINT PRIMARY KEY,
    copy_id      BIGINT NOT NULL REFERENCES copy(copy_id),
    member_id    BIGINT NOT NULL REFERENCES member(member_id),
    loaned_at    TIMESTAMP NOT NULL DEFAULT now(),
    due_at       TIMESTAMP NOT NULL,
    returned_at  TIMESTAMP            -- NULL = still out
);
```

Notice `book` (the title/work) is separated from `copy` (a physical item). Conflating them would force you to duplicate title/author for every physical copy — a normalization violation.

---

## 4. Keys: Primary, Foreign, Composite, Surrogate

- **Candidate key** — any column(s) that uniquely identify a row.
- **Primary key (PK)** — the chosen candidate key. Must be unique and non-null.
- **Foreign key (FK)** — a column referencing a PK in another table; enforces referential integrity.
- **Composite key** — a key made of multiple columns (common in junction tables).
- **Surrogate vs natural key:**
  - *Natural key* — derived from real data (e.g., `email`, `isbn`).
  - *Surrogate key* — system-generated, meaningless (e.g., auto-increment `id`, UUID).

**Practical guidance:** Prefer surrogate keys for primary keys (stable, compact, never change), but still enforce natural uniqueness with a `UNIQUE` constraint. Natural keys make poor PKs because real-world identifiers change (people change emails, products get re-coded).

**UUID vs auto-increment integer:**
- Auto-increment: small, fast, index-friendly, but leaks row counts and is awkward for distributed systems.
- UUID: globally unique, good for sharding/offline generation, but larger and can fragment B-tree indexes (use UUIDv7 or ULID for time-ordered locality).

---

## 5. Normalization (1NF → BCNF)

Normalization eliminates redundancy and update anomalies by organizing columns according to their functional dependencies.

**Start with an unnormalized mess:**

| order_id | customer | customer_email | products |
|----------|----------|----------------|----------|
| 1 | Alice | a@x.com | "Pen, Notebook" |

### First Normal Form (1NF)
*Every cell holds a single atomic value; no repeating groups.*

The `products` column packs multiple values — violation. Split into rows:

| order_id | customer | customer_email | product |
|----------|----------|----------------|---------|
| 1 | Alice | a@x.com | Pen |
| 1 | Alice | a@x.com | Notebook |

### Second Normal Form (2NF)
*Be in 1NF, and every non-key attribute depends on the **whole** primary key (no partial dependency).*

If PK is `(order_id, product)`, then `customer` and `customer_email` depend only on `order_id` — a partial dependency. Split:

```
orders(order_id PK, customer, customer_email)
order_lines(order_id, product)  -- PK (order_id, product)
```

### Third Normal Form (3NF)
*Be in 2NF, and no non-key attribute depends on another non-key attribute (no transitive dependency).*

`customer_email` depends on `customer`, not directly on `order_id`. Extract customers:

```
customers(customer_id PK, name, email)
orders(order_id PK, customer_id FK)
order_lines(order_id, product_id)
```

**3NF in one sentence:** *Every non-key attribute depends on the key, the whole key, and nothing but the key.*

### Boyce-Codd Normal Form (BCNF)
*A stricter 3NF: for every functional dependency X → Y, X must be a superkey.*

BCNF catches edge cases 3NF misses when there are overlapping candidate keys. Example: a table `(student, course, instructor)` where each instructor teaches exactly one course, and each course can have multiple instructors. The dependency `instructor → course` has a non-superkey determinant — a BCNF violation. Decompose into `(student, instructor)` and `(instructor, course)`.

**How far should you normalize?** 3NF is the practical default for OLTP systems. BCNF when you have the overlapping-key scenario. Beyond (4NF/5NF) is rarely needed outside specialized cases.

---

## 6. Constraints & Data Integrity

Integrity rules belong in the database — application code is not a reliable last line of defense.

| Constraint | Purpose | Example |
|-----------|---------|---------|
| `NOT NULL` | Value required | `email VARCHAR NOT NULL` |
| `UNIQUE` | No duplicates | `UNIQUE(email)` |
| `PRIMARY KEY` | Unique + not null identifier | `PRIMARY KEY(id)` |
| `FOREIGN KEY` | Referential integrity | `REFERENCES customers(id)` |
| `CHECK` | Domain rule | `CHECK (price >= 0)` |
| `DEFAULT` | Fallback value | `DEFAULT now()` |

**Types of integrity:**
- **Entity integrity** — every row uniquely identifiable (PK enforces this).
- **Referential integrity** — FKs always point to existing rows. Control deletes with `ON DELETE CASCADE` / `RESTRICT` / `SET NULL`.
- **Domain integrity** — values fall within valid ranges/types (`CHECK`, enums).

```sql
CREATE TABLE product (
    product_id  BIGINT PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    stock       INT NOT NULL DEFAULT 0 CHECK (stock >= 0),
    status      VARCHAR(20) NOT NULL DEFAULT 'active'
                CHECK (status IN ('active','discontinued','draft'))
);
```

---

## 7. Indexing

An index is a sorted data structure (usually a B-tree) that lets the engine find rows without scanning the whole table — trading write speed and disk for read speed.

### What to index
- Columns in `WHERE`, `JOIN`, and `ORDER BY` clauses.
- Foreign keys (almost always — joins and cascade checks use them).
- High-selectivity columns (many distinct values). Indexing a boolean `is_active` rarely helps.

### Key concepts
- **Composite index** — `(a, b, c)` serves queries filtering on `a`, `a,b`, or `a,b,c` (leftmost-prefix rule), but *not* on `b` alone.
- **Covering index** — includes all columns a query needs, so the engine never touches the table.
- **Clustered vs non-clustered** — a clustered index determines physical row order (one per table); non-clustered indexes point to rows.
- **Partial / filtered index** — index only rows matching a condition (`WHERE status = 'active'`).

### Costs
Every index slows `INSERT`/`UPDATE`/`DELETE` and consumes storage. Don't index blindly — measure with `EXPLAIN` and add indexes to fix observed slow queries.

```sql
-- Speeds up "find a customer's recent orders"
CREATE INDEX idx_orders_customer_date
    ON orders (customer_id, created_at DESC);
```

---

## 8. Denormalization & Trade-offs

Denormalization deliberately reintroduces redundancy to avoid expensive joins or aggregations.

**When it's justified:**
- Read-heavy workloads where the same join runs constantly.
- Aggregations too costly to compute on the fly (e.g., a cached `comment_count`).
- Reporting/analytics tables (star schemas in data warehouses).

**The cost:** every redundant copy must be kept in sync. A stored `order_total` is wrong the moment a line item changes unless you maintain it (triggers, application logic, or scheduled recomputation).

**Common patterns:**
- **Precomputed aggregates** — store `post.like_count` instead of `COUNT(*)`.
- **Materialized views** — cache a query result, refreshed periodically.
- **Redundant columns** — copy `customer_name` into `orders` to avoid a join on a hot path.

**Rule:** Normalize first. Denormalize only with evidence (profiling shows a real bottleneck), and document every denormalization plus its sync mechanism.

---

## 9. Query Optimization

### Read the query plan first
`EXPLAIN` (and `EXPLAIN ANALYZE` for real timings) is the single most important optimization tool. Look for:
- **Sequential scans** on large tables in selective queries → missing index.
- **Nested loop joins** over big row counts → maybe need a hash join or better index.
- **Row estimate vs actual** mismatches → stale statistics (`ANALYZE` the table).

### Common techniques
- Add the right index (Section 7); ensure the query is *sargable* (avoid wrapping indexed columns in functions: `WHERE date(created_at) = ...` defeats the index).
- Select only needed columns — avoid `SELECT *`.
- Replace correlated subqueries with joins or window functions where clearer/faster.
- Paginate with **keyset pagination** (`WHERE id > :last_id LIMIT 20`) instead of large `OFFSET`, which scans and discards rows.
- Batch writes instead of row-by-row round trips.
- Keep statistics fresh so the planner makes good choices.

### Anti-patterns
- **N+1 queries** — one query per row in a loop. Fix with a join or `IN (...)` / eager loading.
- Over-indexing that slows writes more than it helps reads.

---

## 10. Transactions & ACID

A **transaction** groups operations so they succeed or fail as a unit. ACID defines its guarantees:

- **Atomicity** — all operations commit, or none do. A transfer that debits one account *must* credit the other or roll back entirely.
- **Consistency** — a transaction moves the DB from one valid state to another, respecting all constraints.
- **Isolation** — concurrent transactions don't corrupt each other; results match some serial order (configurable, see below).
- **Durability** — once committed, data survives crashes (written to durable storage / write-ahead log).

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;   -- both succeed, or ROLLBACK undoes everything
```

### Isolation levels & their anomalies

| Level | Dirty read | Non-repeatable read | Phantom read |
|-------|:---------:|:-------------------:|:------------:|
| Read Uncommitted | allowed | allowed | allowed |
| Read Committed | prevented | allowed | allowed |
| Repeatable Read | prevented | prevented | allowed* |
| Serializable | prevented | prevented | prevented |

*Higher isolation = more correctness, less concurrency. Most systems default to **Read Committed**; raise it where correctness demands.*

---

## 11. Concurrency Control

When many transactions run at once, the engine must prevent interference. Two broad strategies:

### Pessimistic concurrency (locking)
Acquire locks before accessing data; others wait.
- **Shared (read) locks** vs **exclusive (write) locks**.
- Granularity: row, page, or table locks.
- **Risk: deadlocks** — two transactions each holding a lock the other needs. Engines detect and abort one; reduce risk by acquiring locks in a consistent order and keeping transactions short.

```sql
SELECT * FROM inventory WHERE sku = 'ABC'
FOR UPDATE;   -- locks the row until commit
```

### Optimistic concurrency
Assume conflicts are rare; check at commit time. Common pattern: a `version` column.

```sql
UPDATE inventory SET stock = stock - 1, version = version + 1
WHERE sku = 'ABC' AND version = :expected_version;
-- 0 rows affected → someone else changed it → retry
```

Optimistic suits low-contention, read-heavy workloads; pessimistic suits high-contention hot rows. **MVCC** (Multi-Version Concurrency Control), used by PostgreSQL and others, lets readers see a consistent snapshot without blocking writers — the best of both for most read workloads.

---

## 12. Scaling Strategies

### Vertical scaling (scale up)
Add more CPU/RAM/faster disks to one machine. Simple, no app changes, but bounded by hardware limits and a single point of failure. Always the first lever — exhaust it before complexity.

### Horizontal scaling (scale out)
Distribute load across many machines. More complex but effectively unbounded.

#### Replication
Copy data to multiple nodes.
- **Primary-replica (leader-follower):** writes go to the primary, reads fan out to replicas. Great for read-heavy workloads. Beware **replication lag** — replicas may be slightly stale (eventual consistency for reads).
- **Multi-primary:** writes accepted on multiple nodes; powerful but introduces write-conflict resolution complexity.
- Benefits: read scaling, high availability, failover.

#### Sharding (horizontal partitioning)
Split data across nodes by a **shard key**, each node holding a subset.
- **Range sharding** — by value ranges (e.g., A–M, N–Z). Risk: hotspots.
- **Hash sharding** — hash the key for even distribution. Even spread, but range queries become expensive.
- **Directory/lookup sharding** — a lookup table maps keys to shards. Flexible, but the directory is a dependency.

**Choosing a shard key is the highest-stakes decision:** it should distribute load evenly and keep related data (queried together) on the same shard. A bad shard key causes hotspots and cross-shard joins (slow, complex). Cross-shard transactions are hard — design to keep transactions within a single shard.

#### Partitioning (within one DB)
Splitting one big table into segments (by date range, list, or hash) on the same server — improves manageability and can prune scans, distinct from sharding across servers.

### The CAP theorem
In a network partition, a distributed system can guarantee only two of **Consistency, Availability, Partition-tolerance** — and partitions *will* happen, so the real choice is **CP** (refuse some requests to stay consistent) vs **AP** (stay available, accept temporary inconsistency).

---

## 13. SQL vs NoSQL: Choosing Wisely

### SQL (relational) — Postgres, MySQL, SQL Server
- **Strengths:** strong consistency (ACID), rich querying & joins, enforced schema/constraints, mature tooling.
- **Best for:** transactional systems (finance, orders, inventory), complex relationships, ad-hoc analytics, anything where correctness and integrity matter most.

### NoSQL — a family, not one thing

| Type | Examples | Model | Best for |
|------|----------|-------|----------|
| **Document** | MongoDB, Couchbase | JSON-like docs | Flexible/evolving schemas, content, catalogs |
| **Key-Value** | Redis, DynamoDB | simple key→value | Caching, sessions, ultra-fast lookups |
| **Wide-Column** | Cassandra, HBase | column families | Massive write throughput, time-series, IoT |
| **Graph** | Neo4j | nodes & edges | Highly connected data: social, fraud, recommendations |

### Decision factors
- **Data structure:** highly relational with many joins → SQL. Self-contained documents → document store. Connection-centric → graph.
- **Consistency needs:** strict transactional correctness → SQL/CP. Tolerant of eventual consistency for scale → AP NoSQL.
- **Scale & write volume:** extreme horizontal write scale → wide-column. Moderate → SQL scales further than people assume.
- **Schema volatility:** rapidly changing, sparse fields → document. Stable, validated → relational.

**Pragmatic truth:** start with a well-designed relational database unless you have a *specific* reason not to. It handles the vast majority of applications, and "polyglot persistence" (SQL for core data + Redis for cache + a search engine) is common and healthy. Don't pick NoSQL to avoid learning schema design — you still need it, just enforced in your app instead of the DB.

---

## 14. Schema Design Patterns for Real Systems

### E-commerce

Core entities: `customer`, `product`, `category`, `cart`, `order`, `order_item`, `payment`, `inventory`.

Key decisions:
- **Snapshot prices and product details into `order_item`** at purchase time. The order record must reflect what was actually bought, even if the product's price later changes.
- Separate `order` (header: customer, status, totals, address) from `order_item` (lines).
- Model order status as a state machine (`pending → paid → shipped → delivered / cancelled`); consider an `order_status_history` table for audit.
- Inventory updates need transactional safety to prevent overselling (see below).

```sql
CREATE TABLE order_item (
    order_id        BIGINT NOT NULL REFERENCES orders(order_id),
    product_id      BIGINT NOT NULL REFERENCES product(product_id),
    quantity        INT NOT NULL CHECK (quantity > 0),
    unit_price      NUMERIC(10,2) NOT NULL,   -- snapshot, not a join
    product_name    VARCHAR(255) NOT NULL,    -- snapshot
    PRIMARY KEY (order_id, product_id)
);
```

### Inventory management

The classic challenge: **preventing overselling** under concurrency.

```sql
-- Atomic decrement that refuses to go negative
UPDATE inventory
SET stock = stock - :qty
WHERE product_id = :pid AND stock >= :qty;
-- 0 rows affected → insufficient stock → reject the order
```

Patterns: reserved/available stock split (`stock_on_hand`, `stock_reserved`), and an `inventory_movement` ledger table (append-only record of every in/out) so you can audit and reconstruct stock levels.

### Multi-tenant applications

Three approaches, with a clear trade-off curve:

| Model | Isolation | Cost/scale | Complexity |
|-------|-----------|-----------|------------|
| **Shared schema** (a `tenant_id` column on every table) | Lowest | Best (one DB) | Must filter every query |
| **Schema per tenant** (one DB, many schemas) | Medium | Medium | Migrations × N schemas |
| **Database per tenant** | Highest | Lowest | Heaviest ops |

Shared-schema is the common default for SaaS at scale. Critical safeguards:
- A `tenant_id` on every tenant-owned table, in every index and every `WHERE` clause.
- Enforce isolation at the database layer with **Row-Level Security** so a forgotten filter can't leak data across tenants.

```sql
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON invoices
    USING (tenant_id = current_setting('app.tenant_id')::bigint);
```

---

## 15. Security, Maintainability, Performance

### Security
- **Least privilege:** app accounts get only the permissions they need; never connect as a superuser.
- **Prevent SQL injection:** always use parameterized queries / prepared statements — never string-concatenate user input.
- **Encrypt** at rest (disk/tablespace encryption) and in transit (TLS).
- **Hash passwords** with bcrypt/argon2; never store plaintext or reversible encryption for credentials.
- **Audit logging** for sensitive tables; restrict and monitor access to PII.
- **Data minimization & retention:** don't store what you don't need; honor regulations (GDPR, etc.).

### Maintainability
- **Version-control your schema** with migrations (Flyway, Liquibase, Alembic, Rails/Django migrations). Migrations should be forward-only and reviewed like code.
- **Consistent naming conventions** (e.g., snake_case, singular vs plural — pick one). Name FKs and indexes predictably.
- **Document** the schema and the *reasons* behind non-obvious choices (especially denormalizations).
- Avoid storing business logic only in stored procedures unless your team can maintain it; keep it discoverable.

### Performance (operational)
- **Connection pooling** — opening connections is expensive; pool them (PgBouncer, app-side pools).
- **Monitor** slow query logs, cache hit ratios, index usage, replication lag.
- **Right data types** — smaller types = faster (don't use `BIGINT` where `INT` suffices, or `TEXT` for fixed short codes).
- **Archive cold data** out of hot tables to keep them small and fast.

---

## 16. Common Mistakes to Avoid

1. **Designing tables before understanding the queries.** Read patterns should shape the schema.
2. **Using natural keys as primary keys** when they can change. Prefer surrogate PKs.
3. **Storing comma-separated lists in a column** (violates 1NF). Use a related table.
4. **No foreign key constraints** ("we'll enforce it in the app"). The app *will* eventually let bad data through.
5. **`SELECT *` everywhere** and N+1 query loops.
6. **Premature denormalization / premature sharding** before measuring the real bottleneck.
7. **Indexing everything** (kills write performance) or **indexing nothing** (kills reads).
8. **Storing money as `FLOAT`.** Use `DECIMAL`/`NUMERIC` — floats introduce rounding errors.
9. **Naive timestamps without timezone.** Store UTC (`TIMESTAMPTZ`); convert at the edges.
10. **Forgetting the `tenant_id` filter** in multi-tenant apps — a data-leak waiting to happen. Enforce with RLS.
11. **Letting transactions stay open too long**, holding locks and bloating concurrency contention.
12. **No backups / untested backups.** A backup you've never restored is a hope, not a backup.

---

## 17. A Decision-Making Cheat Sheet

**Normalize or denormalize?**
→ Default to 3NF. Denormalize only after profiling proves a join/aggregate is the bottleneck, and only with a documented sync mechanism.

**Surrogate or natural primary key?**
→ Surrogate PK + a `UNIQUE` constraint on the natural key.

**Add an index?**
→ If a column is frequently filtered/joined/sorted *and* has good selectivity, and `EXPLAIN` shows a scan. Verify writes aren't hurt.

**SQL or NoSQL?**
→ SQL unless you have a concrete reason (extreme write scale, deeply connected graph data, genuinely schemaless documents, or specific latency needs a key-value store solves).

**Scale up or out?**
→ Scale up first (simplest). Then add read replicas. Shard only when a single primary's writes are the bottleneck and you've optimized everything else.

**Which isolation level?**
→ Read Committed by default; Serializable for operations where anomalies are unacceptable (financial postings, inventory decrements) — or use an atomic conditional update instead.

**Pessimistic or optimistic locking?**
→ Optimistic for low-contention, read-heavy data; pessimistic (`FOR UPDATE`) for hot rows with frequent conflicting writes.

---

### Final principle

> **Make it correct, then make it fast.** A normalized, constrained, well-indexed relational schema is the right starting point for almost every project. Add complexity — denormalization, NoSQL, sharding — only in response to a measured need, never in anticipation of an imagined one.
