# 🔷 .NET Core – Infosys Interview Questions & Answers

---

## Q1. What is Sealed Class and When to Use It?

A **sealed class** cannot be **inherited**. Once a class is marked `sealed`, no other class can derive from it.

```csharp
public sealed class PaymentProcessor
{
    public void ProcessPayment(decimal amount)
    {
        Console.WriteLine($"Processing payment: {amount}");
    }
}

// This will give compile error:
// public class ExtendedProcessor : PaymentProcessor { }  // ERROR!
```

### When to Use Sealed Class:
1. **Security** – prevent someone from overriding sensitive behavior (e.g., cryptography, authentication)
2. **Performance** – runtime can make optimizations (no virtual dispatch)
3. **Design intent** – you don't want the class to be extended (e.g., `String` in .NET is sealed)
4. **Singleton pattern** – to prevent inheritance breaking the pattern
5. **Value objects** – immutable types that shouldn't be extended

```csharp
// Sealed method in a non-sealed class
public class Base
{
    public virtual void Display() { Console.WriteLine("Base"); }
}

public class Derived : Base
{
    public sealed override void Display() { Console.WriteLine("Derived"); }
    // No further override allowed
}
```

---

## Q2. How Can We Handle Exceptions?

```csharp
// Basic try-catch-finally
try
{
    int result = 10 / int.Parse("0");
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"Division error: {ex.Message}");
}
catch (FormatException ex)
{
    Console.WriteLine($"Format error: {ex.Message}");
}
catch (Exception ex)  // Catch-all (least specific last)
{
    Console.WriteLine($"Unexpected error: {ex.Message}");
    throw;  // Re-throw to preserve stack trace
}
finally
{
    // Always executes (cleanup code)
    Console.WriteLine("Cleanup done");
}

// Custom Exception
public class InsufficientFundsException : Exception
{
    public decimal Balance { get; }
    public InsufficientFundsException(decimal balance)
        : base($"Insufficient funds. Current balance: {balance}")
    {
        Balance = balance;
    }
}

// Global Exception Handling in ASP.NET Core (middleware)
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next; _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new { error = "Internal Server Error" });
        }
    }
}

// Register in Program.cs
app.UseMiddleware<GlobalExceptionMiddleware>();
```

---

## Q3. Explain SOLID Principles

**S – Single Responsibility Principle (SRP)**  
A class should have only ONE reason to change.
```csharp
// BAD: Does too much
public class Employee { public void Save() {} public void GenerateReport() {} }

// GOOD: Separated responsibilities
public class Employee { public string Name { get; set; } }
public class EmployeeRepository { public void Save(Employee e) {} }
public class EmployeeReportService { public void GenerateReport(Employee e) {} }
```

**O – Open/Closed Principle (OCP)**  
Open for extension, closed for modification.
```csharp
public abstract class Shape { public abstract double Area(); }
public class Circle : Shape { public double Radius; public override double Area() => Math.PI * Radius * Radius; }
public class Rectangle : Shape { public double Width, Height; public override double Area() => Width * Height; }
// Add new shape WITHOUT modifying existing code
```

**L – Liskov Substitution Principle (LSP)**  
Subclasses must be substitutable for their base class.
```csharp
// If Bird has Fly(), then Penguin (flightless) violates LSP
// Fix: Separate IFlyable interface
public interface IFlyable { void Fly(); }
public class Eagle : IFlyable { public void Fly() {} }
public class Penguin { /* No Fly */ }
```

**I – Interface Segregation Principle (ISP)**  
Clients should not be forced to implement interfaces they don't use.
```csharp
// BAD: Fat interface
public interface IAnimal { void Eat(); void Fly(); void Swim(); }

// GOOD: Segregated
public interface IEatable { void Eat(); }
public interface IFlyable { void Fly(); }
public interface ISwimmable { void Swim(); }
public class Duck : IEatable, IFlyable, ISwimmable { ... }
```

**D – Dependency Inversion Principle (DIP)**  
Depend on abstractions, not concretions.
```csharp
// BAD: High-level depends on low-level
public class OrderService { private SqlOrderRepository _repo = new SqlOrderRepository(); }

// GOOD: Both depend on abstraction
public interface IOrderRepository { void Save(Order order); }
public class OrderService {
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) { _repo = repo; }
}
// Inject SqlOrderRepository or MockOrderRepository via DI
```

---

## Q4. Synchronous and Asynchronous Programming in ASP.NET Core

```csharp
// SYNCHRONOUS - blocks the thread while waiting
public IActionResult GetData()
{
    var data = _service.GetDataFromDb();  // Thread is blocked here
    return Ok(data);
}

// ASYNCHRONOUS - releases thread while waiting (async/await)
public async Task<IActionResult> GetDataAsync()
{
    var data = await _service.GetDataFromDbAsync();  // Thread released to serve other requests
    return Ok(data);
}

// Service implementation
public class DataService
{
    private readonly HttpClient _client;

    // Sync (avoid in ASP.NET Core)
    public string GetData() => _client.GetStringAsync("https://api.example.com").Result;

    // Async (preferred)
    public async Task<string> GetDataAsync()
        => await _client.GetStringAsync("https://api.example.com");
}

// Running multiple async ops in parallel
public async Task<IActionResult> GetAllAsync()
{
    var task1 = _service.GetUsersAsync();
    var task2 = _service.GetOrdersAsync();
    await Task.WhenAll(task1, task2);  // Both run in parallel
    return Ok(new { Users = task1.Result, Orders = task2.Result });
}
```

