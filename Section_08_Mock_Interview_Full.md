# 🎯 Section 8: Full Mock Interview Rounds — Q&A Simulation
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> **How to use this file**: Cover the "Expected Answer" section → answer aloud → uncover to compare. Do at least 2 full mock rounds per week in the month before your interviews.

---

## 📋 Mock Interview Structure

| Round | Duration | Focus |
|---|---|---|
| **Round 1** | 45 min | C# Fundamentals + OOP |
| **Round 2** | 45 min | .NET Core + Web API + EF Core |
| **Round 3** | 45 min | SQL + Database Design |
| **Round 4** | 60 min | System Design + Architecture (BSK walkthrough) |
| **Round 5** | 30 min | Coding Round (Live Coding) |
| **Round 6** | 30 min | HR + Behavioral |

---

## 🔵 Round 1: C# Fundamentals + OOP (45 Questions)

---

### Q1: Explain OOP in one minute with a real example.
<details>
<summary>✅ Expected Answer</summary>

*"OOP has four pillars: Encapsulation — hiding internal state behind public methods (BSK's `ServiceHealthRegistry` exposes only `IsServiceUp()`, hides internal `ConcurrentDictionary`). Abstraction — showing only what's needed (`ICaseAsyncRepository` interface hides EF Core implementation). Inheritance — `ThirdPartyHealthMonitorService : BackgroundService` inherits background thread management from the base class. Polymorphism — multiple controllers all have different `GetById` implementations but called through the same `IActionResult` return type."*

</details>

---

### Q2: What is the difference between `struct` and `class` in C#?
<details>
<summary>✅ Expected Answer</summary>

*"A `struct` is a value type allocated on the stack (or inline in the parent object). It's copied when assigned. A `class` is a reference type allocated on the heap — assignment copies only the reference. Structs are good for small, immutable data like `Money(decimal Amount, string Currency)`. Classes are for complex objects with behaviour and relationships. Structs can't be null by default, can't be inherited, and are ideal when you want value-equality semantics."*

</details>

---

### Q3: What is boxing and unboxing? Why should you avoid it?
<details>
<summary>✅ Expected Answer</summary>

*"Boxing is converting a value type (like `int`) to `object` — the value gets wrapped in a heap allocation. Unboxing is casting it back. Both operations are expensive: boxing creates a heap object and a GC-tracked reference. Unboxing requires a type check and copies the value back. Use generics (`List<int>` instead of `ArrayList`) to avoid boxing in hot loops. It's the reason `ArrayList` was replaced by `List<T>` in modern C#."*

```csharp
// Boxing (avoid in hot paths)
int num = 42;
object boxed = num;         // Heap allocation!
int unboxed = (int)boxed;   // Type check + copy

// Generics avoid boxing entirely
List<int> nums = new List<int> { 42 }; // No boxing
```

</details>

---

### Q4: What is the difference between `==` and `.Equals()` for strings?
<details>
<summary>✅ Expected Answer</summary>

*"For `string`, both `==` and `.Equals()` compare **values** (content), not references. This is because C# overloads `==` for `string`. However, `.Equals()` allows passing a `StringComparison` enum for case-insensitive or culture-specific comparisons: `s.Equals('BSK', StringComparison.OrdinalIgnoreCase)`. For non-string reference types, `==` compares references (addresses) unless overloaded, while `.Equals()` can be overridden to compare values."*

</details>

---

### Q5: What is `string` interning in C#?
<details>
<summary>✅ Expected Answer</summary>

*"String interning is a .NET optimization where identical string literals are stored once in memory (in the 'intern pool') and shared. `string.Intern()` forces a string into the pool. `string.IsInterned()` checks if it's already there. This means `'hello' == 'hello'` is always true for literals (same reference). It's why you shouldn't use `ReferenceEquals` to compare strings. For dynamically created strings, interning doesn't apply automatically."*

</details>

---

### Q6: Explain `async/await` and what happens under the hood.
<details>
<summary>✅ Expected Answer</summary>

