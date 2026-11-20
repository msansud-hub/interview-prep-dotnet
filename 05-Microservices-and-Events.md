# Microservices and event-driven architecture

## 1. What are microservices and when should you avoid them?

**Answer:** Microservices are independently deployable services organized around business capabilities, with well-defined boundaries and ownership. They can support independent scaling and team autonomy but introduce distributed failures, deployment, observability and data-consistency costs. A well-structured modular monolith can be a better starting point for a small team or simple domain.

**Example:** `Order API -> Order database` and `Inventory API -> Inventory database` can be deployed separately; a small application might keep both modules in one deployable application instead.

## 2. How do you choose service boundaries?

**Answer:** Start with business capabilities and bounded contexts, ownership, change frequency and data boundaries. Keep highly cohesive operations together and avoid splitting merely by technical layer. Evaluate coupling, transaction needs and team ownership; refine boundaries as the domain becomes clearer.

**Example:** Place order status in an `Orders` service and stock quantity in an `Inventory` service, rather than making separate services for controllers, repositories and database tables.

## 3. Synchronous REST versus asynchronous messaging?

**Answer:** Use synchronous calls when the caller needs an immediate answer, with timeouts and resilience. Use asynchronous messages for decoupled processing, buffering, and fan-out when eventual consistency is acceptable. Messaging adds operational complexity, duplicates, ordering concerns and delayed feedback; it does not automatically make a system reliable.

**Example:** `GET /products/42` can return a price immediately; publishing `OrderPlaced` can let a notification worker send an email later.

## 4. Event versus command?

**Answer:** A command requests an action, such as `ReserveInventory`, and usually has an intended handler. An event describes a fact that already happened, such as `OrderPlaced`, and may have multiple independent subscribers. Name contracts clearly, define ownership, and avoid putting sensitive data in event payloads unnecessarily.

**Example:**

```csharp
record ReserveInventory(Guid OrderId);   // Command: request an action
record InventoryReserved(Guid OrderId);  // Event: action happened
```

## 5. What is eventual consistency?

**Answer:** Different components can temporarily see different states while updates propagate; under suitable conditions they converge later. Design user experiences around pending states, reconciliation, timeouts, retries, and compensation. Eventual consistency does not guarantee convergence by itself: failure handling and monitoring must make it happen.

**Example:**

```json
{"status":"Pending"}
```

After a payment event is processed, a later request can return:

```json
{"status":"Paid"}
```

## 6. How do you avoid the database/message dual-write problem?

**Answer:** With the transactional outbox, write the business change and an outbox record in the same local database transaction. A separate publisher reliably sends pending records to the broker and marks them sent. A crash after sending but before marking can cause a duplicate, so consumers must be idempotent. Monitor outbox lag and define retention.

**Example:**
```sql
BEGIN TRANSACTION;
INSERT INTO Orders (Id, Status) VALUES (42, 'Pending');
INSERT INTO Outbox (EventId, EventType) VALUES ('event-42', 'OrderCreated');
COMMIT;
```
The worker publishes the committed outbox record later; a real schema also needs a useful event payload and publisher state.

### Follow-up: What is the problem with saving to a database and then publishing an event?

**Answer:** They are two separate operations. The database save may succeed, then the service may crash before it publishes the message. Other services never learn about the change.

```text
1. Save order: succeeds
2. Publish OrderCreated: service crashes
3. Inventory service never receives the event
```

Publishing first is not a complete fix either: the message may be sent even if saving the order fails.

### Follow-up: How does the transactional outbox fix that gap?

**Answer:** Save the business data and an outbox record in the **same local database transaction**. After commit, a worker publishes the outbox record. If the transaction rolls back, neither record remains.

```text
Transaction:
  save order
  save OrderCreated in outbox
  commit both

Later:
  worker publishes the outbox record
```

The worker can crash after publishing but before marking the record as sent, so a duplicate publish is possible. Give the event a stable ID and make consumers idempotent.

## 7. How do you make a consumer idempotent?

**Answer:** Assign a stable event ID. In one local transaction, record the processed event ID in a table with a unique constraint and apply the database change; duplicates do not reapply the business effect. External side effects require their own idempotency mechanism or an outbox. Define event-ID scope and retention. Simply checking before inserting without atomicity is race-prone.

**Example:** In one transaction insert `event-42` into a uniquely indexed `ProcessedEvents` table and update the order. A duplicate-key conflict rolls back that attempt; after confirming it is the same event, acknowledge the duplicate.

### Follow-up: What does idempotent mean for a message consumer?

**Answer:** Processing the same event again must not repeat its intended business effect. The consumer can record the event ID in the same local transaction as the business update, using a unique constraint to prevent concurrent duplicates.

```text
Event ID: event-42

First delivery: record event-42 + update order, then commit.
Second delivery: event-42 already exists; do not update again.
```