### Key Rules:
- Use `async`/`await` all the way up the call chain
- Never use `.Result` or `.Wait()` – causes **deadlocks**
- Use `ConfigureAwait(false)` in library code
- `Task.WhenAll()` for parallel operations, `Task.WhenAny()` for first-to-complete

---

## Q5. How Does Dependency Injection Change in .NET Core vs .NET Framework?

### .NET Framework (old way)
```csharp
// Had to use third-party containers: Unity, Autofac, Ninject
// Manual setup in Global.asax or bootstrapper
var container = new UnityContainer();
container.RegisterType<IService, MyService>();
GlobalConfiguration.Configuration.DependencyResolver = 
    new UnityDependencyResolver(container);
```

### .NET Core (built-in DI)
```csharp
// Program.cs / Startup.cs - clean, built-in
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddTransient<IEmailService, EmailService>();
builder.Services.AddSingleton<IConfigService, ConfigService>();

// Constructor injection (no attribute needed)
public class OrderController : ControllerBase
{
    private readonly IOrderService _orderService;
    public OrderController(IOrderService orderService)
    {
        _orderService = orderService;
    }
}
```

### Key Differences:
| Aspect | .NET Framework | .NET Core |
|--------|---------------|-----------|
| DI Container | Third-party required | Built-in |
| Configuration | Web.config / XML | appsettings.json / environment |
| Lifetime | Manual setup | AddScoped/Transient/Singleton |
| Testing | Harder to mock | Easy with constructor injection |

---

## Q6. Code Changes for DI During .NET Core Migration

```csharp
// 1. Replace third-party container references
// Remove: using Unity; using Autofac;
// Add: using Microsoft.Extensions.DependencyInjection;

// 2. Register services in Program.cs (formerly Startup.cs)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddSingleton<IConfiguration>(builder.Configuration);

// 3. Update service constructors (remove [Inject] attributes)
// OLD (Unity):
// [InjectionConstructor]
// public OrderService([Dependency] IRepository repo) { }

// NEW (.NET Core):
public OrderService(IRepository repo) { _repo = repo; }

// 4. Use IOptions<T> for configuration injection
public class EmailSettings { public string SmtpServer { get; set; } }
// appsettings.json: { "EmailSettings": { "SmtpServer": "smtp.gmail.com" } }
builder.Services.Configure<EmailSettings>(builder.Configuration.GetSection("EmailSettings"));
// Inject: IOptions<EmailSettings> options → options.Value.SmtpServer
```

---

## Q7. Supporting a Library in Both .NET Framework and .NET Core

### Approach: Multi-targeting in .csproj
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <!-- Target both frameworks -->
    <TargetFrameworks>net48;net8.0</TargetTargetFrameworks>
  </PropertyGroup>

  <!-- Framework-specific packages -->
  <ItemGroup Condition="'$(TargetFramework)' == 'net48'">
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  </ItemGroup>
  <ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
    <PackageReference Include="System.Text.Json" Version="8.0.0" />
  </ItemGroup>
</Project>
```

### Using Preprocessor Directives in Code:
```csharp
public class MyLibrary
{
    public string SerializeData(object obj)
    {
#if NET48
        return Newtonsoft.Json.JsonConvert.SerializeObject(obj);
#else
        return System.Text.Json.JsonSerializer.Serialize(obj);
#endif
    }
}
```

### Best Practice: .NET Standard
- Create library targeting **netstandard2.0** – compatible with both!
- Works in .NET Framework 4.6.1+ and .NET Core 2.0+

---

## Q8. Architecture of Your Application

**Sample Layered Architecture (Clean Architecture):**
```
Solution/
├── API Layer           (Controllers, Middleware, Filters)
├── Application Layer   (Services, DTOs, Interfaces, CQRS Commands/Queries)
├── Domain Layer        (Entities, Domain Models, Business Rules)
└── Infrastructure Layer (Repositories, DbContext, External Services)
```

```
Presentation (API)
    ↓ calls
Application (Services/Use Cases)
    ↓ depends on interfaces
Domain (Business Entities)
    ↑ implemented by
Infrastructure (DB, External APIs)
```

---

## Q9 & Q10. Code for Dependency Inversion Principle

```csharp
// ===== Interfaces (Abstractions) =====
public interface INotificationService
{
    Task SendAsync(string to, string message);
}

public interface IOrderRepository
{
    Task<Order> GetByIdAsync(int id);
    Task SaveAsync(Order order);
}

// ===== Implementations =====
public class EmailNotificationService : INotificationService
{
    public async Task SendAsync(string to, string message)
    {
        Console.WriteLine($"Email sent to {to}: {message}");
        await Task.CompletedTask;
    }
}

public class SqlOrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    public SqlOrderRepository(AppDbContext context) { _context = context; }
    public async Task<Order> GetByIdAsync(int id) => await _context.Orders.FindAsync(id);
    public async Task SaveAsync(Order order) { _context.Orders.Add(order); await _context.SaveChangesAsync(); }
}

