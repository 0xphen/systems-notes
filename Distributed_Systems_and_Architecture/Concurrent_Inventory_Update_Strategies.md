# 🛒 Concurrency-Safe Inventory Updates (Deep Dive)

## 📘 The Question

> You’re designing a **highly concurrent checkout system**. After a purchase, you must **update inventory levels** without **race conditions**. Which architectural approach should you consider?

**Options**

1. **Distributed locks** across all inventory servers to ensure serial access
2. **Event-driven architecture** with **message queues** to decouple inventory updates
3. **Optimistic concurrency control (OCC) with versioning** (compare-and-swap)
4. **Centralized inventory service** with a **single point of write** access

---

## 🧩 First Principles

* **Race condition**: two workers read `qty=1`, both decrement → oversell.
* **We want**: correctness (no oversell), high throughput, availability, simplicity.
* **Two big families**:

  * *Pessimistic*: **block** others first (locks)
  * *Optimistic*: **allow** concurrency, **verify** before commit (OCC)

---

## ⚙️ Option 1 — Distributed Locks (Pessimistic)

### What it is

A cluster-wide lock per **SKU** (or per key) so that **only one writer** can update that item at a time. Typical backends: **ZooKeeper/etcd**, or **Redis** (e.g., `SET key value NX PX ttl`).

### How it works (essentials)

1. Compute the lock key: `lock:inventory:<sku>`.
2. Acquire lock (with a **lease/TTL**).
3. Read current quantity, perform update, write back.
4. Release lock.
5. Use **fencing tokens** (monotonic counters) to prevent **stale holders** from writing after a timeout/leader failover.

### Pros

* **Strong serialization** per SKU → simple mental model.
* Easy to reason about for **very hot items**.

### Cons & pitfalls

* **Throughput bottleneck**: only one writer per SKU at a time.
* **Operational complexity**: leases, renewal, fencing tokens, clock skew, split-brain.
* **Latency**: queues form behind locks under spikes.
* **Deadlocks** if you lock multiple keys in different orders.

### When to use

* A **tiny set of super-hot SKUs** (flash sales) where contention is extreme and you’re OK prioritizing **simplicity over throughput**.
* You already operate a **battle-tested lock service** and can implement **fencing** correctly.
* You need strict **first-come-first-served** semantics for a specific SKU.

### Minimal Redis example (with fencing)

* Keep a fencing counter in the DB; every successful lock acquisition returns `token = ++counter`.
* Every write includes `token`; the DB **rejects** writes with **older tokens** than the latest observed.

---

## 📬 Option 2 — Event-Driven + Message Queues

### What it is

Checkouts emit **events** (e.g., `OrderPlaced`) into a queue/stream (**SQS, Kafka, RabbitMQ**). Inventory service consumes and updates stock.

### How it works (essentials)

* **At-least-once** delivery is the norm → you must handle **retries** and **duplicates**.
* **Partition by SKU** (Kafka partitions, SQS FIFO with `MessageGroupId = sku`) so **all events for a SKU** are processed in order by **one consumer**.

### Pros

* **Decoupling & buffering**: absorb bursts, smooth load.
* **Backpressure & retries** baked in.
* Natural place for **dead-letter queues**.

### Cons & pitfalls

* **Queues don’t remove races** by themselves; you still need **OCC or locks at the write boundary**.
* Often **eventual consistency** for reads unless you build a **read model**.
* Partitioning strategy matters (hot partitions can still bottleneck).

### When to use

* You want scalable **throughput** and **resilience** (retries, DLQ) and are comfortable with **eventual consistency**.
* Combine with **OCC** (or single consumer per SKU partition) for correctness.
* Ideal for **burstiness** and **spiky traffic**.

### Practical partitioning patterns

* **Kafka**: key by `sku` → all updates for a SKU go to the same partition/consumer.
* **SQS FIFO**: `MessageGroupId = sku` ensures strict order per SKU.

---

## ✅ Option 3 — Optimistic Concurrency Control (OCC) with Versioning (Recommended)

### What it is

Store a **version** (or ETag) with each inventory row/document. Update only if **expected version matches** (a CAS—compare-and-swap). If not, **retry** with fresh state.

### How it works (SQL)

```sql
UPDATE inventory
SET quantity = quantity - :k,
    version  = version + 1
WHERE sku = :sku
  AND version = :expected_version
  AND quantity >= :k;          -- avoid negatives
-- rows_affected == 1 => success; else => conflict -> retry
```

### How it works (NoSQL)

Conditional update where `version == expected AND quantity >= k`. On failure → refresh & retry.

### Pros

