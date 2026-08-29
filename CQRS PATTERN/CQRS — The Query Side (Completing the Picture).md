# CQRS — The Query Side (Completing the Picture)

> This file fills the gap flagged in File 2: your code shows the full **write path** (Command → Outbox → Debezium → Mongo sync), but not the actual **read path** — the part where a client asks for doctor data and gets it back. This file shows what that looks like, built the "CQRS-correct" way, using your existing `DoctorData` / `DoctorRepositoryMongo` as the foundation.

---

## 1. Why the Query Side Looks So Different From the Command Side

This is the most important thing to internalize: **the query side should look boring compared to the command side.**

Your command side (`DoctorServiceImps`) has:
- Transactions
- Circuit breakers, retries, bulkheads, rate limiters
- Outbox writes
- Business validation potential

Your query side should have **almost none of that**, because:
- It's not changing state, so there's nothing to roll back → no `@Transactional` needed for simple reads
- A failed read can usually just be retried by the client itself → less need for retry/circuit-breaker wrapping (though rate limiting on a public API is still fine)
- There's no business logic to protect → just fetch and return

If your query side ever starts looking as complex as your command side, that's usually a sign business logic has leaked into it where it shouldn't be.

---

## 2. The Query Object

In simple systems, a "query" can just be a method parameter (like an `id` or a search string). You don't always need a dedicated `Query` class — that's more useful in larger systems using a mediator pattern (like MediatR in .NET, or a custom dispatcher in Java).

For your scale right now, plain method parameters are perfectly fine:

```java
public interface DoctorQueryService {
    DoctorData getDoctorById(String id);
    List<DoctorData> searchDoctorsByName(String name);
    List<DoctorData> getAllDoctors();
}
```

If your system grows and you want explicit query objects (useful when queries need pagination, filters, sorting, etc.), it would look like:

```java
public class SearchDoctorsQuery {
    private String name;
    private String role;
    private int page;
    private int size;
    // getters/constructor
}
```

Either approach is valid CQRS — the object itself isn't the important part, the **separation of read logic from write logic** is.

---

## 3. The Query Handler (Read Service)

Here's what a clean query handler looks like, sitting on top of your existing Mongo repository:

```java
package com.Doctors.Service.QueryService;

import com.Doctors.Data.DoctorData;
import com.Doctors.Repository.DoctorRepositoryMongo;
import lombok.AllArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

@Service
@AllArgsConstructor
public class DoctorQueryServiceImpl implements DoctorQueryService {

    private final DoctorRepositoryMongo doctorRepositoryMongo;

    @Override
    public Optional<DoctorData> getDoctorById(String id) {
        return doctorRepositoryMongo.findById(id);
    }

    @Override
    public List<DoctorData> searchDoctorsByName(String name) {
        return doctorRepositoryMongo.findByFirstNameContainingIgnoreCase(name);
    }

    @Override
    public List<DoctorData> getAllDoctors() {
        return doctorRepositoryMongo.findAll();
    }
}
```

Notice what's **absent** compared to `DoctorServiceImps`:
- No `@Transactional`
- No `CircuitBreaker` / `Retry` / `Bulkhead` / `RateLimiter`
- No outbox table
- No `ObjectMapper` serialization

That's intentional — this is what "the read side is simpler" looks like in real code, not just in theory.

**📌 Best practice note:** Return `Optional<DoctorData>` for single-item lookups instead of throwing/returning `null` directly — it forces the controller layer to explicitly handle the "not found" case (e.g., return a 404) instead of risking a `NullPointerException`.

---

## 4. The Controller Layer (Where Command and Query Finally Meet)

Your two services (`DoctorServiceImps` for commands, `DoctorQueryServiceImpl` for queries) can live behind **separate controllers**, or the same controller with clearly separated methods. Both are fine — CQRS is about the service/model separation, not necessarily the HTTP layer.

```java
@RestController
@RequestMapping("/doctors")
@AllArgsConstructor
public class DoctorController {

    private final DoctorService doctorService;          // Command side
    private final DoctorQueryService doctorQueryService; // Query side

    @PostMapping
    public ResponseEntity<Void> createDoctor(@RequestBody DoctorCreateDto dto) {
        doctorService.createDoctor(dto);
        return ResponseEntity.accepted().build(); // 202: write accepted, read model syncs shortly after
    }

    @GetMapping("/{id}")
    public ResponseEntity<DoctorData> getDoctor(@PathVariable String id) {
        return doctorQueryService.getDoctorById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping
    public List<DoctorData> searchDoctors(@RequestParam(required = false) String name) {
        return name != null
                ? doctorQueryService.searchDoctorsByName(name)
                : doctorQueryService.getAllDoctors();
    }
}
```

**📌 Notice `ResponseEntity.accepted()` (HTTP 202) instead of 200/201 on create** — this is a subtle but important CQRS detail covered more in File 4 (Eventual Consistency): since the write and the read model aren't updated in the same instant, returning 202 ("accepted, being processed") is more honest to the client than implying the doctor is immediately queryable everywhere.

---

## 5. Full End-to-End Flow (Now Complete)

```
1. Client → POST /doctors  (Command)
2. DoctorController → DoctorServiceImps.createDoctor()
3. Saves DoctorInfo to PostgreSQL + saves event to Outbox table (same transaction)
4. Debezium captures the outbox row → publishes to message broker
5. createDoctorFunc() consumes the event → MongoServiceImpl.createDoctor()
6. DoctorData saved to MongoDB (Read Model)

   ... (short delay - eventual consistency) ...

7. Client → GET /doctors/{id}  (Query)
8. DoctorController → DoctorQueryServiceImpl.getDoctorById()
9. Reads directly from MongoDB → returns DoctorData
```

Steps 1–6 are your **Command flow** (already built). Steps 7–9 are the **Query flow** (shown in this file). Together, this is a complete CQRS system.

---

## 6. Quick Recap Table

| Layer | Command Side | Query Side |
|---|---|---|
| Entry point | `POST /doctors` | `GET /doctors/{id}`, `GET /doctors?name=` |
| Service | `DoctorServiceImps` | `DoctorQueryServiceImpl` |
| Database | PostgreSQL | MongoDB |
| Resilience wrapping | Yes (CircuitBreaker, Retry, etc.) | Usually no (kept simple) |
| Transactional | Yes | Usually no |
| Returns | `void` / status only | Actual data |

---

*Next file (already lined up): Eventual Consistency & Its Trade-offs — what the short delay between steps 6 and 7 above actually means for your system, and what can go wrong because of it.*
