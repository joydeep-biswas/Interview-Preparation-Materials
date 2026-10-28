# 14. Persistence, Caching & Service Architecture: Interview FAQs & Playbooks

A comprehensive, production-grade guide to real-world interview scenarios and engineering solutions covering SQLAlchemy 2.0 async engine mechanics, connection pool sizing, Redis distributed caching patterns, Redlock distributed locking, Celery task worker orchestration, and PostgreSQL deadlock resolution.

---

## Architectural Overview: High-Throughput Persistence & Distributed Caching Topology

![High-Throughput Persistence & Distributed Caching Topology](assets/14_persistence_caching_and_service_architecture_faq/01_high_throughput_persistence_distributed_caching_topology.svg)

*D2 Source: [`01_high_throughput_persistence_distributed_caching_topology.d2`](assets/14_persistence_caching_and_service_architecture_faq/01_high_throughput_persistence_distributed_caching_topology.d2)*

---

## Table of Contents & Question Directory

### 1. Relational Persistence & SQLAlchemy 2.0 Async
- [Q1. **`[Medium]`** SQLAlchemy 2.0 `AsyncSession` Lifecycle, Greenlet Concurrency & Context Managers](#q1-medium-sqlalchemy-20-asyncsession-lifecycle-greenlet-concurrency--context-managers)
- [Q2. **`[Hard]`** Database Connection Pool Sizing, Queue Overflows & Pool Pre-Ping Triage](#q2-hard-database-connection-pool-sizing-queue-overflows--pool-pre-ping-triage)
- [Q7. **`[Hard]`** PostgreSQL Deadlock Detection, Lock Contention & Automatic Retry Decorators](#q7-hard-postgresql-deadlock-detection-lock-contention--automatic-retry-decorators)
- [Q8. **`[Medium]`** High-Performance JSONB Querying, GIN Indexing & Document Storage in PostgreSQL](#q8-medium-high-performance-jsonb-querying-gin-indexing--document-storage-in-postgresql)
- [Q9. **`[Hard]`** Handling Zero-Downtime Database Failovers with Async Client Drivers](#q9-hard-handling-zero-downtime-database-failovers-with-async-client-drivers)
- [Q10. **`[Medium]`** Routing Read/Write Queries to Read Replicas Dynamically in SQLAlchemy 2.0](#q10-medium-routing-readwrite-queries-to-read-replicas-dynamically-in-sqlalchemy-20)

### 2. Distributed Caching & Locking
- [Q3. **`[Easy]`** Cache-Aside vs Write-Through vs Write-Behind Caching Topologies](#q3-easy-cache-aside-vs-write-through-vs-write-behind-caching-topologies)
- [Q4. **`[Hard]`** Cache Stampede & Thundering Herd Defenses: Probabilistic Early Expiration (XFetch) & Mutexes](#q4-hard-cache-stampede--thundering-herd-defenses-probabilistic-early-expiration-xfetch--mutexes)
- [Q5. **`[Hard]`** Distributed Locking with Redis Redlock: Clock Drift, Split-Brain & Fencing Tokens](#q5-hard-distributed-locking-with-redis-redlock-clock-drift-split-brain--fencing-tokens)
- [Q11. **`[Medium]`** Redis Key Eviction Policies (LRU vs LFU) & Memory Fragmentation Triage](#q11-medium-redis-key-eviction-policies-lru-vs-lfu--memory-fragmentation-triage)

### 3. Background Workers & Integration Testing
- [Q6. **`[Medium]`** Celery Distributed Task Queue: Task Serialization, Worker Prefetch & Late Acks](#q6-medium-celery-distributed-task-queue-task-serialization-worker-prefetch--late-acks)
- [Q12. **`[Medium]`** Testing Database Repositories with Testcontainers PostgreSQL & AsyncFixtures](#q12-medium-testing-database-repositories-with-testcontainers-postgresql--asyncfixtures)

---

## 1. Relational Persistence & SQLAlchemy 2.0 Async

### Q1. [Medium] SQLAlchemy 2.0 `AsyncSession` Lifecycle, Greenlet Concurrency & Context Managers
**Difficulty:** [Medium]  
**Topic Covered:** SQLAlchemy 2.0, AsyncSession, Greenlet, AsyncEngine, Unit of Work  
**Question:** How does SQLAlchemy 2.0 implement async database operations over C-based database drivers like `asyncpg`? Why are `AsyncSession` instances not thread-safe or coroutine-safe?

#### The Greenlet Execution Bridge
Database drivers (e.g., `asyncpg`) are asynchronous, but SQLAlchemy's internal Core and ORM layers were historically synchronous. SQLAlchemy 2.0 uses **greenlet** (a micro-thread library) to bridge the gap: it runs synchronous ORM mechanics in a greenlet while switching contexts transparently during I/O operations (`await session.execute(...)`).
- **Session Concurrency Rule:** An `AsyncSession` represents a single database transaction. It must never be shared across concurrent coroutines (`asyncio.gather()`). Each concurrent task must acquire its own isolated session via `async_sessionmaker()`.

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy import select
from my_models import User

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db", pool_size=20, max_overflow=10)
async_session = async_sessionmaker(engine, expire_on_commit=False)

async def perform_transaction():
    async with async_session() as session:
        async with session.begin():
            result = await session.execute(select(User).where(User.id == 1))
            user = result.scalar_one_or_none()
```

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#9-sqlalchemy-20-architecture-core-vs-orm)*

---

### Q2. [Hard] Database Connection Pool Sizing, Queue Overflows & Pool Pre-Ping Triage
**Difficulty:** [Hard]  
**Topic Covered:** Connection Pools, HikariCP/QueuePool, Stale Sockets, Sizing Math  
**Question:** In production under sudden load spikes, your FastAPI service begins throwing `TimeoutError: QueuePool limit of size 20 overflow 10 reached, connection timed out, timeout 30.00`. How do you calculate optimal connection pool sizing, and how does `pool_pre_ping` prevent stale socket crashes?

#### Connection Pool Sizing Formula
$$\text{Pool Size} = (\text{Core Count} \times 2) + \text{Effective Spindle Count}$$
For cloud PostgreSQL instances:
- A PostgreSQL server can handle roughly 100 to 300 active connections before context-switching overhead degrades database throughput.
- If you have 10 scaled Kubernetes API pods, each with `pool_size=20, max_overflow=10`, total possible open connections = $10 \times 30 = 300$, threatening pool exhaustion.
- **The Solution:** Keep application pod pool sizes conservative (`pool_size=5, max_overflow=5`) and introduce **PgBouncer** or **AWS RDS Proxy** in transaction pooling mode.
- **Preventing Stale Sockets (`pool_pre_ping=True`):** Cloud firewalls silently drop idle TCP sockets after 15 minutes of inactivity. When a worker borrows an idle connection, it attempts to write to a closed socket and crashes. `pool_pre_ping=True` issues a lightweight `SELECT 1` heartbeat probe before releasing the connection to the application, transparently reconnecting dropped sockets.

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#8-database-connection-pooling--pgbouncer)*

---

## 2. Distributed Caching & Locking

### Q3. [Easy] Cache-Aside vs Write-Through vs Write-Behind Caching Topologies
**Difficulty:** [Easy]  
**Topic Covered:** Caching Patterns, Cache-Aside, Write-Through, Write-Behind, Invalidation  
**Question:** Compare Cache-Aside, Write-Through, and Write-Behind caching strategies. Which is most suitable for high-concurrency read-heavy data services?

| Pattern | Write Flow | Read Flow | Pros | Cons |
| :--- | :--- | :--- | :--- | :--- |
| **Cache-Aside (Lazy Loading)** | Write directly to DB; invalidate cache key (`DEL key`) | Read from cache; on miss, read DB and populate cache | Resilient to cache crashes; caches only accessed data | Potential cache stampede on miss; dual-hop on miss |
| **Write-Through** | Write to cache; cache immediately writes to DB | Read from cache | Cache is always warm and consistent with DB | Write latency penalty (writes wait for both cache and DB) |
| **Write-Behind (Write-Back)** | Write to cache; cache asynchronously batches writes to DB | Read from cache | Extremely low write latency; absorbs massive write spikes | Data loss risk if cache node crashes before flushing to DB |

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#19-caching-patterns--architectural-topologies)*

---

### Q4. [Hard] Cache Stampede & Thundering Herd Defenses: Probabilistic Early Expiration (XFetch) & Mutexes
**Difficulty:** [Hard]  
**Topic Covered:** Cache Stampede, Thundering Herd, Mutex Lock, XFetch Algorithm  
**Question:** A hot cache key storing an aggregated business metric expires at 12:00:00. At 12:00:01, 5,000 concurrent requests observe a cache miss and all execute the heavy 5-second SQL query simultaneously, overwhelming the database. How do you defend against this Cache Stampede?

#### Defense 1: Distributed Mutex Lock on Miss
Only the first worker to acquire a Redis lock recomputes the key; all other workers wait or receive stale data:
```python
async def get_with_lock(key: str):
    val = await redis.get(key)
    if val is not None:
        return val
    async with redis.lock(f"lock:{key}", timeout=10):
        val = await redis.get(key)
        if val is None:
            val = await expensive_db_query()
            await redis.set(key, val, ex=300)
    return val
```

#### Defense 2: Probabilistic Early Expiration (The XFetch Algorithm)
Instead of waiting for the key to expire, workers probabilistically recompute the key as it nears expiration:
$$-\beta \cdot \delta \cdot \ln(U) > \text{TTL}$$
Where $\delta$ is computation time, $\beta > 0$ is aggressiveness, and $U \sim \text{Uniform}(0, 1)$. As TTL approaches 0, probability of early refresh approaches 1, ensuring exactly one worker refreshes the cache before it ever expires!

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#19-caching-patterns--architectural-topologies)*

---

### Q5. [Hard] Distributed Locking with Redis Redlock: Clock Drift, Split-Brain & Fencing Tokens
**Difficulty:** [Hard]  
**Topic Covered:** Redlock, Martin Kleppmann Critique, Fencing Tokens, Clock Drift  
**Question:** Explain the Redis Redlock algorithm for multi-master clusters. What are the known failure modes regarding OS clock jumps and GC pauses, and how do Fencing Tokens provide safety?

- **Redlock Mechanism:** The client attempts to acquire locks on $N$ independent Redis masters (e.g., $N=5$) sequentially using identical keys and random UUID values with a small timeout. If the client acquires $\ge \lfloor N/2 \rfloor + 1$ locks within the validity time, the lock is granted.
- **The Kleppmann Critique & Clock Drift Risk:**
  1. Process A acquires lock for 10 seconds.
  2. Process A encounters a 15-second Stop-The-World GC pause or network partition.
  3. The lock expires on Redis. Process B acquires the lock.
  4. Process A wakes up from GC pause, believes it still holds the lock, and writes to storage, corrupting data!
- **The Production Fix: Fencing Tokens:** Every lock grant returns a monotonically increasing sequence counter ($1, 2, 3$). Storage sinks (PostgreSQL) reject mutations where `token < last_seen_token`, preventing stale clients from overwriting newer state.

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#22-distributed-locking-with-redis)*

---

### Q6. [Medium] Celery Distributed Task Queue: Task Serialization, Worker Prefetch & Late Acks
**Difficulty:** [Medium]  
**Topic Covered:** Celery, RabbitMQ, Worker Prefetch Multiplier, `acks_late`, Idempotency  
**Question:** In an asynchronous Celery cluster parsing large PDF documents, why does setting `prefetch_multiplier=4` cause worker starvation and latency spikes, and when should you enable `acks_late=True`?

- **Prefetch Multiplier Problem:** By default, Celery prefetches $4 \times \text{concurrency}$ tasks per worker from the queue. If tasks vary in duration (Task A takes 1 second, Task B takes 5 minutes), a single worker might grab four 5-minute tasks and hold them in its local memory buffer, starving other idle workers.
- **Production Setting for Heavy AI Tasks:**
  ```python
  celery_app.conf.update(
      worker_prefetch_multiplier=1, # Fetch only 1 task at a time
      task_acks_late=True,          # Acknowledge task only AFTER successful execution
      task_reject_on_worker_lost=True
  )
  ```

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#23-distributed-task-processing-with-celery)*

---

### Q7. [Hard] PostgreSQL Deadlock Detection, Lock Contention & Automatic Retry Decorators
**Difficulty:** [Hard]  
**Topic Covered:** Deadlocks, Row-Level Locking, Acquisition Ordering, Exponential Backoff  
**Question:** Two concurrent transactions update accounts A and B: Tx1 updates A then B; Tx2 updates B then A. PostgreSQL aborts one transaction with `40P01: deadlock detected`. How do you prevent deadlocks structurally and mitigate them in application code?

1. **Deterministic Global Lock Acquisition Ordering:** Enforce that multiple rows are always updated in strict sorted primary-key order:
   ```python
   # Sort accounts to prevent cross-over deadlocks
   account_ids = sorted([acc_a_id, acc_b_id])
   ```
2. **Automatic Retry with Exponential Jitter:**
   ```python
   from tenacity import retry, stop_after_attempt, wait_random_exponential, retry_if_exception_type
   import asyncpg

   @retry(
       retry=retry_if_exception_type(asyncpg.exceptions.DeadlockDetectedError),
       stop=stop_after_attempt(5),
       wait=wait_random_exponential(multiplier=0.1, max=2.0)
   )
   async def execute_transfer(session, from_id, to_id, amount):
       ...
   ```

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#8-database-connection-pooling--pgbouncer)*

---

### Q8. [Medium] High-Performance JSONB Querying, GIN Indexing & Document Storage in PostgreSQL
**Difficulty:** [Medium]  
**Topic Covered:** JSONB, GIN Indexing, Containment Operator `@>`, Path Queries  
**Question:** When storing dynamic AI model metadata or document chunk payloads in PostgreSQL JSONB, why are standard B-Tree indexes ineffective, and how do Generalized Inverted Indexes (GIN) accelerate JSON path searches?

- **B-Tree Limitations:** A B-Tree index on a JSONB column can only index the entire serialized document as a single scalar. It cannot index internal nested keys.
- **GIN (Generalized Inverted Index):** Breaks down JSONB documents into distinct path-and-value tokens:
  ```sql
  CREATE INDEX idx_chunks_metadata_gin ON document_chunks USING GIN (metadata jsonb_path_ops);
  -- Accelerates queries utilizing the containment operator @>
  SELECT * FROM document_chunks WHERE metadata @> '{"author": "DeepMind", "verified": true}';
  ```

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#9-sqlalchemy-20-architecture-core-vs-orm)*

---

### Q9. [Hard] Handling Zero-Downtime Database Failovers with Async Client Drivers
**Difficulty:** [Hard]  
**Topic Covered:** RDS Multi-AZ, Read-Only Replicas, DNS Caching, Connection Reconnect  
**Question:** During an AWS RDS Multi-AZ failover, the primary database node switches IPs. Application pods throw `CannotConnectError` for several minutes even after the new primary is live. How do you resolve DNS caching and stale connection retention in Python?

- **Root Cause: CPython DNS Caching:** The Python process or OS glibc caches the IP address of the RDS database endpoint. After failover, the endpoint resolves to a new IP, but application connection pools continue trying to connect to the old deceased IP.
- **Resolution:**
  1. Configure low TCP keepalive timeouts (`tcp_keepalives_idle=5`, `tcp_keepalives_interval=2`, `tcp_keepalives_count=3`).
  2. Set connection recycle limits in SQLAlchemy: `pool_recycle=1800` (recycle connections every 30 minutes).
  3. Ensure application DNS resolver obeys TTL (`networkaddress.cache.ttl=5`).

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#8-database-connection-pooling--pgbouncer)*

---

### Q10. [Medium] Routing Read/Write Queries to Read Replicas Dynamically in SQLAlchemy 2.0
**Difficulty:** [Medium]  
**Topic Covered:** CQRS, Read Replicas, Dynamic Engine Routing, Replication Lag  
**Question:** How do you configure SQLAlchemy 2.0 to automatically route `SELECT` queries to a read replica while directing all mutations (`INSERT`, `UPDATE`, `DELETE`) to the primary write database?

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

primary_engine = create_async_engine("postgresql+asyncpg://master_host/db")
replica_engine = create_async_engine("postgresql+asyncpg://replica_host/db")

class RoutingAsyncSession(AsyncSession):
    def get_bind(self, mapper=None, clause=None, **kw):
        if self._is_clean() and clause is not None and clause.is_select:
            return replica_engine.sync_engine
        return primary_engine.sync_engine
```

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#9-sqlalchemy-20-architecture-core-vs-orm)*

---

### Q11. [Medium] Redis Key Eviction Policies (LRU vs LFU) & Memory Fragmentation Triage
**Difficulty:** [Medium]  
**Topic Covered:** Maxmemory Policies, volatile-lru, allkeys-lfu, Memory Fragmentation Ratio  
**Question:** Your Redis cache memory reaches `maxmemory`. What are the differences between `volatile-lru`, `allkeys-lru`, and `allkeys-lfu`? What does a `mem_fragmentation_ratio > 1.5` indicate?

- **Eviction Policies:**
  - `allkeys-lru`: Evicts least-recently used keys out of all keys. Best for general caching.
  - `volatile-lru`: Evicts least-recently used keys *only among keys with an explicit TTL expiration set*. Protects persistent keys.
  - `allkeys-lfu`: Evicts least-frequently used keys. Superior when traffic has stable frequency distributions (hot items remain cached even if temporarily idle).
- **Memory Fragmentation Ratio:** Calculated as `used_memory_rss / used_memory`. A ratio $> 1.5$ indicates the OS has allocated 50% more physical RAM than Redis is actively using due to memory allocation chunk fragmentation (jemalloc). Resolved via `activedefrag yes`.

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#19-caching-patterns--architectural-topologies)*

---

### Q12. [Medium] Testing Database Repositories with Testcontainers PostgreSQL & AsyncFixtures
**Difficulty:** [Medium]  
**Topic Covered:** Testcontainers, Integration Testing, Ephemeral Databases, pytest  
**Question:** Why is mocking database queries with `unittest.mock` an anti-pattern for complex queries, and how do you implement real database integration tests using Testcontainers in Python?

```python
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy.ext.asyncio import create_async_engine

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as postgres:
        yield postgres

@pytest.mark.asyncio
async def test_user_repository(postgres_container):
    db_url = postgres_container.get_connection_url().replace("psycopg2", "asyncpg")
    engine = create_async_engine(db_url)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    # Perform real SQL operations with authentic PostgreSQL dialect guarantees
```

*Deep-dive theoretical reference: See [Module 14: Persistence, Caching & Service Architecture](../14_persistence_caching_and_service_architecture.md#4-transaction-isolation-levels--concurrency-anomalies)*
