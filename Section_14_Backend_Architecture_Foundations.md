# Section 14: Backend Architecture Foundations for .NET Developers

> A practical guide to understanding, designing, and explaining backend systems. The goal is not to memorize architecture labels; it is to make sound decisions and explain their tradeoffs.

## How to Use This Guide

Read Sections 1-5 to build a mental model of a backend request. Use Sections 6-12 when discussing reliability, security, performance, and production operation. Finish with the interview walkthrough and checklist. Examples use ASP.NET Core, but the principles apply to other backend stacks.

## 1. What Backend Architecture Means

Backend architecture is the set of boundaries and decisions that shape how a system handles requests, applies business rules, stores data, communicates with other systems, and changes safely over time.

Architecture is not just a folder structure or a diagram. It includes:

- **Responsibilities:** which component owns each decision or operation.
- **Boundaries:** what one module or service may know about another.
- **Data ownership:** which component is authoritative for each piece of data.
- **Communication:** in-process calls, HTTP, or asynchronous messages.
- **Quality attributes:** reliability, security, latency, scalability, maintainability, and cost.
- **Operational behavior:** deployment, configuration, monitoring, recovery, and support.

A design is good when it meets the system's important requirements with complexity the team can operate. No architecture style is automatically best.

### Functional and non-functional requirements

- **Functional requirement:** what the system does, such as registering a claim or issuing a refund.
- **Non-functional requirement:** how well or under what conditions it must work, such as response time, availability, auditability, or data retention.

Non-functional requirements should be made measurable when possible. “Fast” is vague; “95% of reads complete within 300 ms under expected peak load” is a useful target. Architecture choices should follow actual requirements, not fashionable technology.

## 2. Follow a Request Through ASP.NET Core

A typical request passes through a pipeline and then through application components:

```text
Client
  -> Load balancer / reverse proxy
  -> ASP.NET Core middleware
       exception handling, request logging, authentication, authorization
  -> Endpoint / controller
       HTTP concerns: route, status code, request and response formats
  -> Application use case
       coordinates the operation and enforces workflow rules
  -> Domain model
       business rules and invariants
  -> Infrastructure
       EF Core, SQL Server, cache, message broker, external APIs
  -> Response
```

The exact arrangement varies, but each part should have a clear reason to change:

- Middleware handles cross-cutting HTTP concerns, not feature-specific business rules.
- Controllers or minimal API endpoints translate between HTTP and the application; they should not become the home of complex workflows.
- Application services or handlers coordinate a use case and its dependencies.
- Domain logic protects business rules that must remain true regardless of which endpoint called them.
- Infrastructure implements technical details behind interfaces where that boundary provides useful isolation.

A common .NET implementation registers dependencies in `Program.cs` using dependency injection. The composition root knows the concrete implementations; inner business code should not construct infrastructure classes directly.

### .NET dependency-injection lifetimes and background work

| Lifetime  | Instance lifetime                         | Typical use                              |
| --------- | ----------------------------------------- | ---------------------------------------- |
| Transient | New instance each time it is resolved     | Lightweight, stateless helpers           |
| Scoped    | One instance per request scope            | `DbContext` and repositories             |
| Singleton | One instance for the application lifetime | Shared, thread-safe registries or caches |

Do not inject a scoped service such as `DbContext` directly into a singleton. A hosted service registered with `AddHostedService` is a singleton, so create a scope for each unit of background work and dispose it when that work finishes:

```csharp
public sealed class OutboxWorker : BackgroundService
{
     private readonly IServiceScopeFactory _scopeFactory;

     public OutboxWorker(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;

     protected override async Task ExecuteAsync(CancellationToken stoppingToken)
     {
          using var timer = new PeriodicTimer(TimeSpan.FromSeconds(10));

          while (await timer.WaitForNextTickAsync(stoppingToken))
          {
               await using var scope = _scopeFactory.CreateAsyncScope();
               var processor = scope.ServiceProvider.GetRequiredService<IOutboxBatchProcessor>();
               await processor.ProcessBatchAsync(stoppingToken);
          }
     }
}
```

Make singleton services thread-safe because requests and background operations may use them concurrently. Keep asynchronous I/O asynchronous: avoid `.Result` and `.Wait()` in request handling, do not wrap ordinary database or HTTP I/O in `Task.Run`, and pass the request or host `CancellationToken` through to EF Core and `HttpClient` calls. Cancellation is cooperative; it stops unnecessary work but does not undo a database commit that has already completed.

