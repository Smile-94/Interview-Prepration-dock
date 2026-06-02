# Advanced Database Concepts

A reference guide to advanced concepts used in modern database systems, each with an explanation and a concrete example.

---

## 1. WAL (Write-Ahead Logging)

**Explanation:** Before any change is written to the actual data files on disk, the change is first recorded in an append-only log. This guarantees durability and crash recovery — if the system crashes, the database replays the log to restore a consistent state. The rule is simple: *log first, then apply*.

**Why it matters:** Writing sequentially to a log is far faster than random writes to data pages, and it lets the database confirm a commit without immediately flushing every data page.

**Example:**
```
Transaction T1: UPDATE accounts SET balance = 500 WHERE id = 1;

Step 1 (WAL):   Append to log -> [LSN 101] T1: id=1, old=400, new=500
Step 2 (Commit): Append to log -> [LSN 102] T1: COMMIT
Step 3 (Later):  Background process flushes page with balance=500 to disk
```
If the server crashes after Step 2 but before Step 3, on restart the database reads the log, sees T1 committed, and re-applies the change. PostgreSQL, SQLite (in WAL mode), and MySQL/InnoDB (redo log) all use this.

---

## 2. MVCC (Multi-Version Concurrency Control)

**Explanation:** Instead of locking rows for reads, the database keeps multiple versions of a row. Each transaction sees a consistent **snapshot** of the data as it existed when the transaction started. Readers never block writers and writers never block readers.

**Why it matters:** It dramatically improves concurrency. A long-running report can read data without freezing out concurrent updates.

**Example:**
```
Initial:  row id=1, value="A"  (version created by txn 10)

Txn 20 (reader) starts at time T1  -> sees value="A"
Txn 21 (writer) runs:  UPDATE -> creates new version value="B" (txn 21)
Txn 20 reads again -> STILL sees "A" (its snapshot is fixed at T1)
New Txn 22 starts after T1 -> sees "B"
```
Each row version carries metadata (e.g. `xmin`/`xmax` in PostgreSQL) indicating which transactions can see it. Old versions are cleaned up later (see VACUUM below).

---

## 3. ACID Properties

**Explanation:** The four guarantees a transactional database provides:
- **Atomicity** — all operations in a transaction succeed or none do.
- **Consistency** — a transaction moves the database from one valid state to another (constraints hold).
- **Isolation** — concurrent transactions don't interfere with each other's intermediate state.
- **Durability** — once committed, changes survive crashes.

**Example:**
```
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- debit
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- credit
COMMIT;
```
If the second update fails, **Atomicity** rolls back the first one too — money is never lost mid-transfer.

---

## 4. Transaction Isolation Levels

**Explanation:** A spectrum trading consistency for performance, defined by which anomalies they permit:

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible |
| Serializable | Prevented | Prevented | Prevented |

**Example (Non-Repeatable Read):**
```
Txn A: SELECT balance FROM accounts WHERE id=1;  -> 500
Txn B: UPDATE accounts SET balance=600 WHERE id=1; COMMIT;
Txn A: SELECT balance FROM accounts WHERE id=1;  -> 600  (!)
```
Under **Repeatable Read**, Txn A would see 500 both times.

---

## 5. Two-Phase Commit (2PC)

**Explanation:** A protocol to commit a transaction atomically across multiple nodes/databases. A coordinator first asks all participants to **prepare** (vote), then issues a **commit** only if everyone agreed.

**Example:**
```
Phase 1 (Prepare):
  Coordinator -> Node A: "Can you commit?"  -> A: "Yes" (locks resources)
  Coordinator -> Node B: "Can you commit?"  -> B: "Yes"

Phase 2 (Commit):
  Coordinator -> A, B: "Commit!"  -> both apply changes
```
If any node votes "No" in Phase 1, the coordinator tells everyone to abort. Used in distributed transactions (XA transactions).

---

## 6. B-Tree and B+Tree Indexes

**Explanation:** The default index structure in most databases. A balanced tree keeping data sorted, allowing search, insert, and delete in O(log n). In a **B+Tree**, all actual values live in leaf nodes, and leaves are linked together for fast range scans.

**Example:**
```
Searching for key 27:

            [17 | 35]
           /    |    \
     [5|11]  [20|27]  [40|50]   <- leaf nodes (linked: 5..11 -> 17..27 -> 35..50)

Traverse: 27 < 35 -> middle child -> find 27. 3 hops instead of scanning all rows.
```
Range query `WHERE id BETWEEN 20 AND 40` walks the linked leaves efficiently.

