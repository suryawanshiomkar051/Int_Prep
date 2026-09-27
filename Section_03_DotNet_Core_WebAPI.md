# 🌐 Section 3: .NET Core & Web API — Interview Preparation Guide
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> This is the **highest-weightage section** for a .NET backend dev role. Every answer is backed by BSK production code patterns.

---

## 📋 Table of Contents

1. [ASP.NET Core Fundamentals](#1-aspnet-core-fundamentals)
2. [HTTP Verbs & Status Codes](#2-http-verbs--status-codes)
3. [API Routing & Model Binding](#3-api-routing--model-binding)
4. [Authentication & Authorization (JWT)](#4-authentication--authorization-jwt)
5. [Middleware Pipeline](#5-middleware-pipeline)
6. [Dependency Injection (DI)](#6-dependency-injection-di)
7. [Action Filters & Decorators](#7-action-filters--decorators)
8. [Background Services (IHostedService)](#8-background-services-ihostedservice)
9. [Calling External APIs (HttpClient)](#9-calling-external-apis-httpclient)
10. [Caching Strategies](#10-caching-strategies)
11. [API Versioning & Minimal APIs](#11-api-versioning--minimal-apis)
12. [Performance & Resilience](#12-performance--resilience)
13. [Quick-Fire Checklist](#13-quick-fire-checklist)

---

## 1. ASP.NET Core Fundamentals

### Q1: .NET Framework vs .NET Core — Key Differences

| Feature | .NET Framework (Legacy) | .NET Core / .NET 8 (Modern) |
|---|---|---|
| **Cross-Platform** | Windows only | Windows, Linux, macOS (Docker-ready) |
| **Performance** | Good, but legacy overhead | TechEmpower benchmark leader |
| **Entry Point** | `Global.asax` + `Web.config` | Unified `Program.cs` |
| **Dependency Injection** | Requires Autofac/Unity/Ninject | Native, built-in DI container |
| **Deployment** | Machine-wide GAC installation | Self-contained, side-by-side, Dockerized |
| **Hosting** | IIS only | Kestrel (self-hosted), IIS, Nginx |

**Why BSK chose .NET Core 8**:
1. Cross-platform → can run on Linux containers (lower hosting cost)
2. Native async — deeply integrated throughout controllers, EF Core, Kestrel
3. Native DI — no third-party container needed
4. Opt-in middleware pipeline — pay overhead only for what you register

---

### Q2: HTTP Request Lifecycle in ASP.NET Core

```
[Client HTTP Request]
        ↓
1. Kestrel / IIS — Parses HTTP, builds HttpContext object
        ↓
2. Middleware Pipeline
   UseMiddleware<ThirdPartyExceptionMiddleware>()  ← BSK: global error catcher
   UseHttpsRedirection()
   UseCors("CorsPolicy")
   UseAuthentication()    ← MUST be before Authorization
   UseAuthorization()
        ↓
3. Endpoint Routing — matches URL to controller action
        ↓
4. Filter Pipeline
   Authorization Filters → Resource Filters → Model Binding → Action Filters → Exception Filters
        ↓
5. Controller Action Executes (Business logic, DB calls)
        ↓
6. Result Filters → JSON Serialization → Response
        ↓
[HTTP Response back through middleware chain to Kestrel]
```

---

### Q3: .NET Compilation Lifecycle (JIT vs AOT)

```
C# Source Code
      ↓ (Roslyn Compiler)
Intermediate Language (IL) stored in .dll
      ↓ (CLR at runtime)
JIT Compiler → Native Machine Code (x64/ARM)
```

**Native AOT** (modern alternative in .NET 8):
- Compiles C# directly to native binary at build time (no JIT at runtime)
- Faster startup, smaller memory footprint
- Ideal for serverless functions and microservices

---

## 2. HTTP Verbs & Status Codes

### Q4: HTTP Verbs with BSK examples

| Verb | Purpose | BSK Example |
|---|---|---|
| **GET** | Retrieve data (safe, idempotent) | `[HttpGet("GetCaseById")]` — fetch case details |
| **POST** | Create new resource | `[HttpPost("ConsumerCourtManualPayment")]` — submit receipt |
| **PUT** | Replace/Update resource (idempotent) | `[HttpPut("UpdateAcceptCaseStatusInCaseDetails")]` |
| **DELETE** | Delete resource | `[HttpDelete("DeleteDocument")]` — remove file metadata |
| **PATCH** | Partial update (not full replacement) | Update only specific case fields |

---

### Q5: HTTP Status Codes — BSK's BaseResponseStatus pattern

```csharp
// BSK wraps ALL responses in a standardized class
public class BaseResponseStatus
{
    public string StatusCode { get; set; }    // "200", "404", etc.
    public string StatusMessage { get; set; } // Human-readable
    public object ResponseData { get; set; }  // Payload
}

// Usage in CaseController.cs
if (data == null)
{
    baseResponse.StatusCode = StatusCodes.Status404NotFound.ToString();
    baseResponse.StatusMessage = "Data Not Found";
    return Ok(baseResponse); // HTTP 200 wrapping a logical 404
}

// ThirdPartyExceptionMiddleware — direct HTTP 503
context.Response.StatusCode = (int)HttpStatusCode.ServiceUnavailable;
var response = new BaseResponseStatus { StatusCode = "503", StatusMessage = "Service unavailable" };
await context.Response.WriteAsync(JsonSerializer.Serialize(response));
```

**Status Code Groups**:
- `2xx` — Success (200 OK, 201 Created, 204 No Content)
- `4xx` — Client Error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 429 Too Many Requests)
- `5xx` — Server Error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable)

---

## 3. API Routing & Model Binding

### Q6: Attribute Routing in BSK

```csharp
[Route("bsk/api/[controller]")]  // Controller-level: bsk/api/Case
[ApiController]
[Authorize]
public class CaseController : ControllerBase
{
    // Route: GET bsk/api/Case/GetCaseById?Id=...
    [HttpGet("GetCaseById")]
    public async Task<IActionResult> GetCaseById(string Id) { ... }

    // Route: POST bsk/api/Case/ConsumerCourtManualPayment
    [HttpPost("ConsumerCourtManualPayment")]
    public async Task<IActionResult> ConsumerCourtManualPayment([FromBody] ReceiptsModel createReceipt) { ... }

    // Route with path parameter: DELETE bsk/api/Document/DeleteDocument/abc123
    [HttpDelete("DeleteDocument/{id}")]
    public async Task<IActionResult> DeleteDocument(string id) { ... }
}
```

---

### Q7: Model Binding Sources

| Attribute | Source | BSK Example |
|---|---|---|
| `[FromBody]` | JSON request body | `[FromBody] ReceiptsModel createReceipt` |
| `[FromQuery]` | URL query string `?Id=123` | `string Id` (implicit for GET primitives) |
| `[FromRoute]` | URL route segment `/Case/GetById/{id}` | `string id` from `{id}` |
| `[FromForm]` | Multipart form data / file uploads | File upload endpoints |
| `[FromHeader]` | HTTP header | Custom header values |

---

### Q8: Steps to create a new Web API endpoint (BSK workflow)

```
[1] Define Entity Model (DataModel/Case.cs)
        ↓
[2] Register in ApplicationDBContext.cs (DbSet<Case> Cases)
        ↓
[3] Create Repository Interface (Repository/Interface/ICaseAsyncRepository.cs)
        ↓
[4] Implement Repository (Repository/CaseAsyncRepository.cs) — EF Core / Dapper
        ↓
[5] Register in Program.cs:
    builder.Services.AddScoped<ICaseAsyncRepository, CaseAsyncRepository>();
        ↓
[6] Create/Update Controller Action
```

```csharp
// Step 6: Controller
[ApiController]
[Route("bsk/api/[controller]")]
[Authorize]
public class CaseController : ControllerBase
{
    private readonly ICaseAsyncRepository _repo;
    public CaseController(ICaseAsyncRepository repo) => _repo = repo;

    [HttpGet("GetCaseById")]
    public async Task<IActionResult> GetCaseById(string Id)
    {
        var data = await _repo.GetCaseById(Id);
        if (data == null)
            return Ok(new BaseResponseStatus { StatusCode = "404", StatusMessage = "Data Not Found" });
        return Ok(new BaseResponseStatus { StatusCode = "200", ResponseData = data });
    }
}
```

---

## 4. Authentication & Authorization (JWT)

### Q9: JWT Bearer Authentication in BSK

```csharp
// Program.cs — JWT Registration
builder.Services.AddAuthentication(option =>
{
    option.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    option.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
}).AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = jwtTokenConfig.Issuer,
        ValidAudience = jwtTokenConfig.Audience,
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtTokenConfig.Secret)),
        ClockSkew = TimeSpan.Zero  // No tolerance for expired tokens
    };
});
```

```csharp
// On Controllers — enforce JWT authentication
[Route("bsk/api/[controller]")]
[ApiController]
[Authorize]                          // ALL actions require valid JWT
public class DocumentController : ControllerBase { }

// Role-based authorization
[Authorize(Roles = "Admin,Officer")]
public async Task<IActionResult> AdminOnlyEndpoint() { }

// Allow anonymous (login, health check)
[AllowAnonymous]
[HttpPost("Login")]
public async Task<IActionResult> Login([FromBody] LoginModel model) { }
```

**How JWT works**:
1. User logs in → Server validates credentials → Creates JWT with claims (UserId, Role, Expiry)
2. Client stores JWT (usually in localStorage or cookie)
3. Client sends JWT in `Authorization: Bearer <token>` header on every request
4. Server validates token signature, expiry, issuer, audience on each request

---

### Q10: CORS Configuration in BSK

```csharp
// Program.cs — Specific origins only (NEVER use wildcard * in production)
builder.Services.AddCors(options =>
{
    options.AddPolicy("CorsPolicy", policy =>
    {
        policy.WithOrigins("https://claimant.bimasevak.com", "https://admin.bimasevak.com")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials(); // Required for JWT cookies
    });
});

// Middleware pipeline — CORS before authentication
app.UseCors("CorsPolicy");
app.UseAuthentication();
app.UseAuthorization();
```

---

## 5. Middleware Pipeline

### Q11: Middleware Pipeline — Registration Order Matters!

```csharp
// BSK Program.cs middleware order
app.UseMiddleware<ThirdPartyExceptionMiddleware>(); // 1. Outermost — catches all errors

app.UseHttpsRedirection();
app.UseCors("CorsPolicy");           // 2. CORS — before auth

app.UseAuthentication();             // 3. Auth — MUST be before Authorization
app.UseAuthorization();              // 4. Authz — needs authenticated user from step 3

app.MapControllers();                // 5. Routes to controllers
```

**Rule**: Order is critical! Authentication must run before Authorization — otherwise the system won't know who the user is when checking permissions.

---

### Q12: Custom Middleware — BSK ThirdPartyExceptionMiddleware

```csharp
public class ThirdPartyExceptionMiddleware
{
    private readonly RequestDelegate _next;

    public ThirdPartyExceptionMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context); // Pass to next middleware
        }
        catch (ThirdPartyServiceUnavailableException ex)
        {
            LogHelper.Warn("3rd party service down", ("ServiceName", ex.ServiceName));
            await WriteResponseAsync(context, HttpStatusCode.ServiceUnavailable, "503", ex.UserMessage);
        }
        catch (ThirdPartyApiException ex)
        {
            LogHelper.Error("3rd party vendor error response", ex);
            await WriteResponseAsync(context, HttpStatusCode.BadGateway, "502", ex.UserMessage);
        }
    }

    private static async Task WriteResponseAsync(HttpContext context, HttpStatusCode code, string statusCode, string message)
    {
        context.Response.StatusCode = (int)code;
        context.Response.ContentType = "application/json";
        var response = new BaseResponseStatus { StatusCode = statusCode, StatusMessage = message };
        await context.Response.WriteAsync(JsonSerializer.Serialize(response));
    }
}
```

**Benefits**:
- **Encapsulation** — Controllers free of repetitive error handling
- **Security** — No stack traces leaked to clients
- **Consistency** — Standardized `BaseResponseStatus` format always

---

### Q13: Middleware vs Action Filters — When to use which?

| Feature | Middleware | Action Filters |
|---|---|---|
| **Scope** | Global — ALL HTTP requests | MVC-only — Controller action requests |
| **Pipeline Level** | Low-level HttpContext | High-level MVC context (ModelState, params) |
| **Access to MVC** | No controller/action context | Full access to controller attributes, model |
| **Use for** | CORS, auth, compression, global errors, logging | Custom validation, audit, caching per-action |

```csharp
// Action Filter example
public class CustomAuditFilter : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        // Before action: log inputs, check permissions
        var resultContext = await next(); // Execute the action
        // After action: log output, performance metrics
    }
}

// BSK reference: [AuthorizeAction] custom filter (commented for local testing)
// Checks database-driven PageMenu permissions dynamically
```

---

## 6. Dependency Injection (DI)

### Q14: DI Lifetimes — Transient, Scoped, Singleton

```
[Transient]  → New instance EVERY TIME it's requested from DI container
[Scoped]     → New instance ONCE PER HTTP REQUEST (shared within same request)
[Singleton]  → ONE instance for the ENTIRE application lifetime (all users, all threads)
```

**BSK Registrations in Program.cs**:
```csharp
// Scoped (most repositories and DbContext)
builder.Services.AddDbContext<ApplicationDBContext>(...); // Scoped by default
builder.Services.AddScoped<ICaseAsyncRepository, CaseAsyncRepository>();
builder.Services.AddScoped<ConnectionHandler>();

// Singleton (shared state, thread-safe services)
builder.Services.AddSingleton<ISqlCacheService, SqlCacheService>();
builder.Services.AddSingleton<IServiceHealthRegistry, ServiceHealthRegistry>();
// Must be thread-safe ConcurrentDictionary internally!

// Transient (lightweight, stateless services)
builder.Services.AddTransient<IEmailService, EmailService>();
```

**Captive Dependency (DANGER)**:
```csharp
// BAD: Scoped DbContext captured inside Singleton — NEVER do this!
public class SingletonService
{
    private readonly ApplicationDBContext _dbContext; // Scoped!
    public SingletonService(ApplicationDBContext dbContext) // DBContext will NEVER be disposed!
    { _dbContext = dbContext; } // Connection pool exhaustion, stale entities!
}

// GOOD: Use IServiceScopeFactory to create temporary scopes
public class SingletonService
{
    private readonly IServiceScopeFactory _scopeFactory;
    public SingletonService(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;

    public async Task DoWorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDBContext>();
        // DbContext properly scoped and disposed here
    }
}
```

---

### Q15: Legacy DI (Framework) vs Modern DI (.NET Core)

```csharp
// Legacy .NET Framework (Global.asax)
var container = new UnityContainer();
container.RegisterType<ICaseAsyncRepository, CaseAsyncRepository>();
DependencyResolver.SetResolver(new UnityDependencyResolver(container));

// Modern .NET Core (Program.cs) — Clean, built-in
builder.Services.AddScoped<ICaseAsyncRepository, CaseAsyncRepository>();
```

---

## 7. Action Filters & Decorators

### Q16: Key Controller Attributes/Decorators in BSK

```csharp
[Route("bsk/api/[controller]")]  // Controller base route
[ApiController]                  // Enables: auto 400 on invalid model, param source inference
[Authorize]                      // Enforces valid JWT on all actions

public class CaseController : ControllerBase
{
    [HttpGet("GetCaseById")]     // HTTP verb + action route
    public async Task<IActionResult> GetCaseById(
        [FromQuery] string Id)  // Parameter binding from query string
    { }

    [HttpPost("CreateCase")]
    public async Task<IActionResult> CreateCase(
        [FromBody] CreateCaseModel model)  // From JSON body
    { }

    [HttpDelete("DeleteDocument/{id}")]
    public async Task<IActionResult> DeleteDocument(
        [FromRoute] string id)  // From URL path segment
    { }

    [AllowAnonymous]  // Override [Authorize] on controller
    [HttpGet("Health")]
    public IActionResult Health() => Ok("Healthy");
}
```

---

## 8. Background Services (IHostedService)

### Q17: BSK's Background Services — ThirdPartyHealthMonitorService

```csharp
public class ThirdPartyHealthMonitorService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<ThirdPartyHealthMonitorService> _logger;

    public ThirdPartyHealthMonitorService(IServiceProvider sp, ILogger<ThirdPartyHealthMonitorService> logger)
    {
        _serviceProvider = sp;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(60));

        while (!stoppingToken.IsCancellationRequested && await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                using var scope = _serviceProvider.CreateScope(); // Scoped service in Singleton!
                var healthChecker = scope.ServiceProvider.GetRequiredService<IVendorHealthChecker>();
                await healthChecker.ProbeAllVendorsAsync();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Background health probe failed.");
            }
        }
    }
}

// Registration in Program.cs:
builder.Services.AddHostedService<ThirdPartyHealthMonitorService>();
```

**BSK also has**: `CircularListInitializerHostedService` for background case assignment queues.

**Key Rule**: Background services are **Singletons**. To access Scoped services (DbContext, Repositories), use `IServiceProvider.CreateScope()` inside `ExecuteAsync`.

---

## 9. Calling External APIs (HttpClient)

### Q18: IHttpClientFactory — Why and How

**Problem with raw `new HttpClient()`**:
```csharp
// BAD — Socket Exhaustion!
using (var client = new HttpClient())
{
    var response = await client.GetAsync("https://api.com");
} // HttpClient disposed but underlying OS socket stays in TIME_WAIT for 4 minutes!
// High traffic = socket pool exhausted = app crashes!
```

**Solution — IHttpClientFactory (BSK pattern)**:
```csharp
// Program.cs — Named clients with Polly resilience
builder.Services.AddHttpClient("SurepassClient", client =>
    client.BaseAddress = new Uri("https://kyc-api.surepass.io"))
    .AddResilienceHandler(PollyPolicies.SurepassPipeline, PollyPolicies.ConfigureHttpPipeline);

builder.Services.AddHttpClient("AccountingServiceClient", client =>
    client.BaseAddress = new Uri("https://bsk-stg-accountingsvc.shauryatechnosoft.com"))
    .AddResilienceHandler(PollyPolicies.AccountingServicePipeline, PollyPolicies.ConfigureHttpPipeline);

// Repository usage
private readonly IHttpClientFactory _clientFactory;

public async Task<bool> VerifyPanWithSurepass(string panNumber)
{
    var client = _clientFactory.CreateClient("SurepassClient"); // Reuses pooled socket

    var requestPayload = new { pan = panNumber };
    var jsonContent = new StringContent(JsonSerializer.Serialize(requestPayload), Encoding.UTF8, "application/json");

    var response = await client.PostAsync("api/v1/pan-verification", jsonContent);

    if (response.IsSuccessStatusCode)
    {
        var responseString = await response.Content.ReadAsStringAsync();
        return true; // Parse result
    }
    return false;
}
```

---

### Q19: Polly — Retry & Circuit Breaker Policies

```csharp
public static class PollyPolicies
{
    public static void ConfigureHttpPipeline(ResiliencePipelineBuilder<HttpResponseMessage> builder)
    {
        builder
            // Retry with exponential backoff: 2s, 4s, 8s
            .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
            {
                MaxRetryAttempts = 3,
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true  // Adds randomness to prevent thundering herd
            })
            // Circuit Breaker: trip if 50% requests fail in 30s window
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
            {
                FailureRatio = 0.5,
                SamplingDuration = TimeSpan.FromSeconds(30),
                MinimumThroughput = 8,
                BreakDuration = TimeSpan.FromSeconds(15) // Wait 15s before trying again
            });
    }
}
```

**Circuit Breaker states**: `Closed` (normal) → `Open` (tripped, fail fast) → `Half-Open` (test single request) → `Closed` (if test succeeds).

---

## 10. Caching Strategies

### Q20: In-Memory vs Distributed Cache

| Feature | IMemoryCache | IDistributedCache (Redis / SQL) |
|---|---|---|
| **Storage** | Server's RAM | External Redis / SQL Server table |
| **Multi-server** | ❌ Only on one server | ✅ Shared across all app instances |
| **Speed** | Fastest | Fast (network hop) |
| **BSK Usage** | Internal single-server caching | JWT token blacklist (SqlCacheService) |

**BSK Distributed SQL Cache**:
```csharp
// Program.cs
builder.Services.AddDistributedSqlServerCache(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("DefaultConnection");
    options.SchemaName = "dbo";
    options.TableName = "TokensCache";
});

// Usage: Cache-Aside Pattern
public async Task<CaseDto> GetCaseDetailsAsync(Guid caseId)
{
    string cacheKey = $"case:{caseId}";

    var cachedData = await _cache.GetStringAsync(cacheKey);
    if (!string.IsNullOrEmpty(cachedData))
        return JsonSerializer.Deserialize<CaseDto>(cachedData);

    var dbData = await _dbRepo.GetCaseById(caseId.ToString());
    var options = new DistributedCacheEntryOptions()
        .SetAbsoluteExpiration(TimeSpan.FromMinutes(15));

    await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(dbData), options);
    return dbData;
}
```

---

## 11. API Versioning & Minimal APIs

### Q21: API Versioning — Common Approaches

1. **URL Path** (most popular): `bsk/api/v1/Case/GetCaseById`
2. **Query Parameter**: `?api-version=1.0`
3. **HTTP Header**: `X-API-Version: 1.0`

```csharp
// NuGet: Asp.Versioning.Http
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;  // Returns supported versions in response headers
});

[Route("bsk/api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class CaseController : ControllerBase
{
    [HttpGet("GetCaseById")]
    [MapToApiVersion("1.0")]
    public IActionResult GetCaseByIdV1(string Id) { }

    [HttpGet("GetCaseById")]
    [MapToApiVersion("2.0")]
    public IActionResult GetCaseByIdV2(string Id) { } // New v2 behavior
}
```

---

### Q22: Minimal APIs vs Controller-Based APIs

| Feature | Controller-Based | Minimal API |
|---|---|---|
| **Code** | Class with attributes | Lambda in Program.cs |
| **Performance** | Slightly slower startup | Slightly faster startup |
| **Best For** | Large enterprise APIs (BSK) | Small microservices, serverless |

```csharp
// Minimal API (Program.cs)
app.MapGet("bsk/api/Case/GetCaseById/{id}", async (string id, ICaseAsyncRepository repo) =>
{
    var data = await repo.GetCaseById(id);
    return data != null ? Results.Ok(data) : Results.NotFound();
});
```

---

## 12. Performance & Resilience

### Q23: Response Compression & Rate Limiting

```csharp
// Response Compression (reduces payload size)
builder.Services.AddResponseCompression(options => { options.EnableForHttps = true; });
app.UseResponseCompression();

// Rate Limiting (.NET 7+)
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("StrictPolicy", opt =>
    {
        opt.PermitLimit = 10;
        opt.Window = TimeSpan.FromSeconds(30);
        opt.QueueLimit = 2;
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});
app.UseRateLimiter();

// Apply to specific endpoint
[HttpGet("GetCaseById")]
[EnableRateLimiting("StrictPolicy")]
public async Task<IActionResult> GetCaseById(string Id) { }
```

---

### Q24: Serilog Logging in BSK

```csharp
// Program.cs
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .CreateLogger();
builder.Host.UseSerilog();

// Usage in controllers/repositories
LogHelper.Info("GetCaseById started", ("CaseId", Id));
LogHelper.Error("Payment DB transaction failed", ex);
LogHelper.Warn("3rd party service down", ("ServiceName", ex.ServiceName));

// BSK has separate logging targets:
// - App logs: C:\BSK_Logs\app.json
// - Vendor health: C:\BSK_Logs\ThirdPartyHealth\health-{Date}.json
```

---

## 13. Quick-Fire Checklist

| Question | Short Answer |
|---|---|
| `ControllerBase` vs `Controller` | `ControllerBase` for APIs; `Controller` adds View support |
| `IActionResult` vs `ActionResult<T>` | `ActionResult<T>` is type-safe, supports OpenAPI better |
| `[ApiController]` effects | Auto-400 on bad ModelState, param source inference, RFC 7807 errors |
| `ModelState.IsValid` | Automatically checked with `[ApiController]` — returns 400 if invalid |
| `Ok()`, `NotFound()`, `BadRequest()` | `ControllerBase` helper methods for HTTP responses |
| `HttpContext.Items` | Per-request data storage across middleware |
| `.AsNoTracking()` | EF Core: disables change tracking for read-only queries (faster) |
| `IHostedService` vs `BackgroundService` | `BackgroundService` is an easier wrapper around `IHostedService` |
| `CancellationToken` | Pass to all async DB/HTTP calls to allow graceful cancellation |
| HSTS | HTTP Strict Transport Security — forces HTTPS for future browser requests |

---

## 🏁 BSK Talking Points for .NET Core / Web API Section

> *"In BSK, I worked on a production ASP.NET Core 8 Web API serving claims management for an insurance platform. We used JWT Bearer authentication with role-based access control across all controllers. Our middleware pipeline includes a custom `ThirdPartyExceptionMiddleware` for global error handling, Serilog for structured logging, and Polly for resilient external API calls with retry and circuit-breaker policies. We used `IHttpClientFactory` with named clients to prevent socket exhaustion when calling 7 external vendors including Surepass KYC and Razorpay payments. Background services handled vendor health probing and case assignment queues."*

---
*Next: See [Section_04_EF_Core_ORM.md](./Section_04_EF_Core_ORM.md)*