*"When a method is marked `async`, the C# compiler transforms it into a state machine. When `await` is hit, the method's execution is paused and the current thread is returned to the thread pool. When the awaited task completes (e.g., DB returns data), a thread from the pool picks up the continuation from the state machine's saved position. This allows the thread pool to serve other requests during I/O waits — crucial for scalable ASP.NET Core APIs like BSK where many requests hit the server simultaneously."*

</details>

---

### Q7: What is the difference between `Task` and `ValueTask`?
<details>
<summary>✅ Expected Answer</summary>

*"`Task` is always allocated on the heap — even for synchronous completions. `ValueTask` is a struct that avoids heap allocation when the result is already available (cache hit, fast path). Use `ValueTask` in hot paths where the method often completes synchronously — like reading from an in-memory cache. For most application code, `Task` is fine. `ValueTask` must not be awaited multiple times (it's not reusable by default)."*

</details>

---

### Q8: What is the difference between `throw` and `throw ex`?
<details>
<summary>✅ Expected Answer</summary>

*"`throw;` re-throws the current exception **preserving the original stack trace** — the line number where the exception originally occurred is visible in logs. `throw ex;` creates a **new exception** from the caught one and **resets the stack trace** to the current line — making debugging much harder. Always use bare `throw;` when re-throwing."*

```csharp
catch (Exception ex)
{
    LogHelper.Error("Failed", ex);
    throw;    // GOOD: preserves original stack trace
    // throw ex; // BAD: resets stack trace
}
```

</details>

---

### Q9: What is a delegate? How does it differ from an interface?
<details>
<summary>✅ Expected Answer</summary>

*"A delegate is a **type-safe function pointer** — it holds a reference to a method. `Action<T>` (no return), `Func<T, TResult>` (with return), and `Predicate<T>` (returns bool) are built-in delegates. An interface defines a **contract for a type** — many methods. Use delegates when you need to pass a single method as a parameter. Use interfaces when you need a contract with multiple methods. Delegates are the foundation for LINQ lambdas, events, and callbacks in .NET."*

</details>

---

### Q10: What is a sealed class? Can you seal a method?
<details>
<summary>✅ Expected Answer</summary>

*"A `sealed` class cannot be inherited. A `sealed` method in a derived class cannot be overridden further in sub-classes. Use `sealed class` for security-critical utilities (encryption helpers) or performance optimization (JIT can devirtualize calls). Use `sealed override` in a class hierarchy when you want to prevent a derived class from overriding further."*

```csharp
public sealed class EncryptionHelper { } // Cannot be inherited

public class Base { public virtual void Method() { } }
public class Derived : Base { public sealed override void Method() { } } // Can't override in grandchild
```

</details>

---

### Q11: What is covariance and contravariance in generics?
<details>
<summary>✅ Expected Answer</summary>

*"Covariance (`out T`) allows a generic type to be used where a parent type is expected. Contravariance (`in T`) allows a parent type to be substituted where a derived type is expected. Interfaces like `IEnumerable<out T>` are covariant — you can assign `IEnumerable<string>` to `IEnumerable<object>`. `Action<in T>` is contravariant. This matters when working with collections of objects and generic delegates."*

</details>

---

### Q12: What is the difference between `IEnumerable<T>` and `IQueryable<T>`?
<details>
<summary>✅ Expected Answer</summary>

*"`IEnumerable<T>` executes queries **in-memory** — LINQ operations happen on the C# side after fetching all data from the database. `IQueryable<T>` translates LINQ to **SQL** before executing — filtering happens on the database server, returning only matching rows. Always use `IQueryable<T>` with EF Core to prevent fetching millions of rows to then filter in C#. Call `.ToListAsync()` to materialize the query to `IEnumerable`."*

```csharp
// IQueryable — SQL: SELECT * FROM Cases WHERE IsActive = 1
var activeCases = _context.Cases.Where(c => c.IsActive); // Still IQueryable

// IEnumerable — Fetches ALL cases then filters in memory!
var allCases = await _context.Cases.ToListAsync(); // Materializes to IEnumerable
var filtered = allCases.Where(c => c.IsActive);   // In-memory filtering
```

</details>

---

### Q13: Explain `lock`, `Monitor`, `Mutex`, and `SemaphoreSlim` — when to use each.
<details>
<summary>✅ Expected Answer</summary>

