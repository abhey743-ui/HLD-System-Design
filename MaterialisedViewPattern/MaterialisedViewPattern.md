# Materialized View Pattern

## 1. What It Is

The Materialized View pattern solves a mismatch between how data is **written** and how it
needs to be **read**. Data is stored in a normalized, transactional form (e.g. multiple
related tables in SQL), but querying it that way for every read is slow or wasteful —
especially when reads vastly outnumber writes, or when the data actually lives across
multiple services.

Instead of computing the read shape on demand every single time, you **precompute it once**
and store it as a ready-to-read, denormalized copy. Reads then go straight to that
precomputed copy — no joins, no aggregation, no cross-service calls.

It's essentially: *write normalized, read denormalized* — with a background process keeping
the denormalized copy up to date as the source data changes.

---

## 2. The Components Involved

| Component | Role |
|---|---|
| **Source of truth store** | Where writes actually happen — the correct, normalized data (e.g. a relational database). |
| **Event/message broker** | Carries the "something changed" notification from the source of truth to whatever builds the view (e.g. RabbitMQ, Kafka). |
| **Transactional outbox** (recommended, not mandatory) | Guarantees the DB write and the "event published" step happen together atomically — avoids losing events or publishing events for writes that got rolled back. |
| **View builder / consumer** | Listens for change events and updates the materialized view accordingly. This is the piece containing the actual "how do I transform this event into the view shape" logic. |
| **Materialized view store** | The denormalized, read-optimized copy — can be a different database entirely (e.g. MongoDB), a cache, a search index, or even a table in the same SQL database. |
| **Read API** | Whatever serves reads — queries only the materialized view, never the source of truth. |

You don't need every piece listed above in every implementation — the outbox, for example,
is a best practice to avoid inconsistency, not a hard requirement of the pattern itself.

---

## 3. Benefits

- **Fast reads** — no joins, no aggregation, no computation at read time; the answer is
  already sitting there.
- **Decouples read and write workloads** — heavy read traffic doesn't compete with
  write/transactional load on the source database.
- **Freedom to use a different storage technology for reads** — the view can live in
  whatever database best serves the read pattern (document store, search index, cache),
  independent of what the source of truth uses.
- **Service decoupling** — a read service doesn't need to know another service's internal
  schema or call it synchronously; it just reads its own local, pre-built view.
- **Scales reads independently** — the view store can be scaled/replicated on its own
  without touching the write path.

---

## 4. Disadvantages / Trade-offs

- **Eventual consistency** — the view lags behind the source of truth by however long it
  takes the event to travel and be processed. Not suitable where reads must reflect writes
  instantly.
- **Extra infrastructure** — a broker, a consumer/view-builder process, and a second data
  store all need to be built, deployed, and monitored.
- **Duplicated data** — the same information now exists in two places, which means more
  storage and more to keep in sync.
- **Consumer must handle at-least-once delivery** — brokers can redeliver messages, so the
  view-building logic must be idempotent, or the view can end up incorrect or duplicated.
- **Ordering issues** — events can arrive out of order; the view builder has to be written
  defensively so this doesn't corrupt the view.
- **Rebuild complexity** — if the view's shape changes, or the view gets corrupted, you
  need a reliable way to rebuild it from scratch (replay events or re-derive from the
  source of truth).

---

## 5. Where It's Used

- **E-commerce**: order summary / dashboard views that combine data from orders, inventory,
  payments, and shipping into one fast-to-read document, instead of joining across services
  on every page load.
- **CQRS-based systems**: this pattern is essentially the "read model" half of CQRS —
  separating the write model (normalized, transactional) from the read model (denormalized,
  optimized for queries).
- **Reporting / analytics dashboards**: precomputed aggregates or rollups so dashboards
  don't run expensive queries against live transactional data.
- **Search functionality**: syncing transactional data into a search index (e.g.
  Elasticsearch) so full-text search doesn't hit the primary database.
- **Microservice architectures**: a service maintaining its own local, denormalized copy of
  data owned by other services, populated via events, so it can serve reads without
  synchronous cross-service calls.
- **Caching layers with complex derived data**: when what you want to cache isn't just a
  raw row but a computed/combined result, and it needs to survive longer or be queried
  differently than a simple cache entry allows.

---

## 6. In Your Stack (SQL + MongoDB + RabbitMQ)

Mapped onto the components above:

- **Source of truth** → your SQL database (normalized, transactional writes).
- **Event/message broker** → RabbitMQ (carries change notifications).
- **View builder** → a consumer service listening on RabbitMQ, translating events into
  document updates.
- **Materialized view store** → MongoDB (denormalized documents, one per read-shape you
  need, fetched directly with no joins).

This is a very standard combination for this pattern — relational DB for correctness and
transactions, a document store for fast, flexible, pre-shaped reads, connected by a broker
so the two stay loosely coupled.