// ===== High-Level Module (depends on abstractions) =====
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly INotificationService _notificationService;

    // Dependencies injected - NOT created inside
    public OrderService(IOrderRepository repository, INotificationService notificationService)
    {
        _repository = repository;
        _notificationService = notificationService;
    }

    public async Task PlaceOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);
        await _notificationService.SendAsync(order.CustomerEmail, "Order placed successfully!");
    }
}

// ===== Registration (Program.cs) =====
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<INotificationService, EmailNotificationService>();
builder.Services.AddScoped<OrderService>();
```

---

## Q11. Can We Improve Application Performance Using Middleware?

Yes! Middleware can significantly improve performance:

```csharp
// 1. Response Caching Middleware
builder.Services.AddResponseCaching();
app.UseResponseCaching();

[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any)]
public IActionResult GetProducts() => Ok(_service.GetProducts());

// 2. Response Compression Middleware
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
});
app.UseResponseCompression();

// 3. Custom Performance Logging Middleware
public class PerformanceMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceMiddleware> _logger;

    public PerformanceMiddleware(RequestDelegate next, ILogger<PerformanceMiddleware> logger)
    {
        _next = next; _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        await _next(context);
        sw.Stop();
        if (sw.ElapsedMilliseconds > 500)
            _logger.LogWarning($"Slow request: {context.Request.Path} took {sw.ElapsedMilliseconds}ms");
    }
}

// 4. Short-circuit unnecessary processing
app.Use(async (context, next) =>
{
    if (context.Request.Path == "/health")
    {
        await context.Response.WriteAsync("Healthy");
        return;  // Short-circuit – don't call next middleware
    }
    await next();
});
```

---

## Q12, Q20, Q23. How to Handle Exceptions (Complete)

See Q2 above for complete answer. Additional patterns:

```csharp
// Exception Filter (ASP.NET Core)
public class ApiExceptionFilter : IExceptionFilter
{
    private readonly ILogger _logger;
    public ApiExceptionFilter(ILogger<ApiExceptionFilter> logger) { _logger = logger; }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Unhandled exception");
        context.Result = new ObjectResult(new { error = context.Exception.Message })
        {
            StatusCode = 500
        };
        context.ExceptionHandled = true;
    }
}

// Register globally
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ApiExceptionFilter>();
});

// UseExceptionHandler (built-in)
app.UseExceptionHandler("/error");
```

---

## Q13. Have You Used Filters in Exception Handling?

**Filters** in ASP.NET Core run before/after specific stages in the request pipeline:

```csharp
// Types of Filters:
// Authorization → Resource → Action → Exception → Result

// Exception Filter
public class CustomExceptionFilter : ExceptionFilterAttribute
{
    public override void OnException(ExceptionContext context)
    {
        var response = new { message = "An error occurred", detail = context.Exception.Message };
        context.Result = new JsonResult(response) { StatusCode = 500 };
        context.ExceptionHandled = true;
    }
}

// Apply to specific controller
[CustomExceptionFilter]
public class OrdersController : ControllerBase { }

// Action Filter
public class LogActionFilter : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
        => Console.WriteLine($"Before action: {context.ActionDescriptor.DisplayName}");

    public override void OnActionExecuted(ActionExecutedContext context)
        => Console.WriteLine($"After action: {context.ActionDescriptor.DisplayName}");
}
```

---

## Q14. Is It a Good Practice to Use Try/Catch?

### When to USE try/catch:
- When you can **meaningfully handle** the exception
- For **specific exceptions** (not generic Exception)
- When you need to **log and continue** processing
- At **application boundaries** (API endpoints, background jobs)

### When NOT to use try/catch:
- Don't catch to **suppress errors silently** – this hides bugs
- Don't use for **flow control** – use `if` checks instead
- Don't catch `Exception` without re-throwing or logging
- Avoid **empty catch blocks**

```csharp
// BAD practices:
try { ... } catch { }  // Empty catch - hides errors
try { ... } catch (Exception) { }  // Same issue
// Using exceptions for flow control:
try { int.Parse(input); } catch { /* use default */ }  // Use int.TryParse instead

// GOOD practices:
// 1. Catch specific exceptions
try { await _repo.SaveAsync(entity); }
catch (DbUpdateConcurrencyException ex) { throw new ConflictException("Record was modified", ex); }
catch (DbUpdateException ex) { _logger.LogError(ex, "DB save failed"); throw; }

// 2. Use TryParse patterns instead of try/catch
if (int.TryParse(input, out int value))
    ProcessValue(value);

// 3. Global handler for unhandled exceptions
app.UseExceptionHandler(builder =>
{
    builder.Run(async context => {
        var ex = context.Features.Get<IExceptionHandlerFeature>();
        _logger.LogError(ex?.Error, "Unhandled exception");
        await context.Response.WriteAsJsonAsync(new { error = "Server error" });
    });
});
```

---

## Q16-Q17. Code Review Process / PR Review

### What to Check in Code Review:
1. **Correctness** – Does it solve the problem? Edge cases handled?
2. **SOLID principles** – Single responsibility, proper abstractions
3. **Security** – SQL injection, XSS, sensitive data exposed?
4. **Performance** – N+1 queries, unnecessary DB calls, blocking async?
5. **Error handling** – Exceptions caught and logged properly?
6. **Naming conventions** – Meaningful variable/method names
7. **Unit tests** – Are tests present and covering edge cases?
8. **DRY principle** – No repeated code (extract to methods/services)
9. **Null checks** – NullReferenceException prevention
10. **Logging** – Adequate logging with proper log levels
11. **Configuration** – No hardcoded connection strings/secrets
12. **Comments/documentation** – Complex logic explained

---

## Q18. Explain a Design Pattern and Its Benefit

### Most Common Patterns:

**Repository Pattern**
```csharp
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}

