# 🔧 Section 13: Advanced Architecture — Kubernetes, Security & Observability Deep Dive
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> These topics appear in **senior-level** interviews and differentiate mid-level from senior candidates. Even at 2.5 years, understanding these conceptually impresses interviewers.

---

## 📋 Table of Contents

1. [Kubernetes Health Probes — Liveness vs Readiness](#1-kubernetes-health-probes)
2. [Clean Architecture vs N-Tier Architecture](#2-clean-architecture-vs-n-tier-architecture)
3. [Correlation ID & Request Tracing](#3-correlation-id--request-tracing)
4. [Zero-Trust Security](#4-zero-trust-security)
5. [Single Sign-On (SSO) & External Identity Providers](#5-single-sign-on-sso--external-identity-providers)
6. [Memory Leak Detection & Profiling](#6-memory-leak-detection--profiling)
7. [API Versioning & Health Check Setup](#7-api-versioning--health-check-setup)
8. [gRPC in .NET](#8-grpc-in-net)
9. [SignalR — Real-Time Communication](#9-signalr--real-time-communication)
10. [Quick-Fire Advanced Checklist](#10-quick-fire-advanced-checklist)

---

## 1. Kubernetes Health Probes

### Q1: Liveness vs Readiness Probes — Why Both?

```
Liveness  → "Is the process still alive?" (Not deadlocked?)
            FAIL: Kubernetes RESTARTS the pod
            CHECK: /health/live — returns 200 if process is running

Readiness → "Is the app ready to accept traffic?"
            FAIL: Kubernetes REMOVES pod from load balancer (no new requests)
            CHECK: /health/ready — checks SQL Server, Redis, vendors connected
```

**Practical difference**:
- A pod can be **alive** (process running) but **not ready** (DB connection not yet established during startup warmup)
- During BSK startup: the background service initializes connection pools. Liveness passes immediately, Readiness only passes after all dependencies are verified

```csharp
// Program.cs — Health Check Registration
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection"),
        name: "SQLServerDB",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "ready" }) // Only in readiness checks
    .AddCheck<VendorHealthCheck>("VendorAPIs", tags: new[] { "ready" });

// Map two separate endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false, // Skip ALL checks — just verify process is running
    ResponseWriter = async (ctx, report) =>
    {
        ctx.Response.ContentType = "application/json";
        await ctx.Response.WriteAsync("{\"status\":\"alive\"}");
    }
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"), // Only "ready"-tagged checks
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse // Detailed JSON
});
```

**Kubernetes deployment YAML**:
```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 30
  failureThreshold: 3  # Restart after 3 consecutive failures

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3  # Remove from LB after 3 consecutive failures
```

---

### Q2: Custom Health Check Implementation

```csharp
// BSK: Custom health check that pings all vendor APIs
public class VendorHealthCheck : IHealthCheck
{
    private readonly IServiceHealthRegistry _healthRegistry;

    public VendorHealthCheck(IServiceHealthRegistry healthRegistry)
        => _healthRegistry = healthRegistry;

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var downVendors = new List<string>();

        foreach (var vendor in Enum.GetValues<ThirdPartyService>())
        {
            if (!_healthRegistry.IsServiceUp(vendor))
                downVendors.Add(vendor.ToString());
        }

        return Task.FromResult(
            downVendors.Any()
                ? HealthCheckResult.Degraded($"Vendors down: {string.Join(", ", downVendors)}")
                : HealthCheckResult.Healthy("All vendors operational"));
    }
}
```

---

## 2. Clean Architecture vs N-Tier Architecture

### Q3: Clean Architecture Dependency Rule

```
┌─────────────────────────────────────────────┐
│  Infrastructure Layer (Outer)                │
│  EF Core, SQL Server, HttpClient, Serilog   │
├─────────────────────────────────────────────┤
│  Application Layer                           │
│  Use Cases, Service Interfaces              │
├─────────────────────────────────────────────┤
│  Domain Layer (Core — Center)               │
│  Entities, Value Objects, Domain Events     │
│  ← Zero dependencies on frameworks/DB!      │
└─────────────────────────────────────────────┘

Dependency Rule: Dependencies point INWARD only
Inner layers know NOTHING about outer layers
```

**BSK's pragmatic approach** (N-Tier with Clean Architecture principles):
```
Controllers (Presentation)
    ↓ depends on
Repository Interfaces (Abstraction — Application layer)
    ↓ implements
Repository Implementations (Infrastructure — EF Core/Dapper)
    ↓ persists to
SQL Server (Storage)
```

**Key principle BSK follows**:
- Controllers never depend on `CaseAsyncRepository` (concrete)
- Controllers depend on `ICaseAsyncRepository` (interface)
- This is the core of Clean Architecture's Dependency Inversion

---

### Q4: When to use Microservices vs Monolith

| Scenario | Choose |
|---|---|
| Small team (<5 developers) | Monolith — less coordination overhead |
| Single deployment unit needed | Monolith |
| Each domain scales independently | Microservices |
| Different teams own different services | Microservices |
| Starting a new project | Monolith first, extract services when needed |
| BSK (small-medium insurance platform) | Monolith API + specific microservice (Accounting) |

**Martin Fowler's advice**: "Don't start with microservices. Monolith first. Decompose when you have real pain points (scaling, deployment frequency, team boundaries)."

---

## 3. Correlation ID & Request Tracing

### Q5: What is a Correlation ID? How to implement it in BSK?

**Problem**: BSK handles thousands of concurrent requests. Log lines from different requests are interleaved:
```
[10:05:01] CaseId: abc123 - Request started
[10:05:01] CaseId: xyz789 - DB query executed  ← different request!
[10:05:01] CaseId: abc123 - Payment processed
[10:05:01] ERROR: Connection failed            ← which request failed?
```

**Solution — Correlation ID**: Every request gets a unique GUID → injected into all log entries for that request.

```csharp
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    private const string CorrelationHeader = "X-Correlation-ID";

    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        // Reuse incoming correlation ID (from upstream service) or generate new one
        var correlationId = context.Request.Headers.TryGetValue(CorrelationHeader, out var existing)
            ? existing.ToString()
            : Guid.NewGuid().ToString();

        // Add to response so clients/downstream services can trace
        context.Response.Headers[CorrelationHeader] = correlationId;

        // Push to Serilog context — automatically appended to ALL logs in this scope
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
        // After this using block, CorrelationId is removed from log context
    }
}

// Register in Program.cs BEFORE other middleware
app.UseMiddleware<CorrelationIdMiddleware>();
```

**Result in logs**:
```json
{"@t":"2026-09-27T10:05:01","@m":"DB query executed","CorrelationId":"c8f1a2b3-...","CaseId":"abc123"}
{"@t":"2026-09-27T10:05:01","@m":"ERROR Connection failed","CorrelationId":"c8f1a2b3-..."}
// Now search by CorrelationId → see ONLY your request's log lines!
```

---

## 4. Zero-Trust Security

### Q6: Zero-Trust Principle in Backend APIs

**Traditional Security**: Trust everything inside the network perimeter.
**Zero-Trust**: "Never Trust, Always Verify" — assume breach has already happened.

**BSK Zero-Trust implementations**:

```csharp
// 1. Resource-Based Authorization (not just role-based)
// Bad: Only checks "is user logged in"
[Authorize]
public async Task<IActionResult> GetCase(string id) { ... }

// Good: Verifies user CAN access THIS specific case
[Authorize]
public async Task<IActionResult> GetCase(string id)
{
    var @case = await _repo.GetCaseById(id);
    if (@case == null) return NotFound();

    // Zero-trust: verify this user owns this case
    var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
    if (@case.AssignedLawyerId != userId && !User.IsInRole("Admin"))
        return Forbid(); // 403 — authenticated but not authorized for THIS resource

    return Ok(@case);
}

// 2. Validate ALL requests — even from internal services
// Every microservice must present a valid JWT, even coming from another BSK service
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,       // Is it from our trusted issuer?
    ValidateAudience = true,     // Is it meant for this specific API?
    ValidateLifetime = true,     // Is it expired?
    ValidateIssuerSigningKey = true, // Was it signed by our key?
    ClockSkew = TimeSpan.Zero    // No tolerance for expired tokens
};

// 3. Principle of Least Privilege — use minimal claims
// Don't give a vendor-facing endpoint full admin token
// Generate scoped tokens with only the required claims
```

---

## 5. Single Sign-On (SSO) & External Identity Providers

### Q7: How SSO works with ASP.NET Core

```
User logs in once to Identity Provider (Auth0, Entra ID, Keycloak)
    ↓
IdP issues JWT tokens
    ↓
All BSK services validate the JWT LOCALLY (no call to IdP per request)
    ↓
IdP's public keys (JWKS) cached at API startup for signature verification
```

```csharp
// Configure ASP.NET Core to validate Entra ID / Auth0 tokens
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // Authority = IdP's base URL — ASP.NET Core fetches JWKS from here automatically
        options.Authority = "https://login.microsoftonline.com/{tenant-id}/v2.0";
        options.Audience = "api://bsk-api-client-id"; // Our API's registered client ID

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });
```

**What `Authority` does**: ASP.NET Core middleware calls `{Authority}/.well-known/openid-configuration` to discover the JWKS endpoint, downloads public keys, and uses them to verify JWT signatures — **no per-request calls to IdP**.

---

## 6. Memory Leak Detection & Profiling

### Q8: Common Memory Leak Causes in C#

```csharp
// LEAK 1: Static collection growing forever
public static class EventLog
{
    private static readonly List<string> _events = new(); // Never cleared!
    public static void Record(string ev) => _events.Add(ev); // Leak!
}

// FIX: Bounded collection or periodic cleanup
private static readonly LinkedList<string> _events = new();
public static void Record(string ev)
{
    _events.AddLast(ev);
    if (_events.Count > 1000) _events.RemoveFirst(); // Bounded
}

// LEAK 2: Unsubscribed event handlers
public class OrderProcessor
{
    public OrderProcessor(ShipmentService shipment)
    {
        shipment.OnShipped += HandleShipment; // Shipment holds reference to OrderProcessor!
        // If OrderProcessor is "done" but never calls -= HandleShipment → LEAK
    }

    ~OrderProcessor()
    {
        // Too late — finalizer called by GC
    }
}

// FIX: Implement IDisposable, unsubscribe in Dispose
public class OrderProcessor : IDisposable
{
    private readonly ShipmentService _shipment;
    public OrderProcessor(ShipmentService shipment)
    {
        _shipment = shipment;
        _shipment.OnShipped += HandleShipment;
    }
    private void HandleShipment(object? s, EventArgs e) { }
    public void Dispose() => _shipment.OnShipped -= HandleShipment; // Clean up!
}

// LEAK 3: Captive Dependency (see Section 03)
// Scoped DbContext injected into Singleton — never disposed!
```

---

### Q9: .NET Diagnostic Tools for Production

```bash
# 1. dotnet-counters — Real-time metrics in terminal
dotnet-counters monitor --process-id <PID> --counters System.Runtime

# Key metrics to watch:
# gc-heap-size         → growing? Memory leak suspected
# threadpool-thread-count → near max? Thread starvation
# active-timer-count   → too many? Timer leak

# 2. dotnet-dump — Memory dump for post-mortem analysis
dotnet-dump collect -p <PID>          # Capture dump
dotnet-dump analyze <dumpfile>        # Interactive analysis

# Inside analysis:
dumpheap -stat                        # Show all objects by type and count
gcroot <object-address>               # Find what's holding this object alive!

# 3. dotnet-trace — CPU profiling
dotnet-trace collect -p <PID> --duration 00:00:30

# 4. dotnet-stack — Show all thread call stacks (find deadlocks)
dotnet-stack report -p <PID>
```

---

## 7. API Versioning & Health Check Setup

### Q10: API Versioning — Three Approaches

```csharp
// Approach 1: URL Segment (Most common, BSK-style)
// bsk/api/v1/Case/GetCaseById
// bsk/api/v2/Case/GetCaseById

builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // Returns Supported-Versions header
});

[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("bsk/api/v{version:apiVersion}/[controller]")]
public class CaseController : ControllerBase
{
    [HttpGet("GetCaseById")]
    [MapToApiVersion("1.0")]
    public IActionResult GetCaseByIdV1(string id) { /* V1 behavior */ return Ok(); }

    [HttpGet("GetCaseById")]
    [MapToApiVersion("2.0")]
    public IActionResult GetCaseByIdV2(string id) { /* V2 behavior */ return Ok(); }
}

// Approach 2: Query String
// bsk/api/Case/GetCaseById?api-version=2.0

// Approach 3: Custom Header
// X-API-Version: 2.0
```

**Versioning strategy**: Never break existing clients. When V2 changes behavior, keep V1 working until clients migrate.

---

## 8. gRPC in .NET

### Q11: REST vs gRPC — When to use each

| Feature | REST (BSK Current) | gRPC |
|---|---|---|
| **Protocol** | HTTP/1.1 — text (JSON) | HTTP/2 — binary (Protocol Buffers) |
| **Performance** | Good | ~7x faster serialization |
| **Browser Support** | Full | Limited (needs gRPC-Web) |
| **Use Case** | Public APIs, client-facing | Service-to-service internal calls |
| **Streaming** | Limited | Native bi-directional streaming |
| **Schema** | Optional (OpenAPI) | Mandatory (.proto file) |

**BSK Scenario**: BSK API communicates with the Accounting microservice via REST HTTP. A gRPC migration would give better performance for internal calls.

```csharp
// gRPC Service Definition (.proto file)
// service AccountingService {
//     rpc GetVoucher (VoucherRequest) returns (VoucherResponse);
// }

// .NET gRPC Client
var channel = GrpcChannel.ForAddress("https://bsk-accounting-svc:443");
var client = new AccountingService.AccountingServiceClient(channel);
var voucher = await client.GetVoucherAsync(new VoucherRequest { CaseId = "id-123" });
```

---

## 9. SignalR — Real-Time Communication

### Q12: SignalR for Real-Time Dashboards

**BSK Use Case**: Executive dashboard could push live updates (new case registered, payment received) instead of requiring page refresh.

```csharp
// Hub definition
public class DashboardHub : Hub
{
    public async Task SendMetricsUpdate(DashboardMetricsDto metrics)
        => await Clients.All.SendAsync("MetricsUpdated", metrics);
}

// Program.cs
builder.Services.AddSignalR();
app.MapHub<DashboardHub>("/hubs/dashboard");

// Push from API when a case is registered
public class CaseService
{
    private readonly IHubContext<DashboardHub> _hubContext;

    public async Task RegisterCaseAsync(CaseModel model)
    {
        await _caseRepo.SaveAsync(model);

        // Notify all connected dashboard clients in real-time
        await _hubContext.Clients.All.SendAsync("CaseRegistered", new
        {
            model.Id,
            model.ClaimantName,
            Timestamp = DateTime.UtcNow
        });
    }
}
```

**Transport fallback**: WebSockets → Server-Sent Events → Long Polling (SignalR picks the best supported by client automatically).

---

## 10. Quick-Fire Advanced Checklist

| Question | Short Answer |
|---|---|
| Kubernetes Liveness vs Readiness | Liveness: process alive (restart if fail); Readiness: dependencies ready (remove from LB if fail) |
| What is JWKS? | JSON Web Key Set — public keys from IdP for JWT signature verification |
| What is ClockSkew in JWT? | Tolerance for clock differences between token issuer and validator — set to Zero for strict |
| What is Correlation ID? | Unique GUID per request — enables tracing across log entries |
| What is Zero-Trust? | "Never Trust, Always Verify" — validate every request regardless of network origin |
| RBAC vs ABAC | Role-Based: roles grant permissions; Attribute-Based: fine-grained rules on attributes/context |
| gRPC vs REST | gRPC: binary, HTTP/2, faster, service-to-service; REST: JSON, HTTP/1.1, universal |
| SignalR transports | WebSockets → Server-Sent Events → Long Polling (auto-negotiated) |
| `dotnet-dump` purpose | Capture and analyze memory dumps for leak investigation |
| `dotnet-trace` purpose | CPU sampling profiler — find slow methods in production |
| What is BFF? | Backend for Frontend — dedicated API gateway optimized per client type (mobile/web) |
| What is Service Mesh? | Infrastructure layer (Istio/Linkerd) handling service-to-service auth, observability, retries |
| What is Eventual Consistency? | Distributed systems guarantee consistency eventually, not immediately (Saga Pattern) |
| What is Idempotency? | Same request produces same result if executed multiple times — critical for payment APIs |
| Clean Architecture rule | Dependencies point INWARD only — domain has no external dependencies |

---

## 🏁 Talking Points for Advanced Architecture

> *"In BSK, we follow Clean Architecture principles — our controllers depend on repository interfaces, not concrete implementations, which enables easy unit testing. For Kubernetes, I'd configure separate /health/live and /health/ready endpoints — liveness just checks the process is running, while readiness verifies all 7 vendor connections are operational. For observability, I'd add Correlation ID middleware (each request gets a unique GUID injected into all its log entries) and OpenTelemetry for distributed tracing across our monolith-to-accounting-service HTTP boundary."*

---
*Final file in the series. See the Master Index: [Section_12_Master_Index.md](./Section_12_Master_Index.md)*