## 3. Choose Boundaries Before Patterns

### Layered architecture

A layered application separates concerns such as presentation, application, domain, and infrastructure. Layers help organize responsibilities, but they do not guarantee good boundaries. If every feature calls every other feature's repositories and entities, the code may still be tightly coupled.

### Clean Architecture and dependency direction

Clean Architecture emphasizes that business policy should not depend on databases, web frameworks, or vendor SDKs. The outer layer can depend on inner abstractions; the domain should not depend on EF Core or ASP.NET Core.

This does not mean every class needs an interface. Add an abstraction when it protects a meaningful boundary, enables substitution or testing, or keeps a volatile technical detail away from stable business policy. Avoid interfaces that only mirror a class without a useful purpose.

### Feature-oriented organization

For a growing API, organizing by feature can make related code easier to find:

```text
Features/
  Claims/
    RegisterClaimEndpoint.cs
    RegisterClaimHandler.cs
    ClaimRules.cs
    ClaimRepository.cs
  Payments/
    RefundPaymentEndpoint.cs
    RefundPaymentHandler.cs
Infrastructure/
  Persistence/
  ExternalServices/
```

This can coexist with layers. The useful question is whether a developer can understand and change one capability without navigating unrelated parts of the application.

### Modular monolith and microservices

A **modular monolith** is one deployable application with explicit internal modules and controlled dependencies. It is often a strong starting point: one deployment and transaction boundary, with room to evolve.

A **microservice** is an independently deployable service with its own operational and data responsibilities. It adds network failures, deployment coordination, observability needs, and eventual-consistency problems. Split a module into a service when there is a concrete reason, such as independent scaling, ownership by a separate team, deployment isolation, or a distinct security boundary.

A useful progression is:

1. Establish clear modules and data ownership.
2. Measure the actual bottleneck or coordination problem.
3. Extract a service only when the benefit outweighs distributed-system costs.

Do not claim that microservices are inherently more scalable or maintainable. A poorly bounded monolith can be hard to change; poorly bounded microservices can be harder.

## 4. Model Business Rules and Data Ownership

### Domain model

The domain model represents concepts and rules of the business. An entity has identity over time; a value object is defined by its values. Use these concepts when they clarify business behavior, not just to add classes.

For example, a claim may have identity and lifecycle, while a `Money` value should keep its amount and currency together. A rule such as “a claim cannot be approved before required evidence is received” should live close to the claim workflow, not be scattered across controllers and SQL queries.

### Invariants and transactions

An **invariant** is a rule that must remain true, such as “a payment cannot be refunded more than once.” Protect it at the authoritative boundary, and use database constraints or concurrency controls where appropriate. Application validation improves error messages but does not replace database integrity under concurrent requests.

A database transaction provides atomicity within that database. It does not make an HTTP call to another service part of the same atomic operation. When work crosses service boundaries, design explicit intermediate states, retries, idempotency, and compensation where needed.

### Data ownership

Each module or service should have a clear authority for the data it owns. Other components should use a supported contract rather than writing directly into its tables. In a monolith, modules may share one database while still keeping ownership rules; a shared database does not mean every module should freely mutate every table.

## 5. Design APIs as Contracts

An API contract includes more than its URL. It includes request and response schemas, status codes, validation behavior, authentication requirements, compatibility expectations, and error shape.

- Use HTTP methods and status codes consistently. A successful creation commonly returns `201 Created`; validation failures commonly return `400`; missing resources `404`; conflicts with current state `409`; unexpected failures `500`.
- Validate input at the boundary, then enforce business rules in the application/domain logic.
- Return stable DTOs rather than exposing persistence entities. Database schema changes should not accidentally become public API changes.
- Keep error responses useful to clients without exposing stack traces, secrets, or internal implementation details.
- Version deliberately when a breaking contract change is unavoidable. Prefer additive compatible changes when they meet the need.
- Propagate cancellation tokens through ASP.NET Core, EF Core, and outbound calls so abandoned requests can stop consuming resources.

### Idempotency

An operation is idempotent when repeating it has the same intended effect as performing it once. This matters when clients or infrastructure retry after a timeout: the server may have completed the first request even though the response was lost.

For important commands such as payment creation, accept an idempotency key, store its result under a uniqueness constraint, and return the existing result for a repeated key. Do not assume a network timeout means the operation failed.