*"`lock(obj)` is syntactic sugar for `Monitor.Enter/Exit`. Use for **synchronous** thread safety within one process. `Mutex` works across processes (OS-level). `SemaphoreSlim` is used for **async-compatible locking** — you can `await _semaphore.WaitAsync()`, unlike `lock` which blocks the thread. In BSK, we use `SemaphoreSlim` for async cache stampede prevention because you can't use `lock` inside async methods without blocking the thread pool."*

</details>

---

### Q14: What is a memory leak in a .NET application? Give a real scenario.
<details>
<summary>✅ Expected Answer</summary>

*"A memory leak in .NET occurs when objects are referenced and cannot be collected by the GC even though they're no longer needed. Classic causes: (1) Static collections that grow indefinitely — e.g., a static `List<T>` that items are added to but never removed. (2) Event handlers not unsubscribed — the publisher holds a reference to the subscriber preventing GC. (3) Captive Dependencies in DI — injecting a Scoped DbContext into a Singleton service keeps the DbContext alive forever, exhausting DB connections. Use `dotnet-dump` and `dotnet-counters` to diagnose."*

</details>

---

### Q15: What is the difference between `Dispose()` and Finalizer (`~ClassName`)?
<details>
<summary>✅ Expected Answer</summary>

*"`Dispose()` is explicitly called (or automatically via `using`) — deterministic cleanup. Finalizer (`~ClassName`) is called by the GC on an unpredictable schedule when no references remain — non-deterministic cleanup. The recommended pattern: implement `IDisposable`, call `GC.SuppressFinalize(this)` inside `Dispose()` to tell the GC not to run the finalizer (avoiding double-cleanup). Finalizers should only be a safety net for when `Dispose()` was not called."*

</details>

---

## 🟢 Round 2: .NET Core + Web API + EF Core (20 Questions)

---

### Q16: What does `[ApiController]` attribute do?
<details>
<summary>✅ Expected Answer</summary>

*"It enables 3 key behaviors: (1) **Automatic model validation** — returns HTTP 400 with `ValidationProblemDetails` if `ModelState.IsValid` is false, without you writing the check. (2) **Attribute routing required** — no convention routing. (3) **Binding source inference** — complex types inferred as `[FromBody]`, primitive query params inferred as `[FromQuery]`. It also enables Problem Details (RFC 7807) error format by default."*

</details>

---

### Q17: What is the ASP.NET Core request pipeline order? Why does order matter?
<details>
<summary>✅ Expected Answer</summary>

```
ThirdPartyExceptionMiddleware → HTTPS Redirect → CORS → Authentication → Authorization → Controllers
```

*"Order matters because each middleware wraps the next. `UseAuthentication()` MUST come before `UseAuthorization()` — otherwise when Authorization checks permissions, the user identity hasn't been established yet. CORS must be before Authentication because the browser's preflight OPTIONS request doesn't carry auth headers. Our custom exception middleware wraps everything else so it catches errors from any downstream middleware."*

</details>

---

### Q18: What is `IHostedService`? How is it different from `BackgroundService`?
<details>
<summary>✅ Expected Answer</summary>

*"`IHostedService` defines `StartAsync(CancellationToken)` and `StopAsync(CancellationToken)`. `BackgroundService` is an abstract class that implements `IHostedService` and calls `ExecuteAsync(CancellationToken)` in a background thread — so you only override `ExecuteAsync`. BSK uses `BackgroundService` for `ThirdPartyHealthMonitorService` and `CircularListInitializerHostedService`. Key rule: they're registered as Singletons and can't directly inject Scoped services — must use `IServiceScopeFactory.CreateScope()` inside `ExecuteAsync`."*

</details>

---

### Q19: What is the N+1 problem in EF Core? How do you fix it?
<details>
<summary>✅ Expected Answer</summary>

*"N+1 occurs when you load N parent entities and then access a navigation property for each — causing 1 initial query + N additional queries. Example: Load 100 cases (1 query), then loop and access `case.Claimant.Name` with lazy loading (100 more queries) = 101 total. Fix: Use Eager Loading with `.Include(c => c.Claimant)` which generates a single SQL JOIN. Alternatively, for large collections, use `.AsSplitQuery()` to generate separate queries per collection without Cartesian explosion."*

