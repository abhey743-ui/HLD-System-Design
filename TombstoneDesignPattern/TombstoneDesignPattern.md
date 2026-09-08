# Temporal — Confirming Activity Completion from the Application Microservice

Scenario: a workflow (running in the orchestrator service) calls an activity that publishes a message
to a broker and then suspends (`doNotCompleteOnReturn()`). Some **other microservice** (e.g. `accounts`)
consumes that message, does its actual work, and must tell Temporal "this activity is done."

This doc covers **only that application-side confirmation step**.

---

## 1. Dependency — just one

```xml
<dependency>
    <groupId>io.temporal</groupId>
    <artifactId>temporal-sdk</artifactId>
    <version>1.26.1</version>
</dependency>
```

That's it — `ActivityCompletionClient` lives inside `temporal-sdk`, nothing else is required.

---

## 2. Bean — just one

```java
@Configuration
public class TemporalCompletionConfig {

    @Bean
    public WorkflowServiceStubs workflowServiceStubs() {
        return WorkflowServiceStubs.newServiceStubs(
                WorkflowServiceStubsOptions.newBuilder()
                        .setTarget("localhost:7233") // Temporal server address
                        .build()
        );
    }

    @Bean
    public WorkflowClient workflowClient(WorkflowServiceStubs serviceStubs) {
        return WorkflowClient.newInstance(
                serviceStubs,
                WorkflowClientOptions.newBuilder()
                        .setNamespace("default")
                        .build()
        );
    }

    @Bean
    public ActivityCompletionClient activityCompletionClient(WorkflowClient workflowClient) {
        return workflowClient.newActivityCompletionClient();
    }
}
```

`ActivityCompletionClient` is the **only bean you actually use** in your business code — the other two
(`WorkflowServiceStubs`, `WorkflowClient`) just exist to construct it. If you're using
`temporal-spring-boot-starter`, `WorkflowClient` is auto-configured for you, so you'd only need to add the
`ActivityCompletionClient` bean on top of that.

---

## 3. Method to call — where the confirmation actually happens

In your `accounts` service, wherever you consume the broker message and finish the real work:

```java
@Component
@RequiredArgsConstructor
public class AccountUpdateConsumer {

    private final ActivityCompletionClient activityCompletionClient;
    private final AccountService accountService;

    @Bean
    public Consumer<AccountUpdateMessage> accountUpdateIn() {
        return message -> {
            byte[] taskToken = Base64.getDecoder().decode(message.getTaskToken());

            try {
                AccountUpdateResult result = accountService.applyUpdate(message.getPayload());

                // CONFIRMATION: tell Temporal the activity succeeded, pass the result
                activityCompletionClient.complete(taskToken, result);

            } catch (Exception e) {
                // CONFIRMATION (failure path): tell Temporal it failed - triggers activity retry policy
                activityCompletionClient.completeExceptionally(
                        taskToken,
                        new RuntimeException("Account update failed", e)
                );
            }
        };
    }
}
```

### What you're passing in

- **`taskToken`** — came in on the message itself (the orchestrator service put it there when it published).
  Decode it back to `byte[]` exactly as it was encoded.
- **`result`** — must match the return type declared on the `@ActivityMethod` in the shared activity
  interface (the one the orchestrator's workflow code depends on). If it's declared as
  `AccountUpdateResult applyAccountUpdate(...)`, you must pass an `AccountUpdateResult` here — not a
  `String`, not `null` on success, or the workflow will fail to deserialize it.

---

## 4. Two methods on `ActivityCompletionClient` — that's the whole API surface you need

| Method | When to call it |
|---|---|
| `complete(byte[] taskToken, Object result)` | Work succeeded — result must match the activity's declared return type |
| `completeExceptionally(byte[] taskToken, Exception e)` | Work failed — triggers Temporal's normal activity retry policy, same as a thrown exception inside a synchronous activity |

---

## 5. Summary

- **Dependency:** `io.temporal:temporal-sdk` — nothing else.
- **Bean:** `ActivityCompletionClient` (built via `workflowClient.newActivityCompletionClient()`).
- **What you call:** `activityCompletionClient.complete(taskToken, result)` on success, or
  `activityCompletionClient.completeExceptionally(taskToken, exception)` on failure.
- **What you pass:** the `taskToken` from the incoming message, and a result object matching the shared
  activity interface's declared return type.