public class GenericRepository<T> : IRepository<T> where T : class
{
    private readonly DbContext _context;
    public GenericRepository(DbContext context) { _context = context; }
    public async Task<T> GetByIdAsync(int id) => await _context.Set<T>().FindAsync(id);
    // ... other methods
}
```

**Factory Pattern**
```csharp
public interface IPaymentProcessor { void Process(decimal amount); }
public class CreditCardProcessor : IPaymentProcessor { public void Process(decimal amount) {} }
public class PayPalProcessor : IPaymentProcessor { public void Process(decimal amount) {} }

public static class PaymentFactory
{
    public static IPaymentProcessor Create(string type) => type switch
    {
        "CreditCard" => new CreditCardProcessor(),
        "PayPal" => new PayPalProcessor(),
        _ => throw new ArgumentException($"Unknown type: {type}")
    };
}
```

**Singleton Pattern**
```csharp
public class ConfigurationManager
{
    private static ConfigurationManager _instance;
    private static readonly object _lock = new();
    
    private ConfigurationManager() { }
    
    public static ConfigurationManager Instance
    {
        get
        {
            lock (_lock)
            {
                return _instance ??= new ConfigurationManager();
            }
        }
    }
}
```

---

## Q19. .NET Framework vs .NET Core

| Feature | .NET Framework | .NET Core / .NET 5+ |
|---------|---------------|----------------------|
| Platform | Windows only | Cross-platform (Win/Linux/Mac) |
| Performance | Slower | Much faster (optimized) |
| Hosting | IIS only | IIS, Kestrel, Docker, self-hosted |
| Deployment | GAC, xcopy | Self-contained, Docker |
| Open Source | Partially | Fully open source |
| Latest version | 4.8 (end-of-life) | .NET 8 / .NET 9 |
| Modular | Monolithic | Modular (NuGet packages) |
| CLI support | Limited | Full dotnet CLI |
| microservices | Not designed for | First-class support |

---

## Q20. Asynchronous Programming (async/await)

```csharp
// Task-based Asynchronous Pattern (TAP)
public async Task<List<Product>> GetProductsAsync(int categoryId)
{
    // Await I/O operations
    var products = await _dbContext.Products
        .Where(p => p.CategoryId == categoryId)
        .ToListAsync();
    
    // Parallel HTTP calls
    var tasks = products.Select(p => FetchProductDetailsAsync(p.Id));
    var details = await Task.WhenAll(tasks);
    
    return products;
}

// CancellationToken support
public async Task<string> GetDataAsync(CancellationToken cancellationToken)
{
    var response = await _httpClient.GetAsync("https://api.example.com", cancellationToken);
    return await response.Content.ReadAsStringAsync(cancellationToken);
}

// ValueTask for performance (avoid heap allocation when result is cached)
public ValueTask<int> GetCachedValueAsync()
{
    if (_cache.TryGetValue("key", out int value))
        return new ValueTask<int>(value);  // No allocation
    return new ValueTask<int>(FetchFromDbAsync());
}
```

---

## Q21. Generics in C#

```csharp
// Generic Class
public class Stack<T>
{
    private List<T> _items = new();
    public void Push(T item) => _items.Add(item);
    public T Pop() { var item = _items[^1]; _items.RemoveAt(_items.Count - 1); return item; }
}

// Generic Method
public T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) > 0 ? a : b;

// Generic Constraints
public class Repository<T> where T : class, new()
{
    public T Create() => new T();
}

// Generic Interface
public interface IConverter<TInput, TOutput>
{
    TOutput Convert(TInput input);
}

// Usage
var intStack = new Stack<int>();
intStack.Push(1);
var result = Max(10, 20);  // Returns 20
```

### Benefits:
- **Type safety** at compile time
- **Code reusability** – one class/method for multiple types
- **Performance** – avoids boxing/unboxing for value types

---

## Q22. Microservices Architecture

**Definition:** Architectural style where app is built as a collection of small, independently deployable services.

### Key Concepts:
```
API Gateway → [Service A] [Service B] [Service C]
                ↓              ↓             ↓
           [DB_A]         [DB_B]        [DB_C]
```

### Communication:
```csharp
// Synchronous: REST API via HttpClient
var response = await _httpClient.GetAsync("https://order-service/api/orders");

// Asynchronous: Message Queue (RabbitMQ / Azure Service Bus)
await _serviceBus.PublishAsync(new OrderCreatedEvent { OrderId = 1 });
```

### Benefits: Independent deployment, scalability, fault isolation  
### Challenges: Network latency, distributed transactions, complexity

---

## Q23. Dependency Injection

See Q5 for DI details. Lifetime types:

```csharp
// TRANSIENT - New instance every time it's requested
builder.Services.AddTransient<IEmailService, EmailService>();
// Use for: lightweight, stateless services

// SCOPED - One instance per HTTP request
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
// Use for: DbContext, per-request state

