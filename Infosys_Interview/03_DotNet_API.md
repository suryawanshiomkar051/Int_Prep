# 🌐 .NET Web API – Infosys Interview Questions & Answers

---

## Q1. Authentication & Authorization in Web API

### Authentication (Who are you?) vs Authorization (What can you do?)

```csharp
// Program.cs - JWT Authentication setup
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
    options.AddPolicy("ManagerOrAbove", policy => policy.RequireRole("Admin", "Manager"));
});

app.UseAuthentication();  // Must come before UseAuthorization
app.UseAuthorization();

// Generate JWT Token
public string GenerateToken(string username, string role)
{
    var claims = new[] {
        new Claim(ClaimTypes.Name, username),
        new Claim(ClaimTypes.Role, role),
        new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
    };
    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Key"]));
    var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
    var token = new JwtSecurityToken(
        issuer: _config["Jwt:Issuer"],
        audience: _config["Jwt:Audience"],
        claims: claims,
        expires: DateTime.Now.AddHours(1),
        signingCredentials: creds);
    return new JwtSecurityTokenHandler().WriteToken(token);
}

// Protect endpoints
[Authorize]                          // Any authenticated user
[Authorize(Roles = "Admin")]         // Specific role
[Authorize(Policy = "AdminOnly")]    // Named policy
[AllowAnonymous]                     // No auth required
public class OrdersController : ControllerBase { }
```

### Other Auth Mechanisms:
- **API Key** – simple header-based for machine-to-machine
- **OAuth 2.0 / OpenID Connect** – third-party login (Google, Azure AD)
- **Basic Auth** – username:password in header (not recommended for production)

---

## Q2. Use of Decorators (Attributes) in API Controller

**Decorators = Attributes** in C#. They add metadata/behavior to classes/methods.

```csharp
[ApiController]                          // Enables automatic model validation, binding errors
[Route("api/[controller]")]              // Base route: api/orders
[Authorize]                              // All actions require auth
[Produces("application/json")]           // Response content type
public class OrdersController : ControllerBase
{
    // Route decorators
    [HttpGet]                            // GET api/orders
    [HttpGet("{id}")]                    // GET api/orders/5
    [HttpPost]                           // POST api/orders
    [HttpPut("{id}")]                    // PUT api/orders/5
    [HttpDelete("{id}")]                 // DELETE api/orders/5

    // Response type decorators (Swagger documentation)
    [ProducesResponseType(typeof(Order), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]

    // Auth decorators
    [AllowAnonymous]                     // Override class-level [Authorize]
    [Authorize(Roles = "Admin")]

    // Binding decorators
    public IActionResult CreateOrder(
        [FromBody] CreateOrderDto dto,    // From request body (JSON)
        [FromQuery] string currency,      // From query string ?currency=USD
        [FromRoute] int id,               // From route /api/orders/{id}
        [FromHeader] string apiKey,       // From request header
        [FromForm] IFormFile file)        // From form data (file upload)
    { }

    // Caching
    [ResponseCache(Duration = 60)]

    // Custom attributes
    [ValidateModel]
    [LogAction]
    public IActionResult GetOrders() { }
}
```

---

## Q3. Use of Action Filters

**Action Filters** run code **before or after** a controller action executes.

```csharp
// Types of Filters (in execution order):
// 1. Authorization Filters
// 2. Resource Filters
// 3. Action Filters       ← Most commonly used
// 4. Exception Filters
// 5. Result Filters

// Custom Action Filter
public class LogActionFilter : IActionFilter
{
    private readonly ILogger _logger;
    public LogActionFilter(ILogger<LogActionFilter> logger) { _logger = logger; }

    // Runs BEFORE action
    public void OnActionExecuting(ActionExecutingContext context)
    {
        _logger.LogInformation($"Action executing: {context.ActionDescriptor.DisplayName}");
        _logger.LogInformation($"Parameters: {JsonSerializer.Serialize(context.ActionArguments)}");

        // Can short-circuit the action:
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(context.ModelState);
        }
    }

    // Runs AFTER action
    public void OnActionExecuted(ActionExecutedContext context)
    {
        _logger.LogInformation($"Action executed. Status: {(context.Result as ObjectResult)?.StatusCode}");
    }
}

// As Attribute (easier to apply)
public class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            var errors = context.ModelState
                .Where(x => x.Value.Errors.Count > 0)
                .ToDictionary(k => k.Key, v => v.Value.Errors.Select(e => e.ErrorMessage));
            context.Result = new BadRequestObjectResult(new { errors });
        }
    }
}

// Register globally
builder.Services.AddControllers(options =>
{
    options.Filters.Add<LogActionFilter>();
});

// Or per controller/action
[ValidateModel]
public class ProductsController : ControllerBase { }
```

