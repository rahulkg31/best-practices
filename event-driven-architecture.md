# Event-Driven Architecture

This document explains *why* to use events, *how* to design them, and the specific naming, schema, error-handling, and routing rules that are mandatory across services.

------

## 1. What problem event processing solves

Most services start out calling each other over REST. That works until:

- A caller doesn't actually need an immediate answer, but still pays the latency cost of one (blocking on 3+ chained HTTP calls).
- One producer's data is needed by several consumers, and hard-coding "who to call" into the producer creates tight coupling — every new consumer requires a code change in the producer.
- A downstream dependency being briefly unavailable causes the whole request to fail, instead of just being retried later.
- Services need to react to *facts* ("a device came online") rather than being told what to do by an API caller.

Event processing addresses these by replacing direct service-to-service calls with **publish to a broker, let interested parties subscribe**. The producer doesn't know or care who's listening.

**This is not a replacement for REST/gRPC.** Both exist side by side. Use this table to decide:

| Scenario                                                     | Recommended pattern                           |
| ------------------------------------------------------------ | --------------------------------------------- |
| "Do this and give me the result" (synchronous RPC / query)   | REST / gRPC                                   |
| Querying or reading raw state / reference data               | REST / gRPC                                   |
| Caller is a browser, mobile app, or external partner         | REST / gRPC (never events)                    |
| Strong ordering / transactional guarantees needed *instantly* | REST / gRPC (harder with events)              |
| "Something happened; whoever cares can react"                | Event-driven / pub-sub                        |
| One producer, many independent consumers                     | Event-driven (REST is awkward here — N calls) |
| Downstream can be briefly unavailable without failing the caller | Event-driven                                  |
| Caller does not require an immediate blocking response       | Event-driven                                  |
| Long-running background processing or pipeline orchestration | Event-driven                                  |

------

## 2. Core concepts and vocabulary

- **Event**: a fact that already happened (`order.created`, `device.status.changed`) — past tense, not a command.
- **Command / Request**: an intent addressed to the one service that owns the handling logic (`payment.validate`) — imperative, not past tense.
- **Response**: the correlated result of a command, tied back via `correlationId`.
- **Producer**: the service that publishes a message.
- **Consumer**: a service that subscribes to and reacts to a message. Independent consumers don't know about each other.
- **Topic** (Kafka) / **Exchange+Queue** (RabbitMQ) / **Topic** (MQTT): the named channel messages are published to.
- **Consumer group**: a set of consumer instances sharing the work of one topic — Kafka spreads partitions across them, giving horizontal scaling for free.
- **Partition key**: the field used to route related messages to the same partition, preserving order for that entity (e.g. `orderId`, `deviceId`).
- **At-least-once delivery**: the default guarantee for Kafka/RabbitMQ — a consumer may see the same message more than once, so consumers must be **idempotent**.
- **Correlation ID**: a shared identifier used to tie a request to its eventual reply, or to tie a chain of related messages together across services.

------

## 3. Message classification: events, commands, and responses

Every message published across the mesh must be classified at the **type level** as one of three kinds. This distinction drives naming, topic placement, and error handling throughout the rest of this document.

### 3.1 Events — past tense (facts)

Represents something that has already happened. Immutable, broadcast to any number of subscribers, no reply expected.

- **Naming:** `<entity>.<pastTenseVerb>` — e.g. `certificate.added`, `order.created`, `order.shipped`, `device.status.changed`
- **Topic:** `<domain>-events` — named by the domain concept, since many unrelated subscribers look it up that way.

**Use when**: the producer doesn't need to know what happens next (this should be your default pattern).

### 3.2 Commands / requests (intent)

Represents an intent to do something, addressed to the single service that owns the handling logic. Published to a dedicated commands topic — **never mixed with an events topic**.

- **Naming:** `<entity>.<imperativeVerb>` — e.g. `certificate.add`, `payment.validate`, `provisioning.register`
- **Topic:** `<handling-service>-commands` — **named after whoever consumes and acts on it, not the sender, and not necessarily the entity's noun.** This is a common mistake: `order.prepare` is sent *by* `order-service` but is handled by `kitchen-service`, so it belongs on `kitchen-commands`, not `order-commands`.

### 3.3 Responses — correlated result

The outcome of a command, correlated back via `correlationId`. Two sub-cases:

