# REST APIs and security

## 1. What makes an API RESTful?

**Answer:** REST is an architectural style centered on resources, stateless interactions, a uniform interface, and representations. HTTP methods, status codes, resource URIs, and cache semantics help express the contract. In practice discuss consistency, evolvability, security, and business needs rather than claiming that JSON alone makes an API RESTful.

**Example:**

```http
GET /orders/42

HTTP/1.1 200 OK
Content-Type: application/json

{"id":42,"status":"Pending"}
```

The request identifies a resource; the response contains its representation.

## 2. GET, POST, PUT, PATCH and DELETE?

**Answer:** GET retrieves a representation and is safe and idempotent by HTTP semantics. POST usually creates or invokes processing and is not inherently idempotent. PUT replaces the target representation and is idempotent by intended effect. PATCH applies partial changes and is not necessarily idempotent. DELETE is idempotent by intended effect even if a repeated call returns a different status.

**Example:**

```http
POST   /orders       # Create
GET    /orders/42    # Read
PATCH  /orders/42    # Partially update
DELETE /orders/42    # Delete
```

These are illustrative request lines; the comments are explanations, not HTTP syntax.

## 3. Common status codes?

**Answer:** 200 OK; 201 Created, typically with a `Location` for the created resource; 202 Accepted for asynchronous processing; 204 No Content; 400 malformed or invalid request; 401 missing/invalid authentication; 403 authenticated but forbidden; 404 not found; 409 conflict; 412 failed precondition such as stale ETag; 429 rate limited; 500 unexpected server error; 503 service unavailable.

**Example:**

```http
HTTP/1.1 201 Created
Location: /orders/42
```

A successful creation can return this response. Requesting an unknown order can return `404 Not Found`.

### Follow-up: What is the difference between HTTP 401 and 403?

**Answer:** `401 Unauthorized` generally means the request lacks valid authentication. `403 Forbidden` means the server understands the identity but refuses the action. An application may deliberately return `404` to avoid revealing whether a private resource exists.

**Example:** A missing token may receive `401`. A valid token for user A does not give access to user B's order; the API may return `403` according to its policy.

## 4. How would you prevent duplicate order creation after retries?

**Answer:** Accept an idempotency key scoped to the caller and operation. Persist the key, request fingerprint, processing state, and final result atomically with the business change where possible. Concurrent duplicate requests must not both create the order; use a unique constraint or equivalent coordination. Define behavior for a reused key with a different payload and for in-progress operations, retention, and recovery after a crash.

**Example:**

```http
POST /orders
Idempotency-Key: request-123
Content-Type: application/json

{"productId":42,"quantity":1}
```

If the same caller retries the same request with this key, return the stored result rather than creating another order.

### Follow-up: Why isn't a POST request automatically safe to retry?

**Answer:** The server might have completed the operation even if the response was lost. Sending the same request again could create another order or payment.

```http
POST /orders
Idempotency-Key: order-request-123
Content-Type: application/json

{"productId":42,"quantity":1}
```

An API can store the key, request fingerprint and result with the order. The same caller retrying the same request receives the stored result. A unique constraint or equivalent coordination prevents concurrent requests from both creating an order; a reused key with a different body must be handled explicitly.

## 5. How would you design pagination?

**Answer:** Offset pagination is simple but can become expensive and produce inconsistent pages when records change. Keyset/cursor pagination uses a stable, indexed ordering and a continuation cursor; it performs well for deep pages. Include a deterministic tie-breaker, define sort/filter semantics, limit page size, and decide whether clients need a snapshot or accept changes between pages.

**Example:**

```http
GET /orders?afterId=100&limit=20
```

Return the next 20 orders ordered by a unique increasing ID; the last returned ID becomes the next cursor.

## 6. How do you version an API without breaking clients?

**Answer:** Favor compatible additive changes first; avoid removing fields or changing meaning. For breaking changes, use an explicit strategy such as URL or header versioning, document support and deprecation timelines, observe usage, and provide migration guidance. Treat event contracts separately from API versions.