## 6. Choose Communication Based on the Workflow

### Synchronous HTTP or gRPC

Use a synchronous call when the caller needs the result before it can continue and a direct dependency is acceptable. Set a timeout, define failure behavior, and avoid long chains where one user request waits on many services.

### Asynchronous messaging

Use a message broker when work can happen later, consumers need independent processing, or temporary receiver downtime should not block the sender. Messaging adds delivery retries, duplicate handling, ordering questions, schema evolution, and operational overhead.

A reliable event flow commonly combines:

- **Transactional outbox:** save business data and the event record in the same database transaction; publish the event asynchronously.
- **Idempotent consumer:** record or otherwise detect processed message IDs so redelivery does not repeat a business effect.
- **Explicit event contracts:** publish facts that have happened, with stable, versionable schemas.

Most brokers provide at-least-once delivery in common configurations, so consumers should be designed for duplicates. “Exactly once” across a database and broker is not a property to assume casually.

### Sagas and eventual consistency

A **saga** coordinates multiple local transactions across services. If a later step fails, the workflow may need a compensating action. Compensation is a new business action, not a time machine: a refund can compensate for a charge, but it does not erase the original charge from history.

Use eventual consistency only when the product can tolerate a period where different views have not caught up. Communicate pending states clearly and define how the workflow recovers from failures.

## 7. Design for Failure and Resilience

In-process method calls and remote calls have different failure modes. A remote dependency can be slow, unavailable, overloaded, or return malformed data.

- **Timeout:** limit how long a caller waits. Use explicit, realistic timeouts rather than relying on long defaults.
- **Retry:** retry transient failures only, with bounded attempts and backoff. Jitter helps avoid synchronized retry bursts.
- **Circuit breaker:** temporarily stop calls to a dependency that is repeatedly failing, allowing it to recover and protecting the caller.
- **Bulkhead:** isolate resource pools or workloads so one failing dependency cannot consume all capacity.
- **Fallback:** return a safe alternate result only when its meaning is correct for the business. Do not silently present stale or incomplete data as authoritative.
- **Rate limiting:** protect service capacity and provide predictable behavior under excess traffic.

Retries can amplify an outage. Do not blindly retry non-idempotent operations. If a request may have succeeded remotely before the connection failed, use idempotency or reconciliation instead of assuming it is safe to repeat.

A health endpoint should distinguish whether a process is alive from whether it is ready to receive traffic. A temporary vendor outage does not always mean the application itself should be restarted; report dependency health in a way that matches the deployment and recovery strategy.

## 8. Think About Performance and Scale with Evidence

First identify the constrained resource: database, CPU, memory, network, thread pool, or a downstream service. Measure before selecting a fix.

### Database and EF Core

- Project only the fields needed for the response; avoid loading entire entity graphs unnecessarily.
- Watch for N+1 queries, unbounded result sets, missing indexes, and long-running transactions.
- Use pagination for large collections and inspect query plans for important slow queries.
- Use `AsNoTracking` for read-only EF Core queries where tracking is not needed.
- Keep `DbContext` scoped to a request or unit of work; do not share it concurrently.
- Use optimistic concurrency where simultaneous updates can overwrite each other.

**Concurrency is not the same as a transaction.** A transaction makes a group of database operations atomic, but two valid transactions can still read the same old value and overwrite one another. For SQL Server, EF Core can use a `rowversion` column as an optimistic concurrency token:

```csharp
public byte[] RowVersion { get; private set; } = [];

// In the EF Core entity configuration:
builder.Property(claim => claim.RowVersion).IsRowVersion();
```

EF Core includes the original token in the update condition. If another write changed the row first, `SaveChangesAsync` throws `DbUpdateConcurrencyException`; handle it as a conflict by reloading, merging, or asking the caller to retry with current data. Also enforce critical uniqueness and integrity rules in the database, since application-level checks alone can race under concurrent requests.

### Caching

Caching reduces repeated work but introduces staleness and invalidation decisions. Define the key, expiration, invalidation behavior, and acceptable staleness. A cache should not become the sole authority for critical data unless its durability and consistency are intentionally designed for that role.

### Scaling