---

## Q4. Global Error Handling

```csharp
// Method 1: UseExceptionHandler middleware (recommended)
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionFeature = context.Features.Get<IExceptionHandlerPathFeature>();
        var exception = exceptionFeature?.Error;

        var (statusCode, message) = exception switch
        {
            NotFoundException => (404, "Resource not found"),
            UnauthorizedAccessException => (401, "Unauthorized"),
            ValidationException ex => (400, ex.Message),
            _ => (500, "An internal error occurred")
        };

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(new
        {
            error = message,
            traceId = Activity.Current?.Id ?? context.TraceIdentifier
        });
    });
});

// Method 2: Exception Filter
public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;
    public GlobalExceptionFilter(ILogger<GlobalExceptionFilter> logger) { _logger = logger; }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Unhandled exception in {Action}", 
            context.ActionDescriptor.DisplayName);

        context.Result = new ObjectResult(new { 
            error = "Server error", 
            message = context.Exception.Message 
        }) { StatusCode = 500 };
        context.ExceptionHandled = true;
    }
}

// Method 3: Problem Details (RFC 7807 standard)
builder.Services.AddProblemDetails();
app.UseStatusCodePages();
// Returns standardized error format:
// { "type": "...", "title": "...", "status": 404, "detail": "..." }
```

---

## Q5. Logging of Request / Response

```csharp
// Method 1: Request/Response Logging Middleware
public class RequestResponseLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestResponseLoggingMiddleware> _logger;

    public RequestResponseLoggingMiddleware(RequestDelegate next, ILogger<RequestResponseLoggingMiddleware> logger)
    {
        _next = next; _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Log Request
        context.Request.EnableBuffering();  // Allow re-reading body
        var requestBody = await new StreamReader(context.Request.Body).ReadToEndAsync();
        context.Request.Body.Position = 0;

        _logger.LogInformation(
            "HTTP {Method} {Path} | Query: {Query} | Body: {Body}",
            context.Request.Method,
            context.Request.Path,
            context.Request.QueryString,
            requestBody);

        // Capture Response
        var originalBody = context.Response.Body;
        using var memoryStream = new MemoryStream();
        context.Response.Body = memoryStream;

        var stopwatch = Stopwatch.StartNew();
        await _next(context);
        stopwatch.Stop();

        memoryStream.Position = 0;
        var responseBody = await new StreamReader(memoryStream).ReadToEndAsync();
        memoryStream.Position = 0;
        await memoryStream.CopyToAsync(originalBody);
        context.Response.Body = originalBody;

        _logger.LogInformation(
            "HTTP {StatusCode} | Duration: {Duration}ms | Response: {Body}",
            context.Response.StatusCode,
            stopwatch.ElapsedMilliseconds,
            responseBody);
    }
}

// Method 2: Built-in HTTP Logging (ASP.NET Core 6+)
builder.Services.AddHttpLogging(logging =>
{
    logging.LoggingFields = HttpLoggingFields.All;
    logging.RequestBodyLogLimit = 4096;
    logging.ResponseBodyLogLimit = 4096;
});
app.UseHttpLogging();

// Method 3: Serilog with structured logging
// Install: Serilog.AspNetCore
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();
builder.Host.UseSerilog();
app.UseSerilogRequestLogging();
```

---

## Q6. API Routing

