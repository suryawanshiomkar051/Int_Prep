# 🏗️ Section 5: System Design, Architecture & Advanced Topics
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> These topics differentiate **Senior/Mid-level** candidates. Even at 2.5 years, understanding and articulating these shows depth.

---

## 📋 Table of Contents

1. [BSK Architecture — N-Tier Clean Architecture](#1-bsk-architecture--n-tier-clean-architecture)
2. [Design Patterns (Advanced)](#2-design-patterns-advanced)
3. [Microservices vs Monolith](#3-microservices-vs-monolith)
4. [Security Best Practices](#4-security-best-practices)
5. [Docker & DevOps Basics](#5-docker--devops-basics)
6. [Distributed Systems Concepts](#6-distributed-systems-concepts)
7. [Code Quality & Best Practices](#7-code-quality--best-practices)
8. [Coding Round Algorithms (C#)](#8-coding-round-algorithms-c)
9. [BSK Project Scenarios — STAR Answers](#9-bsk-project-scenarios--star-answers)
10. [Quick-Fire Advanced Checklist](#10-quick-fire-advanced-checklist)

---

## 1. BSK Architecture — N-Tier Clean Architecture

### Q1: Describe the BSK Application Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: Presentation Layer (Controllers)                       │
│  CaseController, DocumentController, RoleController, etc.        │
│  - HTTP routing, model binding, response formatting              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Layer 2: Infrastructure / Services Layer                         │
│  Polly Policies, Serilog, AWS S3 Adapters, ThirdParty Clients    │
│  - IHostedService (health monitor, circular assignment)          │
│  - External API clients (Surepass, Razorpay, SMS gateway)        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Layer 3: Data Access Layer (Repositories + EF Core / Dapper)    │
│  ICaseAsyncRepository → CaseAsyncRepository                      │
│  - EF Core for CRUD, Dapper for complex dashboard queries        │
│  - ApplicationDBContext (Unit of Work)                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Layer 4: Persistent Storage                                      │
│  SQL Server 2022 (OLTP) + AWS S3 (Documents)                    │
└─────────────────────────────────────────────────────────────────┘
```

**BSK External Integrations**:
- Surepass (KYC — PAN, Aadhar verification)
- Razorpay (Payment gateway webhooks)
- SMS Gateway (OTP, notifications)
- BSK Accounting Microservice (Separate service for ledger/vouchers)
- AWS S3 (Document storage)

---

### Q2: BSK's Third-Party Health Monitor — Key Feature Deep Dive

```
Program.cs registers Singleton: IServiceHealthRegistry
Program.cs registers HostedService: ThirdPartyHealthMonitorService
                              ↓
PeriodicTimer fires every 60 seconds
                              ↓
Probes health URLs of all vendors (Surepass, SMS, Accounting)
                              ↓
Updates thread-safe IServiceHealthRegistry (ConcurrentDictionary)
                              ↓
Before any vendor API call, repositories check IServiceHealthGuard
_healthGuard.EnsureServiceIsUp(ThirdPartyService.Surepass);
  → Throws ThirdPartyServiceUnavailableException if DOWN
  → Caught by ThirdPartyExceptionMiddleware → Returns 503 response
```

**Why this is important**:
- Prevents **connection timeout waste** — if Surepass is down, fail fast instead of waiting 30s
- Proactive monitoring vs reactive failure
- 503 response instead of hanging indefinitely

---

## 2. Design Patterns (Advanced)

### Q3: CQRS + MediatR Pattern

**CQRS (Command Query Responsibility Segregation)**:
- **Query** = Read data (GET) — return data, no state changes
- **Command** = Write data (POST/PUT/DELETE) — change state, minimal return data

```csharp
// Command (write operation)
public record RegisterCaseCommand(Guid ClaimantId, decimal ClaimAmount) : IRequest<Guid>;

// Query (read operation)
public record GetCaseByIdQuery(Guid CaseId) : IRequest<CaseSummaryDto>;

// Controller — just sends to MediatR bus
[HttpPost("RegisterCase")]
public async Task<IActionResult> RegisterCase([FromBody] RegisterCaseCommand command)
{
    var caseId = await _mediator.Send(command); // MediatR routes to handler
    return Ok(caseId);
}

// Command Handler — business logic here
public class RegisterCaseCommandHandler : IRequestHandler<RegisterCaseCommand, Guid>
{
    private readonly ICaseRepository _repo;
    public RegisterCaseCommandHandler(ICaseRepository repo) => _repo = repo;

    public async Task<Guid> Handle(RegisterCaseCommand request, CancellationToken ct)
    {
        var newCase = new Case { ClaimantId = request.ClaimantId, ClaimAmount = request.ClaimAmount };
        return await _repo.SaveAsync(newCase);
    }
}
```

---

### Q4: Outbox Pattern — Ensuring Event Reliability

**Problem**: API saves Case to DB ✓, then tries to publish event to RabbitMQ ✗ (network drops) → **Data inconsistency**!

**Solution (Transactional Outbox)**:
1. Save Case + OutboxMessage in same SQL transaction (atomic)
2. Background worker polls OutboxMessages table → publishes to RabbitMQ → marks as processed

```csharp
// Same transaction: Save Case + event payload
using var transaction = await _context.Database.BeginTransactionAsync();

_context.Cases.Add(newCase);
_context.OutboxMessages.Add(new OutboxMessage
{
    Payload = JsonSerializer.Serialize(new CaseRegisteredEvent(newCase.Id)),
    CreatedAt = DateTime.UtcNow
});

await _context.SaveChangesAsync(); // BOTH saved atomically
await transaction.CommitAsync();

// Background Worker processes outbox
public class OutboxProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            using var scope = _serviceProvider.CreateScope();
            var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDBContext>();
            var publisher = scope.ServiceProvider.GetRequiredService<IPublishEndpoint>();

            var messages = await dbContext.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .Take(20)
                .ToListAsync();

            foreach (var msg in messages)
            {
                await publisher.Publish(JsonSerializer.Deserialize<CaseRegisteredEvent>(msg.Payload));
                msg.ProcessedAt = DateTime.UtcNow;
            }
            await dbContext.SaveChangesAsync();
        }
    }
}
```

---

### Q5: Saga Pattern — Distributed Transactions

**Problem**: In microservices, `Case Service` and `Accounting Service` have separate databases. ACID across services is impossible.

**Saga** = Sequence of local transactions. If one fails → run **compensating transactions** (undo previous steps).

```
[Choreography Saga]:
Case Service ──►(CaseCreatedEvent)──► Accounting Service ──►(VoucherIssuedEvent)──► Notification Service
                                             │ (Payment FAILS!)
                                             └──►(CancelCaseEvent)──► Case Service (Compensating Rollback)
```

**Types**:
- **Choreography** (Decentralized) — Each service publishes events, next service reacts. Simple but hard to track.
- **Orchestration** (Centralized) — Dedicated Saga Orchestrator tells each service what to do. Better for complex flows.

---

## 3. Microservices vs Monolith

### Q6: Monolith vs Microservices — BSK context

**BSK Architecture**: Primarily a **Monolithic REST API** (single deployable unit) with **Service-Oriented integration** to other microservices (Accounting Service).

| Feature | Monolith (BSK Core API) | Microservices (BSK Accounting Service) |
|---|---|---|
| **Deployment** | Single deployable unit | Independent deployment per service |
| **Communication** | Internal method calls | HTTP/gRPC/Message queues |
| **Data** | Shared SQL Server | Each service owns its DB |
| **Scaling** | Scale the whole app | Scale only the services under load |
| **Complexity** | Simpler to start | Higher operational complexity |

**BSK communicates with Accounting Service via HTTP**:
```csharp
builder.Services.AddHttpClient("AccountingServiceClient", client =>
    client.BaseAddress = new Uri("https://bsk-stg-accountingsvc.shauryatechnosoft.com"))
    .AddResilienceHandler(PollyPolicies.AccountingServicePipeline, PollyPolicies.ConfigureHttpPipeline);
```

---

### Q7: Domain-Driven Design (DDD) — Entity vs Value Object

```csharp
// Entity — defined by IDENTITY (ID), mutable
public class Claimant : Entity
{
    public Guid Id { get; private set; }  // Identity
    public string Name { get; set; }       // Mutable attribute — can change
}

// Value Object — defined by VALUES, immutable
public class Address
{
    public string Street { get; }
    public string City { get; }
    public string ZipCode { get; }

    public Address(string street, string city, string zipCode)
    { Street = street; City = city; ZipCode = zipCode; }

    // Value objects override equality by VALUE (not reference)
    public override bool Equals(object obj) =>
        obj is Address other && Street == other.Street && City == other.City && ZipCode == other.ZipCode;
}

// Two Address objects with same values = EQUAL
var a1 = new Address("MG Road", "Pune", "411001");
var a2 = new Address("MG Road", "Pune", "411001");
a1.Equals(a2); // TRUE — Value equality
```

---

## 4. Security Best Practices

### Q8: OWASP Top 10 — ASP.NET Core protections

**1. SQL Injection**:
```csharp
// EF Core uses parameterized queries automatically
var cases = await _context.Cases.Where(c => c.Id == id).ToListAsync();
// Generated SQL: WHERE Id = @p0  — @p0 is parameterized, never executed as code!

// If using raw SQL — ALWAYS parameterize
var cases = await _context.Cases.FromSqlInterpolated($"SELECT * FROM [Case] WHERE Id = {id}").ToListAsync();
// NEVER: $"SELECT * FROM [Case] WHERE Id = {userInput}"  ← SQL Injection!
```

**2. CSRF (Cross-Site Request Forgery)**:
- BSK uses JWT in `Authorization` header (not cookies) → CSRF attacks are mitigated
- Cookie-based apps: Use `[ValidateAntiForgeryToken]` attribute

**3. XSS (Cross-Site Scripting)**:
- JSON APIs automatically escape special characters
- Never insert unescaped HTML from user input

**4. Secrets Management**:
```csharp
// BAD: Hardcoded in appsettings.json (committed to Git)
"JwtSecret": "my_secret_key_here"

// GOOD: Environment Variables (Docker/CI/CD)
"JwtSecret": "" // Read from env: JWT__Secret

// BEST: Azure Key Vault / AWS Secrets Manager
builder.Configuration.AddAzureKeyVault(
    new Uri("https://bsk-prod-vault.vault.azure.net/"),
    new DefaultAzureCredential());
```

---

### Q9: AES Encryption for Sensitive Data (PAN, Aadhar)

```csharp
public static class EncryptionHelper
{
    private static readonly byte[] Key = Encoding.UTF8.GetBytes("32_byte_secret_key_for_aes_256!"); // From Key Vault!
    private static readonly byte[] Iv  = Encoding.UTF8.GetBytes("16_byte_iv_vector");

    public static string Encrypt(string plainText)
    {
        using var aes = Aes.Create();
        aes.Key = Key; aes.IV = Iv;
        using var encryptor = aes.CreateEncryptor();
        using var ms = new MemoryStream();
        using (var cs = new CryptoStream(ms, encryptor, CryptoStreamMode.Write))
        using (var sw = new StreamWriter(cs))
            sw.Write(plainText);
        return Convert.ToBase64String(ms.ToArray());
    }

    public static string Decrypt(string cipherText)
    {
        using var aes = Aes.Create();
        aes.Key = Key; aes.IV = Iv;
        using var decryptor = aes.CreateDecryptor();
        using var ms = new MemoryStream(Convert.FromBase64String(cipherText));
        using var cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Read);
        using var sr = new StreamReader(cs);
        return sr.ReadToEnd();
    }
}

// Applied via EF Core Value Converter (auto-encrypt/decrypt transparently)
// See Section_04_EF_Core_ORM.md — Q3
```

---

## 5. Docker & DevOps Basics

### Q10: Multi-Stage Docker Build for BSK API

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["BSKAPI/BSKAPI.csproj", "BSKAPI/"]
RUN dotnet restore "BSKAPI/BSKAPI.csproj"
COPY . .
WORKDIR "/src/BSKAPI"
RUN dotnet build "BSKAPI.csproj" -c Release -o /app/build

# Stage 2: Publish (optimized, trimmed binaries)
FROM build AS publish
RUN dotnet publish "BSKAPI.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Stage 3: Lightweight production image (~100MB vs 800MB SDK)
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS final
WORKDIR /app
EXPOSE 8080
COPY --from=publish /app/publish .
USER $APP_UID  # Run as non-root user (security best practice)
ENTRYPOINT ["dotnet", "BSKAPI.dll"]
```

**Why multi-stage?**: Final image is ~100MB (just runtime), not 800MB (SDK + source code). Compiler tools NOT in production = smaller attack surface.

---

### Q11: CI/CD Pipeline — GitHub Actions

```yaml
name: .NET Core CI Pipeline

on:
  push:
    branches: ["main", "develop"]
  pull_request:
    branches: ["main"]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup .NET SDK 8.0
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'

    - name: Restore Dependencies
      run: dotnet restore

    - name: Build
      run: dotnet build --no-restore --configuration Release

    - name: Run Unit Tests
      run: dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage"
```

---

## 6. Distributed Systems Concepts

### Q12: Cache Stampede (Thundering Herd) Problem & Solution

**Problem**: A popular cache item expires. 1000 simultaneous requests → all experience cache miss → all query DB → DB overloaded!

**Solution**: Distributed locking — only the FIRST thread queries DB, others wait:
```csharp
private static readonly SemaphoreSlim _lock = new(1, 1);

public async Task<string> GetDashboardDataAsync()
{
    var data = await _cache.GetStringAsync("dashboard");
    if (data != null) return data; // Cache hit

    await _lock.WaitAsync(); // Only 1 thread enters here
    try
    {
        data = await _cache.GetStringAsync("dashboard"); // Double-check!
        if (data != null) return data;

        data = await QueryHeavyMetricsAsync(); // Only ONE DB call
        await _cache.SetStringAsync("dashboard", data,
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
    }
    finally { _lock.Release(); }

    return data;
}
```

---

### Q13: Rate Limiting Algorithms

| Algorithm | How it works | Best For |
|---|---|---|
| **Fixed Window** | Max N requests per time window. Resets at window end. | Simple, predictable limits |
| **Sliding Window** | Counts requests in a rolling time window. Smoother. | More accurate rate control |
| **Token Bucket** | Virtual bucket with N tokens. Tokens added at constant rate. Allows bursts. | APIs that allow occasional bursts |
| **Leaky Bucket** | Requests processed at constant rate. Queue absorbs bursts. | Smooth, steady output rate |

---

### Q14: API Gateway & BFF Pattern

**API Gateway**: Single entry point for all clients → routes to appropriate microservices.
- Authentication at gateway (once) instead of in each service
- Rate limiting, IP whitelisting
- Protocol translation (HTTPS → gRPC)

**BFF (Backend for Frontend)**:
- **BFF-Mobile**: Lightweight payloads, aggregated responses for mobile bandwidth
- **BFF-Web**: Richer data, cookie-based sessions, heavier JSON

**YARP** = Microsoft's production-grade reverse proxy for .NET.

---

### Q15: Blue-Green vs Canary Deployment

```
Blue-Green:
Client ──► Load Balancer ──► [Blue v1.0  (ACTIVE)]
                         └──► [Green v2.0 (IDLE — test here)]
Switch router when Green is verified → instant switchover, instant rollback

Canary:
Client ──► Router ──► 95% traffic → [v1.0]
                 └──►  5% traffic → [v2.0] (test with small user base)
Gradually increase % → 20% → 50% → 100%
```

| Feature | Blue-Green | Canary |
|---|---|---|
| **Traffic Switch** | 100% at once | Gradual (5% → 100%) |
| **Cost** | High (2x infrastructure) | Low (uses same cluster) |
| **Rollback** | Instant (flip router) | Slow (ramp down %) |
| **Risk** | Full blast or none | Limited blast radius |

---

## 7. Code Quality & Best Practices

### Q16: Code Review Checklist (BSK Standard)

1. **Functional Compliance** — Does it meet the requirement?
2. **Async Patterns** — All async calls properly awaited? No `.Result` or `.Wait()`?
3. **Security** — No hardcoded API keys? Input validated?
4. **Performance** — Using `.AsNoTracking()` for reads? Efficient joins?
5. **Error Handling** — Exceptions caught at right level? Nothing swallowed silently?
6. **Logging** — Key operations logged with context (CaseId, UserId)?
7. **Test Coverage** — Unit/integration tests added?

---

### Q17: Pull Request Best Practices

1. **Small PRs** — Easier to review, faster to merge (< 400 lines diff ideal)
2. **Descriptive titles** — "feat: Add JWT token refresh endpoint" not "Update code"
3. **Link to ticket** — Reference Jira/Azure DevOps ticket
4. **Self-review first** — Review your own PR before requesting review
5. **Respond to comments** — Don't resolve without addressing the feedback

---

## 8. Coding Round Algorithms (C#)

### Q18: Top Algorithm Patterns to Know

**Two Pointers** (String reversal, palindrome, merge sorted arrays):
```csharp
// Reverse string in-place — O(N) time, O(1) space
public static string Reverse(string input)
{
    char[] chars = input.ToCharArray();
    int left = 0, right = chars.Length - 1;
    while (left < right)
    {
        (chars[left], chars[right]) = (chars[right], chars[left]); // Tuple swap
        left++; right--;
    }
    return new string(chars);
}
```

**HashMap Pattern** (Two Sum, first non-repeated char):
```csharp
// Two Sum — O(N) with dictionary
public static int[] TwoSum(int[] nums, int target)
{
    var numMap = new Dictionary<int, int>(); // value → index
    for (int i = 0; i < nums.Length; i++)
    {
        int complement = target - nums[i];
        if (numMap.ContainsKey(complement))
            return new[] { numMap[complement], i };
        numMap[nums[i]] = i;
    }
    throw new ArgumentException("No solution");
}
```

**Sliding Window** (Longest substring, max sum subarray):
```csharp
// Longest substring without repeating chars — O(N)
public static int LengthOfLongestSubstring(string s)
{
    int maxLen = 0, left = 0;
    var charMap = new Dictionary<char, int>();
    for (int right = 0; right < s.Length; right++)
    {
        if (charMap.ContainsKey(s[right]))
            left = Math.Max(left, charMap[s[right]] + 1); // Slide window
        charMap[s[right]] = right;
        maxLen = Math.Max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

**Stack Pattern** (Balanced brackets, expression evaluation):
```csharp
// Balanced brackets — O(N)
public static bool IsBalanced(string input)
{
    var stack = new Stack<char>();
    var map = new Dictionary<char, char> { { ')', '(' }, { ']', '[' }, { '}', '{' } };
    foreach (char c in input)
    {
        if (map.ContainsValue(c)) stack.Push(c);
        else if (map.ContainsKey(c) && (stack.Count == 0 || stack.Pop() != map[c]))
            return false;
    }
    return stack.Count == 0;
}
```

**Floyd's Cycle Detection** (Linked List cycle):
```csharp
public static bool HasCycle(ListNode head)
{
    var slow = head; var fast = head;
    while (fast != null && fast.Next != null)
    {
        slow = slow.Next;       // 1 step
        fast = fast.Next.Next;  // 2 steps
        if (slow == fast) return true; // Cycle detected
    }
    return false;
}
```

**Binary Search** — O(log N):
```csharp
public static int BinarySearch(int[] sorted, int target)
{
    int left = 0, right = sorted.Length - 1;
    while (left <= right)
    {
        int mid = left + (right - left) / 2; // Avoid overflow!
        if (sorted[mid] == target) return mid;
        if (sorted[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

---

## 9. BSK Project Scenarios — STAR Answers

### Q19: Tell me about a challenging technical problem you solved

**STAR Format using BSK context**:

**Situation**: BSK integrates with 7 external vendors (KYC, payment, SMS). When any vendor goes down, API threads would hang for 30 seconds waiting for timeout, causing cascading failures and poor user experience.

**Task**: Implement a proactive vendor health monitoring system that prevents wasted connections and returns immediate, clean error responses when a vendor is unavailable.

**Action**: I designed and implemented the `ThirdPartyHealthMonitorService` as an `IHostedService` that:
1. Probes all vendor health URLs every 60 seconds using `PeriodicTimer`
2. Updates a thread-safe `IServiceHealthRegistry` (`ConcurrentDictionary<string, bool>`)
3. Repositories check `IServiceHealthGuard.EnsureServiceIsUp()` before any vendor call
4. Failures throw `ThirdPartyServiceUnavailableException` caught by custom middleware
5. Returns structured 503 response within milliseconds instead of 30-second timeout

**Result**: Zero thread-blocking timeouts on vendor outages. Dashboard shows real-time vendor status. API responsiveness maintained even during external service degradation.

---

### Q20: How do you handle secrets in production?

> *"In BSK, we follow a layered secrets strategy. In local development, we use `dotnet user-secrets` so credentials never appear in `appsettings.json` or Git history. In staging/production Docker deployments, sensitive values like JWT secrets and DB connection strings are passed as environment variables via the CI/CD pipeline. For enterprise production, we integrate Azure Key Vault using `DefaultAzureCredential` (Managed Identity — no passwords in code) so the server automatically fetches and rotates secrets without any credential exposure."*

---

## 10. Quick-Fire Advanced Checklist

| Question | Short Answer |
|---|---|
| What is gRPC? | High-performance RPC framework using Protocol Buffers (binary). Faster than REST for service-to-service |
| What is SignalR? | Real-time bidirectional communication (WebSockets). Used for live dashboards, notifications |
| What is Kestrel? | Cross-platform HTTP server built into .NET. Default for ASP.NET Core apps |
| What is YARP? | Yet Another Reverse Proxy — Microsoft's .NET-native API Gateway library |
| What is health check endpoint? | `/health/live` (is app running?) + `/health/ready` (is app ready to serve?) — used by Kubernetes |
| What is OpenTelemetry? | Vendor-neutral observability framework — collects traces, metrics, logs across services |
| What is `IAsyncEnumerable<T>`? | Stream results one-by-one asynchronously — used for LLM token streaming, large data paging |
| What is Span<T>? | Stack-allocated view over memory — zero allocation string/byte operations |
| Thread Starvation? | Thread pool exhausted because all threads blocked on synchronous I/O |
| Deadlock? | Thread A waits for Thread B's lock, Thread B waits for Thread A's lock — neither progresses |

---

## 🏁 Final Interview Preparation Checklist

### 2 Weeks Before Interview:
- [ ] Read all 6 section files daily
- [ ] Practice explaining BSK architecture in 2 minutes
- [ ] Practice coding algorithms Q18 in LeetCode

### 1 Week Before:
- [ ] Mock interview with a friend — explain SOLID with BSK examples
- [ ] Revisit SQL queries (joins, CTEs, window functions)
- [ ] Review Dependency Injection lifetimes and Captive Dependencies

### Day Before:
- [ ] Review your BSK project's key features (health monitor, JWT auth, Polly)
- [ ] Prepare 3 STAR stories (challenge, learning, achievement)
- [ ] Good sleep — interview performance drops significantly without rest

### During Interview:
- Always relate answers to **BSK project** for credibility
- If you don't know something → say "I haven't implemented that, but here's how I understand it conceptually..."
- Ask clarifying questions before diving into design questions

---
*See also: [Section_06_Coding_Practice.md](./Section_06_Coding_Practice.md)*
