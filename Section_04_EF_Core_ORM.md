# ⚙️ Section 4: Entity Framework Core & ORM — Interview Preparation Guide
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> BSK uses **EF Core 8** for transactional operations and **Dapper** for performance-critical dashboard queries. This section covers both.

---

## 📋 Table of Contents

1. [EF Core Fundamentals](#1-ef-core-fundamentals)
2. [DbContext & Migrations](#2-dbcontext--migrations)
3. [Loading Strategies](#3-loading-strategies)
4. [Transactions & Unit of Work](#4-transactions--unit-of-work)
5. [EF Core Performance](#5-ef-core-performance)
6. [Interceptors & Global Query Filters](#6-interceptors--global-query-filters)
7. [Dapper — Micro ORM](#7-dapper--micro-orm)
8. [Quick-Fire EF Core Checklist](#8-quick-fire-ef-core-checklist)

---

## 1. EF Core Fundamentals

### Q1: What is EF Core? How does it map to the BSK database?

**Entity Framework Core (EF Core)** is Microsoft's cross-platform ORM (Object-Relational Mapper). It maps C# classes (entities) to database tables and allows CRUD operations using LINQ without writing raw SQL.

```csharp
// Entity Class (Maps to SQL table)
public class Case
{
    public Guid Id { get; set; }
    public string ClaimantId { get; set; }
    public decimal ClaimAmount { get; set; }
    public bool IsCaseReached { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }

    // Navigation property (relationship)
    public Claimant Claimant { get; set; }
    public ICollection<Invoice> Invoices { get; set; }
}

// ApplicationDBContext — Gateway to all tables
public class ApplicationDBContext : DbContext
{
    public ApplicationDBContext(DbContextOptions<ApplicationDBContext> options) : base(options) { }

    public DbSet<Case> Cases { get; set; }
    public DbSet<Claimant> Claimants { get; set; }
    public DbSet<Invoice> Invoices { get; set; }
    public DbSet<Payment> Payments { get; set; }
}
```

---

### Q2: Configuring EF Core in Program.cs

```csharp
builder.Services.AddDbContext<ApplicationDBContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(30); // 30 second query timeout
        });
    options.EnableSensitiveDataLogging(isDevelopment); // Log params in dev only
});
```

---

## 2. DbContext & Migrations

### Q3: EF Core Migrations — Full workflow

```bash
# Add a new migration
dotnet ef migrations add AddCaseAuditTable

# View generated SQL (without executing)
dotnet ef migrations script

# Apply migrations to database
dotnet ef database update

# Remove last migration (only if not applied)
dotnet ef migrations remove

# Revert to specific migration
dotnet ef database update AddCaseTable  # Rolls back to this migration
```

```csharp
// Fluent API Configuration (in ApplicationDBContext.OnModelCreating)
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Primary Key
    modelBuilder.Entity<Case>()
        .HasKey(c => c.Id);

    // Required field
    modelBuilder.Entity<Case>()
        .Property(c => c.ClaimAmount)
        .IsRequired()
        .HasColumnType("decimal(18,2)");

    // One-to-Many relationship
    modelBuilder.Entity<Case>()
        .HasOne(c => c.Claimant)
        .WithMany(cl => cl.Cases)
        .HasForeignKey(c => c.ClaimantId)
        .OnDelete(DeleteBehavior.Restrict); // No cascade delete

    // Index
    modelBuilder.Entity<Case>()
        .HasIndex(c => c.ClaimantId)
        .HasDatabaseName("IX_Case_ClaimantId");

    // Value Converter (auto-encryption for sensitive fields)
    var encryptConverter = new ValueConverter<string, string>(
        v => EncryptionHelper.Encrypt(v),  // Encrypt on write
        v => EncryptionHelper.Decrypt(v)   // Decrypt on read
    );
    modelBuilder.Entity<Person>()
        .Property(p => p.PanNo)
        .HasConversion(encryptConverter);  // Auto-encrypt PAN numbers!
}
```

---

## 3. Loading Strategies

### Q4: Eager, Lazy, and Explicit Loading — and the N+1 Problem

**Eager Loading** — Load related data with the initial query (SQL JOIN):
```csharp
// Use when you KNOW you'll access navigation properties immediately
var caseWithClaimant = await _context.Cases
    .Include(c => c.Claimant)               // JOIN Claimant
    .ThenInclude(cl => cl.Person)           // JOIN Person from Claimant
    .Include(c => c.Invoices)               // JOIN Invoices collection
    .FirstOrDefaultAsync(c => c.Id == id);
```

**Lazy Loading** — Load navigation properties on first access:
```csharp
// Requires virtual navigation properties + Proxies package
// BAD: N+1 Problem!
var cases = await _context.Cases.ToListAsync(); // 1 query
foreach (var c in cases)
{
    Console.WriteLine(c.Claimant.Name); // N QUERIES (100 cases = 101 total SQL calls!)
}
```

**Explicit Loading** — Load later when needed:
```csharp
var singleCase = await _context.Cases.FindAsync(id); // Initial query

// Later — conditionally load claimant
if (needsClaimantDetails)
{
    await _context.Entry(singleCase)
        .Reference(c => c.Claimant)
        .LoadAsync();
}
```

**N+1 Problem Summary**:
- Fetch 100 cases = 1 query
- Access `c.Claimant.Name` in a loop with Lazy Loading = 100 more queries
- **Total = 101 queries!**  → Fix: Use Eager Loading (`.Include()`)

---

## 4. Transactions & Unit of Work

### Q5: EF Core Transactions — Default vs Explicit

**Default (Auto-Transaction)**:
```csharp
// EF Core wraps SaveChangesAsync() in a transaction automatically
_context.Cases.Add(newCase);
_context.Invoices.Add(newInvoice);
await _context.SaveChangesAsync(); // Both saved atomically OR both rollback
```

**Explicit Transaction (Multi-repository, multi-save-changes)**:
```csharp
// BSK pattern: BSK processes claimant registration + case creation atomically
public async Task<bool> ProcessClaimRegistrationAsync(ClaimantModel claimant, CaseModel caseModel)
{
    using var transaction = await _context.Database.BeginTransactionAsync();
    try
    {
        await _claimantRepo.AddAsync(claimant);
        await _context.SaveChangesAsync(); // Saves Claimant, generates ClaimantId

        caseModel.ClaimantId = claimant.Id;
        await _caseRepo.AddAsync(caseModel);
        await _context.SaveChangesAsync(); // Saves Case

        await transaction.CommitAsync(); // Only now is anything written to DB
        return true;
    }
    catch (Exception ex)
    {
        await transaction.RollbackAsync(); // Undo both saves
        LogHelper.Error("Transaction failed, database rolled back", ex);
        return false;
    }
}
```

---

### Q6: Unit of Work Pattern with EF Core

EF Core's `DbContext` **IS** a Unit of Work — it tracks all entity changes and commits them in a single `SaveChangesAsync()` call.

```csharp
// Without explicit Unit of Work (EF Core built-in)
public class CaseService
{
    private readonly ApplicationDBContext _context;

    public async Task RegisterCaseAsync(CaseModel model)
    {
        // Multiple operations — all tracked by DbContext
        _context.Cases.Add(model.Case);
        _context.Invoices.Add(model.Invoice);
        _context.AuditLogs.Add(new AuditLog { Action = "CaseRegistered" });

        await _context.SaveChangesAsync(); // ALL THREE saved in one transaction
    }
}
```

---

## 5. EF Core Performance

### Q7: .AsNoTracking() — When to use

```csharp
// BAD for read-only queries — tracking overhead wastes memory
var cases = await _context.Cases.ToListAsync();

// GOOD — disable tracking for read-only (dashboard/report queries)
var cases = await _context.Cases
    .AsNoTracking()                  // No change tracking
    .Where(c => c.IsActive)
    .Select(c => new CaseSummaryDto(c.Id, c.ClaimantName, c.ClaimAmount))
    .ToListAsync();

// Set as default for entire context (read-heavy scenario)
services.AddDbContext<ApplicationDBContext>(options =>
    options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

**Rule**: Use `.AsNoTracking()` for all GET/read operations. Use tracking (default) only when you need to UPDATE entities.

---

### Q8: Compiled Queries — for hot code paths

```csharp
// Compiled queries are pre-compiled ONCE and reused — faster for frequently called queries
private static readonly Func<ApplicationDBContext, Guid, Task<Case?>> GetCaseByIdCompiled =
    EF.CompileAsyncQuery((ApplicationDBContext ctx, Guid id) =>
        ctx.Cases.AsNoTracking().FirstOrDefault(c => c.Id == id));

// Usage
var caseData = await GetCaseByIdCompiled(_context, caseId);
```

---

### Q9: Split Query for collections (avoiding Cartesian explosion)

```csharp
// Problem: Including multiple collections → Cartesian product = MASSIVE result set
var caseWithData = await _context.Cases
    .Include(c => c.Invoices)    // 10 invoices
    .Include(c => c.Documents)   // 5 documents
    .FirstOrDefaultAsync(c => c.Id == id);
// EF Core generates ONE query with 10×5 = 50 row cartesian product!

// Solution: Split Queries — separate SQL query per collection
var caseWithData = await _context.Cases
    .Include(c => c.Invoices)
    .Include(c => c.Documents)
    .AsSplitQuery()              // 3 separate SQL queries, no cartesian explosion
    .FirstOrDefaultAsync(c => c.Id == id);
```

---

## 6. Interceptors & Global Query Filters

### Q10: SaveChanges Override — Auto Timestamps & Audit

```csharp
// In ApplicationDBContext.cs
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    // Auto-set timestamps
    var entries = ChangeTracker.Entries()
        .Where(e => e.Entity is IAuditableEntity &&
                    (e.State == EntityState.Added || e.State == EntityState.Modified));

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

---

### Q11: EF Core Interceptors (EF 7+)

```csharp
// SaveChangesInterceptor — clean separation from DbContext (SRP)
public class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        // Perform audit logging here...
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}

// Register interceptor
services.AddDbContext<ApplicationDBContext>(options =>
    options.AddInterceptors(new AuditInterceptor()));
```

---

### Q12: Global Query Filters — Soft Delete pattern

```csharp
// In OnModelCreating — applies WHERE clause to ALL queries on this entity
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Soft delete filter — IsDeleted = false always applied
    modelBuilder.Entity<Case>().HasQueryFilter(c => !c.IsDeleted);
    modelBuilder.Entity<Claimant>().HasQueryFilter(cl => !cl.IsDeleted);
}

// Usage — filter applied automatically
var activeCases = await _context.Cases.ToListAsync(); // WHERE IsDeleted = 0 always added

// Override filter when needed (admin queries, restore feature)
var allCases = await _context.Cases.IgnoreQueryFilters().ToListAsync();
```

---

## 7. Dapper — Micro ORM

### Q13: When to use Dapper over EF Core?

**BSK Uses Both**:
- **EF Core**: CRUD operations, transactional state modifications, relationships
- **Dapper**: Complex reporting queries (Executive Dashboard), high-performance reads

```csharp
// Dapper — raw SQL with automatic object mapping
public class DashboardRepository
{
    private readonly IDbConnection _connection;

    public DashboardRepository(IDbConnection connection) => _connection = connection;

    public async Task<IEnumerable<DashboardDto>> GetDashboardMetricsAsync(DateTime start, DateTime end)
    {
        const string sql = @"
            ;WITH LatestStages AS (
                SELECT CaseId, ROW_NUMBER() OVER(PARTITION BY CaseId ORDER BY CreatedAt DESC) AS rn
                FROM CaseLog WITH (NOLOCK)
            )
            SELECT
                COUNT(c.Id) AS TotalCases,
                SUM(c.ClaimAmount) AS TotalClaimAmount,
                AVG(DATEDIFF(DAY, c.CreatedAt, GETDATE())) AS AvgDaysOpen
            FROM [Case] c WITH (NOLOCK)
            LEFT JOIN LatestStages ls ON c.Id = ls.CaseId AND ls.rn = 1
            WHERE c.CreatedAt BETWEEN @StartDate AND @EndDate";

        return await _connection.QueryAsync<DashboardDto>(sql, new { StartDate = start, EndDate = end });
    }
}
```

---

### Q14: Dapper Multi-Mapping (one-to-many with SplitOn)

```csharp
const string sql = @"
    SELECT c.Id, c.ClaimantName, i.Id AS InvoiceId, i.Amount
    FROM [Case] c
    LEFT JOIN Invoice i ON c.Id = i.CaseId
    WHERE c.Id = @CaseId";

var caseDictionary = new Dictionary<Guid, CaseWithInvoicesDto>();

await _connection.QueryAsync<CaseWithInvoicesDto, InvoiceDto, CaseWithInvoicesDto>(
    sql,
    (caseDto, invoice) =>
    {
        if (!caseDictionary.TryGetValue(caseDto.Id, out var caseEntry))
        {
            caseEntry = caseDto;
            caseEntry.Invoices = new List<InvoiceDto>();
            caseDictionary[caseDto.Id] = caseEntry;
        }
        if (invoice != null)
            caseEntry.Invoices.Add(invoice);
        return caseEntry;
    },
    new { CaseId = id },
    splitOn: "InvoiceId"); // Column where second object starts
```

---

## 8. Quick-Fire EF Core Checklist

| Question | Short Answer |
|---|---|
| What is change tracking? | EF Core tracks entity state (Added, Modified, Deleted, Unchanged) |
| `Find()` vs `FirstOrDefault()` | `Find()` checks cache first, then DB; `FirstOrDefault()` always hits DB |
| `Attach()` | Attach existing entity (not from this context) in Unchanged state |
| `Entry(entity).State` | Manually set entity state (Added, Modified) for disconnected scenarios |
| Code First vs DB First | Code First: create DB from classes; DB First: generate classes from DB |
| `ExecuteRawSql()` | Run arbitrary SQL — use for bulk updates avoiding tracking overhead |
| `.Any()` vs `.Count() > 0` | `.Any()` is faster — generates `SELECT TOP 1` or `EXISTS` |
| Lazy Loading packages | `Microsoft.EntityFrameworkCore.Proxies` needed for virtual nav props |
| Migration `__EFMigrationsHistory` | SQL table EF uses to track which migrations have been applied |
| `AddAsync` vs `Add` | `AddAsync` only needed for value generators; `Add` is fine for most cases |

---

## 🏁 BSK Talking Points for EF Core Section

> *"In BSK, I use EF Core 8 for all CRUD operations with explicit transactions for multi-step claim registration flows. I configure Fluent API mappings in `OnModelCreating` — including value converters to auto-encrypt sensitive columns like PAN numbers before writing to the database. For performance, I use `.AsNoTracking()` on all read-only queries and Dapper for the Executive Dashboard which has complex CTEs and window functions. I also override `SaveChangesAsync` to auto-populate `CreatedAt` and `UpdatedAt` timestamps across all entities implementing `IAuditableEntity`."*

---
*Next: See [Section_05_System_Design_Advanced.md](./Section_05_System_Design_Advanced.md)*