</details>

---

### Q20: Explain the difference between Transient, Scoped, and Singleton DI lifetimes with a bug scenario.
<details>
<summary>✅ Expected Answer</summary>

*"Transient = new instance every injection. Scoped = new instance per HTTP request (shared within same request). Singleton = one instance for app lifetime. The classic bug is a **Captive Dependency**: injecting a Scoped `DbContext` into a Singleton service. The Singleton holds the DbContext forever — it never gets disposed, causing DB connection pool exhaustion and stale entity tracking. Fix: inject `IServiceScopeFactory` into the Singleton and create a temporary scope inside each method execution."*

</details>

---

### Q21: How does JWT authentication work in ASP.NET Core? Walk through the full flow.
<details>
<summary>✅ Expected Answer</summary>

*"(1) User POSTs credentials to `/Login`. (2) Server validates credentials against DB. (3) Server creates JWT: Header (algorithm) + Payload (UserId, Role, Expiry claims) + Signature (HMAC-SHA256 signed with secret key). (4) Token sent to client. (5) Client stores token and sends it as `Authorization: Bearer <token>` on subsequent requests. (6) ASP.NET Core's JWT middleware validates: signature (was it tampered?), expiry, issuer, audience. If valid, populates `HttpContext.User` with claims. (7) `[Authorize]` checks `HttpContext.User` has valid identity."*

</details>

---

### Q22: What is CORS? Why does the browser enforce it but Postman doesn't?
<details>
<summary>✅ Expected Answer</summary>

*"CORS is a browser security policy that prevents JavaScript on one origin (domain) from making requests to a different origin. The browser enforces it — Postman is not a browser and doesn't follow browser security policies, so it ignores CORS. In ASP.NET Core, we configure allowed origins with `AddCors()`. For credential-bearing requests (JWT cookies), `AllowCredentials()` must be combined with specific origins (not wildcard `*`). The browser sends a preflight OPTIONS request first for non-simple requests — the server must respond with correct CORS headers."*

</details>

---

### Q23: How do you prevent SQL injection in ASP.NET Core?
<details>
<summary>✅ Expected Answer</summary>

*"EF Core uses parameterized queries automatically — user input is always treated as a literal parameter value, never executed as SQL code. For raw SQL, use `FromSqlInterpolated($'...{id}...')` which parameterizes interpolated values. NEVER use string concatenation to build SQL. With Dapper, always use parameterized queries: `QueryAsync('SELECT * FROM Cases WHERE Id = @Id', new { Id = id })` — never string-concatenate user input into the SQL string."*

</details>

---

### Q24: What is Socket Exhaustion? How does `IHttpClientFactory` solve it?
<details>
<summary>✅ Expected Answer</summary>

*"Creating `new HttpClient()` in a `using` block disposes the object but keeps the underlying OS socket in `TIME_WAIT` state for up to 4 minutes. Under high traffic, the OS runs out of sockets and throws `SocketException`. `IHttpClientFactory` manages a pool of `HttpMessageHandler` instances that are recycled — sockets are reused efficiently. It also automatically respects DNS TTL changes (raw HttpClient caches DNS forever). In BSK, we register named clients (`SurepassClient`, `AccountingServiceClient`) in `Program.cs` and inject `IHttpClientFactory` in repositories."*

</details>

---

### Q25: What is `.AsNoTracking()` in EF Core? When do you use it?
<details>
<summary>✅ Expected Answer</summary>

*"`.AsNoTracking()` disables EF Core's change tracker for a query — entities returned are not monitored for changes. Benefits: (1) Faster query execution — no overhead of tracking entity state. (2) Less memory — tracked entities stay in DbContext's identity map. Use it for ALL read-only queries (GET endpoints, dashboard reports). Only use tracking (default) when you need to UPDATE entities — EF needs to detect what changed. In BSK, all repository read methods use `.AsNoTracking()` as standard practice."*

</details>

---

## 🔴 Round 3: SQL & Database (15 Questions)

---

### Q26: What is the difference between a Clustered and Non-Clustered index?
<details>
<summary>✅ Expected Answer</summary>

