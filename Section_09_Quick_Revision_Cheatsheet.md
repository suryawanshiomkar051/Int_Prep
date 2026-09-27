# ⚡ Section 9: Quick Revision Cheat Sheet — Night-Before Reference
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> **This is your exam-night document.** Read this the evening before every interview. Each topic is condensed to the essential facts you need to recall quickly.

---

## 🔵 C# FUNDAMENTALS — Rapid Reference

### OOP Pillars (one-liners)
```
Encapsulation   → Private fields + Public methods (hide implementation)
Abstraction     → Interfaces show WHAT, not HOW
Inheritance     → ThirdPartyHealthMonitorService : BackgroundService
Polymorphism    → Same interface, different behavior per implementation
```

### Value vs Reference Types
```
Value  → Stack, copied on assignment: int, bool, struct, decimal
Ref    → Heap, reference copied: class, string, array, List<T>
Null   → Value types can't be null. Ref types can. Use int? for nullable value.
```

### Keywords at a Glance
| Keyword | Meaning |
|---|---|
| `sealed` | Can't be inherited (also: method can't be overridden further) |
| `abstract` | Must be overridden; class with abstract method must be abstract |
| `virtual` | CAN be overridden in derived class |
| `override` | Replaces virtual/abstract in derived class |
| `const` | Compile-time constant — value baked into IL |
| `readonly` | Set once: in declaration or constructor — runtime constant |
| `static` | Belongs to type, not instance — shared across all callers |
| `partial` | Class split across multiple files |
| `new` (hiding) | Hides base class member — breaks polymorphism! (not the same as `override`) |

### LINQ Must-Know Methods
```csharp
.Where()         → Filter
.Select()        → Project/Map
.GroupBy()       → Group + aggregate
.OrderBy()       → Sort ascending
.FirstOrDefault()→ Single or null (safe)
.Any()           → Returns bool (faster than Count() > 0)
.All()           → All elements match predicate?
.Sum() / .Count()→ Aggregation
.ToDictionary()  → Build dict from sequence
.ToList()        → Materialize from IQueryable/IEnumerable
.AsNoTracking()  → EF Core: no change tracking (reads only)
```

### Async Quick Rules
```
✅ Use async all the way (controller → repo → db)
✅ Task.WhenAll = parallel tasks
✅ Task.WhenAny = first to complete wins
✅ SemaphoreSlim for async locking (NOT lock{})
❌ Never .Result or .Wait() in async context (deadlock!)
❌ Never async void (can't catch exceptions, use async Task)
```

### DI Lifetimes — 3-Second Memory Aid
```
Transient  → New every injection   (lightweight, stateless services)
Scoped     → New every HTTP request (DbContext, Repositories)
Singleton  → One forever           (ConcurrentDictionary, Config, CacheService)

DANGER: Scoped inside Singleton = Captive Dependency = connection leak!
FIX:    Use IServiceScopeFactory.CreateScope() inside Singleton methods
```

### Modern C# Features
```csharp
// Records (value equality, immutable DTOs)
public record CaseSummaryDto(Guid Id, string Name, decimal Amount);
var updated = dto with { Amount = 75000 }; // Non-destructive mutation

// Pattern matching
var color = model switch {
    { IsSuccess: true } => "green",
    { Amount: > 500000 } => "red",
    null => "gray",
    _ => "blue"
};

// Null operators
?.   → Null-conditional (returns null instead of NullReferenceException)
??   → Null-coalescing (fallback value)
??=  → Null-coalescing assignment
```

---

## 🟢 .NET CORE & WEB API — Rapid Reference

### Request Lifecycle (Order!)
```
Kestrel → Exception MW → HTTPS → CORS → Authentication → Authorization → Controllers
                                  ↑ CORS before Auth (OPTIONS has no auth headers)
                                         ↑ Auth before Authz (must know WHO before checking WHAT)
```

