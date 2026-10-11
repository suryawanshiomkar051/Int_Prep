# 🧩 Generic / General – Infosys Interview Questions & Answers

---

## Q1. Content Technology – jQuery

**jQuery** is a fast, small, and feature-rich JavaScript library that simplifies:
- DOM manipulation
- Event handling
- AJAX calls
- Animations

```javascript
// CDN include
// <script src="https://code.jquery.com/jquery-3.7.1.min.js"></script>

// DOM Selection
$('#myId')              // By ID
$('.myClass')           // By class
$('p')                  // By tag
$('div.container > p')  // CSS selectors

// DOM Manipulation
$('#title').text('Hello World');          // Set text
$('#div1').html('<b>Bold</b>');           // Set HTML
$('#input1').val('New Value');            // Set input value
$('#btn').addClass('active');             // Add class
$('.item').removeClass('hidden');         // Remove class
$('#section').hide();                     // Hide element
$('#section').show();                     // Show element
$('#modal').fadeIn(300);                  // Fade in with animation

// Event Handling
$('#myBtn').on('click', function() {
    alert('Button clicked!');
});

$('#input').on('keyup', function() {
    console.log($(this).val());
});

// AJAX Call to Backend API
$.ajax({
    url: '/api/products',
    method: 'GET',
    headers: { 'Authorization': 'Bearer ' + token },
    success: function(data) {
        console.log('Products:', data);
        // Render data
        $.each(data, function(i, product) {
            $('#list').append(`<li>${product.name} - ₹${product.price}</li>`);
        });
    },
    error: function(xhr, status, error) {
        console.error('Error:', error);
    }
});

// Shorthand AJAX (GET)
$.get('/api/users', function(data) {
    console.log(data);
});

// Shorthand AJAX (POST)
$.post('/api/orders', { productId: 1, qty: 2 }, function(response) {
    console.log('Order created:', response);
});

// Using Promises (modern way)
$.ajax({ url: '/api/data', method: 'GET' })
    .done(data => console.log(data))
    .fail(err => console.error(err));

// Document Ready
$(document).ready(function() {
    // Code runs when DOM is fully loaded
    console.log('DOM ready!');
});
// Shorthand:
$(function() { console.log('DOM ready!'); });
```

---

## Q2. If We Want to Call Backend API – Syntax & Mandatory Parameters

### Using Fetch API (Modern JavaScript)
```javascript
// Mandatory: URL + Method
const response = await fetch('https://api.example.com/orders', {
    method: 'POST',                          // Mandatory: GET/POST/PUT/DELETE
    headers: {
        'Content-Type': 'application/json', // Mandatory for POST/PUT with body
        'Authorization': `Bearer ${token}`  // If API requires auth
    },
    body: JSON.stringify({                   // Mandatory for POST/PUT
        productId: 1,
        quantity: 2
    })
});

if (!response.ok) {
    throw new Error(`HTTP error! Status: ${response.status}`);
}
const data = await response.json();
console.log(data);
```

### Using Axios (Popular library)
```javascript
// GET request
const { data } = await axios.get('https://api.example.com/products', {
    headers: { Authorization: `Bearer ${token}` },
    params: { category: 'electronics', page: 1 }  // Query params
});

// POST request
const response = await axios.post('https://api.example.com/orders', 
    { productId: 1, quantity: 2 },  // Body
    {
        headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${token}`
        }
    }
);

// Axios interceptors (add auth globally)
axios.interceptors.request.use(config => {
    config.headers.Authorization = `Bearer ${localStorage.getItem('token')}`;
    return config;
});
```

### From C# (HttpClient)
```csharp
// GET
using var client = new HttpClient();
client.DefaultRequestHeaders.Authorization = 
    new AuthenticationHeaderValue("Bearer", token);
var response = await client.GetAsync("https://api.example.com/products");
var json = await response.Content.ReadAsStringAsync();
var products = JsonSerializer.Deserialize<List<Product>>(json);

// POST
var content = new StringContent(
    JsonSerializer.Serialize(new { ProductId = 1, Qty = 2 }),
    Encoding.UTF8,
    "application/json");
