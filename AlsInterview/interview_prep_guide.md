# 🧠 Interview Preparation Guide
### Python Django Backend Developer (3+ Years)

**Focus Areas:** Django · DRF · Celery · Redis · PostgreSQL · Docker · System Design (DDD) · E-commerce/Inventory

---

## 1. Python Fundamentals

### Q1. What are the four pillars of OOP and how does Python implement them?
**Encapsulation** (bundling data + methods, name mangling via `__`), **Inheritance** (single/multiple, MRO via C3 linearization), **Polymorphism** (duck typing, method overriding), **Abstraction** (`abc.ABC`, `@abstractmethod`). Python favors duck typing — "if it walks like a duck," the type matters less than the behavior.

### Q2. Explain `is` vs `==`.
`==` compares **values** (calls `__eq__`); `is` compares **identity** (same object in memory, same `id()`).

```python
a = [1, 2]; b = [1, 2]
a == b   # True  (same value)
a is b   # False (different objects)

x = 256; y = 256
x is y   # True  — CPython caches small ints (-5 to 256)
x = 257; y = 257
x is y   # False — outside cache range
```
**Interview tip:** Always use `is` for `None`, `True`, `False` (singletons).

### Q3. What are decorators? Write one.
A decorator is a callable that takes a function and returns a modified function — used for cross-cutting concerns (logging, timing, auth, caching) without touching the original code.

```python
import functools, time

def timeit(func):
    @functools.wraps(func)          # preserves __name__, __doc__
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter()-start:.4f}s")
        return result
    return wrapper

@timeit
def slow(): time.sleep(1)
```

### Q4. What are generators and why use them?
Functions using `yield` that produce values lazily, one at a time, holding state between calls. They're **memory-efficient** — ideal for large datasets or infinite streams.

```python
def read_large_file(path):
    with open(path) as f:
        for line in f:          # doesn't load whole file
            yield line.strip()
```
A list of 10M rows loads everything; a generator holds one row at a time.

### Q5. Difference between a generator and a list comprehension?
`[x for x in range(n)]` builds the full list in memory. `(x for x in range(n))` returns a generator object — evaluated lazily. Use generators when you iterate once and don't need indexing/length.

### Q6. What is a context manager? Write a custom one.
Manages setup/teardown via `__enter__`/`__exit__` — guarantees cleanup even on exceptions (the `with` statement).

```python
from contextlib import contextmanager

@contextmanager
def db_transaction(conn):
    tx = conn.begin()
    try:
        yield tx
        tx.commit()
    except Exception:
        tx.rollback()
        raise
```

### Q7. How does Python manage memory?
- **Reference counting** (primary): each object tracks how many references point to it; freed at count 0.
- **Generational garbage collector**: handles **reference cycles** (objects referencing each other) that ref-counting alone can't free. Three generations; younger objects scanned more often.
- **Memory pools** (pymalloc): small objects managed in arenas to reduce fragmentation.

### Q8. What is the GIL and how does it affect concurrency?
The **Global Interpreter Lock** allows only one thread to execute Python bytecode at a time. → Threads don't give true parallelism for **CPU-bound** work (use `multiprocessing`), but work fine for **I/O-bound** work (network, disk) since the GIL releases during I/O waits.

### Q9. `*args` vs `**kwargs`?
`*args` collects extra **positional** args into a tuple; `**kwargs` collects extra **keyword** args into a dict. Used for flexible function signatures and forwarding.

### Q10. Mutable vs immutable types — and a common gotcha.
Immutable: `int, str, tuple, frozenset`. Mutable: `list, dict, set`. Classic trap — mutable default arguments:

```python
def bad(item, lst=[]):      # lst created ONCE at definition
    lst.append(item); return lst
bad(1); bad(2)              # → [1, 2]  (shared!)

def good(item, lst=None):
    lst = lst or []
    lst.append(item); return lst
```

### Q11. Shallow vs deep copy?
`copy.copy()` copies the top-level object but shares nested references; `copy.deepcopy()` recursively copies everything. Matters for nested lists/dicts.

### Q12. `@staticmethod` vs `@classmethod` vs instance method?
- **Instance** (`self`): accesses instance state.
- **Class** (`cls`): accesses class state; common for alternative constructors (`from_json`).
- **Static**: no implicit first arg; a namespaced utility function.

