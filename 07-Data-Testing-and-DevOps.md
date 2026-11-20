# SQL, EF Core, testing and DevOps

## SQL and data

### 1. What are primary keys, foreign keys and indexes?

**Answer:** A primary key uniquely identifies a row; a foreign key enforces a relationship to another key when configured. An index is an additional data structure that can speed up reads but consumes storage and adds write cost. Choose indexes based on filters, joins, ordering and actual query plans.

**Example:** `Orders.Id` can be a primary key; `Orders.CustomerId` can reference `Customers.Id`; an index on `Orders.CustomerId` can help a query filtering by customer.

### 2. Clustered versus nonclustered index?

**Answer:** In SQL Server, a clustered index organizes table rows by the clustered key; a table can have one. A nonclustered index is a separate structure containing keys and row locators, optionally including columns; a table can have multiple. Exact behavior depends on the database engine.

**Example:**

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId
ON Orders(CustomerId);
```

This creates a separate lookup structure; evaluate write cost before adding it.

### 3. INNER JOIN versus LEFT JOIN?

**Answer:** INNER JOIN returns matching rows from both sides. LEFT JOIN returns all rows from the left side, with nulls for unmatched right-side columns. Conditions on the right-side table in a `WHERE` clause can filter out unmatched rows, effectively changing the result.

**Example:**

```sql
SELECT c.Id, o.Id
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.Id;
```

Customers without orders are included with a null order ID.

### 4. Write a query for the second highest distinct salary.

**Answer:** In SQL Server:

```sql
WITH Ranked AS (
    SELECT Salary, DENSE_RANK() OVER (ORDER BY Salary DESC) AS SalaryRank
    FROM Employees
    WHERE Salary IS NOT NULL
)
SELECT DISTINCT Salary
FROM Ranked
WHERE SalaryRank = 2;
```

Ties share a rank; if fewer than two distinct non-null salaries exist, the query returns no rows. Clarify whether the question requires an employee, a salary, or a null result.

### 5. What are transactions and ACID?

**Answer:** Atomicity means all transaction changes commit or none do; consistency means constraints and invariants hold when a transaction completes; isolation controls interactions between concurrent transactions; durability means committed data survives the failures the database guarantees against. Isolation level affects anomalies and contention.

**Example:**

```sql
BEGIN TRANSACTION;
UPDATE Accounts SET Balance = Balance - 10 WHERE Id = 1;
UPDATE Accounts SET Balance = Balance + 10 WHERE Id = 2;
COMMIT;
```

This groups both updates. Real transfers also require validation, error handling and an appropriate isolation strategy.

### 6. Optimistic versus pessimistic concurrency?

**Answer:** Optimistic concurrency detects conflicting updates, often with a version column, and rejects or retries them. Pessimistic concurrency holds locks to prevent conflicting work, potentially increasing blocking or deadlocks. Choose based on contention, correctness requirements and transaction duration.

**Example:**

```sql
UPDATE Orders
SET Status = 'Paid'
WHERE Id = 42 AND Version = @expectedVersion;
```

Zero rows are affected if the stored version changed. Ensure the database or application updates the version.

#### Follow-up: Why do we need optimistic concurrency?

**Answer:** Two users may read the same row, then both try to update it. Without a concurrency check, the later update might silently overwrite the earlier one.

**Example:** Both users read version 3. User A updates the row, producing version 4. User B sends an update expecting version 3; the server detects the mismatch and asks B to reload or resolve the conflict.

An API can express the same idea using an ETag and `If-Match`. In EF Core, configure a suitable concurrency token and handle `DbUpdateConcurrencyException`.

### 7. How do you investigate a slow SQL query?

**Answer:** Capture a representative query and parameters; inspect the execution plan, indexes, scans, row estimates, blocking, data volume and resource usage. Fix the dominant cause and benchmark both reads and write costs. Avoid adding indexes blindly; also consider pagination, unnecessary columns and N+1 queries.

**Example:**

```sql
SELECT Id
FROM Orders
WHERE CustomerId = @customerId;
```

Compare its actual execution plan before and after a tested index change; inspect both duration and write impact.

### 8. How do you prevent SQL injection?

**Answer:** Parameterize queries or use ORM parameters; never concatenate untrusted input into SQL. Validate any dynamic identifiers against an allowlist because identifiers usually cannot be parameterized like values. Apply least-privilege database permissions.

**Example:**

```csharp
var user = await db.Users
    .FirstOrDefaultAsync(u => u.Email == email);