---

## 7. LSM Tree (Log-Structured Merge Tree)

**Explanation:** An index structure optimized for high write throughput. Writes go to an in-memory table (**memtable**); when full, it's flushed to disk as an immutable sorted file (**SSTable**). Background **compaction** merges SSTables and discards stale data. Used by Cassandra, RocksDB, LevelDB, ScyllaDB.

**Example:**
```
Writes -> Memtable (in RAM, sorted)
            | (flush when full)
            v
        SSTable_1 (disk, immutable)
        SSTable_2
        SSTable_3
            | (compaction merges them)
            v
        SSTable_merged (deduplicated, sorted)
```
Reads may check the memtable + several SSTables (Bloom filters speed this up). Trades read amplification for excellent write performance.

---

## 8. Bloom Filter

**Explanation:** A probabilistic, space-efficient structure that answers "is this element *possibly* in the set?" It can return false positives but never false negatives. Databases use it to avoid expensive disk lookups for keys that definitely don't exist.

**Example:**
```
Insert "apple": hash to bits [2, 5, 9] -> set those bits to 1
Query "mango":  hash to bits [2, 7, 9] -> bit 7 is 0 -> DEFINITELY NOT present
Query "apple":  hash to bits [2, 5, 9] -> all 1 -> PROBABLY present (check disk)
```
In an LSM tree, a Bloom filter per SSTable lets the DB skip files that can't contain the key.

---

## 9. Sharding (Horizontal Partitioning)

**Explanation:** Splitting a large dataset across multiple machines (shards), each holding a subset of rows. Scales writes and storage beyond a single machine. The **shard key** determines where a row lives.

**Example:**
```
Users table sharded by user_id % 3:

  Shard 0 -> user_id 3, 6, 9, ...
  Shard 1 -> user_id 1, 4, 7, ...
  Shard 2 -> user_id 2, 5, 8, ...

Query for user_id=7  -> routed only to Shard 1
```
Trade-off: cross-shard queries and joins become expensive. Choosing a poor shard key causes "hot shards."

---

## 10. Replication (Leader–Follower)

**Explanation:** Copying data across multiple nodes for availability and read scaling. A **leader** (primary) accepts writes and streams them to **followers** (replicas), which serve reads.

**Example:**
```
        Writes
          |
          v
       [Leader] --replication log--> [Follower 1] (reads)
                              \-----> [Follower 2] (reads)
```
**Synchronous** replication waits for followers to confirm (safer, slower). **Asynchronous** doesn't wait (faster, risks losing recent writes on failover). Replication lag means followers may briefly serve stale data (eventual consistency).

---

## 11. CAP Theorem

**Explanation:** In a distributed system you can guarantee at most two of three: **Consistency** (every read sees the latest write), **Availability** (every request gets a response), **Partition tolerance** (the system works despite network splits). Since partitions are unavoidable in distributed systems, the real choice is **CP** vs **AP**.

**Example:**
```
Network partition splits nodes A and B.

CP choice: B refuses requests it can't confirm -> consistent but unavailable.
AP choice: B serves possibly-stale data -> available but inconsistent.
```
MongoDB/HBase lean CP; Cassandra/DynamoDB lean AP (tunable).

---

## 12. Eventual Consistency

**Explanation:** A weaker consistency model where, if no new updates are made, all replicas will *eventually* converge to the same value. Common in AP systems prioritizing availability.

**Example:**
```
t=0  Write "X=5" to Node A
t=1  Read from Node B -> "X=3" (stale, not yet propagated)
t=2  Replication completes
t=3  Read from Node B -> "X=5" (converged)
```
Acceptable for things like social media like counts; unacceptable for bank balances.

---

## 13. VACUUM / Garbage Collection

**Explanation:** In MVCC systems, deleted and updated rows leave behind dead versions ("dead tuples"). VACUUM reclaims that space and updates statistics. Without it, tables bloat and performance degrades.

**Example:**
```
UPDATE row id=1 ten times -> 10 old versions + 1 live version on disk.
VACUUM -> removes the 10 dead versions no transaction can see anymore.
```
PostgreSQL's autovacuum runs this automatically; tuning it is a common operational task.

---

## 14. Materialized View