### Q13. What does `__slots__` do?
Replaces the per-instance `__dict__` with a fixed set of attributes → less memory and faster attribute access. Useful when you create millions of small objects (e.g., value objects in DDD).

### Q14. Explain list vs tuple vs set vs dict (Big-O for lookups).
- `list`: ordered, mutable, O(n) membership.
- `tuple`: ordered, immutable, hashable.
- `set`: unordered, unique, **O(1)** membership.
- `dict`: key→value, **O(1)** average lookup (hash table).

### Q15. What are Python's "magic"/dunder methods? Name a few useful ones.
`__init__`, `__repr__`, `__eq__`, `__hash__`, `__len__`, `__iter__`, `__enter__`/`__exit__`, `__call__`. They let your objects integrate with built-in syntax and operators.

---

## 2. Django & DRF

### Q1. Explain Django's MTV architecture.
**Model** (data/DB layer, ORM) → **Template** (presentation) → **View** (business logic, request handling). Django's "MTV" maps to the classic MVC: View ≈ Controller, Template ≈ View. The URL dispatcher routes requests to views.

### Q2. What is the request/response lifecycle in Django?
`WSGI/ASGI server → Middleware (request phase) → URL resolver → View → (Model/ORM) → View returns Response → Middleware (response phase) → server → client`.

### Q3. What is middleware? Give a real use case.
A hook layer that processes every request/response globally. Real uses: authentication, CORS, request logging, rate limiting, timezone setting.

```python
class TimingMiddleware:
    def __init__(self, get_response): self.get_response = get_response
    def __call__(self, request):
        import time; start = time.time()
        response = self.get_response(request)
        response["X-Response-Time"] = f"{time.time()-start:.3f}s"
        return response
```

### Q4. `select_related` vs `prefetch_related`?
Both fix the **N+1 query problem**.
- `select_related`: SQL **JOIN**, for **ForeignKey/OneToOne** (single related object).
- `prefetch_related`: separate query + Python join, for **ManyToMany/reverse FK** (multiple objects).

```python
# N+1: 1 + N queries
for order in Order.objects.all():
    print(order.customer.name)        # query per order

# Fixed: 1 query
Order.objects.select_related("customer")
# M2M:
Product.objects.prefetch_related("tags")
```

### Q5. What are Django signals? When NOT to use them?
Decoupled notifications on events (`post_save`, `pre_delete`, `m2m_changed`). Good for audit logs, cache invalidation. **Avoid** for core business logic — they create hidden, hard-to-debug control flow. Prefer explicit service methods (especially in DDD).

### Q6. How does Django ORM lazy evaluation work?
QuerySets are **lazy** — no DB hit until evaluated (iteration, `list()`, `len()`, slicing with step, `bool()`). This lets you chain filters efficiently. Use `.exists()` not `if queryset:` to check existence cheaply.

### Q7. What's the difference between `null=True` and `blank=True`?
`null=True` → DB column allows NULL. `blank=True` → form/validation allows empty. For string fields, prefer `blank=True` only (use `""` not NULL to avoid two "empty" states).

### Q8. Explain DRF serializers. `Serializer` vs `ModelSerializer`.
Serializers convert complex types (querysets, model instances) ↔ JSON, and handle **validation**. `ModelSerializer` auto-generates fields from a model; plain `Serializer` is fully manual (good for non-model data).

```python
class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ["id", "name", "price", "stock"]

    def validate_price(self, value):
        if value <= 0:
            raise serializers.ValidationError("Price must be positive")
        return value
```

### Q9. `APIView` vs `GenericAPIView` vs `ViewSet`?
- **APIView**: full manual control, you define `get/post/...`.
- **GenericAPIView + mixins**: reusable CRUD building blocks.
- **ViewSet/ModelViewSet**: groups related actions; wired via **routers** → minimal boilerplate for standard CRUD.

### Q10. What are routers in DRF?
They auto-generate URL patterns for ViewSets (`list`, `create`, `retrieve`, `update`, `destroy`), enforcing REST conventions and reducing `urls.py` boilerplate.