var postResponse = await client.PostAsync("https://api.example.com/orders", content);
```

### Mandatory Parameters Summary:
| Parameter | Required? | Description |
|-----------|-----------|-------------|
| URL | ✅ Yes | API endpoint |
| Method | ✅ Yes | HTTP verb |
| Content-Type | ✅ For POST/PUT | Tells server the body format |
| Authorization | ⚠️ If API secured | Bearer token / API Key |
| Body | ✅ For POST/PUT | Request payload |

---

## Q3. Difference Between Clustered and Non-Clustered Index

*(Detailed answer in SQL section Q7, but summarized here)*

| Feature | Clustered Index | Non-Clustered Index |
|---------|----------------|---------------------|
| Data storage | Data rows physically sorted by key | Separate B-tree structure with pointer to data |
| Per table | Only **1** | Up to **999** (SQL Server) |
| Auto-created | Yes, on PRIMARY KEY | No (must be created explicitly) |
| Speed | Faster for range queries | Faster for point lookups |
| Disk space | No extra space | Requires extra storage |
| Modification | Expensive (physically reorders data) | Less expensive |

```sql
-- Only 1 clustered index per table (usually PK)
CREATE CLUSTERED INDEX IX_Orders_OrderDate ON Orders(OrderDate);

-- Multiple non-clustered indexes allowed
CREATE NONCLUSTERED INDEX IX_Orders_Customer ON Orders(CustomerID);
CREATE NONCLUSTERED INDEX IX_Orders_Status ON Orders(Status) INCLUDE (OrderDate, Total);
```

---

## Q4. How Many Clustered and Non-Clustered Indexes Can Be in a Table?

| Index Type | SQL Server | MySQL (InnoDB) |
|------------|-----------|----------------|
| Clustered | **1 per table** | 1 per table (primary key IS the clustered index) |
| Non-Clustered | **Up to 999** | Up to 64 secondary indexes |

### Key Points:
- **Clustered index = 1** because it defines the physical order of data rows
- When you create a **PRIMARY KEY**, SQL Server automatically creates a clustered index on it (unless specified otherwise)
- You can create a clustered index on a non-PK column
- **More indexes = faster reads but slower writes** (every index must be updated on INSERT/UPDATE/DELETE)

```sql
-- Check existing indexes
SELECT i.name, i.type_desc, c.name AS column_name
FROM sys.indexes i
JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
JOIN sys.columns c ON ic.object_id = c.object_id AND ic.column_id = c.column_id
WHERE i.object_id = OBJECT_ID('Orders');
```

---

## Q5. Can We Call a Stored Procedure from a Stored Function?

**In SQL Server: Generally NO.**

### Why?
- **Stored Functions** in SQL Server must be deterministic and **cannot have side effects**
- Stored Procedures can have side effects (INSERT, UPDATE, DELETE, output params)
- SQL Server does NOT allow calling a stored procedure from within a function

```sql
-- This will cause an ERROR in SQL Server:
CREATE FUNCTION fn_GetEmployee(@id INT)
RETURNS VARCHAR(100)
AS
BEGIN
    EXEC sp_GetEmployee @id  -- ❌ ERROR! Cannot call SP from function
    RETURN 'Error'
END
```

### What You CAN Do:
1. **Call a function from a stored procedure** (Yes ✅)
2. **Call one function from another function** (Yes ✅)
3. **Call one stored procedure from another** (Yes ✅)
4. **Use CLR Table-Valued Functions** (workaround in some cases)

```sql
-- Function called from SP (WORKS)
CREATE PROCEDURE sp_ProcessOrder @orderId INT
AS
BEGIN
    DECLARE @tax DECIMAL = dbo.fn_CalculateTax(@orderId)  -- ✅ OK
    PRINT @tax