**Example:**

```http
GET /v1/orders/42
GET /v2/orders/42
```

Keep v1 working while clients migrate to the documented breaking contract in v2.

## 7. Authentication versus authorization?

**Answer:** Authentication verifies identity; authorization decides what that identity may do. Validate access-token signature, issuer, audience, and expiry using appropriate middleware and trusted keys; enforce scopes/roles and resource-level permissions. A valid token alone does not grant access to every object.

**Example:** A verified token identifies user A; an ownership check still rejects user A's request for user B's private order with `403` or a policy-appropriate `404`.

### Follow-up: Why does authentication normally come before authorization?

**Answer:** Authentication establishes who the caller is. Authorization checks whether that caller may perform an action. If authorization runs before identity has been established, it may not have the information it needs.

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Both require the appropriate authentication and authorization services and policies to be configured separately. The API must also check access to the specific resource, not just verify that the user is signed in.

## 8. OAuth 2.0, OpenID Connect and JWT?

**Answer:** OAuth 2.0 delegates authorization; OpenID Connect adds an identity layer on top of OAuth. JWT is a token format, not a complete authorization protocol. A JWT payload is usually encoded, not encrypted. Use short-lived access tokens and avoid placing secrets in them or in logs.

**Example:**

```http
Authorization: Bearer <access-token>
```

The API validates the token. The angle-bracket value is a placeholder, not a real credential.

## 9. How should APIs handle concurrency?

**Answer:** Use optimistic concurrency, for example an ETag with `If-Match` or a database version column. Reject stale updates with a clear precondition/conflict response; the caller can fetch and reconcile. ETags are not a replacement for authorization or transactions.

**Example:**

```http
ETag: "v1"
```

The client sends `If-Match: "v1"` when updating. If the current version is `"v2"`, the API can return `412 Precondition Failed`.

## 10. How do you secure an API?

**Answer:** Use TLS, identity and least-privilege authorization, validate inputs and content size, protect secrets, restrict network exposure, and rate-limit abuse. Configure CORS narrowly for browsers; CORS is not an authentication mechanism. Avoid mass assignment by using request DTOs, and check object-level authorization on every relevant resource.

**Example:** An update DTO containing only `DisplayName` does not expose an internal `IsAdmin` property for clients to set; still authorize the user and validate the value.

## 11. How do you handle long-running requests?

**Answer:** Validate and accept the request, persist a job, return 202 and a status URI, and process asynchronously. Provide status, failure details safe for the caller, retry/idempotency behavior, and an expiration policy. A 202 means accepted, not completed.

**Example:**

```http
HTTP/1.1 202 Accepted
Location: /exports/123
```

The client can later call `GET /exports/123` to check the export's status.

## 12. What is rate limiting, and how should a client respond?

**Answer:** A rate limit bounds calls in a time window; a quota often bounds a larger usage allowance. Respond with 429 and a `Retry-After` header when applicable. Clients should honor it, use exponential backoff with jitter, and avoid retrying non-idempotent operations without safeguards.

**Example:**

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 10
```

The client waits 10 seconds before retrying an otherwise safe request.

## 13. How do you design consistent error responses?

**Answer:** Use a documented error format such as Problem Details, with stable codes, relevant validation errors and a trace ID. Do not leak implementation details. Log the full internal exception with the same trace ID and map known failures to appropriate client or server status codes.

**Example:**

```json
{
  "type": "https://example.com/problems/not-found",
  "title": "Order not found",
  "status": 404,
  "traceId": "abc123"
}
```

The response gives the caller a safe error and a support correlation ID.

## 14. How would you test an API?

**Answer:** Unit-test business rules, integration-test endpoints with real middleware and a suitable test database, and contract-test important consumers. Cover authentication/authorization, validation, status codes, idempotency, concurrency, and failure paths. Add load tests using representative traffic and latency targets.

**Example:**

```csharp
var response = await client.GetAsync("/orders/999");
Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
```

This integration test checks a missing-order response using suitable test data.
