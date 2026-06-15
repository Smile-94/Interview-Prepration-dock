# Production-Grade Excel Ingestion System — Design Document

**Stack:** Django + DRF, Celery, Redis, PostgreSQL, S3 (django-storages), openpyxl / xlsxwriter

---

## 1. Goals and Constraints

The system ingests very large XLS/XLSX files (hundreds of thousands to millions of rows), validates every row against strict business rules, persists valid rows in transactional batches, and produces a styled Excel error report containing only the failed rows with per-cell highlighting and human-readable error messages. Processing is fully asynchronous, idempotent, resumable after worker crashes, observable (progress %, audit metrics), and never loads a whole workbook into memory.

Non-goals: real-time streaming ingestion (this is batch-oriented), editing files in place.

---

## 2. High-Level Architecture

```
                ┌──────────────────────────────────────────────────────┐
                │                      Client / UI                     │
                │  (upload → poll progress → download error report)    │
                └───────┬──────────────────────────────▲───────────────┘
                        │ 1. presigned PUT             │ 6. progress API / WebSocket
                        ▼                              │
   ┌────────────┐   2. notify    ┌─────────────────────┴───┐
   │  S3 bucket │◄───────────────│   Django API (DRF)      │
   │ (uploads,  │                │  - create ImportJob     │
   │  reports)  │                │  - enqueue Celery task  │
   └─────┬──────┘                └───────────┬─────────────┘
         │                                   │ 3. job_id on queue
         │ 4. stream file                    ▼
         │                        ┌─────────────────────────┐
         └───────────────────────►│  Celery worker(s)       │
                                  │  ┌───────────────────┐  │
                                  │  │ StreamingParser   │  │  rows → chunks (e.g. 2 000)
                                  │  ├───────────────────┤  │
                                  │  │ ValidationEngine  │  │  business rules + FK checks
                                  │  ├───────────────────┤  │
                                  │  │ BatchWriter       │  │  bulk_create in atomic blocks
                                  │  ├───────────────────┤  │
                                  │  │ ErrorReportWriter │  │  failed rows → styled XLSX
                                  │  └───────────────────┘  │
                                  └──────┬─────────┬────────┘
                                         │         │ 5. progress + checkpoints
                                         ▼         ▼
                                  ┌──────────┐ ┌─────────┐
                                  │ Postgres │ │  Redis  │
                                  │ (data +  │ │ (progress│
                                  │ ImportJob│ │  cache,  │
                                  │ + audit) │ │  broker) │
                                  └──────────┘ └─────────┘
                                         │
                                         ▼ 7. completion
                                  ┌─────────────────┐
                                  │ Notifications   │ (email / WebSocket / webhook)
                                  └─────────────────┘
```

The web request path does nothing heavy: it records metadata, enqueues a task, and returns `202 Accepted` with a `job_id`. All parsing, validation, and writing happens in Celery workers that stream the file directly from storage.

---

## 3. Clean Architecture & Project Layout

The import pipeline lives in a dedicated app with strict layering. Domain logic (validators, row schemas, chunking policy) is plain Python with zero Django imports, so it is trivially unit-testable. Infrastructure (S3, ORM repositories, Celery) implements small interfaces consumed by the application layer.

```
apps/imports/
├── domain/                     # pure Python, no Django
│   ├── schema.py               # column definitions, types, constraints
│   ├── validators.py           # field & row validators (pure functions)
│   ├── results.py              # RowResult, ValidationError value objects
│   └── ports.py                # Protocols: FileSource, RecordRepository,
│                               #   ReferenceDataProvider, ProgressReporter
├── application/
│   ├── import_service.py       # the use case: orchestrates parse→validate→write
│   └── chunking.py             # chunk iterator, checkpoint logic
├── infrastructure/
│   ├── parsers.py              # XLSX/XLS streaming readers
│   ├── repositories.py         # Django ORM bulk writer, reference caches
│   ├── storage.py              # S3/local via django-storages
│   ├── error_report.py         # styled XLSX error file writer
│   └── progress.py             # Redis-backed progress reporter
├── interface/
│   ├── api/                    # DRF views & serializers (thin)
│   ├── tasks.py                # Celery tasks (thin wrappers around service)
│   └── notifications.py
├── models.py                   # ImportJob, ImportRowError (audit)
└── tests/
```

The `ImportService` receives its collaborators through the constructor (`parser`, `validator`, `writer`, `progress`, `error_sink`), so any piece can be swapped or faked in tests. Celery tasks contain no business logic — they resolve dependencies and call the service.

---