- **Vertical scaling:** give one instance more resources; straightforward but bounded.
- **Horizontal scaling:** run more instances; requires stateless request handling or shared/distributed state, and attention to connection pools and downstream capacity.
- A queue can smooth bursts, but it does not increase downstream processing capacity by itself. Monitor queue age and define backpressure or load shedding.

## 9. Security Is an Architectural Concern

Security is not a final middleware checkbox. Define trust boundaries and minimize what each identity and component is allowed to do.

### Authentication, authorization, and rate limiting in an API

**Authentication** answers “Who is making this request?” **Authorization** answers “May this identity perform this action on this resource?” A valid JWT proves an identity and carries claims; it does not automatically grant access to every endpoint or record.

For a JWT-protected ASP.NET Core API, validate tokens issued by a trusted identity provider. Validate signature, issuer, audience, and lifetime; do not write custom token-validation code or store signing secrets in source control.

```csharp
builder.Services
     .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
     .AddJwtBearer(options =>
     {
          options.Authority = builder.Configuration["Authentication:Authority"];
          options.Audience = builder.Configuration["Authentication:Audience"];
     });

builder.Services.AddAuthorization(options =>
{
     options.AddPolicy("Claims.Read", policy =>
          policy.RequireAuthenticatedUser()
                 .RequireClaim("permission", "claims.read"));
});

// Middleware order matters. Rate limiting follows routing and authentication
// here so a policy can partition authenticated traffic by user.
app.UseRouting();
app.UseAuthentication();
app.UseRateLimiter();
app.UseAuthorization();
```

Apply authorization at the endpoint, then perform resource-level checks in the use case when access depends on the specific claim or tenant:

```csharp
app.MapGet("/claims/{id:guid}", GetClaim)
     .RequireAuthorization("Claims.Read")
     .RequireRateLimiting("per-client");
```

- Return **401 Unauthorized** when credentials are missing or invalid; the caller must authenticate.
- Return **403 Forbidden** when the caller is authenticated but lacks permission.
- Roles can express broad job functions; claims or policies are often better for specific permissions. Check ownership and tenant boundaries against the requested resource, not only against token claims.
- For browser sign-in, OpenID Connect is commonly used to authenticate users. APIs typically validate access tokens; OAuth 2.0 describes delegated authorization. Choose a supported identity provider and standard library rather than inventing a token protocol.

**Rate limiting** bounds request volume to protect capacity and reduce accidental or abusive overload. It is not a replacement for authentication, authorization, input validation, or a web application firewall. ASP.NET Core provides fixed-window, sliding-window, token-bucket, and concurrency limiters; choose based on whether the requirement is a request quota over time or a cap on simultaneous work.

Example fixed-window policy, partitioned by authenticated user and otherwise by client IP:

```csharp
builder.Services.AddRateLimiter(options =>
{
     options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
     options.AddPolicy("per-client", httpContext =>
     {
          var userId = httpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
          var clientKey = userId ?? httpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown";

          return RateLimitPartition.GetFixedWindowLimiter(clientKey, _ =>
               new FixedWindowRateLimiterOptions
               {
                    PermitLimit = 100,
                    Window = TimeSpan.FromMinutes(1),
                    QueueLimit = 0,
                    AutoReplenishment = true
               });
     });
});
```

In production, tune limits to measured capacity and product needs. Consider stricter policies for expensive operations and separate limits for anonymous and authenticated traffic. If the API runs on multiple instances, an in-memory limiter applies per instance, not as one global quota; use gateway or distributed coordination when a shared limit is required. Behind a reverse proxy, trust forwarded client IP headers only from configured, trusted proxies, or clients may spoof their apparent IP. A throttled request should receive `429 Too Many Requests`; clients should back off rather than retrying immediately.

- Authenticate who the caller is; authorize whether that caller may perform this action on this resource.
- Enforce authorization on the server for each operation. Hiding a button in the client is not access control.
- Validate and normalize untrusted input; use parameterized database access and safe output handling.
- Store secrets in a managed secret store or protected configuration provider, not in source control.
- Use TLS for network communication and encrypt sensitive data at rest when required by the threat model or policy.
- Limit service identities and database permissions to the smallest necessary scope.
- Audit security-sensitive actions without logging passwords, tokens, payment details, or unnecessary personal data.
- Threat-model high-impact workflows: identify assets, actors, trust boundaries, abuse cases, and mitigations.

## 10. Make the System Operable

A production service needs enough information to answer: “Is it working?”, “What is failing?”, and “Which user operation was affected?”