### Q11. How do authentication and permissions differ in DRF?
**Authentication** = *who are you?* (identifies the user: Token, JWT, Session). **Permissions** = *what can you do?* (`IsAuthenticated`, `IsAdminUser`, custom `has_object_permission`). Authentication runs first.

```python
class IsOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user
```

### Q12. How would you implement JWT auth?
Use `djangorestframework-simplejwt`: client logs in → receives **access** (short-lived) + **refresh** (long-lived) tokens → sends `Authorization: Bearer <access>` → refreshes when expired. Store securely (httpOnly cookie / secure storage), rotate refresh tokens, blacklist on logout.

### Q13. How do you handle versioning in DRF APIs?
URL path (`/api/v1/`), namespace, header, or query-param versioning. Path versioning is most common and cache-friendly. Set `DEFAULT_VERSIONING_CLASS`.

### Q14. How does DRF pagination work? Which type for large datasets?
`PageNumberPagination`, `LimitOffsetPagination`, `CursorPagination`. For **large, frequently-changing** datasets use **CursorPagination** — stable ordering, no "page drift," and O(1) performance vs OFFSET scanning.

### Q15. How do you optimize a slow DRF endpoint?
Profile first → fix N+1 with `select_related/prefetch_related` → `.only()/.defer()` to limit columns → add DB indexes → cache responses (Redis) → paginate → use `SerializerMethodField` sparingly → offload heavy work to Celery.

### Q16. What is `get_queryset()` vs `queryset` attribute?
`queryset` is static; override `get_queryset()` for **dynamic** filtering (per-user, per-request scoping — essential for multi-tenant/multi-warehouse).

```python
def get_queryset(self):
    return Order.objects.filter(warehouse=self.request.user.warehouse)
```

### Q17. How do you handle file uploads in DRF?
Use `FileField`/`ImageField` in the serializer with `MultiPartParser`. For large files or media, upload to S3 (via `django-storages`) and process asynchronously with Celery.

### Q18. How do throttling/rate limiting work in DRF?
`AnonRateThrottle`, `UserRateThrottle`, `ScopedRateThrottle` with `DEFAULT_THROTTLE_RATES`. Backed by the cache (Redis). Protects against abuse and brute-force.

### Q19. What are model managers and custom QuerySets?
A `Manager` is the query interface (`objects`). Custom managers/`QuerySet` methods encapsulate reusable query logic (`Product.objects.in_stock()`), keeping views thin — fits DDD repository thinking.

### Q20. How do you keep business logic out of views (fat models / services)?
Avoid "fat views." Use **service layers** / domain services for orchestration, **model methods** for entity behavior, and **selectors** for reads. In DDD ERP work this keeps domain rules testable and framework-independent.

---

## 3. Database & SQL

### Q1. What is an index and how does it work?
A data structure (usually a **B-tree**) that speeds lookups/sorts by avoiding full table scans — at the cost of slower writes and extra storage. Like a book's index: jump to the page instead of reading every page.

### Q2. When does an index NOT help?
Low-cardinality columns (e.g., boolean), leading wildcard `LIKE '%x'`, functions on the column (`WHERE LOWER(name)=...` unless you index the expression), or very small tables. Too many indexes slow down INSERT/UPDATE.

### Q3. Explain database normalization (1NF–3NF).
- **1NF**: atomic values, no repeating groups.
- **2NF**: 1NF + no partial dependency on part of a composite key.
- **3NF**: 2NF + no transitive dependency (non-key → non-key).
Reduces redundancy and update anomalies. **Denormalize** selectively for read-heavy performance.

### Q4. Normalization vs denormalization — when to denormalize?
Normalize for write integrity; denormalize (duplicate data, precomputed totals) for read-heavy reporting/dashboards to cut JOINs. E-commerce example: store an `order_total` snapshot rather than recomputing from line items every read.