```csharp
// Convention-based Routing (MVC)
app.MapControllerRoute(name: "default", pattern: "{controller=Home}/{action=Index}/{id?}");

// Attribute Routing (Web API - preferred)
[ApiController]
[Route("api/v1/[controller]")]  // api/v1/orders
public class OrdersController : ControllerBase
{
    [HttpGet]                       // GET api/v1/orders
    [HttpGet("{id:int}")]           // GET api/v1/orders/5
    [HttpGet("{id:int}/items")]     // GET api/v1/orders/5/items
    [HttpGet("pending")]            // GET api/v1/orders/pending
    [HttpGet("by-date/{date:datetime}")] // GET api/v1/orders/by-date/2026-10-17

    // Route constraints
    [HttpGet("{id:int:min(1)}")]    // int, minimum value 1
    [HttpGet("{name:alpha}")]       // alphabetic only
    [HttpGet("{id:guid}")]          // GUID format
}

// Multiple routes on one action
[HttpGet("all")]
[HttpGet("list")]
public IActionResult GetAll() { }

// Catch-all route
[HttpGet("{**path}")]
public IActionResult CatchAll(string path) { }
```

---

## Q7. HTTP Verbs – GET / POST / PUT / DELETE

| Verb | Purpose | Idempotent? | Body? | Response |
|------|---------|-------------|-------|----------|
| GET | Read resource | Yes | No | 200 OK |
| POST | Create resource | No | Yes | 201 Created |
| PUT | Full update (replace) | Yes | Yes | 200 OK / 204 No Content |
| PATCH | Partial update | No | Yes | 200 OK |
| DELETE | Remove resource | Yes | Optional | 204 No Content |

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET api/products
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll()
        => Ok(await _service.GetAllAsync());

    // GET api/products/5
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetById(int id)
    {
        var product = await _service.GetByIdAsync(id);
        return product is null ? NotFound() : Ok(product);
    }

    // POST api/products
    [HttpPost]
    public async Task<ActionResult<ProductDto>> Create([FromBody] CreateProductDto dto)
    {
        var created = await _service.CreateAsync(dto);
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }

    // PUT api/products/5
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateProductDto dto)
    {
        await _service.UpdateAsync(id, dto);
        return NoContent();
    }

    // DELETE api/products/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        await _service.DeleteAsync(id);
        return NoContent();
    }
}
```

---

## Q8. Model Binding in API Controller

**Model Binding** automatically maps HTTP request data to action method parameters.

```csharp
// Sources of binding:
[HttpGet("{id}")]
public IActionResult Get(
    [FromRoute] int id,          // Route: /api/products/5
    [FromQuery] string filter,   // Query: ?filter=active
    [FromHeader] string apiKey,  // Header: X-Api-Key: abc
    [FromBody] CreateDto dto,    // Body: JSON
    [FromForm] IFormFile file,   // Form data
    [FromServices] ILogger logger) // From DI container

// Complex model binding
public class OrderFilter
{
    [FromQuery(Name = "start")]   // Maps ?start=... to StartDate
    public DateTime? StartDate { get; set; }
    
    [FromQuery]
    public string Status { get; set; }
    
    [FromQuery]
    public int Page { get; set; } = 1;
    
    [FromQuery]
    public int PageSize { get; set; } = 10;
}
[HttpGet]
public IActionResult GetOrders([FromQuery] OrderFilter filter) { }

