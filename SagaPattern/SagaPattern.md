# Saga Design Pattern

## 1. What Is the Saga Pattern

A Saga is a way to manage a **distributed transaction** — an operation that spans multiple services or databases — without relying on a single ACID transaction across all of them (which distributed systems generally can't offer cheaply, if at all).

Instead of one big atomic commit, a saga breaks the operation into a **sequence of local transactions**, one per service. Each local transaction:

- Commits its own change durably (it's a real, standalone transaction in that service's own database).
- Publishes something (an event, or a workflow signal) that triggers the next step.
- Has a **corresponding compensating transaction** — an operation that semantically undoes it if a later step in the sequence fails.

If every step succeeds, the saga completes normally. If a step fails, the saga runs the compensations for every step that already succeeded, **in reverse order**, to unwind the operation.

The key trade-off to understand: a saga gives you **Atomicity, Consistency, and Durability** (the operation either fully completes or is fully compensated, and each committed step survives failures) but **not Isolation**. Unlike a real ACID transaction, other parts of the system can observe the intermediate, partially-completed state while the saga is still running. Compensations don't restore the exact previous database state either — they logically reverse the business outcome (e.g. "refund the payment," not "roll back to a byte-for-byte previous row").

## 2. Why You Need It

Without sagas, distributed operations tend to fail in one of two ways:

- **Two-phase commit (2PC)** — technically gives you atomicity across services, but requires a coordinator to hold locks on every participant until all of them vote to commit. This doesn't scale, doesn't tolerate participant unavailability well, and is rarely practical across services owned by different teams or even different companies.
- **No coordination at all** — a step fails partway through, and now you have a half-created appointment, a payment that went through with no booking, inventory reserved with no fulfilled order, etc. — permanent inconsistency with no defined way to recover.

Sagas give you a middle ground: real local transactions (so each step is safe and fast on its own), plus an explicit, code-defined way to unwind a partial failure.

## 3. Core Requirements for Any Saga

Regardless of implementation style, every saga needs:

- **A compensating action for every step** that can't be trivially skipped. (Some steps genuinely have no compensation — e.g. "send confirmation email" can't be unsent — those are just accepted as non-reversible, or handled with a follow-up "sorry, ignore that" message.)
- **Idempotent forward actions and idempotent compensations.** Both the forward step and its compensation must be safe to run more than once, because retries (network failures, timeouts, at-least-once delivery) are a given in distributed systems, and a compensation may run even if the step it's undoing never actually completed.
- **Compensations that can handle "the forward step never happened."** If a step fails before its side effect actually took place, the compensation for it may still fire — it needs to be a safe no-op in that case.
- **Timeouts on every step**, so a saga doesn't block indefinitely waiting on a service that will never respond — a timeout should itself trigger compensation.
- **State tracking** — something needs to know which steps have completed so it knows what to compensate on failure. This is the part that differs the most between implementation styles (see below).

## 4. Ways to Implement It

### a) Choreography (event-driven, no central coordinator)

Each service listens for events from the previous service, performs its local transaction, and publishes its own event (success or failure) for the next service to react to. There's no single piece of code that "owns" the whole saga — the sequence emerges from services reacting to each other's events.

- **Pros:** no single point of failure/bottleneck, services stay loosely coupled, easy to add a new participant without touching existing ones.
- **Cons:** the overall flow isn't visible in one place — you have to trace events across services to understand or debug a saga. Compensation logic gets scattered (each service needs to know how to react to failure events from others), and cyclical dependencies between event flows can get hard to reason about as the saga grows.

### b) Orchestration (central coordinator)

A single orchestrator (a dedicated service, or a workflow engine) explicitly calls each step in sequence, tracks progress, and explicitly triggers compensations in reverse order on failure. Participant services don't need to know about the saga at all — they just expose an API/activity for the orchestrator to call.

- **Pros:** the entire flow is defined in one place — much easier to read, test, and debug. State tracking, retries, and timeouts are handled by the orchestrator rather than reimplemented per service.
- **Cons:** the orchestrator becomes a critical piece of infrastructure (though this is mitigated if it's a durable workflow engine rather than a plain service). Participants are indirectly coupled through the orchestrator's knowledge of their APIs.

### c) Workflow-engine-based orchestration (Temporal, Camunda, AWS Step Functions, etc.)

A specialization of orchestration where a durable workflow engine handles the hard parts of saga bookkeeping for you: automatic retries with backoff, durable state that survives process crashes, and built-in ordering guarantees for each workflow execution. You write the saga as fairly ordinary-looking code (a list/stack of compensations, appended as each step succeeds, run in reverse on failure), and the engine handles persistence, retries, and recovery.

- **Pros:** removes most of the manual state-tracking and retry plumbing that hand-rolled orchestration requires; workflow history gives you built-in observability into exactly what happened and when.
- **Cons:** ties you to a specific engine's execution model and determinism constraints; still requires the same idempotency discipline for every activity and compensation as any other saga implementation.

> We've already documented the workflow-engine-based approach for our own use case — see the Temporal implementation notes and code we put together separately. This file is intentionally concept-only; refer to that doc for the actual saga code, activities, and compensation logic used in the Appointment service.

### d) Hybrid approaches

In practice, larger systems sometimes mix the two: choreography for a loosely-coupled group of "fire and forget" side effects (e.g. notifications, analytics events) alongside orchestration for the core steps that genuinely need ordered compensation (payment, inventory, booking). This avoids forcing every single side effect through a central orchestrator while still keeping the steps that matter for consistency easy to reason about.

## 5. Choosing Between Them

| Factor | Favors Choreography | Favors Orchestration |
|---|---|---|
| Number of steps/services involved | Few, simple | Many, or likely to grow |
| Need to see/debug the whole flow easily | Less important | Important |
| Team ownership | Each service team owns its own reactions | One team/one workflow owns the whole saga |
| Existing infra | Mature event bus already in place | Workflow engine (e.g. Temporal) already in place |
| Coupling tolerance | Want services fully decoupled | Fine with participants depending on a shared orchestrator's contract |

## 6. Common Pitfalls

- **Undefined or missing compensations** for a step — the saga has no way to unwind past that point.
- **Non-idempotent compensations** — a compensation retried after a transient failure double-refunds, double-cancels, etc.
- **No timeout on a step** — a saga can block forever waiting on a service that's down, holding resources (locks, reservations) the whole time.
- **Treating a saga like a transaction with isolation** — code elsewhere in the system that assumes an in-progress saga's intermediate state is invisible will see inconsistent data; this has to be designed for explicitly (e.g. status flags, "pending" states surfaced to callers).
- **Losing track of progress** — without durable state tracking (a workflow engine, an event log, or an explicit saga-state table), a crash mid-saga can leave no record of which compensations still need to run.

## 7. Suggested Outline for This File

1. What is the Saga pattern — local transactions + compensations, ACD not ACID
2. Why it's needed — problems with 2PC and uncoordinated failure
3. Core requirements — compensations, idempotency, timeouts, state tracking
4. Implementation styles — choreography, orchestration, workflow-engine-based, hybrid
5. Choosing between them
6. Common pitfalls
7. Pointer to the Temporal-based implementation doc/code for this repo