- **Direct request/reply results** → published to the dedicated `<domain>-replies` topic (e.g. `payment.validate.result` → `payment-replies`).
- **Business failure outcomes** (a domain rule rejected the command) → published as a first-class fact on the domain's **events** topic — never the replies topic and never a technical DLQ — using the `noun.verb.failed` convention (e.g. `certificate.add.failed` → `certificate-events`).

### Worked example: pizza order end-to-end

```html
                     CUSTOMER PLACES ORDER
                             │
                             ▼
                     ┌───────────────┐
                     │ order-service │
                     └───────┬───────┘
              ┌──────────────┴──────────────┐
     (1) publishes EVENT              (2) publishes COMMAND
     type: order.placed                type: order.prepare
              │                                │
              ▼                                ▼
   topic: order-events                topic: kitchen-commands
   named by DOMAIN                    named by HANDLER
   (fact already happened,            (⚠️ NOT order-commands —
    many possible subscribers)         even though sender + type
              │                         say "order", kitchen-service
      ┌───────┴───────┐                 is the sole consumer)
      ▼               ▼                            │
  [billing]     [analytics]                        ▼
                                          ┌────────────────┐
                                          │ kitchen-service│
                                          └────────┬───────┘
                                                   │ (3) prepares the pizza
                                                   │ (4) publishes EVENT
                                                   │     type: order.prepared
                                                   ▼
                                          topic: order-events ◄── DOMAIN naming
                                                   │
                                  ┌────────────────┼────────────────┐
                                  ▼                ▼                ▼
                          [delivery-service]  [notification]   [analytics]
                                  │
                                  │ (5) picks up, (6) delivers to customer
                                  │ (7) publishes EVENT: order.delivered
                                  ▼
                          topic: order-events ◄── DOMAIN naming again
                                  │
                     ┌────────────┼────────────┐
                     ▼            ▼             ▼
                [billing]   [notification]  [analytics]
```

------

## 4. Choosing a broker

All three options below can do "publish a message, let someone else consume it." The differences are about delivery guarantees, throughput, and what kind of system is on the other end.

|                   | Kafka                                                        | RabbitMQ                                                     | MQTT                                                |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------- |
| Best for          | High-throughput backend event streams, event sourcing, analytics pipelines | Task queues, routing rules, RPC-style messaging              | Lightweight devices, IoT, unreliable networks       |
| Message retention | Retains events for a configured period (replayable)          | Consumed and gone (unless using streams)                     | Not retained by default                             |
| Throughput        | Very high                                                    | Moderate–high                                                | Low–moderate, optimized for small payloads          |
| Ordering          | Per-partition                                                | Per-queue (with caveats)                                     | Not guaranteed                                      |
| Consumer model    | Pull-based, consumer groups                                  | Push-based                                                   | Pub/sub, QoS levels                                 |
| Typical use here  | Service-to-service domain events (orders, provisioning, inventory) | Background job queues, complex routing (e.g. topic exchanges by region) | Device telemetry, sensor data, constrained networks |

------

## 5. Communication patterns

### Notification (fire-and-forget)

Producer publishes a fact, doesn't wait for or expect a reply. This should be your default.

```
order-service -> order.created -> inventory-service reacts independently
```

**Use when**: the producer doesn't need to know what happens next.

### State synchronization (data replication)

A consumer keeps its own local copy of another service's data, updated via events, so it doesn't need to call the owning service synchronously on every read.

```
user-service -> user.updated -> search-service updates its own index
```

**Use when**: a service reads another service's data frequently and can tolerate slight staleness.

### Event choreography (chained reactions)

No central coordinator. Each service reacts to the previous event and emits its own.

```
order.created -> payment-service -> payment.completed -> shipping-service -> shipment.created
```

**Use when**: the workflow is linear and doesn't need rollback/compensation logic. **Trade-off**: harder to see the "whole picture" — you have to trace events across services to understand the flow.

### Orchestrated saga

A central orchestrator directly commands each participant and handles compensating actions (rollback) if a step fails.

**Use when**: the workflow spans multiple services and needs to be undone partway through (e.g. "payment succeeded but shipping failed → refund"). Don't reach for this unless you actually need rollback — it's more moving parts than choreography.

### Fan-out to multiple consumers

One event, several independent consumer groups, each doing its own thing.

```
order.created -> inventory-service (decrement stock)
             -> analytics-service (log for reporting)
             -> fraud-service (run a check)
```

**Use when**: multiple services genuinely need the same fact independently. This is where Kafka's consumer-group model shines over point-to-point alternatives.

### Async request/reply (use sparingly)