// Data Annotations for validation
public class CreateProductDto
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; }

    [Range(0.01, 99999.99)]
    public decimal Price { get; set; }

    [RegularExpression(@"^[A-Z]{3}$", ErrorMessage = "Invalid currency code")]
    public string Currency { get; set; }
}
```

---

## Q9. HTTP Response Codes

| Code | Name | When to Use |
|------|------|-------------|
| **200** | OK | Successful GET/PUT/PATCH |
| **201** | Created | Successful POST (resource created) |
| **204** | No Content | Successful DELETE or PUT with no body |
| **400** | Bad Request | Invalid request data / validation failure |
| **401** | Unauthorized | Not authenticated |
| **403** | Forbidden | Authenticated but not authorized |
| **404** | Not Found | Resource doesn't exist |
| **409** | Conflict | Resource already exists / concurrency conflict |
| **422** | Unprocessable Entity | Semantic validation error |
| **429** | Too Many Requests | Rate limit exceeded |
| **500** | Internal Server Error | Unexpected server error |
| **503** | Service Unavailable | Server down / overloaded |

```csharp
// Using ActionResult return types
return Ok(data);                                    // 200
return Created($"/api/orders/{id}", createdOrder); // 201
return CreatedAtAction(nameof(GetById), new { id }, order); // 201
return NoContent();                                 // 204
return BadRequest(new { error = "Invalid data" }); // 400
return Unauthorized();                              // 401
return Forbid();                                    // 403
return NotFound($"Order {id} not found");          // 404
return Conflict("Record already exists");           // 409
return StatusCode(500, "Internal Server Error");    // 500
return Problem("Server error", statusCode: 503);   // 503
```

---

## Q10. API Versioning

```csharp
// Install: Asp.Versioning.Mvc

// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    // Choose versioning strategy:
}).AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Method 1: URL Segment versioning (most common)
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok("Version 1 products");
}

[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok("Version 2 products with more features");
}
// Usage: GET /api/v1/products  or  GET /api/v2/products

// Method 2: Query string versioning
// GET /api/products?api-version=1.0

// Method 3: Header versioning
// X-Api-Version: 1.0

// Deprecate old versions
[ApiVersion("1.0", Deprecated = true)]
```

---

## Q11. Steps to Create Web API in Your Application

```
Step 1: Create Project
  dotnet new webapi -n MyApi --framework net8.0
  cd MyApi

Step 2: Project Structure
  MyApi/
  ├── Controllers/       → API endpoints
  ├── Models/            → Request/Response DTOs
  ├── Services/          → Business logic interfaces & implementations
  ├── Repositories/      → Data access layer
  ├── Middleware/        → Custom middleware
  ├── Filters/           → Action/Exception filters
  ├── appsettings.json   → Configuration
  └── Program.cs         → App entry point & DI registration
```

```csharp
// Step 3: Define Model
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Step 4: Create DTO
public record CreateProductDto(string Name, decimal Price);
public record ProductDto(int Id, string Name, decimal Price);

// Step 5: Create Interface & Service
public interface IProductService
{
    Task<IEnumerable<ProductDto>> GetAllAsync();
    Task<ProductDto?> GetByIdAsync(int id);
    Task<ProductDto> CreateAsync(CreateProductDto dto);
}

public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    public ProductService(IProductRepository repository) { _repository = repository; }
    // ... implementations
}

// Step 6: Create Controller
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    public ProductsController(IProductService service) { _service = service; }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll()
        => Ok(await _service.GetAllAsync());
}

// Step 7: Register in Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer(...);
builder.Services.AddAuthorization();

var app = builder.Build();
if (app.Environment.IsDevelopment()) { app.UseSwagger(); app.UseSwaggerUI(); }
app.UseExceptionHandler();
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();

// Step 8: Test with Swagger (http://localhost:5000/swagger)
// Step 9: Write unit tests
// Step 10: Add logging, monitoring (ApplicationInsights/Serilog)
```

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep*

---

---

# ⭐ BONUS – Important Questions (Added by Interviewer Pattern Analysis)

---

## BQ1. What is CORS and How Do You Configure It in ASP.NET Core?

**CORS (Cross-Origin Resource Sharing)** is a browser security feature that blocks web pages from making requests to a different domain than the one that served the page.

```csharp
// Problem: Frontend at http://localhost:3000 calls API at http://localhost:5000
// Browser blocks this — "Cross-Origin Request Blocked"

