# .NET and ASP.NET Core

## 1. What is .NET?

**Answer:** .NET is a development platform with the CLR runtime, base libraries, SDK, and language support including C#. The CLR manages execution, garbage collection, exceptions, and more. ASP.NET Core builds web applications and APIs on .NET. Distinguish modern cross-platform .NET from the older Windows-focused .NET Framework.

**Example:**

```csharp
Console.WriteLine("Hello");
```

Run the project with `dotnet run`. The CLR executes the code and `Console.WriteLine` uses a base library.

## 2. Explain the ASP.NET Core request pipeline.

**Answer:** A request passes through middleware in configured order; each component can handle the request or call the next component. Responses unwind through the pipeline. Exception handling must be placed early enough to observe later failures; authentication normally precedes authorization, with routing and endpoint execution configured appropriately.

**Example:**
```csharp
app.UseExceptionHandler("/error");
app.UseAuthentication();
app.UseAuthorization();
app.MapGet("/hello", () => "Hello");
```
Authentication runs before authorization; configure the corresponding services and error endpoint separately.

### Follow-up: How does middleware know when to run?

**Answer:** Middleware is registered in a pipeline. Each component runs in order on the way in, and can call `next` to continue. After `next` completes, code after it runs on the way back out.

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");
    await next(context);
    Console.WriteLine("After");
});

app.MapGet("/hello", () => "Hello");
```

When `/hello` is requested, the first message is written before the endpoint and the second afterward. A middleware component may short-circuit instead of calling `next`.

## 3. Middleware versus filters?

**Answer:** Middleware works at the HTTP pipeline level and can apply broadly across endpoints. MVC filters run within the MVC action pipeline and are useful for action-specific concerns. Use middleware for cross-cutting HTTP concerns; use filters when MVC-specific context is required.

**Example:**

```csharp
app.Use(async (context, next) =>
{
    await next(context);
});
```

This is middleware. An MVC action decorated with `[Authorize]` uses an authorization filter in the MVC pipeline.

## 4. Explain DI lifetimes.

**Answer:** **Transient:** new instance per resolution. **Scoped:** one instance per scope, typically per HTTP request. **Singleton:** one instance per container lifetime. A singleton must be thread-safe and must not capture a scoped service; create an explicit scope for scoped work in a background service. EF Core `DbContext` is typically scoped and is not thread-safe.

**Example:**

```csharp
services.AddTransient<Formatter>();
services.AddScoped<OrderService>();
services.AddSingleton<AppCache>();
```

A scoped service is reused within the same request scope.

### Follow-up: What do transient, scoped and singleton mean in a real API?

**Answer:** They describe how long the dependency injection container reuses an instance.

```csharp
services.AddTransient<Formatter>();
services.AddScoped<OrderService>();
services.AddSingleton<AppCache>();
```

- **Transient:** each resolution normally creates a new instance.
- **Scoped:** resolutions within the same request scope share an instance; another request normally gets another instance.
- **Singleton:** resolutions share one instance for the container's lifetime.

**Example:** Two requests generally have different scoped `OrderService` instances but share the same singleton `AppCache`. A singleton may be used concurrently, so its mutable state needs appropriate thread safety.

### Follow-up: Why is it a problem when a singleton holds a scoped service?

**Answer:** The singleton outlives a request, so holding a request-scoped object can accidentally keep it past its intended lifetime and share it across requests. For example, an EF Core `DbContext` is normally scoped and is not thread-safe. Scope validation can detect this misconfiguration.

**Safe approach:** If a singleton background worker needs scoped work, create a scope for each unit of work and resolve the scoped service inside it.

```csharp
using var scope = scopeFactory.CreateScope();
var service = scope.ServiceProvider.GetRequiredService<OrderService>();
await service.ProcessAsync(cancellationToken);
```

When the scope ends, the container disposes its scoped services. This fragment belongs inside a method; `scopeFactory` is an injected `IServiceScopeFactory`.

## 5. How does configuration work?

**Answer:** ASP.NET Core combines configuration providers, such as JSON files, environment variables, and command-line arguments; later providers override earlier ones according to registration order. Bind related settings to options classes and validate important configuration at startup. Keep secrets in a managed secret store, not source control.

**Example:**

```csharp
int limit = builder.Configuration.GetValue<int>("Orders:MaxPageSize");
```

With the default providers, an environment variable `Orders__MaxPageSize` can override the JSON value.

## 6. How do you validate API requests?

**Answer:** Use binding and validation rules on request DTOs; add business validation in the appropriate application layer. Return a consistent 400 or 422 policy for invalid input, with field-level details where safe. `[ApiController]` can automatically produce 400 responses for model validation failures.

**Example:**

```csharp
public class CreateOrder
{
    [Required]
    public string? Product { get; set; }
}
```

An `[ApiController]` using standard model validation returns a 400 response when the product is missing.

## 7. What is `ProblemDetails`?

**Answer:** It is a standard structure for machine-readable HTTP error details, including fields such as type, title, status, and detail. Return a stable problem type and correlation identifier; avoid exposing stack traces, internal SQL, or secrets. Use centralized exception handling and distinguish expected errors from unexpected failures.

**Example:**

```csharp
return Results.Problem(
    statusCode: 404,
    title: "Order not found");