END
```

> In **MySQL**, stored procedures CAN be called from stored procedures but not from functions either (same restriction).

---

## Q6. Explain SOLID Principles

*(Full detailed answer in .NET Core section Q3 – see 02_DotNet_Core.md)*

### Quick Reference:
| Letter | Principle | One-liner |
|--------|-----------|-----------|
| **S** | Single Responsibility | One class = one job |
| **O** | Open/Closed | Extend, don't modify |
| **L** | Liskov Substitution | Subtypes must honor parent contract |
| **I** | Interface Segregation | Don't force unused methods |
| **D** | Dependency Inversion | Depend on abstractions, not implementations |

---

## Q7. Difference Between Interface and Abstract Class

| Feature | Interface | Abstract Class |
|---------|-----------|----------------|
| Instance creation | ❌ Cannot instantiate | ❌ Cannot instantiate |
| Multiple inheritance | ✅ A class can implement multiple | ❌ Only one abstract class |
| Fields/state | ❌ No fields (only properties) | ✅ Can have fields |
| Constructors | ❌ No constructor | ✅ Can have constructor |
| Access modifiers | All public (default) | Any access modifier |
| Implementation | No code (C# 8+ allows default) | Can have implementation |
| `is-a` vs `can-do` | `can-do` (behavior contract) | `is-a` (shared base) |
| Use case | Define capabilities | Define common base behavior |

```csharp
// Interface - defines WHAT to do (contract/capability)
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(decimal amount);
    bool ValidateCard(string cardNumber);
    // C# 8+ default implementation:
    string GetCurrency() => "INR";  // Optional default
}

// Abstract class - defines HOW partially (common base)
public abstract class Animal
{
    public string Name { get; set; }  // Field allowed
    protected int Age;                // Protected field
    
    public Animal(string name) { Name = name; }  // Constructor
    
    public void Breathe() { Console.WriteLine("Breathing..."); }  // Concrete method
    
    public abstract void MakeSound();  // Must be implemented by subclass
    
    public virtual void Move() { Console.WriteLine("Moving..."); }  // Can be overridden
}

public class Dog : Animal
{
    public Dog(string name) : base(name) { }
    public override void MakeSound() { Console.WriteLine("Woof!"); }
}

// A class can implement MULTIPLE interfaces
public class PayPalProcessor : IPaymentProcessor, IRefundable, ILoggable
{
    public async Task<PaymentResult> ProcessAsync(decimal amount) { ... }
    public bool ValidateCard(string cardNumber) { ... }
    // implement IRefundable and ILoggable members
}

// When to use:
// Interface: unrelated classes share behavior (IDisposable, IEnumerable, ILogger)
// Abstract Class: related classes share common base logic (Animal→Dog/Cat, Shape→Circle/Rectangle)
```

### Real Example:
```csharp
// Abstract: All shapes share the concept of Area/Perimeter
public abstract class Shape
{
    public abstract double Area();
    public abstract double Perimeter();
    
    // Common implemented method
    public void PrintInfo()
        => Console.WriteLine($"Area: {Area()}, Perimeter: {Perimeter()}");
}

// Interface: Different, unrelated things can be serializable
public interface ISerializable
{
    string Serialize();
    void Deserialize(string data);
}
```

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep*

---

---

# ⭐ BONUS – Important Questions (Added by Interviewer Pattern Analysis)

---

## BQ1. Four Pillars of OOP with Code Examples

### 1. Encapsulation – Hiding internal state
```csharp
public class BankAccount
{
    private decimal _balance;  // Private — cannot be accessed directly

    public decimal Balance => _balance;  // Read-only property

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        _balance += amount;
    }

    public void Withdraw(decimal amount)
    {
        if (amount > _balance) throw new InvalidOperationException("Insufficient funds");
        _balance -= amount;
    }
}
// Caller cannot do: account._balance = 999999; ← protected!
// Must use: account.Deposit(1000);
```

### 2. Inheritance – Code reuse through hierarchy
```csharp
public class Vehicle
{
    public string Brand { get; set; }
    public int Speed { get; set; }

    public virtual void Start() => Console.WriteLine($"{Brand} starting...");
    public void Stop() => Console.WriteLine($"{Brand} stopping.");
}

public class Car : Vehicle
{
    public int Doors { get; set; }
    public override void Start()
    {
        base.Start();  // Call parent
        Console.WriteLine("Car engine revving!");
    }
}

public class ElectricCar : Car
{
    public int BatteryLevel { get; set; }
    public override void Start() => Console.WriteLine("Silent electric start!");
}
```

### 3. Polymorphism – One interface, many forms
```csharp
// Runtime polymorphism (method overriding)
public abstract class Shape
{
    public abstract double Area();
    public void PrintArea() => Console.WriteLine($"Area: {Area()}");  // Calls overridden version
}

public class Circle : Shape
{
    public double Radius { get; set; }
    public override double Area() => Math.PI * Radius * Radius;
}

public class Rectangle : Shape
{
    public double Width, Height;
    public override double Area() => Width * Height;
}