*"A Clustered Index stores the actual data rows at the leaf level — it physically reorders the table rows. There can be only ONE per table (data can only be sorted one way). A Non-Clustered Index stores index keys plus pointers (bookmarks) to the actual data rows. Up to 999 per table. Created automatically with PRIMARY KEY (clustered) and manually for query optimization. A table without a clustered index is called a Heap."*

</details>

---

### Q27: What is a covering index? When do you need it?
<details>
<summary>✅ Expected Answer</summary>

*"A Key Lookup occurs when a non-clustered index finds the matching key but the query needs columns not in that index — so SQL jumps back to the clustered index to fetch them. This is expensive. A Covering Index includes extra columns using the `INCLUDE` clause so SQL can serve the query entirely from the non-clustered index without the key lookup.*

```sql
CREATE NONCLUSTERED INDEX IX_Case_ClaimantId
ON [Case] (ClaimantId)
INCLUDE (ClaimAmount, StatusId, CreatedAt); -- No key lookup for these columns
```

*In BSK, we added covering indexes on frequently filtered columns with common SELECT columns to eliminate key lookups visible in execution plans."*

</details>

---

### Q28: What is WITH (NOLOCK)? When should you use it?
<details>
<summary>✅ Expected Answer</summary>

*"`WITH (NOLOCK)` is a SQL Server table hint that allows dirty reads — reading data without acquiring shared locks, even if another transaction has modified those rows but not committed yet. BSK uses it extensively on dashboard queries to prevent read operations from blocking concurrent write operations (claim registrations). Risks: you might read uncommitted data that gets rolled back (phantom reads). Acceptable for reporting/analytics dashboards where approximate data is fine, but NEVER for financial transactions, balance calculations, or any operation requiring exact current state."*

</details>

---

### Q29: Write a SQL query to find the second-highest salary.
<details>
<summary>✅ Expected Answer</summary>

```sql
-- Method 1: Subquery
SELECT MAX(ClaimAmount) AS SecondHighest
FROM [Case]
WHERE ClaimAmount < (SELECT MAX(ClaimAmount) FROM [Case]);

-- Method 2: DENSE_RANK (handles ties correctly)
SELECT ClaimAmount FROM (
    SELECT ClaimAmount, DENSE_RANK() OVER (ORDER BY ClaimAmount DESC) AS RNK
    FROM [Case]
) ranked
WHERE RNK = 2;
```

*"DENSE_RANK is preferred because it handles ties — if two cases have the same highest amount, they both get rank 1, and the next distinct amount gets rank 2."*

</details>

---

### Q30: What is the difference between `DELETE`, `TRUNCATE`, and `DROP`?
<details>
<summary>✅ Expected Answer</summary>

| Feature | DELETE | TRUNCATE | DROP |
|---|---|---|---|
| Scope | Specific rows | All rows | Entire table + structure |
| WHERE clause | YES | NO | NO |
| Transaction logged | YES (row by row) | Minimal (page dealloc) | NO |
| Rollback possible | YES | YES (within transaction) | NO |
| Resets IDENTITY | NO | YES | N/A |
| Triggers fired | YES | NO | NO |

</details>

---

### Q31: What is a CTE? Write a recursive CTE for an organizational hierarchy.
<details>
<summary>✅ Expected Answer</summary>

```sql
-- Non-recursive CTE (BSK dashboard pattern)
WITH LatestStages AS (
    SELECT CaseId, ROW_NUMBER() OVER(PARTITION BY CaseId ORDER BY CreatedAt DESC) AS rn
    FROM CaseLog
)
SELECT * FROM Cases c LEFT JOIN LatestStages ls ON c.Id = ls.CaseId AND ls.rn = 1;

-- Recursive CTE (org hierarchy)
WITH OrgHierarchy AS (
    -- Anchor: Start with top-level employees (no manager)
    SELECT Id, Name, ManagerId, 0 AS Level
    FROM Employee WHERE ManagerId IS NULL

    UNION ALL

    -- Recursive: Find employees reporting to previous level
    SELECT e.Id, e.Name, e.ManagerId, oh.Level + 1
    FROM Employee e
    INNER JOIN OrgHierarchy oh ON e.ManagerId = oh.Id
)
SELECT * FROM OrgHierarchy ORDER BY Level, Name;
```