```

This produces an HTTP error with a structured problem body.

## 8. How do you use `HttpClient` correctly?

**Answer:** Use `IHttpClientFactory` or another deliberately managed long-lived client strategy to reuse connections and control handler lifetime. Configure timeouts, logging, and appropriate resilience. Dispose request/response messages when appropriate. Retry only transient failures and only when the operation is safe to retry; use bounded retries with jitter and cancellation.

**Example:**

```csharp
services.AddHttpClient("catalog", client =>
{
    client.Timeout = TimeSpan.FromSeconds(5);
});
```

Inject `IHttpClientFactory` and call `CreateClient("catalog")`.

## 9. How would you build a background worker?

**Answer:** Use `BackgroundService` for long-running work hosted with the application, honoring the stopping token. Create scopes to resolve scoped dependencies. For durable or independently scalable work, use a broker-backed worker or another durable scheduling system. Make processing idempotent and monitor failures and backlog.

**Example:**

```csharp
services.AddHostedService<OrderWorker>();
```

`OrderWorker` derives from `BackgroundService`; its `ExecuteAsync` method should pass the stopping token into its work.

## 10. How does structured logging help?

**Answer:** Log named fields such as order ID, event ID, and trace ID rather than concatenating messages. This makes logs searchable and correlatable across services. Never log access tokens or sensitive payloads unnecessarily; control log volume and retention.

**Example:**

```csharp
logger.LogInformation("Order {OrderId} created", orderId);
```

A structured logging provider stores `OrderId` as a named property.

## 11. How would you investigate a slow API?

**Answer:** Check the latency percentile and error rate, correlate traces, and identify whether time is spent in database calls, downstream HTTP, locks, CPU, or thread-pool starvation. Examine query plans and connection pools, reproduce with a representative load, make a measured change, and compare before/after. Do not assume adding `async` fixes CPU-bound work.

**Example:** A trace shows a 1,000 ms request with an 850 ms database query: inspect that query and its execution plan before changing API code.

## 12. What are health checks?

**Answer:** A liveness check indicates whether the process should be restarted; readiness indicates whether it should receive traffic. Keep liveness independent of a temporarily unavailable downstream dependency to avoid restart storms. Restrict detailed health information and design readiness around actual traffic-serving needs.

**Example:**

```csharp
app.MapGet("/health/live", () => Results.Ok());
```

This is a simple liveness endpoint. Implement a separate readiness check for dependencies required to serve traffic.

## 13. What are minimal APIs and controllers?

**Answer:** Both can build HTTP endpoints. Minimal APIs have concise route definitions and are convenient for small services; controllers offer an established organization and MVC features. Choose based on team conventions and requirements, not an assumption that one is universally faster or more scalable.

**Example:**

```csharp
app.MapGet("/orders/{id}", (int id) =>
    Results.Ok(new { id }));
```

This is a minimal API. A controller can express the same route with `[HttpGet("{id}")]`.

## 14. What happens when request cancellation occurs?

**Answer:** The server exposes a request cancellation token that can be passed to database and HTTP calls. Cancellation is cooperative; a canceled client request does not prove a remote operation was rolled back. For critical mutations, combine cancellation with idempotency and a way to query the final status.

**Example:**

```csharp
var orders = await db.Orders.ToListAsync(
    HttpContext.RequestAborted);
```

The request token is passed to EF Core; cancellation may stop the query if the provider supports it.

## Quick self-check

1. Can a singleton safely hold a request-scoped `DbContext`? **No; create a scope for each unit of work.**
2. Does `AsNoTracking()` track changes for automatic saving? **No.**
3. Can an outbox publisher send the same event twice? **Yes; consumers must handle duplicates.**
4. Does Service Bus duplicate detection prevent every redelivery? **No.**
5. Does an AKS pod failing readiness always need a restart? **No; readiness controls traffic, while liveness controls restarts.**