// Polymorphic usage — same code works for ANY shape
List<Shape> shapes = new() { new Circle { Radius = 5 }, new Rectangle { Width = 4, Height = 6 } };
foreach (var shape in shapes)
    shape.PrintArea();  // Each calls its own Area()

// Compile-time polymorphism (method overloading)
public class Calculator
{
    public int Add(int a, int b) => a + b;
    public double Add(double a, double b) => a + b;
    public int Add(int a, int b, int c) => a + b + c;
    // Same name, different signatures
}
```

### 4. Abstraction – Hide complexity, show essentials
```csharp
// Abstract class
public abstract class PaymentGateway
{
    // User only sees these methods — implementation hidden
    public abstract bool ProcessPayment(decimal amount);
    public abstract bool Refund(string transactionId);

    // Common concrete implementation
    public string GenerateTransactionId() => Guid.NewGuid().ToString();
}

// Interface (pure abstraction)
public interface ILogger
{
    void Log(string message);
    void LogError(string message, Exception ex);
}

// Implementation details are hidden from callers
public class StripePaymentGateway : PaymentGateway
{
    public override bool ProcessPayment(decimal amount)
    {
        // Complex Stripe API logic hidden here
        return true;
    }
    public override bool Refund(string transactionId) => true;
}
```

---

## BQ2. Method Overloading vs Method Overriding

```csharp
// METHOD OVERLOADING (Compile-time / Static polymorphism)
// Same class, same name, DIFFERENT parameters
public class MathHelper
{
    public int Multiply(int a, int b) => a * b;
    public double Multiply(double a, double b) => a * b;
    public int Multiply(int a, int b, int c) => a * b * c;
    // ✅ Different: number of params, param types
    // ❌ Cannot overload by return type only
}

// METHOD OVERRIDING (Runtime / Dynamic polymorphism)
// Parent and child class, same name, same signature, different behavior
public class Animal
{
    public virtual string Sound() => "...";  // virtual = can be overridden
}

public class Dog : Animal
{
    public override string Sound() => "Woof!";  // override = replaces parent
}

public class Cat : Animal
{
    public override string Sound() => "Meow!";
    public sealed override string ToString() => "I am a Cat";  // sealed = no further override
}

// new keyword: HIDES parent method (not polymorphic!)
public class Tiger : Animal
{
    public new string Sound() => "Roar!";  // Hides, NOT overrides
}

Animal tiger = new Tiger();
Console.WriteLine(tiger.Sound());  // "..." — calls Animal.Sound() not Tiger.Sound()!
// Because new hides, not overrides — runtime type binding doesn't apply

// Key difference:
Tiger t = new Tiger();
Console.WriteLine(t.Sound());   // "Roar!" — calls Tiger.Sound() (compile-time type)

Animal d = new Dog();
Console.WriteLine(d.Sound());  // "Woof!" — override: runtime type wins
```

| Feature | Overloading | Overriding |
|---------|-------------|------------|
| Class | Same class | Parent + Child class |
| Signature | Different | Same (name + params) |
| When resolved | Compile time | Runtime |
| Keyword | None needed | `virtual` + `override` |
| Polymorphism | Static | Dynamic |

---

## BQ3. What is the Cloud? Important Cloud Concepts for Interviews

```
Cloud computing = Delivering IT resources (servers, storage, DB, networking)
over the internet on a pay-as-you-go basis.

Cloud Service Models:
┌─────────────────────────────────────────────────────┐
│  IaaS (Infrastructure as a Service)                  │
│  → Virtual machines, networking, storage             │
│  → Example: Azure VMs, AWS EC2                       │
│  → You manage: OS, runtime, apps, data              │
├─────────────────────────────────────────────────────┤
│  PaaS (Platform as a Service)                        │
│  → Managed runtime + database environments           │
│  → Example: Azure App Service, AWS Elastic Beanstalk │
│  → You manage: apps and data only                   │
├─────────────────────────────────────────────────────┤
│  SaaS (Software as a Service)                        │
│  → Ready-to-use software                            │
│  → Example: Office 365, Salesforce, Gmail           │
│  → You manage: nothing (just use it)                │
└─────────────────────────────────────────────────────┘
```

```csharp
// Azure services commonly asked in interviews:

