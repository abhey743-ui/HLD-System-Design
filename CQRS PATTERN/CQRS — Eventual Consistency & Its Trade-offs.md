# CQRS — Eventual Consistency & Its Trade-offs

> This file explains the concept your system runs on whether you named it or not: **eventual consistency**. Your write side (PostgreSQL) and read side (MongoDB) are not updated at the same instant — there's a small gap. This file explains what that gap means, what can go wrong because of it, and how to reason about it. (The Outbox/Debezium *mechanics* that cause this gap are covered in your other repo — here we only talk about the *consequences*.)

---

## 1. What "Eventual Consistency" Actually Means

**Strong consistency** (what you get inside PostgreSQL alone): the moment a write commits, every subsequent read sees it. No gap, no delay.

**Eventual consistency** (what you get across your Postgres → Mongo pipeline): the write commits immediately in PostgreSQL, but the read model (MongoDB) will *eventually* reflect that change — not instantly. There's a window, however small, where:

```
PostgreSQL: doctor exists ✅
MongoDB:    doctor doesn't exist yet ❌ (still processing the event)
```

This isn't a bug — it's the trade-off you accepted by splitting into two databases connected by an async event pipeline (Outbox → Debezium → Consumer). You get scalability and decoupling *in exchange for* this small window of staleness.

---

## 2. Why This Trade-off Is Usually Worth It

Before listing the risks, it's worth being clear on why this is a deliberate, reasonable choice and not a flaw:

- Making the write side wait for the read side to update (to get strong consistency) would mean your `createDoctor()` call now depends on Mongo being available too — reintroducing the exact coupling CQRS was meant to remove.
- Most business cases can tolerate a delay measured in milliseconds-to-seconds. A doctor record doesn't need to be searchable in Mongo within the exact same millisecond it's created in Postgres — "soon after" is fine for almost all real use cases.
- You get to scale, cache, and evolve the read side independently — a much bigger win than the cost of a short delay.

---

## 3. What Can Actually Go Wrong (and Whether Your System Handles It)

### 3.1 Stale Reads (the expected, harmless case)

**What happens:** A client creates a doctor, then immediately calls `GET /doctors/{id}` a few milliseconds later, and gets a 404 because Mongo hasn't synced yet.

**Is this a problem?** Not really — it's the expected behavior of an eventually consistent system. The fix isn't in your backend logic; it's in managing expectations:
- Returning `202 Accepted` on create (as shown in File 3) instead of `200 OK` signals "this is being processed" rather than "this is done and fully available everywhere."
- If your frontend needs the created doctor immediately, it can use the data it already has from the create request/response rather than immediately re-fetching from the read side.

### 3.2 Duplicate Events (Debezium/consumer redelivery)

**What happens:** Message brokers and CDC tools generally guarantee **at-least-once delivery**, not exactly-once. That means `createDoctorFunc()` could receive the *same* event twice (e.g., after a consumer restart or an ack failure).

**What your code currently does:** Looking at `MongoServiceImpl.createDoctor()`, it always does an insert-style `save()`. If the same event arrives twice, you'd end up with **two `DoctorData` documents for the same doctor** in Mongo (since Mongo will generate a new `_id` unless you explicitly set one).

**What to consider fixing:** Make the write **idempotent** — e.g., use a deterministic ID (the same ID as the Postgres record, or a value from the event payload) as the Mongo document's `_id`, and use an upsert instead of a plain insert:

```java
// Instead of: doctorRepository.save(doctorData)   [always inserts new]
// Prefer something like an upsert keyed on a stable business ID,
// e.g., re-using the doctor's Postgres ID as Mongo's _id.
```

This way, processing the same event twice just overwrites the same document instead of creating a duplicate.

### 3.3 Out-of-Order Events

**What happens:** In theory, if a doctor is created and then quickly updated (once you add an `updateDoctor` command later), it's possible — though less likely with a single-partition setup — for the "update" event to be processed *before* the "create" event lands, depending on your broker/partitioning configuration.

**Why this matters for you going forward:** Right now you only have `createDoctor`, so this isn't a live risk yet. But the moment you add update/delete commands flowing through the same outbox-to-Mongo pipeline, ordering becomes something to actively think about (e.g., partitioning events by doctor ID so all events for the same doctor are processed in order).

### 3.4 Partial Failure Mid-Sync

**What happens:** The event is consumed by `createDoctorFunc()`, but the Mongo save fails (network blip, validation error, etc.) *after* the event was already acknowledged.

**What your config currently does:** Looking at your YAML config —
```yaml
acknowledge-mode: AUTO
max-attempt: 1
requeue-rejected: false
republish-to-dql: true
auto-bind-dql: true
```
— `AUTO` acknowledgment means the message is acked as soon as it's received, regardless of whether processing succeeds afterward. Combined with `max-attempt: 1` and `requeue-rejected: false`, a failure inside `createDoctorFunc()` (like a Mongo write failure) would send the message to the **dead-letter-queue** rather than retry it in place — which is reasonable, but it does mean that specific doctor record silently never makes it into Mongo unless something is watching the DLQ.

**Worth considering:** A monitoring/alerting hook on the DLQ (`createDoctorFunc.CreateDoctor-write-operation`) so a failed sync doesn't go unnoticed — otherwise you'd have a doctor "created" from the user's perspective but permanently missing from search/read results.

---

## 4. How to Reason About "How Eventually" Consistent Your System Is

A useful mental framing: ask **"what's my replication lag budget?"** — i.e., how long is it acceptable for Postgres and Mongo to disagree?

- For most CRUD-style apps like doctor registration: **seconds** of lag is completely fine.
- For something like a real-time bidding system or financial ledger: even milliseconds of lag might not be acceptable, and CQRS with async sync might be the wrong tool entirely for that specific read.

The point isn't to eliminate the lag (you can't, without giving up the benefits of separate databases) — it's to make sure every part of your system (API responses, frontend UX, monitoring) is honest about the fact that the lag exists, rather than silently assuming instant consistency.

---

## 5. Quick Recap Table

| Risk | Cause | Your System's Current Handling | Suggested Improvement |
|---|---|---|---|
| Stale read right after create | Async sync delay | Not explicitly handled | Return `202 Accepted`; avoid immediate re-fetch assumptions |
| Duplicate read-side records | At-least-once delivery | Plain `save()` → possible duplicates | Use idempotent upsert with a stable ID |
| Out-of-order events | Multiple events per entity, no ordering guarantee | Not yet relevant (create-only) | Partition by doctor ID once update/delete are added |
| Silent sync failure | `AUTO` ack + `max-attempt: 1` + DLQ | Failed messages go to DLQ | Add monitoring/alerting on the DLQ |

---

## 6. TL;DR

- Eventual consistency = write and read sides update at *different times*, not instantly together — this is the cost of the decoupling CQRS gives you.
- It's a deliberate, reasonable trade-off for almost all business use cases — not a defect.
- The real engineering work in an eventually-consistent CQRS system isn't preventing the delay — it's handling **what happens during** the delay: stale reads, duplicate events, ordering, and silent failures.
- Your system already has good bones (outbox pattern, DLQ config) — the main gaps worth addressing are **idempotency on the Mongo write** and **DLQ monitoring**.

---

*That completes the core CQRS documentation set: Concept → Command/Query Walkthrough → Query Side → Eventual Consistency. Let me know if you'd like a file on Event Sourcing (how it differs from what you have), or anything else.*