Simulates a synchronous call over an async broker using a `correlationId` and a reply topic (see §3.3 and §6). The caller keeps an in-memory map of `correlationId -> pending callback`, publishes a **command**, and a listener resolves the entry when the matching **response** arrives on the domain's `-replies` topic (with a timeout).

**Use when**: there's a genuine synchronous dependency (e.g. real-time fraud check) that can't be redesigned as choreography. **Avoid by default** — it reintroduces the coupling and timeout-handling problems events were meant to remove.

------

## 6. Message schema — keep it lightweight

Don't over-engineer the envelope. A minimal structure covers almost all cases:

```json
{
  "id": "9f1c1e2e-6b7a-4e1a",
  "source": "order-service",
  "type": "order.created",
  "time": "2026-07-27T10:15:30Z",
  "data": {}
}
```

- **id**: unique per message — used for consumer-side dedup.
- **source**: which service published it.
- **type**: what happened (event) or what's being requested (command), in `noun.verb` form — past tense for events, imperative for commands.
- **time**: when it happened, for debugging and ordering context.
- **data**: the actual payload.

Only add fields beyond this (`correlationId`, `causationId`, `replyTo`, `traceId`) once you have a concrete need — e.g. `correlationId`/`replyTo` for the request/reply pattern, or `traceId` once you adopt distributed tracing. For plain fire-and-forget events, drop `correlationId`/`replyTo`/`error` entirely.

**Command:**

```json
{
  "id": "req-001",
  "source": "order-service",
  "type": "payment.validate",
  "time": "2026-07-27T10:15:30Z",
  "correlationId": "corr-8823",
  "replyTo": "payment-replies",
  "data": { "orderId": "order-12345", "amount": 49.98 }
}
```

**Response — success:**

```json
{
  "id": "reply-001",
  "source": "payment-service",
  "type": "payment.validate.result",
  "time": "2026-07-27T10:15:31Z",
  "correlationId": "corr-8823",
  "data": {}
}
```

**Response — business failure** (uniform error block, mandatory across all services):

```json
{
  "id": "e98b-4a21-992a",
  "source": "payment-service",
  "type": "payment.validate.failed",
  "time": "2026-07-27T10:15:31Z",
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "The account balance is insufficient for this charge."
  },
  "data": {}
}
```