</details>

---

### Q32: What is a Deadlock? How do you prevent it in SQL Server?
<details>
<summary>✅ Expected Answer</summary>

*"A deadlock occurs when Thread A holds Lock 1 and waits for Lock 2, while Thread B holds Lock 2 and waits for Lock 1 — neither can proceed. SQL Server detects this and kills the 'deadlock victim' (the cheaper-to-undo transaction). Prevention strategies: (1) Access tables in consistent order across all transactions. (2) Use `WITH (NOLOCK)` for reads (dirty reads acceptable). (3) Keep transactions short. (4) Use `READ COMMITTED SNAPSHOT ISOLATION (RCSI)` — readers don't block writers. (5) Ensure adequate indexing so queries hold locks briefly."*

</details>

---

## 🟡 Round 4: System Design (10 Questions)

---

### Q33: Design a rate-limiting system for BSK's API.
<details>
<summary>✅ Expected Answer</summary>

*"For BSK, I'd use the built-in .NET 7+ `RateLimiter` middleware with a Fixed Window policy for general endpoints and Token Bucket for upload/document endpoints (allows burst). Registration:*

```csharp
builder.Services.AddRateLimiter(options => {
    options.AddFixedWindowLimiter("Standard", opt => {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 10;
    });
    options.RejectionStatusCode = 429;
});
```

*For distributed rate limiting (multi-server), use Redis with a sliding window algorithm — store request counts per client IP/token in Redis with TTL. The current in-memory limiter only works for single-server deployments."*

</details>

---

### Q34: How would you design a notification system for BSK (email + SMS on claim status changes)?
<details>
<summary>✅ Expected Answer</summary>

*"(1) Claim status changes in the API layer. (2) Instead of directly calling email/SMS providers synchronously, publish a `ClaimStatusChangedEvent` to a message queue (RabbitMQ via MassTransit or Azure Service Bus). (3) A separate `NotificationService` subscribes to this event, determines the notification channels (email for claimant, SMS for agent), and calls respective providers. (4) Use the Outbox Pattern to ensure the event is published reliably — save event to DB in the same transaction as the status change, a background worker publishes it. (5) This decouples notification failures from claim processing — a SendGrid outage doesn't affect claim workflows."*

</details>

---

### Q35: Walk me through how you'd handle a database schema migration in production without downtime.
<details>
<summary>✅ Expected Answer</summary>

*"Zero-downtime migrations require backward-compatible changes in multiple phases: (1) **Add column as nullable**: `ALTER TABLE Case ADD NewColumn NVARCHAR(255) NULL` — old code still works, new column is optional. (2) Deploy new API code that reads/writes the new column. (3) Backfill existing rows in batches: `UPDATE CASE SET NewColumn = default WHERE NewColumn IS NULL AND Id BETWEEN @start AND @end`. (4) Once all rows backfilled and new code deployed, make column NOT NULL. (5) Remove old code that used the old pattern. Avoid adding NOT NULL columns without defaults in a single migration — SQL Server must lock the entire table to backfill."*

</details>

---

### Q36: How would you add full-text search to BSK's document repository?
<details>
<summary>✅ Expected Answer</summary>

*"For BSK's document search: (1) **Short-term**: SQL Server Full-Text Search — enable on relevant tables, use `CONTAINS()` or `FREETEXT()` predicates. Simple but limited to keyword matching. (2) **Better option**: Elasticsearch — index document metadata + extracted text. Supports fuzzy search, typo tolerance, ranking by relevance. Use `NEST` or `Elastic.Clients.Elasticsearch` .NET client. (3) **For semantic search**: Generate text embeddings using OpenAI `text-embedding-3-small`, store in a vector database (pgvector on PostgreSQL or Qdrant). Query using cosine similarity — 'medical claim' matches 'health insurance' even without keyword overlap."*

</details>

---

### Q37: Describe the Circuit Breaker pattern. What states does it have?
<details>
<summary>✅ Expected Answer</summary>