// Fix: Configure CORS in Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    // Policy 1: Specific origins (Production - restrictive)
    options.AddPolicy("ProductionPolicy", policy =>
    {
        policy.WithOrigins("https://myapp.com", "https://www.myapp.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Content-Type", "Authorization")
              .AllowCredentials();  // Allow cookies/auth headers
    });

    // Policy 2: Allow any origin (Development only!)
    options.AddPolicy("DevelopmentPolicy", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

var app = builder.Build();

// Apply globally
if (app.Environment.IsDevelopment())
    app.UseCors("DevelopmentPolicy");
else
    app.UseCors("ProductionPolicy");

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

// Apply per controller/action
[EnableCors("ProductionPolicy")]
public class ProductsController : ControllerBase { }

[DisableCors]  // No CORS for this action
public IActionResult InternalEndpoint() { }
```

### CORS Headers Explained:
- `Access-Control-Allow-Origin: https://myapp.com` — who can call
- `Access-Control-Allow-Methods: GET, POST` — which methods
- `Access-Control-Allow-Headers: Authorization` — which headers
- `Access-Control-Allow-Credentials: true` — allow cookies

---

## BQ2. How to Implement Rate Limiting in ASP.NET Core?

**Rate Limiting** prevents API abuse by limiting how many requests a client can make in a time window.

```csharp
// .NET 7+ has built-in rate limiting
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    // Fixed Window: 100 requests per minute per IP
    options.AddFixedWindowLimiter("FixedPolicy", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 5;
    });

    // Sliding Window: Smoother than fixed
    options.AddSlidingWindowLimiter("SlidingPolicy", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 6;  // 10-second segments
    });

    // Token Bucket: Allows bursts
    options.AddTokenBucketLimiter("TokenBucketPolicy", opt =>
    {
        opt.TokenLimit = 20;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
        opt.TokensPerPeriod = 5;
        opt.AutoReplenishment = true;
    });

    // Concurrency: Max concurrent requests
    options.AddConcurrencyLimiter("ConcurrencyPolicy", opt =>
    {
        opt.PermitLimit = 10;
        opt.QueueLimit = 5;
    });

    // Per-user rate limiting
    options.AddPolicy("PerUserPolicy", context =>
    {
        var userId = context.User.FindFirst("sub")?.Value ?? context.Connection.RemoteIpAddress?.ToString();
        return RateLimitPartition.GetFixedWindowLimiter(userId, _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 50,
            Window = TimeSpan.FromMinutes(1)
        });
    });

    // What to return when limit exceeded
    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        context.HttpContext.Response.Headers["Retry-After"] = "60";
        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "Too many requests. Please try again later.",
            retryAfter = 60
        }, token);
    };
});

var app = builder.Build();
app.UseRateLimiter();  // Must be AFTER UseRouting

// Apply to specific endpoints
[EnableRateLimiting("FixedPolicy")]
public class OrdersController : ControllerBase
{
    [HttpPost]
    [EnableRateLimiting("TokenBucketPolicy")]  // Override controller policy
    public IActionResult CreateOrder() { }

    [DisableRateLimiting]  // No rate limiting for this action
    public IActionResult HealthCheck() { }
}
```

---

## BQ3. What is Swagger / OpenAPI and How to Configure It?

**Swagger/OpenAPI** is a specification for documenting REST APIs — provides interactive UI to test APIs.

```csharp
// Install: Swashbuckle.AspNetCore

// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    // API Info
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "API for managing orders",
        Contact = new OpenApiContact { Name = "Omkar", Email = "omkar@example.com" }
    });

    // Add JWT Authentication to Swagger UI
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "Enter: Bearer {token}"
    });
    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme { Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" } },
            Array.Empty<string>()
        }
    });

    // Include XML comments from code
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    options.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFile));
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
        options.RoutePrefix = string.Empty;  // Swagger at root URL
    });
}

// Documenting endpoints with XML comments
/// <summary>Creates a new order</summary>
/// <param name="dto">Order creation data</param>
/// <returns>Created order</returns>
/// <response code="201">Order created successfully</response>
/// <response code="400">Invalid request data</response>
[HttpPost]
[ProducesResponseType(typeof(OrderDto), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
public async Task<IActionResult> Create([FromBody] CreateOrderDto dto) { }
```

---

## BQ4. REST vs SOAP – Key Differences

| Feature | REST | SOAP |
|---------|------|------|
| Protocol | HTTP only | HTTP, SMTP, TCP |
| Data Format | JSON, XML, any | XML only |
| Speed | Faster (lightweight) | Slower (XML overhead) |
| Standards | Flexible | Strict WS-* standards |
| WSDL | No | Yes (contract-first) |
| Security | HTTPS + OAuth | WS-Security (built-in) |
| Statefulness | Stateless | Can be stateful |
| Caching | ✅ Supported | ❌ Not native |
| Best for | Web/mobile APIs, microservices | Enterprise, banking, legacy |

```csharp
// REST API call (simple):
var response = await httpClient.GetAsync("https://api.example.com/orders/1");
var order = await response.Content.ReadFromJsonAsync<Order>();

// SOAP call (complex XML envelope):
/*
POST /OrderService HTTP/1.1
SOAPAction: "GetOrder"

<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <GetOrder xmlns="http://example.com/">
      <OrderId>1</OrderId>
    </GetOrder>
  </soap:Body>
</soap:Envelope>
*/

// RESTful API principles (REST constraints):
// 1. Stateless — each request is self-contained
// 2. Client-Server — separation of concerns
// 3. Cacheable — responses can be cached
// 4. Uniform Interface — consistent resource URIs
// 5. Layered System — client doesn't know if it's hitting a proxy
```

---

## BQ5. How to Implement Pagination in Web API

```csharp
// 1. Define pagination request model
public class PaginationParams
{
    private const int MaxPageSize = 100;
    public int Page { get; set; } = 1;
    private int _pageSize = 10;
    public int PageSize
    {
        get => _pageSize;
        set => _pageSize = value > MaxPageSize ? MaxPageSize : value;
    }
    public string? SortBy { get; set; } = "Id";
    public string SortOrder { get; set; } = "asc";
    public string? Search { get; set; }
}

// 2. Paginated response model
public class PagedResult<T>
{
    public IEnumerable<T> Data { get; set; }
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool HasPreviousPage => Page > 1;
    public bool HasNextPage => Page < TotalPages;
}

// 3. Repository/Service implementation
public async Task<PagedResult<ProductDto>> GetProductsAsync(PaginationParams p)
{
    var query = _dbContext.Products.AsQueryable();

    // Search filter
    if (!string.IsNullOrEmpty(p.Search))
        query = query.Where(x => x.Name.Contains(p.Search));

    // Sort
    query = p.SortOrder.ToLower() == "desc"
        ? query.OrderByDescending(x => EF.Property<object>(x, p.SortBy!))
        : query.OrderBy(x => EF.Property<object>(x, p.SortBy!));

    var total = await query.CountAsync();  // Count before pagination

    // Paginate with Skip + Take
    var items = await query
        .Skip((p.Page - 1) * p.PageSize)   // e.g., Page 3 = skip 20
        .Take(p.PageSize)                   // Take only this page's items
        .Select(x => new ProductDto(x.Id, x.Name, x.Price))
        .ToListAsync();

    return new PagedResult<ProductDto>
    {
        Data = items,
        TotalCount = total,
        Page = p.Page,
        PageSize = p.PageSize
    };
}

// 4. Controller
[HttpGet]
public async Task<ActionResult<PagedResult<ProductDto>>> GetProducts([FromQuery] PaginationParams p)
{
    var result = await _service.GetProductsAsync(p);
    // Add pagination info to response headers
    Response.Headers.Add("X-Total-Count", result.TotalCount.ToString());
    Response.Headers.Add("X-Total-Pages", result.TotalPages.ToString());
    return Ok(result);
}

// 5. Usage: GET /api/products?page=2&pageSize=10&sortBy=Price&sortOrder=desc&search=laptop
```

---

## BQ6. What is Idempotency and Why Does It Matter in APIs?

**Idempotency:** Calling the same API multiple times produces the **same result** as calling it once.

```csharp
// HTTP Method Idempotency:
// GET    → ✅ Idempotent (reading same data every time)
// PUT    → ✅ Idempotent (replacing same resource)
// DELETE → ✅ Idempotent (deleting once = deleting twice)
// POST   → ❌ NOT idempotent (creates new resource each call!)
// PATCH  → ❌ NOT idempotent (incremental changes)

// Problem: Client sends POST, network timeout, client retries → Duplicate order!
// Solution: Idempotency Key

public class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> CreateOrder(
        [FromBody] CreateOrderDto dto,
        [FromHeader(Name = "Idempotency-Key")] string idempotencyKey)
    {
        if (string.IsNullOrEmpty(idempotencyKey))
            return BadRequest("Idempotency-Key header is required");

        // Check if this key was already processed
        var cached = await _cache.GetAsync<OrderDto>($"idempotency:{idempotencyKey}");
        if (cached != null)
            return Ok(cached);  // Return SAME response as before — no duplicate!

        // First time — create the order
        var order = await _orderService.CreateAsync(dto);

        // Cache result for 24 hours (with the idempotency key)
        await _cache.SetAsync($"idempotency:{idempotencyKey}", order,
            TimeSpan.FromHours(24));

        return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
    }
}

// Client sends:
// POST /api/orders
// Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
// { "productId": 1, "qty": 2 }

// If retried with same key → same response, no duplicate order!
```

---

## BQ7. HttpClient Best Practices (IHttpClientFactory)

```csharp
// ❌ WRONG: Creating HttpClient directly (causes socket exhaustion!)
public class OrderService
{
    public async Task<string> GetData()
    {
        using var client = new HttpClient();  // BAD! Creates new socket each time
        return await client.GetStringAsync("https://api.example.com");
    }
}

// ✅ RIGHT: Use IHttpClientFactory (manages connection pooling)

// Register named client
builder.Services.AddHttpClient("PaymentAPI", client =>
{
    client.BaseAddress = new Uri("https://payment.example.com");
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

// Register typed client (preferred)
builder.Services.AddHttpClient<PaymentServiceClient>(client =>
{
    client.BaseAddress = new Uri("https://payment.example.com");
})
.AddPolicyHandler(GetRetryPolicy())    // Polly retry policy
.AddPolicyHandler(GetCircuitBreaker()); // Circuit breaker

public class PaymentServiceClient
{
    private readonly HttpClient _client;
    public PaymentServiceClient(HttpClient client) { _client = client; }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var response = await _client.PostAsJsonAsync("/api/payments", request);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PaymentResult>();
    }
}

// Polly retry policy (Install: Microsoft.Extensions.Http.Polly)
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
    => HttpPolicyExtensions
        .HandleTransientHttpError()    // 5xx, 408
        .OrResult(r => r.StatusCode == HttpStatusCode.TooManyRequests)
        .WaitAndRetryAsync(3, retryAttempt =>
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));  // Exponential backoff

// Circuit Breaker
static IAsyncPolicy<HttpResponseMessage> GetCircuitBreaker()
    => HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30));
```

---

## BQ8. Minimal APIs in .NET 6+ vs Controller-Based APIs

```csharp
// MINIMAL API (Program.cs only — no controllers needed)
// Great for microservices, simple endpoints

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

// Define endpoints directly
app.MapGet("/api/products", async (IProductService service) =>
    await service.GetAllAsync());

app.MapGet("/api/products/{id:int}", async (int id, IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
});

app.MapPost("/api/products", async (CreateProductDto dto, IProductService service) =>
{
    var created = await service.CreateAsync(dto);
    return Results.CreatedAtRoute("GetProduct", new { id = created.Id }, created);
}).WithName("GetProduct")
  .RequireAuthorization()
  .WithOpenApi();

app.MapPut("/api/products/{id}", async (int id, UpdateProductDto dto, IProductService service) =>
{
    await service.UpdateAsync(id, dto);
    return Results.NoContent();
});

app.MapDelete("/api/products/{id}", async (int id, IProductService service) =>
{
    await service.DeleteAsync(id);
    return Results.NoContent();
});

// Group related endpoints
var ordersGroup = app.MapGroup("/api/orders").RequireAuthorization();
ordersGroup.MapGet("/", GetOrders);
ordersGroup.MapPost("/", CreateOrder);

app.Run();

// COMPARISON:
// Controller-based: More structure, better for large apps, easier testing, built-in conventions
// Minimal API: Less boilerplate, faster startup, great for microservices, Lambda-friendly
```

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep | ⭐ = Extra Important*