// SINGLETON - One instance for app lifetime
builder.Services.AddSingleton<ICacheService, CacheService>();
// Use for: caches, configuration, heavy resources
```

---

## Q24. .NET Core Types of Lifetime Instances

| Lifetime | Created | Disposed | Use Case |
|----------|---------|----------|----------|
| **Transient** | Every request | After use | Stateless services |
| **Scoped** | Per HTTP request | End of request | DbContext, Unit of Work |
| **Singleton** | First request | App shutdown | Cache, Config, Logger |

⚠️ **Captive Dependency Problem:** Never inject Transient/Scoped into Singleton!

---

## Q25. Net Core Middleware

```csharp
// Middleware pipeline processes requests in ORDER
app.UseExceptionHandler();    // 1st - catch all exceptions
app.UseHttpsRedirection();    // 2nd
app.UseStaticFiles();         // 3rd
app.UseRouting();             // 4th
app.UseAuthentication();      // 5th - who are you?
app.UseAuthorization();       // 6th - what can you do?
app.MapControllers();         // 7th

// Custom Middleware
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    public RequestLoggingMiddleware(RequestDelegate next) { _next = next; }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine($"→ {context.Request.Method} {context.Request.Path}");
        await _next(context);  // Call next middleware
        Console.WriteLine($"← {context.Response.StatusCode}");
    }
}

// Extension method for clean registration
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(this IApplicationBuilder app)
        => app.UseMiddleware<RequestLoggingMiddleware>();
}
```

---

## Q26. Why Do We Need .NET Core and Its Advantages?

1. **Cross-platform** – runs on Windows, Linux, macOS
2. **High performance** – one of the fastest web frameworks (TechEmpower benchmarks)
3. **Modular** – include only what you need (no monolithic System.Web)
4. **Open Source** – community-driven, transparent
5. **Cloud-native** – Docker, Kubernetes, microservices support
6. **Built-in DI** – no third-party containers needed
7. **Unified platform** – .NET 5+ merges Framework and Core
8. **Side-by-side versioning** – multiple versions on same machine
9. **Better tooling** – dotnet CLI, hot reload
10. **Modern C# features** – async streams, records, nullable references

---

## Q27. LINQ and Left Join in LINQ

```csharp
// Basic LINQ queries
var employees = dbContext.Employees.ToList();

// WHERE
var seniors = employees.Where(e => e.Salary > 50000).ToList();

// SELECT (projection)
var names = employees.Select(e => new { e.Name, e.Department }).ToList();

// ORDER BY
var ordered = employees.OrderByDescending(e => e.Salary).ToList();

// GROUP BY
var byDept = employees.GroupBy(e => e.DeptID)
                      .Select(g => new { Dept = g.Key, Count = g.Count() });

// INNER JOIN
var result = from e in dbContext.Employees
             join d in dbContext.Departments on e.DeptID equals d.DeptID
             select new { e.Name, d.DeptName };

// LEFT JOIN (GroupJoin + SelectMany)
var leftJoinResult = from e in dbContext.Employees
                     join d in dbContext.Departments on e.DeptID equals d.DeptID into deptGroup
                     from dept in deptGroup.DefaultIfEmpty()  // DefaultIfEmpty = LEFT JOIN
                     select new {
                         EmployeeName = e.Name,
                         DepartmentName = dept != null ? dept.DeptName : "No Department"
                     };

// Method syntax for left join
var methodSyntax = dbContext.Employees
    .GroupJoin(
        dbContext.Departments,
        emp => emp.DeptID,
        dept => dept.DeptID,
        (emp, depts) => new { emp, depts })
    .SelectMany(
        x => x.depts.DefaultIfEmpty(),
        (x, dept) => new { x.emp.Name, DeptName = dept != null ? dept.DeptName : "None" });
```

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep*

---

---

# ⭐ BONUS – Important Questions (Added by Interviewer Pattern Analysis)

---

## BQ1. Value Types vs Reference Types in C#

```csharp
// VALUE TYPES: Stored on STACK, copied on assignment
// int, double, float, bool, char, struct, enum
int a = 10;
int b = a;   // b gets a COPY
b = 20;
Console.WriteLine(a);  // Still 10 — independent copy

struct Point { public int X, Y; }
Point p1 = new Point { X = 1, Y = 2 };
Point p2 = p1;   // Copied
p2.X = 99;
Console.WriteLine(p1.X);  // Still 1

// REFERENCE TYPES: Stored on HEAP, variable holds reference (pointer)
// class, string, array, interface, delegate
class Employee { public string Name; }

Employee e1 = new Employee { Name = "Alice" };
Employee e2 = e1;   // Both point to SAME object
e2.Name = "Bob";
Console.WriteLine(e1.Name);  // "Bob" — same object!

// Exception: string is reference type but IMMUTABLE (behaves like value type)
string s1 = "Hello";
string s2 = s1;
s2 = "World";         // Creates new string object
Console.WriteLine(s1); // Still "Hello"

// Passing to methods:
void ModifyValue(int x) { x = 99; }      // Copy — original unchanged
void ModifyRef(Employee emp) { emp.Name = "Changed"; }  // Reference — original changes!
void ModifyRefParam(ref int x) { x = 99; }  // Explicitly pass by reference