*"Circuit Breaker prevents cascading failures when calling unreliable external services. It has 3 states: (1) **Closed** — normal operation, requests pass through. Failures are counted. (2) **Open** — threshold exceeded (e.g., 50% failure rate). All requests fail immediately (fast fail) without attempting the external call. Timer starts. (3) **Half-Open** — after the break duration (e.g., 15s), one test request is allowed through. If it succeeds → back to Closed. If it fails → back to Open. BSK uses Polly's `AddCircuitBreaker()` for all 7 vendor HTTP clients."*

</details>

---

## 💻 Round 5: Live Coding (Solve in ~15 min)

---

### Coding Q38: Valid Parentheses
**Problem**: Given `"({[]})"` → `true`. Given `"({)}"` → `false`.

<details>
<summary>✅ Expected Solution</summary>

```csharp
public static bool IsValid(string s)
{
    var stack = new Stack<char>();
    var pairs = new Dictionary<char, char>
    {
        { ')', '(' }, { ']', '[' }, { '}', '{' }
    };

    foreach (char c in s)
    {
        if (pairs.ContainsValue(c))
        {
            stack.Push(c);
        }
        else if (pairs.ContainsKey(c))
        {
            if (stack.Count == 0 || stack.Pop() != pairs[c])
                return false;
        }
    }
    return stack.Count == 0;
}
// Time: O(N), Space: O(N)
```

</details>

---

### Coding Q39: Find Longest Common Prefix
**Problem**: Given `["flower","flow","flight"]` → `"fl"`.

<details>
<summary>✅ Expected Solution</summary>

```csharp
public static string LongestCommonPrefix(string[] strs)
{
    if (strs.Length == 0) return "";
    string prefix = strs[0];

    for (int i = 1; i < strs.Length; i++)
    {
        // Trim prefix until current string starts with it
        while (!strs[i].StartsWith(prefix))
        {
            prefix = prefix.Substring(0, prefix.Length - 1);
            if (prefix.Length == 0) return "";
        }
    }
    return prefix;
}
// Time: O(S) where S = total chars. Space: O(1) extra
```

</details>

---

### Coding Q40: Maximum Subarray Sum (Kadane's Algorithm)
**Problem**: Given `[-2,1,-3,4,-1,2,1,-5,4]` → `6` (subarray `[4,-1,2,1]`).

<details>
<summary>✅ Expected Solution</summary>

```csharp
public static int MaxSubArray(int[] nums)
{
    int currentSum = nums[0];
    int maxSum = nums[0];

    for (int i = 1; i < nums.Length; i++)
    {
        // Either extend current subarray or start fresh from current element
        currentSum = Math.Max(nums[i], currentSum + nums[i]);
        maxSum = Math.Max(maxSum, currentSum);
    }
    return maxSum;
}
// Time: O(N), Space: O(1) — Kadane's Algorithm
// Explain: If adding nums[i] makes currentSum negative, restart from nums[i]
```

</details>

---

### Coding Q41: Count Islands (Graph BFS/DFS)
**Problem**: Given a 2D grid of `'1'` (land) and `'0'` (water), count number of islands.

<details>
<summary>✅ Expected Solution</summary>

```csharp
public static int NumIslands(char[][] grid)
{
    int count = 0;
    for (int r = 0; r < grid.Length; r++)
    {
        for (int c = 0; c < grid[0].Length; c++)
        {
            if (grid[r][c] == '1')
            {
                count++;
                Dfs(grid, r, c); // Sink the island
            }
        }
    }
    return count;
}

private static void Dfs(char[][] grid, int r, int c)
{
    if (r < 0 || r >= grid.Length || c < 0 || c >= grid[0].Length || grid[r][c] != '1')
        return;

    grid[r][c] = '0'; // Mark visited (sink the land)

    Dfs(grid, r + 1, c); // Down
    Dfs(grid, r - 1, c); // Up
    Dfs(grid, r, c + 1); // Right
    Dfs(grid, r, c - 1); // Left
}
// Time: O(M×N), Space: O(M×N) recursion stack in worst case
```

</details>

---

### Coding Q42: Remove Nth Node From End of List
**Problem**: Given a linked list, remove the nth node from the end.

<details>
<summary>✅ Expected Solution</summary>

