# Tombstone Design Pattern (Temporal Workflows)

## 1. Problem Statement

Whenever a workflow (or any distributed operation) can **create** something and later **delete/undo** it, three problems show up together:

1. **Retries break idempotency.** Temporal retries Activities automatically on failure/timeout. If "create" or "delete" isn't idempotent, a retry can create a duplicate record or fail on a "not found" for something already deleted.
2. **Ordering isn't guaranteed.** A create can still be in-flight (queued, retrying, or just slow) when a compensating delete is triggered. If the delete "wins the race" and runs first, the late-arriving create can **resurrect** something that was supposed to be gone. This isn't Temporal-specific — it happens anywhere a create and a delete for the same logical entity can be in flight concurrently (queues, CDC pipelines, CQRS projections, replicated caches, etc.).
3. **Read models lag writes.** In a CQRS-style setup (write DB + materialized view / read DB), the write side and read side are updated asynchronously. If you roll back (compensate) an entry, the compensating "delete" event can reach the read side **before** the original "create/update" event does. Result: the read model applies create *after* delete, and the deleted record reappears — permanently wrong until something else fixes it.

The Tombstone pattern exists to make "did this get deleted, and can late writes still land?" a question you can answer deterministically, instead of a race.

## 2. What the Tombstone Pattern Is

Instead of relying on the *absence* of a row to mean "deleted," you keep an explicit **tracking/tombstone record** for every entity that goes through a create → (maybe) delete lifecycle. That record has a **status**, not just existence:

- It is written **before** the create side-effect happens.
- Every activity (create, update, delete, projection-apply) **checks and updates this record** instead of just acting on the target entity blindly.
- A "delete" doesn't just remove data — it **marks** the tracking record so that any create-side effect that arrives afterward knows to no-op instead of re-creating/resurrecting the entity.

In other words: a tombstone is a **marker that outlives the delete**, so late or duplicate writes for that entity have something to check against.

## 3. Core Idea — State Machine

The tracking table is really a small state machine per `entityId` / `workFlowId`:

```
PENDING_CREATE → CREATED → PENDING_DELETE → DELETED
        │                        ▲
        └────────────────────────┘
        (compensation while create still in flight)
```

Rules that make this safe under retries and races:

