# Advanced Django Model Design for Scale

*A Principal Engineer's guide to building scalable, performant, and maintainable Django models on PostgreSQL.*

---

## Table of Contents

1. [Architectural Patterns](#1-architectural-patterns)
2. [Database Optimization](#2-database-optimization)
3. [Field Selection for Scale](#3-field-selection-for-scale)
4. [Relationship Management](#4-relationship-management)
5. [Zero-Downtime Migrations](#5-zero-downtime-migrations)
6. [Top Three Non-Negotiable Code Review Rules](#6-top-three-non-negotiable-code-review-rules)

---

## 1. Architectural Patterns

### 1.1 Fat Models vs. Service Layers
A "Model" in Django represents a piece of data, like an Invoice. The text argues that an Invoice should be smart enough to handle its own basic, internal math and rules.

Think of it like a physical paper invoice.

- What it should do: If you know the subtotal and the tax rate, the invoice should be able to tell you its own total. If you want to stamp it "PAID", the invoice should first check to make sure it was actually sent to the customer.

- The Rule: If the logic is just doing math or checking rules using only its own data, it belongs in the Model.

The Django community's "Fat Models, Thin Views" mantra is a good *starting* heuristic, but it breaks down at scale. The right question isn't "fat or thin" — it's *"who owns this logic, and how many models does it touch?"*

**Lean into Fat Models when** logic concerns a single model instance or its closely-related data, and is intrinsic to what the object *is*. Invariants, derived properties, and state transitions belong on the model.

```python
class Invoice(models.Model):
    subtotal = models.DecimalField(max_digits=12, decimal_places=2)
    tax_rate = models.DecimalField(max_digits=5, decimal_places=4)
    status = models.CharField(max_length=20, choices=InvoiceStatus.choices)

    @property
    def total(self) -> Decimal:
        # Intrinsic, single-object computation -> belongs on the model.
        return (self.subtotal * (1 + self.tax_rate)).quantize(Decimal("0.01"))

    def mark_paid(self) -> None:
        # A guarded state transition the model itself should enforce.
        if self.status != InvoiceStatus.SENT:
            raise InvalidTransition(f"Cannot pay an invoice in state {self.status}")
        self.status = InvoiceStatus.PAID
```
A "Service Layer" is a separate file or function that acts like a project manager. It coordinates multiple different things at once.

Going back to our physical invoice—the piece of paper cannot call the bank, charge a credit card, write a note in the accountant's ledger, and mail a receipt to the customer.

- What it should do: You need a "Manager" (the Service Layer) to orchestrate this. The manager grabs the invoice, talks to the credit card company, marks the invoice as paid, and tells the mailroom to send an email.

- The Rule: If the code needs to talk to the outside world (send emails, process payments) or coordinate multiple different Models at the same time, it belongs in a Service Layer.

**Extract into a Service Layer when** logic orchestrates *multiple* models, has side effects (sending email, charging a card, hitting a queue), needs transaction boundaries spanning several aggregates, or represents a *business process* rather than an object property.

```python
# services/billing.py
@transaction.atomic
def charge_invoice(*, invoice: Invoice, payment_method: PaymentMethod) -> Charge:
    """Orchestrates several models + external I/O -> a service, not a model method."""
    invoice = Invoice.objects.select_for_update().get(pk=invoice.pk)

    charge_result = payment_gateway.charge(
        amount=invoice.total, source=payment_method.token
    )
    charge = Charge.objects.create(
        invoice=invoice, amount=invoice.total, gateway_id=charge_result.id
    )
    invoice.mark_paid()            # delegate the intrinsic transition to the model
    invoice.save(update_fields=["status"])

    LedgerEntry.objects.create(invoice=invoice, charge=charge, amount=invoice.total)
    enqueue_receipt_email.delay(invoice.pk)   # side effect, off the request path
    return charge
```

> **Rule of thumb:** A model method should never `import requests`, send an email, or call `.delay()`. The moment it crosses an aggregate boundary or does I/O, it's a service.

**Anti-pattern — the God Model:** A 2,000-line `User` model with `send_invoice()`, `calculate_shipping()`, and `sync_to_salesforce()` methods. It becomes untestable (every test needs the whole world mocked) and a merge-conflict magnet.

**Anti-pattern — anemic models + everything in views:** Validation logic copy-pasted across five views. The model becomes a dumb bag of columns and invariants drift.

An "anti-pattern" is a common bad habit in programming. The text points out two opposite extremes to avoid:

    - The "God Model" (Doing Too Much): This is when you force the Model to do everything. Imagine your User model knows how to log in, but also calculates shipping rates, sends marketing emails, and talks to Salesforce. It becomes a massive, 2,000-line mess that is impossible to update without breaking something.

    - The "Anemic Model" (Doing Too Little): This is when your Model is completely dumb. It's just a blank piece of paper. It doesn't even know how to calculate its own total. Because the model is useless, your "Views" (the web pages) have to do all the math themselves. If you have five different web pages that show the invoice, you have to copy-paste that math five times. If the tax rate changes, you have to remember to update it in five different places.

### 1.2 Custom Managers and Chainable QuerySets

The most common mistake is putting reusable query logic on a custom `Manager`. Manager methods **don't chain**. The correct pattern is to put logic on a custom `QuerySet` and promote it to the manager via `.as_manager()`.

```python
class ArticleQuerySet(models.QuerySet):
    def published(self):
        return self.filter(status=ArticleStatus.PUBLISHED, published_at__lte=timezone.now())

    def by_author(self, author):
        return self.filter(author=author)

    def with_engagement(self):
        # Annotate once, reuse everywhere — avoids ad-hoc aggregation in views.
        return self.annotate(
            comment_count=models.Count("comments", distinct=True),
            like_count=models.Count("likes", distinct=True),
        )

    def popular(self, threshold=100):
        return self.with_engagement().filter(like_count__gte=threshold)


class Article(models.Model):
    # Promote the QuerySet to a Manager. Every method is now chainable.
    objects = ArticleQuerySet.as_manager()
```

Now queries compose fluently and read like the domain:

```python
Article.objects.published().by_author(user).popular().order_by("-published_at")
```

**When you need both a custom Manager and a custom QuerySet** (e.g., a manager-level default filter plus chainable methods), use `Manager.from_queryset()`:

```python
class ActiveManager(models.Manager.from_queryset(ArticleQuerySet)):
    def get_queryset(self):
        return super().get_queryset().filter(is_deleted=False)


class Article(models.Model):
    objects = models.Manager()          # always keep an unfiltered default first
    active = ActiveManager()            # filtered, still chainable
```

> **Critical ordering rule:** The first manager defined becomes `_default_manager`, used by migrations, related-object access, and `dumpdata`. **Never** make a *filtering* manager your first/default manager — it will silently hide rows from the ORM internals and admin. Keep a plain `objects = models.Manager()` first.

### 1.3 Abstract Base Classes vs. Multi-Table Inheritance (MTI)

| Aspect | Abstract Base Class | Multi-Table Inheritance |
|---|---|---|
| DB tables | None for the base; fields copied into each child | One table per model in the chain |
| Queries | Single table, no joins | **Implicit JOIN** on every access |
| Use case | Sharing fields/methods (mixins) | True "is-a" with a queryable parent |
| Cost | Free | A join per row, per access |

**Default to abstract base classes.** They are the workhorse for sharing common fields with zero query overhead.

```python
class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True          # no table is created for this model
        get_latest_by = "created_at"


class Product(TimeStampedModel):   # created_at / updated_at become columns ON product
    name = models.CharField(max_length=200)
```

**Avoid MTI in hot paths.** It looks elegant but every child access triggers a hidden join, and saving a child writes to two tables. It's a frequent, silent source of N+1 and write amplification.

```python
# MTI: looks clean, performs poorly at scale
class Place(models.Model):
    name = models.CharField(max_length=100)

class Restaurant(Place):          # creates restaurant table with a OneToOne to place
    serves_pizza = models.BooleanField(default=False)

# Restaurant.objects.all()  ->  every iteration joins place + restaurant
```

If you need polymorphism, prefer **explicit composition** (a `OneToOneField` you control and can `select_related`) or a proxy model + type field, rather than MTI's implicit joins.

---

## 2. Database Optimization

### 2.1 Index Strategy: B-Tree vs. GIN vs. BRIN

The default `db_index=True` creates a **B-Tree** index — perfect for equality and range queries on high-cardinality scalar columns (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`). But B-Tree is the wrong tool for several common Postgres workloads.

```python
from django.contrib.postgres.indexes import GinIndex, BrinIndex, BTreeIndex

class Event(models.Model):
    name = models.CharField(max_length=200)
    tags = ArrayField(models.CharField(max_length=50))      # array
    metadata = models.JSONField()                           # jsonb
    search_vector = SearchVectorField(null=True)            # full-text
    created_at = models.DateTimeField(auto_now_add=True)    # naturally ordered

    class Meta:
        indexes = [
            # B-Tree: high-cardinality lookups & ordering
            models.Index(fields=["name"]),

            # GIN: "does this contain X" — arrays, JSONB, full-text search.
            # Use for __contains, __contained_by, JSONB key/path, @@ FTS.
            GinIndex(fields=["tags"]),
            GinIndex(fields=["metadata"]),
            GinIndex(fields=["search_vector"]),

            # BRIN: huge, append-only, naturally-ordered tables (time-series).
            # Tiny on disk; only helps when physical order ~ logical order.
            BrinIndex(fields=["created_at"]),
        ]
```

**Decision guide:**

- **B-Tree (default):** Foreign keys, status fields, dates you filter/sort on, anything with `=`/range semantics. Your bread and butter.
- **GIN:** "Contains" queries against composite values — `ArrayField`, `JSONField`, `hstore`, and `SearchVectorField`. Writes are slower (the index is larger and more expensive to update), reads on containment are dramatically faster. Add `fastupdate` / `gin_pending_list_limit` tuning for write-heavy tables.
- **BRIN:** Massive, append-only tables where the indexed column correlates with physical row order (e.g., an `created_at` on a log/events table that only grows). A BRIN index can be *thousands of times smaller* than the equivalent B-Tree while still pruning huge ranges. Useless on randomly-ordered data.

**Partial and covering indexes** eliminate dead weight and extra heap lookups:

```python
class Order(models.Model):
    class Meta:
        indexes = [
            # Partial: index only the rows you actually query.
            # Tiny index that the 'pending orders' dashboard hits constantly.
            models.Index(
                fields=["created_at"],
                condition=models.Q(status="pending"),
                name="order_pending_created_idx",
            ),
            # Covering (INCLUDE): satisfy the query from the index alone, no heap fetch.
            models.Index(
                fields=["customer_id"],
                include=["total", "status"],
                name="order_customer_covering_idx",
            ),
        ]
```

**Anti-patterns:**
- A B-Tree on a `JSONField` (it indexes the whole blob as one opaque value — useless for key lookups; you wanted GIN).
- Indexing a low-cardinality boolean with a plain index (use a *partial* index instead, or none).
- Redundant indexes: a B-Tree on `(a, b)` already serves queries on `a` alone — don't also create one on `(a)`.

### 2.2 Database-Level Constraints over Application Validation

Application-level validation (`clean()`, serializers, forms) is for *UX* — friendly error messages. It is **not** a guarantee. Bulk operations, raw SQL, `update()`, data migrations, and a second app touching the same DB all bypass it. **Integrity invariants belong in the database**, declared in `Meta.constraints`.

```python
class Subscription(models.Model):
    user = models.ForeignKey("User", on_delete=models.CASCADE)
    plan = models.CharField(max_length=20)
    seats = models.PositiveIntegerField()
    starts_at = models.DateTimeField()
    ends_at = models.DateTimeField()
    discount_pct = models.DecimalField(max_digits=5, decimal_places=2, default=0)

    class Meta:
        constraints = [
            # 1. Uniqueness with a condition (one ACTIVE subscription per user).
            models.UniqueConstraint(
                fields=["user"],
                condition=models.Q(plan="active"),
                name="uniq_active_subscription_per_user",
            ),
            # 2. Value range enforced by the DB, not just the form.
            models.CheckConstraint(
                check=models.Q(discount_pct__gte=0) & models.Q(discount_pct__lte=100),
                name="discount_pct_between_0_and_100",
            ),
            # 3. Cross-field invariant the ORM can't express on a single field.
            models.CheckConstraint(
                check=models.Q(ends_at__gt=models.F("starts_at")),
                name="subscription_ends_after_start",
            ),
            models.CheckConstraint(
                check=models.Q(seats__gte=1),
                name="subscription_at_least_one_seat",
            ),
        ]
```

For overlap prevention (e.g., no two bookings of the same room at the same time), reach for a Postgres `ExclusionConstraint` with a range type — something no amount of `clean()` can race-safely guarantee:

```python
from django.contrib.postgres.constraints import ExclusionConstraint
from django.contrib.postgres.fields import RangeOperators

class Booking(models.Model):
    room = models.ForeignKey("Room", on_delete=models.CASCADE)
    period = DateTimeRangeField()

    class Meta:
        constraints = [
            ExclusionConstraint(
                name="exclude_overlapping_bookings",
                expressions=[
                    ("room", RangeOperators.EQUAL),
                    ("period", RangeOperators.OVERLAPS),
                ],
            ),
        ]
```

> Keep app-level validation *too* — for the friendly message — but treat the DB constraint as the source of truth. Catch `IntegrityError` and translate it into a user-facing error.

---

## 3. Field Selection for Scale

### 3.1 Primary Keys: `BigAutoField` vs. `UUIDField`

This is a genuine trade-off, not a one-size answer.

| | `BigAutoField` (default) | `UUIDField` |
|---|---|---|
| Size | 8 bytes | 16 bytes |
| Index locality | Excellent (monotonic inserts) | Poor with random UUIDv4 (index fragmentation, page splits) |
| Enumeration risk | Exposes counts/order in URLs | Opaque |
| Client-side generation | No (needs a round-trip) | Yes (generate before insert) |
| Cross-system merging | Collision-prone | Globally unique |

**Default to `BigAutoField`.** It's compact, has perfect index locality, and Django uses it automatically via `DEFAULT_AUTO_FIELD`. Set it project-wide:

```python
# settings.py
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

**Reach for UUIDs** when you must hide row counts, generate IDs client-side, or merge data across databases. If you do, **avoid random UUIDv4 as a clustered/primary key** — its randomness destroys B-Tree insert locality and bloats the index. Use a time-ordered UUID (UUIDv7) so inserts stay roughly sequential:

```python
class ApiToken(models.Model):
    # UUIDv7 is time-ordered: keeps index locality while staying opaque & non-enumerable.
    id = models.UUIDField(primary_key=True, default=uuid7, editable=False)
```

A common best-of-both pattern: keep an internal `BigAutoField` PK for joins/locality, and expose a separate indexed `UUIDField` (or short slug) externally.

```python
class Order(models.Model):
    id = models.BigAutoField(primary_key=True)                      # internal, fast joins
    public_id = models.UUIDField(default=uuid4, unique=True, editable=False)  # external API
```

### 3.2 Money: never use `FloatField`

Floating point cannot represent `0.10` exactly; it *will* cost you real money in rounding drift. Use `DecimalField`, and store the currency explicitly.

```python
class LineItem(models.Model):
    # 19 total digits, 4 after the decimal — handles large sums + sub-cent precision.
    amount = models.DecimalField(max_digits=19, decimal_places=4)
    currency = models.CharField(max_length=3, default="USD")  # ISO 4217
```

For multi-currency apps, consider the `django-money` package (`MoneyField`), which pairs amount + currency and prevents accidental cross-currency arithmetic. Whatever you choose: **do all arithmetic in `Decimal`, never `float`.** Some teams store integer minor units (cents) instead — also valid, just be consistent.

### 3.3 Auditing & Soft-Deletion

Soft-deletion (flagging rows instead of removing them) is the standard auditing pattern, but the naive implementation has two sharp edges: it breaks `unique` constraints and it can silently leak "deleted" rows.

```python
class SoftDeleteQuerySet(models.QuerySet):
    def alive(self):
        return self.filter(deleted_at__isnull=True)

    def dead(self):
        return self.filter(deleted_at__isnull=False)

    def delete(self):
        # Override bulk delete to soft-delete instead of issuing SQL DELETE.
        return self.update(deleted_at=timezone.now())

    def hard_delete(self):
        return super().delete()


class SoftDeleteModel(TimeStampedModel):
    deleted_at = models.DateTimeField(null=True, blank=True, db_index=True)

    # Keep BOTH managers: a default that shows everything, plus a filtered one.
    objects = SoftDeleteQuerySet.as_manager()

    class Meta:
        abstract = True

    def delete(self, using=None, keep_parents=False):
        self.deleted_at = timezone.now()
        self.save(update_fields=["deleted_at"])
```

Two non-obvious requirements:

1. **Uniqueness must become conditional.** A plain `unique=True` will reject a *new* row whose value matches a soft-deleted one. Use a partial unique constraint scoped to live rows:

   ```python
   class Account(SoftDeleteModel):
       email = models.EmailField()

       class Meta:
           constraints = [
               models.UniqueConstraint(
                   fields=["email"],
                   condition=models.Q(deleted_at__isnull=True),
                   name="uniq_email_when_alive",
               ),
           ]
   ```

2. **Beware the default-manager trap.** If your *default* manager filters out deleted rows, related lookups, admin, and migrations can behave surprisingly (and cascades won't see the rows). The safer convention shown above is: the default manager returns *all* rows; callers explicitly chain `.alive()`. Pick one convention and document it loudly — silent filtering on the default manager is a frequent source of "where did my row go?" bugs.

For full audit trails (who changed what, when), use a dedicated library like `django-simple-history` or `django-auditlog` rather than hand-rolling history tables.

### 3.4 `Choices` Classes

Always use Django's enumeration types (`TextChoices` / `IntegerChoices`) instead of loose tuples or magic strings. They give you type-safe references, a single source of truth, and clean `.label` access.

```python
class OrderStatus(models.TextChoices):
    DRAFT = "draft", "Draft"
    PENDING = "pending", "Pending Payment"
    PAID = "paid", "Paid"
    SHIPPED = "shipped", "Shipped"
    CANCELLED = "cancelled", "Cancelled"


class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.DRAFT,
    )

# Reference symbolically — never hardcode "paid" across the codebase:
Order.objects.filter(status=OrderStatus.PAID)
order.status = OrderStatus.SHIPPED
```

Prefer `TextChoices` over `IntegerChoices`: stored values are self-documenting in the database, and you avoid the "what does status `3` mean?" archaeology during incidents.

---

## 4. Relationship Management

### 4.1 Killing N+1 at the Source

N+1 is the single most common Django performance killer. The fix lives in the *query*, but good model design makes the fix natural.

```python
# N+1: one query for orders, then one per order for its customer (and again for items).
for order in Order.objects.all():
    print(order.customer.name)          # +1 query each
    print(order.items.count())          # +1 query each

# Fixed: select_related for FK/O2O (SQL JOIN), prefetch_related for reverse/M2M (2nd query).
orders = (
    Order.objects
    .select_related("customer")                 # forward FK -> JOIN
    .prefetch_related("items__product")         # reverse FK / M2M -> batched IN query
)
```

Encode the safe access pattern as a QuerySet method so callers can't forget it:

```python
class OrderQuerySet(models.QuerySet):
    def with_details(self):
        return self.select_related("customer").prefetch_related("items__product")
```

Use `Prefetch` objects to filter or order the prefetched set, and `only()`/`defer()` to trim columns on wide tables:

```python
from django.db.models import Prefetch

Order.objects.prefetch_related(
    Prefetch("items", queryset=LineItem.objects.filter(quantity__gt=0).select_related("product"))
)
```

> Add `django-debug-toolbar` in dev and a query-count assertion in CI (`assertNumQueries`). N+1 should fail the build, not surface in production APM.

### 4.2 `related_name` and `related_query_name`

Always set `related_name` explicitly. The default (`<model>_set`) is ugly, collides when a model has two FKs to the same target, and makes reverse queries read poorly.

```python
class Comment(models.Model):
    article = models.ForeignKey(
        "Article", on_delete=models.CASCADE, related_name="comments"
    )
    author = models.ForeignKey(
        "User", on_delete=models.CASCADE, related_name="comments_written"
    )
    # Two FKs to the same model REQUIRE distinct related_names:
    reviewed_by = models.ForeignKey(
        "User", on_delete=models.SET_NULL, null=True, related_name="comments_reviewed"
    )

# Clean reverse access:
article.comments.all()
user.comments_written.all()
```

Use `related_name="+"` to disable the reverse accessor entirely when you'll never need it (saves Django from building unused descriptors and avoids namespace clutter).

### 4.3 Cascade Rules: `PROTECT` vs `CASCADE` vs others

`on_delete` is a **business rule about data lifecycle**, not a default to copy-paste. Choosing wrong either orphans data or silently deletes things you needed.

```python
class Order(models.Model):
    # PROTECT: you must NEVER lose financial records by deleting a customer.
    customer = models.ForeignKey("Customer", on_delete=models.PROTECT, related_name="orders")

class LineItem(models.Model):
    # CASCADE: a line item has no meaning without its order — delete with the parent.
    order = models.ForeignKey("Order", on_delete=models.CASCADE, related_name="items")

    # SET_NULL: keep the historical line item even if the product is discontinued.
    product = models.ForeignKey(
        "Product", on_delete=models.SET_NULL, null=True, related_name="line_items"
    )

class AuditEntry(models.Model):
    # SET_DEFAULT / SET(): preserve the audit row, point it at a sentinel.
    actor = models.ForeignKey(
        "User", on_delete=models.SET(get_sentinel_user), related_name="+"
    )
```

Decision guide:
- **`PROTECT`** — financial, legal, or referenced-elsewhere data. Forces an explicit decision; surfaces accidental deletes as errors.
- **`CASCADE`** — true child/owned data with no independent existence (line items, OAuth tokens, profile rows).
- **`SET_NULL`** — the FK is optional context you want to preserve historically (requires `null=True`).
- **`RESTRICT`** — like PROTECT but allows the delete if *another* cascade path also removes the row.

> **Default to `PROTECT` when unsure.** A blocked delete is a recoverable annoyance; a `CASCADE` that wipes a customer's entire order history is a catastrophe. `CASCADE` should be a deliberate statement of ownership, never the path of least resistance.

### 4.4 Through Models for M2M

The moment a many-to-many relationship needs *any* extra data on the relationship itself (joined-at, role, quantity), use an explicit `through` model. Adding it later is a painful migration.

```python
class Membership(models.Model):
    user = models.ForeignKey("User", on_delete=models.CASCADE)
    team = models.ForeignKey("Team", on_delete=models.CASCADE)
    role = models.CharField(max_length=20, choices=Role.choices)
    joined_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["user", "team"], name="uniq_user_team"),
        ]

class Team(models.Model):
    members = models.ManyToManyField("User", through="Membership", related_name="teams")
```

---

## 5. Zero-Downtime Migrations

The cardinal sin in production migrations is acquiring an `ACCESS EXCLUSIVE` lock on a large, hot table — it blocks reads *and* writes for the lock's duration. Several seemingly-innocent operations do exactly this on older Postgres or when done carelessly.

### 5.1 Adding a column with a default

On modern PostgreSQL (11+), adding a column with a *constant* default is fast (metadata-only). But adding a column with a **volatile** default (e.g., `default=uuid4`, `default=timezone.now`) forces a full table rewrite under an exclusive lock. The safe, version-agnostic pattern is three steps:

```python
# Step 1 — add the column as nullable, NO default. Fast, non-blocking.
class Migration(migrations.Migration):
    operations = [
        migrations.AddField(
            model_name="order",
            name="reference_code",
            field=models.CharField(max_length=32, null=True),
        ),
    ]
```

```python
# Step 2 — backfill in batches in a SEPARATE data migration (see 5.2).
#          Never backfill millions of rows in one UPDATE — it holds locks
#          and bloats WAL. Chunk it.
def backfill(apps, schema_editor):
    Order = apps.get_model("shop", "Order")
    qs = Order.objects.filter(reference_code__isnull=True)
    batch = 5000
    while qs.exists():
        ids = list(qs.values_list("pk", flat=True)[:batch])
        Order.objects.filter(pk__in=ids).update(reference_code=Func(...))  # or per-row
```

```python
# Step 3 — once backfilled, add the NOT NULL constraint.
# Use a validated CHECK / NOT VALID + VALIDATE pattern to avoid a full-table lock.
operations = [
    migrations.AlterField(
        model_name="order",
        name="reference_code",
        field=models.CharField(max_length=32),  # null=False
    ),
]
```

For the NOT NULL step on very large tables, add a `CHECK (reference_code IS NOT NULL) NOT VALID` constraint, then `VALIDATE CONSTRAINT` (which takes only a `SHARE UPDATE EXCLUSIVE` lock, allowing concurrent writes), rather than a direct `SET NOT NULL`.

### 5.2 Separate schema changes from data migrations

**Never** mix `AddField` and a data backfill in the same migration. Reasons:

- A data backfill inside a schema migration runs inside the schema-change transaction, *extending* how long locks are held.
- Schema migrations should be fast and reversible; data migrations are slow and often irreversible.
- Mixing them makes rollback all-or-nothing and breaks the "deploy code, then migrate" choreography.

The canonical sequence for an online change:

1. **Schema migration** — add nullable column (fast).
2. **Deploy code** that writes to *both* old and new state (dual-write), tolerating nulls.
3. **Data migration** — batched backfill (`RunPython`, with `elidable=True` and a no-op reverse).
4. **Schema migration** — tighten constraints (`NOT NULL`, unique) now that data is clean.
5. **Cleanup deploy** — remove dual-write / old field.

### 5.3 Indexes: always `CONCURRENTLY`

`CREATE INDEX` takes a write lock for the whole build. On a hot table that's an outage. Use Django's `AddIndexConcurrently` (from `django.contrib.postgres.operations`), which builds without blocking writes — but note it **cannot run inside a transaction**, so the migration must be marked `atomic = False`.

```python
from django.contrib.postgres.operations import AddIndexConcurrently

class Migration(migrations.Migration):
    atomic = False   # CONCURRENTLY cannot run in a transaction block
    operations = [
        AddIndexConcurrently(
            "event",
            models.Index(fields=["created_at"], name="event_created_idx"),
        ),
    ]
```

> **Tooling tip:** Adopt `django-pg-zero-downtime-migrations` or run `django-migration-linter` in CI. They mechanically catch the dangerous operations (unsafe defaults, blocking index creation, renames, type changes) before they reach production. Also set a `lock_timeout` and `statement_timeout` on the migration connection so a migration *fails fast* instead of queuing behind a lock and stalling the whole app.

**Other locking traps:** renaming a column/table (do add-new + dual-write + drop-old instead), changing a column type (often rewrites the table), and adding a `unique=True` to an existing column (build a unique index `CONCURRENTLY`, then attach it as the constraint).

---

## 6. Top Three Non-Negotiable Code Review Rules

After everything above, these are the three things I will *block a PR* over — because each one is cheap to get right in review and brutally expensive to fix in production.

**1. Integrity invariants must be enforced in the database, not just in Python.**
If a business rule must always be true (uniqueness, a value range, a cross-field relationship, no overlap), I expect a `UniqueConstraint`, `CheckConstraint`, `ExclusionConstraint`, or a non-null FK in `Meta` — not a lonely `clean()` method. App validation is for friendly errors; the database is for guarantees. Bulk operations, raw SQL, and the next service to touch this table will all bypass your Python.

**2. Every `on_delete` is a deliberate, justified decision — and defaults to `PROTECT`.**
I will not approve a `ForeignKey` where `on_delete` looks copy-pasted. The author must be able to articulate *why* deleting the parent should cascade, protect, or null. When in doubt, `PROTECT`: a blocked delete is an inconvenience, a wrong `CASCADE` is data loss that no migration can undo.

**3. No migration may take a blocking lock on a hot table, and no schema change may be mixed with a data backfill.**
Adding a column with a volatile default, creating an index non-concurrently, or backfilling millions of rows inside a schema migration are all rejected on sight. The expected shape is: nullable add → deploy → batched backfill in a separate data migration → tighten constraints → `AddIndexConcurrently` with `atomic = False`. If the PR touches a large table's schema, I want to see the lock analysis in the description.

---

*Build models that the database itself defends, that read like the domain they model, and that can be deployed at 3pm on a Friday without anyone holding their breath.*