## 4. Data Model

```python
class ImportJob(models.Model):
    class Status(models.TextChoices):
        PENDING = "pending"; RUNNING = "running"
        COMPLETED = "completed"; COMPLETED_WITH_ERRORS = "completed_with_errors"
        FAILED = "failed"

    id            = models.UUIDField(primary_key=True, default=uuid4)
    created_by    = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    source_file   = models.FileField(upload_to="imports/uploads/%Y/%m/")
    file_sha256   = models.CharField(max_length=64, db_index=True)   # idempotency key
    import_type   = models.CharField(max_length=50)                  # maps to a schema
    status        = models.CharField(max_length=30, choices=Status.choices,
                                     default=Status.PENDING)

    total_rows        = models.PositiveIntegerField(null=True)
    processed_rows    = models.PositiveIntegerField(default=0)
    succeeded_rows    = models.PositiveIntegerField(default=0)
    failed_rows       = models.PositiveIntegerField(default=0)
    last_committed_chunk = models.IntegerField(default=-1)           # resume checkpoint

    error_report_file = models.FileField(null=True, upload_to="imports/reports/%Y/%m/")
    started_at    = models.DateTimeField(null=True)
    finished_at   = models.DateTimeField(null=True)
    duration_ms   = models.PositiveBigIntegerField(null=True)
    celery_task_id = models.CharField(max_length=64, blank=True)

    class Meta:
        constraints = [
            # idempotency: the same file can't be actively imported twice
            models.UniqueConstraint(
                fields=["file_sha256", "import_type"],
                condition=Q(status__in=["pending", "running"]),
                name="uniq_active_import_per_file",
            )
        ]


class ImportRowError(models.Model):           # optional row-level audit trail
    job        = models.ForeignKey(ImportJob, on_delete=models.CASCADE,
                                   related_name="row_errors")
    row_number = models.PositiveIntegerField()
    column     = models.CharField(max_length=100, blank=True)
    message    = models.TextField()
    raw_data   = models.JSONField()
```

`ImportJob` is both the workflow state machine and the audit record: totals, timings, who, when, and where the error report lives. `ImportRowError` makes failures queryable; it is written in bulk alongside the error-report file and can be capped (e.g., first 10 000 errors) to bound storage.

---

## 5. Upload Flow

Direct-to-S3 upload keeps multi-hundred-MB files off the Django workers entirely:

1. Client calls `POST /api/imports/presign/` with filename, size, and import type. The API enforces a size cap and extension whitelist, then returns a presigned S3 PUT URL plus an upload key.
2. Client PUTs the file to S3.
3. Client calls `POST /api/imports/` with the key. The API verifies the object exists, computes/records its SHA-256 (or trusts an S3 checksum), creates the `ImportJob`, and enqueues `process_import.delay(job_id)`. If an active job with the same hash exists, the API returns that job instead of creating a duplicate — request-level idempotency for free.
4. The response is `202 Accepted` with `{job_id, status_url}`.

For a local/dev deployment the same flow degrades to a normal multipart upload saved through Django's file storage abstraction; nothing downstream changes because workers only ever see a `FileSource` port.

---

## 6. Streaming Parsing (Constant Memory)

**XLSX** is a zip of XML; `openpyxl` in read-only mode parses it as a SAX stream, yielding one row at a time without materializing the sheet:

```python
def iter_xlsx_rows(file_obj) -> Iterator[tuple[int, dict]]:
    wb = load_workbook(file_obj, read_only=True, data_only=True)
    ws = wb.worksheets[0]
    rows = ws.iter_rows(values_only=True)
    header = normalize_header(next(rows))          # validate against schema here
    for idx, values in enumerate(rows, start=2):   # 1-based + header row
        if not any(v is not None for v in values):
            continue                               # skip blank rows
        yield idx, dict(zip(header, values))
    wb.close()
```

**XLS** (legacy BIFF) cannot be streamed by `xlrd`, which loads the whole book. Two strategies, chosen by file size: small XLS files (≤ ~20 MB) are read with `xlrd` directly; larger ones are converted once to XLSX with headless LibreOffice (`soffice --convert-to xlsx`) in the worker and then streamed normally. The conversion step is hidden behind the parser factory, so the rest of the pipeline is format-agnostic.

The worker streams the object from S3 to a temp file (`smart_open` or `boto3` download) rather than into RAM — openpyxl's read-only mode needs a seekable file handle, and a spooled temp file keeps memory flat.

Rows are grouped into chunks (default **2 000 rows**, tunable per import type) by a simple generator in `application/chunking.py`. Chunk size balances transaction size, lock duration, retry blast radius, and progress granularity.