This failure event is published to `payment-events` (the domain's events topic), **not** `payment-replies` and never a DLQ — see §8.2.

------

## 7. Topic naming, partitioning, and lifecycle

1. **Naming pattern**
   - Events: `<domain>-events` (e.g. `order-events`, `payment-events`, `provisioning-events`, `device-status-events`). Filter by the `type` field on the consumer side rather than exploding topic count per event type.
   - Commands: `<handling-service>-commands` — matches `<domain>-commands` when the handler owns the domain (e.g. `payment-commands`), but must reflect the true handler when it diverges (e.g. `onem2m-commands` for provisioning commands actually handled by `onem2m-service`).
   - Replies: `<domain>-replies`. A domain only needs a `-commands` or `-replies` topic if it actually receives commands or issues direct correlated replies — don't create one speculatively.
2. **Partitioning strategy**: every message published to a topic — command, event, or reply — must include a **partition key** corresponding to its aggregate root (e.g. `orderId`, `deviceId`, `userId`, `certificateId`). This guarantees all chronologically related state changes for a specific entity land in the same partition, preserving ordered processing.
3. **Topic lifecycle configuration**:
   - **Retention**: minimum **7 days** for events topics (longer if regulatory compliance requires it), to allow consumer replay during incident recovery or DR drills. Commands topics may use shorter retention, since replaying a stale command risks re-triggering side effects.
   - **Cleanup policy**: default to `delete` (time/size-based expiration). Reserve `compact` strictly for state-synchronization or lookup-table topics.

------

## 8. Error handling

### 8.1 Two mutually exclusive categories

|          | Technical errors                                             | Business errors                                              |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Examples | DB timeouts, network issues, unhandled exceptions, transient broker drops | `insufficient_funds`, `inventory_out_of_stock`, `account_suspended`, `credit_limit_exceeded` |
| Nature   | System intended to process successfully; environment/infra prevented it | Payload is structurally sound; domain rules reject the transaction |
| Handling | Retry → DLQ if exhausted (§8.2)                              | Emit a `noun.verb.failed` event to the domain's events topic (§8.3) — never a DLQ |

Getting this split wrong is the most common mistake: **routing a business rejection to a technical DLQ hides it from anyone who should react to it as a fact** (e.g. billing, analytics, the customer-facing UI).

### 8.2 Technical error pipeline & DLQ governance

1. **Local retry with exponential backoff**: attempt processing 3–5 times, with randomized jitter to avoid thundering-herd against recovering downstream dependencies.
2. **DLQ routing**: if retries are exhausted, divert the raw, unhandled message to a dedicated dead-letter topic named `<source-topic>.dlq` (e.g. `order-events.dlq`, `payment-commands.dlq`). Both commands and events topics get their own DLQ, since either kind can fail for technical reasons.
3. **DLQ governance**:
   - **Zero-silence policy**: every DLQ must have automated alerting (Datadog, Prometheus, PagerDuty) that triggers immediately when lag or inflow exceeds `0`.
   - **No auto-retries from a DLQ**: messages there require manual intervention, data patching, or admin tooling *after* the root cause is fixed.

### 8.3 Business failure handling

Business failures must never be routed to a technical DLQ. Emit a distinct failure event using the `noun.verb.failed` convention (e.g. `payment.validate.failed`, `order.create.failed`, `user.register.failed`) into the domain's **events** topic — never the commands topic that originated the intent. Use the uniform error schema block from §6.

### 8.4 Idempotency

Because brokers deliver at-least-once, every consumer must be idempotent: track processed `event.id` values (a `processed_events` table, or a natural check like "only update if the status actually differs") before acting on a message a second time.

------

## 9. Handling bulk operations and batches

When an operation generates multiple distinct entities in a batch (e.g. generating 100 certificates, importing 500 products, syncing 1,000 inventory items), **do not create batch-level topics or monolithic batch event arrays**. Emit individual, granular events for each entity instead — this keeps partition-key routing, replay, and DLQ handling consistent with everything else in this document.

```
Generate 100 Certificates
         │
         ▼
Certificate Service
         │
         ├── certificate.added → cert-001 (Partition Key: cert-001)
         ├── certificate.added → cert-002 (Partition Key: cert-002)
         ├── certificate.added → cert-003 (Partition Key: cert-003)
         ├── ...
         └── certificate.added → cert-100 (Partition Key: cert-100)
```

------

## 10. Consumer routing: the registry pattern

As a domain topic grows to carry dozens of distinct event types, monolithic `switch`/`case` or `if/else` routing blocks become a maintenance bottleneck (deep indentation, merge conflicts) and are prohibited. Use a centralized registry map to dispatch incoming events dynamically instead:

```java
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.logging.Logger;

public class EventDispatcher {
    private static final Logger LOGGER = Logger.getLogger(EventDispatcher.class.getName());

    // Registry mapping event type strings to their respective handler functions
    private static final Map<String, Function<Map<String, Object>, CompletableFuture<Void>>> handlerRegistry = Map.of(
        "payment.validated", PaymentHandlers::handlePaymentValidated,
        "payment.validation.failed", PaymentHandlers::handlePaymentFailed,
        "payment.refund.processed", PaymentHandlers::handlePaymentRefunded,
        "fraud.check.passed", FraudHandlers::handleFraudChecked
    );

    /**
     * Dispatches an event to its registered asynchronous handler.
     */
    public static CompletableFuture<Void> dispatchEvent(Map<String, Object> event) {
        String type = (String) event.get("type");
        var handler = handlerRegistry.get(type);

        if (handler == null) {
            LOGGER.warning(String.format("[Warning] No handler registered for event type: %s", type));
            return CompletableFuture.completedFuture(null); // Safe skip
        }

        return handler.apply(event);
    }
}
```

------

## 11. Reliability checklist

- **Idempotency**: track processed `event.id`s before acting (§8.4) — duplicates will happen.
- **Correct error routing**: technical failures → retry → DLQ; business failures → `.failed` event on the domain's events topic. Never mix the two (§8).
- **DLQ alerting**: zero-silence monitoring on every `*.dlq` topic (§8.2).
- **Partition/routing key**: key every message by the entity ID it belongs to (§7).
- **Granular batch events**: never emit one event for a whole batch (§9).
- **Consumer routing**: use a registry/dispatch map, not nested conditionals, once a topic has more than a handful of event types (§10).
- **Timeouts**: if using async request/reply, always set one — an unanswered request is the easiest way to leak memory.
- **Topic hygiene**: don't create a `-commands` or `-replies` topic speculatively; only add one when a domain actually needs it (§7).