```csharp
public static ListNode RemoveNthFromEnd(ListNode head, int n)
{
    // Dummy node to handle edge case of removing the head
    var dummy = new ListNode(0) { Next = head };
    var fast = dummy;
    var slow = dummy;

    // Move fast n+1 steps ahead
    for (int i = 0; i <= n; i++) fast = fast.Next;

    // Move both until fast reaches end
    while (fast != null)
    {
        fast = fast.Next;
        slow = slow.Next;
    }

    // slow.Next is the node to remove
    slow.Next = slow.Next.Next;

    return dummy.Next;
}
// Two-pointer technique: gap of n between fast and slow.
// When fast hits null, slow is at the node BEFORE the one to delete.
```

</details>

---

## 🟤 Round 6: Rapid Fire (Answer in 30 seconds each)

### Q43–Q60: Rapid Fire Questions

| # | Question | Expected Answer |
|---|---|---|
| 43 | What is `record` vs `class`? | `record` has value equality, immutable by default, `with` expression for mutation |
| 44 | What is `init` accessor? | Property can only be set during object initialization (constructor or initializer) |
| 45 | What is `required` modifier (C#11)? | Property must be set in object initializer — compile error if missing |
| 46 | What is pattern matching `is`? | `if (obj is CaseModel c)` — type check + cast + null check in one |
| 47 | What is `Span<T>`? | Stack-allocated memory view — zero heap allocation for string slicing operations |
| 48 | What is `stackalloc`? | Allocate array on stack — no GC, must use with `Span<T>` |
| 49 | What is `ArrayPool<T>`? | Rent/Return arrays from shared pool — avoids LOH allocations |
| 50 | Primary Key vs Unique Key? | PK: no NULL, exactly 1 per table; UK: allows one NULL, multiple per table |
| 51 | `INNER JOIN` vs `LEFT JOIN`? | INNER: only matching rows; LEFT: all left rows + matched right (NULL if no match) |
| 52 | What is a Heap in SQL? | A table with no clustered index — rows in no particular order |
| 53 | What is `SCOPE_IDENTITY()`? | Last identity value generated in current scope — safer than `@@IDENTITY` |
| 54 | What is EF Core `AsOf()`? | Temporal table query — get data as it existed at a specific point in time |
| 55 | What is `Polly`? | .NET resilience library for retry, circuit breaker, timeout, bulkhead policies |
| 56 | What is `MassTransit`? | .NET library abstracting RabbitMQ, Azure Service Bus message publishing |
| 57 | What is Kestrel? | Cross-platform, high-performance HTTP server built into ASP.NET Core |
| 58 | What is YARP? | Yet Another Reverse Proxy — Microsoft's .NET-native API Gateway library |
| 59 | What is `dotnet-counters`? | Real-time .NET performance metrics monitor (GC, thread pool, request rates) |
| 60 | What is Native AOT? | Ahead-of-Time compilation — no JIT at runtime, faster startup, smaller footprint |

---

## 📊 Mock Interview Self-Scoring Guide

After each mock round, rate yourself honestly:

| Score | Meaning | Action |
|---|---|---|
| 9-10/10 | Answered correctly with confidence | Move on |
| 7-8/10 | Answered but missed key details | Review once |
| 5-6/10 | Partial answer, significant gaps | Study section + practice aloud |
| <5/10 | Couldn't answer | Full re-read of relevant section |

---

## 🎯 Mock Interview Schedule

| Date | Round | Focus |
|---|---|---|
| Week 1, Day 1 | Round 1 (C# Fundamentals) | Q1-Q15 |
| Week 1, Day 3 | Round 2 (.NET Core) | Q16-Q25 |
| Week 1, Day 5 | Round 3 (SQL) | Q26-Q32 |
| Week 2, Day 1 | Round 4 (System Design) | Q33-Q37 |
| Week 2, Day 3 | Round 5 (Live Coding) | Q38-Q42 |
| Week 2, Day 5 | Full Mock (All rounds) | Q1-Q60 |
| Week 3+ | Repeat targeting your weak areas | Based on scores |

---
*Next: See [Section_09_Quick_Revision_Cheatsheet.md](./Section_09_Quick_Revision_Cheatsheet.md)*