```

EF Core normally sends the email value as a SQL parameter instead of concatenating it into the query text.

## EF Core

### 9. What is EF Core and what does a `DbContext` do?

**Answer:** EF Core is an object-relational mapper. `DbContext` represents a unit of work, manages change tracking and database operations. It is normally scoped per request and is not thread-safe. Use asynchronous database methods and avoid sharing a context across parallel operations.

**Example:**

```csharp
db.Orders.Add(order);
await db.SaveChangesAsync(cancellationToken);
```

The context tracks the new order and saves it.

#### Follow-up: What does EF Core change tracking actually do?

**Answer:** The context remembers tracked entities and their changes. When `SaveChangesAsync` runs, EF Core generates database commands for the changes.

```csharp
var order = await db.Orders.FindAsync(orderId);
if (order != null)
{
    order.Status = "Paid";
    await db.SaveChangesAsync();
}
```

The retrieved order is normally tracked. The context sees its new status when saving. The example omits business validation and concurrency handling.

#### Follow-up: Why is sharing one `DbContext` across parallel tasks unsafe?

**Answer:** `DbContext` is not thread-safe and does not support multiple parallel operations on the same instance. Await one operation before starting the next, or use separate contexts and scopes for genuinely independent work.

```csharp
var orders = await db.Orders.ToListAsync();
var customers = await db.Customers.ToListAsync();
```

This example runs the queries sequentially on one context. Separate contexts can run concurrently when appropriate, but consider database capacity and consistency.

### 10. Tracking versus `AsNoTracking()`?

**Answer:** Tracked entities allow EF Core to detect changes for updates; no-tracking queries are often suitable for read-only results and reduce change-tracking overhead. Select only required fields with projections. Measure realistic queries instead of assuming no-tracking is always the bottleneck.

**Example:**

```csharp
var orders = await db.Orders
    .AsNoTracking()
    .Select(o => new { o.Id, o.Status })
    .ToListAsync();
```

This projects a read-only list without tracking entities.

#### Follow-up: If I use `AsNoTracking`, can I still save changes?

**Answer:** The query does not track the returned entities. Changing such an object does not automatically make the context update it. You can use a separate tracked query or explicitly attach an entity with carefully chosen state.

```csharp
var order = await db.Orders
    .AsNoTracking()
    .FirstOrDefaultAsync(o => o.Id == orderId);

// Changing order.Status here is not automatically tracked by db.
```

No-tracking queries are often useful for read-only results. If a projection contains no entity instances, there may be no entities to track regardless.

### 11. What is the N+1 query problem?

**Answer:** One query fetches a set of items and then another query is made for each item, often through lazy loading. Use an appropriate projection, eager loading or explicitly batched queries, and inspect generated SQL. Be aware that loading multiple collections can also cause large joins or duplicated data.

**Example:** Fetching 100 orders and then lazily loading each order's customer can cause 101 queries. Eager loading is one possible remedy:

```csharp
var orders = await db.Orders
    .Include(o => o.Customer)
    .ToListAsync();
```

Inspect the resulting SQL and choose the loading strategy appropriate to the query.

#### Follow-up: What is an N+1 query, and how can I recognize it?

**Answer:** One query loads N parent records, then another query loads related data for each parent. For 100 orders, that can mean 1 query for orders plus up to 100 queries for customers. Look at database logs or traces rather than guessing.

```csharp
var orders = await db.Orders
    .Include(o => o.Customer)
    .ToListAsync();