Only acknowledge successful processing after the transaction commits. For an external payment or email, the local deduplication table alone is not enough: use an external idempotency key or a suitable outbox design.

## 8. Exactly-once delivery versus exactly-once effect?

**Answer:** A broker's delivery and deduplication guarantees do not by themselves guarantee exactly-once business effects across databases and external systems. A consumer can commit a change and crash before acknowledging, causing redelivery. Combine at-least-once delivery with transactional deduplication and idempotent external operations to achieve the desired effect within clearly stated boundaries.

**Example:** `process event -> commit database -> crash before ACK -> redelivery`; an atomic processed-event record prevents the second delivery from applying the database change twice.

## 9. What is a saga?

**Answer:** A saga coordinates a business process spanning multiple local transactions. Each step commits locally; on failure, compensating actions address prior steps. In orchestration a coordinator directs steps; in choreography services react to events. Compensation is a new business action, not an automatic database rollback; make steps and compensations idempotent and observable.

**Example:** `ReserveInventory -> ChargePayment`; if payment fails, send `ReleaseInventory`. The release is another business operation, not a rollback of the original transaction.

### Follow-up: What is a saga, and why is compensation not a rollback?

**Answer:** A saga coordinates a business process across services that each commit their own local transaction. When a later step fails, the system may send a compensating command to undo or offset an earlier business action.

```text
Reserve inventory -> charge payment
                         |
                    payment fails
                         |
                  release inventory
```

The release is a new transaction and can itself fail or require retry. Make both the original step and the compensation idempotent, and monitor stuck workflows.

## 10. What are retries, circuit breakers, timeouts and bulkheads?

**Answer:** A timeout limits waiting; a retry repeats a transient failure with a bounded policy and jitter; a circuit breaker temporarily stops calls to a failing dependency; a bulkhead isolates resources so one failure does not consume all capacity. Coordinate time budgets and retry counts across layers to avoid retry storms. Do not retry validation errors or unsafe mutations blindly.

**Example:** For an idempotent GET, use a 2-second attempt timeout and at most two retries with jitter; open a circuit after repeated failures and cap concurrent requests. Set a larger but bounded overall deadline.

## 11. How do you handle poison messages?

**Answer:** Classify transient versus permanent failures. Retry transient errors within a bounded policy; put irrecoverable or repeatedly failing messages in a dead-letter queue. Alert, inspect reason and payload safely, fix the root cause, and replay using a controlled idempotent process. Track dead-letter count and age.

**Example:** A message with invalid required fields fails validation repeatedly -> dead-letter it with a reason -> correct the underlying issue -> replay under a controlled procedure.

## 12. How do you handle message ordering?

**Answer:** Decide the scope requiring order, such as per order ID, rather than assuming global ordering. Azure Service Bus sessions can group and process related messages sequentially when properly configured; retries, multiple producers and downstream effects still require careful design. Include entity version/sequence where appropriate, and define handling for duplicates or out-of-order events.

**Example:**

```csharp
var message = new ServiceBusMessage(body)
{
    SessionId = "order-42",
    MessageId = "event-1"
};
```

Messages for this order share a session on a session-enabled entity; consumer sequencing still needs careful handling.

## 13. How do you evolve event schemas?

**Answer:** Prefer backward-compatible additions and make consumers tolerant of unknown fields. Document required fields, types and semantic meaning; use explicit versions for incompatible changes. Test producer and consumer contracts, run old and new consumers during migration when needed, and plan replay of historical messages.

**Example:**

```json
{
  "orderId": 42,
  "currency": "USD"
}
```

An older consumer that ignores unknown fields can continue reading `orderId` after the optional `currency` field is added.

## 14. What is observability in distributed systems?

**Answer:** Combine structured logs, metrics, and distributed traces. Propagate trace context across HTTP and messaging, retain a business correlation ID when useful, and instrument queue lag, failures and latency percentiles. Use dashboards and alerts tied to user-facing service-level objectives, not only CPU usage.

**Example:** Log `OrderId=42` and the current trace ID in both order and payment services, then follow the distributed trace to see which call caused a delay.

## 15. API gateway versus service mesh?

**Answer:** An API gateway manages entry traffic, routing and cross-cutting policies such as authentication and rate limits. A service mesh handles service-to-service traffic concerns such as identity, encryption, and telemetry, depending on the implementation. They solve different problems and neither removes the need for application-level authorization.

**Example:** `Client -> APIM -> Order service` applies gateway policies; `Order service -> Inventory service` may use mesh-provided mutual TLS if a service mesh is deployed.

## 16. How do you deploy changes safely across services?

**Answer:** Keep API and event changes compatible across mixed versions; deploy consumers before producers where an additive change requires it. Use feature flags, canary or blue-green deployments, health checks, and rollback plans. Handle database changes with expand-migrate-contract; rolling back application code may not reverse data changes.

**Example:** Add an optional database column -> deploy code compatible with both schemas -> backfill data -> only later require the new column, after all callers have migrated.