- **Logs** describe discrete events and should include structured fields such as operation name, outcome, and correlation/trace ID.
- **Metrics** show aggregated behavior over time, such as request rate, error rate, latency percentiles, queue depth, and dependency failures.
- **Traces** connect work across application boundaries and help locate latency in a request path.

Use a shared trace context across HTTP and message boundaries. Avoid high-cardinality metric labels such as raw user IDs. Health checks are useful for orchestration, but they do not replace logs, metrics, traces, or alerts.

Operational design also includes graceful shutdown, bounded background work, database migration strategy, configuration validation, backups, restore testing, and a way to roll back or disable a risky change.

## 11. Deliver Changes Safely

Architecture includes how changes reach production:

- Keep configuration outside code and validate required settings at startup.
- Use automated unit tests for business rules and integration tests for important boundaries such as database mappings and message handling.
- Run backward-compatible database changes in stages when old and new application versions may overlap during deployment.
- Use health checks and readiness to control traffic during startup and shutdown.
- Use feature flags or staged rollout for changes whose impact is uncertain; have an explicit owner and removal plan for temporary flags.
- Monitor the release against defined indicators and have a rollback or mitigation path.

A deployment that cannot be safely observed or recovered from is an architectural risk, even if the code is well structured.

## 12. A Practical Architecture Decision Method

When asked to design or change a backend, use this sequence:

1. **Clarify the use case:** who calls it, what must happen, and what is the business outcome?
2. **State constraints:** expected load, latency, availability, data sensitivity, team ownership, budget, and existing systems.
3. **Identify critical data and invariants:** who owns the data, what must be atomic, and what may be eventually consistent?
4. **Draw the simplest request/data flow:** include dependencies and trust boundaries.
5. **Choose a communication style:** direct call for immediate answers; asynchronous messaging for decoupled work that can happen later.
6. **Walk through failures:** timeout, duplicate request, dependency outage, partial completion, and deployment overlap.
7. **Cover security and operations:** authorization, secrets, audit, telemetry, alerts, recovery, and rollback.
8. **Name tradeoffs and revisit triggers:** say why the choice fits now and what evidence would justify changing it.

This method is more convincing than listing technology names because it shows that choices follow requirements.

## 13. Interview Walkthrough: Registering and Processing a Claim

Suppose an API registers a claim, stores its details, and asks an accounting service to create a ledger entry.

A reasonable first design for a small-to-medium .NET product could be:

1. An authenticated `POST /claims` endpoint validates the request and checks the caller's authorization.
2. The application use case applies claim rules and writes the claim plus an outbox event in one SQL transaction.
3. The API returns `201 Created` with the claim identifier and current status, such as `PendingAccounting`.
4. A background publisher sends the event. The accounting consumer handles duplicate delivery idempotently and records the result.
5. A success or failure event updates the claim workflow. Failures are retried when transient; terminal failures are visible for investigation and follow a defined business recovery path.
6. Logs, metrics, and traces correlate the API request and asynchronous processing without exposing personal or financial data.

The tradeoff is that the response does not guarantee accounting has completed. If the product requires accounting to finish before confirming the claim, a synchronous call may be appropriate, but the endpoint must define timeout and partial-failure behavior. The requirement decides the flow.

### Sample interview explanation

> “I would start by clarifying whether claim registration must wait for accounting, the expected traffic, and what consistency the business requires. If the claim can be accepted before accounting finishes, I would persist the claim and an outbox record atomically, then process the event asynchronously. The consumer would be idempotent because messages can be redelivered. I would expose a pending status, instrument the workflow, and define retry and manual-recovery behavior. If the user needs an immediate accounting result, I would keep that dependency synchronous with a bounded timeout and an explicit failure response. I would not introduce a separate microservice or broker unless the ownership, scale, or availability requirements justify the extra operational cost.”

## 14. Quick Review Checklist

- Can I explain the request path from HTTP endpoint to database and back?
- Are business rules independent of controllers and infrastructure details?
- Is each module's data ownership clear?
- Do I know which operations need a database transaction and which cross service boundaries?
- Are retries bounded, and can retried commands safely handle duplicates?
- Have I considered authorization, sensitive data, and audit requirements?
- Can production support identify slow requests and failed dependencies?
- Is the service deployable, observable, and recoverable?
- Can I explain why this design is sufficient now and what would make me change it?