// Azure Blob Storage (file storage)
BlobServiceClient blobClient = new BlobServiceClient(connectionString);
BlobContainerClient container = blobClient.GetBlobContainerClient("images");
await container.UploadBlobAsync("profile.jpg", fileStream);

// Azure Service Bus (messaging)
ServiceBusClient sbClient = new ServiceBusClient(connectionString);
ServiceBusSender sender = sbClient.CreateSender("orders-queue");
await sender.SendMessageAsync(new ServiceBusMessage(JsonSerializer.Serialize(order)));

// Azure Key Vault (secrets management)
var client = new SecretClient(new Uri(kvUri), new DefaultAzureCredential());
KeyVaultSecret secret = await client.GetSecretAsync("ConnectionString");
string connectionString = secret.Value;

// Cloud deployment terms:
// Horizontal scaling: Add MORE servers (scale out)
// Vertical scaling: Add MORE power to existing server (scale up)
// Auto-scaling: Automatically add/remove based on load
// Load Balancer: Distributes traffic across multiple servers
// CDN: Content Delivery Network — serves static content from edge locations
```

---

## BQ4. Explain the Code-First and Database-First Approaches in Entity Framework

```csharp
// CODE-FIRST: Write C# classes → EF generates database
// Define entity
public class Employee
{
    public int Id { get; set; }
    [Required, MaxLength(100)]
    public string Name { get; set; }
    public decimal Salary { get; set; }
    public int DepartmentId { get; set; }
    public Department Department { get; set; }  // Navigation property
}

public class Department
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Employee> Employees { get; set; }  // Navigation
}

// DbContext
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    
    public DbSet<Employee> Employees { get; set; }
    public DbSet<Department> Departments { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Fluent API configuration
        modelBuilder.Entity<Employee>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Salary).HasColumnType("decimal(18,2)");
            entity.HasOne(e => e.Department)
                  .WithMany(d => d.Employees)
                  .HasForeignKey(e => e.DepartmentId)
                  .OnDelete(DeleteBehavior.Restrict);
        });

        // Seed data
        modelBuilder.Entity<Department>().HasData(
            new Department { Id = 1, Name = "Engineering" },
            new Department { Id = 2, Name = "HR" }
        );
    }
}

// Run migrations:
// dotnet ef migrations add InitialCreate
// dotnet ef database update

// CRUD Operations
public class EmployeeRepository
{
    private readonly AppDbContext _context;
    public EmployeeRepository(AppDbContext context) { _context = context; }

    public async Task<Employee?> GetByIdAsync(int id)
        => await _context.Employees
            .Include(e => e.Department)  // Eager loading
            .FirstOrDefaultAsync(e => e.Id == id);

    public async Task<List<Employee>> GetAllAsync()
        => await _context.Employees.AsNoTracking().ToListAsync();  // No change tracking = faster reads

    public async Task AddAsync(Employee employee)
    {
        _context.Employees.Add(employee);
        await _context.SaveChangesAsync();
    }

    public async Task UpdateAsync(Employee employee)
    {
        _context.Employees.Update(employee);
        await _context.SaveChangesAsync();
    }

    public async Task DeleteAsync(int id)
    {
        var emp = await _context.Employees.FindAsync(id);
        if (emp != null)
        {
            _context.Employees.Remove(emp);
            await _context.SaveChangesAsync();
        }
    }
}

// DATABASE-FIRST: Existing database → EF scaffolds classes
// Command: dotnet ef dbcontext scaffold "Server=...;Database=MyDB" Microsoft.EntityFrameworkCore.SqlServer -o Models
```

---

## BQ5. Unit Testing Best Practices (xUnit + Moq)

```csharp
// Install: xunit, xunit.runner.visualstudio, Moq, FluentAssertions

// System Under Test
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailService _emailService;

    public OrderService(IOrderRepository repository, IEmailService emailService)
    {
        _repository = repository;
        _emailService = emailService;
    }

    public async Task<Order> PlaceOrderAsync(CreateOrderDto dto)
    {
        if (dto.Quantity <= 0)
            throw new ArgumentException("Quantity must be positive");

        var order = new Order { ProductId = dto.ProductId, Quantity = dto.Quantity };
        await _repository.SaveAsync(order);
        await _emailService.SendConfirmationAsync(dto.CustomerEmail, order);
        return order;
    }
}