### Controller Attributes
```csharp
[Route("bsk/api/[controller]")]  // Base route
[ApiController]                  // Auto-400 + binding inference
[Authorize]                      // Require valid JWT (all actions)
[AllowAnonymous]                 // Bypass auth on specific action
[HttpGet/Post/Put/Delete("path")]// HTTP verb + route
[FromBody]   [FromQuery]  [FromRoute]  [FromHeader]  // Binding sources
```

### JWT Quick Flow
```
POST /Login → Validate creds → Create JWT (Header.Payload.Signature)
Client sends: Authorization: Bearer <token>
Server: Validate signature + expiry + issuer + audience
        → Populate HttpContext.User.Claims
        → [Authorize] reads User.IsAuthenticated
```

### Middleware vs Action Filter
```
Middleware   → All requests, low-level HttpContext, outside MVC
Action Filter→ MVC only, high-level action context, model state access
```

### Polly Pattern
```
Retry       → Retry on transient failures (3 attempts, exponential backoff)
Circuit Breaker → Open after N failures, fail-fast for break duration
Timeout     → Abandon after X seconds
Bulkhead    → Limit concurrent calls to protect resources
```

### IHostedService Key Rule
```
Background services are SINGLETONS
→ Can't inject SCOPED services directly
→ Must use IServiceScopeFactory.CreateScope() inside ExecuteAsync()
```

---

## 🔴 SQL SERVER — Rapid Reference

### Join Types (One-Liner)
```
INNER JOIN  → Only matching rows in BOTH tables
LEFT JOIN   → ALL left + matched right (NULL if no match)
RIGHT JOIN  → ALL right + matched left (rarely used)
FULL JOIN   → All from both (NULL where no match)
CROSS JOIN  → Every row × every row (cartesian)
```

### Index Rules
```
Clustered    → 1 per table, stores actual data rows at leaf
Non-Clustered→ Up to 999, stores key + pointer to data
Covering     → INCLUDE extra cols → avoids key lookup
WITH NOLOCK  → Dirty read, no shared locks, for dashboards/reports only
```

### Window Functions Quick Reference
```sql
ROW_NUMBER() OVER (PARTITION BY X ORDER BY Y DESC)  -- Unique sequential per group
RANK()       -- Gaps when tied
DENSE_RANK() -- No gaps when tied
LEAD(col)    -- Next row value
LAG(col)     -- Previous row value
SUM/AVG/COUNT OVER (...)  -- Running total/average
```

### CTE Pattern
```sql
;WITH CTE_Name AS (
    SELECT ..., ROW_NUMBER() OVER (PARTITION BY CaseId ORDER BY CreatedAt DESC) AS rn
    FROM Table
)
SELECT * FROM CTE_Name WHERE rn = 1;  -- Latest record per group
```

### Performance Rules
```
✅ WITH (NOLOCK) for read-heavy reports
✅ Parameterized queries (EF Core does this automatically)
✅ SARGable predicates (no functions on indexed columns!)
✅ .AsNoTracking() for reads
✅ Covering indexes (INCLUDE columns)
❌ YEAR(CreatedAt) = 2025 → NON-SARGable!
✅ CreatedAt BETWEEN '2025-01-01' AND '2025-12-31' → SARGable
```

### ACID Quick
```
Atomicity   → All or nothing
Consistency → DB always in valid state (FK constraints honored)
Isolation   → Concurrent transactions don't see each other's uncommitted data
Durability  → Committed data survives crashes (transaction log)
```

---

## 🟡 EF CORE — Rapid Reference

### Migration Commands
```bash
dotnet ef migrations add <Name>   # Create new migration
dotnet ef database update          # Apply to DB
dotnet ef migrations remove        # Remove last (if not applied)
dotnet ef migrations script        # View SQL without executing
```