### Q5. What are transactions and the ACID properties?
**Atomicity** (all-or-nothing), **Consistency** (valid state→valid state), **Isolation** (concurrent txns don't corrupt each other), **Durability** (committed = persisted). Critical for orders/payments/inventory deductions.

### Q6. Explain isolation levels.
`READ UNCOMMITTED` → dirty reads; `READ COMMITTED` (Postgres default) → no dirty reads; `REPEATABLE READ` → no non-repeatable reads; `SERIALIZABLE` → full isolation, no phantoms. Higher isolation = more locking/contention. Choose per use case.

### Q7. What concurrency anomalies exist?
**Dirty read** (read uncommitted data), **non-repeatable read** (row changes between reads), **phantom read** (new rows appear), **lost update** (two writers overwrite). Inventory systems must guard against lost updates on stock.

### Q8. How do you prevent overselling stock (race condition)?
Use atomic DB operations and row locks:

```python
# Atomic decrement — no read-modify-write race
Product.objects.filter(id=pid, stock__gte=qty).update(stock=F("stock") - qty)

# Or pessimistic lock inside a transaction
with transaction.atomic():
    p = Product.objects.select_for_update().get(id=pid)
    if p.stock >= qty:
        p.stock -= qty; p.save()
```

### Q9. Optimistic vs pessimistic locking?
**Pessimistic** (`SELECT ... FOR UPDATE`): lock row upfront — good for high contention. **Optimistic** (version column, check on write): no locks, retry on conflict — good for low contention/high throughput.

### Q10. How do you find and fix a slow query?
`EXPLAIN ANALYZE` → look for **Seq Scan** on large tables, bad row estimates, expensive sorts/joins → add indexes, rewrite subqueries as JOINs, avoid `SELECT *`, fix N+1, add `LIMIT`, update table statistics, consider partitioning.

### Q11. What is a covering index?
An index that contains **all columns** a query needs, so the DB answers from the index alone ("index-only scan") without touching the table. Postgres: `INCLUDE` columns.

### Q12. What causes the N+1 problem at the DB level and how to detect it?
One query loads parents, then one query **per** parent loads children. Detect with `django-debug-toolbar`, query-count assertions, or DB logs. Fix with eager loading.

### Q13. When would you use a materialized view?
For expensive aggregations queried often but tolerant of slight staleness (analytics dashboards). Refresh on a schedule (e.g., via Celery beat).

### Q14. How do you scale PostgreSQL reads?
**Read replicas** + Django database routers (writes → primary, reads → replica), connection pooling (PgBouncer), caching hot reads in Redis, and partitioning large tables (e.g., by date for orders/logs).

### Q15. What is connection pooling and why does it matter?
Reusing DB connections instead of opening one per request avoids costly connection setup and protects the DB from connection exhaustion under load. Tools: PgBouncer, `CONN_MAX_AGE`.

---

## 4. System Design (VERY IMPORTANT)

### Q1. How do you design a scalable REST API?
Stateless services (scale horizontally behind a load balancer), versioning, pagination, idempotency keys for writes, rate limiting, caching (CDN + Redis), async offloading (Celery), observability (logs/metrics/tracing), and a clean layered architecture.

```
Client → CDN → Load Balancer → API (N stateless instances)
                                  ├── Redis (cache/session/locks)
                                  ├── PostgreSQL (primary + read replicas)
                                  └── Celery → Broker (Redis/RabbitMQ) → Workers
```

### Q2. Design a multi-warehouse inventory system.
Core entities: `Product`, `Warehouse`, `StockItem(product, warehouse, quantity, reserved)`, `StockMovement` (immutable ledger: IN/OUT/TRANSFER). Track **available = quantity − reserved**. Reserve stock at checkout, deduct on fulfillment, restore on cancel. Use atomic updates + `select_for_update` to prevent oversell. Keep a movement ledger for auditability (event-sourced flavor).

```
Order placed → reserve stock (atomic) → payment →
fulfill: StockMovement OUT + decrement → ship
cancel/expire reservation → release reserved qty
```

### Q3. Design a caching strategy with Redis.
Patterns:
- **Cache-aside (lazy)**: app checks Redis → miss → DB → populate cache. Most common.
- **Write-through**: write to cache + DB together (consistency, slower writes).
- **TTL** on every key + **invalidation** on writes.
Cache hot/expensive reads: product catalog, config, sessions, computed aggregates.

```python
def get_product(pid):
    key = f"product:{pid}"
    if (cached := redis.get(key)):
        return json.loads(cached)
    product = Product.objects.get(id=pid)
    redis.setex(key, 300, json.dumps(serialize(product)))   # 5-min TTL
    return product
```
Mention **cache stampede** protection (locks / jittered TTL) and the **eviction policy** (`allkeys-lru`).

### Q4. How does Celery work and when do you use it?
Offloads slow/blocking tasks from the request cycle for responsiveness.
```
Django → .delay() → Broker (Redis/RabbitMQ) → Worker executes → Result backend
```
Use cases: emails/SMS, PDF/report generation, image processing, third-party API calls, scheduled jobs (**Celery Beat**). Make tasks **idempotent** and configure **retries** with backoff.

```python
@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def send_invoice(self, order_id):
    try:
        generate_and_email_invoice(order_id)
    except SMTPException as exc:
        raise self.retry(exc=exc)
```

### Q5. Redis as cache vs broker vs lock — explain.
Same Redis, different roles: **cache** (TTL key-value), **Celery broker** (task queue), **distributed lock** (`SET NX EX` / Redlock) to serialize critical sections across instances (e.g., a single nightly stock-reconciliation job).

### Q6. Monolith vs Microservices — which and why?
**Monolith**: simple deploy, easy transactions, faster early development — best for small/medium teams. **Microservices**: independent scaling/deploy, tech flexibility — but adds network, distributed-transaction, and ops complexity. Pragmatic path: **modular monolith** (DDD bounded contexts as modules) → split out services only when scaling/team boundaries demand it.

### Q7. What is Domain-Driven Design (DDD)?
Modeling software around the **business domain** using a **ubiquitous language**. Key tactical patterns: **Entities** (identity), **Value Objects** (immutable, no identity — `Money`, `Address`), **Aggregates** (consistency boundary with a root), **Repositories** (persistence abstraction), **Domain Services**, **Domain Events**. Strategic: **Bounded Contexts**. Keeps domain logic independent of Django/DRF.

### Q8. How do you structure a Django project with DDD?
```
src/
  domain/        # entities, value objects, domain services (pure Python)
  application/   # use-cases / services orchestrating the domain
  infrastructure/# Django ORM models, repositories, Celery, Redis
  interfaces/    # DRF views, serializers (thin)
```
Dependencies point **inward** (interfaces → application → domain). ORM models live in infrastructure and map to domain entities.

### Q9. What is an aggregate and why does it matter for inventory?
An **aggregate** is a cluster of objects treated as one unit with one **root** controlling all changes, enforcing invariants in a single transaction. Example: `Order` (root) + `OrderLines` — you never modify a line directly; the `Order` enforces totals and stock rules, guaranteeing consistency.

### Q10. How would you ensure eventual consistency across services?
**Domain/Integration events** + message broker. E.g., `OrderPlaced` event → Inventory service consumes it → reserves stock. Use the **outbox pattern** (write event + state in one DB transaction, relay async) to avoid lost messages. Idempotent consumers handle duplicates.

### Q11. How do you handle distributed transactions?
Avoid 2PC where possible. Use the **Saga pattern**: a sequence of local transactions with **compensating actions** on failure (e.g., payment fails → release reserved stock). Choreography (events) or orchestration (a coordinator).

### Q12. Design a rate limiter.
**Token bucket** or **sliding window** in Redis. Key per user/IP, atomic `INCR` with expiry. Token bucket allows bursts; sliding-window-log is precise but heavier. DRF throttling backed by Redis covers most cases.

### Q13. How do you make an API idempotent for payments/orders?
Client sends an **Idempotency-Key**; server stores the key→result. Duplicate requests (retries, double-clicks) return the original result instead of creating a second order/charge.

### Q14. How do you design for high availability?
Multiple stateless app instances across zones, load balancer health checks, DB primary + replicas with failover, Redis with replication/Sentinel, graceful degradation (serve stale cache if DB is slow), and autoscaling.

### Q15. How do you observe and debug a production system?
**Three pillars:** structured **logs** (correlation IDs), **metrics** (latency, error rate, queue depth — Prometheus/Grafana), **traces** (OpenTelemetry). Plus Sentry for exceptions and alerting on SLOs.

---

## 5. Real-World Scenario Questions

### S1. "Design a scalable e-commerce system."
1. **Requirements**: catalog, cart, checkout, payments, inventory, orders, search.
2. **Architecture**: CDN → LB → stateless Django/DRF → Postgres (primary+replicas) → Redis (cache/sessions/locks) → Celery (emails, invoices) → S3 (media) → search (Elasticsearch).
3. **Hot paths**: cache catalog, paginate listings, async non-critical work.
4. **Consistency**: reserve→deduct stock atomically; idempotent checkout.
5. **Scale**: horizontal app scaling, read replicas, partition orders by date.
6. **Resilience**: retries, circuit breakers on payment gateway.

### S2. "An API endpoint is suddenly slow under high traffic. Diagnose."
1. Check **metrics/APM** — is it DB, CPU, external call, or queue?
2. Inspect **query count** (N+1?) and `EXPLAIN ANALYZE` slow queries.
3. Add **caching** for repeated reads; add missing **indexes**.
4. Offload heavy work to **Celery**.
5. Add **pagination/rate limiting**.
6. Scale horizontally + connection pooling. Verify with load testing.

### S3. "Optimize a slow report query touching millions of rows."
Profile with `EXPLAIN ANALYZE` → add composite/covering indexes → precompute via **materialized view** refreshed by Celery Beat → paginate/stream results → consider partitioning by date → cache the final report in Redis with TTL.

### S4. "Handle 10k concurrent checkout requests without overselling."
Atomic stock decrement (`F()` + `stock__gte`) or `select_for_update`; reserve-then-confirm flow with reservation expiry; queue spikes via Celery; idempotency keys; Redis distributed lock only for truly serial sections. Load-test to validate.

### S5. "Users report duplicate orders on double-click."
Add **Idempotency-Key** per submit; disable button client-side; DB **unique constraint** on a request token; return the existing order on retry.

### S6. "Background emails are delayed for hours."
Inspect Celery **queue depth** and worker count; check the broker; separate queues by priority (transactional vs bulk); autoscale workers; add monitoring/alerting on queue lag; ensure tasks are idempotent and have retries.

### S7. "Design notifications (email + SMS + push) at scale."
Producer publishes a `NotificationRequested` event → Celery workers per channel → provider abstraction with retries/fallback → templating service → user preferences/opt-out → delivery tracking. Rate-limit per provider.

### S8. "Migrate a monolith to services without downtime."
**Strangler pattern**: identify a bounded context (e.g., Inventory) → extract behind the same API → route a slice of traffic → dual-write/events for data sync → cut over gradually → decommission old path. Keep rollback ready.

### S9. "Secure a healthcare SaaS (e-prescription) handling sensitive data."
Encrypt in transit (TLS) and at rest; field-level encryption for PHI; strict RBAC + object-level permissions; full **audit logging**; short-lived JWTs + refresh rotation; rate limiting; input validation (Pydantic/DRF); least-privilege DB users; compliance mindset (HIPAA-style controls). Never log sensitive payloads.

### S10. "Implement multi-tenancy for the ERP."
Options: **shared DB + tenant_id** (simple, scope every query via `get_queryset`), **schema-per-tenant** (isolation, `search_path`), or **DB-per-tenant** (strongest isolation, costly). Choose by isolation/compliance needs. Always enforce tenant scoping in a base manager/middleware to prevent leakage.

---

## 6. Coding & Problem Solving

### C1. Reverse words in a string in place (logic).
```python
def reverse_words(s: str) -> str:
    return " ".join(s.split()[::-1])
# "hello world foo" -> "foo world hello"
```

### C2. Find the first non-repeating character.
```python
from collections import Counter
def first_unique(s: str) -> str | None:
    counts = Counter(s)
    return next((c for c in s if counts[c] == 1), None)
# O(n) time, O(k) space
```

### C3. Two Sum (hash map).
```python
def two_sum(nums, target):
    seen = {}
    for i, n in enumerate(nums):
        if (need := target - n) in seen:
            return [seen[need], i]
        seen[n] = i
# O(n) instead of O(n^2)
```

### C4. Detect a cycle in a linked list (Floyd's).
```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow is fast:
            return True
    return False
```

### C5. Group anagrams.
```python
from collections import defaultdict
def group_anagrams(words):
    groups = defaultdict(list)
    for w in words:
        groups["".join(sorted(w))].append(w)
    return list(groups.values())
```

### C6. Merge overlapping intervals.
```python
def merge(intervals):
    intervals.sort(key=lambda x: x[0])
    out = [intervals[0]]
    for start, end in intervals[1:]:
        if start <= out[-1][1]:
            out[-1][1] = max(out[-1][1], end)
        else:
            out.append([start, end])
    return out
```

### C7. Flatten a nested list (recursion/generator).
```python
def flatten(lst):
    for item in lst:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item
# list(flatten([1,[2,[3,4]],5])) -> [1,2,3,4,5]
```

### C8. LRU cache (using OrderedDict).
```python
from collections import OrderedDict
class LRUCache:
    def __init__(self, capacity): self.cap, self.d = capacity, OrderedDict()
    def get(self, key):
        if key not in self.d: return -1
        self.d.move_to_end(key); return self.d[key]
    def put(self, key, val):
        if key in self.d: self.d.move_to_end(key)
        self.d[key] = val
        if len(self.d) > self.cap: self.d.popitem(last=False)
```

### C9. Find duplicates in a list efficiently.
```python
def find_duplicates(nums):
    seen, dupes = set(), set()
    for n in nums:
        (dupes if n in seen else seen).add(n)
    return list(dupes)
```

### C10. FizzBuzz (clean version).
```python
def fizzbuzz(n):
    for i in range(1, n + 1):
        print("FizzBuzz" if i % 15 == 0 else
              "Fizz" if i % 3 == 0 else
              "Buzz" if i % 5 == 0 else i)
```

---

## 7. Behavioral Questions

> Use the **STAR** method: **S**ituation → **T**ask → **A**ction → **R**esult.

### B1. "Tell me about a challenging bug you solved."
*Sample:* "In our multi-warehouse system, stock occasionally went negative under load (**S**). I had to eliminate overselling (**T**). I reproduced it as a read-modify-write race, then replaced the logic with an atomic `F()` decrement guarded by `stock__gte` and added `select_for_update` for the reservation flow (**A**). Negative stock dropped to zero and checkout throughput stayed stable (**R**)."

### B2. "How do you handle tight deadlines?"
Prioritize ruthlessly (must-have vs nice-to-have), break work into milestones, communicate risks early, cut scope rather than quality, and protect critical paths with tests. I'd rather ship a smaller correct slice than a rushed broken feature.

### B3. "Describe a time you disagreed with a teammate."
Focus on the data, not the person — I proposed evaluating both approaches against requirements (e.g., signals vs explicit service calls), we benchmarked/reviewed trade-offs, and aligned on the more maintainable option. Disagree-and-commit once decided.

### B4. "How do you approach learning a new technology?"
Read the official docs, build a small spike, then apply it to a real task. For FastAPI/Pydantic I prototyped an endpoint, compared it to our DRF patterns, and documented findings for the team.

### B5. "Tell me about a project you're proud of."
The DDD-based ERP: I helped structure bounded contexts and a service layer that kept domain rules independent of Django, which made testing and onboarding dramatically easier and reduced regression bugs.

### B6. "How do you handle production incidents?"
Stay calm, mitigate first (roll back / serve from cache), communicate status, find root cause with logs/metrics/traces, fix forward, then write a blameless post-mortem with action items to prevent recurrence.

### B7. "How do you ensure code quality?"
Code reviews, tests (unit + integration), type hints/Pydantic validation, linters/formatters in CI, small focused PRs, and clear documentation. Quality is cheaper than debugging in prod.

### B8. "How do you handle feedback/criticism?"
I treat it as data to improve. In reviews I ask clarifying questions, apply what's valid, and keep ego out of it — the goal is a better product, not being right.

### B9. "Describe a time you improved performance significantly."
I cut an endpoint from ~2s to ~150ms by fixing N+1 queries with `select_related`, adding a composite index, and caching the hot read in Redis — verified with the debug toolbar and load tests.

### B10. "Where do you see yourself growing?"
Deepening distributed-systems and architecture skills, mentoring juniors, and owning end-to-end design of scalable services — building on my DDD and system-design work.

---

## 8. Rapid Revision Section

### ⚡ Key Django/DRF Concepts
- **MTV** = Model · Template · View (View ≈ Controller).
- Fix **N+1** → `select_related` (FK/O2O, JOIN) · `prefetch_related` (M2M/reverse).
- QuerySets are **lazy**; use `.exists()`, `.only()`, `.defer()`, `.iterator()`.
- **Serializer** validates + transforms; **ViewSet + Router** = CRUD with minimal code.
- **Auth** = identity · **Permissions** = access. JWT = access + refresh.
- Scope multi-tenant/warehouse data in `get_queryset()`.
- Keep business logic in **services/models**, not views or signals.

### ⚡ Redis + Celery Flow
```
CACHE:   request → Redis hit? → return ; miss → DB → set TTL → return
CELERY:  view.delay() → broker (Redis) → worker runs task → result backend
SCHEDULE: Celery Beat → periodic tasks (reports, cleanups)
LOCK:    SET key NX EX  → serialize critical section across instances
```
- Tasks must be **idempotent** + have **retries with backoff**.
- Redis roles: **cache · broker · distributed lock · sessions**.
- Watch for **cache stampede** (locks/jittered TTL) and **eviction** (`allkeys-lru`).

### ⚡ API Best Practices
- Stateless · versioned (`/api/v1/`) · paginated (cursor for big data).
- **Idempotency keys** for writes (orders/payments).
- Rate limiting + throttling (Redis-backed).
- Validate input (DRF/Pydantic); never trust the client.
- Cache hot reads; offload heavy work to Celery.
- Proper status codes + consistent error format.
- Observability: logs (correlation IDs) · metrics · traces.

### ⚡ Database Quick Hits
- Index high-cardinality, frequently-filtered/joined columns.
- `EXPLAIN ANALYZE` before optimizing; watch for **Seq Scan**.
- Prevent oversell: atomic `F()` update or `select_for_update`.
- Pick **isolation level** by need; default `READ COMMITTED`.
- Scale reads with replicas + pooling (PgBouncer); partition big tables.

### ⚡ System Design / DDD
- Entities · Value Objects · Aggregates (root = consistency boundary) · Repositories · Domain Events · Bounded Contexts.
- Prefer **modular monolith** → split to microservices when justified.
- Cross-service consistency → **events + outbox + Saga** (compensating actions).
- Dependencies point **inward**: interfaces → application → domain.

---

## 🔥 Most Expected Questions for This Profile

These map directly to your stack, projects, and seniority — **prepare these cold**:

1. **`select_related` vs `prefetch_related`** and how you fixed an N+1 in production.
2. **How do you prevent overselling stock** in a multi-warehouse inventory system? *(atomic `F()` / `select_for_update`)*
3. **Design a scalable e-commerce + POS + inventory system** end to end.
4. **Explain your Redis caching strategy** (cache-aside, TTL, invalidation, stampede).
5. **How does Celery work**, and what do you offload to it? *(idempotency + retries)*
6. **Explain DDD** and how you structured the ERP project (aggregates, bounded contexts, service layer).
7. **Monolith vs microservices** — what did you choose and why? *(modular monolith)*
8. **How do you secure a healthcare SaaS** (e-prescription) handling sensitive data?
9. **Optimize a slow query** on millions of rows — full methodology.
10. **Make an order/payment API idempotent** (handle retries & double-clicks).
11. **DRF auth & permissions** — JWT flow + object-level permissions.
12. **Transactions & isolation levels** — preventing lost updates on inventory.
13. **How do you keep business logic out of views/signals?** *(services, fat models, selectors)*
14. **Multi-tenancy approach** for the ERP and how you enforce tenant isolation.
15. **Where do FastAPI/Pydantic fit** alongside Django, and why use one over the other?

---

### ✅ Final Prep Tips
- Tie every answer back to your **real projects** (ERP, e-commerce/POS, e-prescription) — concrete beats generic.
- For system design: **clarify requirements → sketch architecture → discuss trade-offs → mention scale & failure modes**.
- Always state the **"why,"** not just the "what."
- Practice explaining out loud; structure (STAR / step-by-step) signals seniority.

**Good luck — you've got this! 🚀**
