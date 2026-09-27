# BSK Enterprise System — Comprehensive Interview Q&A Study Guide

Welcome to the **BSK Enterprise System Study Guide**. This document contains exhaustive, production-grade answers to all key interview questions on **.NET API / Web API**, **.NET Core (Intermediate/Beginner)**, and **Code Quality & Processes**. 

Every answer is tailored to the actual **Bima Sevak Kendra (BSK) API** production codebase, using its architecture, files, and patterns as real-world references.

---

## 📋 Table of Contents

1. [🌐 .NET API / Web API (Questions 1 – 12)](#-net-api--web-api)
2. [⚙️ .NET Core & C# (Questions 13 – 38)](#%EF%B8%8F-net-core--c)
3. [🔁 Code Quality & Process (Questions 39 – 42)](#-code-quality--process)

---

## 🌐 .NET API / Web API

### Q1: HTTP Verbs — GET / POST / PUT / DELETE (BSK Context)
**Question**: What are HTTP verbs, and how are they used in a RESTful API? Provide examples from the BSK project.
**Answer**:
HTTP Verbs (or Methods) specify the action to be performed on a resource. A RESTful API maps these verbs to CRUD (Create, Read, Update, Delete) operations:
*   **`GET`**: Retrieves data from the server. It must be safe and idempotent (making multiple identical requests returns the same data without side effects).
    *   *BSK Reference*: `CaseController.cs` has `[HttpGet("GetCaseById")]` to fetch specific claimant case details:
        ```csharp
        [HttpGet("GetCaseById")]
        public async Task<IActionResult> GetCaseById(string Id) { ... }
        ```
*   **`POST`**: Submits data to the server to create a new resource or initiate a process. It is neither safe nor idempotent.
    *   *BSK Reference*: `CaseController.cs` has `[HttpPost("ConsumerCourtManualPayment")]` to submit a new receipt transaction record.
*   **`PUT`**: Replaces or updates an existing resource entirely. It is idempotent (repeated calls with the same payload yield the same final state).
    *   *BSK Reference*: `CaseController.cs` uses `[HttpPut("UpdateAcceptCaseStatusInCaseDetails")]` to update a case's acceptance status.
*   **`DELETE`**: Deletes or archives a resource. It is idempotent.
    *   *BSK Reference*: `DocumentController.cs` contains `[HttpDelete("DeleteDocument")]` to remove a file's metadata from the system.

---

### Q2: HTTP Response Codes
**Question**: What are HTTP response codes, and how does the BSK project return them to the client?
**Answer**:
HTTP response status codes indicate whether a specific HTTP request was successfully completed. They are categorized into five classes:
1.  **2xx Success**: e.g., `200 OK` (Request succeeded), `201 Created`.
2.  **4xx Client Error**: e.g., `400 BadRequest` (Invalid input), `401 Unauthorized` (Authentication required), `404 NotFound` (Resource doesn't exist), `409 Conflict` (State conflict, e.g., duplicate record).
3.  **5xx Server Error**: e.g., `500 InternalServerError` (Crash), `502 BadGateway` (Third-party failed), `503 ServiceUnavailable` (Overloaded or down).

**BSK Project Implementation**:
BSK wraps all API responses in a standardized class called `BaseResponseStatus`. It leverages C#'s `StatusCodes` helper from the `Microsoft.AspNetCore.Http` namespace.
```csharp
public class BaseResponseStatus
{
    public string StatusCode { get; set; } // Matches HTTP Status as a string
    public string StatusMessage { get; set; } // Human-readable status message
    public object ResponseData { get; set; } // Payload returned to client
    // Additional generic fields: ResponseData1, ResponseData2, etc.
}
```

*Example from `CaseController.cs` (`GetCaseById`)*:
```csharp
if (data == null)
{
    baseResponse.StatusCode = StatusCodes.Status404NotFound.ToString();
    baseResponse.StatusMessage = "Data Not Found";
    return Ok(baseResponse); // Returns 200 OK wrapping a 404 response body, OR direct:
}
```

*Example from `ThirdPartyExceptionMiddleware.cs` (Global response configuration)*:
```csharp
// Returns a direct HTTP 503 Service Unavailable when an external vendor is down
context.Response.StatusCode = (int)HttpStatusCode.ServiceUnavailable;
var response = new BaseResponseStatus { StatusCode = "503", StatusMessage = "Service is unavailable" };
await context.Response.WriteAsync(JsonSerializer.Serialize(response));
```

---

### Q3: API Routing in ASP.NET Core
**Question**: How does routing work in ASP.NET Core APIs? Explain attribute routing and BSK's routing configuration.
**Answer**:
Routing matches incoming HTTP requests to controller action methods. ASP.NET Core supports **Convention-based routing** (usually for MVC apps with views) and **Attribute Routing** (highly preferred for REST APIs).

**Attribute Routing**:
Decorators are applied directly to controllers and actions to map request URLs. BSK relies entirely on attribute routing:
1.  **Controller Level**: `[Route("bsk/api/[controller]")]`
    *   `[controller]` is a token replaced by the controller class name minus the "Controller" suffix. E.g., `CaseController` becomes `bsk/api/Case`.
2.  **Action Level**: Action verbs combine with route extensions.
    *   `[HttpGet("GetCaseById")]` maps to `GET bsk/api/Case/GetCaseById?Id=...`.
3.  **Route Parameters**: Routes can contain dynamic parameters:
    *   `[HttpDelete("DeleteDocument/{id}")]` maps `id` straight from the path.

---

### Q4: Model Binding in API Controller
**Question**: What is Model Binding? How are different sources mapped (Body, Query, Route) in the BSK codebase?
**Answer**:
Model Binding automatically maps data from HTTP requests (headers, route parameters, query strings, request bodies) to action method parameters.

**Binding Sources in ASP.NET Core**:
*   **`[FromBody]`**: Binds complex types from the JSON body of the request.
    *   *BSK Reference (`CaseController.cs`)*:
        ```csharp
        [HttpPost("ConsumerCourtManualPayment")]
        public async Task<IActionResult> ConsumerCourtManualPayment([FromBody] ReceiptsModel createReceipt)
        ```
*   **`[FromQuery]`** (Default for primitive types in `GET`): Extracts values from the URL query string (e.g., `?Id=123`).
    *   *BSK Reference (`CaseController.cs`)*:
        ```csharp
        [HttpGet("GetCaseById")]
        public async Task<IActionResult> GetCaseById(string Id) // Implicitly [FromQuery]
        ```
*   **`[FromRoute]`**: Binds parameter from a route segment match (e.g., `DeleteCase/{Id}`).
*   **`[FromForm]`**: Used to bind files or form-data (e.g., multi-part file uploads).

---

### Q5: Authentication & Authorization in Web API
**Question**: How does BSK implement secure token-based authentication and authorization? Provide config and decorator examples.
**Answer**:
BSK uses **JWT (JSON Web Token) Bearer Authentication** and **Role-Based Access Control (RBAC)**.

1.  **Registration in `Program.cs`**:
    The authentication services are registered, configuring key settings such as token lifetime validation and issuer key signing:
    ```csharp
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
            ClockSkew = TimeSpan.Zero
        };
    });
    ```
2.  **Usage on Controllers (Decorators)**:
    Controllers are decorated with `[Authorize]` to enforce that requests contain a valid, unexpired JWT.
    *   *BSK Reference (`DocumentController.cs`)*:
        ```csharp
        [Route("bsk/api/[controller]")]
        [ApiController]
        [Authorize] // Enforces secure JWT on all actions in this controller
        public class DocumentController : ControllerBase { ... }
        ```
3.  **Role-Based Access**:
    Can be written as `[Authorize(Roles = "Admin,Officer")]` to lock down routes to specific user types defined in BSK's database (Admin, Officer, Agent, Lawyer, Claimant, Partner).

---

### Q6: Use of Decorators/Attributes in API Controllers
**Question**: What are decorators in C#? What are the standard attributes used on a BSK API controller?
**Answer**:
Decorators (technically called **Attributes** in C#) are metadata tags placed above classes, methods, or parameters to modify their runtime behavior without changing the core code logic.

**Standard BSK Controller Decorators**:
*   **`[ApiController]`**: Enables API-specific behaviors like automatic HTTP 400 validation on model state errors, parameter source inference, and RFC 7807 error logs.
*   **`[Route("bsk/api/[controller]")]`**: Specifies controller-level base route.
*   **`[Authorize]`**: Enforces valid authentication tokens.
*   **`[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]`**: Dictate action HTTP verbs.
*   **`[FromBody]`, `[FromQuery]`**: Configure parameter binding sources.

---

### Q7: Use of Action Filters in ASP.NET Core
**Question**: What is an Action Filter, and what scenarios are they used for? Are there references in BSK?
**Answer**:
Action Filters run code **before** and **after** an action method executes. They are ideal for cross-cutting concerns like logging, audit trails, custom authorization checks, or input sanitization.

**Filter Pipeline Lifecycle**:
```
Request → Exception Middleware → Authorization Filters → Action Filters (Before) → Action Executing → Action Filters (After) → Response
```

*BSK Reference*:
We see references to a custom action filter called `[AuthorizeAction]` (commented out on `CaseController.cs` for local testing/legacy transition):
```csharp
// [AuthorizeAction] -- Action filter designed to check custom database-driven PageMenu permissions dynamically.
public class CaseController : ControllerBase { ... }
```
A dynamic action filter typically inherits from `ActionFilterAttribute` or implements `IAsyncActionFilter`:
```csharp
public class CustomAuditFilter : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        // Code before the action runs (e.g., Log inputs)
        var resultContext = await next();
        // Code after the action runs (e.g., Log performance metrics)
    }
}
```

---

### Q8: Global Error Handling (Middleware approach)
**Question**: How does BSK handle uncaught errors globally? Explain how the custom middleware operates.
**Answer**:
Instead of adding try-catch blocks to every single method, BSK uses ASP.NET Core's global middleware pipeline to capture uncaught exceptions and convert them into clean, standardized JSON responses.

*BSK Reference (`ThirdPartyExceptionMiddleware.cs`)*:
```csharp
public class ThirdPartyExceptionMiddleware
{
    private readonly RequestDelegate _next;

    public ThirdPartyExceptionMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context); // Passes control to the next middleware
        }
        catch (ThirdPartyServiceUnavailableException ex)
        {
            // Catches vendor service outages dynamically
            LogHelper.Warn("3rd party service down", ("ServiceName", ex.ServiceName));
            await WriteResponseAsync(context, HttpStatusCode.ServiceUnavailable, "503", ex.UserMessage);
        }
        catch (ThirdPartyApiException ex)
        {
            // Catches when vendor returns a bad response (400, 401, 500)
            LogHelper.Error("3rd party vendor error response", ex);
            await WriteResponseAsync(context, HttpStatusCode.BadGateway, "502", ex.UserMessage);
        }
    }
}
```
**Benefits**:
*   **Encapsulation**: Keeps controllers free of repetitive error-handling boilerplate.
*   **Security**: Prevents leakage of internal database or server stack traces to clients.
*   **Consistency**: Standardizes the response format via `BaseResponseStatus`.

---

### Q9: Logging of Requests and Responses
**Question**: How does BSK configure and perform application and health logging?
**Answer**:
BSK implements structured logging using **Serilog**. Logging is set up in `Program.cs` and loads configuration dynamically from `appsettings.json`.

1.  **Configuration in `Program.cs`**:
    ```csharp
    Log.Logger = new LoggerConfiguration()
        .ReadFrom.Configuration(builder.Configuration)
        .CreateLogger();
    builder.Host.UseSerilog(); // Directs ASP.NET Core log streams to Serilog
    ```
2.  **Separate Audits & Destinations**:
    BSK establishes distinct logging targets for specialized systems to optimize parsing:
    *   **Application Event Logs**: Saved structured logs to disk (e.g., `C:\BSK_Logs\app.json`). Managed by helper utility `LogHelper`.
    *   **Vendor Outage / Health Logs**: Logs external vendor probes specifically to `C:\BSK_Logs\ThirdPartyHealth\health-{Date}.json` via `HealthCheckLogger`.
3.  **Code-level Usage in Controller**:
    ```csharp
    LogHelper.Info("GetCaseById started", ("CaseId", Id));
    ```

---

### Q10: API Versioning
**Question**: What is API versioning, and how can it be implemented in an ASP.NET Core API?
**Answer**:
API versioning allows developers to evolve the API contract (URI structure, payloads, behaviors) without breaking existing client integrations (mobile apps, partners).

**Common Implementation Approaches**:
1.  **URL Path Versioning** (Most Popular):
    ```
    bsk/api/v1/Case/GetCaseById
    bsk/api/v2/Case/GetCaseById
    ```
    *How to configure in Controller*: `[Route("bsk/api/v{version:apiVersion}/[controller]")]`
2.  **Query Parameter Versioning**: `bsk/api/Case/GetCaseById?api-version=1.0`
3.  **HTTP Header Versioning**: Header key `X-API-Version: 1.0`

**Implementing in .NET Core**:
Using NuGet package `Asp.Versioning.Http`:
```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // Returns supported versions in response headers
});
```

---

### Q11: Calling a Backend/External API
**Question**: If we want to call an external third-party API (e.g., Surepass, SMS gateway), what is the best-practice syntax and parameters in .NET Core? How does BSK do this?
**Answer**:
In modern .NET Core, external API calls must be managed through `IHttpClientFactory` rather than instantiating a raw `new HttpClient()` (which causes Socket Exhaustion under load).

**BSK Implementation**:
BSK defines **Named & Resilient HttpClients** in `Program.cs` and hooks them up with **Polly resilience pipelines** for retries, circuit breakers, and custom timeouts:
```csharp
builder.Services.AddHttpClient("SurepassClient", client => 
    client.BaseAddress = new Uri("https://kyc-api.surepass.io"))
    .AddResilienceHandler(PollyPolicies.SurepassPipeline, PollyPolicies.ConfigureHttpPipeline);
```

**Injecting and Calling in Repository (`DocumetVerificationRepository.cs`)**:
```csharp
private readonly IHttpClientFactory _clientFactory;
// Inject IHttpClientFactory in constructor...

public async Task<bool> VerifyPanWithSurepass(string panNumber)
{
    var client = _clientFactory.CreateClient("SurepassClient"); // Reuses socket connection pooled by the OS
    
    var requestPayload = new { pan = panNumber };
    var jsonContent = new StringContent(JsonSerializer.Serialize(requestPayload), Encoding.UTF8, "application/json");

    // Call the external API asynchronously
    var response = await client.PostAsync("api/v1/pan-verification", jsonContent);
    
    if (response.IsSuccessStatusCode)
    {
        var responseString = await response.Content.ReadAsStringAsync();
        // Parse and return result...
        return true;
    }
    return false;
}
```

---

### Q12: Steps to Create a Web API in your Application
**Question**: Walk through the architectural steps to create a new Web API endpoint in the BSK application.
**Answer**:
To build an endpoint (e.g., `/bsk/api/Case/GetCaseById`) in BSK, we follow a clean 4-tier structural workflow:

```
[1] Database / EF Entity ──► [2] Async Repository (Data Access) ──► [3] Service / Business Layer ──► [4] Controller Endpoint
```

1.  **Define the Domain Model/Entity**:
    Ensure the table and mapping exist in `DataModel/Case.cs` and register it inside `Data/ApplicationDBContext.cs`.
2.  **Create/Modify the Repository Interface & Implementation**:
    *   Interface: `Repository/Interface/ICaseAsyncRepository.cs`
        ```csharp
        Task<CaseModel> GetCaseById(string id);
        ```
    *   Concrete Class: `Repository/CaseAsyncRepository.cs` (utilizes Entity Framework or Dapper to fetch from SQL Server).
3.  **Register dependencies in `Program.cs`**:
    ```csharp
    builder.Services.AddScoped<ICaseAsyncRepository, CaseAsyncRepository>();
    ```
4.  **Create the API Controller Action**:
    Decorate a controller with attributes, inject the repository via constructor, and write the async action method:
    ```csharp
    [ApiController]
    [Route("bsk/api/[controller]")]
    public class CaseController : ControllerBase
    {
        private readonly ICaseAsyncRepository _repo;
        public CaseController(ICaseAsyncRepository repo) => _repo = repo;

        [HttpGet("GetCaseById")]
        public async Task<IActionResult> GetCaseById(string Id)
        {
            var data = await _repo.GetCaseById(Id);
            return Ok(new BaseResponseStatus { StatusCode = "200", ResponseData = data });
        }
    }
    ```

---

## ⚙️ .NET Core & C#

### Q13: Sealed Class
**Question**: What is a sealed class in C#? When and why should we use it?
**Answer**:
A `sealed` class is a class that **cannot be inherited** by other classes.
*   **Syntax**:
    ```csharp
    public sealed class TokenHelper
    {
        // Methods and properties
    }
    ```

**When to Use**:
1.  **Security & Domain Logic Integrity**: When creating core utilities (like encryption helpers, hashing functions, or license managers) that should never be altered or overridden by subclasses.
2.  **Performance Optimization**: 
    *   When the JIT (Just-In-Time) compiler encounters a `sealed` class, it knows there are no derived versions. It can resolve virtual method calls directly (via static dispatch) instead of checking the virtual method table (vtable). This process, known as **devirtualization**, results in minor performance gains and enables code inlining.

---

### Q14: .NET Framework vs .NET Core
**Question**: Contrast standard legacy .NET Framework with modern .NET Core (current .NET 8.0).
**Answer**:

| Feature | .NET Framework (Legacy) | .NET Core / .NET 8.0 (Modern) |
|---|---|---|
| **Cross-Platform** | Windows only. | Windows, Linux, macOS. (Ideal for Docker). |
| **Performance** | Good, but legacy overhead. | Engineered for high-throughput (TechEmpower benchmark leader). |
| **Startup Entry** | Controlled by `Global.asax` and Web.config. | Unified `Program.cs` with lightweight middleware pipeline. |
| **Dependency Injection** | Requires external libraries (Autofac, Unity). | Native, built-in DI engine out-of-the-box. |
| **Deployment Model** | Machine-wide GAC installation. | Self-contained, side-by-side or Dockerized deployment. |

---

### Q15: Why .NET Core? Advantages and Value
**Question**: Why do modern enterprise applications choose .NET Core (like BSK choosing ASP.NET Core 8)?
**Answer**:
1.  **Unmatched Throughput**: Core handles thousands of concurrent claims and heavy background processes (like the circular case assignment hosted service) with minimal CPU/Memory footprints.
2.  **Cross-Platform Deployments**: Allows BSK to run on Linux containers, significantly lowering server licensing overheads compared to old Windows Server models.
3.  **Modern Middleware Pipeline**: Purely opt-in system; we only pay performance overhead for the middlewares we actually register (CORS, Auth, custom exceptions).
4.  **Native Async Stack**: Deeply integrated async programming model throughout (controllers down to SQL driver), freeing CPU threads during heavy DB or REST API network calls.

---

### Q16: Dependency Injection Lifetimes in .NET Core
**Question**: Explain Transient, Scoped, and Singleton lifetimes in .NET Core. Provide real references from BSK registrations.
**Answer**:
The service lifetime defines how long a resolved object instance survives before the DI container disposes of it:

```
[Transient] Request Resolve ──► New Instance Every Time
[Scoped]    Request Resolve ──► New Instance per HTTP Request Pipeline
[Singleton] Request Resolve ──► One Instance Shared Across All Users / Lifetimes
```

1.  **Transient (`AddTransient`)**:
    *   Created **every time they are requested** from the DI container. Best for lightweight, stateless services.
2.  **Scoped (`AddScoped`)**:
    *   Created **once per HTTP request pipeline**. The same instance is shared across all classes called during a single request (e.g., Controller, Service, and Repository). Highly recommended for database context operations.
    *   *BSK Reference*:
        ```csharp
        builder.Services.AddDbContext<ApplicationDBContext>(...); // Scoped by default
        builder.Services.AddScoped<ICaseAsyncRepository, CaseAsyncRepository>();
        builder.Services.AddScoped<ConnectionHandler>();
        ```
3.  **Singleton (`AddSingleton`)**:
    *   Created **once on application startup** and shared throughout the entire app lifespan across all users.
    *   *BSK Reference*:
        ```csharp
        builder.Services.AddSingleton<ISqlCacheService, SqlCacheService>();
        // Vendor health registry must be a Singleton to keep thread-safe UP/DOWN states for all users:
        builder.Services.AddSingleton<IServiceHealthRegistry, ServiceHealthRegistry>();
        ```

---

### Q17: Generics in C#
**Question**: What are Generics in C#, and how do they benefit applications?
**Answer**:
Generics allow you to write classes, interfaces, or methods with placeholders (type parameters) for the type of data they store or manipulate, deferring the type specification until execution.

*C# Example*:
```csharp
public class Repository<T> where T : class
{
    private readonly DbContext _context;
    public Repository(DbContext context) => _context = context;

    public async Task AddAsync(T entity) => await _context.Set<T>().AddAsync(entity);
}
```
**Core Advantages**:
1.  **Type Safety**: Catch data mapping issues at compile time rather than runtime.
2.  **No Boxing/Unboxing**: Operations on value types don't require conversion to `object` memory heap locations, protecting garbage collection memory throughput.
3.  **Code Reusability**: Prevents writing repetitive repositories like `ClaimantRepository`, `DocumentRepository`, etc.

---

### Q18 & Q19: Exception Handling in C# & ASP.NET Core
**Question**: How does exception handling operate in C#? What are the standard blocks, and how is this applied at scale?
**Answer**:
In C#, exceptions are handled using `try`, `catch`, and `finally` blocks:
*   **`try`**: Wraps the code execution block that might fail.
*   **`catch`**: Handles specific exceptions if they occur.
*   **`finally`**: Executes cleanup logic (e.g., closing database connections, disposing of file streams) regardless of whether an exception was thrown.

**Production-grade architectural pattern**:
Instead of writing heavy try-catch blocks in every controller method (which bloats code and is hard to maintain), modern ASP.NET Core apps leverage **Global Middleware Exception Handling** (see Q8).

*Local Try-Catch in Repository (e.g., for database transaction rollbacks)*:
```csharp
public async Task<bool> SaveTransactionAsync(PaymentModel model)
{
    using var transaction = await _dbContext.Database.BeginTransactionAsync();
    try
    {
        _dbContext.Payments.Add(model);
        await _dbContext.SaveChangesAsync();
        await transaction.CommitAsync();
        return true;
    }
    catch (DbUpdateException ex)
    {
        await transaction.RollbackAsync(); // Roll back DB state
        LogHelper.Error("Payment DB transaction failed", ex);
        throw; // Rethrow to global exception middleware
    }
}
```

---

### Q20: Exception Filters
**Question**: Have you used Exception Filters in C# or ASP.NET Core? How do they work?
**Answer**:
There are two types of Exception Filters in the .NET ecosystem:
1.  **C# Catch Exception Filters (using the `when` clause)**:
    Introduced in C# 6, this lets you catch an exception only if a certain condition is met. This is highly performant because it evaluates the condition before unwinding the call stack.
    ```csharp
    try
    {
        await _repository.CallSurepassApi();
    }
    catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.Unauthorized)
    {
        // Handles unauthorized cases dynamically (e.g., refresh token)
    }
    ```
2.  **ASP.NET Core `IExceptionFilter` / `ExceptionFilterAttribute`**:
    Filters that run when an unhandled exception occurs during controller action execution.
    ```csharp
    public class CustomExceptionFilter : IExceptionFilter
    {
        public void OnException(ExceptionContext context)
        {
            // Log and construct custom response...
            context.Result = new ObjectResult(new { Error = context.Exception.Message });
        }
    }
    ```
    *Comparison*: While Exception Filters are good, **Global Middleware Exception Handling** is preferred for APIs because it captures errors happening outside of controllers (e.g., in Routing or Middleware).

---

### Q21 & Q22: Is it a Good Practice to use Try/Catch?
**Question**: Is it always a good practice to use try/catch blocks? What are the key guidelines?
**Answer**:
Using try-catch is essential, but **misusing or overusing it is a bad practice**. 

**When it is BAD**:
1.  **Empty / Silent Catches**: Catching an exception and doing nothing (`catch(Exception ex) {}`). This swallows the error, making bugs impossible to diagnose.
2.  **Wrapping Everything**: Placing `try-catch` blocks around every simple, safe block of code. It adds code noise and hides architectural bugs that *should* fail fast.
3.  **Using Exceptions for Control Flow**: E.g., throwing a `FormatValidationException` to tell a user their email format is wrong. Checking with an `if` condition is much faster; throwing exceptions carries significant stack trace generation overhead.

**When it is GOOD**:
1.  **Boundary Calls**: Around unmanaged resources, database transactions, network HTTP integrations, or file I/O operations where unexpected external states can occur.
2.  **Global Level Catch**: Centralizing uncaught exceptions inside a middleware to log details, audit, and return a clean HTTP 500 status code.

---

### Q23 & Q24: Synchronous vs Asynchronous Programming in ASP.NET Core
**Question**: Contrast Synchronous and Asynchronous programming models. Why is `async/await` critical for BSK?
**Answer**:
*   **Synchronous**: A calling thread starts a task (like calling SQL to get a case details) and sits idle, **blocked**, waiting for SQL to return data before continuing.
*   **Asynchronous**: A calling thread initiates a task, registers a callback, and is **freed up immediately** to handle other incoming HTTP requests. Once SQL returns the data, the thread pool assigns *any* available thread to resume the method execution.

```
Synchronous:  [Thread 1] ────► [DB Query (Blocked Waiting)] ────► [Resume Response]  (Scalability limit: Max threads)
Asynchronous: [Thread 1] ──► [Initiate DB Query] ── (Thread 1 Freed to serve other users) ──► [Thread 2 resumes on completion]
```

**Why it is critical for BSK**:
BSK needs to serve many concurrent claims and handle background health checks simultaneously. If it used synchronous DB calls, the IIS/Kestrel Thread Pool would easily hit **Thread Starvation** under high load. Using `async/await` allows the API to serve thousands of requests with a small, lightweight pool of OS threads.

*BSK Reference*:
```csharp
[HttpGet("GetCaseById")]
public async Task<IActionResult> GetCaseById(string Id)
{
    // Thread is immediately released back to thread pool during EF query execution
    var data = await _repository.GetCaseById(Id); 
    return Ok(data);
}
```

---

### Q25: .NET Core Middleware Pipeline
**Question**: What is .NET Core Middleware? How are they ordered, and what happens to a request passing through?
**Answer**:
Middleware are software components assembled into an application pipeline to handle requests and responses. Each middleware component can either process a request and pass it to the next component (`_next`) or short-circuit the pipeline (e.g., if authorization fails).

**BSK Pipeline Registration (`Program.cs`)**:
```csharp
// 1. Custom Exception Middleware (wraps everything to catch errors downstream)
app.UseMiddleware<ThirdPartyExceptionMiddleware>();

app.UseHttpsRedirection();
app.UseCors("CorsPolicy");

// 2. Authentication must run before Authorization
app.UseAuthentication(); 
app.UseAuthorization();

// 3. Routing resolves the controller endpoint
app.MapControllers();
```
**Ordering Rule**: Ordering is critical because execution flows down the pipeline, then back up. If `UseAuthorization` ran before `UseAuthentication`, the system would not know who the user is when checking permissions, causing it to fail.

---

### Q26: Improving Performance Using Middleware
**Question**: Can we improve web application performance using middleware? Provide concepts and code references.
**Answer**:
Yes! Middleware is highly effective for global performance optimizations because it executes before requests hit the business layers.

**Key Optimization Middlewares**:
1.  **Response Compression Middleware**: Compresses JSON responses (Gzip/Brotli) to reduce network payload sizes.
    ```csharp
    builder.Services.AddResponseCompression(options => { options.EnableForHttps = true; });
    // In pipeline:
    app.UseResponseCompression();
    ```
2.  **Response Caching Middleware**: Automatically caches responses based on cache headers to avoid hitting database queries repeatedly.
    ```csharp
    builder.Services.AddResponseCaching();
    // In pipeline:
    app.UseResponseCaching();
    ```
3.  **Rate Limiting Middleware**: Protects resources from brute-force or high-frequency API abuse.
    ```csharp
    app.UseRateLimiter();
    ```

---

### Q27, Q28 & Q29: Dependency Injection Deep Dive
**Question**: What is DI? How does it differ between legacy .NET Framework and .NET Core? What changes did .NET Core bring?
**Answer**:
**Dependency Injection (DI)** is a software design pattern where objects do not instantiate their dependencies directly; instead, dependencies are provided (injected) by an external container (IoC).

1.  **Framework vs .NET Core DI**:
    *   **Legacy Framework**: Had no built-in DI system. Developers had to manually import, configure, and manage third-party containers like Autofac, Ninject, or Unity in complex config XML files or `Global.asax`.
    *   **Modern .NET Core**: Standardizes DI natively. The runtime exposes `IServiceCollection` straight in `Program.cs`. Third-party DI containers are no longer necessary for standard projects.
2.  **Registration and Migration Code Changes**:
    *   *Legacy (Global.asax)*:
        ```csharp
        // Legacy Unity mapping
        var container = new UnityContainer();
        container.RegisterType<ICaseAsyncRepository, CaseAsyncRepository>();
        DependencyResolver.SetResolver(new UnityDependencyResolver(container));
        ```
    *   *Modern (`Program.cs`)*:
        ```csharp
        // Clean C# builder syntax
        builder.Services.AddScoped<ICaseAsyncRepository, CaseAsyncRepository>();
        ```

---

### Q30: LINQ and Left Join in C#
**Question**: What is LINQ? How do you write a Left Join in LINQ? Provide a code example.
**Answer**:
**LINQ (Language Integrated Query)** allows developers to write type-safe queries on data collections (in-memory arrays, XML, SQL databases via EF) directly in C#.

**LINQ Left Join**:
To perform a Left Join in LINQ, we use a combination of `join`, `into`, and the `DefaultIfEmpty()` method. This ensures that records from the "left" table are included even if there is no matching record in the "right" table.

*Example code (Left Joining Cases with Invoices)*:
```csharp
public async Task<List<CaseInvoiceDto>> GetCasesWithInvoicesLeftJoin()
{
    var cases = await _dbContext.Cases.ToListAsync();
    var invoices = await _dbContext.Invoices.ToListAsync();

    var query = from c in cases
                join i in invoices on c.Id equals i.CaseId into joinedGroup
                from inv in joinedGroup.DefaultIfEmpty() // Left Join logic
                select new CaseInvoiceDto
                {
                    CaseId = c.Id,
                    ClaimantName = c.ClaimantName,
                    InvoiceNumber = inv != null ? inv.InvoiceNumber : "N/A", // Handled if null
                    InvoiceAmount = inv != null ? inv.Amount : 0
                };

    return query.ToList();
}
```

---

### Q31: Content Technology — jQuery
**Question**: What is jQuery? How does a frontend interface use it to interact with modern .NET APIs?
**Answer**:
**jQuery** is a fast, small, and feature-rich JavaScript library designed to simplify HTML DOM traversal, event handling, animations, and AJAX calls across different browsers.

In web applications, jQuery is often used to consume REST APIs dynamically without triggering a full page reload:

*AJAX Script Example connecting to BSK's `GetCaseById` endpoint*:
```javascript
function loadCaseDetails(caseId) {
    $.ajax({
        url: 'https://yourdomain.com/bsk/api/Case/GetCaseById',
        type: 'GET',
        dataType: 'json',
        data: { Id: caseId },
        headers: {
            'Authorization': 'Bearer ' + localStorage.getItem('jwtToken') // Send JWT
        },
        success: function(response) {
            if (response.statusCode === "200") {
                var caseData = response.responseData;
                $('#claimantName').text(caseData.claimantName);
                $('#caseStatus').text(caseData.statusMessage);
            } else {
                alert('Error: ' + response.statusMessage);
            }
        },
        error: function(xhr, status, error) {
            console.error('API call failed: ', error);
        }
    });
}
```

---

### Q32 & Q33: SOLID Principles & Dependency Inversion Example
**Question**: Explain the SOLID principles. Provide a real-world code example for the Dependency Inversion Principle using BSK.
**Answer**:
The SOLID principles are five design guidelines for writing clean, maintainable, and extensible software:
1.  **S (Single Responsibility)**: A class should have only one reason to change.
2.  **O (Open/Closed)**: Software entities should be open for extension but closed for modification.
3.  **L (Liskov Substitution)**: Subtypes must be substitutable for their base types without breaking the app.
4.  **I (Interface Segregation)**: Clients should not be forced to depend on interfaces they do not use.
5.  **D (Dependency Inversion)**: High-level modules should not depend on low-level modules; both should depend on abstractions (interfaces).

**Dependency Inversion Code Example (BSK Structure)**:

*❌ Bad Implementation (High coupling - high-level controller creates low-level repository)*:
```csharp
public class CaseController : ControllerBase
{
    private readonly CaseAsyncRepository _repository; // Tight Coupling

    public CaseController()
    {
        // Tight coupling: If CaseAsyncRepository changes its constructor, CaseController breaks!
        _repository = new CaseAsyncRepository(new ApplicationDBContext()); 
    }
}
```

*✅ Good Implementation (BSK's current design - depends on Interface abstraction)*:
```csharp
// Abstraction Layer
public interface ICaseAsyncRepository
{
    Task<object> GetCaseById(string Id);
}

// Low-Level Implementation Detail
public class CaseAsyncRepository : ICaseAsyncRepository
{
    private readonly ApplicationDBContext _context;
    public CaseAsyncRepository(ApplicationDBContext context) => _context = context;

    public async Task<object> GetCaseById(string Id) => await _context.Cases.FindAsync(Id);
}

// High-Level Module (Depends ONLY on abstraction ICaseAsyncRepository)
public class CaseController : ControllerBase
{
    private readonly ICaseAsyncRepository _repository; // Inverted Dependency

    public CaseController(ICaseAsyncRepository repository) // DI resolves this dynamically
    {
        _repository = repository; 
    }
}
```

---

### Q34: Microservices Architecture
**Question**: What are Microservices? How does BSK interact with other services?
**Answer**:
**Microservices** is an architectural style that structures an application as a collection of small, autonomous, and loosely coupled services, each responsible for a single business capability.

*BSK Interaction with other Services*:
While BSK API runs as a monolithic REST API managing claims and documents, it relies on a **Service-Oriented integration pattern** to talk to other microservices in the company ecosystem.
For example, BSK communicates with a separate dedicated **Accounting Service** using a typed HTTP client with Polly resilience policies:
```csharp
builder.Services.AddHttpClient("AccountingServiceClient", client => 
    client.BaseAddress = new Uri("https://bsk-stg-accountingsvc.shauryatechnosoft.com"))
    .AddResilienceHandler(PollyPolicies.AccountingServicePipeline, PollyPolicies.ConfigureHttpPipeline);
```
This isolates the financial ledger and voucher creation transactions into a dedicated network service, ensuring high availability and separation of concerns.

---

### Q35: Repository Design Pattern
**Question**: Explain the Repository Pattern and its benefits. How does BSK implement it?
**Answer**:
The **Repository Pattern** acts as an abstraction layer between the database (Entity Framework Core) and the business logic layers (Controllers/Services). It encapsulates query logic and data persistence behind standard collection-like methods.

**BSK Implementation**:
BSK separates repository interfaces from database queries, which are then injected into controllers:
*   Interface: `IRoleAsyncRepository.cs`
*   Implementation: `RoleAsyncRepository.cs`

**Core Benefits**:
1.  **Decoupling**: Business controllers don't need to know whether the data is fetched from SQL Server, memory cache, or an external API.
2.  **Testability**: Enables mock implementations (e.g., using `Moq` libraries) of interfaces like `ICaseAsyncRepository` for unit tests, removing the need to connect to a live database.
3.  **Centralized Query Management**: Changes to queries or performance tuning (like adding `.AsNoTracking()`) happen in the repository without affecting the controllers.

---

### Q36: Supporting Libraries in both .NET Framework and .NET Core
**Question**: How can a library be written to support both legacy .NET Framework and modern .NET Core?
**Answer**:
To support both platforms simultaneously, you use **.NET Standard** (usually `.NET Standard 2.0`) as the library target, or leverage **Multi-Targeting** in the `.csproj` file.

1.  **Multi-Targeting in `.csproj`**:
    Configure the project file to compile separate assemblies for different framework targets:
    ```xml
    <Project Sdk="Microsoft.NET.Sdk">
      <PropertyGroup>
        <TargetFrameworks>net48;net8.0</TargetFrameworks> <!-- Target both Framework 4.8 and Core 8 -->
      </PropertyGroup>
    </Project>
    ```
2.  **Conditional Compilation**:
    Use preprocessor directives inside the C# code to handle platform-specific API calls:
    ```csharp
    public string GetPlatformDetail()
    {
        #if NETFRAMEWORK
            return "Running on legacy Windows .NET Framework: " + AppDomain.CurrentDomain.FriendlyName;
        #elif NET8_0
            return "Running on modern cross-platform .NET 8: " + Environment.ProcessPath;
        #endif
    }
    ```

---

### Q37: Architecture of the BSK Application
**Question**: Outline the structural layers of the BSK API application.
**Answer**:
The BSK Enterprise System follows a production-hardened **N-Tier Clean Architecture** model:

```
[1] Presentation Layer (Controllers)
           │
           ▼
[2] Business Logic / Infrastructure Layer (Polly, Logging, Hosted Services)
           │
           ▼
[3] Data Access Layer (Repositories, Connection Handler, DbContext)
           │
           ▼
[4] Database (SQL Server 2022)
```

1.  **Presentation (Controllers)**: Orchestrates request mapping, input parameter bindings, validation status, and handles client responses.
2.  **Infrastructure (Services)**: Outsources tasks such as Serilog configuration, AWS S3 storage adapters, and external vendor communication.
3.  **Data Access (Repositories & EF)**: Manages Entity Framework queries and connection pools.
4.  **Persistent Storage**: SQL Server database.

---

### Q38: Technical Implementation of a Key BSK Feature
**Question**: Walk through the technical implementation details of a key feature in BSK (e.g., Third-Party Health Monitoring).
**Answer**:
BSK implements a robust **Third-Party Health Monitor Service** as a background worker (`IHostedService`) to proactively check the health status of external vendors.

**Technical Architecture Flow**:
```
Program.cs registers HostedService (ThirdPartyHealthMonitorService)
                            │
                            ▼
           Starts a thread-safe PeriodicTimer (every 60s)
                            │
                            ▼
           Probes health URLs of external vendor services (e.g., Surepass API)
                            │
                            ▼
           Updates the thread-safe singleton IServiceHealthRegistry
                            │
                            ▼
           Controllers check status via IServiceHealthGuard before making vendor calls
```

1.  **Registration in `Program.cs`**:
    ```csharp
    builder.Services.AddSingleton<IServiceHealthRegistry, ServiceHealthRegistry>();
    builder.Services.AddScoped<IServiceHealthGuard, ServiceHealthGuard>();
    builder.Services.AddHostedService<ThirdPartyHealthMonitorService>();
    ```
2.  **Probing in `ThirdPartyHealthMonitorService.cs`**:
    Runs an asynchronous background thread that checks external HTTP endpoints. If a probe fails, it flags the service as `DOWN` in the registry and logs a warning.
3.  **Enforcement via `ServiceHealthGuard`**:
    Before executing a transaction (such as PAN verification), repositories check the guard:
    ```csharp
    _healthGuard.EnsureServiceIsUp(ThirdPartyService.Surepass); // Throws exception immediately if DOWN
    ```
    This prevents the application from locking up or wasting resources on connection timeouts when a known outage exists.

---

## 🔁 Code Quality & Process

### Q39: Try/Catch Best Practices
**Question**: Summarize best practices for using try-catch blocks in modern C# code.
**Answer**:
1.  **Do not swallow exceptions**: Always log the stack trace or rethrow (`throw;` instead of `throw ex;` which resets the stack trace).
2.  **Catch specific exceptions first**: Catch specific exceptions (e.g., `SqlException`) before catching the generic `Exception` base class.
3.  **Leverage global error handling**: Use middleware to capture unhandled errors rather than wrapping every method in repetitive try-catch blocks.
4.  **Use exception filters**: Use catch filters (`catch (Exception ex) when (...)`) to handle exceptions conditionally.

---

### Q40 & Q41: Code Review & Pull Request (PR) Processes
**Question**: What are the key elements of a professional Code Review and Pull Request (PR) review?
**Answer**:
A robust Code Review process ensures code quality, security, and architectural consistency.

**PR Review Checklist**:
1.  **Functional Compliance**: Does the code fulfill the requirement?
2.  **Code Readability & Standards**: Are variables appropriately named, methods concise, and coding standards met?
3.  **Asynchronous Patterns**: Are async calls properly awaited without blocking thread calls?
4.  **Security Checks**:
    *   No hardcoded credentials or API keys.
    *   Input data is validated before processing.
5.  **Performance Considerations**:
    *   Queries use `.AsNoTracking()` where possible to optimize memory usage.
    *   Efficient database join operations.
6.  **Test Coverage**: Are unit or integration tests provided?

---

### Q42: Interface vs Abstract Class
**Question**: What is the difference between an Interface and an Abstract Class? When should you use each in C#?
**Answer**:

| Feature | Interface | Abstract Class |
|---|---|---|
| **Multiple Inheritance** | A class can implement **multiple interfaces**. | A class can inherit from **only one abstract class**. |
| **State (Fields)** | Cannot contain instance fields (stateless). | Can contain instance fields, constructors, and member state. |
| **Implementation** | Primarily defines the contract. | Can contain fully implemented concrete methods. |
| **Default Modifiers** | Historically `public`. | Members can use any access modifier (`protected`, `private`, etc.). |

**Guidelines**:
*   **Use an Interface** to define a standard contract across unrelated classes (e.g., `IDisposable` or `ILogger`).
*   **Use an Abstract Class** when creating a family of closely related classes that share common base logic and properties (e.g., a base `Vehicle` class with derived `Car` and `Truck` implementations).

---

## 🚀 Advanced Supplemental Q&As (From Your Codebase)

### Q43: Background Tasks using IHostedService and BackgroundService
**Question**: How do you implement background tasks or long-running worker services in ASP.NET Core? How does BSK use them?
**Answer**:
In ASP.NET Core, long-running background processes are implemented by inheriting from `BackgroundService` (a convenient wrapper implementing the `IHostedService` interface). They execute on separate threads without blocking the primary HTTP request-response thread pool.

**BSK Production Implementation**:
BSK uses two background services: `ThirdPartyHealthMonitorService` (for probing vendor endpoints) and `CircularListInitializerHostedService` (for background queues).
```csharp
public class ThirdPartyHealthMonitorService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<ThirdPartyHealthMonitorService> _logger;

    public ThirdPartyHealthMonitorService(IServiceProvider serviceProvider, ILogger<ThirdPartyHealthMonitorService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Executes periodically in the background as long as the app is running
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(60));
        while (!stoppingToken.IsCancellationRequested && await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                using var scope = _serviceProvider.CreateScope();
                var healthChecker = scope.ServiceProvider.GetRequiredService<IVendorHealthChecker>();
                await healthChecker.ProbeAllVendorsAsync();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Background health probe cycle failed.");
            }
        }
    }
}
```

**Key Architectural Rule**:
Background services are registered as **Singletons**. If they need to access a **Scoped** service (like `ApplicationDBContext` or a repository), they **cannot** inject it directly via the constructor (which causes a *dependency capture anomaly*). Instead, they must inject the `IServiceProvider` (as shown above) and create a temporary scoped context (`CreateScope()`) inside their execution block.

---

### Q44: Distributed Caching (`IDistributedCache` with SQL Server)
**Question**: What is Distributed Caching? How does it differ from In-Memory caching, and how is it configured in BSK?
**Answer**:
*   **In-Memory Cache (`IMemoryCache`)**: Stores cache items directly in the web server's RAM. It is extremely fast, but it is bound to that single server instance. In a load-balanced multi-server (web farm) environment, Server A won't have access to Server B's cache, leading to data inconsistency.
*   **Distributed Cache (`IDistributedCache`)**: Stores cache items in a centralized external server (such as Redis or a dedicated SQL Server table). All instances of the web app read/write to this shared cache.

**BSK Implementation**:
BSK uses a **SQL Server Distributed Cache** to manage JWT token blacklists and active sessions. It is registered in `Program.cs`:
```csharp
builder.Services.AddDistributedSqlServerCache(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("DefaultConnection");
    options.SchemaName = "dbo";
    options.TableName = "TokensCache";
});
```

**Usage inside code**:
```csharp
private readonly IDistributedCache _cache;
// Inject IDistributedCache...

public async Task CacheUserSession(string tokenId, string userDataJson)
{
    var options = new DistributedCacheEntryOptions()
        .SetAbsoluteExpiration(TimeSpan.FromHours(2));
        
    await _cache.SetStringAsync(tokenId, userDataJson, options);
}
```

---

### Q45: Micro-ORM (Dapper) vs. Object-Relational Mapper (EF Core)
**Question**: When do you choose Dapper over Entity Framework Core? Can they coexist in the same application?
**Answer**:
Yes, Dapper (a micro-ORM) and Entity Framework Core (a full-featured ORM) can co-exist perfectly in the same project, and BSK leverages both depending on the performance requirements:

| Feature | Entity Framework Core | Dapper |
|---|---|---|
| **Design** | Full-featured ORM with unit-of-work tracking and linq-to-sql generation. | Lightweight, high-performance extension methods on standard ADO.NET connections. |
| **Performance** | Good, but carries tracking and change detection overhead. | Extremely fast (close to raw ADO.NET speeds). |
| **SQL Control** | Abstracted. Generates queries automatically. | Explicit. You write raw, highly optimized SQL queries manually. |
| **Use Case** | CRUD operations, transactional state modifications. | Complex reporting, highly optimized search dashboards. |

**BSK Coexistence Strategy**:
*   **EF Core** is used for normal claims processing, document operations, and status changes, taking advantage of automatic transactions, entity relationships, and robust change tracking.
*   **Dapper** (via `QueryDbHandle`) is used for performance-critical queries like **`Q000_ExecutiveDashboard`** where highly nested queries, window functions (`ROW_NUMBER()`), and `UNION ALL` statements must be mapped directly to DTO structures with zero tracking overhead.

---

### Q46: Resiliency and Fault Tolerance (Polly Policies)
**Question**: What is the Circuit Breaker pattern, and how does Polly enhance the stability of external integration calls in ASP.NET Core?
**Answer**:
When calling external APIs (e.g., payment gateways or KYC verification engines), a transient network hiccup or temporary vendor downtime should not crash your application. **Polly** is a .NET resilience library that intercepts outgoing HTTP calls to execute custom retry patterns.

**BSK Implementation**:
BSK configures resilient policies in `Program.cs` that are bound to outgoing HTTP client factory pools:
1.  **Retry Policy**: Automatically retries a failed request (e.g., HTTP 5xx or network timeout) up to 3 times with **Exponential Backoff** (waiting 2s, then 4s, then 8s) to avoid overwhelming the target server.
2.  **Circuit Breaker Policy**: If the target server fails repeatedly (e.g., 5 times consecutively), Polly **trips the circuit**. For the next 30 seconds, all outgoing calls fail instantly with an exception *without* wasting resources attempting connection requests. This gives the external server time to recover.

```csharp
// Example custom policy configuration:
public static class PollyPolicies
{
    public static readonly string SurepassPipeline = "SurepassPipeline";

    public static void ConfigureHttpPipeline(ResiliencePipelineBuilder<HttpResponseMessage> builder)
    {
        builder
            .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
            {
                MaxRetryAttempts = 3,
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true
            })
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
            {
                FailureRatio = 0.5, // Trip if 50% of requests fail in a window
                SamplingDuration = TimeSpan.FromSeconds(30),
                MinimumThroughput = 8,
                BreakDuration = TimeSpan.FromSeconds(15)
            });
    }
}
```

---

### Q47: CORS (Cross-Origin Resource Sharing)
**Question**: What is CORS, and how do you configure it securely in an ASP.NET Core API?
**Answer**:
**CORS (Cross-Origin Resource Sharing)** is a browser security mechanism that restricts a frontend web application (e.g., hosted on `https://claimant.bimasevak.com`) from calling an API hosted on a different domain (e.g., `https://api.bimasevak.com`) unless the API explicitly permits it via response headers (`Access-Control-Allow-Origin`).

**How to Configure Securely**:
1.  **Avoid Wildcards (`*`) in Production**: Never allow all origins (`AllowAnyOrigin()`) on sensitive APIs. This exposes the API to CSRF attacks.
2.  **Register CORS Policy**: Configure specific, trusted domains in `Program.cs`:
    ```csharp
    builder.Services.AddCors(options =>
    {
        options.AddPolicy("CorsPolicy", policy =>
        {
            policy.WithOrigins("https://claimant.bimasevak.com", "https://admin.bimasevak.com")
                  .AllowAnyHeader()
                  .AllowAnyMethod()
                  .AllowCredentials(); // Required if sending cookies/session tokens
        });
    });
    ```
3.  **Activate in Middleware Pipeline**: Place CORS early in the pipeline (before Authentication and Controllers):
    ```csharp
    app.UseCors("CorsPolicy");
    ```

---

### Q48: IHttpClientFactory and Socket Exhaustion
**Question**: What is Socket Exhaustion, and how does using `IHttpClientFactory` resolve it?
**Answer**:
A common mistake in C# is creating a new `HttpClient` inside a using block for every external request:
```csharp
// ❌ BAD PRACTICE
using (var client = new HttpClient())
{
    var response = await client.GetAsync("https://api.com");
}
```
**The Problem (Socket Exhaustion)**:
Although the `using` block disposes of the client, the underlying OS socket remains open in a `TIME_WAIT` state for up to 4 minutes. If your application receives high traffic making many outbound calls, the server will quickly run out of available network sockets, crashing the app.

**The Solution (`IHttpClientFactory`)**:
`IHttpClientFactory` acts as a centralized **socket connection pooling engine**. It manages the lifetime of underlying `HttpClientHandler` instances, recycling them efficiently to prevent socket exhaustion while respecting DNS changes automatically.

*BSK Reference*:
BSK registers named clients centrally, allowing repository services to simply request client instances via factory injection:
```csharp
// Constructor injection
private readonly IHttpClientFactory _httpClientFactory;

public async Task CallExternalService()
{
    // Reuses pooled sockets seamlessly
    var client = _httpClientFactory.CreateClient("SurepassClient"); 
    var response = await client.GetAsync("api/v1/status");
}
```

---

### Q49: Parallel Async Execution — Task.WhenAll vs. Task.WhenAny
**Question**: What is the difference between `Task.WhenAll` and `Task.WhenAny`? How do you run multiple asynchronous operations in parallel in .NET?
**Answer**:
*   **`Task.WhenAll`**: Takes a collection of tasks and asynchronously waits for **all** of them to complete. If any task throws an exception, the aggregate exception contains failures from all tasks.
*   **`Task.WhenAny`**: Takes a collection of tasks and asynchronously waits for **any single task** to complete. It returns the task that completed first.

**Production Scenario**:
Suppose BSK needs to call three independent external APIs simultaneously (e.g., check PAN KYC, check dynamic SMS status, and ping the Accounting ledger) to finalize a case.
*   *Synchronous Sequential (Slow)*: Waiting for API 1 (1s) ➔ API 2 (1s) ➔ API 3 (1s) = Total latency of **3 seconds**.
*   *Parallel with `Task.WhenAll` (Fast)*: Triggering all three calls simultaneously and waiting for all to complete = Total latency of **only 1 second** (whichever API is slowest).

```csharp
public async Task<VerificationResultDto> VerifyAllServicesAsync(string pan, string mobile)
{
    // Start tasks simultaneously (do NOT use await here)
    Task<bool> panTask = _kycService.VerifyPanAsync(pan);
    Task<bool> smsTask = _smsService.PingGatewayAsync(mobile);
    Task<bool> ledgerTask = _ledgerService.CheckStatusAsync();

    // Run in parallel and await combined completion
    await Task.WhenAll(panTask, smsTask, ledgerTask);

    // Retrieve results safely without blocking
    return new VerificationResultDto
    {
        IsPanVerified = panTask.Result,
        IsSmsActive = smsTask.Result,
        IsLedgerOnline = ledgerTask.Result
    };
}
```

---

### Q50: Garbage Collection (GC) in .NET
**Question**: How does Garbage Collection (GC) work in .NET? Explain Generations, the Large Object Heap (LOH), and how to write memory-friendly code.
**Answer**:
The .NET Garbage Collector manages the allocation and release of memory. It operates on a **Generational model** based on the empirical rule that newly created objects tend to have short lifetimes.

**GC Generations**:
1.  **Generation 0 (Gen 0)**: Contains short-lived objects (e.g., local variables inside a controller action). Collection here is extremely frequent and fast.
2.  **Generation 1 (Gen 1)**: Acts as a buffer between Gen 0 and Gen 2. Objects that survive Gen 0 collections are promoted to Gen 1.
3.  **Generation 2 (Gen 2)**: Contains long-lived objects (e.g., Singleton services, static configurations, DB connection pools). Collection here is costly and involves pausing threads (Stop-The-World).
4.  **Large Object Heap (LOH)**: Objects **greater than 85,000 bytes** (e.g., large byte arrays from S3 document downloads) go directly here. LOH is not compacted during normal collections (due to pointer copy performance penalties), leading to memory fragmentation over time.
5.  **Pinned Object Heap (POH)**: Introduced in .NET 5 to pin objects in memory without blocking garbage collector movements.

**How to Write Memory-Friendly Backend Code**:
*   **Dispose Unmanaged Resources**: Implement `IDisposable` and use `using` statements for database connections, file streams, and network streams.
*   **Avoid LOH allocations for temporary data**: Reuse buffers via `ArrayPool<T>.Shared` when handling raw document byte array transformations.
*   **Avoid string concatenation inside loops**: Strings are immutable (every concatenation creates a new Gen 0 object). Use `StringBuilder` or `ReadOnlySpan<char>`.

---

### Q51: Value Types vs. Reference Types (Stack vs. Heap)
**Question**: Contrast Value Types and Reference Types. What are the memory implications, and what are structs and C# records?
**Answer**:

| Feature | Value Types (`struct`, `int`, `enum`, `bool`) | Reference Types (`class`, `interface`, `string`, `delegate`) |
|---|---|---|
| **Memory Allocation** | Allocated on the **Stack** (or inline within parent objects on the heap). | Allocated on the **Heap**; the pointer to the heap address is stored on the Stack. |
| **Assignment** | Copies the actual value (Copy-by-value). | Copies the memory address reference (Copy-by-reference). |
| **GC Overhead** | Deallocated instantly when execution leaves scope (No GC overhead). | Cleaned up dynamically by the Garbage Collector (GC overhead). |
| **Nullability** | Cannot be null by default (unless defined as nullable, e.g. `int?`). | Can be null. |

**Modern C# Enhancements**:
1.  **`readonly struct`**: Value types designed to represent small immutable data groups. Perfect for custom mathematics, HSL colors, or geo-coordinates. Since they are readonly, the runtime prevents defensive copying, increasing execution speeds.
2.  **`record`** (Introduced in C# 9): A reference type (or value type via `record struct`) with built-in **Value-based equality**. Records are immutable by default using the `init` property setter and include the `with` expression to clone records elegantly.
    *   *BSK Reference*: Perfect for DTO structures returned by repositories to controllers:
        ```csharp
        public record CaseSummaryDto(Guid Id, string ClaimantName, decimal ClaimAmount);
        ```

---

### Q52: EF Core Performance: Eager, Lazy, and Explicit Loading
**Question**: What are the differences between Eager, Lazy, and Explicit loading in Entity Framework Core? What is the N+1 query problem?
**Answer**:

1.  **Eager Loading**:
    Loads related data from the database as part of the initial query using the `.Include()` and `.ThenInclude()` operators. This compiles to SQL `JOIN` statements.
    *   *BSK Usage*: Used when you *know* you will immediately access child records (e.g., loading a case along with its claimant details).
        ```csharp
        var caseDetails = await _context.Cases
            .Include(c => c.Claimant)
            .FirstOrDefaultAsync(c => c.Id == id);
        ```
2.  **Lazy Loading**:
    Loads related data automatically from the database the first time the navigation property is accessed. (Requires the proxy package `Microsoft.EntityFrameworkCore.Proxies` and all navigation properties to be marked `virtual`).
    *   *⚠️ Pitfall (The N+1 Select Query Problem)*: If you fetch 100 Cases and loop through them to read their Claimant names, Lazy Loading will execute **1 initial query** to load the cases, and then **100 individual queries** to load the claimant details for each row. This devastates database performance.
3.  **Explicit Loading**:
    Loads related entities explicitly at a later stage using the `Entry()` API on the DbContext.
    ```csharp
    var singleCase = await _context.Cases.FindAsync(id);
    // Explicitly load Claimant later only if a condition is met
    await _context.Entry(singleCase).Reference(c => c.Claimant).LoadAsync();
    ```

---

### Q53: EF Core Interceptors vs. SaveChanges Overrides
**Question**: How can you intercept and modify database operations globally in EF Core? How is this used for automated auditing?
**Answer**:
To capture cross-cutting concerns like logging query performance, encrypting fields before saving, or auto-populating timestamps (`CreatedAt`, `UpdatedAt`), EF Core provides two approaches:

**Approach 1: Overriding `SaveChangesAsync` in DbContext**:
This is a standard approach where you capture the state of tracking entities before calling the database.
```csharp
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    var entries = ChangeTracker.Entries()
        .Where(e => e.Entity is IAuditableEntity && (e.State == EntityState.Added || e.State == EntityState.Modified));

    foreach (var entry in entries)
    {
        var entity = (IAuditableEntity)entry.Entity;
        entity.UpdatedAt = DateTime.UtcNow;
        if (entry.State == EntityState.Added)
        {
            entity.CreatedAt = DateTime.UtcNow;
        }
    }
    return await base.SaveChangesAsync(cancellationToken);
}
```

**Approach 2: EF Core Interceptors (`SaveChangesInterceptor`)**:
Introduced in EF Core 7, interceptors run outside the DbContext class, keeping it clean and SRP compliant. You register them inside `DbContextOptions`:
```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, 
        InterceptionResult<int> result, 
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        if (context == null) return base.SavingChangesAsync(eventData, result, cancellationToken);

        // Perform audit mappings dynamically...
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

---

### Q54: Secure Configurations (Secret Manager & Azure Key Vault)
**Question**: How should API keys, database connection strings, and certificates be managed securely in .NET Core production deployments?
**Answer**:
Hardcoding sensitive credentials (like SMTP passwords, S3 Secret Keys, or SQL Connection Strings) in `appsettings.json` is a major security vulnerability, as these configurations are often checked into source control (Git).

**.NET Secure Configuration Architecture**:
1.  **Local Development**: Use **Secret Manager (`dotnet user-secrets`)**. This stores credentials in a local JSON file in the developer's user profile directory (`%APPDATA%\Microsoft\UserSecrets\<GUID>`) completely outside the project folder. It cannot be accidentally committed to Git.
2.  **Production Environment Variables**: In Docker container cloud deployments, read values from secure environment variables. In .NET Core, `appsettings.json` keys are mapped directly to environment variables at runtime (e.g. `ConnectionStrings__DefaultConnection` maps to `ConnectionStrings:DefaultConnection`).
3.  **Enterprise Cloud Vaults**: Integrate cloud vaults directly into the `.NET Configuration ConfigurationBuilder` pipeline. 
    *   *Azure Key Vault / AWS Secrets Manager Integration (`Program.cs`)*:
        ```csharp
        builder.Configuration.AddAzureKeyVault(
            new Uri("https://bsk-prod-vault.vault.azure.net/"),
            new DefaultAzureCredential() // Uses Managed Identity (No passwords in code)
        );
        ```
    Once registered, calling `builder.Configuration["S3SecretKey"]` will transparently fetch secrets directly from Azure Key Vault or AWS Secrets Manager.

---

### Q55: Thread Safety in Singleton Services
**Question**: What are the risks of using standard variables in a Singleton service? How do you ensure thread safety?
**Answer**:
A **Singleton** service exists as a single instance shared across all users and execution threads simultaneously.

**The Risk**:
If a Singleton class contains a standard instance variable (like a raw list `List<T>` or dictionary `Dictionary<TKey, TValue>`), multiple concurrent API requests will attempt to write to it at the same time. This leads to **Race Conditions**, memory corruption, and throws collection modification exceptions.

**How to Ensure Thread Safety**:
1.  **Avoid State**: Keep Singleton classes stateless. Pass variables through method arguments instead of storing them as class properties.
2.  **Use Concurrent Collections**: Replace standard collections with thread-safe wrappers from `System.Collections.Concurrent`. E.g., `ConcurrentDictionary<TKey, TValue>` or `ConcurrentQueue<T>`.
3.  **Use Thread Synchronization (Locking)**:
    *   *BSK Reference*: BSK manages active vendor states in a thread-safe singleton.
    ```csharp
    public class ServiceHealthRegistry : IServiceHealthRegistry
    {
        // Thread-safe dictionary prevents lock crashes on concurrent reads/writes
        private readonly ConcurrentDictionary<string, bool> _healthStates = new();

        public void SetHealthStatus(string serviceName, bool isUp)
        {
            _healthStates.AddOrUpdate(serviceName, isUp, (key, oldValue) => isUp);
        }

        public bool IsServiceUp(string serviceName)
        {
            return _healthStates.TryGetValue(serviceName, out var isUp) && isUp;
        }
    }
    ```
4.  **SemaphoreSlim**: For asynchronous locking (locks using C# `lock()` statement cannot be awaited inside).

---

### Q56: Minimal APIs vs. Controller-Based APIs
**Question**: What are Minimal APIs? Contrast them with Controller-based routing in .NET Core.
**Answer**:
Introduced in .NET 6, **Minimal APIs** allow developers to construct fully functional REST endpoints with minimal code overhead by bypassing the traditional controller abstraction.

**Structural Comparison**:
*   *Controller-based (Heavyweight)*: Requires creating a class inheriting from `ControllerBase`, applying routing attributes, injecting constructors, and mapping actions.
*   *Minimal API (Lightweight)*: Maps endpoints directly inside `Program.cs` using lambda expressions.

```csharp
// Minimal API Endpoint mapping in Program.cs
app.MapGet("bsk/api/Case/GetCaseById/{id}", async (string id, ICaseAsyncRepository repo) =>
{
    var data = await repo.GetCaseById(id);
    return data != null ? Results.Ok(data) : Results.NotFound();
});
```

**Key Trade-offs**:
*   **Performance**: Minimal APIs are slightly faster in startup times and request execution speeds because they bypass the MVC controller metadata discovery pipeline.
*   **Complexity**: Controller-based APIs are much better for massive, enterprise-grade architectures (like BSK) because they keep controllers separated, clean, and organized using Dependency Injection. Minimal APIs are ideal for microservices and serverless functions.

---

### Q57: CQRS (Command Query Responsibility Segregation) & MediatR
**Question**: What is CQRS, and how does the MediatR library facilitate decoupling in .NET applications?
**Answer**:
**CQRS (Command Query Responsibility Segregation)** is a design pattern that separates reading data (Queries) from writing data (Commands). 

*   **Queries**: Fetch data without modifying state (Read-only).
*   **Commands**: Modify data (insert, update, delete) and return status changes without returning heavy data models.

**Using MediatR (Mediator Pattern)**:
MediatR decouples the API controller from the actual business handler. Controllers no longer inject repositories directly; they simply send a generic request object, and MediatR routes it to the correct handler in the business layer.

*Controller Code*:
```csharp
[HttpPost("RegisterCase")]
public async Task<IActionResult> RegisterCase([FromBody] RegisterCaseCommand command)
{
    // Sends the command into the MediatR in-memory bus
    var caseId = await _mediator.Send(command);
    return Ok(caseId);
}
```

*Handler Code (Encapsulated Business Logic)*:
```csharp
public class RegisterCaseCommandHandler : IRequestHandler<RegisterCaseCommand, Guid>
{
    private readonly ICaseRepository _repo;
    public RegisterCaseCommandHandler(ICaseRepository repo) => _repo = repo;

    public async Task<Guid> Handle(RegisterCaseCommand request, CancellationToken cancellationToken)
    {
        // Execute domain validations and DB updates here
        var newCase = new Case { ClaimantId = request.ClaimantId };
        return await _repo.SaveAsync(newCase);
    }
}
```
**Benefits**: Establishes clean division of labor, simplifies unit testing, and scales read and write pipelines independently.

---

### Q58: Database Transaction Management (Unit of Work)
**Question**: How do you manage database transactions across multiple repository calls to ensure atomic operations?
**Answer**:
In enterprise claims APIs like BSK, register transactions must be **atomic**: if creating a `Case` succeeds but creating its associated `Invoice` fails, the database must roll back to its original state to prevent orphaned data.

**How EF Core Manages Transactions**:
By default, every call to `SaveChangesAsync()` is wrapped inside a database transaction. If you modify 3 different entities and call `SaveChangesAsync()` once, EF Core ensures all succeed or fail together.

**Explicit Transactions (Across Multiple Repositories)**:
If operations are spread across different repositories, you must manage transactions explicitly using the DbContext:
```csharp
public async Task<bool> ProcessClaimRegistrationAsync(ClaimantModel claimant, CaseModel caseModel)
{
    // Begin transaction explicitly
    using var transaction = await _context.Database.BeginTransactionAsync();
    try
    {
        await _claimantRepo.AddAsync(claimant);
        await _context.SaveChangesAsync(); // Saves to generate ClaimantId

        caseModel.ClaimantId = claimant.Id;
        await _caseRepo.AddAsync(caseModel);
        await _context.SaveChangesAsync();

        // Commit transaction if all succeeded
        await transaction.CommitAsync();
        return true;
    }
    catch (Exception ex)
    {
        // Rollback database changes on any exception
        await transaction.RollbackAsync();
        LogHelper.Error("Transaction failed, database rolled back", ex);
        return false;
    }
}
```

---

### Q59: String vs. StringBuilder (Performance and Memory)
**Question**: Why is using string concatenation (`+=`) inside a loop bad for memory performance in .NET? How does `StringBuilder` work?
**Answer**:
In .NET, a **`System.String`** object is **immutable**. Once created, its value in the memory heap cannot be modified.

**The String Concatenation Problem**:
```csharp
// ❌ BAD PERFORMANCE
string log = "";
for (int i = 0; i < 1000; i++)
{
    log += $"Line {i}\n"; // Allocates 1,000 separate string objects in Gen 0 Heap!
}
```
In this loop, C# creates a brand new string object in the memory heap on every iteration, leaving the old string eligible for garbage collection. This causes significant heap allocation pressure and triggers frequent GC pauses.

**The Solution (`StringBuilder`)**:
`StringBuilder` allocates a contiguous character buffer inside memory. When you call `.Append()`, it modifies this existing buffer directly without creating new string allocations on the heap. Once finished, calling `.ToString()` creates the final string object.
```csharp
// ✅ EXCELLENT PERFORMANCE
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
{
    sb.AppendLine($"Line {i}"); // Modifies the internal character buffer directly
}
string finalLog = sb.ToString(); // Allocates only ONE string on the heap!
```

---

### Q60: C# Records and Pattern Matching
**Question**: What are the key features of C# Records? How is Pattern Matching used to write clean condition expressions?
**Answer**:
Modern C# (versions 9 through 12) introduced elegant features to simplify data modeling and control flow logic.

**C# Records**:
Designed to represent immutable data models with built-in value-based equality.
```csharp
// Defines a record with read-only properties
public record PartnerVerification(Guid PartnerId, string DocumentType, bool IsVerified);

// Value comparison
var p1 = new PartnerVerification(guid, "PAN", true);
var p2 = new PartnerVerification(guid, "PAN", true);
bool isEqual = (p1 == p2); // Returns TRUE (Classes would return false due to reference mismatch)

// Mutation using 'with' (Non-destructive mutation)
var p3 = p1 with { DocumentType = "Aadhar" }; // Clones p1 but changes only DocumentType
```

**Advanced Pattern Matching**:
Pattern matching allows testing expressions to extract data dynamically, replacing heavy, nested `if-else` or traditional `switch` blocks.

```csharp
// BSK Scenario: Resolving Case Status Color Codes via Pattern Matching Switch Expression
public string GetStatusColor(CaseModel model) => model switch
{
    { IsSuccessOrDropped: true } => "green",
    { IsCaseReached: false } => "yellow",
    { ClaimAmount: > 500000 } => "red-bold", // Relational pattern
    null => "gray",
    _ => "blue" // Default fallback
};
```

---

## ⚙️ Advanced .NET Request Lifecycles, Compilation & SDK Tools

### Q61: The ASP.NET Core HTTP Request Lifecycle
**Question**: Trace the exact lifecycle of an incoming HTTP request in ASP.NET Core. What steps does it take from the network socket to the controller and back?
**Answer**:
When an HTTP request hits an ASP.NET Core API, it goes through a highly optimized, structured lifecycle:

```
[Client Request]
       │
       ▼
 1. [Kestrel / IIS Server] (Receives raw bytes, parses HTTP headers)
       │
       ▼
 2. [Middleware Pipeline] (UseCors ➔ UseRouting ➔ UseAuthentication ➔ UseAuthorization)
       │
       ▼
 3. [Endpoint Routing Middleware] (Matches route to target Controller action)
       │
       ▼
 4. [Filter Pipeline]
       │   ├── Authorization Filters (Verifies user claims/scopes)
       │   ├── Resource Filters (Executes before model binding; e.g. custom caching)
       │   ├── Model Binding (Parses query/body parameters into C# objects)
       │   ├── Action Filters (Runs immediately before and after action method)
       │   └── Exception Filters (Intercepts unhandled errors)
       │
       ▼
 5. [Controller Action Execution] (Business logic, DB calls, returns IActionResult)
       │
       ▼
 6. [Result Filters] (Formats output; e.g. serializes JSON response)
       │
       ▼
 [HTTP Response sent back through Middlewares to Kestrel]
```

**Key Phases**:
1.  **Web Server (Kestrel)**: The default cross-platform HTTP web server. It handles socket connections, SSL handshakes, and builds the `HttpContext` object representing the request/response container.
2.  **Middleware Pipeline**: Executes sequentially. If a middleware (like authentication) fails, it short-circuits the request and returns a response immediately, bypassing the controller.
3.  **Filters**: Action-specific cross-cutting concerns that execute *inside* the MVC context.

---

### Q62: .NET Compilation & Execution Lifecycle (JIT vs. AOT)
**Question**: How does a C# file compile and run? Explain the role of Roslyn, Intermediate Language (IL), the CLR, JIT compilation, and Native AOT.
**Answer**:

```
[C# Source Code] ──► (Roslyn Compiler) ──► [Intermediate Language (IL) inside .dll]
                                                    │
                                                    ▼
[Machine CPU Instructions] ◄── (JIT Compiler) ◄── [Common Language Runtime (CLR)]
```

**Traditional Lifecycle**:
1.  **Roslyn Compilation (Compile-Time)**: The C# compiler (`Roslyn`) reads your C# code and compiles it into **Intermediate Language (IL)**, stored inside `.dll` assembly files along with descriptive metadata.
2.  **Deployment**: The IL DLLs are distributed to the server.
3.  **CLR Runtime Execution (Run-Time)**: The **Common Language Runtime (CLR)** loads the IL. The first time a method is called, the **Just-In-Time (JIT) Compiler** compiles the IL bytecodes into native binary machine code instructions specific to the host CPU (x64, ARM) and caches them in memory for future calls.

**Native AOT (Ahead-of-Time) Compilation (Modern alternative)**:
Introduced in recent .NET versions, **Native AOT** bypasses the IL and JIT steps entirely.
*   *How it works*: During compilation, Roslyn compiles the C# source code directly into native binary machine code for the target OS/CPU.
*   *Benefits*: Extremely fast startup times and highly reduced memory footprint (as the JIT compiler and CLR runtime are not loaded into memory). Essential for serverless functions and high-performance microservices.

---

### Q63: Advanced Dependency Injection & Captive Dependencies
**Question**: What is a Captive Dependency? Provide a C# example, explain why it is dangerous, and show how to resolve it.
**Answer**:
A **Captive Dependency** occurs when a service with a short lifetime is injected into a service with a longer lifetime, effectively "capturing" and holding onto it, preventing its timely disposal.

**The Classic Trap**:
Injecting a **Scoped** service (like `ApplicationDBContext` which should last for exactly one HTTP request) into a **Singleton** service (which lasts forever).
```csharp
// ❌ DANGEROUS: Scoped DBContext is captured inside a Singleton service!
public class SingletonSettingsRegistry
{
    private readonly ApplicationDBContext _dbContext; // Scoped

    public SingletonSettingsRegistry(ApplicationDBContext dbContext)
    {
        _dbContext = dbContext; // DBContext will NEVER be disposed, leading to memory leaks and db crashes!
    }
}
```
**Why it's dangerous**:
The Scoped DBContext will remain alive in memory forever because the Singleton service lives forever. The database connections will remain open, leading to **connection pool exhaustion**, stale tracked entities, and thread-safety exceptions when multiple HTTP threads call the Singleton concurrently.

**How to Resolve Safely (Service Locator Pattern via Scope Factory)**:
Instead of injecting the scoped service directly, inject the **`IServiceScopeFactory`** and create a temporary scoped context *inside* the execution method:
```csharp
// ✅ SECURE & SAFE
public class SingletonSettingsRegistry
{
    private readonly IServiceScopeFactory _scopeFactory;

    public SingletonSettingsRegistry(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task<string> GetSettingAsync(string key)
    {
        // Create a temporary scope
        using var scope = _scopeFactory.CreateScope();
        
        // Resolve scoped DbContext safely inside this local execution thread
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDBContext>();
        
        var setting = await dbContext.Settings.FirstOrDefaultAsync(s => s.Key == key);
        return setting?.Value;
    } // Temporary scope is disposed here, closing the DB connection immediately!
}
```

---

### Q64: Host Application Lifecycles (`IHostApplicationLifetime`)
**Question**: How do you execute custom setup or graceful cleanup actions on ASP.NET Core host startup and shutdown?
**Answer**:
To run custom logic during startup transitions or handle graceful resource cleanups (like flushing memory logs, closing active channels, or notifying third parties of server shutdown) during shutdown sequences, you inject the **`IHostApplicationLifetime`** service.

**C# Implementation**:
```csharp
public class GracefulShutdownService
{
    private readonly IHostApplicationLifetime _appLifetime;
    private readonly ILogger<GracefulShutdownService> _logger;

    public GracefulShutdownService(IHostApplicationLifetime appLifetime, ILogger<GracefulShutdownService> logger)
    {
        _appLifetime = appLifetime;
        _logger = logger;

        // Register event callbacks
        _appLifetime.ApplicationStarted.Register(OnStarted);
        _appLifetime.ApplicationStopping.Register(OnStopping);
        _appLifetime.ApplicationStopped.Register(OnStopped);
    }

    private void OnStarted()
    {
        _logger.LogInformation("Application started successfully - initializing active queues.");
    }

    private void OnStopping()
    {
        _logger.LogWarning("Application is stopping - pausing new requests and flushing memory pools.");
        // Place cleanup logic here before the process terminates
    }

    private void OnStopped()
    {
        _logger.LogInformation("Application stopped completely.");
    }
}
```

---

### Q65: .NET SDK CLI Development and Watch Tools
**Question**: What are the primary commands in the `.NET` SDK CLI? How does Hot Reload (`dotnet watch`) work under the hood?
**Answer**:
The `.NET` SDK provides a unified Command Line Interface (CLI) to build, run, and publish applications across platforms.

**Core CLI Commands**:
*   `dotnet restore`: Restores Nuget package dependencies declared in the `.csproj` file.
*   `dotnet build`: Compiles IL assemblies, checking for syntax errors. (Automatically triggers `dotnet restore` first).
*   `dotnet run`: Compiles and executes the entry point DLL.
*   `dotnet test`: Executes unit/integration test suites.
*   `dotnet publish`: Compiles and publishes the optimized release binaries (with pruned configurations) to a deployment folder ready for hosting.

**Hot Reload (`dotnet watch`)**:
`dotnet watch` monitors source files for changes. When a C# file is modified:
1.  Instead of shutting down the web server, recompiling the entire project, and restarting Kestrel (which takes 5-10 seconds), the tool compiles *only* the specific code block that changed.
2.  It utilizes the **Roslyn compiler's delta-update engine** to patch the running IL code in memory inside the active process dynamically.
3.  The browser UI is updated instantly, maintaining the active application state (e.g. login sessions) for the developer.

---

### Q66: .NET Diagnostic and Performance SDK Tools
**Question**: What are the primary .NET CLI diagnostic tools? How do they help troubleshoot production issues?
**Answer**:
When an API is running in production inside a Docker container on Linux, you cannot attach the Visual Studio UI debugger. You must use the dedicated **.NET Diagnostic SDK Tools**:

1.  **`dotnet-dump`**:
    *   *Purpose*: Captures and analyzes OS memory dumps of running C# processes.
    *   *Troubleshooting*: Run `dotnet-dump collect -p <PID>` when memory is spiking. Open the dump file to view which objects are leaking and holding heap memory allocations.
2.  **`dotnet-trace`**:
    *   *Purpose*: A lightweight CPU sampling profiler that tracks method execution durations.
    *   *Troubleshooting*: Captures slow API cycles to trace exactly which C# method or SQL call is blocking threads.
3.  **`dotnet-counters`**:
    *   *Purpose*: Real-time performance metrics monitor.
    *   *Troubleshooting*: Monitors CPU utilization, Garbage Collector heap sizes, active thread pool counts, and HTTP request counts in real-time inside the terminal.
4.  **`dotnet-stack`**:
    *   *Purpose*: Dumps active call stacks of all threads.
    *   *Troubleshooting*: Instantly pinpoints where deadlocks are occurring in synchronous locks.

---

### Q67: Middleware vs. Action Filters
**Question**: What is the difference between Middleware and Action Filters in ASP.NET Core? When should you use one over the other?
**Answer**:

| Feature | Middleware | Action Filters |
|---|---|---|
| **Scope** | Global. Operates on all HTTP requests hitting the server. | Specific. Operates only on requests that match MVC Controller Actions. |
| **Pipeline Level** | Low-level HTTP context (Headers, raw streams). Runs outside the MVC routing engine. | High-level MVC context (Action parameters, Model State, routing data). |
| **Short-Circuiting** | Can return raw response instantly (e.g., UseCors, StaticFiles). | Short-circuits MVC execution by setting `ActionExecutingContext.Result`. |
| **Access to MVC** | No access to MVC constructs (e.g., cannot read target controller attributes easily). | Full access to target controller, parameter validators, and model properties. |

**Guidelines**:
*   *Use Middleware for*: App-wide, low-level infrastructural concerns: CORS configuration, request logging headers, gzip compression, URL redirection, and global error catching.
*   *Use Action Filters for*: Action-specific business concerns: validating `ModelState` globally, checking custom parameter permissions (`[HasClaims]`), or injecting controller caching logic.

---

### Q68: Global Exceptions vs. Developer Exception Page
**Question**: Explain how ASP.NET Core handles unhandled exceptions. What is the difference between the Developer Exception Page and Global Exception Handling middleware in production?
**Answer**:
If an API action throws an unhandled exception (e.g. database connection drops), the request flows backwards out of the pipeline. If not caught, the server crashes or returns raw, unformatted HTML errors to the client.

**Dev vs. Prod Configuration (`Program.cs`)**:
```csharp
if (app.Environment.IsDevelopment())
{
    // 1. DEVELOPMENT ENVIRONMENT
    app.UseDeveloperExceptionPage(); 
}
else
{
    // 2. PRODUCTION ENVIRONMENT
    app.UseExceptionHandler("/error"); // Redirects to a safe error page or middleware
    app.UseHsts(); // Enforces strict HTTPS transport security
}
```

**Key Differences**:
1.  **Developer Exception Page (`UseDeveloperExceptionPage`)**:
    Renders a detailed HTML page containing the raw C# stack trace, line numbers of the failure, request headers, and routing parameters. **Must never be enabled in production** as it exposes database schemas and critical code configurations to potential attackers.
2.  **Global Exception Handler (`UseExceptionHandler`)**:
    Interposes a global try-catch block. On failure, it clears the response headers and routes the request to a dedicated `/error` path that returns a clean, standard, and secure JSON response payload conforming to the **Problem Details (RFC 7807)** standard, keeping stack traces hidden from the outside world.

---

### Q69: Garbage Collector (GC) Modes: Workstation vs. Server
**Question**: What are the differences between Workstation GC and Server GC in the .NET runtime? How do they affect performance?
**Answer**:
The .NET runtime offers two primary Garbage Collection modes optimized for different hosting scenarios:

1.  **Workstation GC (Default for desktops)**:
    *   *Design*: Optimized for low latency and high responsiveness.
    *   *Thread allocation*: Runs concurrently on a single background GC thread.
    *   *Performance*: Minimal CPU interruptions, keeping UI interfaces smooth. Ideal for desktop apps (WPF) or local console tools.
2.  **Server GC (Default for ASP.NET Core server hosting)**:
    *   *Design*: Optimized for maximum request throughput.
    *   *Thread allocation*: Creates **one dedicated GC thread and one heap per CPU core**.
    *   *Performance*: Extremely fast on multi-core servers as collections run in parallel. However, it takes up more memory overhead. Ideal for heavy backend APIs processing thousands of concurrent requests.

**Configuration inside `.csproj` or `runtimeconfig.json`**:
```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
</PropertyGroup>
```

---

### Q70: Assemblies, DLLs, and AppDomains in .NET Core
**Question**: What is an Assembly in .NET? How does assembly loading in .NET Core (using `AssemblyLoadContext`) differ from the older AppDomains framework?
**Answer**:
*   **Assembly**: The primary deployment and versioning unit in `.NET`. An assembly is represented by a physical `.dll` or `.exe` file containing compiled Intermediate Language (IL) code, metadata structures, and resource files.
*   **Older .NET Framework (`AppDomains`)**:
    *   Used "Application Domains" (`AppDomains`) to isolate applications hosted inside the same process. However, AppDomains were heavy, complex, and tied directly to Windows security boundaries.
*   **Modern .NET Core (`AssemblyLoadContext`)**:
    *   *What it is*: A lightweight mechanism for managing assembly loading boundaries within a single process.
    *   *Dynamic Plugin Loading*: `AssemblyLoadContext` allows you to load assembly DLLs dynamically at runtime, isolate them, and most importantly, **unload them completely** from memory when no longer needed. This enables building modular plugin architectures (like dynamic report generators in BSK) without leaking memory over time.

---

### 💡 Preparation Tip
Keep this guide in your local workspace. Using these real-world references during technical discussions demonstrates strong, production-level expertise.