**Explanation:** Unlike a regular view (a stored query run on demand), a materialized view stores the *precomputed result* on disk. Reads are fast, but the data must be refreshed to stay current.

**Example:**
```sql
CREATE MATERIALIZED VIEW daily_sales AS
  SELECT date, SUM(amount) AS total
  FROM orders GROUP BY date;

-- Querying daily_sales is instant (no aggregation at read time)
REFRESH MATERIALIZED VIEW daily_sales;  -- recompute when underlying data changes
```
Great for expensive aggregations queried frequently but updated infrequently.

---

## 15. Deadlock Detection

**Explanation:** A deadlock occurs when two transactions each hold a lock the other needs, waiting forever. The database detects cycles in the "waits-for" graph and aborts one transaction (the victim) to break the cycle.

**Example:**
```
Txn A: locks row 1, then wants row 2
Txn B: locks row 2, then wants row 1

A waits for B, B waits for A -> cycle detected.
DB aborts Txn B -> Txn A proceeds; B is retried by the application.
```

---

## 16. Optimistic vs. Pessimistic Locking

**Explanation:**
- **Pessimistic:** Lock the data up front, assuming conflicts are likely. Others wait.
- **Optimistic:** Don't lock; assume conflicts are rare. At commit, check whether the data changed; if so, abort and retry. Often implemented with a version column.

**Example (Optimistic):**
```sql
-- read row with version=7
UPDATE products SET stock = 9, version = 8
WHERE id = 1 AND version = 7;

-- If 0 rows updated, someone else changed it first -> retry.
```

---

## 17. Connection Pooling

**Explanation:** Opening a database connection is expensive. A pool maintains a set of reusable open connections that application threads borrow and return, reducing latency and protecting the database from connection overload.

**Example:**
```
App requests -> [Pool: 10 ready connections] -> Database
  Thread borrows conn #3, runs query, returns conn #3 to pool.
  Pool caps total connections so the DB isn't overwhelmed by 10,000 clients.
```
Tools: PgBouncer, HikariCP.

---

## 18. Query Optimizer & Execution Plan

**Explanation:** The optimizer analyzes a SQL query and chooses the cheapest way to execute it (which index to use, join order, join algorithm) based on table statistics. The chosen strategy is the **execution plan**.

**Example:**
```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Plan A: Seq Scan (read all rows)         cost=1000
-- Plan B: Index Scan on customer_id idx     cost=8   <- chosen
```
Running `ANALYZE` keeps statistics fresh so the optimizer makes good choices.

---

## 19. Idempotency

**Explanation:** An operation is idempotent if performing it multiple times has the same effect as performing it once. Critical for safe retries in distributed systems and APIs.

**Example:**
```
Not idempotent: UPDATE accounts SET balance = balance + 100;  (retry adds 200)
Idempotent:     UPDATE accounts SET balance = 600 WHERE id=1; (retry = same result)

API with idempotency key:
  POST /charge  Idempotency-Key: abc123
  Retried request with same key -> server returns the original result, no double charge.
```

---

## 20. Change Data Capture (CDC)

**Explanation:** A technique to capture row-level changes (inserts/updates/deletes) from a database — typically by reading the WAL/replication log — and stream them to other systems in near real-time.

**Example:**
```
PostgreSQL WAL --> Debezium --> Kafka --> { Search index, Cache, Data warehouse }

INSERT INTO orders ... -> emitted as event:
  { "op": "c", "table": "orders", "after": { "id": 99, "amount": 50 } }
```
Powers real-time analytics, cache invalidation, and microservice data sync without dual writes.

---

## Quick Reference Summary

| Concept | Solves | Used By |
|---|---|---|
| WAL | Durability & crash recovery | PostgreSQL, InnoDB, SQLite |
| MVCC | Read/write concurrency | PostgreSQL, Oracle, MySQL |
| 2PC | Distributed atomic commit | XA transactions |
| B+Tree | Fast lookups & range scans | Most relational DBs |
| LSM Tree | High write throughput | Cassandra, RocksDB |
| Bloom Filter | Skip useless disk reads | LSM-based stores |
| Sharding | Horizontal scale | Citus, MongoDB, Vitess |
| Replication | Availability & read scale | Nearly all DBs |
| CDC | Real-time data streaming | Debezium, Maxwell |
| Materialized View | Fast precomputed reads | PostgreSQL, Oracle |