```

Eager loading or a projection can reduce round trips. Check generated SQL: loading many related collections can create large result sets, so `Include` is not always the best answer.

### 12. How would you handle database migrations?

**Answer:** Review generated migrations and SQL, test them on representative data, back up and plan recovery. For zero-downtime deployments, use compatible expand-migrate-contract steps; deploy code that works during the transition. A schema rollback alone cannot always restore lost or transformed data.

**Example:**

```powershell
dotnet ef migrations script
```

This generates SQL for review. Apply a reviewed migration through your deployment process rather than assuming it is safe.

#### Follow-up: Why can't a rollback always undo a database change?

**Answer:** Deploying older code does not automatically restore data that a migration deleted or transformed. Use compatible, gradual schema changes and a recovery plan.

**Example:** First add a new column without removing the old one; deploy code that can work during the transition; backfill and verify data; remove the old column only after old code is no longer needed.

## Testing

### 13. Unit versus integration versus end-to-end testing?

**Answer:** Unit tests isolate business logic; integration tests verify components working together, such as an API and database; end-to-end tests exercise a user-visible flow through deployed components. Use a balanced test suite: fast focused tests for rules and targeted integration/E2E tests for important boundaries.

**Example:**

```csharp
Assert.Equal(90m, ApplyDiscount(100m, 0.1m));
```

This unit-tests a pricing rule. An integration test can send an HTTP request and verify a row in a test database.

#### Follow-up: If my unit tests pass, why do I need integration tests?

**Answer:** Unit tests can confirm a business rule in isolation, but a fake database or mocked HTTP client may behave differently from the real dependency. Integration tests check important connections between components.

**Example:** A unit test checks an order total. An integration test sends an HTTP request through the API and verifies the order in a test database. For provider-specific SQL behavior, test against the relevant database engine when feasible.

### 14. Mock versus fake versus real dependency?

**Answer:** A mock verifies or controls interactions; a fake is a simpler working implementation; a real dependency exercises production behavior. Use mocks for external boundaries where useful, but avoid asserting internal implementation details. For SQL translation, transactions and migrations, test against the actual database engine when feasible; an in-memory substitute can behave differently.

**Example:** An in-memory `FakeOrderRepository` can test service decisions; a SQL Server integration test is needed to verify SQL Server-specific query and transaction behavior.

### 15. How would you test event-driven processing?

**Answer:** Test successful handling, duplicate delivery, concurrent duplicates, transient failures, permanent failures, lock expiration and redelivery. Verify that database updates and deduplication are atomic, that the outbox is eventually published, and that replay does not repeat external effects. Include a realistic broker/database integration test for critical flows.

**Example:** Deliver the same event ID twice, then assert the database contains one processed-event record and the order's balance changed only once.

### 16. How do you test an API's resilience?

**Answer:** Simulate timeouts, downstream 5xx/429 responses, slow dependencies, cancellation and partial failures. Verify retry limits, backoff, circuit behavior, idempotency, useful error responses and observability. Run controlled load tests to detect thread starvation, exhausted pools and queue buildup.

**Example:** Make a test dependency return `503` twice, then `200`; assert an idempotent request succeeds within the configured retry limit, and also test exhaustion.

## CI/CD and operations

### 17. What belongs in a .NET service pipeline?

**Answer:** Restore, build, run focused unit and integration tests, perform appropriate dependency/security scans, package a versioned artifact or image, and deploy it using least-privilege credentials. Promote the same artifact through environments; verify health and key business metrics after deployment and retain rollback capability.

**Example:**

```powershell
dotnet restore
dotnet build --no-restore
dotnet test --no-build
```

After these steps, scan and publish the versioned image. The test command assumes the test project was built by the preceding build.

#### Follow-up: Why should the same built artifact move through environments?

**Answer:** If the application is rebuilt separately for staging and production, the binaries or dependencies may differ. Promoting the same versioned image or artifact makes it clearer that production received what was tested.

```text
Build and test -> image digest X -> staging -> approval -> production
```

Configuration can still differ by environment, so test configuration, secrets, connectivity and deployment settings separately. A rollback must also account for compatible database changes.

### 18. Blue-green versus canary deployment?

**Answer:** Blue-green switches traffic between two environments and supports quick routing rollback if the old environment remains compatible. Canary sends a small fraction of traffic to the new version and expands after observing results. Both require compatible data changes, monitoring and a tested rollback; routing reversal cannot undo irreversible data changes.

**Example:** Blue-green switches all traffic from blue to green; a canary might route 5% to the new version, observe errors, and gradually increase traffic.

### 19. What is an SLI, SLO and SLA?

**Answer:** An SLI measures service behavior, such as successful request rate or latency. An SLO is an internal target for that indicator over a period. An SLA is a formal agreement with defined consequences for missing commitments. Define measurement windows and exclusions explicitly.

**Example:** SLI: 99.95% successful requests over 30 days; SLO: at least 99.9%. An SLA, if one exists, separately defines the contractual measurement and consequences.

### 20. How would you respond to a production incident?

**Answer:** Establish impact and severity, communicate with stakeholders, stabilize the system or roll back if safe, and use telemetry to identify the cause. Preserve evidence and record decisions. After recovery, verify the user-facing flow, write a blameless review and track preventive actions. Never hide an incident or claim a root cause before evidence supports it.

**Example:** After a deployment, 5xx responses rise -> notify the incident channel -> roll back if safe -> verify order creation -> investigate traces and document follow-up actions.
