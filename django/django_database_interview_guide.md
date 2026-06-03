# Django Database Mastery: Principal-Level Interview Preparation Guide

> **Audience:** Django developers with 3+ years of experience targeting Senior/Staff/Principal backend roles.
> **Scope:** Relationship optimization, query expressions, indexing, concurrency control, and zero-downtime migrations.
> **Philosophy:** Every abstraction the ORM offers is a contract with the underlying database engine. Senior engineers reason in *round-trips, locks, query plans, and write amplification* — not in Python sugar.

---

## Table of Contents

1. [Relationship Optimization: `select_related` vs `prefetch_related`](#1-relationship-optimization)
2. [Database Expressions: `F`, `Q`, `Subquery`, Conditional Annotations](#2-database-expressions)
3. [Advanced Indexing Strategies](#3-advanced-indexing-strategies)
4. [Concurrency Management: Transactions & Pessimistic Locking](#4-concurrency-management)
5. [Zero-Downtime Schema & Data Migrations at Scale](#5-zero-downtime-migrations)

---

## 1. Relationship Optimization

### 1.1 `select_related` — The JOIN Strategy

#### Architectural Intent
Without select_related, Django asks the database for a record (like an Order). Then, if you need the customer attached to that order, Django has to go back to the database a second time.

select_related solves this by telling the database: "Hey, while you're getting this Order, grab the Customer attached to it and stitch them together before sending it back." You make one round-trip instead of two (or hundreds).

**Why It Only Allows "Single" Relationships**
It strictly works for Foreign Keys (e.g., an Order has one Customer) and One-to-One fields.

- The Reason: Imagine an Order has 50 different items in it (a Many-to-Many relationship). If you tried to grab the Order and all 50 items using a standard database JOIN, the database would send back 50 rows. The Order's ID, total price, and date would be duplicated 50 times in memory—once for every item. This is called "row explosion." Because select_related only allows "single" relationships, this massive memory multiplication can't happen.

`select_related` resolves **forward `ForeignKey` and `OneToOneField`** relationships by emitting a **single SQL query containing an `INNER`/`LEFT OUTER JOIN`**. It works only for single-valued relationships because a JOIN multiplies rows; for a single-valued relationship the cardinality on the "one" side is bounded, so the row explosion is constrained. The mapping to the engine is direct: Django pushes the relational composition down to the database's join planner, which can leverage indexes on the join keys (`PK`/`FK`) and execute a hash/merge/nested-loop join far more efficiently than N application-level lookups. The cost you trade is **wider rows and potential duplicate parent data** transmitted over the wire.

#### Django ORM Implementation

```python
# models.py
from django.db import models


class Author(models.Model):
    name = models.CharField(max_length=255)
    country = models.CharField(max_length=64)


class Publisher(models.Model):
    name = models.CharField(max_length=255)


class Book(models.Model):
    title = models.CharField(max_length=255)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")
    publisher = models.ForeignKey(Publisher, on_delete=models.PROTECT, related_name="books")
    published_at = models.DateField()


# Without optimization — triggers N+1 queries
def list_books_naive():
    books = Book.objects.all()
    return [(b.title, b.author.name, b.publisher.name) for b in books]


# With select_related — single JOIN query
def list_books_optimized():
    books = Book.objects.select_related("author", "publisher").all()
    return [(b.title, b.author.name, b.publisher.name) for b in books]
```

#### Generated Raw SQL

```sql
-- list_books_optimized()
SELECT
    "app_book"."id",
    "app_book"."title",
    "app_book"."author_id",
    "app_book"."publisher_id",
    "app_book"."published_at",
    "app_author"."id",
    "app_author"."name",
    "app_author"."country",
    "app_publisher"."id",
    "app_publisher"."name"
FROM "app_book"
INNER JOIN "app_author"    ON ("app_book"."author_id"    = "app_author"."id")
INNER JOIN "app_publisher" ON ("app_book"."publisher_id" = "app_publisher"."id");
```

> **Engine note:** Both joins use `INNER JOIN` because `author` and `publisher` are non-nullable FKs. A nullable FK (`null=True`) produces a `LEFT OUTER JOIN` to preserve `Book` rows with `NULL` parents.

#### Interview Problem — Diagnosing N+1 on a Dashboard Endpoint

> *"A `/api/books/` endpoint returning 500 books takes 4 seconds. Your APM shows 1,001 queries. Diagnose and fix."*

**Diagnosis:** `1 + (500 × 2)` = 1,001 queries — the classic N+1 signature. The base query fetches 500 books; each `b.author.name` and `b.publisher.name` access lazily fires its own `SELECT`. At ~4ms network round-trip each, 1,000 extra queries ≈ 4s of pure latency, dominated by round-trip overhead, not query execution.

**Solution:**

```python
# 1 query instead of 1,001
books = (
    Book.objects
    .select_related("author", "publisher")
    .only("title", "author__name", "publisher__name")  # narrow the wire payload
)
```

Adding `.only()` restricts the SELECT projection so the wide JOIN doesn't drag unneeded columns across the network. **Result: 1,001 queries → 1 query.**

---

### 1.2 `prefetch_related` — The Separate-Query Strategy

#### Architectural Intent

1. **The Problem: The "Data Explosion"**
    Imagine you have an Author who has written 1,000 books.

    If you try to use the previous tool (`select_related`) to fetch the Author and their books in one trip, the database is forced to send you back a list of 1,000 rows. The problem? The Author's name, biography, and email address will be duplicated and downloaded in every single one of those 1,000 rows.

    This is called multiplicative row explosion. You are downloading the exact same parent data over and over again, which wastes massive amounts of memory and clogs up your network.

2. **The Solution: Two Smart Trips**

    Because "Many-to-Many" or "Reverse" relationships (like one Author to many Books) cause this explosion, Django uses `prefetch_related` to handle them differently.

    Instead of forcing one giant, bloated database trip, it makes exactly two highly efficient trips:

    - Trip 1: "Get me the Authors." (Downloads the authors perfectly, with no duplicates).

    - Trip 2: "Get me all the Books that belong to the Authors I just downloaded." (This is the IN (...) clause).

3. **The Assembly: Python Does the Sorting**

    Now your application has two neat, separate lists sitting in its memory: a list of Authors and a list of Books.

    Instead of forcing the database to connect them (which caused the explosion), Django uses Python to act like a mailroom sorting facility. It loops through the lists in your server's memory and matches the right books to the right authors.

#### Django ORM Implementation

```python
from django.db.models import Prefetch


# Reverse FK + filtered prefetch via Prefetch object
def authors_with_recent_books():
    recent_books = Book.objects.filter(published_at__year__gte=2020)
    return (
        Author.objects
        .prefetch_related(
            Prefetch("books", queryset=recent_books, to_attr="recent_books")
        )
    )

# Access pattern — zero additional queries inside the loop
def render(authors):
    return [
        {"author": a.name, "recent": [b.title for b in a.recent_books]}
        for a in authors
    ]
```

#### Generated Raw SQL

```sql
-- Query 1: fetch the authors
SELECT "app_author"."id", "app_author"."name", "app_author"."country"
FROM "app_author";

-- Query 2: fetch the prefetched books for those authors
SELECT "app_book"."id", "app_book"."title", "app_book"."author_id",
       "app_book"."publisher_id", "app_book"."published_at"
FROM "app_book"
WHERE "app_book"."published_at" >= '2020-01-01'
  AND "app_book"."author_id" IN (1, 2, 3, 4, 5, ...);  -- PKs from Query 1
```

> **Engine note:** The `IN (...)` list is materialized from the first query's result set. For very large parent sets (10k+ PKs), this `IN` clause can blow past planner limits; mitigate with batching or `Subquery`-based prefetching.

#### Interview Problem — Choosing the Right Tool

> *"You have a serializer that nests `author → books → reviews`. `select_related('author__books__reviews')` throws no error but performance is worse than before. Why, and what's correct?"*

**Diagnosis:** `books` is a *reverse* relationship (many books per author) and `reviews` is reverse off books. Forcing these into `select_related` JOINs causes **row multiplication**: an author with 50 books × 20 reviews each = 1,000 fully-duplicated author+book rows returned, ballooning the result set and memory. The planner cannot avoid it.

**Solution — mix both strategies by relationship cardinality:**

```python
authors = (
    Author.objects
    .prefetch_related(
        Prefetch(
            "books",
            queryset=Book.objects
                .select_related("publisher")          # forward FK → JOIN
                .prefetch_related("reviews"),          # reverse → separate query
        )
    )
)
# Total queries: authors(1) + books(1) + publishers(joined, 0) + reviews(1) = 3
```

**Rule of thumb:** *Forward single-valued → `select_related` (JOIN). Reverse/M2M multi-valued → `prefetch_related` (separate query). Nest them by following each hop's cardinality.*

---

## 2. Database Expressions

### 2.1 `F` Objects — Server-Side Field Arithmetic

#### Architectural Intent

`F` objects reference a model field's value **at the database level**, allowing arithmetic and comparisons to execute **inside the SQL engine** rather than round-tripping the value into Python. The two decisive benefits: (1) **atomicity** — `F("stock") - 1` compiles to `stock = stock - 1`, evaluated by the DB under its row lock, eliminating the read-modify-write race; (2) **round-trip elimination** — no fetch of the current value is required. This maps directly to the engine's ability to evaluate scalar expressions during `UPDATE`/`SELECT` execution.

#### Django ORM Implementation

```python
from django.db.models import F


class Product(models.Model):
    name = models.CharField(max_length=255)
    stock = models.IntegerField()
    price = models.DecimalField(max_digits=10, decimal_places=2)


# Atomic decrement — no race condition, no Python read
def decrement_stock(product_id, qty):
    Product.objects.filter(id=product_id, stock__gte=qty).update(
        stock=F("stock") - qty
    )


# Cross-field comparison evaluated in-engine
def overpriced_relative_to_stock():
    return Product.objects.filter(price__gt=F("stock") * 10)
```

#### Generated Raw SQL

```sql
-- decrement_stock(42, 3)
UPDATE "app_product"
SET "stock" = ("app_product"."stock" - 3)
WHERE "app_product"."id" = 42 AND "app_product"."stock" >= 3;

-- overpriced_relative_to_stock()
SELECT "app_product"."id", "app_product"."name",
       "app_product"."stock", "app_product"."price"
FROM "app_product"
WHERE "app_product"."price" > ("app_product"."stock" * 10);
```

#### Interview Problem — The Lost Update Race

> *"Two concurrent requests both buy the last unit. Both read `stock = 1` in Python, both write `stock = 0`. You oversold. Fix without `select_for_update`."*

**Diagnosis:** Read-modify-write in application code is non-atomic across transactions. The window between `SELECT` and `UPDATE` permits interleaving.

**Solution:**

```python
# Atomic conditional decrement; returns rows affected
updated = Product.objects.filter(
    id=product_id, stock__gte=1
).update(stock=F("stock") - 1)

if updated == 0:
    raise OutOfStock("No units available")
```

The `WHERE stock >= 1` plus `SET stock = stock - 1` executes as a **single atomic statement under the row lock the DB takes for the `UPDATE`**. The second concurrent request's `WHERE` clause fails to match (stock already 0), `update()` returns `0`, and we reject cleanly. No oversell.

---

### 2.2 `Q` Objects — Composable Boolean Logic

#### Architectural Intent

`Q` objects encapsulate filter predicates as composable, lazily-evaluated trees that map to SQL `WHERE` clauses with `AND`/`OR`/`NOT` operators. They exist because keyword-argument filters are implicitly `AND`-only; `Q` lets you build **`OR` logic, negation, and dynamically-assembled predicates** that compile to the engine's boolean expression evaluation.

#### Django ORM Implementation

```python
from django.db.models import Q


def search_products(term=None, in_stock=None, max_price=None):
    predicate = Q()
    if term:
        predicate &= (Q(name__icontains=term) | Q(sku__iexact=term))
    if in_stock:
        predicate &= Q(stock__gt=0)
    if max_price is not None:
        predicate &= Q(price__lte=max_price)
    return Product.objects.filter(predicate)


# Negation
def non_clearance():
    return Product.objects.filter(~Q(name__startswith="CLEARANCE"))
```

#### Generated Raw SQL

```sql
-- search_products(term="widget", in_stock=True, max_price=50)
SELECT "app_product"."id", "app_product"."name", "app_product"."sku",
       "app_product"."stock", "app_product"."price"
FROM "app_product"
WHERE (
    ("app_product"."name" ILIKE '%widget%' OR UPPER("app_product"."sku") = UPPER('widget'))
    AND "app_product"."stock" > 0
    AND "app_product"."price" <= 50
);

-- non_clearance()
SELECT ... FROM "app_product"
WHERE NOT ("app_product"."name"::text LIKE 'CLEARANCE%');
```

#### Interview Problem — Dynamic Filter Builder Performance

> *"Your faceted-search builder concatenates `Q` objects from user input. Under load, certain `OR`-heavy queries do full table scans. Why and what do you do?"*

**Diagnosis:** `OR` predicates across different columns frequently defeat the planner's ability to use a single B-tree index — the planner may fall back to a sequential scan or expensive `BitmapOr`. `ILIKE '%term%'` (leading wildcard) is inherently non-sargable and cannot use a standard B-tree at all.

**Solution:** Replace leading-wildcard text matching with a **GIN trigram index** (PostgreSQL), and ensure `OR`-ed columns each have supporting indexes so the planner can `BitmapOr` index scans instead of seq-scanning:

```python
from django.contrib.postgres.indexes import GinIndex
from django.contrib.postgres.operations import TrigramExtension

class Product(models.Model):
    # ...
    class Meta:
        indexes = [
            GinIndex(fields=["name"], name="prod_name_trgm",
                     opclasses=["gin_trgm_ops"]),
        ]
```

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX prod_name_trgm ON app_product USING GIN (name gin_trgm_ops);
```

Now `name ILIKE '%widget%'` can use the trigram index, converting a seq-scan into an index scan.

---

### 2.3 `Subquery` & `OuterRef` — Correlated Subqueries

#### Architectural Intent

`Subquery` embeds a nested `SELECT` correlated to the outer query via `OuterRef`, mapping to the engine's correlated-subquery execution. It lets you compute a **per-row scalar** (e.g., "the title of each author's most recent book") without a separate query per row and without the row explosion of a JOIN+aggregate. The engine evaluates the inner query once per outer row (or optimizes it into a join/lateral depending on the planner).

#### Django ORM Implementation

```python
from django.db.models import Subquery, OuterRef


def authors_with_latest_book_title():
    latest = (
        Book.objects
        .filter(author=OuterRef("pk"))
        .order_by("-published_at")
        .values("title")[:1]
    )
    return Author.objects.annotate(latest_book=Subquery(latest))
```

#### Generated Raw SQL

```sql
SELECT
    "app_author"."id",
    "app_author"."name",
    "app_author"."country",
    (
        SELECT U0."title"
        FROM "app_book" U0
        WHERE U0."author_id" = "app_author"."id"
        ORDER BY U0."published_at" DESC
        LIMIT 1
    ) AS "latest_book"
FROM "app_author";
```

#### Interview Problem — "Latest Row Per Group" at Scale

> *"Return each author with their most recent book. A junior used `prefetch_related` + Python `max()`. It loads every book into memory. Optimize for a 10M-row books table."*

**Diagnosis:** Loading all books to compute a per-author max is O(books) memory and network. We only need one scalar per author.

**Solution (PostgreSQL — prefer `DISTINCT ON` or lateral):** The correlated `Subquery` above pushes the "latest per group" entirely into the engine. For maximum performance, back it with a **composite index** matching the subquery's filter+sort:

```python
class Book(models.Model):
    class Meta:
        indexes = [
            models.Index(fields=["author", "-published_at"],
                         name="book_author_recent")
        ]
```

The index `(author_id, published_at DESC)` lets the engine satisfy each correlated subquery with an **index-only backward scan + `LIMIT 1`** — O(log n) per author, no table heap access for the sort.

---

### 2.4 Conditional Annotations — `Case`/`When` & Aggregate Filtering

#### Architectural Intent

`Case`/`When` compiles to SQL `CASE WHEN ... THEN ... END`, evaluating conditional logic inside the engine. Combined with `aggregate(filter=...)` (compiling to the SQL standard `FILTER (WHERE ...)` clause), it enables **single-pass conditional aggregation** — computing multiple segmented metrics in one table scan instead of one query per segment.

#### Django ORM Implementation

```python
from django.db.models import Case, When, Value, Count, Sum, DecimalField


def order_segmentation():
    return Order.objects.aggregate(
        total=Count("id"),
        high_value=Count("id", filter=Q(total__gte=1000)),
        revenue_premium=Sum(
            Case(
                When(customer_tier="premium", then=F("total")),
                default=Value(0),
                output_field=DecimalField(max_digits=12, decimal_places=2),
            )
        ),
    )
```

#### Generated Raw SQL

```sql
SELECT
    COUNT("app_order"."id") AS "total",
    COUNT("app_order"."id") FILTER (WHERE "app_order"."total" >= 1000) AS "high_value",
    SUM(
        CASE WHEN "app_order"."customer_tier" = 'premium'
             THEN "app_order"."total"
             ELSE 0
        END
    ) AS "revenue_premium"
FROM "app_order";
```

#### Interview Problem — Dashboard Firing Six Queries

> *"A KPI dashboard runs six separate `COUNT`/`SUM` queries for six segments, scanning the orders table six times. Collapse to one scan."*

**Solution:** The single `aggregate()` above computes all six metrics in **one table scan** using `FILTER (WHERE ...)` and `CASE`. Six full scans → one. On a large `orders` table this is a 6× reduction in I/O and round-trips.

---

## 3. Advanced Indexing Strategies

### 3.1 Composite Indexes & Column Ordering

#### Architectural Intent

A composite (multi-column) B-tree index orders rows by a tuple of columns. The **leftmost-prefix rule** governs usability: an index on `(a, b, c)` accelerates queries filtering on `a`, `a+b`, or `a+b+c`, but **not** `b` alone or `c` alone. Column order must mirror your query's equality-then-range access pattern: **equality columns first, range/sort column last**. This maps to how the engine traverses the B-tree — it can seek on the leading equality prefix then range-scan the trailing column.

#### Django ORM Implementation

```python
class Order(models.Model):
    customer = models.ForeignKey("Customer", on_delete=models.CASCADE)
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField()
    total = models.DecimalField(max_digits=12, decimal_places=2)

    class Meta:
        indexes = [
            # Supports: filter(status=...) + filter(created_at__gte=...) ORDER BY created_at
            models.Index(fields=["status", "-created_at"], name="order_status_created"),
        ]
```

#### Generated Raw SQL

```sql
CREATE INDEX "order_status_created"
ON "app_order" ("status", "created_at" DESC);

-- Query the index supports with an index range scan:
SELECT * FROM "app_order"
WHERE "status" = 'pending' AND "created_at" >= '2025-01-01'
ORDER BY "created_at" DESC;
```

#### Interview Problem — Index Exists but Isn't Used

> *"You have `Index(['created_at', 'status'])` but `WHERE status='x' ORDER BY created_at` does a seq-scan. Fix it."*

**Diagnosis:** Column order is wrong. With `(created_at, status)`, an equality filter on `status` cannot use the leading `created_at` column, so the planner can't seek. **Reorder to `(status, created_at)`** so the equality predicate hits the leftmost column and the sort is satisfied by the index's natural order — eliminating both the seq-scan and the sort step.

---

### 3.2 Partial Indexes

#### Architectural Intent

A partial index indexes **only rows matching a predicate**, producing a smaller, denser index that's cheaper to maintain and scan. The canonical use: a status column where queries overwhelmingly target a minority value (e.g., `status = 'pending'` among millions of `'completed'` rows). The engine uses the partial index only when the query's `WHERE` provably implies the index predicate.

#### Django ORM Implementation

```python
from django.db.models import Q


class Order(models.Model):
    # ...
    class Meta:
        indexes = [
            models.Index(
                fields=["created_at"],
                name="order_pending_idx",
                condition=Q(status="pending"),  # partial
            ),
        ]
```

#### Generated Raw SQL

```sql
CREATE INDEX "order_pending_idx"
ON "app_order" ("created_at")
WHERE ("status" = 'pending');

-- Used by:
SELECT * FROM "app_order"
WHERE "status" = 'pending' ORDER BY "created_at";
```

#### Interview Problem — Bloated Index on a Skewed Column

> *"A worker polls `status='pending'` orders. The full index on `status` is 3GB because 99% of rows are `'completed'`. Shrink it."*

**Solution:** The partial index above only stores `'pending'` rows (perhaps 0.5% of the table), shrinking the index from 3GB to megabytes. Smaller index → fewer pages → faster scans and dramatically lower write-amplification on every insert/update of non-pending rows.

---

### 3.3 PostgreSQL-Specific Indexes (GIN / BRIN / Covering)

#### Architectural Intent

PostgreSQL exposes specialized index access methods Django surfaces via `django.contrib.postgres.indexes`:
- **GIN** — inverted index for `JSONB`, arrays, and full-text/trigram search (multiple values per row).
- **BRIN** — block-range index storing min/max per block range; tiny and ideal for **naturally-ordered, append-only** columns like timestamps on huge tables.
- **Covering indexes (`INCLUDE`)** — append non-key columns so the index satisfies the query without a heap fetch (index-only scan).

#### Django ORM Implementation

```python
from django.contrib.postgres.indexes import GinIndex, BrinIndex
from django.contrib.postgres.fields import ArrayField


class Event(models.Model):
    tags = ArrayField(models.CharField(max_length=32))
    payload = models.JSONField()
    occurred_at = models.DateTimeField()
    tenant_id = models.BigIntegerField()

    class Meta:
        indexes = [
            GinIndex(fields=["payload"], name="event_payload_gin"),
            GinIndex(fields=["tags"], name="event_tags_gin"),
            BrinIndex(fields=["occurred_at"], name="event_occurred_brin"),
            # Covering index for an index-only scan
            models.Index(fields=["tenant_id"], include=["occurred_at"],
                         name="event_tenant_cover"),
        ]
```

#### Generated Raw SQL

```sql
CREATE INDEX "event_payload_gin"  ON "app_event" USING GIN ("payload");
CREATE INDEX "event_tags_gin"     ON "app_event" USING GIN ("tags");
CREATE INDEX "event_occurred_brin" ON "app_event" USING BRIN ("occurred_at");
CREATE INDEX "event_tenant_cover" ON "app_event" ("tenant_id") INCLUDE ("occurred_at");

-- GIN enables containment queries:
SELECT * FROM "app_event" WHERE "payload" @> '{"level": "error"}';
SELECT * FROM "app_event" WHERE "tags" @> ARRAY['urgent'];
```

#### Interview Problem — Indexing a 2-Billion-Row Time-Series Table

> *"An append-only events table has 2B rows. A B-tree on `occurred_at` is 60GB and slows inserts. Queries always filter by time ranges. What index?"*

**Solution:** Use **BRIN** on `occurred_at`. Because the table is append-only, physical row order correlates with time, so BRIN's per-block-range min/max summaries let the planner skip entire block ranges outside the query window. A BRIN index on 2B rows is typically **a few MB versus 60GB**, with negligible insert overhead — the right structure when data has strong physical-logical ordering correlation.

---

## 4. Concurrency Management

### 4.1 Explicit Transactions — `atomic()`

#### Architectural Intent

`transaction.atomic()` maps to `BEGIN ... COMMIT/ROLLBACK`, giving you a unit of work with ACID guarantees. Nested `atomic()` blocks use **savepoints** (`SAVEPOINT`/`RELEASE`/`ROLLBACK TO`). The critical senior-level concern is **transaction duration**: long transactions hold locks and inflate the database's snapshot horizon (MVCC bloat in PostgreSQL). Keep transactions short and **never perform network I/O (e.g., calling external APIs) inside one**.

#### Django ORM Implementation

```python
from django.db import transaction


@transaction.atomic
def transfer_funds(from_id, to_id, amount):
    src = Account.objects.select_for_update().get(id=from_id)
    dst = Account.objects.select_for_update().get(id=to_id)
    if src.balance < amount:
        raise InsufficientFunds()
    src.balance -= amount
    dst.balance += amount
    src.save(update_fields=["balance"])
    dst.save(update_fields=["balance"])


# Defer side effects until after a successful commit
def place_order(cart):
    with transaction.atomic():
        order = Order.objects.create(...)
        transaction.on_commit(lambda: send_confirmation_email.delay(order.id))
    return order
```

#### Generated Raw SQL

```sql
BEGIN;
SELECT * FROM "app_account" WHERE "id" = 1 FOR UPDATE;
SELECT * FROM "app_account" WHERE "id" = 2 FOR UPDATE;
UPDATE "app_account" SET "balance" = 900 WHERE "id" = 1;
UPDATE "app_account" SET "balance" = 1100 WHERE "id" = 2;
COMMIT;
```

#### Interview Problem — Email Sent for a Rolled-Back Order

> *"Customers get 'order confirmed' emails even when the transaction rolls back on a payment failure. Why, and fix it."*

**Diagnosis:** The email task was enqueued **inside** the transaction, before commit. If the transaction rolls back, the side effect already fired. **Solution:** wrap the dispatch in `transaction.on_commit(...)`, which only executes after a successful `COMMIT` — guaranteeing no confirmation for a rolled-back order.

---

### 4.2 Pessimistic Locking — `select_for_update`

#### Architectural Intent

`select_for_update()` emits `SELECT ... FOR UPDATE`, acquiring **row-level write locks** that block other transactions from modifying (or also locking) those rows until commit. This is **pessimistic** concurrency control: you assume conflict is likely and serialize access up front. Key options map directly to SQL: `nowait=True` → `FOR UPDATE NOWAIT` (fail instantly on contention), `skip_locked=True` → `FOR UPDATE SKIP LOCKED` (ignore locked rows — the foundation of a DB-backed work queue), and `of=(...)` to lock only specific joined tables. **It must run inside `atomic()`.**

#### Django ORM Implementation

```python
from django.db import transaction


# Work-queue pattern: each worker grabs a distinct unlocked job
@transaction.atomic
def claim_next_job():
    job = (
        Job.objects
        .select_for_update(skip_locked=True)
        .filter(status="queued")
        .order_by("created_at")
        .first()
    )
    if job is None:
        return None
    job.status = "processing"
    job.save(update_fields=["status"])
    return job
```

#### Generated Raw SQL

```sql
BEGIN;
SELECT * FROM "app_job"
WHERE "status" = 'queued'
ORDER BY "created_at" ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;          -- workers never collide on the same row

UPDATE "app_job" SET "status" = 'processing' WHERE "id" = 7;
COMMIT;
```

#### Interview Problem — Concurrent Workers Double-Processing Jobs

> *"Ten Celery workers poll a `Job` table. Occasionally two workers grab the same job and process it twice. Fix without a separate broker."*

**Diagnosis:** Plain `filter(status='queued').first()` followed by an update has a race window — multiple workers read the same `'queued'` row before any commits its status change.

**Solution:** `select_for_update(skip_locked=True)`. The first worker locks the row; concurrent workers' `SKIP LOCKED` **skips the locked row and selects the next available one**, guaranteeing each job is claimed by exactly one worker with zero blocking. This turns the relational table into a safe, contention-free work queue. Use `nowait=True` instead when you'd rather fail fast than skip.

---

## 5. Zero-Downtime Migrations

### 5.1 The Core Constraint: Lock Compatibility & Backward Compatibility

#### Architectural Intent

Zero-downtime migration rests on two principles. **(1) Lock avoidance:** DDL statements take locks; an `ACCESS EXCLUSIVE` lock on a hot table blocks all reads and writes, causing an outage. The goal is to choose DDL forms that take weak locks or hold strong locks for microseconds. **(2) Backward/forward compatibility:** during a rolling deploy, *old code and new code run simultaneously against one schema*. Every migration step must be compatible with both the currently-deployed and the about-to-deploy application versions. The canonical safe pattern is **expand → migrate → contract**.

---

### 5.2 Adding a Column / NOT NULL Safely

#### Interview Problem — Adding a `NOT NULL` Column to a 500M-Row Table

> *"Add a non-null `country` column with a default. Your first attempt locked the table for 8 minutes and took down the site. Do it with zero downtime."*

**Diagnosis:** In older PostgreSQL, `ADD COLUMN ... NOT NULL DEFAULT` rewrote the whole table under `ACCESS EXCLUSIVE`. (PG 11+ optimizes constant defaults, but `NOT NULL` validation and non-constant defaults still bite, and other engines differ.) The safe, engine-agnostic pattern decomposes into reversible steps:

#### Multi-Step Implementation

```python
# STEP 1 (expand): add NULLABLE column, no default backfill at the table level.
# migration 0002 — fast, weak lock.
class Migration(migrations.Migration):
    operations = [
        migrations.AddField(
            model_name="user",
            name="country",
            field=models.CharField(max_length=2, null=True),
        ),
    ]
```

```python
# STEP 2 (migrate): backfill in batches to avoid one giant locking UPDATE.
def backfill_country(apps, schema_editor):
    User = apps.get_model("app", "User")
    qs = User.objects.filter(country__isnull=True)
    while True:
        batch_ids = list(qs.values_list("id", flat=True)[:5000])
        if not batch_ids:
            break
        User.objects.filter(id__in=batch_ids).update(country="US")

class Migration(migrations.Migration):
    operations = [migrations.RunPython(backfill_country, migrations.RunPython.noop)]
```

```python
# STEP 3 (contract): add the NOT NULL constraint using NOT VALID + VALIDATE
# to avoid a long ACCESS EXCLUSIVE full-table scan.
class Migration(migrations.Migration):
    atomic = False  # required so the two statements aren't in one long lock
    operations = [
        migrations.RunSQL(
            sql="""
                ALTER TABLE app_user
                ADD CONSTRAINT country_not_null
                CHECK (country IS NOT NULL) NOT VALID;
            """,
            reverse_sql="ALTER TABLE app_user DROP CONSTRAINT country_not_null;",
        ),
        migrations.RunSQL(
            sql="ALTER TABLE app_user VALIDATE CONSTRAINT country_not_null;",
            reverse_sql=migrations.RunSQL.noop,
        ),
    ]
```

```sql
-- The lock-conscious DDL:
ALTER TABLE app_user ADD COLUMN country varchar(2) NULL;             -- instant
-- ... batched UPDATEs ...
ALTER TABLE app_user ADD CONSTRAINT country_not_null
    CHECK (country IS NOT NULL) NOT VALID;                           -- weak lock, instant
ALTER TABLE app_user VALIDATE CONSTRAINT country_not_null;          -- SHARE UPDATE EXCLUSIVE, allows reads/writes
```

**Why it works:** `NOT VALID` adds the constraint without scanning existing rows (it only enforces on new writes), taking a brief lock. `VALIDATE CONSTRAINT` then scans under a weak `SHARE UPDATE EXCLUSIVE` lock that **does not block reads or writes**. The 8-minute outage becomes zero perceptible downtime.

---

### 5.3 Creating Indexes Concurrently

#### Interview Problem — Index Build Froze Writes

> *"`CREATE INDEX` on a high-traffic table blocked all writes for the build duration. Avoid it."*

**Solution:** Use `AddIndexConcurrently`, which emits `CREATE INDEX CONCURRENTLY` — built without an exclusive write lock (at the cost of being non-transactional and slower).

```python
from django.contrib.postgres.operations import AddIndexConcurrently
from django.db import migrations, models


class Migration(migrations.Migration):
    atomic = False  # CONCURRENTLY cannot run inside a transaction block
    operations = [
        AddIndexConcurrently(
            model_name="order",
            index=models.Index(fields=["status", "-created_at"],
                                name="order_status_created"),
        ),
    ]
```

```sql
CREATE INDEX CONCURRENTLY "order_status_created"
ON "app_order" ("status", "created_at" DESC);
```

> **Caveat:** `CONCURRENTLY` can leave an `INVALID` index if it fails mid-build; the migration must be re-runnable and you should monitor for invalid indexes.

---

### 5.4 Renaming / Dropping Columns Safely (Expand–Contract)

#### Interview Problem — Renaming a Column Without Breaking Running Pods

> *"You need to rename `User.name` to `full_name`. A naive `RenameField` migration deploys, but old pods still query `name` and 500 during the rollout. Design the zero-downtime sequence."*

**Diagnosis:** A single rename is **not** backward compatible — old code references the old column, new code the new one, but only one name exists at any instant. You must run a sequence where both names coexist.

**Solution — expand/contract across multiple deploys:**

1. **Deploy A (expand):** Add new `full_name` column (nullable). Application writes to **both** `name` and `full_name`; reads still from `name`.
2. **Backfill:** Batched `RunPython` copies `name → full_name`.
3. **Deploy B (switch reads):** Application reads from `full_name`, still dual-writes.
4. **Deploy C (contract):** Stop writing `name`; drop the old column.

```python
# Deploy A — add the new column
migrations.AddField(
    model_name="user",
    name="full_name",
    field=models.CharField(max_length=255, null=True),
)

# Backfill migration
def copy_name(apps, schema_editor):
    User = apps.get_model("app", "User")
    qs = User.objects.filter(full_name__isnull=True)
    while True:
        ids = list(qs.values_list("id", flat=True)[:5000])
        if not ids:
            break
        for u in User.objects.filter(id__in=ids):
            User.objects.filter(id=u.id).update(full_name=u.name)

# Deploy C — drop the old column only after no code references it
migrations.RemoveField(model_name="user", name="name")
```

```sql
-- Deploy A
ALTER TABLE app_user ADD COLUMN full_name varchar(255) NULL;   -- instant
-- Backfill: batched UPDATEs
-- Deploy C (after all pods stopped using "name")
ALTER TABLE app_user DROP COLUMN name;                          -- brief lock
```

**Principle:** Every intermediate state is simultaneously compatible with the previous and next application version. State is migrated while *both* columns exist; the destructive `DROP` happens only once nothing references the old column. This is the universal blueprint for renames, type changes, and table splits at scale.

---

### 5.5 Separating State from Database Operations

#### Architectural Intent

`migrations.SeparateDatabaseAndState` lets you change what Django *thinks* the schema is (state) independently from what SQL runs (database ops). This is essential when you've already applied a change manually (e.g., `CREATE INDEX CONCURRENTLY` out-of-band) and need Django's migration state to catch up **without re-running the DDL**, or when faking a column rename at the ORM level while doing the real move via expand/contract.

```python
migrations.SeparateDatabaseAndState(
    state_operations=[
        migrations.AlterField(
            model_name="order",
            name="status",
            field=models.CharField(max_length=20, db_index=True),
        ),
    ],
    database_operations=[],  # no-op: index already created CONCURRENTLY
)
```

---

## Appendix: Senior-Level Mental Checklist

| Concern | Question to ask | Tool |
|---|---|---|
| N+1 queries | Am I accessing a relation inside a loop? | `select_related` / `prefetch_related` |
| Lost updates | Read-modify-write in Python? | `F()` atomic update / `select_for_update` |
| Index unused | Does column order match equality-then-range? | composite index reordering |
| Index bloat | Is the indexed column heavily skewed? | partial index |
| Huge ordered table | Is physical order correlated with the column? | BRIN |
| JSONB / array search | Containment/full-text queries? | GIN |
| Long lock on DDL | Does this `ALTER` rewrite or scan the table? | `NOT VALID` + `VALIDATE`, `CONCURRENTLY` |
| Side effects on rollback | Network I/O inside a transaction? | `transaction.on_commit` |
| Rename/drop during rollout | Will old & new code coexist? | expand → migrate → contract |

> **Closing principle:** Performance work at scale is the relentless minimization of three quantities — **round-trips, lock duration, and rows touched**. Every technique in this guide reduces at least one of them.