Total row count for progress reporting comes from `ws.max_row` (read from the sheet's dimension record — cheap). When the dimension record is missing or unreliable, the system falls back to byte-offset progress on the underlying file, which is approximate but monotonic.

---

## 7. Validation Engine

Validation is declarative. Each import type defines a schema of columns; each column carries type coercion, requiredness, and constraint validators; the schema can also define cross-field row validators.

```python
# domain/schema.py
PRODUCT_SCHEMA = ImportSchema(
    columns=[
        Column("sku",        required=True,  coerce=str,
               validators=[regex(r"^[A-Z0-9-]{4,32}$")]),
        Column("name",       required=True,  coerce=str, validators=[max_len(255)]),
        Column("price",      required=True,  coerce=decimal,
               validators=[min_value(Decimal("0.01"))]),
        Column("quantity",   required=True,  coerce=int, validators=[min_value(0)]),
        Column("category",   required=True,  coerce=str,
               validators=[fk_exists("category")]),          # relational integrity
        Column("launch_date", required=False, coerce=date_ddmmyyyy),
    ],
    row_validators=[
        lambda row: err("launch_date", "Launch date cannot precede creation date")
        if row.get("launch_date") and row["launch_date"] < row["created_date"] else None
    ],
    unique_within_file=["sku"],
)
```

Each row produces a `RowResult`: either `Valid(payload)` or `Invalid(errors: list[CellError])` where `CellError = (column, message)`. Crucially, validation **collects all errors for a row** instead of failing fast, so the error report tells the user everything wrong with a row at once.

**Relational integrity without N+1 queries.** Validators like `fk_exists` never hit the database per row. A `ReferenceDataProvider` resolves them in one of two modes:

- *Preload*: for small reference tables (categories, statuses, warehouses), load all valid keys into an in-memory `set` once per job.
- *Per-chunk batch lookup*: for large reference tables (customers, SKUs), collect the distinct keys appearing in the chunk and run a single `filter(key__in=keys).values_list(...)` query, caching hits in an LRU dict for subsequent chunks.

**In-file uniqueness** (e.g., duplicate SKU on rows 1 040 and 88 211) is checked with a seen-keys structure. For very large files a plain set of short keys is fine (a few hundred MB ceiling is far away for typical key sizes); if keys are long, hash them to 16 bytes first.

---

## 8. Batch Writing & Transaction Strategy

Each validated chunk is written inside its own atomic block:

```python
# infrastructure/repositories.py
class DjangoRecordRepository:
    def write_chunk(self, job: ImportJob, chunk_no: int, records: list[dict]) -> int:
        with transaction.atomic():
            objs = [Product(**r, import_job_id=job.id) for r in records]
            Product.objects.bulk_create(
                objs,
                batch_size=1000,
                update_conflicts=True,                 # idempotent upsert (PG 15+/Django 4.1+)
                unique_fields=["sku"],
                update_fields=["name", "price", "quantity", "category_id"],
            )
            ImportJob.objects.filter(id=job.id).update(
                last_committed_chunk=chunk_no,
                succeeded_rows=F("succeeded_rows") + len(records),
                processed_rows=F("processed_rows") + ...,
            )
        return len(records)
```

Key decisions:

**Chunk-level atomicity, not file-level.** A chunk either fully commits — including the checkpoint update — or fully rolls back. Committing the checkpoint *in the same transaction* as the data is what makes crash recovery exact: after a worker dies, the resumed task skips every chunk ≤ `last_committed_chunk` and re-processes from the next one with no duplicates and no gaps.

**Optional all-or-nothing mode.** Some imports must be atomic across the whole file. For those, rows are written to a *staging table* (or written with `import_job_id` tagging), and a finalize step either promotes them (`INSERT ... SELECT`, or flips a `visible` flag) or deletes them in one statement. This keeps the streaming/chunking machinery identical and pushes the atomicity decision to the end.

**Unexpected DB error inside a chunk** (e.g., a constraint the validator didn't know about): the chunk transaction rolls back, then the writer retries the chunk in *bisect mode* — split in half recursively until the offending rows are isolated; those rows are routed to the error report with the DB error message, and the rest commit. This guarantees one bad row can never poison 1 999 good ones.

`bulk_create` bypasses model `save()` and signals — intentional for throughput; any side effects that matter are applied explicitly in the finalize step.

---

## 9. Celery Task Design — Retries, Idempotency, Fault Tolerance

```python
# interface/tasks.py
@shared_task(
    bind=True,
    acks_late=True,                       # message re-delivered if worker dies
    reject_on_worker_lost=True,
    autoretry_for=(OperationalError, BotoCoreError, SoftTimeLimitExceeded),
    retry_backoff=True, retry_backoff_max=600, retry_jitter=True,
    max_retries=5,
    soft_time_limit=60 * 55, time_limit=60 * 60,
    queue="imports",                      # dedicated queue, dedicated workers
)
def process_import(self, job_id: str):
    job = ImportJob.objects.select_for_update(skip_locked=True)...  # claim guard
    service = build_import_service(job)   # composition root / DI
    service.run(resume_from=job.last_committed_chunk + 1)
```

A single sequential task per file (rather than fan-out per chunk) is the default because it preserves ordering, keeps the seen-keys uniqueness check coherent, and makes checkpointing trivial. The task is fully resumable, so `acks_late` + checkpoints give fault tolerance without parallel complexity. For import types with fully independent rows and extreme volume, the design allows a fan-out variant: an orchestrator splits the file into chunk descriptors, chunk tasks run in parallel, and a Celery *chord* finalizes — at the cost of moving uniqueness checks into the database (`ON CONFLICT`).

Idempotency exists at three levels: the API dedupes by file hash; the task claims the job with `select_for_update(skip_locked=True)` and refuses to run a job already `RUNNING` on a live worker; and writes are conflict-safe upserts keyed on natural keys, so even a duplicated chunk replay is harmless.

Long-running protection: the soft time limit fires before the hard limit, the service catches `SoftTimeLimitExceeded`, commits nothing partial (the current chunk simply rolls back), and the autoretry resumes from the checkpoint.

If retries are exhausted, the task's `on_failure` handler marks the job `FAILED`, records the exception in the audit log, and notifies the user.

---

## 10. Error Report Generation

Failed rows are buffered per chunk and appended to the report incrementally; the report writer keeps at most one chunk of failures in memory.

Layout: the original columns in their original order, followed by an **`Validation Errors`** column. Every invalid cell gets a red fill and a cell comment carrying its specific message; the errors column aggregates all messages for the row (`price: must be ≥ 0.01; category: 'Gadgets' does not exist`).

```python
# infrastructure/error_report.py
RED = PatternFill(start_color="FFC7CE", end_color="FFC7CE", fill_type="solid")
RED_FONT = Font(color="9C0006")

class ErrorReportWriter:
    def __init__(self, schema):
        self.wb = Workbook()
        self.ws = self.wb.active
        self.col_index = {c.name: i + 1 for i, c in enumerate(schema.columns)}
        self._write_header(schema)

    def append(self, row_no: int, raw: dict, errors: list[CellError]):
        r = self.ws.max_row + 1
        for name, idx in self.col_index.items():
            cell = self.ws.cell(row=r, column=idx, value=sanitize(raw.get(name)))
        for err in errors:
            if err.column in self.col_index:
                cell = self.ws.cell(row=r, column=self.col_index[err.column])
                cell.fill, cell.font = RED, RED_FONT
                cell.comment = Comment(err.message, "Import Validator")
        self.ws.cell(row=r, column=len(self.col_index) + 1,
                     value="; ".join(f"{e.column}: {e.message}" for e in errors))
```

Memory note: standard openpyxl keeps the error workbook in memory, which is fine while failures are a minority. The writer monitors its own row count; past a threshold (e.g., 200 000 failed rows) it switches to `xlsxwriter` with `constant_memory=True`, keeping the red highlighting and the errors column but dropping cell comments (which require buffering). In the degenerate "almost everything failed" case the report is also split into multiple files of N rows each — both to bound memory and because a million-row error file is unusable anyway.

`sanitize()` defends against **formula injection**: any value beginning with `=`, `+`, `-`, or `@` is prefixed with `'` so Excel renders it as text. The finished report is uploaded to S3 under `imports/reports/`, linked on the `ImportJob`, and served to the user via a short-lived presigned GET URL.

---

## 11. Progress Tracking & Notifications

Progress is written to Redis after every chunk (`{processed, total, pct, phase}`) because Redis writes are cheap; the `ImportJob` row in Postgres is updated with the same numbers transactionally per chunk (it already is, as part of the checkpoint), so Redis is a fast read cache, not the source of truth.

Clients consume progress via `GET /api/imports/{id}/` (polling) or a WebSocket group `import.{job_id}` through Django Channels, where the worker publishes chunk-completion events. Phases are reported too (`uploading → parsing → validating/writing → generating_report → done`) so the UI can show more than a bare percentage.

On completion the notification component (an application-layer port with email/WebSocket/webhook adapters) sends a summary: totals, duration, and the error-report download link. Notifications are sent from a separate small Celery task so a flaky SMTP server can never fail or retry the import itself.

---

## 12. Logging, Auditing & Observability

Every job produces a structured (JSON) log stream tagged with `job_id`: job started, per-chunk timings (`chunk=42 rows=2000 valid=1991 invalid=9 db_ms=130`), retries, and a terminal summary line mirroring the `ImportJob` fields — total processed, inserted, failed, wall time, rows/sec. The `ImportJob` table itself is the durable audit record and powers an admin dashboard (jobs per day, failure rates, slowest imports). Metrics (Prometheus/StatsD) export counters for rows processed/failed and histograms for chunk latency; alerts fire on jobs stuck in `RUNNING` beyond an SLA or on elevated failure ratios. Sentry captures unhandled exceptions with `job_id` context.

Retention: uploaded files and error reports get S3 lifecycle rules (e.g., delete after 90 days); `ImportRowError` rows are pruned with the same horizon.

---

## 13. Security

File handling: extension whitelist *and* magic-byte sniffing (an `.xlsx` must be a ZIP starting `PK`, an `.xls` an OLE2 container), hard size limits at presign time and again at processing time, optional ClamAV scan as a pre-processing phase, and zip-bomb defense (cap decompressed XML size / row count before parsing). The uploads bucket is private; all access is via presigned URLs with short TTLs. Workers run with least-privilege IAM (read uploads prefix, write reports prefix).

Application: only authenticated, authorized users may create imports for their tenant; `ImportJob` querysets are always tenant-scoped; rate limiting on the presign endpoint; all cell values are treated as untrusted strings until coerced by the schema (never `eval`, never trust formula results — `data_only=True` reads cached values, not formulas); formula-injection sanitization on every value written to the error report; secrets via environment/SSM, never in code.

---

## 14. Performance Summary

Memory stays flat regardless of file size: streaming parser + chunk buffers + bounded error writer. Throughput levers, in order of impact: `bulk_create` with conflict handling (thousands of rows per statement), reference-data caching to eliminate per-row queries, chunk size tuning, a dedicated `imports` Celery queue with `worker_prefetch_multiplier=1` so one giant file can't starve others, and horizontal scaling by adding workers (each job is sequential, but jobs parallelize perfectly). For PostgreSQL, `COPY` via `psycopg` can replace `bulk_create` in the staging-table mode when raw speed dominates. Realistic target: 5–20k rows/sec end-to-end depending on validation complexity.

---

## 15. Testing Strategy

The layering pays off here. Domain validators and schemas are tested as pure functions with table-driven cases (valid row, each constraint violated, multiple simultaneous errors). The parser is tested against generated workbooks (including pathological ones: blank rows, missing headers, wrong sheet, 0-byte file, xls vs xlsx). `ImportService` is tested with in-memory fakes for every port — no DB, no S3, milliseconds per test. Integration tests run the real repository against a test Postgres (checkpoint/resume behavior, bisect-on-DB-error, upsert idempotency: run the same chunk twice, assert no duplicates). Celery is exercised with `task_always_eager` for flow tests plus one non-eager test against a real broker for `acks_late` semantics. The error report writer is verified by re-opening its output with openpyxl and asserting fills, comments, and the errors column. Finally, a property-based test (Hypothesis) generates random rows against the schema and asserts the invariant `valid + invalid == total` and that no invalid row ever reaches the repository fake.

---

## 16. End-to-End Flow Recap

1. Client presigns and uploads to S3; API creates `ImportJob` (deduped by SHA-256) and enqueues the task.
2. Worker claims the job, streams the file, and for each 2 000-row chunk: validate (collect all cell errors, batch FK lookups) → `bulk_create` upsert + checkpoint in one transaction → buffer failures to the error writer → push progress to Redis.
3. On crash or transient failure, Celery redelivers; the task resumes from `last_committed_chunk + 1`. Persistent DB errors inside a chunk are bisected down to the guilty rows.
4. After the last chunk: error report (red cells, comments, error column, sanitized) is uploaded; `ImportJob` is finalized with totals and timing; row errors are bulk-saved for audit; notifications go out with the report link.
5. The user polls or receives a WebSocket event, sees `COMPLETED_WITH_ERRORS (98 712 inserted, 1 288 failed)`, downloads the report, fixes the highlighted cells, and re-uploads only the failed rows — which flows through the exact same pipeline.