* **Prevents lost updates** without global locks.
* **High throughput**: only conflicting writers retry; others fly through.
* Works with **sharding/partitions**; no cross-node coordination.
* Pairs perfectly with **queues** and **idempotency**.

### Cons & pitfalls

* **Retries** under contention (usually cheap, but monitor).
* Must ensure **idempotency** (e.g., unique `order_id` for de-dup).
* Beware of **starvation** if one SKU is extremely hot → mitigate with **exponential backoff** and optional small **per-key queue**.

### When to use

* **General case** for high-concurrency inventory: you want **correctness + scale** with minimal complexity.
* Traffic is mixed, and you prefer no global locking.

### Practical safeguards

* **Bounded retries** + **backoff**.
* **Metrics**: conflict rate, retries, tail latency.
* **Schema**: index `(sku, version)`; keep `quantity >= 0` check.
* **Outbox or transactional messaging** if you publish events after commit.

---

## 🧭 Option 4 — Centralized Single-Writer Service

### What it is

A single logical **writer** (leader) handles **all writes** (or one writer per **shard** of SKUs). Reads can be served from replicas.

### How it works

* **Leader election** (Raft/Paxos) or simply **route all writes** to a designated service.
* For scale, **shard** SKUs (e.g., hash(sku) → shard N), each shard has **one writer**.

### Pros

* Very **simple semantics** (per writer): inherent serialization.
* Easy to enforce business rules at one place.

### Cons & pitfalls

* Without sharding, it becomes a **bottleneck/SPOF**.
* With sharding, you’ve effectively rebuilt **“single-writer per key space”**—similar complexity to partitioned OCC/locks.
* **Failover complexity** (leader change, in-flight ops).

### When to use

* Small/medium scale with strong ops maturity and **clear shard boundaries**.
* You want to centralize **policy logic** (reservations, oversell rules) in one service per shard.

---

## 🧠 Recommended Architecture

> **Use OCC with versioning (CAS) at the datastore** for correctness and scale.
> Pair it with **event-driven queues** for decoupling/backpressure and **idempotency** for safe retries.

**Why this wins**

* It **serializes only conflicts**, not all traffic.
* No cluster-wide locks → **lower latency, higher throughput**.
* Works with relational and NoSQL stores; easy to test and roll out.

---

## 🧪 Concrete Implementation Blueprint

### 1) Data model

```sql
CREATE TABLE inventory (
  sku       TEXT PRIMARY KEY,
  quantity  INT  NOT NULL CHECK (quantity >= 0),
  version   INT  NOT NULL
);
```

### 2) Write path (pseudocode)

```python
def reserve_stock(sku, k, order_id):
    for attempt in range(MAX_RETRIES):
        item = db.get(sku)                       # qty, version
        if item.quantity < k:
            return OutOfStock

        rows = db.update_if(
            sku=sku,
            expected_version=item.version,
            set_qty=item.quantity - k,
            set_version=item.version + 1
        )
        if rows == 1:
            # emit event idempotently (outbox or transactional publisher)
            publish("StockReserved", order_id=order_id, sku=sku, k=k)
            return Success

        # conflict: someone else updated -> backoff and retry
        sleep(jittered_backoff(attempt))

    return ConflictTooBusy  # optional fallback path
```

### 3) Idempotency

* Use `order_id` (or `payment_intent_id`) as a **natural de-dup key**.
* Keep an **idempotency table** recording processed `order_id`s.

### 4) Eventing (optional but recommended)

* **Producer**: place `OrderPlaced` → Queue (SQS/Kafka).
* **Consumer**: `inventory-worker` reads partitioned by `sku` and performs OCC update.
* **DLQ** + alerting on repeated failures.

### 5) Observability

* Metrics: `conflict_rate`, `retry_count`, `rows_affected`, `reservation_latency_p95/p99`.
* Alerts on spikes in retries → indicates hot SKUs; consider short **per-SKU queues** or **backoff tuning**.

---

## 🎯 Quick Decision Guide

| Situation                                                              | Best Fit                             |
| ---------------------------------------------------------------------- | ------------------------------------ |
| Most inventory systems; need scale + correctness                       | **OCC with versioning**              |
| Extreme hot-spot SKUs; accept throughput loss for strict serialization | **Distributed locks** (with fencing) |
| Spiky traffic; need decoupling/backpressure                            | **Event-driven queue** **+ OCC**     |
| Centralized policy per shard; ops maturity for leader/shard mgmt       | **Single-writer per shard**          |

---

## ✅ Final Answer (one-liner)

**Employ Optimistic Concurrency Control with versioning (compare-and-swap) at the datastore, complemented by idempotent operations and—optionally—an event queue for decoupling and retries.** It prevents race conditions while maintaining high throughput and operational simplicity.
