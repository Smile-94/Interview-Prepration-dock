# 🐍 Django ORM Mastery: From Zero to Production

> **Authored for:** Mid-to-Senior Django Developers preparing for backend interviews  
> **Mentor Perspective:** 10+ Years Senior Django Engineer  
> **Focus:** Real-world systems — E-Commerce, SaaS, Inventory, Multi-Tenant Architectures

---

## 📋 Table of Contents

- [How to Use This Guide](#how-to-use-this-guide)
- [🟢 Beginner Level](#-beginner-level)
  - [1. Django Models — The Foundation](#1-django-models--the-foundation)
  - [2. CRUD Operations](#2-crud-operations)
  - [3. QuerySets and Lazy Evaluation](#3-querysets-and-lazy-evaluation)
  - [4. Field Lookups](#4-field-lookups)
  - [5. Basic Relationships (ForeignKey, ManyToMany, OneToOne)](#5-basic-relationships)
- [🟡 Intermediate Level](#-intermediate-level)
  - [6. select_related vs prefetch_related](#6-select_related-vs-prefetch_related)
  - [7. Q Objects and Complex Queries](#7-q-objects-and-complex-queries)
  - [8. F Expressions](#8-f-expressions)
  - [9. Aggregation and Annotation](#9-aggregation-and-annotation)
  - [10. Custom Managers and QuerySets](#10-custom-managers-and-querysets)
- [🔴 Advanced Level](#-advanced-level)
  - [11. Transactions and Atomicity](#11-transactions-and-atomicity)
  - [12. Subqueries and OuterRef](#12-subqueries-and-outerref)
  - [13. Query Optimization and the N+1 Problem](#13-query-optimization-and-the-n1-problem)
  - [14. Indexing Strategies](#14-indexing-strategies)
  - [15. Raw SQL — When and Why](#15-raw-sql--when-and-why)
- [⚫ Expert Level](#-expert-level)
  - [16. Multi-Database Handling](#16-multi-database-handling)
  - [17. Multi-Tenant Architectures](#17-multi-tenant-architectures)
  - [18. Caching Strategies with Redis](#18-caching-strategies-with-redis)
  - [19. ORM vs Raw SQL — Trade-offs at Scale](#19-orm-vs-raw-sql--trade-offs-at-scale)
  - [20. Scaling ORM in High-Traffic Systems](#20-scaling-orm-in-high-traffic-systems)
- [⚠️ Common Pitfalls](#️-common-pitfalls)
- [⚡ Performance Optimization Cheatsheet](#-performance-optimization-cheatsheet)
- [🚀 Rapid Revision — Interview Prep](#-rapid-revision--interview-prep)

---

## How to Use This Guide

Each section follows this pattern:

```
📖 Concept Explanation  →  💻 Code Example  →  🎯 Interview Questions  →  ✅ Answers with Trade-offs
```

Read linearly if you're building from scratch. Jump to sections if you're revising specific topics. Every code example is **production-grade**, not tutorial-grade — meaning it handles edge cases, uses proper abstractions, and reflects real team codebases.

---

# 🟢 Beginner Level

---

## 1. Django Models — The Foundation

### 📖 Concept

A Django model is a Python class that maps directly to a database table. It's the single source of truth for your data structure — schema, constraints, and behavior all live here. The ORM handles the SQL translation, but **you** are responsible for designing the model correctly because a bad schema is almost impossible to fix in production without downtime.

Think of models as your domain objects. In an e-commerce system, `Product`, `Order`, `Customer`, and `Invoice` are domain objects. Getting their fields, constraints, and relationships right **at design time** saves you from painful migrations and data corruption later.

### 💻 Code Example — E-Commerce Product Model

```python
# products/models.py
import uuid
from django.db import models
from django.core.validators import MinValueValidator
from django.utils.translation import gettext_lazy as _


class TimestampedModel(models.Model):
    """
    Abstract base model providing created_at and updated_at timestamps.
    All production models should inherit from this.
    """
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Category(TimestampedModel):
    name = models.CharField(max_length=200, unique=True)
    slug = models.SlugField(max_length=200, unique=True)
    description = models.TextField(blank=True)
    is_active = models.BooleanField(default=True)

    class Meta:
        verbose_name = _("category")
        verbose_name_plural = _("categories")
        ordering = ["name"]

    def __str__(self):
        return self.name


class Product(TimestampedModel):
    class Status(models.TextChoices):
        DRAFT = "draft", _("Draft")
        ACTIVE = "active", _("Active")
        ARCHIVED = "archived", _("Archived")

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    sku = models.CharField(max_length=100, unique=True, db_index=True)
    name = models.CharField(max_length=500)
    slug = models.SlugField(max_length=500, unique=True)
    category = models.ForeignKey(
        Category,
        on_delete=models.PROTECT,   # Never silently delete a category with products
        related_name="products",
    )
    price = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(0)],
    )
    stock = models.PositiveIntegerField(default=0)
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.DRAFT,
        db_index=True,
    )
    description = models.TextField(blank=True)

    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["status", "category"]),
            models.Index(fields=["sku"]),
        ]

    def __str__(self):
        return f"{self.sku} — {self.name}"

    @property
    def is_in_stock(self) -> bool:
        return self.stock > 0

    @property
    def is_available(self) -> bool:
        return self.status == self.Status.ACTIVE and self.is_in_stock
```

**Why `on_delete=models.PROTECT`?** In e-commerce, you don't want a category deletion to cascade and wipe out thousands of products. `PROTECT` raises a `ProtectedError`, forcing a deliberate decision. Use `CASCADE` only when child records are truly meaningless without the parent (e.g., `OrderItem` → `Order`).

**Why UUID primary key?** In distributed systems or when you expose IDs in URLs/APIs, sequential integers leak business data (total orders, total users). UUIDs are opaque and safe.

---

### 🎯 Interview Questions

---

**Q1 (Conceptual): What is the difference between `null=True` and `blank=True` in Django model fields?**

**✅ Answer:**

| Parameter | Database Level | Form/Validation Level |
|-----------|---------------|----------------------|
| `null=True` | Stores SQL `NULL` in the column | No effect on forms |
| `blank=True` | No DB change | Allows empty string in forms/serializers |

```python
# WRONG — Common beginner mistake
description = models.CharField(max_length=500, null=True)
# This stores NULL in a text field. Django convention: use empty string for text.

# CORRECT
description = models.CharField(max_length=500, blank=True, default="")

# CORRECT for optional relationships or numeric fields
discount_price = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
```

**The Rule:**
- For `CharField`, `TextField`: use `blank=True`, avoid `null=True` (two empty states = confusion)
- For `ForeignKey`, `DecimalField`, `DateField`: use `null=True, blank=True` when optional

---

**Q2 (Scenario): You're building an inventory system. A `Product` can belong to multiple `Warehouse` locations with a different quantity at each location. How do you model this?**

**✅ Answer:**

This is a **Many-to-Many with extra data** — a classic use case for an explicit through model:

```python
class Warehouse(TimestampedModel):
    name = models.CharField(max_length=200)
    location = models.CharField(max_length=500)
    is_active = models.BooleanField(default=True)

    def __str__(self):
        return self.name


class ProductInventory(TimestampedModel):
    """
    Through model — Product ↔ Warehouse with quantity per location.
    Use this instead of ManyToManyField when you need extra attributes.
    """
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name="inventories")
    warehouse = models.ForeignKey(Warehouse, on_delete=models.CASCADE, related_name="inventories")
    quantity = models.PositiveIntegerField(default=0)
    reorder_threshold = models.PositiveIntegerField(default=10)

    class Meta:
        unique_together = ("product", "warehouse")  # One record per product-warehouse pair
        indexes = [
            models.Index(fields=["product", "warehouse"]),
        ]

    def __str__(self):
        return f"{self.product.sku} @ {self.warehouse.name}: {self.quantity}"

    @property
    def needs_reorder(self) -> bool:
        return self.quantity <= self.reorder_threshold
```

**Why not `ManyToManyField` directly?** Django's implicit M2M through table only stores the relationship keys — no room for `quantity` or `reorder_threshold`. An explicit through model gives you full control.

---

**Q3 (Tricky): Why should you almost never use `unique_together` in a new project today?**

**✅ Answer:**

`unique_together` is a legacy option. Django 4.x+ recommends `UniqueConstraint` from `Meta.constraints`, which is strictly more powerful:

```python
# Legacy — avoid in new code
class Meta:
    unique_together = ("product", "warehouse")

# Modern — preferred
class Meta:
    constraints = [
        models.UniqueConstraint(
            fields=["product", "warehouse"],
            name="unique_product_warehouse",
        ),
        # Can also add conditions:
        models.UniqueConstraint(
            fields=["sku"],
            condition=models.Q(status="active"),
            name="unique_active_sku",
        ),
    ]
```

`UniqueConstraint` supports:
- `condition=` for partial unique constraints
- `deferrable=` for deferred constraint checking
- `include=` for covering indexes (PostgreSQL)
- Named constraints (easier to reference in migrations and error messages)

---

## 2. CRUD Operations

### 📖 Concept

CRUD (Create, Read, Update, Delete) is where you'll spend 80% of your ORM time. The key insight is that Django's ORM gives you multiple ways to do each operation — and the choice matters for **performance, atomicity, and signal firing**.

### 💻 Code Examples

```python
# ============================================================
# CREATE
# ============================================================

# Method 1: save() — fires pre_save and post_save signals
product = Product(sku="LAPTOP-001", name="Pro Laptop", price=999.99)
product.save()

# Method 2: create() — same as above but one-liner
product = Product.objects.create(sku="LAPTOP-001", name="Pro Laptop", price=999.99)

# Method 3: get_or_create() — atomic, prevents duplicate inserts
# Returns: (instance, created: bool)
product, created = Product.objects.get_or_create(
    sku="LAPTOP-001",
    defaults={"name": "Pro Laptop", "price": 999.99, "status": Product.Status.DRAFT},
)
# IMPORTANT: Only `sku` is used in the WHERE clause.
# `defaults` are only applied on creation, not lookup.

# Method 4: update_or_create() — upsert pattern
product, created = Product.objects.update_or_create(
    sku="LAPTOP-001",
    defaults={"price": 1099.99, "status": Product.Status.ACTIVE},
)

# Method 5: bulk_create() — high-performance batch inserts
products = [
    Product(sku=f"ITEM-{i}", name=f"Item {i}", price=10.00)
    for i in range(1000)
]
# ignore_conflicts=True skips rows that violate unique constraints
Product.objects.bulk_create(products, batch_size=500, ignore_conflicts=True)


# ============================================================
# READ
# ============================================================

# Single object — raises DoesNotExist or MultipleObjectsReturned
product = Product.objects.get(sku="LAPTOP-001")

# Safe single object
product = Product.objects.filter(sku="LAPTOP-001").first()  # Returns None if not found

# Queryset (lazy — no DB hit yet)
active_products = Product.objects.filter(status=Product.Status.ACTIVE)

# Force evaluation
product_list = list(active_products)           # Evaluates immediately
count = active_products.count()                # Single COUNT query — not len()
exists = active_products.exists()              # Faster than count() > 0
first = active_products.first()


# ============================================================
# UPDATE
# ============================================================

# Method 1: instance.save() — fetches, modifies, saves (2 queries)
# Use update_fields to avoid overwriting concurrent changes
product = Product.objects.get(sku="LAPTOP-001")
product.price = 899.99
product.save(update_fields=["price", "updated_at"])  # Only updates these columns

# Method 2: queryset.update() — single SQL UPDATE, no signals
# Best for bulk updates or when you don't need signal logic
Product.objects.filter(
    status=Product.Status.DRAFT,
    created_at__lt=timezone.now() - timedelta(days=30),
).update(status=Product.Status.ARCHIVED)

# Method 3: bulk_update() — update multiple instances efficiently
products = list(Product.objects.filter(category__name="Electronics"))
for p in products:
    p.price = p.price * 0.9  # 10% discount
Product.objects.bulk_update(products, fields=["price"], batch_size=500)


# ============================================================
# DELETE
# ============================================================

# Single delete — fires pre_delete and post_delete signals
product = Product.objects.get(sku="LAPTOP-001")
product.delete()

# Bulk delete — single SQL, no signals, be careful
Product.objects.filter(status=Product.Status.ARCHIVED).delete()

# Soft delete pattern (production best practice)
class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(deleted_at__isnull=True)

class Product(TimestampedModel):
    deleted_at = models.DateTimeField(null=True, blank=True)
    objects = SoftDeleteManager()
    all_objects = models.Manager()  # Access including deleted

    def soft_delete(self):
        self.deleted_at = timezone.now()
        self.save(update_fields=["deleted_at"])
```

---

### 🎯 Interview Questions

---

**Q1 (Tricky): What is the difference between `queryset.update()` and calling `.save()` on each instance?**

**✅ Answer:**

| Aspect | `queryset.update()` | `instance.save()` |
|--------|--------------------|--------------------|
| SQL queries | **1 UPDATE** | 1 SELECT + 1 UPDATE per object |
| Django signals | ❌ Not fired | ✅ `pre_save`, `post_save` fired |
| `auto_now` fields | ❌ Not updated automatically | ✅ Updated |
| `update_fields` | N/A | Use to limit columns |
| Use case | Bulk ops without side effects | Single object with signal logic |

```python
# This fires signals — good when post_save sends notifications
order.status = "shipped"
order.save(update_fields=["status"])

# This does NOT fire signals — bad if post_save sends a shipping email
Order.objects.filter(id=order.id).update(status="shipped")
```

**Real-world rule:** Use `.update()` for bulk data migrations and background jobs. Use `.save()` when the model has business logic in signals or `save()` overrides.

---

**Q2 (Scenario): Your e-commerce platform needs to atomically decrement stock when an order is placed, but only if stock > 0. How do you do this without a race condition?**

**✅ Answer:**

This is a **check-then-act** problem — a classic race condition. The solution uses F expressions with a conditional update:

```python
from django.db.models import F
from django.db import transaction

def reserve_stock(product_id: int, quantity: int) -> bool:
    """
    Atomically reserve stock. Returns True if successful, False if insufficient.
    """
    with transaction.atomic():
        # Single atomic UPDATE WHERE stock >= quantity
        rows_updated = Product.objects.filter(
            id=product_id,
            stock__gte=quantity,       # Condition checked in DB, not Python
        ).update(
            stock=F("stock") - quantity  # F expression — DB does the math
        )
        return rows_updated == 1

# Usage
success = reserve_stock(product_id=123, quantity=2)
if not success:
    raise InsufficientStockError("Not enough stock")
```

**Why this works:** The `WHERE stock >= quantity` check and the `UPDATE` happen in a single SQL statement — the database handles the atomicity. No Python-level race condition is possible.

**Why NOT this:**

```python
# RACE CONDITION — two threads can both read stock=1 and both decrement
product = Product.objects.get(id=product_id)
if product.stock >= quantity:
    product.stock -= quantity
    product.save()
```

---

## 3. QuerySets and Lazy Evaluation

### 📖 Concept

QuerySets are **lazy** — they don't hit the database until you explicitly evaluate them. This is one of Django's most important design decisions, and also the source of some of the most subtle bugs.

Understanding **when** a QuerySet is evaluated is critical for avoiding unnecessary database queries and understanding how to chain operations efficiently.

### 💻 Code Examples

```python
# ============================================================
# QuerySets are lazy — no DB query yet
# ============================================================
qs = Product.objects.filter(status="active")        # No query
qs = qs.exclude(stock=0)                            # No query
qs = qs.select_related("category")                  # No query
qs = qs.order_by("-created_at")                     # No query

# ============================================================
# These force evaluation (hit the database)
# ============================================================
list(qs)                    # Iteration
qs[0]                       # Slicing
qs[0:10]                    # Slicing (returns new QuerySet, not evaluated until iterated)
bool(qs)                    # Boolean check — use .exists() instead!
len(qs)                     # Use .count() instead!
repr(qs)                    # Repr (development only)
for product in qs: ...      # For loop

# ============================================================
# QuerySet caching — the hidden footgun
# ============================================================
qs = Product.objects.filter(status="active")

# First access — queries the DB and caches the result
for p in qs:
    print(p.name)

# Second access — uses CACHED result (no new query)
for p in qs:
    print(p.price)

# BUT slicing bypasses cache:
qs[0]   # Query 1
qs[0]   # Query 2 — NOT cached!

# AND calling filter() on an evaluated QuerySet re-queries:
qs.filter(price__gt=100)  # New query, cache not used


# ============================================================
# QuerySet chaining — composable queries (DRY principle)
# ============================================================
class ProductQuerySet(models.QuerySet):
    def active(self):
        return self.filter(status=Product.Status.ACTIVE)

    def in_stock(self):
        return self.filter(stock__gt=0)

    def in_category(self, category_slug: str):
        return self.filter(category__slug=category_slug)

    def available(self):
        return self.active().in_stock()


class ProductManager(models.Manager):
    def get_queryset(self):
        return ProductQuerySet(self.model, using=self._db)

    def available(self):
        return self.get_queryset().available()


# Usage — chainable and readable
products = (
    Product.objects
    .available()
    .in_category("electronics")
    .order_by("-created_at")[:20]
)
```

---

### 🎯 Interview Questions

---

**Q1 (Tricky): What does `len(queryset)` vs `queryset.count()` actually do, and which should you use?**

**✅ Answer:**

```python
qs = Product.objects.filter(status="active")

# BAD — fetches ALL rows from DB, loads into memory, then counts Python list length
count = len(qs)
# SQL: SELECT * FROM products WHERE status='active'  (all columns, all rows!)

# GOOD — single lightweight SQL COUNT
count = qs.count()
# SQL: SELECT COUNT(*) FROM products WHERE status='active'
```

**Exception:** If you've already evaluated the QuerySet (i.e., you're iterating over it anyway), `len()` is fine because the data is already in memory and avoids a second DB round-trip.

```python
products = list(Product.objects.filter(status="active"))
# Already evaluated, use len() — no extra query
count = len(products)
```

---

**Q2 (Scenario): A teammate writes the following code in a view. What's wrong?**

```python
def product_list_view(request):
    products = Product.objects.filter(status="active")
    if len(products) == 0:
        return render(request, "empty.html")
    return render(request, "products.html", {"products": products})
```

**✅ Answer:**

Multiple issues:

1. `len(products)` fetches **all** rows (including all columns) just to check if the list is empty — potentially very expensive.
2. The QuerySet is evaluated twice: once for `len()`, once in the template.

**Fixed version:**

```python
def product_list_view(request):
    products = Product.objects.filter(status="active").select_related("category")

    if not products.exists():   # Single lightweight: SELECT 1 FROM ... LIMIT 1
        return render(request, "empty.html")

    return render(request, "products.html", {"products": products})
    # Template evaluation is the only time the full queryset hits the DB
```

---

## 4. Field Lookups

### 📖 Concept

Field lookups are the `__` (double underscore) syntax that translates Python to SQL `WHERE` clauses. Mastering them eliminates the need for raw SQL in 95% of cases.

### 💻 Code Examples

```python
from django.utils import timezone
from datetime import timedelta

# ============================================================
# Common lookups
# ============================================================
Product.objects.filter(name__exact="Pro Laptop")        # = (case-sensitive)
Product.objects.filter(name__iexact="pro laptop")       # = (case-insensitive)
Product.objects.filter(name__contains="Laptop")         # LIKE '%Laptop%'
Product.objects.filter(name__icontains="laptop")        # ILIKE '%laptop%'
Product.objects.filter(name__startswith="Pro")          # LIKE 'Pro%'
Product.objects.filter(name__endswith="Laptop")         # LIKE '%Laptop'

# Numeric comparisons
Product.objects.filter(price__gt=100)                   # > 100
Product.objects.filter(price__gte=100)                  # >= 100
Product.objects.filter(price__lt=500)                   # < 500
Product.objects.filter(price__lte=500)                  # <= 500
Product.objects.filter(price__range=(100, 500))         # BETWEEN 100 AND 500

# Null checks
Product.objects.filter(deleted_at__isnull=True)         # IS NULL
Product.objects.filter(deleted_at__isnull=False)        # IS NOT NULL

# In/Not in
Product.objects.filter(status__in=["active", "draft"])  # IN (...)
Product.objects.exclude(status__in=["archived"])        # NOT IN (...)

# Date/time lookups
Product.objects.filter(
    created_at__date=timezone.now().date()              # Cast to date
)
Product.objects.filter(created_at__year=2024)
Product.objects.filter(created_at__month=12)
Product.objects.filter(
    created_at__gte=timezone.now() - timedelta(days=30) # Last 30 days
)

# ============================================================
# Relationship traversal — the real power
# ============================================================
# Products in the "Electronics" category (JOIN traversal)
Product.objects.filter(category__name="Electronics")

# Products in active categories
Product.objects.filter(category__is_active=True)

# Multi-level traversal (SaaS: Orders → Customer → Subscription plan)
Order.objects.filter(customer__subscription__plan__name="Premium")

# ============================================================
# Span relationships — multi-level, multi-hop
# ============================================================
# Orders containing a specific product SKU (M2M traversal)
Order.objects.filter(items__product__sku="LAPTOP-001").distinct()
# Note: .distinct() prevents duplicate Order rows when items match multiple times
```

---

### 🎯 Interview Questions

---

**Q1 (Tricky): When does Django's `filter()` generate a JOIN vs a subquery? Why does this matter?**

**✅ Answer:**

By default, `filter()` with a relationship traversal generates a `JOIN`. This matters because:

```python
# This generates a JOIN — can return DUPLICATE rows if M2M
orders = Order.objects.filter(items__product__sku="LAPTOP-001")
# SQL: SELECT orders.* FROM orders
#      INNER JOIN order_items ON order_items.order_id = orders.id
#      INNER JOIN products ON products.id = order_items.product_id
#      WHERE products.sku = 'LAPTOP-001'
# If an order has 3 laptop items, it appears 3 TIMES in results!

# Fix: always use .distinct() with M2M traversal
orders = Order.objects.filter(items__product__sku="LAPTOP-001").distinct()
```

**Performance implication:** JOINs on large tables can be expensive. In such cases, `filter(id__in=subquery)` can be faster because it avoids duplicates without `DISTINCT` and the planner may choose a better execution path:

```python
from django.db.models import Subquery, OuterRef

laptop_order_ids = OrderItem.objects.filter(
    product__sku="LAPTOP-001"
).values("order_id")

orders = Order.objects.filter(id__in=laptop_order_ids)
```

---

## 5. Basic Relationships

### 📖 Concept

Relationships define how models connect. The choice of relationship type and `on_delete` behavior are architectural decisions with real-world consequences — getting them wrong means data integrity issues in production.

### 💻 Code Examples — E-Commerce Order System

```python
# orders/models.py
from django.conf import settings
from django.db import models
from products.models import Product, TimestampedModel


class Address(TimestampedModel):
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="addresses",
    )
    line1 = models.CharField(max_length=500)
    line2 = models.CharField(max_length=500, blank=True)
    city = models.CharField(max_length=200)
    country = models.CharField(max_length=100)
    postal_code = models.CharField(max_length=20)
    is_default = models.BooleanField(default=False)

    class Meta:
        verbose_name_plural = "addresses"

    def __str__(self):
        return f"{self.line1}, {self.city}, {self.country}"


class Order(TimestampedModel):
    class Status(models.TextChoices):
        PENDING = "pending", "Pending"
        CONFIRMED = "confirmed", "Confirmed"
        SHIPPED = "shipped", "Shipped"
        DELIVERED = "delivered", "Delivered"
        CANCELLED = "cancelled", "Cancelled"
        REFUNDED = "refunded", "Refunded"

    # ForeignKey — many orders belong to one customer
    customer = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.PROTECT,       # Don't delete user if they have orders
        related_name="orders",
    )
    # OneToOne — an order has exactly one shipping address snapshot
    shipping_address = models.ForeignKey(
        Address,
        on_delete=models.SET_NULL,     # Keep order even if address deleted
        null=True,
        related_name="orders",
    )
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDING,
        db_index=True,
    )
    total_price = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    notes = models.TextField(blank=True)

    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["customer", "status"]),
            models.Index(fields=["status", "created_at"]),
        ]

    def __str__(self):
        return f"Order #{self.pk} — {self.customer.email}"


class OrderItem(TimestampedModel):
    """
    Line items — snapshot price at time of purchase.
    Never reference product.price directly; it changes over time.
    """
    order = models.ForeignKey(
        Order,
        on_delete=models.CASCADE,      # Items die with the order
        related_name="items",
    )
    product = models.ForeignKey(
        Product,
        on_delete=models.PROTECT,      # Don't delete products with order history
        related_name="order_items",
    )
    quantity = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)  # Snapshot!

    class Meta:
        unique_together = ("order", "product")

    @property
    def total_price(self) -> "Decimal":
        return self.unit_price * self.quantity
```

---

# 🟡 Intermediate Level

---

## 6. select_related vs prefetch_related

### 📖 Concept

This is the **most asked Django interview question** at the intermediate level. Both solve the N+1 problem but in fundamentally different ways. Using the wrong one can actually make performance worse.

| | `select_related` | `prefetch_related` |
|---|---|---|
| **Relationship type** | ForeignKey, OneToOne | ManyToMany, reverse ForeignKey |
| **SQL strategy** | Single JOIN query | Separate query per relationship, Python-side joining |
| **Best for** | Single object follow | Collections (lists) |
| **Traversal depth** | Unlimited with `__` | Unlimited with `Prefetch()` |

### 💻 Code Examples

```python
from django.db.models import Prefetch


# ============================================================
# PROBLEM: N+1 Query (the silent killer)
# ============================================================
orders = Order.objects.filter(status="confirmed")
for order in orders:
    # DANGER: Each iteration fires a new SQL query for customer
    print(order.customer.email)
# Result: 1 query for orders + N queries for customers = N+1

# ============================================================
# FIX 1: select_related — FK/OneToOne (JOIN)
# ============================================================
orders = Order.objects.select_related(
    "customer",
    "shipping_address",
).filter(status="confirmed")

for order in orders:
    print(order.customer.email)        # No extra query
    print(order.shipping_address.city) # No extra query

# SQL: SELECT orders.*, auth_user.*, address.*
#      FROM orders
#      INNER JOIN auth_user ON auth_user.id = orders.customer_id
#      LEFT OUTER JOIN address ON address.id = orders.shipping_address_id

# ============================================================
# FIX 2: prefetch_related — Reverse FK / M2M
# ============================================================
orders = Order.objects.prefetch_related(
    "items",            # Reverse FK — OrderItem.order
    "items__product",   # FK on OrderItem
).filter(status="confirmed")

for order in orders:
    for item in order.items.all():   # No extra query — uses prefetch cache
        print(item.product.name)

# SQL: 3 queries total:
# 1. SELECT * FROM orders WHERE status='confirmed'
# 2. SELECT * FROM order_items WHERE order_id IN (1, 2, 3, ...)
# 3. SELECT * FROM products WHERE id IN (...)

# ============================================================
# Advanced: Prefetch with custom QuerySet
# ============================================================
# Only prefetch active items in confirmed orders
active_items_prefetch = Prefetch(
    "items",
    queryset=OrderItem.objects.select_related("product").filter(
        product__status=Product.Status.ACTIVE
    ),
    to_attr="active_items",  # Stored as list attribute, not manager
)

orders = Order.objects.prefetch_related(active_items_prefetch).filter(
    status="confirmed"
)

for order in orders:
    for item in order.active_items:  # Accesses the to_attr list
        print(item.product.name)

# ============================================================
# Combining both
# ============================================================
orders = (
    Order.objects
    .select_related("customer", "shipping_address")  # FK joins
    .prefetch_related(
        Prefetch(
            "items",
            queryset=OrderItem.objects.select_related("product"),
        )
    )
    .filter(status="confirmed")
    .order_by("-created_at")
)
# Total: 2 SQL queries regardless of how many orders/items
```

---

### 🎯 Interview Questions

---

**Q1 (Scenario): You have a SaaS dashboard that displays a list of companies, each with their subscription plan and recent users. The page is slow. How do you investigate and fix it?**

**✅ Answer:**

**Step 1: Detect with Django Debug Toolbar or logging**

```python
# settings/development.py
LOGGING = {
    "version": 1,
    "handlers": {"console": {"class": "logging.StreamHandler"}},
    "loggers": {
        "django.db.backends": {
            "level": "DEBUG",
            "handlers": ["console"],
        }
    },
}
```

**Step 2: Identify the N+1**

```python
# Slow view — N+1 on plan and users
companies = Company.objects.filter(is_active=True)
for company in companies:
    print(company.plan.name)          # N queries
    print(company.users.count())      # N more queries
```

**Step 3: Fix it**

```python
from django.db.models import Count, Prefetch

companies = (
    Company.objects
    .select_related("plan")           # JOIN for plan (FK)
    .prefetch_related(
        Prefetch(
            "users",
            queryset=User.objects.only("id", "email", "last_login"),
        )
    )
    .annotate(user_count=Count("users"))  # COUNT in the main query
    .filter(is_active=True)
    .order_by("name")
)

for company in companies:
    print(company.plan.name)          # No extra query
    print(company.user_count)         # No extra query (annotated)
    for user in company.users.all():  # Uses prefetch cache
        print(user.email)
```

**Total queries: 2** (1 main + 1 users prefetch), regardless of company count.

---

**Q2 (Tricky): When can `prefetch_related` actually be slower than no prefetch?**

**✅ Answer:**

`prefetch_related` can be counterproductive when:

1. **You only need 1–2 objects** — the extra query overhead isn't worth it
2. **You filter the prefetch in Python after the fact** — destroys the benefit

```python
# Prefetch loads ALL items, then Python filters — wasteful
orders = Order.objects.prefetch_related("items")
for order in orders:
    # This filter BYPASSES the prefetch cache and fires a NEW query!
    expensive_items = order.items.filter(unit_price__gt=100)
```

**Fix: Use `Prefetch()` with a queryset:**

```python
expensive_prefetch = Prefetch(
    "items",
    queryset=OrderItem.objects.filter(unit_price__gt=100),
    to_attr="expensive_items",
)
orders = Order.objects.prefetch_related(expensive_prefetch)
for order in orders:
    for item in order.expensive_items:  # Uses prefetch cache
        print(item.unit_price)
```

---

## 7. Q Objects and Complex Queries

### 📖 Concept

`Q` objects let you build complex `WHERE` clauses with `AND`, `OR`, and `NOT` operations. They're essential for search features, filtering UIs, and permission systems where conditions are dynamic.

### 💻 Code Examples

```python
from django.db.models import Q


# ============================================================
# Basic Q usage
# ============================================================
# Products that are either in "Electronics" OR price < 50
products = Product.objects.filter(
    Q(category__name="Electronics") | Q(price__lt=50)
)

# Products in "Electronics" AND price < 500 AND in stock
products = Product.objects.filter(
    Q(category__name="Electronics") &
    Q(price__lt=500) &
    Q(stock__gt=0)
)

# NOT — active but NOT out of stock
products = Product.objects.filter(
    Q(status="active") & ~Q(stock=0)
)


# ============================================================
# Real-world: Dynamic search for SaaS admin panel
# ============================================================
def search_products(
    query: str | None = None,
    category_ids: list[int] | None = None,
    min_price: float | None = None,
    max_price: float | None = None,
    in_stock_only: bool = False,
    status: str | None = None,
) -> models.QuerySet:
    """
    Composable search function — build Q objects conditionally.
    This pattern avoids the anti-pattern of chaining .filter() in if/else blocks.
    """
    filters = Q()  # Start with empty Q (always True — no-op)

    if query:
        filters &= Q(name__icontains=query) | Q(sku__icontains=query) | Q(description__icontains=query)

    if category_ids:
        filters &= Q(category_id__in=category_ids)

    if min_price is not None:
        filters &= Q(price__gte=min_price)

    if max_price is not None:
        filters &= Q(price__lte=max_price)

    if in_stock_only:
        filters &= Q(stock__gt=0)

    if status:
        filters &= Q(status=status)

    return Product.objects.filter(filters).select_related("category").distinct()


# ============================================================
# Q objects for permission-based filtering (SaaS multi-tenancy)
# ============================================================
def get_visible_products(user) -> models.QuerySet:
    """
    Admins see everything. Staff see active + their org's products.
    Regular users see only active, in-stock products.
    """
    if user.is_superuser:
        return Product.objects.all()

    if user.is_staff:
        return Product.objects.filter(
            Q(status=Product.Status.ACTIVE) |
            Q(organization=user.organization)
        )

    return Product.objects.filter(
        Q(status=Product.Status.ACTIVE) &
        Q(stock__gt=0)
    )
```

---

### 🎯 Interview Questions

---

**Q1 (Tricky): What is the difference between chaining `.filter()` and using `Q()` with `&`? Are they always equivalent?**

**✅ Answer:**

For simple AND conditions they're equivalent. But with **reverse FK or M2M traversal**, they produce different SQL and different results:

```python
# Chained filter — generates separate JOIN per filter (intersection)
# Orders where items include BOTH product 1 AND product 2 in the SAME item
Order.objects.filter(items__product_id=1).filter(items__product_id=2)
# SQL: Two separate JOINs — finds orders with at least one of each product

# Q with & — generates single JOIN (union condition on same row)
Order.objects.filter(Q(items__product_id=1) & Q(items__product_id=2))
# SQL: Single JOIN — tries to find a SINGLE order_item matching BOTH (usually empty!)

# With OR — Q is the only option
Order.objects.filter(Q(items__product_id=1) | Q(items__product_id=2))
```

**Rule of thumb:** With M2M/reverse FK, chained `.filter()` filters per-JOIN (multi-valued relationship). `Q` objects filter per-row. When in doubt, use `.distinct()` and verify with `.query`.

```python
# Debug: print the actual SQL
print(str(Order.objects.filter(items__product_id=1).filter(items__product_id=2).query))
```

---

## 8. F Expressions

### 📖 Concept

`F()` expressions reference model field values at the **database level**, allowing you to update fields relative to their current value without pulling data into Python. This is essential for concurrency-safe updates and performance-critical batch operations.

### 💻 Code Examples

```python
from django.db.models import F, ExpressionWrapper, DecimalField
from django.db.models.functions import Greatest, Least


# ============================================================
# Basic F expressions
# ============================================================

# Increment without fetching — no race condition
Product.objects.filter(id=1).update(stock=F("stock") - 1)

# Bulk price adjustment — 10% discount on all Electronics
Product.objects.filter(category__name="Electronics").update(
    price=F("price") * 0.9
)

# Compare two fields on the same model
# Find products where stock is below reorder level
from products.models import ProductInventory
low_stock = ProductInventory.objects.filter(
    quantity__lt=F("reorder_threshold")
)


# ============================================================
# F with annotations
# ============================================================
from django.db.models import ExpressionWrapper, FloatField

# Calculate profit margin as a computed field
products_with_margin = Product.objects.annotate(
    profit_margin=ExpressionWrapper(
        (F("price") - F("cost_price")) / F("price") * 100,
        output_field=DecimalField(max_digits=5, decimal_places=2),
    )
).filter(profit_margin__lt=20)  # Margins below 20%


# ============================================================
# F with ordering — order by computed expression
# ============================================================
from django.db.models.functions import Upper

# Order by uppercase name (case-insensitive sort)
products = Product.objects.order_by(Upper("name"))

# Order by a field on a related model
orders = Order.objects.order_by(F("customer__last_name").asc(nulls_last=True))


# ============================================================
# Real-world: Flash sale — apply discount safely
# ============================================================
def apply_flash_sale(category_id: int, discount_percent: int) -> int:
    """
    Apply a percentage discount to all active products in a category.
    Returns number of products updated.
    Safe for concurrent execution.
    """
    updated = Product.objects.filter(
        category_id=category_id,
        status=Product.Status.ACTIVE,
    ).update(
        sale_price=Greatest(
            F("price") * (1 - discount_percent / 100),
            F("min_price"),          # Never go below minimum price
        )
    )
    return updated
```

---

### 🎯 Interview Questions

---

**Q1 (Scenario): Why is `F()` critical for a high-traffic inventory system, and what bug occurs without it?**

**✅ Answer:**

Without `F()`, stock decrements have a **lost update** race condition:

```python
# Thread 1: reads stock=5
# Thread 2: reads stock=5
# Thread 1: sets stock=4, saves
# Thread 2: sets stock=4, saves   ← WRONG — should be 3!

product = Product.objects.get(id=1)  # Thread-unsafe read
product.stock -= 1
product.save()
```

With `F()`, the math happens in the database atomically:

```python
Product.objects.filter(id=1).update(stock=F("stock") - 1)
# SQL: UPDATE products SET stock = stock - 1 WHERE id = 1
# The DB handles atomicity — no lost update possible.
```

For strict safety (e.g., preventing negative stock), combine with a conditional update as shown in the CRUD section.

---

## 9. Aggregation and Annotation

### 📖 Concept

Aggregation computes a summary over a QuerySet (e.g., total revenue). Annotation adds a computed column per row (e.g., total items per order). Both translate to SQL `GROUP BY`, `COUNT`, `SUM`, `AVG`, and window functions.

### 💻 Code Examples

```python
from django.db.models import (
    Avg, Count, DecimalField, ExpressionWrapper,
    F, Max, Min, Sum, Value, When, Case
)
from django.db.models.functions import Coalesce, TruncMonth


# ============================================================
# Basic Aggregation
# ============================================================
from orders.models import Order, OrderItem

stats = Order.objects.aggregate(
    total_orders=Count("id"),
    total_revenue=Sum("total_price"),
    avg_order_value=Avg("total_price"),
    max_order_value=Max("total_price"),
    min_order_value=Min("total_price"),
)
# Returns a dict: {"total_orders": 1024, "total_revenue": Decimal("89432.50"), ...}

# Aggregate with filter
stats = Order.objects.filter(status="delivered").aggregate(
    revenue=Coalesce(Sum("total_price"), Value(0), output_field=DecimalField()),
    # Coalesce: returns first non-null — handles case where no orders exist
)


# ============================================================
# Annotation — computed column per row
# ============================================================

# Annotate each order with its item count
orders = Order.objects.annotate(
    item_count=Count("items"),
    order_total=Sum("items__unit_price"),
)
for order in orders:
    print(order.item_count, order.order_total)  # No extra queries


# ============================================================
# Grouping — orders per customer per month (SaaS billing report)
# ============================================================
monthly_revenue = (
    Order.objects
    .filter(status="delivered")
    .annotate(month=TruncMonth("created_at"))
    .values("month")
    .annotate(
        order_count=Count("id"),
        revenue=Sum("total_price"),
        avg_value=Avg("total_price"),
    )
    .order_by("month")
)
# SQL: SELECT DATE_TRUNC('month', created_at), COUNT(*), SUM(total_price), ...
#      FROM orders WHERE status='delivered' GROUP BY month ORDER BY month


# ============================================================
# Conditional Aggregation — the power move
# ============================================================

# Count orders by status in a single query
order_status_summary = Order.objects.aggregate(
    total=Count("id"),
    pending=Count("id", filter=Q(status="pending")),
    confirmed=Count("id", filter=Q(status="confirmed")),
    shipped=Count("id", filter=Q(status="shipped")),
    cancelled=Count("id", filter=Q(status="cancelled")),
)
# SQL: SELECT
#        COUNT(*),
#        COUNT(*) FILTER (WHERE status = 'pending'),
#        ...
#      FROM orders
# All in ONE query!


# ============================================================
# Case/When — conditional value annotation
# ============================================================
from django.db.models import Case, IntegerField, When

products_with_tier = Product.objects.annotate(
    price_tier=Case(
        When(price__lt=50, then=Value("budget")),
        When(price__lt=200, then=Value("mid-range")),
        When(price__gte=200, then=Value("premium")),
        default=Value("unknown"),
        output_field=models.CharField(),
    )
)

# Group by computed tier
tier_stats = (
    Product.objects
    .annotate(
        price_tier=Case(
            When(price__lt=50, then=Value("budget")),
            When(price__lt=200, then=Value("mid-range")),
            When(price__gte=200, then=Value("premium")),
            default=Value("unknown"),
            output_field=models.CharField(),
        )
    )
    .values("price_tier")
    .annotate(count=Count("id"), avg_price=Avg("price"))
    .order_by("price_tier")
)
```

---

### 🎯 Interview Questions

---

**Q1 (Scenario): You need a dashboard showing, for each product, the total units sold and total revenue — all in one query. How?**

**✅ Answer:**

```python
from django.db.models import Sum, Count, F, DecimalField, ExpressionWrapper

product_sales = (
    Product.objects
    .filter(order_items__order__status="delivered")
    .annotate(
        units_sold=Sum("order_items__quantity"),
        revenue=Sum(
            ExpressionWrapper(
                F("order_items__quantity") * F("order_items__unit_price"),
                output_field=DecimalField(max_digits=12, decimal_places=2),
            )
        ),
        order_count=Count("order_items__order", distinct=True),
    )
    .filter(units_sold__isnull=False)  # Exclude products with no sales
    .order_by("-revenue")
    .values("id", "sku", "name", "units_sold", "revenue", "order_count")
)
```

**Key points:**
- `ExpressionWrapper` is needed when multiplying two fields with different types
- `Count(..., distinct=True)` prevents double-counting via JOIN expansion
- `.values()` at the end limits the selected columns (more efficient)

---

## 10. Custom Managers and QuerySets

### 📖 Concept

Custom managers and QuerySets are the Django ORM's answer to the Repository pattern. They encapsulate query logic, prevent duplication, and keep views/services thin and readable. A well-designed manager is one of the biggest markers of a senior Django developer.

### 💻 Code Examples

```python
# core/managers.py
from django.db import models
from django.utils import timezone


class SoftDeleteQuerySet(models.QuerySet):
    """Reusable soft-delete QuerySet. Mix into any model."""

    def active(self):
        return self.filter(deleted_at__isnull=True)

    def deleted(self):
        return self.filter(deleted_at__isnull=False)

    def delete(self):
        """Override bulk delete to be soft."""
        return self.update(deleted_at=timezone.now())

    def hard_delete(self):
        """Bypass soft delete for compliance/GDPR."""
        return super().delete()


class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return SoftDeleteQuerySet(self.model, using=self._db).active()

    def with_deleted(self):
        return SoftDeleteQuerySet(self.model, using=self._db)

    def only_deleted(self):
        return SoftDeleteQuerySet(self.model, using=self._db).deleted()


# products/managers.py
class ProductQuerySet(models.QuerySet):
    def available(self):
        return self.filter(status=Product.Status.ACTIVE, stock__gt=0)

    def by_category(self, *slugs: str):
        return self.filter(category__slug__in=slugs)

    def with_sales_data(self):
        """Annotate with sales summary — use only when needed."""
        from django.db.models import Sum, Count
        return self.annotate(
            total_sold=Sum("order_items__quantity"),
            revenue=Sum(
                ExpressionWrapper(
                    F("order_items__quantity") * F("order_items__unit_price"),
                    output_field=DecimalField(max_digits=12, decimal_places=2),
                )
            ),
        )

    def low_stock(self, threshold: int = 10):
        return self.filter(stock__lte=threshold, stock__gt=0)

    def out_of_stock(self):
        return self.filter(stock=0)

    def with_category(self):
        return self.select_related("category")


class ProductManager(SoftDeleteManager):
    def get_queryset(self):
        return ProductQuerySet(self.model, using=self._db).active()

    # Convenience proxies for common patterns
    def available(self):
        return self.get_queryset().available()

    def low_stock(self, threshold: int = 10):
        return self.get_queryset().low_stock(threshold)


# Usage in Product model
class Product(TimestampedModel):
    deleted_at = models.DateTimeField(null=True, blank=True)

    objects = ProductManager()           # Default: excludes deleted
    all_objects = models.Manager()       # Raw access

    class Meta:
        base_manager_name = "all_objects"  # Use for related manager traversal

# ============================================================
# Usage — reads like business logic, not ORM calls
# ============================================================
# In a view or service:
available = Product.objects.available().with_category()[:20]
low_stock = Product.objects.low_stock(5).with_sales_data()
deleted = Product.objects.with_deleted().only_deleted()
```

---

# 🔴 Advanced Level

---

## 11. Transactions and Atomicity

### 📖 Concept

A transaction ensures that a set of database operations either all succeed or all fail together — the foundation of data integrity. In production, misunderstanding transactions leads to partial data writes, ghost orders, and billing inconsistencies.

### 💻 Code Examples

```python
from django.db import transaction
from django.db.models import F


# ============================================================
# atomic() — explicit transaction block
# ============================================================
def place_order(customer, cart_items: list[dict]) -> Order:
    """
    Create order, decrement stock, and create order items atomically.
    If any step fails, everything is rolled back.
    """
    with transaction.atomic():
        # Create the order record
        order = Order.objects.create(
            customer=customer,
            status=Order.Status.PENDING,
        )

        total = Decimal("0")

        for item_data in cart_items:
            product_id = item_data["product_id"]
            quantity = item_data["quantity"]

            # Lock the row for update to prevent race conditions
            product = (
                Product.objects
                .select_for_update()     # SELECT ... FOR UPDATE (row-level lock)
                .get(id=product_id)
            )

            if product.stock < quantity:
                raise InsufficientStockError(
                    f"Only {product.stock} units of {product.sku} available"
                )

            # Create line item with price snapshot
            OrderItem.objects.create(
                order=order,
                product=product,
                quantity=quantity,
                unit_price=product.price,
            )

            # Decrement stock
            product.stock = F("stock") - quantity
            product.save(update_fields=["stock"])

            total += product.price * quantity

        order.total_price = total
        order.status = Order.Status.CONFIRMED
        order.save(update_fields=["total_price", "status"])

        return order


# ============================================================
# savepoints — nested transactions
# ============================================================
def process_bulk_orders(orders_data: list[dict]) -> dict:
    """
    Process multiple orders. Failed individual orders don't abort the batch.
    """
    results = {"success": [], "failed": []}

    with transaction.atomic():   # Outer transaction for the batch
        for order_data in orders_data:
            try:
                with transaction.atomic():   # Savepoint per order
                    order = place_order(**order_data)
                    results["success"].append(order.id)
            except (InsufficientStockError, Exception) as e:
                # Savepoint rolled back; outer transaction continues
                results["failed"].append({"data": order_data, "error": str(e)})

    return results


# ============================================================
# on_commit — defer side effects until after commit
# ============================================================
def place_order_with_notifications(customer, cart_items):
    with transaction.atomic():
        order = place_order(customer, cart_items)

        # WRONG: If transaction rolls back after this, email was sent but order doesn't exist!
        # send_order_confirmation_email(order)

        # CORRECT: Run only if the transaction commits successfully
        transaction.on_commit(
            lambda: send_order_confirmation_email.delay(order.id)  # Celery task
        )
        transaction.on_commit(
            lambda: update_inventory_cache.delay(order.id)
        )

    return order


# ============================================================
# select_for_update — prevent lost updates with explicit locking
# ============================================================
def transfer_credits(from_user_id: int, to_user_id: int, amount: Decimal):
    """Safe credit transfer — no negative balances, no lost updates."""
    with transaction.atomic():
        # Lock both rows in consistent order (lower ID first) to prevent deadlock
        users = (
            UserProfile.objects
            .select_for_update()
            .filter(user_id__in=[from_user_id, to_user_id])
            .order_by("user_id")
        )

        user_map = {u.user_id: u for u in users}
        sender = user_map[from_user_id]
        receiver = user_map[to_user_id]

        if sender.credits < amount:
            raise InsufficientCreditsError("Not enough credits")

        sender.credits = F("credits") - amount
        receiver.credits = F("credits") + amount

        sender.save(update_fields=["credits"])
        receiver.save(update_fields=["credits"])
```

---

### 🎯 Interview Questions

---

**Q1 (Tricky): What is the `on_commit` hook, and why is it critical when using Celery with Django?**

**✅ Answer:**

Without `on_commit`, tasks queued inside a transaction can execute **before** the transaction commits, reading stale or non-existent data:

```python
# DANGEROUS
with transaction.atomic():
    order = Order.objects.create(...)
    send_email_task.delay(order.id)   # Celery worker might run before commit!
    # If the transaction rolls back here, the email task references a non-existent order

# SAFE
with transaction.atomic():
    order = Order.objects.create(...)
    transaction.on_commit(
        lambda: send_email_task.delay(order.id)
    )
    # send_email_task only fires after the transaction successfully commits
```

**Real consequence:** In production, this causes `Order.DoesNotExist` errors in Celery tasks that run milliseconds after being queued — a race condition that's hard to reproduce in development.

---

**Q2 (Scenario): Explain deadlock risk with `select_for_update` and how to prevent it.**

**✅ Answer:**

Deadlocks occur when two transactions lock rows in different orders:

```
Transaction A: Locks User #1, waits for User #2
Transaction B: Locks User #2, waits for User #1
→ Deadlock!
```

**Prevention:** Always acquire locks in a consistent order:

```python
# Always sort by ID before locking — guarantees consistent lock order
users = UserProfile.objects.select_for_update().filter(
    user_id__in=[from_user_id, to_user_id]
).order_by("user_id")   # ← Consistent ordering prevents deadlock
```

**Additional options:**

```python
# SKIP LOCKED — skip rows already locked (for queue processing)
Product.objects.select_for_update(skip_locked=True).filter(needs_processing=True)[:10]

# NOWAIT — fail immediately instead of waiting (detect contention fast)
try:
    Product.objects.select_for_update(nowait=True).get(id=product_id)
except OperationalError:
    raise ResourceLockedException("Product is being processed by another request")
```

---

## 12. Subqueries and OuterRef

### 📖 Concept

Subqueries let you embed one query inside another, accessing the outer query's fields via `OuterRef`. They're often more performant than fetching data in Python and filtering, especially for "latest record per group" patterns.

### 💻 Code Examples

```python
from django.db.models import OuterRef, Subquery, Exists


# ============================================================
# Basic Subquery — latest order per customer
# ============================================================
latest_order = Order.objects.filter(
    customer=OuterRef("pk"),       # References the outer Customer query
    status="delivered",
).order_by("-created_at").values("total_price")[:1]
# [:1] is crucial — limits subquery to single row for scalar use

customers = Customer.objects.annotate(
    last_order_value=Subquery(
        latest_order,
        output_field=DecimalField(),
    )
)
# SQL: SELECT *, (
#        SELECT total_price FROM orders
#        WHERE customer_id = customers.id AND status='delivered'
#        ORDER BY created_at DESC LIMIT 1
#      ) AS last_order_value
#      FROM customers


# ============================================================
# Exists — more efficient than COUNT for boolean checks
# ============================================================
# Filter customers who have at least one pending order
recent_order = Order.objects.filter(
    customer=OuterRef("pk"),
    status="pending",
)

active_customers = Customer.objects.filter(
    Exists(recent_order)            # SELECT 1 WHERE EXISTS (subquery)
)
# More efficient than: Customer.objects.filter(orders__status='pending').distinct()


# ============================================================
# Real-world: Products with their most recent discount
# ============================================================
latest_discount = ProductDiscount.objects.filter(
    product=OuterRef("pk"),
    is_active=True,
).order_by("-created_at").values("discount_percent")[:1]

products = Product.objects.annotate(
    current_discount=Subquery(latest_discount, output_field=models.IntegerField()),
    is_on_sale=Exists(
        ProductDiscount.objects.filter(
            product=OuterRef("pk"),
            is_active=True,
        )
    )
).filter(is_on_sale=True)


# ============================================================
# Subquery vs Prefetch — know when to use which
# ============================================================
# Use Subquery when you need ONE value per outer row
# Use Prefetch when you need a COLLECTION per outer row

# SUBQUERY: latest review rating per product (scalar)
latest_rating = ProductReview.objects.filter(
    product=OuterRef("pk")
).order_by("-created_at").values("rating")[:1]

Product.objects.annotate(latest_rating=Subquery(latest_rating))

# PREFETCH: all reviews per product (collection)
Product.objects.prefetch_related(
    Prefetch("reviews", queryset=ProductReview.objects.order_by("-created_at"))
)
```

---

## 13. Query Optimization and the N+1 Problem

### 📖 Concept

The N+1 problem is the single most common performance issue in Django applications. It occurs when fetching a list of N objects causes N additional queries. In production with real data volumes, this turns a 20ms page into a 5,000ms disaster.

### 💻 Detection and Fix Examples

```python
# ============================================================
# Detection Tools
# ============================================================

# 1. Django Debug Toolbar (development)
# Add 'debug_toolbar' to INSTALLED_APPS — shows query count per request

# 2. django-silk for profiling in staging
# Records every query with execution time and call stack

# 3. Manual detection with assertNumQueries in tests
from django.test import TestCase

class OrderViewTest(TestCase):
    def test_order_list_query_count(self):
        # Create 10 test orders
        orders = [Order.objects.create(customer=self.user) for _ in range(10)]

        with self.assertNumQueries(2):   # Should be exactly 2 queries
            response = self.client.get("/api/orders/")
            self.assertEqual(response.status_code, 200)


# ============================================================
# Common N+1 patterns and fixes
# ============================================================

# --- PATTERN 1: Accessing FK in a loop ---

# BAD: N+1
orders = Order.objects.all()
for order in orders:
    print(order.customer.email)     # 1 query per order

# GOOD: select_related
orders = Order.objects.select_related("customer").all()
for order in orders:
    print(order.customer.email)     # 0 extra queries


# --- PATTERN 2: Reverse FK in a loop ---

# BAD: N+1
orders = Order.objects.all()
for order in orders:
    items = order.items.all()       # 1 query per order
    for item in items:
        print(item.product.name)    # + 1 query per item = N*M+N+1 total!

# GOOD: prefetch_related with select_related
orders = Order.objects.prefetch_related(
    Prefetch("items", queryset=OrderItem.objects.select_related("product"))
).all()


# --- PATTERN 3: Conditional logic in loop ---

# BAD: Re-querying inside a condition
orders = Order.objects.filter(status="confirmed")
for order in orders:
    if order.items.filter(product__category__name="Electronics").exists():
        # EXISTS query per order!
        pass

# GOOD: Annotate the condition
orders = Order.objects.filter(status="confirmed").annotate(
    has_electronics=Exists(
        OrderItem.objects.filter(
            order=OuterRef("pk"),
            product__category__name="Electronics",
        )
    )
)
for order in orders:
    if order.has_electronics:   # No extra query
        pass


# --- PATTERN 4: Property access that hides queries ---

# BAD: @property with hidden DB call
class Order(models.Model):
    @property
    def customer_name(self):
        return self.customer.get_full_name()  # Triggers query if customer not prefetched

# GOOD: Use select_related in the consuming queryset, or annotate
orders = Order.objects.annotate(
    customer_name=Concat(
        "customer__first_name", Value(" "), "customer__last_name",
        output_field=CharField()
    )
)
```

---

### 🎯 Interview Questions

---

**Q1 (Scenario): You have a REST API endpoint that returns a list of orders with customer info and items. A load test shows 500ms response time for 50 orders. Walk through your optimization process.**

**✅ Answer:**

```python
# Step 1: Identify — log queries
import logging
logger = logging.getLogger("django.db.backends")

# Step 2: Baseline (terrible)
def order_list_view_v1(request):
    orders = Order.objects.all()  # No optimization
    # Template/serializer accesses customer and items → N+1+M queries

# Step 3: Add select_related and prefetch_related
def order_list_view_v2(request):
    orders = (
        Order.objects
        .select_related("customer", "shipping_address")
        .prefetch_related(
            Prefetch(
                "items",
                queryset=OrderItem.objects
                    .select_related("product__category")
                    .only("id", "order_id", "quantity", "unit_price",
                          "product__name", "product__sku",
                          "product__category__name"),
            )
        )
        .only(  # Fetch only needed columns
            "id", "status", "total_price", "created_at",
            "customer__id", "customer__email", "customer__first_name",
            "shipping_address__city", "shipping_address__country",
        )
        .filter(status__in=["confirmed", "shipped"])
        .order_by("-created_at")
    )
    # Now: 2-3 queries regardless of result count

# Step 4: Add pagination
    paginator = Paginator(orders, 25)
    page_obj = paginator.get_page(request.GET.get("page"))
    # Count query + 1 data query + 1 prefetch = 3 total

# Step 5: Add caching for read-heavy data
    cache_key = f"orders:page:{page_obj.number}"
    cached = cache.get(cache_key)
    if cached:
        return Response(cached)
    data = OrderSerializer(page_obj, many=True).data
    cache.set(cache_key, data, timeout=60)
    return Response(data)
```

**Result:** 500ms → ~15ms at 50 orders. The N+1 fix alone usually gives 10-50x improvement.

---

## 14. Indexing Strategies

### 📖 Concept

Indexes are the most impactful performance tool after fixing N+1. Without indexes, PostgreSQL does a full table scan on every query. With proper indexes, queries go from O(N) to O(log N). But indexes have a write cost — every INSERT/UPDATE must update the index. Over-indexing is a real problem.

### 💻 Code Examples

```python
from django.db import models
from django.contrib.postgres.indexes import BrinIndex, GinIndex, HashIndex


class Order(TimestampedModel):
    customer = models.ForeignKey(...)
    status = models.CharField(max_length=20, choices=Status.choices, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)
    total_price = models.DecimalField(...)
    search_vector = SearchVectorField(null=True)

    class Meta:
        indexes = [
            # Composite index — covers (customer, status) queries
            models.Index(fields=["customer", "status"], name="order_customer_status_idx"),

            # Composite index for time-range queries
            models.Index(fields=["status", "-created_at"], name="order_status_date_idx"),

            # Partial index — only index confirmed/shipped orders (smaller, faster)
            models.Index(
                fields=["customer", "created_at"],
                condition=Q(status__in=["confirmed", "shipped", "delivered"]),
                name="order_active_customer_date_idx",
            ),

            # BRIN index — for append-only time-series tables (very small overhead)
            BrinIndex(fields=["created_at"], name="order_created_brin_idx"),

            # GIN index — for full-text search
            GinIndex(fields=["search_vector"], name="order_search_gin_idx"),

            # Hash index — equality-only lookups on UUIDs (faster than B-tree for =)
            HashIndex(fields=["tracking_number"], name="order_tracking_hash_idx"),
        ]
```

**Index Design Rules:**

```
1. Index columns that appear in WHERE clauses (most important)
2. Index FK columns (Django doesn't do this automatically for all DBs)
3. Composite index: put equality-filter columns first, range-filter last
4. Partial indexes: dramatically reduce index size for filtered queries
5. Covering indexes: include extra columns to avoid table lookups (PostgreSQL)
6. Monitor with: EXPLAIN ANALYZE SELECT ...
```

```sql
-- Always explain your queries in production tuning
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders
WHERE customer_id = 123 AND status = 'confirmed'
ORDER BY created_at DESC
LIMIT 25;

-- Look for: Seq Scan → should be Index Scan
-- Look for: actual rows vs estimated rows (bad estimates = stale stats)
-- Run: ANALYZE orders;  to update statistics
```

---

## 15. Raw SQL — When and Why

### 📖 Concept

The ORM is powerful but not omnipotent. Raw SQL is justified for:
- Complex `WINDOW` functions
- Recursive CTEs
- Stored procedure calls
- Query plan hints
- Batch operations where ORM overhead matters

Use it with discipline: raw SQL bypasses Django's protections (SQL injection if done wrong, migrations don't track it).

### 💻 Code Examples

```python
from django.db import connection


# ============================================================
# Method 1: raw() — maps to model instances
# ============================================================
# Safe — parameterized query, never use string formatting
products = Product.objects.raw(
    "SELECT * FROM products WHERE price BETWEEN %s AND %s ORDER BY price",
    [min_price, max_price]
)
# Returns a RawQuerySet — supports iteration and some queryset operations

# ============================================================
# Method 2: connection.cursor() — for non-model queries
# ============================================================
def get_revenue_by_region() -> list[dict]:
    """Complex aggregation better expressed in SQL."""
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT
                a.country,
                COUNT(DISTINCT o.id) AS order_count,
                SUM(o.total_price) AS revenue,
                AVG(o.total_price) AS avg_order_value
            FROM orders o
            INNER JOIN addresses a ON a.id = o.shipping_address_id
            WHERE o.status = %s
              AND o.created_at >= %s
            GROUP BY a.country
            ORDER BY revenue DESC
            LIMIT %s
        """, ["delivered", timezone.now() - timedelta(days=90), 20])

        columns = [col[0] for col in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]


# ============================================================
# Method 3: Window functions (PostgreSQL)
# ============================================================
from django.db.models import Window, RowNumber, Rank
from django.db.models.functions import Rank

# Rank products by price within each category (Window Function)
ranked_products = Product.objects.annotate(
    price_rank=Window(
        expression=Rank(),
        partition_by=[F("category_id")],
        order_by=F("price").asc(),
    )
).filter(price_rank__lte=3)  # Top 3 cheapest per category
# SQL: SELECT *, RANK() OVER (PARTITION BY category_id ORDER BY price ASC) AS price_rank
#      FROM products
#      WHERE price_rank <= 3  — PostgreSQL handles this in a CTE automatically


# ============================================================
# When to use Raw SQL checklist:
# ============================================================
# ✅ Use raw SQL:
# - Recursive CTEs (e.g., hierarchical category trees)
# - Window functions not supported by ORM
# - Batch INSERT with ON CONFLICT DO UPDATE (upsert with conditions)
# - COPY command for bulk CSV import (psycopg2 directly)
# - Query plan hints (pg_hint_plan)
# - Database-specific features (PostGIS, pg_trgm)

# ❌ Avoid raw SQL:
# - Simple CRUD (ORM is cleaner and safer)
# - When you're just avoiding learning the ORM
# - When it bypasses critical business logic in model.save()
```

---

# ⚫ Expert Level

---

## 16. Multi-Database Handling

### 📖 Concept

Large systems often require multiple databases: a primary for writes, one or more replicas for reads, and separate databases for analytics or archiving. Django has built-in multi-database support via database routers.

### 💻 Code Examples

```python
# settings/production.py
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "app_primary",
        "HOST": "primary.db.internal",
        "PORT": "5432",
        "CONN_MAX_AGE": 60,
        "OPTIONS": {
            "pool_min_conn": 2,
            "pool_max_conn": 10,
        },
    },
    "replica_1": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "app_replica",
        "HOST": "replica-1.db.internal",
        "TEST": {"NAME": "app_primary"},  # Use primary DB in tests
    },
    "replica_2": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "app_replica",
        "HOST": "replica-2.db.internal",
        "TEST": {"NAME": "app_primary"},
    },
    "analytics": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "app_analytics",
        "HOST": "analytics.db.internal",
    },
}

DATABASE_ROUTERS = ["core.db_routers.PrimaryReplicaRouter"]


# core/db_routers.py
import random


class PrimaryReplicaRouter:
    """
    Routes read queries to replicas, writes to primary.
    All auth/session tables always go to primary.
    """
    REPLICA_DBS = ["replica_1", "replica_2"]
    ALWAYS_PRIMARY = {"auth", "sessions", "contenttypes", "admin"}

    def db_for_read(self, model, **hints):
        """Route reads to a random replica."""
        if model._meta.app_label in self.ALWAYS_PRIMARY:
            return "default"
        return random.choice(self.REPLICA_DBS)

    def db_for_write(self, model, **hints):
        """All writes go to primary."""
        return "default"

    def allow_relation(self, obj1, obj2, **hints):
        """Allow relations if both objects are in the same database pool."""
        db_set = {"default", "replica_1", "replica_2"}
        if obj1._state.db in db_set and obj2._state.db in db_set:
            return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """Only migrate on primary."""
        return db == "default"


# Explicit database selection in code
class AnalyticsQuerySet(models.QuerySet):
    def get_queryset(self):
        return super().get_queryset().using("analytics")

# Or at call site:
Product.objects.using("replica_1").filter(status="active")
Product.objects.using("analytics").values("category").annotate(count=Count("id"))

# In transactions — always use primary
with transaction.atomic(using="default"):
    order = Order.objects.using("default").create(...)
```

---

## 17. Multi-Tenant Architectures

### 📖 Concept

Multi-tenancy is the hallmark of SaaS platforms. There are three strategies, each with different trade-offs:

| Strategy | Isolation | Complexity | Scale |
|----------|-----------|------------|-------|
| **Schema-per-tenant** | High | High | Medium |
| **Row-level (shared schema)** | Low | Low | High |
| **Database-per-tenant** | Very High | Very High | Low |

### 💻 Row-Level Multi-Tenancy with Custom Manager

```python
# core/models.py
import threading

_tenant_local = threading.local()


def get_current_tenant():
    return getattr(_tenant_local, "tenant", None)


def set_current_tenant(tenant):
    _tenant_local.tenant = tenant


class TenantQuerySet(models.QuerySet):
    def for_tenant(self, tenant):
        return self.filter(tenant=tenant)

    def current(self):
        tenant = get_current_tenant()
        if tenant is None:
            raise ValueError("No tenant in context. Use TenantMiddleware.")
        return self.filter(tenant=tenant)


class TenantManager(models.Manager):
    def get_queryset(self):
        return TenantQuerySet(self.model, using=self._db).current()


class TenantModel(TimestampedModel):
    """
    Abstract base for all tenant-scoped models.
    Automatically filters by current tenant.
    """
    tenant = models.ForeignKey(
        "accounts.Tenant",
        on_delete=models.CASCADE,
        related_name="+",
    )

    objects = TenantManager()
    global_objects = models.Manager()   # Bypass tenant filter (admin use only)

    class Meta:
        abstract = True


# Middleware — sets tenant from subdomain or JWT claim
# core/middleware.py
class TenantMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        tenant = self._resolve_tenant(request)
        if tenant:
            set_current_tenant(tenant)
        response = self.get_response(request)
        set_current_tenant(None)   # Clean up
        return response

    def _resolve_tenant(self, request):
        # Strategy 1: Subdomain (acme.app.com → tenant 'acme')
        host = request.get_host().split(":")[0]
        subdomain = host.split(".")[0]
        return Tenant.objects.filter(subdomain=subdomain).first()

        # Strategy 2: JWT claim
        # return getattr(request, "tenant", None)


# Usage — all queries automatically scoped to current tenant
class Product(TenantModel):
    name = models.CharField(max_length=500)
    price = models.DecimalField(max_digits=10, decimal_places=2)

# In a view:
# Product.objects.all()  → automatically adds WHERE tenant_id = {current_tenant}
# Product.objects.filter(status="active")  → tenant-safe
```

---

## 18. Caching Strategies with Redis

### 📖 Concept

Database queries are the primary bottleneck in most Django applications. Redis caching eliminates repeated queries for read-heavy data. The challenge is **cache invalidation** — knowing when to expire stale data.

### 💻 Code Examples

```python
# settings/production.py
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://redis.internal:6379/0",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
            "SOCKET_CONNECT_TIMEOUT": 5,
            "SOCKET_TIMEOUT": 5,
            "CONNECTION_POOL_KWARGS": {"max_connections": 50},
            "COMPRESSOR": "django_redis.compressors.zlib.ZlibCompressor",
        },
        "KEY_PREFIX": "app",
        "TIMEOUT": 300,   # 5 minutes default
    }
}


# ============================================================
# Pattern 1: Cache-aside (manual, most common)
# ============================================================
from django.core.cache import cache
from django.core.cache.utils import make_template_fragment_key


def get_active_categories() -> list:
    """
    Cache the active category list for 10 minutes.
    Invalidated on category save/delete via signals.
    """
    cache_key = "categories:active:v2"   # Version in key = easy cache busting
    result = cache.get(cache_key)

    if result is None:
        result = list(
            Category.objects
            .filter(is_active=True)
            .values("id", "name", "slug")
            .order_by("name")
        )
        cache.set(cache_key, result, timeout=600)   # 10 minutes

    return result


# Signal-based invalidation
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver

@receiver([post_save, post_delete], sender=Category)
def invalidate_category_cache(sender, **kwargs):
    cache.delete("categories:active:v2")
    cache.delete_many([
        f"category:{kwargs['instance'].id}:products",
        "homepage:featured_categories",
    ])


# ============================================================
# Pattern 2: Per-object caching with versioning
# ============================================================
def get_product(product_id: int) -> dict | None:
    cache_key = f"product:{product_id}:detail"
    product_data = cache.get(cache_key)

    if product_data is None:
        try:
            product = (
                Product.objects
                .select_related("category")
                .get(id=product_id, status=Product.Status.ACTIVE)
            )
            product_data = {
                "id": str(product.id),
                "sku": product.sku,
                "name": product.name,
                "price": str(product.price),
                "category": product.category.name,
                "is_in_stock": product.is_in_stock,
            }
            cache.set(cache_key, product_data, timeout=300)
        except Product.DoesNotExist:
            return None

    return product_data


# ============================================================
# Pattern 3: Cache stampede prevention with lock
# ============================================================
import time

def get_product_safe(product_id: int) -> dict | None:
    """Prevent thundering herd / cache stampede."""
    cache_key = f"product:{product_id}:detail"
    lock_key = f"product:{product_id}:lock"

    result = cache.get(cache_key)
    if result is not None:
        return result

    # Only one worker regenerates; others wait briefly
    acquired = cache.add(lock_key, "1", timeout=10)   # NX — only set if not exists
    if acquired:
        try:
            result = _fetch_product_from_db(product_id)
            cache.set(cache_key, result, timeout=300)
        finally:
            cache.delete(lock_key)
    else:
        # Wait for the lock holder to populate cache
        for _ in range(10):
            time.sleep(0.1)
            result = cache.get(cache_key)
            if result is not None:
                return result

    return result


# ============================================================
# Pattern 4: QuerySet-level caching with django-cacheops (library)
# ============================================================
# pip install django-cacheops
# settings.py:
# CACHEOPS = {
#     'products.product': {'ops': 'all', 'timeout': 60*15},
#     'products.category': {'ops': ('fetch', 'get'), 'timeout': 60*60},
# }

# After setup, queries are auto-cached and invalidated:
# Product.objects.filter(status='active').cache()  ← explicit
# Category.objects.all()  ← auto-cached per CACHEOPS config
```

---

## 19. ORM vs Raw SQL — Trade-offs at Scale

### 📖 Concept

This is a senior-level architectural question. The answer is never "always use ORM" or "always use raw SQL" — it's about understanding the trade-off matrix.

```
┌─────────────────────────┬──────────────────────┬──────────────────────┐
│ Criterion               │ Django ORM           │ Raw SQL              │
├─────────────────────────┼──────────────────────┼──────────────────────┤
│ SQL injection safety    │ ✅ Parameterized auto  │ ⚠️ Manual discipline  │
│ Portability (DB change) │ ✅ Easy swap           │ ❌ Rewrite needed     │
│ Readability             │ ✅ Pythonic            │ ⚠️ Context-dependent  │
│ Complex queries         │ ⚠️ Verbose/impossible  │ ✅ Natural expression  │
│ Performance ceiling     │ ⚠️ ORM overhead        │ ✅ Full DB capability  │
│ Migrations              │ ✅ Tracked             │ ❌ Manual migration    │
│ Test isolation          │ ✅ Fixtures/factories  │ ⚠️ Harder             │
│ Window functions        │ ✅ Django 3.x+         │ ✅ Native             │
│ CTEs / Recursive        │ ❌ Not native          │ ✅ Full support        │
│ Bulk operations         │ ✅ bulk_create/update   │ ✅ COPY command faster │
└─────────────────────────┴──────────────────────┴──────────────────────┘
```

### 💻 Decision Framework

```python
# Rule 1: Use ORM for standard CRUD and simple aggregations
# The ORM handles 90% of cases better than raw SQL

# Rule 2: Use ORM Window functions — Django 3.x+ supports them well
from django.db.models import Window
from django.db.models.functions import Rank, Lead, Lag, NthValue

Product.objects.annotate(
    running_total=Window(
        expression=Sum("price"),
        partition_by=[F("category_id")],
        order_by=F("price").asc(),
        frame=RowRange(start=None, end=0),
    )
)

# Rule 3: Use raw SQL for CTEs (recursive or complex)
def get_category_tree(root_id: int) -> list[dict]:
    """Recursive CTE — not expressible in ORM."""
    with connection.cursor() as cursor:
        cursor.execute("""
            WITH RECURSIVE category_tree AS (
                SELECT id, name, parent_id, 0 AS depth
                FROM categories
                WHERE id = %s

                UNION ALL

                SELECT c.id, c.name, c.parent_id, ct.depth + 1
                FROM categories c
                INNER JOIN category_tree ct ON c.parent_id = ct.id
            )
            SELECT * FROM category_tree ORDER BY depth, name
        """, [root_id])
        columns = [col[0] for col in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]

# Rule 4: Use database views for complex read patterns
# Create the view in a migration's RunSQL, then map to an unmanaged model
class RevenueByRegionView(models.Model):
    """Maps to a PostgreSQL materialized view."""
    country = models.CharField(max_length=100)
    revenue = models.DecimalField(max_digits=15, decimal_places=2)
    order_count = models.IntegerField()

    class Meta:
        managed = False   # Django won't create/modify this table
        db_table = "mv_revenue_by_region"

# Access like a normal model:
top_regions = RevenueByRegionView.objects.order_by("-revenue")[:10]
```

---

## 20. Scaling ORM in High-Traffic Systems

### 📖 Concept

When your system handles millions of rows and thousands of concurrent requests, ORM usage patterns that work at small scale become bottlenecks. These are the patterns that distinguish senior engineers who've operated production systems.

### 💻 Code Examples

```python
# ============================================================
# 1. Use .only() and .defer() to reduce column fetching
# ============================================================

# BAD: SELECT * — fetches large text columns unnecessarily
products = Product.objects.filter(status="active")

# GOOD: Fetch only what you need
products = Product.objects.only(
    "id", "sku", "name", "price", "stock"
).filter(status="active")

# GOOD: Defer expensive columns
products = Product.objects.defer("description", "metadata").filter(status="active")

# BEST for read-only APIs: .values() — returns dicts, skips model instantiation
products = Product.objects.filter(
    status="active"
).values("id", "sku", "name", "price")
# 5-20x faster than model instantiation for large result sets


# ============================================================
# 2. Iterator() for large QuerySet processing — memory control
# ============================================================

# BAD: Loads entire queryset into memory
for product in Product.objects.all():   # 1M products = OOM
    process_product(product)

# GOOD: Stream in chunks using iterator()
for product in Product.objects.iterator(chunk_size=2000):
    process_product(product)
# Fetches 2000 rows at a time, releases memory per chunk


# ============================================================
# 3. Pagination for API endpoints — cursor-based for large datasets
# ============================================================

# Offset pagination — breaks at large offsets (LIMIT 25 OFFSET 100000 is slow!)
# Bad for large, frequently-updated datasets

# Cursor pagination — always fast, but one-directional
class CursorPaginatedProductView:
    def get(self, request):
        cursor = request.query_params.get("cursor")
        page_size = 25

        qs = Product.objects.filter(
            status="active"
        ).order_by("-created_at", "-id")

        if cursor:
            # Decode cursor: (created_at, id)
            cursor_date, cursor_id = decode_cursor(cursor)
            qs = qs.filter(
                Q(created_at__lt=cursor_date) |
                Q(created_at=cursor_date, id__lt=cursor_id)
            )

        products = list(qs[:page_size + 1])   # Fetch one extra to detect next page
        has_next = len(products) > page_size
        products = products[:page_size]

        next_cursor = None
        if has_next and products:
            last = products[-1]
            next_cursor = encode_cursor(last.created_at, last.id)

        return Response({
            "results": ProductSerializer(products, many=True).data,
            "next_cursor": next_cursor,
        })


# ============================================================
# 4. Connection pooling — critical for high concurrency
# ============================================================
# Use PgBouncer or django-db-geventpool

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "CONN_MAX_AGE": 0,  # Use 0 with PgBouncer (PgBouncer manages the pool)
        # With PgBouncer in transaction mode, set CONN_MAX_AGE=0
        # Without PgBouncer, set CONN_MAX_AGE=60 for persistent connections
    }
}


# ============================================================
# 5. Read replica for heavy analytics queries
# ============================================================
class AnalyticsService:
    def get_sales_report(self, date_from, date_to) -> list[dict]:
        """Heavy aggregation → run on replica to protect primary."""
        return (
            Order.objects
            .using("replica_1")   # Explicit replica for analytics
            .filter(
                status="delivered",
                created_at__range=(date_from, date_to),
            )
            .values("customer__country")
            .annotate(
                revenue=Sum("total_price"),
                order_count=Count("id"),
            )
            .order_by("-revenue")
        )


# ============================================================
# 6. Bulk operations — never loop + save for large datasets
# ============================================================
from django.db import transaction

def sync_product_prices(price_data: list[dict]) -> None:
    """
    Update thousands of product prices efficiently.
    Input: [{"sku": "ABC-001", "price": 99.99}, ...]
    """
    # Fetch existing products in one query
    skus = [d["sku"] for d in price_data]
    products = {p.sku: p for p in Product.objects.filter(sku__in=skus)}

    # Prepare updates
    to_update = []
    for data in price_data:
        product = products.get(data["sku"])
        if product:
            product.price = data["price"]
            to_update.append(product)

    # Single bulk update
    with transaction.atomic():
        Product.objects.bulk_update(
            to_update,
            fields=["price"],
            batch_size=500,
        )
```

---

# ⚠️ Common Pitfalls

---

## The Top 10 Django ORM Mistakes in Production

---

### 1. Calling `.count()` or `len()` when `.exists()` is enough

```python
# BAD
if Product.objects.filter(status="active").count() > 0:
    ...
# Better
if Product.objects.filter(status="active").exists():
    ...
```

---

### 2. Forgetting `update_fields` in `.save()`

```python
# BAD — overwrites every column, risks losing concurrent updates
product.price = 99.99
product.save()

# GOOD — only updates changed fields
product.price = 99.99
product.save(update_fields=["price", "updated_at"])
```

---

### 3. Modifying a QuerySet you're iterating

```python
# BAD — unexpected behavior
for product in Product.objects.filter(stock=0):
    product.status = "archived"
    product.save()    # save() may alter the iteration

# GOOD — collect IDs first
ids = list(Product.objects.filter(stock=0).values_list("id", flat=True))
Product.objects.filter(id__in=ids).update(status="archived")
```

---

### 4. Using `.get()` without try/except

```python
# BAD — crashes if product doesn't exist
product = Product.objects.get(sku="INVALID")

# GOOD
try:
    product = Product.objects.get(sku="INVALID")
except Product.DoesNotExist:
    return Response({"error": "Not found"}, status=404)

# BETTER — use get_object_or_404 in views
from django.shortcuts import get_object_or_404
product = get_object_or_404(Product, sku="INVALID")
```

---

### 5. Filtering prefetched data with `.filter()` (breaks cache)

```python
# BAD — fires new query despite prefetch
orders = Order.objects.prefetch_related("items")
for order in orders:
    expensive = order.items.filter(unit_price__gt=100)   # Cache miss!

# GOOD — use to_attr with filtered Prefetch
from django.db.models import Prefetch
orders = Order.objects.prefetch_related(
    Prefetch("items", queryset=OrderItem.objects.filter(unit_price__gt=100), to_attr="expensive_items")
)
for order in orders:
    expensive = order.expensive_items   # From cache
```

---

### 6. Creating objects in a loop

```python
# BAD — N INSERT statements
for sku in sku_list:
    Product.objects.create(sku=sku, name=f"Product {sku}", price=9.99)

# GOOD — single INSERT
Product.objects.bulk_create([
    Product(sku=sku, name=f"Product {sku}", price=9.99)
    for sku in sku_list
], batch_size=500)
```

---

### 7. Not handling `MultipleObjectsReturned` with `.get()`

```python
# This raises MultipleObjectsReturned if data is dirty
product = Product.objects.get(name="Laptop")

# Safe alternative
product = Product.objects.filter(name="Laptop").first()
```

---

### 8. Using `.all()` unnecessarily in views

```python
# Redundant — .all() returns the same queryset
Product.objects.all().filter(status="active")

# Direct
Product.objects.filter(status="active")
```

---

### 9. Hardcoding database names (breaks test isolation)

```python
# BAD
Product.objects.using("replica_1").filter(...)

# GOOD — use the router or a setting
DB_REPLICA = getattr(settings, "ANALYTICS_DB", "default")
Product.objects.using(DB_REPLICA).filter(...)
```

---

### 10. Ignoring database-level constraints

```python
# Python-level validation is not enough for concurrency
def save(self):
    if Product.objects.filter(sku=self.sku).exists():
        raise ValueError("SKU must be unique")  # Race condition between check and insert!
    super().save()

# GOOD — let the database enforce constraints
class Product(models.Model):
    sku = models.CharField(max_length=100, unique=True)  # DB-level unique constraint
    # Catch IntegrityError in calling code
```

---

# ⚡ Performance Optimization Cheatsheet

---

## Quick Reference

```
Query Reduction
───────────────
✅ Use select_related()      — eliminates FK N+1 (JOIN)
✅ Use prefetch_related()    — eliminates reverse FK/M2M N+1
✅ Use Prefetch(to_attr=)    — custom prefetch with filter
✅ Use annotate()            — move Python math to SQL
✅ Use exists()              — over count() > 0 or bool(qs)
✅ Use values()/values_list()— skip model instantiation

Data Volume Reduction
─────────────────────
✅ Use only()/defer()        — fetch fewer columns
✅ Paginate always           — never return unbounded querysets
✅ Use iterator()            — for bulk processing (memory control)
✅ Add indexes               — for every WHERE/ORDER BY column
✅ Use partial indexes       — smaller, faster for filtered queries

Write Optimization
──────────────────
✅ Use bulk_create()         — batch inserts (500 rows per batch)
✅ Use bulk_update()         — batch updates
✅ Use update() on queryset  — single SQL UPDATE
✅ Use F() expressions       — atomic in-DB arithmetic
✅ Use update_fields=        — limit columns on save()

Caching
───────
✅ Cache slow aggregations   — revenue totals, category counts
✅ Use on_commit() for tasks — avoid phantom task runs
✅ Version your cache keys   — "v2" for easy cache busting
✅ Use select_for_update()   — row locks over application locks

Monitoring
──────────
✅ Log slow queries          — SET log_min_duration_statement = 100ms
✅ Use EXPLAIN ANALYZE       — understand query plans
✅ assertNumQueries in tests — prevent query regression
✅ Track p95/p99 latency     — not just averages
```

---

# 🚀 Rapid Revision — Interview Prep

---

## 50 Quick-Fire Questions & Answers

---

**1. What is lazy evaluation in QuerySets?**
QuerySets don't hit the database until explicitly evaluated (iteration, slicing, `list()`, `bool()`, etc.).

**2. What's the difference between `filter()` and `exclude()`?**
`filter()` = `WHERE condition`. `exclude()` = `WHERE NOT condition`. They're equivalent to `~Q()`.

**3. When does `select_related` NOT work?**
On ManyToMany or reverse ForeignKey relationships — use `prefetch_related` for those.

**4. What does `Prefetch(to_attr=)` do?**
Stores the prefetched queryset as a list attribute instead of a manager, enabling access without re-querying.

**5. What's the N+1 problem?**
Fetching N objects and then executing 1 additional query per object = N+1 total queries.

**6. How do you detect N+1 in development?**
Django Debug Toolbar, `assertNumQueries` in tests, or enabling `django.db.backends` logging.

**7. What is `F()` expression used for?**
Referencing field values at the database level, avoiding Python-level reads for math operations.

**8. Difference between `aggregate()` and `annotate()`?**
`aggregate()` returns a single dict summary. `annotate()` adds a computed column to each row.

**9. What does `Q()` allow you to do?**
Build complex `OR`, `AND`, `NOT` conditions that aren't possible with chained `.filter()`.

**10. Why use `transaction.atomic()`?**
Ensures all DB operations inside the block either all commit or all rollback.

**11. What is `select_for_update()`?**
Locks selected rows with `SELECT ... FOR UPDATE` until the transaction ends.

**12. How do you prevent a `save()` from updating all fields?**
Use `product.save(update_fields=["field1", "field2"])`.

**13. What is `get_or_create()` and when does it risk race conditions?**
It's safe because it uses `get_or_create` in a single atomic operation at the DB level (uses INSERT ... ON CONFLICT or similar). But in some DB configurations, two simultaneous calls can both attempt INSERT.

**14. What's the difference between `null=True` and `blank=True`?**
`null=True` = store SQL NULL. `blank=True` = allow empty in forms/serializers.

**15. Why avoid `null=True` on CharField?**
It creates two possible "empty" states: `NULL` and `""` — use only `blank=True, default=""`.

**16. What does `on_delete=PROTECT` do?**
Raises `ProtectedError` if you try to delete the parent when children reference it.

**17. When would you use `on_delete=SET_NULL`?**
When the child record should survive parent deletion with the FK set to `NULL`.

**18. How does `values()` improve performance?**
Returns dicts instead of model instances — skips model instantiation overhead, faster for read-only use.

**19. What does `only()` do vs `values()`?**
`only()` returns model instances with only specified fields loaded. `values()` returns dicts.

**20. When is `iterator()` useful?**
Processing large QuerySets without loading everything into memory. Use `chunk_size` parameter.

**21. What is `Subquery()`?**
Allows embedding a QuerySet as a subquery in a larger query, often with `OuterRef`.

**22. What is `OuterRef()`?**
References a field from the outer query inside a subquery (equivalent to SQL correlated subquery).

**23. What does `Exists()` do?**
Creates an `EXISTS (SELECT 1 ...)` SQL expression — more efficient than `Count() > 0`.

**24. What's a database router in Django?**
A class that tells Django which database to use for reads/writes for specific models.

**25. Why use UUID primary keys?**
Prevents leaking business metrics, safe for distributed generation, opaque in URLs.

**26. What is a through model in M2M?**
An explicit intermediary model for a ManyToMany relationship that holds extra attributes.

**27. What does `distinct()` do and when is it needed?**
Adds `DISTINCT` to SQL, preventing duplicate rows when joins cause row multiplication.

**28. How does `bulk_create()` differ from `create()` in a loop?**
`bulk_create()` sends a single INSERT statement; loop creates N statements. Drastically faster.

**29. What's the difference between `delete()` on a queryset vs instance?**
Queryset delete: single SQL, no signals. Instance delete: fires `pre_delete`/`post_delete` signals.

**30. When does `queryset.update()` NOT update `auto_now` fields?**
Always — `auto_now` fields require `.save()` to update automatically. Use `update(updated_at=timezone.now())` explicitly.

**31. What is a soft delete?**
Setting a `deleted_at` timestamp instead of physically removing rows, preserving audit history.

**32. How do you implement a custom default manager?**
Override `get_queryset()` in a `Manager` subclass and assign it as `objects` on the model.

**33. What is `base_manager_name` in Meta?**
Specifies which manager Django uses for related object access — important when default manager filters.

**34. How do custom QuerySets enable the DRY principle?**
Chainable methods like `.available().in_category("electronics")` encapsulate reusable query logic.

**35. What is `ExpressionWrapper` used for?**
Wraps arithmetic/logical expressions that involve mixed field types, specifying the `output_field`.

**36. When should you use `Case/When`?**
For conditional annotations — equivalent to SQL `CASE WHEN ... THEN ... ELSE ... END`.

**37. What are window functions in Django ORM?**
The `Window()` expression with functions like `Rank()`, `Sum()`, `Lead()`, `Lag()` over partitions.

**38. How does `TruncMonth` work in annotations?**
Truncates a datetime to month precision, enabling `GROUP BY month` reporting.

**39. What is connection pooling and why does it matter?**
Reuses DB connections instead of opening new ones per request — critical for high concurrency.

**40. What is `CONN_MAX_AGE`?**
The number of seconds Django keeps a DB connection alive. Set to 0 with PgBouncer, 60+ otherwise.

**41. How do you run a query on a specific database?**
Use `.using("database_name")` on the queryset.

**42. What is a partial index and when would you use one?**
An index with a `WHERE` condition — smaller and faster for queries that filter by that condition.

**43. What does `EXPLAIN ANALYZE` tell you?**
The query execution plan, actual vs estimated rows, time per step, index usage, and join strategy.

**44. What is cache stampede / thundering herd?**
Multiple requests simultaneously find an empty cache and all query the DB at once. Prevent with locks (`cache.add()`).

**45. Why use `transaction.on_commit()`?**
To ensure side effects (emails, Celery tasks) only execute if the transaction actually commits.

**46. What is the risk of `select_for_update()` without consistent ordering?**
Deadlock — two transactions lock rows in different orders and wait for each other indefinitely.

**47. What is multi-tenancy in SaaS?**
Serving multiple customers (tenants) from a single application instance, with data isolation between them.

**48. When would you choose raw SQL over ORM?**
Recursive CTEs, complex window functions, bulk COPY operations, query plan hints, or DB-specific features.

**49. What is `managed = False` on a model?**
Tells Django not to create/modify the table via migrations — used for database views or legacy tables.

**50. What is the best way to test query count?**
`self.assertNumQueries(N, callable)` in Django's `TestCase` — ensures no query regressions.

---

## Key Mental Models for Interviews

```
ORM EVALUATION CHECKLIST (when reviewing any Django view/service):

1. N+1? → select_related / prefetch_related
2. All columns? → only() / values() / defer()
3. Unbounded queryset? → Paginate
4. Python math on DB data? → F() expression / annotate()
5. Multiple queries for stats? → aggregate() / annotate()
6. Race condition on write? → F() + select_for_update() + atomic()
7. Side effects in transaction? → on_commit()
8. Frequently read, rarely changed? → Cache with Redis
9. Large dataset processing? → iterator()
10. Complex reporting? → Raw SQL / DB view / replica
```

---

## Final Advice from a Senior Engineer

> **"The ORM is not magic — it's a translation layer. Master both SQL and the ORM, and you'll know which to reach for. The best Django engineers are the ones who can look at a slow page and immediately visualize the SQL being generated."**

The questions interviewers really want answered are:

1. **Can you identify performance problems** before they hit production?
2. **Do you understand the trade-offs** — not just the happy path?
3. **Have you operated production systems** — or do you only know tutorials?

The content in this guide gives you the vocabulary and pattern library to answer all three convincingly.

---

## Resources for Continued Learning

- [Django ORM Official Docs](https://docs.djangoproject.com/en/stable/topics/db/queries/)
- [Django Performance Optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
- [Two Scoops of Django](https://www.feldroy.com/books/two-scoops-of-django-3-x) — Production patterns
- [PostgreSQL EXPLAIN docs](https://www.postgresql.org/docs/current/sql-explain.html) — Understanding query plans
- [django-debug-toolbar](https://django-debug-toolbar.readthedocs.io/) — Essential development tool
- [django-silk](https://github.com/jazzband/django-silk) — Production-safe profiling

---

*Last updated: 2025 | Django 4.x+ | PostgreSQL 15+*

*⭐ Star this repo if it helped you land that interview!*
