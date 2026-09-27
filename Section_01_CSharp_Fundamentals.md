# 📘 Section 1: C# Fundamentals — Interview Preparation Guide
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> **How to use this file**: Read each question → cover the answer → try to say it aloud → then verify. Focus on **BSK Project References** — use them as real-world examples in your interviews.

---

## 📋 Table of Contents

1. [OOP Pillars](#1-oop-pillars)
2. [Classes & Types](#2-classes--types)
3. [Interfaces vs Abstract Classes](#3-interfaces-vs-abstract-classes)
4. [Generics](#4-generics)
5. [Collections & LINQ](#5-collections--linq)
6. [Exception Handling](#6-exception-handling)
7. [Async / Await & Multithreading](#7-async--await--multithreading)
8. [Memory Management & GC](#8-memory-management--gc)
9. [Modern C# Features (C# 9–12)](#9-modern-c-features-c9-12)
10. [Design Patterns in C#](#10-design-patterns-in-c)
11. [Quick-Fire Checklist](#11-quick-fire-checklist)

---

## 1. OOP Pillars

### Q1: What are the 4 pillars of OOP? Give a real example for each.
**Answer**:
1. **Encapsulation** – Bundle data and behavior together, hide internal state. E.g., BSK's `ServiceHealthRegistry` keeps its `ConcurrentDictionary` private and exposes only `IsServiceUp()` and `SetHealthStatus()` methods.
2. **Abstraction** – Expose only what's needed. `ICaseAsyncRepository` interface defines `GetCaseById()` without exposing SQL/EF internals.
3. **Inheritance** – A child class inherits from a parent. `ThirdPartyHealthMonitorService : BackgroundService` inherits infrastructure from the base class.
4. **Polymorphism** – Same method name, different behavior. Multiple controllers all override `ControllerBase` methods.

---

### Q2: Difference between `is` and `as` operators in C#?

```csharp
// 'is' with pattern matching (C# 7+) — preferred
if (entity is AuditableEntity auditEntity)
{
    auditEntity.UpdatedAt = DateTime.UtcNow;
}

// 'as' — tries to cast, returns null if it fails (no exception)
var auditEntity = entity as AuditableEntity;
if (auditEntity != null) { /* safe to use */ }
```
> `is` with pattern matching is preferred in modern C#. `as` can return null silently — always null-check.

---

## 2. Classes & Types

### Q3: What is a `sealed` class? When should you use it?

A `sealed` class **cannot be inherited** by any other class.

```csharp
public sealed class TokenHelper
{
    public static string GenerateJwt(string userId) { /* ... */ return ""; }
}
```

**When to use**:
- **Security**: Core utility classes (encryption helpers, license validators) that must never be altered.
- **Performance**: JIT compiler can **devirtualize** calls (resolves method at compile time instead of vtable lookup).

---

### Q4: Value Types vs. Reference Types — explain with memory allocation

| Feature | Value Types (int, bool, struct) | Reference Types (class, string) |
|---|---|---|
| **Memory** | Stack (or inline in parent heap object) | Heap; Stack holds a pointer |
| **Assignment** | Copies the actual value | Copies the reference (address) |
| **GC Overhead** | Deallocated instantly at scope exit | Managed by Garbage Collector |
| **Nullability** | Cannot be null (unless int? nullable) | Can be null |

```csharp
// Value Type — copy-by-value
int a = 10;
int b = a;  // b is a completely separate copy
b = 20;     // a is still 10

// Reference Type — copy-by-reference
var list1 = new List<int> { 1, 2, 3 };
var list2 = list1; // Both point to SAME object
list2.Add(4);       // list1 also has 4 now!
```

---

### Q5: What is a `struct`? When over a `class`?

```csharp
public readonly struct Money
{
    public decimal Amount { get; init; }
    public string Currency { get; init; }
    public Money(decimal amount, string currency) { Amount = amount; Currency = currency; }
}
```

**Use struct when**: Small (<16 bytes), short-lived, immutable. Examples: `Point`, `Color`, `Guid`.

---

## 3. Interfaces vs Abstract Classes

### Q6: Interface vs Abstract Class — with BSK example

| Feature | Interface | Abstract Class |
|---|---|---|
| **Multiple Inheritance** | YES — a class can implement multiple | NO — only one abstract class |
| **State (Fields)** | No instance fields | Can have fields, constructors, state |
| **Implementation** | Contract only (default impl since C# 8) | Can have concrete methods |

```csharp
// Interface — what any repository MUST do
public interface ICaseAsyncRepository
{
    Task<object> GetCaseById(string Id);
}

// Abstract Class — shared base logic for all BSK repositories
public abstract class BaseRepository
{
    protected readonly ApplicationDBContext _context;
    protected BaseRepository(ApplicationDBContext context) => _context = context;
    protected void LogOperation(string op) => Console.WriteLine($"[DB] {op}");
}

// Concrete class: inherits abstract + implements interface
public class CaseAsyncRepository : BaseRepository, ICaseAsyncRepository
{
    public CaseAsyncRepository(ApplicationDBContext context) : base(context) { }

    public async Task<object> GetCaseById(string Id)
    {
        LogOperation($"GetCaseById: {Id}");
        return await _context.Cases.FindAsync(Id);
    }
}
```

**Rule**: Use **Interface** to define what something does. Use **Abstract Class** to share what something is.

---

## 4. Generics

### Q7: What are Generics? Why important?

```csharp
// Generic Repository — ONE class replaces CaseRepository, DocumentRepository, etc.
public class GenericRepository<T> where T : class
{
    private readonly ApplicationDBContext _context;
    public GenericRepository(ApplicationDBContext context) => _context = context;

    public async Task<T> GetByIdAsync(object id) => await _context.Set<T>().FindAsync(id);
    public async Task AddAsync(T entity) => await _context.Set<T>().AddAsync(entity);
}
```

**Core benefits**:
1. **Type Safety** — Catch errors at compile time.
2. **No Boxing/Unboxing** — Better performance with value types.
3. **Code Reuse** — One class instead of many.

**Generic Constraints**:
```csharp
where T : class              // T must be reference type
where T : struct             // T must be value type
where T : IEvent             // T must implement IEvent
where T : new()              // T must have parameterless constructor
where T : class, IEntity, new() // Multiple constraints
```

---

## 5. Collections & LINQ

### Q8: When to use which collection?

| Collection | Use When |
|---|---|
| `List<T>` | Ordered, index-based access, frequent reads |
| `Dictionary<K,V>` | Key-Value lookup, O(1) average access |
| `HashSet<T>` | Unique items only, O(1) contains check |
| `Queue<T>` | FIFO processing (first in, first out) |
| `Stack<T>` | LIFO processing (last in, first out) |
| `ConcurrentDictionary<K,V>` | Thread-safe key-value in Singleton services |

**BSK**: `ServiceHealthRegistry` uses `ConcurrentDictionary` (Singleton accessed by multiple threads).

---

### Q9: Key LINQ queries with BSK business context

```csharp
// Filter active high-value cases
var highValueCases = cases.Where(c => c.IsActive && c.ClaimAmount > 500000);

// Group by policy type and sum claims
var totalByPolicy = cases
    .Where(c => c.IsActive)
    .GroupBy(c => c.PolicyType)
    .ToDictionary(g => g.Key, g => g.Sum(c => c.ClaimAmount));

// Top claimant by total claim amount
var topClaimant = cases
    .GroupBy(c => c.ClaimantName)
    .Select(g => new { Name = g.Key, Total = g.Sum(c => c.ClaimAmount) })
    .OrderByDescending(x => x.Total)
    .FirstOrDefault();

// LINQ Left Join (cases with optional invoices)
var result = from c in cases
             join i in invoices on c.Id equals i.CaseId into invoiceGroup
             from inv in invoiceGroup.DefaultIfEmpty()
             select new
             {
                 c.ClaimantName,
                 InvoiceAmount = inv != null ? inv.Amount : 0
             };
```

> Use **method syntax** (`.Where().GroupBy()`) — it's the modern preferred style.

---

## 6. Exception Handling

### Q10: try/catch/finally best practices with BSK DB transactions

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
    catch (DbUpdateException ex)  // Specific first!
    {
        await transaction.RollbackAsync();
        LogHelper.Error("Payment DB transaction failed", ex);
        throw; // Re-throw WITHOUT resetting stack trace (NOT throw ex;)
    }
    finally
    {
        // Always runs — cleanup code here
    }
}
```

**Best Practices**:
1. Never swallow silently: `catch (Exception ex) {}` — hides bugs
2. Catch specific exceptions first (`DbUpdateException` before `Exception`)
3. Use `throw;` not `throw ex;` — `throw ex` resets the stack trace
4. Use Global Middleware for most exceptions, not try-catch in every method
5. Never use exceptions for control flow (email validation → use `if`, not `throw`)

---

### Q11: Exception Filters with `when` clause (C# 6+)

```csharp
try
{
    await _repository.CallSurepassApi();
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.Unauthorized)
{
    // Only catches 401 — other HttpRequestExceptions fall through
    // Efficient: condition evaluated BEFORE stack unwinds
}
catch (HttpRequestException ex)
{
    // Catches all other HTTP exceptions
}
```

---

## 7. Async / Await & Multithreading

### Q12: Why use async/await? Thread blocking diagram

```
Synchronous:  [Thread 1] ─► [DB Query BLOCKED 500ms] ─► [Response]
                             Thread IDLE, cannot serve other users!

Asynchronous: [Thread 1] ─► [Start DB Query] ─► Thread FREED to serve others
                                                       |
              [Any Thread] ◄─────────── [DB returns, resume here]
```

**BSK context**: BSK handles concurrent HTTP requests. Synchronous DB calls cause **Thread Starvation** in Kestrel's thread pool. With `async/await`, threads are freed during I/O wait.

---

### Q13: Task.WhenAll vs Task.WhenAny

```csharp
// Task.WhenAll — run all in parallel, wait for ALL
Task<bool> panTask    = _kycService.VerifyPanAsync(pan);
Task<bool> smsTask    = _smsService.PingGatewayAsync(mobile);
Task<bool> ledgerTask = _ledgerService.CheckStatusAsync();

await Task.WhenAll(panTask, smsTask, ledgerTask);
// Sequential: 3 seconds. Parallel: max(each) = ~1 second!

// Task.WhenAny — return when FIRST task completes (timeout pattern)
var completedTask = await Task.WhenAny(apiCallTask, Task.Delay(TimeSpan.FromSeconds(5)));
if (completedTask == apiCallTask) return await apiCallTask;
else throw new TimeoutException("API call timed out.");
```

---

### Q14: SemaphoreSlim for async locking

```csharp
private static readonly SemaphoreSlim _lock = new(1, 1);

public async Task<string> GetCachedDataAsync()
{
    var data = await _cache.GetStringAsync("key");
    if (data != null) return data;

    await _lock.WaitAsync(); // Async-friendly lock (unlike lock{} which blocks)
    try
    {
        data = await _cache.GetStringAsync("key"); // Double-check pattern
        if (data != null) return data;
        data = await FetchFromDatabaseAsync();
        await _cache.SetStringAsync("key", data);
    }
    finally
    {
        _lock.Release(); // ALWAYS release in finally
    }
    return data;
}
```

---

## 8. Memory Management & GC

### Q15: .NET GC Generations explained

```
Gen 0  — Short-lived (local vars, temp objects). Collected FREQUENTLY, FAST.
Gen 1  — Survived Gen 0. Buffer zone.
Gen 2  — Long-lived (Singletons, static config). RARELY collected, EXPENSIVE.
LOH    — Objects > 85,000 bytes. Not compacted → memory fragmentation risk.
```

**GC-friendly code**:
```csharp
// BAD — 1000 heap allocations (string is immutable)
string log = "";
for (int i = 0; i < 1000; i++) log += $"Line {i}\n";

// GOOD — single buffer
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++) sb.AppendLine($"Line {i}");
string final = sb.ToString(); // ONE allocation

// For temp byte arrays — use pool
var buffer = ArrayPool<byte>.Shared.Rent(4096);
try { /* use buffer */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }
```

---

### Q16: IDisposable and using

```csharp
// using statement — auto-calls Dispose() even on exception
using (var handler = new DatabaseHandler())
{
    // use handler
} // Dispose() called automatically

// Modern using declaration (C# 8+)
using var transaction = await _dbContext.Database.BeginTransactionAsync();
// Disposed at end of scope — cleaner code
```

---

## 9. Modern C# Features (C#9-12)

### Q17: C# Records

```csharp
// BSK DTO pattern using records
public record CaseSummaryDto(Guid Id, string ClaimantName, decimal ClaimAmount);

// Value-based equality
var c1 = new CaseSummaryDto(id, "Omkar", 50000m);
var c2 = new CaseSummaryDto(id, "Omkar", 50000m);
bool equal = (c1 == c2); // TRUE — classes would return false!

// Non-destructive mutation with 'with'
var updated = c1 with { ClaimAmount = 75000m }; // Clone with one change
```

---

### Q18: Pattern Matching

```csharp
// BSK: Resolve case status color
public string GetStatusColor(CaseModel model) => model switch
{
    { IsSuccessOrDropped: true }  => "green",
    { IsCaseReached: false }      => "yellow",
    { ClaimAmount: > 500000 }     => "red-bold",
    null                          => "gray",
    _                             => "blue"
};
```

---

### Q19: Null Safety operators

```csharp
string? city = claimant?.Address?.City; // ?. null-conditional
string name  = claimant?.Name ?? "Unknown"; // ?? null-coalescing
claimant.Tags ??= new List<string>();       // ??= assign if null
```

---

## 10. Design Patterns in C#

### Q20: SOLID Principles — BSK Examples

| Principle | BSK Example |
|---|---|
| S – Single Responsibility | `CaseAsyncRepository` only handles Case DB operations |
| O – Open/Closed | Polly policies added without changing HttpClient code |
| L – Liskov Substitution | Any `ICaseAsyncRepository` impl works in controller |
| I – Interface Segregation | Separate `IHealthChecker`, `IHealthRegistry` interfaces |
| D – Dependency Inversion | Controller depends on `ICaseAsyncRepository`, not `CaseAsyncRepository` |

```csharp
// D — Dependency Inversion
// BAD — hard coupling
public class CaseController : ControllerBase {
    private readonly CaseAsyncRepository _repo = new CaseAsyncRepository(new ApplicationDBContext());
}

// GOOD — BSK pattern (depends on interface, injected by DI)
public class CaseController : ControllerBase {
    private readonly ICaseAsyncRepository _repo;
    public CaseController(ICaseAsyncRepository repo) => _repo = repo;
}
```

---

### Q21: Repository Pattern & Singleton Pattern

**Repository**:
```
Controller ──► ICaseAsyncRepository (Interface) ──► CaseAsyncRepository (EF Core)
```
Benefits: Decoupling, Testability (Moq), Centralized Query Logic.

**Singleton** (using Lazy<T>):
```csharp
public sealed class AppConfig
{
    private static readonly Lazy<AppConfig> _instance = new(() => new AppConfig());
    private AppConfig() { }
    public static AppConfig Instance => _instance.Value;
}
```

---

## 11. Quick-Fire Checklist

| Question | Short Answer |
|---|---|
| `string` vs `String` | Same — `string` is C# alias for `System.String` |
| `const` vs `readonly` | `const` = compile-time; `readonly` = set once at runtime |
| `static` keyword | Belongs to type, not instance |
| `virtual` vs `override` | `virtual` allows override; `override` replaces it |
| `abstract` | Must be overridden; class with abstract member must be abstract |
| Boxing/Unboxing | Value type to `object` (boxing) and back — avoid in hot loops |
| `IEnumerable` vs `IQueryable` | `IEnumerable` — in-memory; `IQueryable` — SQL-translated (EF Core) |
| `Span<T>` | Stack-allocated view over memory — zero allocation string ops |
| `ValueTask<T>` | Optimization over `Task<T>` for hot paths that often return sync |

---

## 🏁 BSK Talking Points for C# Section

> *"In my BSK project, I used modern C# features like records for DTOs, pattern matching for status resolution, and nullable reference types for safer null handling. I applied SOLID principles throughout — controllers depend on repository interfaces, not concrete classes — making unit testing with Moq easy. I used `async/await` throughout to prevent thread starvation and leveraged `SemaphoreSlim` for async-safe cache stampede prevention."*

---
*Next: See [Section_02_SQL_Database.md](./Section_02_SQL_Database.md)*
