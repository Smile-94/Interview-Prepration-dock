# Advanced System Design Interview Questions & Answers
### For Python/Django Engineers Targeting Staff-Level Roles

> **Target Audience:** Software engineers with 3+ years of Python/Django experience, bridging framework-specific knowledge into large-scale distributed systems design.

---

## Table of Contents

- [Part 1: Scaling the Monolith (Django Specific)](#part-1-scaling-the-monolith-django-specific)
  - [Q1: WSGI vs ASGI — Internals, Bottlenecks & Migration](#q1-wsgi-vs-asgi--internals-bottlenecks--migration)
  - [Q2: The N+1 Query Problem at Scale](#q2-the-n1-query-problem-at-scale)
  - [Q3: Connection Pooling — Why Django Gets It Wrong by Default](#q3-connection-pooling--why-django-gets-it-wrong-by-default)
  - [Q4: Session Scaling — From Database Sessions to Distributed State](#q4-session-scaling--from-database-sessions-to-distributed-state)
- [Part 2: Asynchronous Processing & Decoupling](#part-2-asynchronous-processing--decoupling)
  - [Q5: Celery Internals — Worker Crashes, Task States & Idempotency](#q5-celery-internals--worker-crashes-task-states--idempotency)
  - [Q6: Dead Letter Queues, Retry Strategies & Poison Pills](#q6-dead-letter-queues-retry-strategies--poison-pills)
- [Part 3: Database Architectures at Scale](#part-3-database-architectures-at-scale)
  - [Q7: PostgreSQL Scaling Limits & Read Replicas](#q7-postgresql-scaling-limits--read-replicas)
  - [Q8: Replication Lag — Detection & Application-Level Handling](#q8-replication-lag--detection--application-level-handling)
  - [Q9: Pessimistic vs Optimistic Locking Under High Concurrency](#q9-pessimistic-vs-optimistic-locking-under-high-concurrency)
- [Part 4: Microservices & Real-World Scenarios](#part-4-microservices--real-world-scenarios)
  - [Q10: Design an E-Commerce Checkout System for 10,000 TPS](#q10-design-an-e-commerce-checkout-system-for-10000-tps)

---

## Part 1: Scaling the Monolith (Django Specific)

---

### Q1: WSGI vs ASGI — Internals, Bottlenecks & Migration

**Interview Question:** *"Your Django application is deployed with Gunicorn (WSGI). Users complain about slow WebSocket connections and high latency during traffic spikes. A senior engineer suggests migrating to ASGI. Walk me through what WSGI and ASGI actually are at the protocol level, where the bottleneck lives, and how you'd execute the migration with minimal risk."*

---

#### Answer

##### 1. WSGI Internals — The Synchronous Request-Response Contract

WSGI (PEP 3333) is a **synchronous, single-call interface** between a web server (e.g., Nginx, Gunicorn) and a Python web application. The contract is defined by a single callable:

```python
# The WSGI application callable — the entire protocol in one function signature
def application(environ: dict, start_response: callable) -> Iterable[bytes]:
    """
    environ:        Dictionary of CGI-style request variables (HTTP method, headers, body stream).
    start_response: A callable the app must invoke to send the HTTP status and headers.
    Returns:        An iterable of byte strings forming the response body.
    """
    start_response('200 OK', [('Content-Type', 'text/html')])
    return [b'<html>Hello, World</html>']
```

Gunicorn is a **pre-fork** WSGI server. On startup, it forks N worker processes (typically `2 * CPU_cores + 1`). Each worker handles **exactly one request at a time**. This is the root of the bottleneck:

```
                         ┌──────────────┐
   Incoming Requests ──► │  Gunicorn    │
        (1000/sec)        │  Master      │
                         └──────┬───────┘
                  ┌─────────────┼─────────────┐
                  ▼             ▼             ▼
           Worker 1       Worker 2       Worker 3
          (1 request)    (1 request)    (1 request)
          [BLOCKED on     [BLOCKED on    [BUSY computing]
           DB I/O]         network I/O]
```

**The critical insight:** If Worker 1 is waiting 50ms for a PostgreSQL query to return, that OS thread/process is **100% idle** but **unavailable** to serve another request. This is catastrophic for I/O-bound workloads. With 4 workers and an average DB latency of 100ms, your theoretical max throughput is `4 / 0.1s = 40 req/sec`, even though your CPU is barely utilized.

##### 2. ASGI Internals — The Asynchronous, Event-Driven Contract

ASGI (PEP 675-inspired) is a **tri-callable, bidirectional** interface supporting the full lifecycle of HTTP, WebSocket, and background task protocols.

```python
# The ASGI application callable — fundamentally different contract
async def application(scope: dict, receive: callable, send: callable) -> None:
    """
    scope:   Connection metadata (type: 'http' | 'websocket' | 'lifespan', headers, path).
    receive: An awaitable that yields incoming events (HTTP body chunks, WS messages).
    send:    An awaitable to push outgoing events (HTTP response start/body, WS messages).
    
    The key: receive and send are async callables. While awaiting, the event loop
    serves other requests — no blocking.
    """
    assert scope['type'] == 'http'
    
    # Wait for the full request body — non-blocking
    event = await receive()
    
    # Send the response in two events: start, then body
    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [[b'content-type', b'text/html']],
    })
    await send({
        'type': 'http.response.body',
        'body': b'<html>Hello, World</html>',
        'more_body': False,
    })
```

Uvicorn (ASGI server) uses a **single-process event loop** (via `asyncio`) with optional multiple worker processes. While one coroutine awaits a DB response, the event loop switches to serve another request. I/O wait translates to **zero wasted CPU cycles**.

```
                         ┌────────────────────────────────────┐
   Incoming Requests ──► │  Uvicorn Event Loop (single thread) │
        (1000/sec)        │                                    │
                         │  Coroutine A  ──► await db query   │
                         │  Coroutine B  ──► await redis get  │──► all running "concurrently"
                         │  Coroutine C  ──► computing result │
                         └────────────────────────────────────┘
```

**WebSocket support:** WSGI has no concept of a long-lived connection. ASGI's `scope/receive/send` model naturally supports WebSockets — `receive` yields WS message events, and `send` pushes frames back. This is architecturally impossible in WSGI.

##### 3. Where Django Sits Today

Django has supported ASGI since version 3.0, but with important nuances:

| Feature | Django + WSGI | Django + ASGI |
|---|---|---|
| Sync views | ✅ Native | ✅ Runs in thread pool executor |
| `async def` views | ❌ Not supported | ✅ Native |
| WebSockets | ❌ Impossible | ✅ Via Channels or native Django |
| ORM in async context | N/A | ⚠️ Must use `sync_to_async` wrapper |
| Middleware | ✅ Fully sync | ⚠️ Mix of sync/async, use `AsyncMiddlewareMixin` |

**The `sync_to_async` trap:** Django's ORM is synchronous. Running it from an async view requires care:

```python
# views.py — WRONG: This will raise SynchronousOnlyOperation
import asyncio
from django.http import JsonResponse
from .models import Order

async def get_orders(request):
    # ❌ The ORM will detect it's called from an async context and raise an error
    orders = list(Order.objects.filter(user=request.user))
    return JsonResponse({'count': len(orders)})

# views.py — CORRECT: Wrap ORM calls
from asgiref.sync import sync_to_async
from django.http import JsonResponse
from .models import Order

async def get_orders(request):
    # ✅ sync_to_async runs the ORM call in Django's thread pool executor
    # thread_sensitive=True (default) ensures it runs in the main thread — required
    # for thread-local state like database connections
    get_orders_from_db = sync_to_async(
        lambda: list(Order.objects.filter(user=request.user).select_related('product')),
        thread_sensitive=True
    )
    orders = await get_orders_from_db()
    return JsonResponse({'count': len(orders)})
```

##### 4. Migration Strategy — Zero-Downtime Execution Plan

**Phase 1: Audit & Dependency Check**

```bash
# Check for sync-only third-party middleware
pip show django-debug-toolbar django-cors-headers  # Must verify ASGI compatibility
grep -r "process_request\|process_response" your_project/middleware/  # Find sync middleware

# Check for blocking calls in views
grep -rn "time.sleep\|requests.get\|urllib" your_project/  # These block the event loop
```

**Phase 2: Update ASGI entry point**

```python
# asgi.py — Django generates this; verify it's correct
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')
application = get_asgi_application()
```

**Phase 3: Gunicorn → Uvicorn transition (with Nginx)**

```nginx
# nginx.conf — Route to Uvicorn workers
upstream django_asgi {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    keepalive 32;  # Maintain persistent connections to upstream
}

server {
    listen 80;
    location / {
        proxy_pass http://django_asgi;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;       # Required for WebSocket
        proxy_set_header Connection "upgrade";        # Required for WebSocket
        proxy_set_header Host $host;
        proxy_read_timeout 300s;
    }
}
```

```bash
# Run Uvicorn with multiple workers (process-based parallelism)
uvicorn myproject.asgi:application \
    --host 0.0.0.0 \
    --port 8001 \
    --workers 4 \                  # Matches CPU core count
    --loop uvloop \                # High-performance event loop (Cython-based)
    --http h11 \                   # HTTP/1.1 parser
    --access-log \
    --log-level info
```

**Phase 4: Migrate sync middleware to async**

```python
# middleware.py
from django.utils.deprecation import MiddlewareMixin

# Before: Sync-only middleware
class AuthMiddleware(MiddlewareMixin):
    def process_request(self, request):
        token = request.headers.get('Authorization')
        if not token:
            from django.http import HttpResponse
            return HttpResponse(status=401)

# After: ASGI-compatible async middleware
import asyncio
from django.http import HttpResponse

class AsyncAuthMiddleware:
    async_capable = True
    sync_capable = False

    def __init__(self, get_response):
        self.get_response = get_response
        # Detect async vs sync context automatically
        if asyncio.iscoroutinefunction(self.get_response):
            self._is_coroutine = asyncio.coroutines._is_coroutine

    async def __call__(self, request):
        token = request.headers.get('Authorization')
        if not token:
            return HttpResponse(status=401)
        
        # Await the next middleware/view in the chain
        response = await self.get_response(request)
        return response
```

**Key Takeaway:** WSGI is a synchronous, one-request-per-worker model. ASGI is an event-driven model that handles concurrency through coroutines, enabling WebSockets and dramatically reducing wasted I/O wait time. The migration is incremental — Django runs sync views inside ASGI transparently via thread pools, so you don't need to rewrite everything at once.

---

### Q2: The N+1 Query Problem at Scale

**Interview Question:** *"Explain the N+1 query problem in a Django context. Now assume you have an endpoint that serves 500 requests per second, and each request triggers 50 database queries due to N+1. Your PostgreSQL instance is at 95% CPU. Walk me through the full diagnosis and resolution strategy, including ORM tools, query analysis, and caching layers."*

---

#### Answer

##### 1. What N+1 Is — The ORM Trap

The N+1 problem occurs when fetching a list of N objects (1 query) and then lazily fetching a related object for each one (N additional queries). Django's ORM is lazy by default — it doesn't execute queries until the queryset is evaluated.

```python
# models.py
class Author(models.Model):
    name = models.CharField(max_length=200)
    bio = models.TextField()

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    published_date = models.DateField()

# views.py — Classic N+1 example
def book_list(request):
    books = Book.objects.all()  # Query 1: SELECT * FROM book
    
    result = []
    for book in books:
        # Query 2, 3, 4, ... N+1: SELECT * FROM author WHERE id = X
        # Django hits the DB on EVERY .author access because it's not prefetched
        result.append({
            'title': book.title,
            'author': book.author.name  # <-- N separate queries
        })
    return JsonResponse({'books': result})
```

At 500 req/sec with 100 books, this generates `500 * (1 + 100) = 50,500` queries per second. PostgreSQL has a max connection overhead, lock contention on catalog tables, and CPU cost per query parse/plan — this is fatal.

##### 2. Diagnosis — Finding N+1 in Production

**Tool 1: Django Debug Toolbar (development)**

```python
# settings.py (dev only)
INSTALLED_APPS = ['debug_toolbar', ...]
MIDDLEWARE = ['debug_toolbar.middleware.DebugToolbarMiddleware', ...]
INTERNAL_IPS = ['127.0.0.1']
```

**Tool 2: Django's connection.queries (staging)**

```python
# Use this in a management command or test to expose all queries
from django.db import connection, reset_queries
from django.conf import settings

settings.DEBUG = True  # Must be True for query logging

reset_queries()
# ... run the view logic here ...

print(f"Total queries: {len(connection.queries)}")
for i, q in enumerate(connection.queries):
    print(f"[{i}] ({q['time']}s) {q['sql'][:200]}")
```

**Tool 3: `nplusone` library (automated detection in tests)**

```python
# tests.py
from nplusone.ext.django import nplusone

class BookAPITest(TestCase):
    @nplusone(raise_on_n_plus_one=True)
    def test_book_list_no_n_plus_one(self):
        # This test FAILS if N+1 is detected — use in CI to prevent regressions
        Author.objects.create(name="Tolkien", bio="...")
        Book.objects.bulk_create([
            Book(title=f"Book {i}", author_id=1) for i in range(10)
        ])
        response = self.client.get('/api/books/')
        self.assertEqual(response.status_code, 200)
```

**Tool 4: PostgreSQL `pg_stat_statements`**

```sql
-- Find the most frequent query patterns
SELECT 
    query,
    calls,
    total_exec_time / calls AS avg_ms,
    rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;
```

If you see the same parameterized query (e.g., `SELECT * FROM author WHERE id = $1`) with thousands of `calls`, that's your N+1.

##### 3. Resolution — ORM-Level Fixes

**`select_related` — SQL JOIN for ForeignKey/OneToOne**

```python
# Before: N+1
books = Book.objects.all()

# After: Single JOIN query
# SQL: SELECT book.*, author.* FROM book INNER JOIN author ON book.author_id = author.id
books = Book.objects.select_related('author').all()

# Chain multiple relations
books = Book.objects.select_related('author', 'publisher__country').all()
```

**`prefetch_related` — Separate query + Python-side join for ManyToMany/reverse FK**

```python
class Book(models.Model):
    tags = models.ManyToManyField('Tag')
    reviews = models.related.RelatedManager  # reverse FK

# prefetch_related issues 2 queries total, then joins in Python:
# Query 1: SELECT * FROM book
# Query 2: SELECT * FROM book_tags INNER JOIN tag ON ... WHERE book_id IN (1,2,3,...)
books = Book.objects.prefetch_related('tags', 'reviews').all()

# Custom prefetch with filtering/ordering
from django.db.models import Prefetch

recent_reviews = Review.objects.filter(rating__gte=4).order_by('-created_at')
books = Book.objects.prefetch_related(
    Prefetch('reviews', queryset=recent_reviews, to_attr='top_reviews')
).all()

# Access the prefetched reviews without hitting DB
for book in books:
    print(book.top_reviews)  # Already in memory
```

**Annotate to push computation to DB**

```python
from django.db.models import Count, Avg, F, ExpressionWrapper, fields

# Instead of: for book in books: len(book.reviews.all())  [N+1]
# Do this — compute everything in a single SQL query:
books = Book.objects.annotate(
    review_count=Count('reviews'),
    avg_rating=Avg('reviews__rating'),
    # Computed fields
    days_since_publish=ExpressionWrapper(
        Now() - F('published_date'),
        output_field=fields.DurationField()
    )
).select_related('author').prefetch_related('tags')
```

##### 4. Caching Layer — When Even Optimized Queries Are Too Slow

```python
# cache_utils.py
from django.core.cache import cache
from django.core.serializers import serialize
import json

BOOK_LIST_CACHE_KEY = 'api:book_list:v2'
BOOK_LIST_CACHE_TTL = 60  # seconds

def get_cached_book_list():
    cached = cache.get(BOOK_LIST_CACHE_KEY)
    if cached is not None:
        return cached
    
    # Cache miss — hit DB with fully optimized query
    books = (
        Book.objects
        .select_related('author')
        .prefetch_related('tags')
        .annotate(review_count=Count('reviews'))
        .only('id', 'title', 'published_date', 'author__name')  # Projection
        .order_by('-published_date')[:100]
    )
    
    data = [
        {
            'id': b.id,
            'title': b.title,
            'author': b.author.name,
            'tags': [t.name for t in b.tags.all()],
            'review_count': b.review_count,
        }
        for b in books
    ]
    
    cache.set(BOOK_LIST_CACHE_KEY, data, BOOK_LIST_CACHE_TTL)
    return data

# Cache invalidation on write
def invalidate_book_list_cache():
    cache.delete(BOOK_LIST_CACHE_KEY)

# models.py — Tie invalidation to model saves
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver

@receiver([post_save, post_delete], sender=Book)
def on_book_change(sender, **kwargs):
    invalidate_book_list_cache()
```

**At 500 req/sec with a 60-second cache TTL, only 1 query hits the DB per minute instead of 30,000.**

---

### Q3: Connection Pooling — Why Django Gets It Wrong by Default

**Interview Question:** *"You've scaled your Django app to 20 Gunicorn workers across 3 servers (60 total workers). Your DBA tells you PostgreSQL is hitting its `max_connections` limit of 100 and is rejecting connections. Explain how Django manages database connections by default, why this is a problem at scale, and how PgBouncer solves it architecturally."*

---

#### Answer

##### 1. Django's Default Connection Behavior — Per-Worker, Per-Request

Django opens a database connection **per thread/worker** and holds it for the duration of the request. Each Gunicorn worker (a separate OS process) maintains its own connection. There is **no sharing** of connections across workers.

```
Server 1 (20 workers)  ──► 20 PostgreSQL connections
Server 2 (20 workers)  ──► 20 PostgreSQL connections
Server 3 (20 workers)  ──► 20 PostgreSQL connections
                                                     Total: 60 connections
```

This appears fine, but consider:
- **Celery workers:** 10 workers × 4 concurrency = 40 more connections
- **Management commands, cron jobs, admin:** 5–10 more connections
- **Django's CONN_MAX_AGE:** If set to `None` (persistent), connections are held even when idle

```python
# settings.py — The default is CONN_MAX_AGE=0 (reconnect every request)
# Many engineers set CONN_MAX_AGE=600 to "speed things up"
# But this means each worker holds a persistent connection even when idle

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'secret',
        'HOST': 'db-primary.internal',
        'PORT': '5432',
        'CONN_MAX_AGE': 600,  # ⚠️ Each of 60 workers holds a connection for 10 minutes
    }
}
```

**PostgreSQL's `max_connections` is a hard limit.** Every connection beyond this limit gets:
```
FATAL:  sorry, too many clients already
```

Each PostgreSQL connection costs ~5–10MB of RAM and a dedicated backend process. At 100 connections and 8MB each, that's 800MB just for connection overhead — before any real work is done.

##### 2. PgBouncer — A Connection Multiplexer

PgBouncer is a lightweight proxy that **multiplexes many client connections onto a small pool of actual PostgreSQL connections**.

```
                  ┌─────────────────────────────────┐
 60 Django        │          PgBouncer               │
 workers   ──────►│  Client connections: up to 2000  │──────► 20 PostgreSQL connections
                  │  Server connections (to PG): 20  │       (max_connections can be lowered)
                  └─────────────────────────────────┘
```

PgBouncer operates in three pooling modes, each with different transactional semantics:

| Mode | Connection Released | Safe for Django? | Use Case |
|---|---|---|---|
| **Session** | When client disconnects | ✅ Yes | Compatibility with `SET`, advisory locks, prepared statements |
| **Transaction** | After each `COMMIT`/`ROLLBACK` | ✅ Yes (with caveats) | Most web applications |
| **Statement** | After each SQL statement | ❌ No (breaks transactions) | Read-only analytics |

**Transaction mode** is the sweet spot for Django. A worker holds a PostgreSQL connection only for the duration of a transaction, then returns it to the pool. An idle worker consumes zero connections from PostgreSQL.

##### 3. PgBouncer Configuration

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
# Virtual database name = actual PostgreSQL connection string
mydb = host=db-primary.internal port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool configuration
pool_mode = transaction
max_client_conn = 2000          # Max Django (client) connections PgBouncer accepts
default_pool_size = 20          # Actual PostgreSQL server connections
reserve_pool_size = 5           # Extra connections for bursts
reserve_pool_timeout = 3.0      # Seconds before using reserve pool

# Timeouts
server_idle_timeout = 600       # Close idle server connections after 10 min
client_idle_timeout = 0         # Never close idle client connections
query_wait_timeout = 120        # Reject queries waiting longer than 120s

# Performance
server_reset_query = DISCARD ALL  # Reset connection state between clients (session pooling)
# For transaction pooling, reset is implicit
ignore_startup_parameters = extra_float_digits  # Required for some PostgreSQL drivers
```

```bash
# userlist.txt — hashed passwords
"myuser" "md5<hash_of_password_plus_username>"
```

**Django settings pointing to PgBouncer:**

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'secret',
        'HOST': 'pgbouncer.internal',  # Point to PgBouncer, not PostgreSQL directly
        'PORT': '6432',
        'CONN_MAX_AGE': 0,  # CRITICAL: Must be 0 in transaction pooling mode
        # With transaction pooling, PgBouncer manages the pool — Django should not
        # hold connections persistently. CONN_MAX_AGE > 0 defeats the purpose.
        'OPTIONS': {
            'connect_timeout': 10,
        }
    }
}
```

**Why `CONN_MAX_AGE=0` is mandatory with transaction pooling:** If Django holds a connection persistently, it will execute `SET` statements, use advisory locks, or rely on `LISTEN/NOTIFY` — all of which are session-scoped. Transaction pooling may give Django a different physical PostgreSQL connection for each transaction, so session state is lost. Setting `CONN_MAX_AGE=0` forces Django to return the connection after each request.

##### 4. Caveats of Transaction Pooling

```python
# These Django/PostgreSQL features BREAK with transaction pooling:

# 1. Advisory locks (session-scoped)
from django.db import connection
connection.cursor().execute("SELECT pg_advisory_lock(12345)")  # ❌

# 2. LISTEN/NOTIFY (session-scoped)
connection.cursor().execute("LISTEN my_channel")  # ❌

# 3. Temporary tables
connection.cursor().execute("CREATE TEMP TABLE t (id INT)")  # ❌ May vanish

# 4. Prepared statements (must be disabled via PostgreSQL driver)
# In psycopg2:
DATABASES['default']['OPTIONS'] = {'options': '-c statement_timeout=30000'}
# In psycopg3, set prepare_threshold=None
```

---

### Q4: Session Scaling — From Database Sessions to Distributed State

**Interview Question:** *"Your Django app stores sessions in the database (the default). At 10,000 active users, you notice session-related database queries are consuming 30% of your DB load. Walk me through the trade-offs of each session backend, and design a solution that handles 100,000 concurrent users."*

---

#### Answer

##### 1. Django's Session Backends — A Comparative Analysis

Django provides four built-in session backends:

**Backend 1: Database Sessions (default)**

```python
# settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.db'

# Every request with a logged-in user executes:
# SELECT django_session.* WHERE session_key = '<key>' AND expire_date > NOW()
# And on write: UPDATE django_session SET session_data = '...' WHERE session_key = '<key>'
```

At 10,000 active users with 5 requests/min each, that's `10,000 * 5 = 50,000` session reads per minute — plus writes on every mutation. The `django_session` table requires a B-tree index on `session_key` and regular `clearsessions` maintenance.

**Backend 2: Cached Sessions (Redis/Memcached)**

```python
# settings.py — Recommended for scale
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'sessions'  # Use a dedicated cache config

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://redis-primary.internal:6379/0',
        'OPTIONS': {'CLIENT_CLASS': 'django_redis.client.DefaultClient'},
    },
    'sessions': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://redis-sessions.internal:6379/1',  # Separate Redis DB/instance
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
            'IGNORE_EXCEPTIONS': True,  # Degrade gracefully if Redis is down
        },
        'TIMEOUT': 86400,  # 24 hours
    }
}

SESSION_COOKIE_AGE = 86400       # 24 hours
SESSION_SAVE_EVERY_REQUEST = False  # Only save when session is modified
SESSION_COOKIE_SECURE = True     # HTTPS only
SESSION_COOKIE_HTTPONLY = True   # No JavaScript access
SESSION_COOKIE_SAMESITE = 'Lax' # CSRF protection
```

**Backend 3: Cached Database Sessions (hybrid)**

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
# Read: Check cache first → fall back to DB on miss → repopulate cache
# Write: Write to both cache and DB simultaneously
# 
# Good for: Durability + speed. Bad for: Extra DB writes, complexity.
```

**Backend 4: Cookie-based Sessions**

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.signed_cookies'
# The ENTIRE session data is stored in the cookie, signed with SECRET_KEY.
# Pros: Zero server-side state — infinite scalability.
# Cons: 4KB cookie size limit, session data visible to user (though tamper-proof),
#        SECRET_KEY rotation invalidates ALL sessions.
```

##### 2. Redis Session Architecture for 100,000 Users

**Redis Session Key Structure:**

```
django.contrib.sessions.cache:<session_key_hash>
→ Serialized session data (default: PickleSerializer → use JSONSerializer)
```

```python
# settings.py — Secure serializer (avoid Pickle RCE vulnerability)
SESSION_SERIALIZER = 'django.contrib.sessions.serializers.JSONSerializer'
# Note: This restricts session values to JSON-serializable types only.
# Custom objects must be converted to dicts before storing in request.session.
```

**High-Availability Redis for Sessions:**

```python
# settings.py — Redis Sentinel for automatic failover
CACHES = {
    'sessions': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://redis-sessions-sentinel.internal:26379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.SentinelClient',
            'SENTINELS': [
                ('sentinel-1.internal', 26379),
                ('sentinel-2.internal', 26379),
                ('sentinel-3.internal', 26379),
            ],
            'SENTINEL_SERVICE_NAME': 'mymaster',
            'SENTINEL_KWARGS': {'socket_timeout': 0.5},
            'CONNECTION_POOL_KWARGS': {'max_connections': 50},
        }
    }
}
```

**Sizing Calculation:**

```
100,000 active sessions
× 500 bytes average session payload (user_id, permissions, CSRF token)
= ~50MB Redis memory for session data

Redis pipeline throughput: ~100,000 GET/sec per core
Session reads: 100,000 users × 10 req/min / 60 = ~16,667 reads/sec → Well within limits

Use Redis `EXPIRE` for TTL management (automatic expiry — no `clearsessions` cron needed)
```

**Custom session middleware to track session metrics:**

```python
# middleware.py
import time
from django.core.cache import caches

class SessionMetricsMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        
        # Log session cache hit/miss for monitoring
        if hasattr(request, 'session') and request.session.accessed:
            duration_ms = (time.monotonic() - start) * 1000
            # Emit to your metrics system (Prometheus, Datadog, etc.)
            # metrics.histogram('session.access.duration_ms', duration_ms)
        
        return response
```

---

## Part 2: Asynchronous Processing & Decoupling

---

### Q5: Celery Internals — Worker Crashes, Task States & Idempotency

**Interview Question:** *"You're using Celery with Redis as a broker to process payment confirmation emails. A worker crashes mid-task. Explain exactly what happens to that task — where is its state, can it be recovered, and how do you guarantee the email is never sent twice? Walk through the acknowledgment model and how you'd design the task to be idempotent."*

---

#### Answer

##### 1. Celery's Acknowledgment Model — The Root of All Reliability Questions

Celery's delivery guarantee is determined by **when the broker message is acknowledged**. Understanding this is the key to answering "what happens on a crash."

By default (`acks_late=False`), Celery **acknowledges the message immediately when it is received by the worker**, before execution begins:

```
Broker (Redis/RabbitMQ)                 Celery Worker
        │                                       │
        │── LPOP / BRPOP (get task) ───────────►│
        │                                       │  ACK sent immediately
        │◄── ACK ──────────────────────────────-│  (message deleted from broker)
        │                                       │
        │                                       │  Worker starts executing task...
        │                                       │  **CRASH HERE**
        │                                       ✗
        │
        │  Message is GONE. Task is lost forever.
```

**This is the default behavior. Crashed tasks are silently lost.**

With `acks_late=True`, the message is acknowledged only **after the task function returns**:

```
Broker (Redis/RabbitMQ)                 Celery Worker
        │                                       │
        │── LPOP / BRPOP (get task) ───────────►│
        │                                       │  Task begins execution...
        │                                       │  **CRASH HERE**
        │                                       ✗
        │
        │  No ACK received. Message visibility
        │  timeout expires (RabbitMQ) or Redis
        │  RPOPLPUSH restores to queue.
        │  Task is REQUEUED and re-executed.
```

##### 2. Redis vs RabbitMQ — Crash Behavior Differences

**With Redis as Broker (your scenario):**

Celery uses a reliable queue pattern via `BRPOPLPUSH`:
1. Task is atomically moved from the main queue to a "unacked" backup queue
2. A visibility timeout exists (managed by Celery's `broker_transport_options`)
3. On worker crash, a separate process/thread (in `acks_late` mode) must restore messages from the unacked queue

```python
# settings.py — Critical Redis broker configuration
CELERY_BROKER_URL = 'redis://redis.internal:6379/0'
CELERY_RESULT_BACKEND = 'redis://redis.internal:6379/1'  # Separate DB

# Acknowledgment behavior
CELERY_TASK_ACKS_LATE = True  # ✅ Acknowledge AFTER task completes
CELERY_TASK_REJECT_ON_WORKER_LOST = True  # Re-queue on worker crash (not discard)

# Visibility timeout — worker must complete within this window or task is requeued
CELERY_BROKER_TRANSPORT_OPTIONS = {
    'visibility_timeout': 3600,  # 1 hour — must be > your longest task duration
}

# Prevent a single worker from taking too many tasks at once
CELERY_WORKER_PREFETCH_MULTIPLIER = 1  # Take only 1 task at a time
# Default is 4 — problematic because a crash loses up to 4 tasks
```

**With RabbitMQ as Broker:**

RabbitMQ has native `basic.ack` semantics. With `acks_late=True` and worker crash, RabbitMQ automatically re-queues the unacknowledged message when the TCP connection closes. More reliable than Redis for this use case.

##### 3. Task State Machine

Celery tasks transition through these states:

```
PENDING → STARTED → SUCCESS
                 └─► FAILURE → RETRY → STARTED → ...
                            └─► FAILURE (max retries exceeded)
```

```python
# You can inspect state programmatically
from celery.result import AsyncResult

result = AsyncResult(task_id)
print(result.state)   # 'PENDING', 'STARTED', 'SUCCESS', 'FAILURE', 'RETRY'
print(result.result)  # Return value or exception
```

##### 4. Designing an Idempotent Email Task

**The core problem:** With `acks_late=True` and retries, the same task may execute **more than once**. Sending a payment confirmation email twice is a bad user experience and potentially a compliance issue.

**Idempotency Key Strategy:**

```python
# tasks.py
from celery import shared_task
from django.core.cache import cache
from django.core.mail import send_mail
from django.utils import timezone
import logging

logger = logging.getLogger(__name__)

@shared_task(
    bind=True,
    acks_late=True,
    reject_on_worker_lost=True,
    max_retries=3,
    default_retry_delay=60,          # 60 second initial retry delay
    autoretry_for=(Exception,),      # Auto-retry on any exception
    retry_backoff=True,              # Exponential backoff: 60s, 120s, 240s
    retry_backoff_max=600,           # Cap at 10 minutes
    retry_jitter=True,               # Add randomness to prevent thundering herd
    task_soft_time_limit=25,         # Raise SoftTimeLimitExceeded after 25s
    task_time_limit=30,              # Kill worker process after 30s (hard limit)
)
def send_payment_confirmation(self, order_id: int, user_email: str):
    """
    Idempotent task: safe to call multiple times for the same order_id.
    Uses Redis-based locking to prevent duplicate email delivery.
    """
    # --- Idempotency Gate ---
    # Construct a unique key for this logical operation
    idempotency_key = f"email:payment_confirmation:sent:{order_id}"
    
    # Atomic SET NX (set if not exists) — expires after 24 hours
    was_already_sent = not cache.add(
        idempotency_key,
        '1',
        timeout=86400  # 24 hours — within this window, duplicates are blocked
    )
    
    if was_already_sent:
        logger.info(
            "Skipping duplicate email for order %s — already sent (idempotency key found)",
            order_id
        )
        return {'status': 'skipped', 'reason': 'already_sent', 'order_id': order_id}
    
    # --- Pre-condition Check ---
    # Re-fetch the order to confirm its current state
    # (The order might have been cancelled between task dispatch and execution)
    from orders.models import Order
    try:
        order = Order.objects.select_related('user', 'product').get(
            id=order_id,
            status='payment_confirmed'  # Guard: only send if still in expected state
        )
    except Order.DoesNotExist:
        logger.warning("Order %s not found or not in 'payment_confirmed' state. Dropping task.", order_id)
        # Delete idempotency key so it can be re-triggered if needed
        cache.delete(idempotency_key)
        return {'status': 'skipped', 'reason': 'order_state_mismatch', 'order_id': order_id}
    
    # --- Execution ---
    try:
        send_mail(
            subject=f"Payment Confirmed — Order #{order_id}",
            message=f"Hi {order.user.first_name}, your payment of ${order.total} was received.",
            from_email='noreply@myshop.com',
            recipient_list=[user_email],
            fail_silently=False,
        )
        
        # --- Durable Audit Trail ---
        # Record the send in the DB for support/audit purposes
        Order.objects.filter(id=order_id).update(
            confirmation_email_sent_at=timezone.now()
        )
        
        logger.info("Payment confirmation email sent for order %s to %s", order_id, user_email)
        return {'status': 'sent', 'order_id': order_id}
        
    except Exception as exc:
        # If email failed, remove the idempotency key so retry can proceed
        cache.delete(idempotency_key)
        logger.error("Email sending failed for order %s: %s", order_id, exc)
        raise self.retry(exc=exc)
```

**Dispatching the task with a task_id that you control:**

```python
# views.py or payment_service.py
import uuid
from .tasks import send_payment_confirmation

def confirm_payment(order):
    # Use a deterministic task ID based on the order — prevents duplicate dispatch
    deterministic_task_id = f"payment-confirm-{order.id}"
    
    send_payment_confirmation.apply_async(
        args=[order.id, order.user.email],
        task_id=deterministic_task_id,  # Celery deduplicates if this ID is already running
        countdown=5,                    # Small delay to let the DB transaction commit
        expires=3600,                   # Don't execute if sitting in queue for > 1 hour
    )
```

---

### Q6: Dead Letter Queues, Retry Strategies & Poison Pills

**Interview Question:** *"A Celery task keeps failing with an unhandled exception and retrying indefinitely, consuming all worker capacity — a 'poison pill' scenario. Design a complete retry strategy using exponential backoff, a dead letter queue, and an alerting mechanism. Show the implementation for both Redis and RabbitMQ brokers."*

---

#### Answer

##### 1. The Poison Pill Problem

A poison pill is a task that always fails — perhaps due to corrupt input data, a downstream API returning a permanent error, or a bug in the task code. Without protection:

```
Task enters queue → Worker picks it up → Task fails → Immediately requeued →
Worker picks it up → Task fails → Immediately requeued → ...
[Worker is now permanently busy with one broken task, starving legitimate work]
```

##### 2. Exponential Backoff Configuration

```python
# celery_config.py
from kombu import Queue, Exchange

# Define routing topology upfront
task_queues = [
    Queue('default',    Exchange('default'),    routing_key='default'),
    Queue('high',       Exchange('high'),        routing_key='high'),
    Queue('dead_letter',Exchange('dead_letter'), routing_key='dead_letter'),
]

task_default_queue = 'default'
task_routes = {
    'myapp.tasks.critical_task': {'queue': 'high'},
}
```

```python
# tasks.py — Full retry strategy
from celery import shared_task
from celery.exceptions import MaxRetriesExceededError
from celery.utils.log import get_task_logger
import requests

logger = get_task_logger(__name__)

class PermanentFailure(Exception):
    """Raised when a task should never be retried (e.g., invalid input data)."""
    pass

@shared_task(
    bind=True,
    queue='default',
    acks_late=True,
    reject_on_worker_lost=True,

    # Retry configuration
    max_retries=5,
    default_retry_delay=30,          # Base delay in seconds
    retry_backoff=True,              # Enables exponential backoff
    retry_backoff_max=3600,          # Cap: never wait more than 1 hour
    retry_jitter=True,               # Adds ±25% randomness to prevent thundering herd

    # Time limits
    task_soft_time_limit=120,
    task_time_limit=150,

    # Routing for failures
    # Note: DLQ routing is handled in the on_failure hook below
)
def process_webhook(self, webhook_id: int, payload: dict):
    """
    Retry schedule with jitter (approximate):
    Attempt 1 (initial):  immediate
    Attempt 2 (retry 1):  ~30s
    Attempt 3 (retry 2):  ~60s
    Attempt 4 (retry 3):  ~120s
    Attempt 5 (retry 4):  ~240s
    Attempt 6 (retry 5):  ~480s (capped behaviour)
    After attempt 6:      send_to_dead_letter_queue() is called
    """
    try:
        from webhooks.models import WebhookDelivery
        delivery = WebhookDelivery.objects.get(id=webhook_id)
        
        # Permanent failure condition — don't retry, don't DLQ
        if not delivery.endpoint_url.startswith('https://'):
            raise PermanentFailure(f"Webhook {webhook_id} has insecure endpoint URL")
        
        response = requests.post(
            delivery.endpoint_url,
            json=payload,
            timeout=10,
            headers={'X-Webhook-Signature': delivery.compute_hmac(payload)}
        )
        response.raise_for_status()
        
        delivery.mark_delivered(status_code=response.status_code)
        return {'status': 'delivered', 'webhook_id': webhook_id}
        
    except PermanentFailure as exc:
        # Do NOT retry — log and give up immediately
        logger.error("Permanent failure for webhook %s: %s. Not retrying.", webhook_id, exc)
        send_to_dead_letter_queue.delay(
            task_name=self.name,
            task_id=self.request.id,
            webhook_id=webhook_id,
            payload=payload,
            error=str(exc),
            reason='permanent_failure',
        )
        return  # Return without raising — Celery will mark as SUCCESS

    except MaxRetriesExceededError:
        # Max retries hit — send to DLQ for human review
        logger.critical(
            "Max retries exceeded for webhook %s (task %s). Sending to DLQ.",
            webhook_id, self.request.id
        )
        send_to_dead_letter_queue.delay(
            task_name=self.name,
            task_id=self.request.id,
            webhook_id=webhook_id,
            payload=payload,
            error="Max retries exceeded",
            reason='max_retries_exceeded',
        )
        raise  # Re-raise so Celery marks task as FAILURE

    except requests.exceptions.Timeout as exc:
        logger.warning("Timeout on webhook %s, attempt %s", webhook_id, self.request.retries)
        raise self.retry(exc=exc)

    except requests.exceptions.HTTPError as exc:
        if exc.response.status_code in (400, 404, 410):
            # 4xx client errors are permanent — the endpoint is broken
            raise PermanentFailure(f"HTTP {exc.response.status_code} — permanent client error")
        # 5xx errors are transient — retry
        raise self.retry(exc=exc)

    except Exception as exc:
        logger.exception("Unexpected error processing webhook %s", webhook_id)
        raise self.retry(exc=exc)
```

##### 3. Dead Letter Queue Implementation

```python
# tasks.py — DLQ handler task
from django.utils import timezone
import json

@shared_task(queue='dead_letter', acks_late=True)
def send_to_dead_letter_queue(
    task_name: str,
    task_id: str,
    error: str,
    reason: str,
    **task_kwargs
):
    """
    Receives failed tasks. Persists them to DB for audit and sends alerts.
    """
    from dead_letters.models import DeadLetterRecord

    record = DeadLetterRecord.objects.create(
        original_task_name=task_name,
        original_task_id=task_id,
        task_kwargs=json.dumps(task_kwargs),
        error_message=error[:2000],  # Truncate to DB field limit
        failure_reason=reason,
        failed_at=timezone.now(),
        status='pending_review',
    )
    
    # Alert on-call engineering via PagerDuty/Slack
    notify_dead_letter_alert.delay(dead_letter_id=record.id)
    
    logger.error(
        "Task '%s' (id=%s) moved to Dead Letter Queue. Record ID: %s. Reason: %s",
        task_name, task_id, record.id, reason
    )
```

```python
# models.py
class DeadLetterRecord(models.Model):
    STATUSES = [
        ('pending_review', 'Pending Review'),
        ('replaying', 'Replaying'),
        ('resolved', 'Resolved'),
        ('discarded', 'Discarded'),
    ]
    
    original_task_name = models.CharField(max_length=255)
    original_task_id = models.CharField(max_length=255, unique=True)
    task_kwargs = models.TextField()  # JSON-serialized original arguments
    error_message = models.TextField()
    failure_reason = models.CharField(max_length=100)
    failed_at = models.DateTimeField()
    status = models.CharField(max_length=50, choices=STATUSES, default='pending_review')
    resolved_at = models.DateTimeField(null=True, blank=True)
    resolution_notes = models.TextField(blank=True)
    
    class Meta:
        indexes = [models.Index(fields=['status', 'failed_at'])]
```

**Management command to replay DLQ tasks:**

```python
# management/commands/replay_dead_letter.py
from django.core.management.base import BaseCommand
from dead_letters.models import DeadLetterRecord
import json
import importlib

class Command(BaseCommand):
    help = 'Replay tasks from the Dead Letter Queue'

    def add_arguments(self, parser):
        parser.add_argument('--id', type=int, help='Specific DLQ record ID to replay')
        parser.add_argument('--all-pending', action='store_true')

    def handle(self, *args, **options):
        if options['id']:
            records = DeadLetterRecord.objects.filter(id=options['id'])
        elif options['all_pending']:
            records = DeadLetterRecord.objects.filter(status='pending_review')
        
        for record in records:
            record.status = 'replaying'
            record.save()
            
            # Dynamically import and re-dispatch the original task
            module_path, func_name = record.original_task_name.rsplit('.', 1)
            module = importlib.import_module(module_path)
            task_func = getattr(module, func_name)
            
            kwargs = json.loads(record.task_kwargs)
            task_func.apply_async(kwargs=kwargs)
            
            self.stdout.write(f"Replayed DLQ record {record.id} ({record.original_task_name})")
```

##### 4. RabbitMQ Native DLQ Setup

RabbitMQ has first-class support for Dead Letter Exchanges (DLX):

```python
# celery_config.py — RabbitMQ DLX configuration
from kombu import Queue, Exchange

# Define the dead letter exchange
dead_letter_exchange = Exchange('dead_letters', type='direct', durable=True)

task_queues = [
    Queue(
        'default',
        Exchange('default', type='direct'),
        routing_key='default',
        queue_arguments={
            # Messages that exceed max_retries or TTL are routed here
            'x-dead-letter-exchange': 'dead_letters',
            'x-dead-letter-routing-key': 'dead_letter',
            'x-message-ttl': 3600000,  # 1 hour max TTL per message
        }
    ),
    Queue(
        'dead_letter',
        dead_letter_exchange,
        routing_key='dead_letter',
        durable=True,
    ),
]
```

---

## Part 3: Database Architectures at Scale

---

### Q7: PostgreSQL Scaling Limits & Read Replicas

**Interview Question:** *"Your PostgreSQL primary is handling 5,000 queries per second. CPU is at 70%, and write latency has spiked to 200ms. Your read:write ratio is 80:20. Design a read replica architecture and walk through how Django's database router directs traffic. What are the operational risks?"*

---

#### Answer

##### 1. PostgreSQL Scaling Ceiling Analysis

Before adding replicas, understand PostgreSQL's vertical limits:

```sql
-- Check your current performance metrics on the primary
SELECT 
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    ROUND(100.0 * blks_hit / NULLIF(blks_read + blks_hit, 0), 2) AS cache_hit_ratio
FROM pg_stat_database
WHERE datname = 'mydb';

-- Identify slow queries consuming primary resources
SELECT 
    query,
    calls,
    mean_exec_time,
    total_exec_time,
    rows
FROM pg_stat_statements
WHERE mean_exec_time > 100  -- queries slower than 100ms
ORDER BY total_exec_time DESC
LIMIT 10;
```

PostgreSQL primary bottlenecks:
- **WAL write throughput:** All writes go through Write-Ahead Log. At high write volumes, `fsync` becomes the bottleneck.
- **Lock contention:** `SELECT FOR UPDATE`, table-level locks, deadlocks increase with concurrent writers.
- **Vacuuming:** MVCC requires periodic VACUUM. On high-write tables, autovacuum competes with queries.
- **Connection overhead:** Each backend process costs ~5MB RAM.

For an 80:20 read/write ratio, read replicas directly offload 80% of query volume from the primary.

##### 2. PostgreSQL Streaming Replication Setup

```bash
# On the Primary: postgresql.conf
wal_level = replica              # Enable WAL streaming
max_wal_senders = 5              # Max simultaneous replica connections
max_replication_slots = 5        # For logical replication and HA
wal_keep_size = 1GB              # Keep WAL segments for replicas that fall behind
synchronous_commit = on          # Durable writes (set to 'off' for performance boost, at durability cost)

# pg_hba.conf — Allow replica connections
# TYPE  DATABASE    USER         ADDRESS           METHOD
  host  replication replicator   replica-1.internal md5
  host  replication replicator   replica-2.internal md5
```

```bash
# Bootstrap the replica (run on replica server)
pg_basebackup \
    -h primary.internal \
    -U replicator \
    -p 5432 \
    -D /var/lib/postgresql/data \
    --wal-method=stream \
    --checkpoint=fast \
    --progress \
    --verbose

# postgresql.conf on replica
hot_standby = on               # Allow read queries on replica
max_standby_streaming_delay = 30s  # Cancel conflicting queries after 30s lag

# recovery.conf (or postgresql.conf in PG12+)
primary_conninfo = 'host=primary.internal user=replicator password=secret'
primary_slot_name = 'replica1_slot'  # Use replication slot to prevent WAL gap
```

##### 3. Django Database Router — Full Implementation

```python
# settings.py
DATABASES = {
    'default': {  # Primary — all writes go here
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'pg-primary.internal',
        'PORT': '5432',
        'USER': 'myuser', 'PASSWORD': 'secret',
        'CONN_MAX_AGE': 0,
    },
    'replica1': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'pg-replica-1.internal',  # Read replica 1 (e.g., us-east-1a)
        'PORT': '5432',
        'USER': 'myuser_readonly', 'PASSWORD': 'secret',
        'CONN_MAX_AGE': 0,
        'TEST': {'MIRROR': 'default'},  # Use primary during test runs
    },
    'replica2': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'pg-replica-2.internal',  # Read replica 2 (e.g., us-east-1b)
        'PORT': '5432',
        'USER': 'myuser_readonly', 'PASSWORD': 'secret',
        'CONN_MAX_AGE': 0,
        'TEST': {'MIRROR': 'default'},
    },
}

DATABASE_ROUTERS = ['myproject.db_router.PrimaryReplicaRouter']
```

```python
# db_router.py
import random
import threading
from django.conf import settings

# Thread-local storage: force queries to primary within a request/transaction context
_thread_local = threading.local()

class PrimaryReplicaRouter:
    """
    Routes read queries to replicas (round-robin) and all writes to the primary.
    Respects manual primary hints via context manager for post-write reads.
    """
    
    READ_REPLICAS = ['replica1', 'replica2']
    PRIMARY = 'default'
    
    def db_for_read(self, model, **hints):
        """
        Route reads.
        - If a migration is running: always use primary.
        - If we're in a 'force_primary' context (post-write): use primary.
        - Otherwise: round-robin across replicas.
        """
        if getattr(_thread_local, 'force_primary', False):
            return self.PRIMARY
        
        # Check model-level routing hint
        if hints.get('instance') and hasattr(hints['instance'], '_state'):
            db = hints['instance']._state.db
            if db and db != self.PRIMARY:
                return db  # Respect existing DB hint (e.g., object fetched from replica1)
        
        return random.choice(self.READ_REPLICAS)
    
    def db_for_write(self, model, **hints):
        """All writes always go to primary."""
        return self.PRIMARY
    
    def allow_relation(self, obj1, obj2, **hints):
        """
        Allow relations between objects from any configured DB.
        Required when objects are fetched from different databases.
        """
        db_set = {self.PRIMARY} | set(self.READ_REPLICAS)
        if obj1._state.db in db_set and obj2._state.db in db_set:
            return True
        return None
    
    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """Migrations only run on the primary."""
        return db == self.PRIMARY


# context_managers.py
from contextlib import contextmanager

@contextmanager
def use_primary():
    """
    Context manager to force all queries within a block to use the primary.
    Essential after writes to avoid reading stale data from replicas.
    
    Usage:
        order = Order.objects.create(...)
        with use_primary():
            order_from_db = Order.objects.get(id=order.id)  # Guaranteed fresh
    """
    old_value = getattr(_thread_local, 'force_primary', False)
    _thread_local.force_primary = True
    try:
        yield
    finally:
        _thread_local.force_primary = old_value


# Decorator version for entire views
def primary_reads(func):
    """Decorator: forces all DB reads within the view to use the primary."""
    from functools import wraps
    @wraps(func)
    def wrapper(*args, **kwargs):
        with use_primary():
            return func(*args, **kwargs)
    return wrapper
```

**Usage in views:**

```python
# views.py
from django.db import transaction
from .db_router import use_primary
from .models import Order

def create_order(request):
    with transaction.atomic():
        order = Order.objects.create(
            user=request.user,
            total=cart.total,
            status='pending'
        )
    
    # ⚠️ Without use_primary(), Django might read from a lagging replica
    # and return a 404 for an order that was just created
    with use_primary():
        # Guaranteed to read from primary — sees the just-committed write
        return redirect('order_detail', order_id=order.id)

# For read-heavy views that can tolerate slight staleness:
def product_catalog(request):
    # This automatically goes to a replica via the router
    products = Product.objects.filter(is_active=True).select_related('category')
    return render(request, 'catalog.html', {'products': products})
```

---

### Q8: Replication Lag — Detection & Application-Level Handling

**Interview Question:** *"After deploying read replicas, a user creates an order and immediately gets a 404 when redirected to the order detail page. Explain replication lag, how to measure it, and implement multiple strategies to handle it in Django — including the 'read your own writes' pattern."*

---

#### Answer

##### 1. What Replication Lag Is — Mechanically

PostgreSQL streaming replication is **asynchronous by default**. The sequence is:

```
Primary:  COMMIT → Write WAL → Send WAL to replica → Return to client
                                      ↓ (network + I/O latency)
Replica:  Receive WAL → Apply WAL → Update local data pages

Replication Lag = Time between primary COMMIT and replica applying that WAL record
```

Typical lag: 10ms–5s under normal load. Under heavy write load or network issues: 10s–minutes. This means a user who just created an order may query a replica that hasn't yet replicated that row.

##### 2. Measuring Lag

```sql
-- On the primary: Check replication lag per connected replica
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    -- LSN difference converted to bytes
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes,
    -- Human-readable time lag
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

-- On the replica: Self-reported lag from replica's perspective
SELECT 
    now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

##### 3. Application-Level Strategies

**Strategy 1: Route post-write reads to primary (use_primary context manager)**

Already shown in Q7. This is the simplest and most reliable approach but adds primary load.

**Strategy 2: Read-Your-Own-Writes via LSN Tracking**

PostgreSQL can report the current WAL LSN after a write. Store it in the session. Before reading from a replica, check if the replica has caught up to at least that LSN.

```python
# services/lsn_tracking.py
from django.db import connection

def get_current_lsn() -> str:
    """Get the current WAL position from the primary after a write."""
    with connection.cursor() as cursor:
        cursor.execute("SELECT pg_current_wal_lsn()::text")
        return cursor.fetchone()[0]

def replica_has_caught_up(required_lsn: str, db_alias: str) -> bool:
    """Check if a replica has replicated up to the given LSN."""
    from django.db import connections
    with connections[db_alias].cursor() as cursor:
        cursor.execute(
            "SELECT pg_last_wal_replay_lsn() >= %s::pg_lsn",
            [required_lsn]
        )
        return cursor.fetchone()[0]
```

```python
# middleware.py — LSN-aware routing middleware
from django.db import connection
from .services.lsn_tracking import get_current_lsn, replica_has_caught_up
from .db_router import _thread_local

class LsnAwareReplicaMiddleware:
    """
    After any write request, stores the WAL LSN in the session.
    On subsequent read requests, checks if the chosen replica has caught up.
    Falls back to primary if replica is behind.
    """
    WRITE_METHODS = {'POST', 'PUT', 'PATCH', 'DELETE'}
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Before request: check if we need to verify replica lag
        required_lsn = request.session.get('min_db_lsn')
        if required_lsn and request.method not in self.WRITE_METHODS:
            self._maybe_force_primary(required_lsn)
        
        response = self.get_response(request)
        
        # After write requests: capture current LSN and store in session
        if request.method in self.WRITE_METHODS and hasattr(response, 'status_code'):
            if 200 <= response.status_code < 400:
                try:
                    lsn = get_current_lsn()
                    request.session['min_db_lsn'] = lsn
                    request.session['lsn_set_at'] = timezone.now().isoformat()
                except Exception:
                    pass  # Don't fail requests due to LSN tracking failure
        
        return response
    
    def _maybe_force_primary(self, required_lsn: str):
        """Force primary if replica hasn't caught up."""
        # Try replica1 first
        if not replica_has_caught_up(required_lsn, 'replica1'):
            _thread_local.force_primary = True
```

**Strategy 3: Monotonic Read Guarantee via Version Token**

A simpler alternative — store a write timestamp in the session. Within a configurable window after a write, force reads to the primary:

```python
# middleware.py
from django.utils import timezone
import datetime

class MonotonicReadMiddleware:
    """
    After a write, forces reads to the primary for a window long enough
    for replicas to catch up (configured as POST_WRITE_PRIMARY_WINDOW_SECONDS).
    """
    POST_WRITE_PRIMARY_WINDOW_SECONDS = 5  # Tune based on observed replication lag
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        last_write_at_str = request.session.get('last_write_at')
        
        if last_write_at_str:
            last_write_at = datetime.datetime.fromisoformat(last_write_at_str)
            elapsed = (timezone.now() - last_write_at).total_seconds()
            
            if elapsed < self.POST_WRITE_PRIMARY_WINDOW_SECONDS:
                _thread_local.force_primary = True  # Still within window
            else:
                # Window passed — safe to use replicas again
                del request.session['last_write_at']
                _thread_local.force_primary = False
        
        response = self.get_response(request)
        
        # Track writes
        if request.method in ('POST', 'PUT', 'PATCH', 'DELETE'):
            if 200 <= response.status_code < 400:
                request.session['last_write_at'] = timezone.now().isoformat()
        
        return response
```

---

### Q9: Pessimistic vs Optimistic Locking Under High Concurrency

**Interview Question:** *"You have a flash sale where 10,000 users simultaneously try to purchase the last 100 units of a product. Explain what happens without any concurrency control, then implement both Pessimistic and Optimistic locking in Django/PostgreSQL. When would you choose one over the other?"*

---

#### Answer

##### 1. The Race Condition — What Happens Without Locking

```python
# BROKEN — Classic race condition (lost update problem)
def purchase(request, product_id, quantity):
    product = Product.objects.get(id=product_id)
    
    # T1 reads: stock = 100
    # T2 reads: stock = 100  (T1 hasn't committed yet)
    if product.stock >= quantity:
        # T1 computes: new_stock = 100 - 1 = 99
        # T2 computes: new_stock = 100 - 1 = 99  (T2 read the same old value)
        product.stock -= quantity
        product.save()
        # T1 commits: stock = 99
        # T2 commits: stock = 99  ← WRONG! Stock should be 98 after two purchases
        # With 10,000 concurrent users, you could sell 10,000 items from 100 stock
        return HttpResponse("Purchase successful")
    return HttpResponse("Out of stock", status=400)
```

##### 2. Pessimistic Locking — `SELECT FOR UPDATE`

Pessimistic locking acquires an exclusive row-level lock the moment the row is read. All other transactions that try to read or update the same row are **blocked** until the lock holder commits or rolls back.

```python
# views.py — Pessimistic locking with SELECT FOR UPDATE
from django.db import transaction
from django.db.models import F

@transaction.atomic
def purchase_pessimistic(request, product_id: int, quantity: int):
    """
    Pessimistic locking: serialize access to the product row.
    T2 BLOCKS at the SELECT FOR UPDATE until T1 commits.
    Guarantees correctness at the cost of throughput.
    """
    try:
        # SELECT * FROM product WHERE id = %s FOR UPDATE NOWAIT
        # NOWAIT: Don't block — immediately raise LockNotAvailable if row is locked.
        # Use SKIP LOCKED to skip locked rows (e.g., for queue processing).
        product = (
            Product.objects
            .select_for_update(nowait=False, skip_locked=False)
            .get(id=product_id)
        )
    except Product.DoesNotExist:
        return HttpResponse("Product not found", status=404)
    
    # Now we hold the lock. No other transaction can touch this row.
    if product.stock < quantity:
        # Lock is released on transaction end (via atomic context manager)
        return HttpResponse("Insufficient stock", status=409)
    
    # Use F() expression to avoid a Python-level read
    # This generates: UPDATE product SET stock = stock - 1 WHERE id = X
    Product.objects.filter(id=product_id).update(stock=F('stock') - quantity)
    
    Order.objects.create(
        user=request.user,
        product=product,
        quantity=quantity,
        unit_price=product.price,
    )
    
    # Transaction commits here → lock released → next waiter proceeds
    return HttpResponse("Purchase successful", status=201)
```

**The key clauses:**

```python
# SKIP LOCKED — Skip rows locked by other transactions (useful for queue workers)
Product.objects.select_for_update(skip_locked=True).filter(status='pending')

# NOWAIT — Immediately raise django.db.utils.OperationalError (LockNotAvailable)
# instead of waiting. Use for fast-fail scenarios (user-facing APIs).
try:
    product = Product.objects.select_for_update(nowait=True).get(id=product_id)
except OperationalError:
    return HttpResponse("Server busy, please try again", status=503)

# OF — Lock specific tables in a multi-table query (PostgreSQL 9.5+)
product = Product.objects.select_related('category').select_for_update(of=('self',)).get(id=product_id)
```

##### 3. Optimistic Locking — Version Field Pattern

Optimistic locking assumes conflicts are rare. No locks are held. Instead, a version counter is included in the `WHERE` clause of the UPDATE, causing it to affect 0 rows if another transaction has already modified the row.

```python
# models.py — Add a version field
class Product(models.Model):
    name = models.CharField(max_length=200)
    stock = models.PositiveIntegerField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    version = models.PositiveIntegerField(default=0)  # Optimistic lock version
    
    class Meta:
        indexes = [models.Index(fields=['id', 'version'])]
```

```python
# views.py — Optimistic locking
from django.db import transaction
from django.db.models import F

MAX_OPTIMISTIC_RETRIES = 3

def purchase_optimistic(request, product_id: int, quantity: int):
    """
    Optimistic locking: no blocking. Retries on conflict.
    Best for: Low contention scenarios. Degrades under high contention.
    """
    for attempt in range(MAX_OPTIMISTIC_RETRIES):
        # Read current state — no lock acquired
        try:
            product = Product.objects.get(id=product_id)
        except Product.DoesNotExist:
            return HttpResponse("Not found", status=404)
        
        if product.stock < quantity:
            return HttpResponse("Insufficient stock", status=409)
        
        # Attempt update with version check
        # SQL: UPDATE product
        #      SET stock = stock - 1, version = version + 1
        #      WHERE id = X AND version = <version_we_read> AND stock >= quantity
        rows_updated = Product.objects.filter(
            id=product_id,
            version=product.version,     # The optimistic lock check
            stock__gte=quantity           # Guard against race on the stock itself
        ).update(
            stock=F('stock') - quantity,
            version=F('version') + 1
        )
        
        if rows_updated == 1:
            # Success — our version matched, update was applied
            Order.objects.create(
                user=request.user,
                product_id=product_id,
                quantity=quantity,
                unit_price=product.price,
            )
            return HttpResponse("Purchase successful", status=201)
        
        # rows_updated == 0: another transaction committed first, version mismatch
        # Retry the loop
        if attempt < MAX_OPTIMISTIC_RETRIES - 1:
            import time
            time.sleep(0.05 * (attempt + 1))  # Brief backoff before retry
    
    # All retries exhausted
    return HttpResponse("Could not complete purchase due to high demand. Please try again.", status=409)
```

##### 4. Decision Matrix

| Factor | Pessimistic | Optimistic |
|---|---|---|
| **Contention level** | High (>30% conflict rate) | Low (<10% conflict rate) |
| **Transaction duration** | Short (milliseconds) | Variable |
| **Retry complexity** | None (blocking) | Must implement |
| **Deadlock risk** | Yes (mitigate with consistent ordering) | None |
| **Throughput under contention** | Degrades linearly | Degrades exponentially |
| **Django ORM support** | `select_for_update()` | Version field + filtered update |
| **Best use case** | Flash sales, inventory deduction | User profile updates, settings |

**For the flash sale scenario (10,000 concurrent users, 100 units):** Use **pessimistic locking** (`SELECT FOR UPDATE`). The contention rate is nearly 100% — almost every transaction is competing for the same rows. Optimistic locking would cause constant retries, amplifying database load by 3–5× per conflicting transaction.

**Additional optimization — pre-check with Redis:**

```python
# Fast pre-check using Redis atomic DECR before hitting PostgreSQL
import redis
r = redis.Redis.from_url(settings.REDIS_URL)

def purchase_with_redis_precheck(request, product_id: int, quantity: int):
    redis_key = f"product:stock:{product_id}"
    
    # Atomic decrement — returns new value
    new_stock = r.decrby(redis_key, quantity)
    
    if new_stock < 0:
        # Undo the decrement — insufficient stock
        r.incrby(redis_key, quantity)
        return HttpResponse("Out of stock", status=409)
    
    # Stock "reserved" in Redis — now persist to DB
    # This can be done asynchronously via Celery
    persist_purchase.delay(product_id=product_id, user_id=request.user.id, quantity=quantity)
    return HttpResponse("Purchase successful", status=201)
```

---

## Part 4: Microservices & Real-World Scenarios

---

### Q10: Design an E-Commerce Checkout System for 10,000 TPS

**Interview Question:** *"Design the backend architecture for an e-commerce checkout flow that must handle 10,000 transactions per second during peak sale events. Cover: service decomposition, API Gateway design with rate limiting (implement Token Bucket in Redis), and how you'd handle the distributed transaction between inventory deduction and payment processing using the Saga pattern vs 2PC."*

---

#### Answer

##### 1. Requirements & Back-of-Envelope Calculation

```
Target: 10,000 TPS peak checkout throughput
Average checkout: 5 service calls (cart validation, inventory, pricing, payment, order creation)
Total internal RPCs: 10,000 × 5 = 50,000 req/sec internal traffic

P99 checkout latency target: < 2 seconds
Average order value: $75
Peak: Black Friday 10× normal load

Infrastructure estimate:
- API Gateway: 3 × c5.2xlarge (load balanced) — handles ~5,000 req/sec each
- Checkout Service: 10 × c5.4xlarge with uvicorn/async Django — ~1,000 TPS each
- Inventory Service: 5 × c5.2xlarge — fast reads from Redis + async DB writes
- Payment Service: 3 × c5.xlarge (bottlenecked by payment gateway API, not compute)
- PostgreSQL: 1 primary (r5.4xlarge) + 3 read replicas
- Redis Cluster: 6 nodes (3 primary + 3 replica), 64GB each
- Message Queue (RabbitMQ/SQS): 3-node cluster
```

##### 2. Service Decomposition & Architecture

```
                          ┌─────────────────────────────────┐
  Client (Browser/App)    │         CDN / DDoS Protection    │
         │                └─────────────┬───────────────────┘
         │                              │
         ▼                              ▼
  ┌──────────────────────────────────────────────────────────┐
  │                    API Gateway                            │
  │  • TLS Termination   • Auth (JWT Validation)             │
  │  • Rate Limiting     • Request Routing                   │
  │  • Request Logging   • Response Caching (GET only)       │
  └────────────┬─────────────────────────────────────────────┘
               │
    ┌──────────┼──────────────────────┐
    ▼          ▼                      ▼
┌────────┐ ┌──────────┐         ┌──────────────┐
│  Cart  │ │ Checkout │         │   Product    │
│Service │ │ Service  │         │   Service    │
└────────┘ └────┬─────┘         └──────────────┘
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
┌──────────┐ ┌──────┐ ┌─────────┐
│Inventory │ │Price │ │ Payment │
│ Service  │ │  Svc │ │ Service │
└──────────┘ └──────┘ └─────────┘
       │                    │
       ▼                    ▼
   ┌────────┐          ┌─────────────┐
   │  Redis │          │  Payment    │
   │(stock) │          │  Gateway   │
   └────────┘          │ (Stripe)   │
                       └─────────────┘
                    
Message Bus (RabbitMQ):
  checkout.completed → [Order Service, Notification Service, Analytics]
  payment.failed     → [Refund Service, Notification Service]
```

##### 3. Rate Limiting — Token Bucket in Redis

The Token Bucket algorithm allows controlled bursting. Each user/IP has a "bucket" that fills at a fixed rate. Each request consumes a token. If the bucket is empty, the request is rejected.

```python
# rate_limiter.py — Token Bucket implementation using Redis atomic Lua script
import redis
import time
from dataclasses import dataclass
from typing import Tuple

@dataclass
class RateLimitResult:
    allowed: bool
    remaining_tokens: int
    retry_after_seconds: float

class TokenBucketRateLimiter:
    """
    Token Bucket algorithm implemented atomically in Redis Lua.
    
    Algorithm:
    1. Bucket has a max capacity (burst limit).
    2. Tokens refill at a fixed rate (tokens/second).
    3. Each request consumes 1 token.
    4. Request is rejected if bucket is empty.
    
    Atomicity: The Lua script runs atomically on Redis, preventing race conditions.
    """
    
    # Lua script for atomic token bucket check-and-consume
    LUA_SCRIPT = """
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])         -- Max tokens (burst limit)
    local refill_rate = tonumber(ARGV[2])      -- Tokens added per second
    local now = tonumber(ARGV[3])              -- Current Unix timestamp (milliseconds)
    local requested = tonumber(ARGV[4])        -- Tokens to consume (usually 1)
    
    -- Get current state from Redis
    local last_refill = tonumber(redis.call('HGET', key, 'last_refill')) or now
    local tokens = tonumber(redis.call('HGET', key, 'tokens')) or capacity
    
    -- Calculate tokens added since last refill
    local elapsed_seconds = (now - last_refill) / 1000.0
    local new_tokens = elapsed_seconds * refill_rate
    tokens = math.min(capacity, tokens + new_tokens)
    
    local allowed = 0
    local retry_after = 0
    
    if tokens >= requested then
        -- Consume tokens and allow the request
        tokens = tokens - requested
        allowed = 1
    else
        -- Calculate when enough tokens will be available
        local deficit = requested - tokens
        retry_after = deficit / refill_rate
    end
    
    -- Persist updated state with TTL (2× the refill window)
    redis.call('HSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) * 2)
    
    return {allowed, math.floor(tokens), retry_after * 1000}
    """
    
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self._script = self.redis.register_script(self.LUA_SCRIPT)
    
    def check_and_consume(
        self,
        identifier: str,         # e.g., "user:123" or "ip:192.168.1.1"
        capacity: int = 100,     # Max burst: 100 tokens
        refill_rate: float = 10, # Refill at 10 tokens/second
        requested: int = 1,
    ) -> RateLimitResult:
        key = f"rate_limit:token_bucket:{identifier}"
        now_ms = int(time.time() * 1000)
        
        result = self._script(
            keys=[key],
            args=[capacity, refill_rate, now_ms, requested]
        )
        
        allowed, remaining, retry_after_ms = result
        return RateLimitResult(
            allowed=bool(allowed),
            remaining_tokens=int(remaining),
            retry_after_seconds=retry_after_ms / 1000.0
        )


# Checkout-specific rate limits
CHECKOUT_RATE_LIMITS = {
    'checkout_initiate': {'capacity': 10, 'refill_rate': 2},   # 2/sec, burst of 10
    'payment_attempt':   {'capacity': 3,  'refill_rate': 0.1}, # 1 per 10 sec, burst of 3
    'cart_update':       {'capacity': 50, 'refill_rate': 5},   # 5/sec, burst of 50
}
```

```python
# middleware.py — API Gateway rate limiting middleware
from django.http import JsonResponse
from .rate_limiter import TokenBucketRateLimiter
import redis

r = redis.Redis.from_url(settings.REDIS_URL, decode_responses=True)
limiter = TokenBucketRateLimiter(r)

class CheckoutRateLimitMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Identify the requester (prefer user ID over IP for logged-in users)
        if request.user.is_authenticated:
            identifier = f"user:{request.user.id}"
        else:
            identifier = f"ip:{self.get_client_ip(request)}"
        
        # Apply endpoint-specific limits
        endpoint_key = self.get_endpoint_key(request)
        if endpoint_key and endpoint_key in CHECKOUT_RATE_LIMITS:
            config = CHECKOUT_RATE_LIMITS[endpoint_key]
            result = limiter.check_and_consume(
                identifier=f"{identifier}:{endpoint_key}",
                **config
            )
            
            if not result.allowed:
                response = JsonResponse(
                    {
                        'error': 'rate_limit_exceeded',
                        'message': f'Too many requests. Try again in {result.retry_after_seconds:.1f} seconds.',
                        'retry_after': result.retry_after_seconds
                    },
                    status=429
                )
                response['Retry-After'] = str(int(result.retry_after_seconds) + 1)
                response['X-RateLimit-Remaining'] = '0'
                return response
            
            response = self.get_response(request)
            response['X-RateLimit-Remaining'] = str(result.remaining_tokens)
            return response
        
        return self.get_response(request)
    
    def get_client_ip(self, request) -> str:
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            return x_forwarded_for.split(',')[0].strip()
        return request.META.get('REMOTE_ADDR', '0.0.0.0')
    
    def get_endpoint_key(self, request) -> str:
        path = request.path
        if '/api/checkout/initiate' in path:
            return 'checkout_initiate'
        if '/api/checkout/payment' in path:
            return 'payment_attempt'
        if '/api/cart' in path and request.method in ('POST', 'PUT', 'PATCH'):
            return 'cart_update'
        return ''
```

##### 4. Distributed Transactions — Saga Pattern vs 2PC

The checkout flow spans multiple services. **Atomicity is the fundamental challenge:** if payment succeeds but inventory deduction fails (or vice versa), the system is in an inconsistent state.

**Option A: Two-Phase Commit (2PC) — Why Not to Use It**

2PC requires a central coordinator that runs a prepare phase (all participants lock resources) and a commit phase (all participants commit). Under 10,000 TPS:

```
Problems with 2PC:
1. BLOCKING: All participants hold locks during coordinator round-trip.
   At 10ms network latency × 2 phases = 20ms of locked inventory rows.
   At 10,000 TPS, the inventory table is permanently locked.

2. COORDINATOR SPOF: If the coordinator crashes between prepare and commit,
   participants are stuck holding locks indefinitely until coordinator recovery.

3. Latency: 2 full round trips to all participants mandatory.
   Unacceptable at scale.

Conclusion: 2PC is appropriate only for same-datacenter, low-TPS scenarios
(e.g., financial batch processing at <100 TPS).
```

**Option B: Saga Pattern — The Correct Choice at 10,000 TPS**

A Saga is a sequence of local transactions, each publishing an event that triggers the next step. If a step fails, compensating transactions run in reverse to undo previous steps.

```
                    Choreography-based Saga (Event-Driven)

User clicks "Pay"
       │
       ▼
1. Checkout Service creates Order (status: PENDING)
   └─► Publishes: order.created → Message Bus
       │
       ▼
2. Inventory Service consumes order.created
   └─► Reserves stock (marks as reserved, not deducted)
   └─► Publishes: inventory.reserved OR inventory.insufficient
       │
       ├── inventory.insufficient ──► Checkout Service marks order FAILED
       │                              User notified: "Item sold out"
       │
       └── inventory.reserved ──────► Payment Service consumes inventory.reserved
           │
           ▼
3. Payment Service processes payment
   └─► Calls Stripe API
   └─► Publishes: payment.succeeded OR payment.failed
       │
       ├── payment.failed ──────────► Inventory Service compensates (releases reservation)
       │                              Checkout Service marks order FAILED
       │                              User notified: "Payment declined"
       │
       └── payment.succeeded ───────► Inventory Service confirms deduction (RESERVED → DEDUCTED)
                                      Checkout Service marks order CONFIRMED
                                      Notification Service sends email
                                      Analytics Service records transaction
```

**Implementation:**

```python
# checkout_service/sagas.py

# Step 1: Checkout Service initiates the saga
from dataclasses import dataclass, asdict
from enum import Enum
import uuid

class OrderStatus(str, Enum):
    PENDING = 'pending'
    INVENTORY_RESERVED = 'inventory_reserved'
    PAYMENT_PROCESSING = 'payment_processing'
    CONFIRMED = 'confirmed'
    FAILED = 'failed'
    COMPENSATING = 'compensating'  # Running compensating transactions

@dataclass
class CheckoutSagaData:
    saga_id: str
    order_id: int
    user_id: int
    product_id: int
    quantity: int
    amount_cents: int
    payment_method_token: str

def initiate_checkout_saga(cart, user, payment_token) -> dict:
    """
    Creates the order record and kicks off the Saga by publishing the first event.
    """
    saga_id = str(uuid.uuid4())  # Unique saga identifier for correlation
    
    # Local transaction — create order in PENDING state
    order = Order.objects.create(
        user=user,
        product=cart.product,
        quantity=cart.quantity,
        total_cents=cart.total_cents,
        status=OrderStatus.PENDING,
        saga_id=saga_id,  # Store saga_id for tracing
    )
    
    saga_data = CheckoutSagaData(
        saga_id=saga_id,
        order_id=order.id,
        user_id=user.id,
        product_id=cart.product_id,
        quantity=cart.quantity,
        amount_cents=cart.total_cents,
        payment_method_token=payment_token,
    )
    
    # Publish event to message bus — triggers next saga step
    publish_event('order.created', asdict(saga_data))
    
    return {'saga_id': saga_id, 'order_id': order.id, 'status': 'processing'}
```

```python
# inventory_service/event_handlers.py

def handle_order_created(event_data: dict):
    """
    Inventory Service: reserve stock when order is created.
    This is a compensatable step.
    """
    saga_id = event_data['saga_id']
    product_id = event_data['product_id']
    quantity = event_data['quantity']
    
    # Attempt to reserve stock atomically
    reserved = (
        Product.objects
        .filter(id=product_id, stock__gte=quantity)
        .update(
            stock=F('stock') - quantity,
            reserved_stock=F('reserved_stock') + quantity
        )
    )
    
    if reserved:
        InventoryReservation.objects.create(
            saga_id=saga_id,
            product_id=product_id,
            quantity=quantity,
            status='reserved'
        )
        publish_event('inventory.reserved', event_data)
    else:
        publish_event('inventory.insufficient', {
            **event_data,
            'reason': 'insufficient_stock'
        })

def handle_payment_failed(event_data: dict):
    """
    Compensating transaction: release inventory reservation on payment failure.
    This is the key to Saga's rollback mechanism.
    """
    saga_id = event_data['saga_id']
    
    reservation = InventoryReservation.objects.filter(
        saga_id=saga_id,
        status='reserved'
    ).first()
    
    if reservation:
        # Compensate: move stock from reserved back to available
        Product.objects.filter(id=reservation.product_id).update(
            stock=F('stock') + reservation.quantity,
            reserved_stock=F('reserved_stock') - reservation.quantity
        )
        reservation.status = 'released'
        reservation.save()
        
        publish_event('inventory.compensation_completed', {'saga_id': saga_id})
```

```python
# payment_service/event_handlers.py

def handle_inventory_reserved(event_data: dict):
    """
    Payment Service: process payment once inventory is secured.
    """
    import stripe
    
    try:
        charge = stripe.PaymentIntent.create(
            amount=event_data['amount_cents'],
            currency='usd',
            payment_method=event_data['payment_method_token'],
            confirm=True,
            idempotency_key=event_data['saga_id'],  # Stripe idempotency key
            metadata={'order_id': event_data['order_id'], 'saga_id': event_data['saga_id']}
        )
        
        PaymentRecord.objects.create(
            saga_id=event_data['saga_id'],
            order_id=event_data['order_id'],
            stripe_payment_intent_id=charge.id,
            amount_cents=event_data['amount_cents'],
            status='succeeded'
        )
        
        publish_event('payment.succeeded', {
            **event_data,
            'payment_intent_id': charge.id
        })
        
    except stripe.error.CardError as e:
        publish_event('payment.failed', {
            **event_data,
            'failure_reason': e.code,  # e.g., 'card_declined', 'insufficient_funds'
        })
    except Exception as exc:
        # Unexpected failure — also trigger compensation
        publish_event('payment.failed', {**event_data, 'failure_reason': 'internal_error'})
        raise
```

**Saga vs 2PC Decision Summary:**

| Criteria | 2PC | Saga (Choreography) |
|---|---|---|
| **Throughput at 10K TPS** | ❌ Impossible (lock contention) | ✅ Scales horizontally |
| **Latency** | ❌ 2 round-trips, blocking | ✅ Async, non-blocking |
| **Failure recovery** | ❌ Coordinator SPOF | ✅ Each service self-heals |
| **Eventual consistency** | Immediate | ✅ Within seconds |
| **Complexity** | Simple to reason about | Requires careful compensating tx design |
| **Debugging** | Easy (one transaction) | Harder (distributed trace required) |
| **Use case** | Low-volume, same-datacenter, banking batch | High-volume, microservices, e-commerce |

**Observability for the Saga:**

```python
# Every event should carry a correlation ID for distributed tracing
# Use OpenTelemetry to trace across service boundaries

from opentelemetry import trace
from opentelemetry.propagate import inject, extract

def publish_event(event_type: str, data: dict):
    """Publish event with distributed trace context."""
    tracer = trace.get_tracer(__name__)
    with tracer.start_as_current_span(f"publish.{event_type}") as span:
        span.set_attribute("saga.id", data.get('saga_id', 'unknown'))
        span.set_attribute("event.type", event_type)
        
        # Inject trace context into the event payload for downstream services
        headers = {}
        inject(headers)
        
        message_bus.publish(
            routing_key=event_type,
            body=data,
            headers=headers,  # Trace context propagated to consumers
        )
```

---

## Summary Reference Card

| Topic | Key Concept | Django Tool / Pattern |
|---|---|---|
| WSGI vs ASGI | Sync vs async I/O model | Gunicorn → Uvicorn migration |
| N+1 Queries | Lazy ORM evaluation | `select_related`, `prefetch_related`, `annotate` |
| Connection Pooling | Per-worker connections | PgBouncer transaction mode, `CONN_MAX_AGE=0` |
| Session Scaling | Server-side state cost | `SESSION_ENGINE = cache` + Redis HA |
| Task Reliability | Broker acknowledgment timing | `acks_late=True`, `reject_on_worker_lost=True` |
| Idempotency | Safe re-execution | Redis `SET NX` idempotency key + DB state guard |
| Dead Letter Queue | Poison pill isolation | Custom DLQ task + DB audit record |
| Read Replicas | Write/read separation | `DATABASE_ROUTERS`, round-robin reads |
| Replication Lag | Async WAL application | `use_primary()` context manager, LSN tracking |
| Pessimistic Lock | Serialized row access | `select_for_update(nowait=True)` |
| Optimistic Lock | Conflict detection on write | Version field + filtered `.update()` |
| Rate Limiting | Token Bucket | Redis Lua atomic script |
| Distributed Tx | Cross-service atomicity | Saga pattern with compensating transactions |

---

*Document Version: 1.0 | Audience: Python/Django Engineers (3+ YoE) | Level: Staff / Senior*