### Loading Strategies
```csharp
// Eager (JOIN in SQL — RECOMMENDED)
.Include(c => c.Claimant).ThenInclude(cl => cl.Person)

// Lazy (N+1 PROBLEM — avoid in loops!)
var name = case.Claimant.Name; // Fires extra query!

// Explicit (load on demand)
await _context.Entry(c).Reference(x => x.Claimant).LoadAsync();

// Split Query (separate SQL per collection — no cartesian explosion)
.Include(c => c.Invoices).AsSplitQuery()
```

### Key Performance Tips
```csharp
.AsNoTracking()        // Reads only — no change tracking overhead
.Select(x => new DTO)  // Project to DTO — don't load entire entity
.AsSplitQuery()        // For multiple collection includes
EF.CompileAsyncQuery() // Pre-compiled query for hot paths
```

---

## 🟤 ALGORITHMS — Rapid Reference

### Time Complexity — Must Know
```
O(1)       → Dictionary lookup, array index
O(log N)   → Binary search, heap insert/delete
O(N)       → Linear scan, single loop
O(N log N) → Merge sort, heap sort
O(N²)      → Nested loops, bubble sort
O(2^N)     → Naive recursion (Fibonacci without memoization)
```

### Algorithm Patterns
```
Two Pointers     → Reverse, palindrome, merge sorted arrays
Sliding Window   → Longest substring, max sum subarray
HashMap          → Two sum, frequency count, anagram grouping
Stack            → Balanced brackets, expression evaluation
BFS (Queue)      → Level order, shortest path in unweighted graph
DFS (Recursion)  → Tree traversal, count islands, path finding
Binary Search    → Sorted array, search insert position
Two Pointers     → Fast/Slow for linked list cycle detection
DP/Memoization   → Fibonacci, coin change, longest common subsequence
```

### Key Algorithms Code Snippets
```csharp
// Binary Search
int mid = left + (right - left) / 2; // NOT (left+right)/2 — avoids overflow!

// Floyd's Cycle Detection
while (fast != null && fast.Next != null) { slow = slow.Next; fast = fast.Next.Next; }

// Sliding Window max length
while window has invalid char: left++;
maxLen = Math.Max(maxLen, right - left + 1);

// BST Validation
Validate(node, min: null, max: null)
Left subtree: Validate(node.Left, min: min, max: node.Value)
Right subtree: Validate(node.Right, min: node.Value, max: max)
```

---

## 🔵 DESIGN PATTERNS — Rapid Reference

### SOLID (One-Line Each)
```
S → One class = one reason to change (CaseRepository only does Case DB ops)
O → Add behavior by extension, not modification (Polly policies added externally)
L → Subclass must be substitutable for base (any ICaseRepo impl works in controller)
I → Don't force unused methods (IHealthChecker separate from IHealthRegistry)
D → Depend on abstractions (Controller → ICaseAsyncRepository, not CaseAsyncRepository)
```

### Patterns Quick Recall
```
Repository   → Interface between controller and DB (ICaseAsyncRepository)
Factory      → Creates objects without specifying exact class (IHttpClientFactory)
Singleton    → One instance, use Lazy<T> (AppConfig, ServiceHealthRegistry)
Observer     → Publisher/Subscriber events (background service polling)
Decorator    → Wrap existing class with new behavior (Polly wraps HttpClient)
Strategy     → Swap algorithms at runtime (different Polly policies per vendor)
CQRS         → Separate read (Query) from write (Command) paths
Outbox       → Save event + data in same transaction → publish reliably
```

---

## 🎯 SYSTEM DESIGN — Key Concepts

### BSK Architecture in One Paragraph
> *"BSK is an N-Tier Clean Architecture ASP.NET Core 8 API with Controllers → Repositories → EF Core/Dapper → SQL Server. External integrations (7 vendors) use IHttpClientFactory with Polly resilience (retry + circuit breaker). A BackgroundService proactively monitors vendor health every 60s using a thread-safe ConcurrentDictionary registry. JWT authentication with distributed SQL cache for token blacklisting. Structured logging via Serilog to both file and console."*