- **Create activity**: only proceeds if status is `PENDING_CREATE` or absent-and-being-inserted-now (guarded by the unique `workFlowId`/`entityId`). If it's already `CREATED`, skip — that's the idempotency guarantee.
- **Delete/compensation activity**: it must **check the current status first**.
  - If status is `CREATED` → proceed with delete, set status to `PENDING_DELETE`, then `DELETED` once confirmed.
  - If status is still `PENDING_CREATE` (create hasn't landed yet) → **don't just no-op**. Flip the row straight to a tombstoned state (e.g. `PENDING_DELETE`) so that when the create activity *does* eventually run/retry, it sees the tombstone and refuses to create — instead it immediately finalizes as `DELETED`.
- **Any operation that mutates the entity** always re-reads the tracking row before it re-checks; the entity table itself is never trusted as the source of truth for "does this still need to exist."

This is the mechanism that fixes the ordering problem: the tombstone is a piece of state that **persists independently of which physical write (create or delete) actually lands first**.

## 4. Table / Entity Design

Your entity is the right shape — a couple of additions make it hold up under concurrency (a status enum instead of a raw string, a version column for optimistic locking, and timestamps for TTL cleanup and debugging):

```java
package com.Appointment.Temporal.TemporalEntities;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import java.time.Instant;

@Entity
@Getter
@Setter
@Table(
    name = "book_appointment_temporal_table",
    uniqueConstraints = @UniqueConstraint(columnNames = {"workFlowId", "entityId"})
)
public class BookAppointmentTemporalTable {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    @Column(nullable = false)
    private Long id;

    @Column(nullable = false)
    private String workFlowId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TombstoneStatus status;   // PENDING_CREATE, CREATED, PENDING_DELETE, DELETED

    @Column(nullable = false)
    private String entityId;

    @Lob
    private String payload;

    @Version
    private Long version;             // optimistic locking — guards concurrent activity retries

    private Instant createdAt;
    private Instant updatedAt;
}

public enum TombstoneStatus {
    PENDING_CREATE,
    CREATED,
    PENDING_DELETE,
    DELETED
}
```

The unique constraint on `(workFlowId, entityId)` is what makes the *insert* itself idempotent — a retried "start" activity that tries to insert the same tracking row again just hits a constraint violation, which you catch and treat as "already registered."

## 5. Workflow / Activity Flow (Temporal)

```java
// Workflow (orchestration) — deterministic, no side effects itself
public void execute(BookAppointmentRequest request) {
    tombstoneActivities.registerPendingCreate(request.getWorkFlowId(), request.getEntityId(), request.getPayload());

    try {
        createActivities.createAppointment(request.getWorkFlowId(), request.getEntityId());
        tombstoneActivities.markCreated(request.getWorkFlowId(), request.getEntityId());

        // ... rest of the saga (write DB, publish event for the materialized view, etc.)

    } catch (ActivityFailure e) {
        // Compensation path
        tombstoneActivities.markForDeletion(request.getWorkFlowId(), request.getEntityId());
        deleteActivities.deleteAppointment(request.getWorkFlowId(), request.getEntityId());
        tombstoneActivities.markDeleted(request.getWorkFlowId(), request.getEntityId());
    }
}
```

```java
// Activity — create side, must check the tombstone before acting
@Override
public void createAppointment(String workFlowId, String entityId) {
    var row = repository.findByWorkFlowIdAndEntityId(workFlowId, entityId);

    if (row.getStatus() == TombstoneStatus.PENDING_DELETE
        || row.getStatus() == TombstoneStatus.DELETED) {
        // A compensation already claimed this entity — do not create/resurrect it.
        return;
    }

    if (row.getStatus() == TombstoneStatus.CREATED) {
        return; // already done — idempotent no-op on retry
    }

    appointmentService.create(entityId, row.getPayload());
}
```

```java
// Activity — compensation/delete side
@Override
public void markForDeletion(String workFlowId, String entityId) {
    var row = repository.findByWorkFlowIdAndEntityId(workFlowId, entityId);
    row.setStatus(TombstoneStatus.PENDING_DELETE); // tombstone written BEFORE the physical delete runs
    row.setUpdatedAt(Instant.now());
    repository.save(row); // @Version protects this against a concurrent create-activity retry
}
```

The important detail: **the tombstone write happens before the physical delete/compensation runs**, and every create-path activity is required to consult it. That ordering is what closes the race window.

## 6. Why This Fixes the Write-DB / Materialized-View Race

Your scenario, concretely:

1. Activity creates/updates an entity in the write DB.
2. A change event is queued to update the materialized view (read side) — but hasn't been applied yet.
3. Something triggers a rollback/compensation. The compensating delete reaches the read side **first** (different queue, different latency, retried consumer, whatever).
4. The original create/update event finally arrives and gets applied **after** the delete — the read model now shows data that should be gone.

With a tombstone in place, the read-side projector doesn't blindly apply events in arrival order — it **checks the tombstone status for that entity before applying a create/update event**:

- Delete/compensation event arrives first → projector applies it, and (this is the key part) also writes/checks the tombstone marker as `DELETED`.
- Create/update event arrives late → projector checks the tombstone, sees `DELETED`, and **discards the event instead of applying it**.

This turns "whoever arrives last wins" (wrong) into "the tombstone is the source of truth, and events are checked against it" (correct) — regardless of network/queue ordering.

## 7. Use Cases

- **CQRS materialized views** — preventing exactly the create-after-delete resurrection race described above.
- **Saga compensation in distributed transactions** (Temporal, Camunda, Step Functions) — making sure a compensation that runs while the forward step is still in-flight doesn't get undone by that forward step landing later.
- **Event-driven / message-queue systems** — consumers that may process events out of order or more than once (at-least-once delivery).
- **Soft deletes / audit trails** — keeping a marker instead of a hard delete so replication, caches, and downstream consumers can converge safely.
- **Cache invalidation** — a cache repopulated by a stale read shouldn't resurrect data that was just invalidated/deleted.
- **Your case: appointment booking cancellation** — booking creates records in multiple places (write DB, materialized view); cancellation must be safe regardless of whether the original creation has fully propagated everywhere yet.

## 8. Best Practices / Things to Call Out in the Doc

- **Idempotency keys** — use `workFlowId` (or a dedicated idempotency key) as the natural key everywhere, not just for the tombstone table.
- **Optimistic locking (`@Version`)** — protects against two activities (a retried create and a compensation) updating the same row concurrently.
- **TTL / cleanup job** — tombstone rows shouldn't live forever; schedule a purge once you're confident no late event can still arrive (based on your max retry/queue-lag window).
- **State machine validation** — reject illegal transitions (e.g., `DELETED → CREATED`) at the repository/service layer, don't just rely on callers being well-behaved.
- **Monitoring** — alert on tombstones stuck in `PENDING_CREATE`/`PENDING_DELETE` past an expected window; that usually means a stuck workflow or a lost activity.
- **Pair with the Outbox pattern** — if events are published to update the materialized view, publish them transactionally with the tombstone status change so the two never drift apart.

## 9. Suggested MD File Outline (Table of Contents)

1. Problem statement — retries, idempotency, ordering, CQRS race condition
2. What is the Tombstone pattern
3. State machine diagram
4. Table/entity design (with code)
5. Workflow + activity code (with code)
6. Why it solves the write/read race condition
7. Use cases
8. Best practices
9. FAQ / gotchas (optional — e.g. "why not just check `if (exists)` before delete?")