int num = 5;
ModifyValue(num);     // num still 5
ModifyRefParam(ref num); // num is now 99
```

| | Value Type | Reference Type |
|--|-----------|----------------|
| Stored in | Stack | Heap |
| Holds | Actual value | Memory address |
| Copy behavior | Full copy | Copy of reference |
| Null allowed | ❌ (unless `int?`) | ✅ |
| Examples | int, bool, struct | class, array, string |

---

## BQ2. Garbage Collection (GC) in .NET

The **Garbage Collector** automatically manages memory — allocates and frees heap objects you no longer reference.

```csharp
// GC Generations (performance optimization)
// Gen 0: New, short-lived objects (collected most often)
// Gen 1: Objects that survived Gen 0 collection
// Gen 2: Long-lived objects (collected least often, expensive)

// Objects are promoted: Gen0 → Gen1 → Gen2 if they survive GC

// Finalize vs Dispose:
// Finalizer (~): Called by GC (non-deterministic)
// Dispose(): Called explicitly (deterministic cleanup)

// IDisposable pattern (for unmanaged resources: files, DB connections, streams)
public class DatabaseConnection : IDisposable
{
    private SqlConnection _connection;
    private bool _disposed = false;

    public DatabaseConnection(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
        _connection.Open();
    }

    // Called by GC if Dispose() wasn't called (safety net)
    ~DatabaseConnection()
    {
        Dispose(false);
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);  // Tell GC: no need to call finalizer
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                _connection?.Close();    // Managed resource
                _connection?.Dispose();
            }
            // Unmanaged resources would be freed here regardless of 'disposing'
            _disposed = true;
        }
    }
}

// using statement auto-calls Dispose()
using (var conn = new DatabaseConnection("Server=..."))
{
    // Use connection
}  // Dispose() called automatically here

// C# 8+ using declaration
using var conn = new DatabaseConnection("Server=...");
// Dispose() called at end of scope

// Manual GC (avoid in production — trust the GC!)
GC.Collect();           // Force collection (not recommended)
GC.Collect(0);          // Collect Gen 0 only
GC.GetTotalMemory(false); // Get current heap size
```

---

## BQ3. Boxing and Unboxing

**Boxing:** Converting a value type → reference type (wraps in object on heap)  
**Unboxing:** Converting a reference type back → value type (extracts from heap)

```csharp
// BOXING (implicit): value type → object (heap allocation happens!)
int num = 42;
object boxed = num;     // Boxing: int copied to heap, 'boxed' holds reference
Console.WriteLine(boxed); // 42

// UNBOXING (explicit): object → value type
int unboxed = (int)boxed;  // Unboxing: must cast to correct type!
// InvalidCastException if wrong type:
// double d = (double)boxed;  // ❌ Error!

// Performance problem: boxing creates heap allocation + GC pressure
// BAD: ArrayList uses boxing for value types
System.Collections.ArrayList list = new System.Collections.ArrayList();
list.Add(1);    // Boxing!
list.Add(2);    // Boxing!
list.Add(3);    // Boxing!
int x = (int)list[0];  // Unboxing!

// GOOD: Use generics to avoid boxing
List<int> genericList = new List<int>();
genericList.Add(1);   // No boxing! (type-safe, stored as int)
genericList.Add(2);
int y = genericList[0];  // No unboxing!

// Real-world boxing scenario to watch out for:
// Dictionary with value type values
var dict = new Dictionary<string, int>();
dict["count"] = 5;   // No boxing (generic)

// String.Format can cause boxing:
string s1 = string.Format("{0}", 42);  // int 42 gets boxed!
string s2 = $"{42}";   // Also boxes... but compiler optimizes
string s3 = 42.ToString();  // Best: no boxing
```

---

## BQ4. Delegates and Events in C#

A **delegate** is a type-safe function pointer — holds reference to a method.  
An **event** is a delegate wrapper that restricts who can invoke it.

```csharp
// Delegate declaration
public delegate int MathOperation(int a, int b);

// Methods matching delegate signature
int Add(int a, int b) => a + b;
int Multiply(int a, int b) => a * b;

// Using delegates
MathOperation op = Add;
Console.WriteLine(op(3, 4));  // 7
op = Multiply;
Console.WriteLine(op(3, 4));  // 12

// Multicast delegate (multiple methods)
MathOperation multiOp = Add;
multiOp += Multiply;  // Chained (last result returned)
multiOp(3, 4);        // Calls both Add and Multiply

// Built-in delegate types (avoid declaring custom)
Func<int, int, int> add = (a, b) => a + b;      // Has return value
Action<string> print = msg => Console.WriteLine(msg);  // No return (void)
Predicate<int> isEven = n => n % 2 == 0;        // Returns bool

// EVENTS: Publisher-Subscriber pattern
public class OrderService
{
    // Event declaration using EventHandler
    public event EventHandler<OrderEventArgs> OrderPlaced;

    public void PlaceOrder(Order order)
    {
        // Place order logic...
        // Raise event (notify subscribers)
        OrderPlaced?.Invoke(this, new OrderEventArgs { Order = order });
    }
}

public class OrderEventArgs : EventArgs
{
    public Order Order { get; set; }
}

// Subscriber
public class EmailService
{
    public EmailService(OrderService orderService)
    {
        orderService.OrderPlaced += OnOrderPlaced;  // Subscribe
    }

    private void OnOrderPlaced(object sender, OrderEventArgs e)
    {
        Console.WriteLine($"Email sent for order: {e.Order.Id}");
    }
}