### Caching Strategy (Cache-Aside Pattern)
```
1. Check cache (IDistributedCache / IMemoryCache)
2. If HIT → return cached data
3. If MISS → query DB → store in cache with TTL → return data
Use SemaphoreSlim to prevent cache stampede (only 1 thread queries DB)
```

### Circuit Breaker States
```
Closed → Open → Half-Open → Closed
  Normal  (fail fast)  (test)    (recovered)
```

---

## 📌 BSK PROJECT QUICK FACTS

| Aspect | Details |
|---|---|
| **Framework** | ASP.NET Core 8 Web API |
| **Database** | SQL Server 2022 |
| **ORM** | EF Core (CRUD) + Dapper (Dashboard/Reports) |
| **Auth** | JWT Bearer with distributed SQL cache for token blacklisting |
| **Vendors** | Surepass (KYC), Razorpay (Payments), SMS Gateway, AWS S3 |
| **Resilience** | Polly retry + circuit breaker on all 7 vendor HTTP clients |
| **Caching** | SQL Server Distributed Cache (TokensCache table) |
| **Background** | ThirdPartyHealthMonitorService + CircularListInitializerHostedService |
| **Logging** | Serilog (structured JSON logs, file + console sinks) |
| **Architecture** | N-Tier Clean Architecture, Repository Pattern |
| **DI** | Native .NET DI (Scoped repos, Singleton registries, Transient helpers) |
| **Security** | AES encryption on PAN/Aadhar via EF Value Converters |
| **SQL Patterns** | WITH (NOLOCK), covering indexes, CTEs, ROW_NUMBER(), UNION ALL |

---

## 🚨 TOP 10 INTERVIEW TRAPS — Don't Fall For These

| Trap | Correct Answer |
|---|---|
| "Use `throw ex;` to re-throw" | NO! Use `throw;` — `throw ex` resets stack trace |
| "`==` compares references for strings" | NO — C# overloads `==` for string value comparison |
| "Singleton is always thread-safe" | NO — only if implementation uses `ConcurrentDictionary` or `Lazy<T>` |
| "Lazy Loading is always bad" | NO — bad in loops (N+1). Fine for single object access |
| "Use `Count() > 0` to check if any" | NO — use `.Any()` — generates `SELECT TOP 1` / `EXISTS` |
| "`async void` is fine for fire-and-forget" | NO — exceptions in `async void` crash the process |
| "WITH NOLOCK is always safe" | NO — dirty reads! Only for read-only dashboards/reports |
| "Captive dependency is resolved by making everything Singleton" | NO — creates thread-safety issues with DbContext |
| "UNION removes duplicates automatically" | YES but UNION ALL is faster and preferred unless deduplication needed |
| "`int?` is stored differently from `int`" | YES — nullable value types use `Nullable<T>` which adds a `HasValue` bool |

---

## 🕐 Night-Before Checklist

**Evening Before Interview**:
- [ ] Read this entire cheat sheet (30 min)
- [ ] Say your self-introduction aloud (30-second and 2-minute versions)
- [ ] Review 2 STAR stories from Section 7
- [ ] Read your current project's key features (BSK project quick facts above)
- [ ] Prepare your questions for the interviewer (3-5 questions)

**Morning of Interview**:
- [ ] Review DI Lifetimes table one more time
- [ ] Review SOLID principles with BSK examples
- [ ] Double-check interview link/location details
- [ ] Eat well, stay hydrated

**Just Before Interview**:
- [ ] Close Slack, Teams, phone notifications
- [ ] Have this cheat sheet open on a second monitor (for initial waiting time)
- [ ] Take 3 deep breaths — slower speech = sounds more confident

---
*See also: [Section_10_Resume_Talking_Points.md](./Section_10_Resume_Talking_Points.md)*