// Unit Tests with xUnit + Moq
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _mockRepo;
    private readonly Mock<IEmailService> _mockEmail;
    private readonly OrderService _sut;  // System Under Test

    public OrderServiceTests()
    {
        _mockRepo = new Mock<IOrderRepository>();
        _mockEmail = new Mock<IEmailService>();
        _sut = new OrderService(_mockRepo.Object, _mockEmail.Object);
    }

    [Fact]
    public async Task PlaceOrder_ValidInput_SavesAndSendsEmail()
    {
        // ARRANGE
        var dto = new CreateOrderDto { ProductId = 1, Quantity = 2, CustomerEmail = "test@test.com" };
        _mockRepo.Setup(r => r.SaveAsync(It.IsAny<Order>())).Returns(Task.CompletedTask);
        _mockEmail.Setup(e => e.SendConfirmationAsync(It.IsAny<string>(), It.IsAny<Order>()))
                  .Returns(Task.CompletedTask);

        // ACT
        var result = await _sut.PlaceOrderAsync(dto);

        // ASSERT
        Assert.NotNull(result);
        Assert.Equal(1, result.ProductId);
        _mockRepo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);   // Called exactly once
        _mockEmail.Verify(e => e.SendConfirmationAsync("test@test.com", It.IsAny<Order>()), Times.Once);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100)]
    public async Task PlaceOrder_InvalidQuantity_ThrowsArgumentException(int qty)
    {
        // ARRANGE
        var dto = new CreateOrderDto { Quantity = qty };

        // ACT & ASSERT
        await Assert.ThrowsAsync<ArgumentException>(() => _sut.PlaceOrderAsync(dto));
        _mockRepo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Never);  // Not called!
    }
}

// Testing principles (AAA pattern):
// Arrange: Set up test data and mocks
// Act: Call the method being tested
// Assert: Verify the result
```

---

## BQ6. What is Caching and How to Implement It in .NET?

```csharp
// Types of Caching:
// 1. In-Memory Cache (single server)
// 2. Distributed Cache (Redis – multiple servers)
// 3. Response Cache (HTTP level)
// 4. Output Cache (ASP.NET Core 7+)

// IN-MEMORY CACHE
builder.Services.AddMemoryCache();

public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repo;
    private const string CacheKey = "AllProducts";

    public ProductService(IMemoryCache cache, IProductRepository repo)
    {
        _cache = cache; _repo = repo;
    }

    public async Task<List<Product>> GetAllAsync()
    {
        // Try to get from cache first
        if (_cache.TryGetValue(CacheKey, out List<Product>? cached))
            return cached!;  // Cache HIT

        // Cache MISS — fetch from DB
        var products = await _repo.GetAllAsync();

        // Store in cache with options
        var options = new MemoryCacheEntryOptions()
            .SetAbsoluteExpiration(TimeSpan.FromMinutes(10))  // Expire after 10 min
            .SetSlidingExpiration(TimeSpan.FromMinutes(2))    // Reset if accessed within 2 min
            .SetPriority(CacheItemPriority.High);
        _cache.Set(CacheKey, products, options);

        return products;
    }

    public async Task UpdateProductAsync(Product product)
    {
        await _repo.UpdateAsync(product);
        _cache.Remove(CacheKey);  // Invalidate cache after update!
    }

    // GetOrCreate pattern (cleaner)
    public async Task<List<Product>> GetProductsCached()
        => await _cache.GetOrCreateAsync(CacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            return await _repo.GetAllAsync();
        }) ?? new List<Product>();
}

// DISTRIBUTED CACHE (Redis)
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "MyApp:";
});

public class DistributedCacheService
{
    private readonly IDistributedCache _cache;
    public DistributedCacheService(IDistributedCache cache) { _cache = cache; }

    public async Task<Product?> GetProductAsync(int id)
    {
        var key = $"product:{id}";
        var json = await _cache.GetStringAsync(key);
        if (json != null) return JsonSerializer.Deserialize<Product>(json);

        var product = await _repo.GetByIdAsync(id);
        if (product != null)
            await _cache.SetStringAsync(key, JsonSerializer.Serialize(product),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
                });
        return product;
    }
}

// Cache-Aside pattern: Load on demand, expire, refresh
// Write-Through: Write to cache + DB simultaneously
// Write-Behind: Write to cache first, sync to DB asynchronously
```

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep | ⭐ = Extra Important*