// Usage
var orderService = new OrderService();
var emailService = new EmailService(orderService);  // Subscribes
orderService.PlaceOrder(new Order { Id = 1 });      // Triggers event → email sent
```

---

## BQ5. IEnumerable vs IQueryable vs IList

```csharp
// IEnumerable<T>: In-memory iteration (LINQ to Objects)
// Executes in C# memory — good for collections already loaded
IEnumerable<Employee> emps = employees.Where(e => e.Salary > 50000);
// Translates to: loads ALL employees from DB, then filters in memory ❌

// IQueryable<T>: SQL generation (LINQ to SQL/EF)
// Builds expression tree — executes on database — only fetches needed data
IQueryable<Employee> query = dbContext.Employees.Where(e => e.Salary > 50000);
// Translates to: SELECT * FROM Employees WHERE Salary > 50000 ✅

// Practical difference with Entity Framework:
// IQueryable — deferred, SQL is built
IQueryable<Employee> q1 = dbContext.Employees
    .Where(e => e.DeptID == 2)
    .OrderBy(e => e.Name);
// SQL built so far: SELECT * FROM Employees WHERE DeptID=2 ORDER BY Name

var result = q1.Take(10).ToList();
// Final SQL: SELECT TOP 10 * FROM Employees WHERE DeptID=2 ORDER BY Name
// Only 10 rows transferred from DB!

// IEnumerable — already executed in memory:
IEnumerable<Employee> e1 = dbContext.Employees.ToList()  // ALL rows from DB!
    .Where(e => e.DeptID == 2)   // Filter in C# memory
    .OrderBy(e => e.Name)
    .Take(10);                    // Take in C# memory

// IList<T>: In-memory, indexed access, add/remove
IList<Employee> list = new List<Employee>();
list.Add(new Employee());
list.RemoveAt(0);
int count = list.Count;
var first = list[0];  // Index access

// Summary:
// IEnumerable → Use when data is already in memory
// IQueryable  → Use when building queries against DB (EF/LINQ to SQL)
// IList       → Use when you need index access and modification
```

---

## BQ6. ref vs out vs in Parameters

```csharp
// ref: Pass by reference — variable MUST be initialized before passing
void DoubleIt(ref int value)
{
    value *= 2;  // Modifies original
}
int num = 5;
DoubleIt(ref num);
Console.WriteLine(num);  // 10

// out: Pass by reference — variable NEED NOT be initialized, MUST be assigned inside method
bool TryDivide(int a, int b, out int result)
{
    if (b == 0) { result = 0; return false; }
    result = a / b;
    return true;
}
// result doesn't need to be initialized
if (TryDivide(10, 2, out int result))
    Console.WriteLine(result);  // 5

// TryParse pattern uses out
if (int.TryParse("123", out int parsed))
    Console.WriteLine(parsed);  // 123

// in: Pass by reference but READ ONLY (performance: avoids copy of large structs)
struct LargeStruct { public double[] Data; }
void Process(in LargeStruct data)
{
    // data.Data = null;  // ❌ Error — read only!
    Console.WriteLine(data.Data.Length);
}

// Summary:
// ref  → in and out, must init before
// out  → out only, must assign inside
// in   → in only (read-only reference, no copy)
```

---

## BQ7. Records in C# (C# 9+)

**Records** are immutable reference types designed for data storage. They auto-generate `Equals`, `GetHashCode`, `ToString`, and a copy constructor.

```csharp
// Record declaration (all properties are init-only by default)
public record Person(string FirstName, string LastName, int Age);

var p1 = new Person("John", "Doe", 30);
var p2 = new Person("John", "Doe", 30);

// Value equality (not reference equality like class)
Console.WriteLine(p1 == p2);       // True (same data)
Console.WriteLine(p1.Equals(p2));  // True

// ToString auto-generated
Console.WriteLine(p1); // Person { FirstName = John, LastName = Doe, Age = 30 }

// Non-destructive mutation with 'with'
var p3 = p1 with { Age = 31 };  // Creates new record with Age changed
Console.WriteLine(p1.Age);  // Still 30 (immutable)
Console.WriteLine(p3.Age);  // 31

// Positional deconstruction
var (firstName, lastName, age) = p1;

// Record class vs record struct
public record class ProductRecord(int Id, string Name, decimal Price);  // Reference type
public record struct PointRecord(int X, int Y);  // Value type (C# 10+)

// Record with additional members
public record Order(int Id, DateTime PlacedAt)
{
    public string FormattedDate => PlacedAt.ToString("dd-MMM-yyyy");
    
    // Can add validation
    public Order
    {
        if (Id <= 0) throw new ArgumentException("Invalid ID");
    }
}

// When to use records vs class vs struct:
// record  → DTOs, API responses, immutable data models
// class   → Mutable entities, services, complex behavior
// struct  → Small value objects (Point, Money, Color)
```

---

## BQ8. yield Return and Lazy Evaluation

The `yield` keyword creates an **iterator** — generates values one at a time instead of creating a full collection.

```csharp
// Without yield: Creates entire list in memory first
List<int> GetNumbers(int max)
{
    var result = new List<int>();
    for (int i = 0; i <= max; i++)
        if (i % 2 == 0)
            result.Add(i);
    return result;  // All even numbers created before any consumed
}

// With yield: Generates values LAZILY (on demand)
IEnumerable<int> GetEvenNumbers(int max)
{
    for (int i = 0; i <= max; i++)
        if (i % 2 == 0)
            yield return i;  // Returns one value, pauses, resumes on next iteration
}

// If consumer only needs first 3 numbers, only 3 are generated!
foreach (var n in GetEvenNumbers(1000000).Take(3))
    Console.WriteLine(n);  // Only generates 0, 2, 4 — not all 500001!

// Real-world use: reading large files line by line
IEnumerable<string> ReadLargeFile(string path)
{
    using var reader = new StreamReader(path);
    string line;
    while ((line = reader.ReadLine()) != null)
        yield return line;  // One line at a time, file not fully loaded
}

// Processing in chunks
IEnumerable<IEnumerable<T>> Chunk<T>(IEnumerable<T> source, int size)
{
    var chunk = new List<T>(size);
    foreach (var item in source)
    {
        chunk.Add(item);
        if (chunk.Count == size)
        {
            yield return chunk;
            chunk = new List<T>(size);
        }
    }
    if (chunk.Count > 0) yield return chunk;
}

// yield break: Stop iteration
IEnumerable<int> Until(int limit)
{
    for (int i = 0; ; i++)
    {
        if (i > limit) yield break;  // Stop
        yield return i;
    }
}
```

---

## BQ9. Null Safety – Nullable Reference Types, Null-Coalescing, Null-Conditional

```csharp
// Enable nullable reference types (C# 8+) in .csproj:
// <Nullable>enable</Nullable>

// Nullable value types
int? nullableInt = null;
if (nullableInt.HasValue) Console.WriteLine(nullableInt.Value);
Console.WriteLine(nullableInt ?? 0);  // Default if null

// Null-conditional operator (?.)
string? name = null;
int? length = name?.Length;    // null (no NullReferenceException!)
string upper = name?.ToUpper() ?? "UNKNOWN";

// Chain null-conditionals
var city = customer?.Address?.City ?? "Unknown City";
int count = order?.Items?.Count ?? 0;

// Null-coalescing assignment (??=)
List<string>? items = null;
items ??= new List<string>();  // Only assigns if items is null
items.Add("First");

// Null-forgiving operator (!) — suppress warning (use carefully)
string value = GetValueThatMightBeNull()!;  // "I know it's not null"

// Guard clauses pattern
public void ProcessOrder(Order? order)
{
    // Guard at top — fail fast
    ArgumentNullException.ThrowIfNull(order);         // .NET 6+
    // or: if (order == null) throw new ArgumentNullException(nameof(order));
    
    ArgumentNullException.ThrowIfNull(order.Customer);
    
    if (string.IsNullOrWhiteSpace(order.Customer.Email))
        throw new ArgumentException("Email required");
    
    // Now safely work with order
    ProcessValidOrder(order);
}

// Pattern matching with null
object obj = GetSomeObject();
if (obj is string s && s.Length > 0)
    Console.WriteLine(s.ToUpper());

// Switch expression with null
string result = order switch
{
    null => "No order",
    { Status: "Pending" } => "Order is pending",
    { Status: "Shipped" } o => $"Order {o.Id} shipped",
    _ => "Unknown status"
};
```

---

## BQ10. String Handling and StringBuilder

```csharp
// String is IMMUTABLE — every modification creates a NEW object
string s = "Hello";
s += " World";   // Creates new string "Hello World", old "Hello" is garbage
s = s.Replace("Hello", "Hi");  // Another new string

// String methods
string text = "  Hello, Infosys!  ";
Console.WriteLine(text.Trim());           // "Hello, Infosys!"
Console.WriteLine(text.ToUpper());        // "  HELLO, INFOSYS!  "
Console.WriteLine(text.Contains("sys")); // True
Console.WriteLine(text.StartsWith("  Hello")); // True
Console.WriteLine(text.Replace(",", ";")); // "  Hello; Infosys!  "
Console.WriteLine(text.Split(',')[1]);    // " Infosys!  "
Console.WriteLine(text.IndexOf("I"));    // 8

string[] words = { "C#", ".NET", "Core" };
string joined = string.Join(", ", words);  // "C#, .NET, Core"

// String comparison (always use StringComparison!)
bool equal = string.Equals("hello", "HELLO", StringComparison.OrdinalIgnoreCase);
int cmp = string.Compare("apple", "banana", StringComparison.Ordinal);

// String interpolation vs Format
string name = "Omkar";
int age = 25;
string s1 = $"Name: {name}, Age: {age}";          // Best (modern)
string s2 = string.Format("Name: {0}, Age: {1}", name, age);

// StringBuilder: for repeated string concatenation (MUTABLE, no new objects)
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append($"Item {i}, ");   // Modifies same buffer — no garbage!
string result = sb.ToString();

sb.AppendLine("New line");
sb.Insert(0, "START: ");
sb.Replace("Item", "Product");
sb.Remove(0, 7);  // Remove 7 chars from position 0

// When to use StringBuilder:
// String concat in loop → ALWAYS use StringBuilder
// < 5 concatenations → string is fine

// Verbatim string
string path = @"C:\Users\Omkar\Documents\file.txt";  // No need to escape \
string multiLine = @"Line 1
Line 2
Line 3";

// Raw string literals (C# 11+)
string json = """
{
    "name": "Omkar",
    "age": 25
}
""";
```

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep | ⭐ = Extra Important*
