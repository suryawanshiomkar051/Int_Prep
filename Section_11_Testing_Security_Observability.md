# 🧪 Section 11: Testing, Security & Observability — Interview Preparation Guide
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> Companies increasingly test for **unit testing knowledge** at the 2-3 year level. These questions separate candidates who write real, maintainable code from those who only write features.

---

## 📋 Table of Contents

1. [Unit Testing with xUnit + Moq](#1-unit-testing-with-xunit--moq)
2. [Integration Testing with WebApplicationFactory](#2-integration-testing-with-webapplicationfactory)
3. [Test-Driven Development (TDD) Basics](#3-test-driven-development-tdd-basics)
4. [OAuth 2.0 & OpenID Connect](#4-oauth-20--openid-connect)
5. [OWASP Top 10 — Full Reference](#5-owasp-top-10--full-reference)
6. [OpenTelemetry & Distributed Tracing](#6-opentelemetry--distributed-tracing)
7. [Prometheus & Grafana](#7-prometheus--grafana)
8. [Event-Driven Architecture & RabbitMQ](#8-event-driven-architecture--rabbitmq)
9. [Quick-Fire Checklist](#9-quick-fire-checklist)

---

## 1. Unit Testing with xUnit + Moq

### Q1: What is unit testing? What are the 3 parts of a good test?

**Answer**: A unit test verifies a single, isolated unit of logic (one method) without real external dependencies (no DB, no HTTP calls). Dependencies are replaced with **mocks** or **fakes**.

**AAA Pattern** (Arrange, Act, Assert):
```csharp
[Fact] // xUnit test method attribute
public async Task GetCaseDetails_ShouldReturnValidDto_WhenCaseExists()
{
    // ARRANGE — Setup dependencies and test data
    var mockRepo = new Mock<ICaseAsyncRepository>();
    var mockCache = new Mock<IDistributedCache>();

    var expectedCase = new CaseModel { Id = Guid.NewGuid(), ClaimantName = "Omkar Kulkarni" };

    mockRepo.Setup(r => r.GetCaseById(It.IsAny<string>()))
            .ReturnsAsync(expectedCase);

    mockCache.Setup(c => c.GetAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
             .ReturnsAsync((byte[])null); // Simulate cache miss

    var service = new CaseService(mockRepo.Object, mockCache.Object);

    // ACT — Execute the method under test
    var result = await service.GetCaseDetailsAsync(expectedCase.Id);

    // ASSERT — Verify the expected outcome
    Assert.NotNull(result);
    Assert.Equal("Omkar Kulkarni", result.ClaimantName);

    // Verify mock was called exactly once — detects logic bugs
    mockRepo.Verify(r => r.GetCaseById(It.IsAny<string>()), Times.Once);
}
```

---

### Q2: Mock vs Stub vs Fake — What's the difference?

| Type | Description | Example |
|---|---|---|
| **Stub** | Returns pre-configured data, no behavior verification | `mockRepo.Setup(r => r.GetById(id)).ReturnsAsync(case)` |
| **Mock** | Returns data AND verifies interactions (was method called? How many times?) | `mockRepo.Verify(r => r.GetById(...), Times.Once)` |
| **Fake** | Real but simplified implementation (e.g., in-memory DB) | `InMemoryDatabase` in EF Core tests |
| **Spy** | Like mock but partially uses real implementation | Rare in .NET |

---

### Q3: Common Moq Setups — Quick Reference

```csharp
var mock = new Mock<ICaseAsyncRepository>();

// 1. Setup return value
mock.Setup(r => r.GetCaseById(It.IsAny<string>())).ReturnsAsync(new CaseModel());

// 2. Setup with specific argument match
mock.Setup(r => r.GetCaseById("specific-id-123")).ReturnsAsync(specificCase);

// 3. Setup to throw exception
mock.Setup(r => r.GetCaseById(It.IsAny<string>())).ThrowsAsync(new DbException("DB down"));

// 4. Setup void method
mock.Setup(r => r.UpdateCaseStatus(It.IsAny<string>(), It.IsAny<int>())).Returns(Task.CompletedTask);

// 5. Verify — was it called?
mock.Verify(r => r.GetCaseById("specific-id"), Times.Once);    // Exactly once
mock.Verify(r => r.GetCaseById(It.IsAny<string>()), Times.Never); // Never called
mock.Verify(r => r.GetCaseById(It.IsAny<string>()), Times.AtLeastOnce);

// 6. Capture argument with callback
string capturedId = null;
mock.Setup(r => r.GetCaseById(It.IsAny<string>()))
    .Callback<string>(id => capturedId = id) // Capture the argument
    .ReturnsAsync(new CaseModel());
// After act: Assert.Equal("expected-id", capturedId);
```

---

### Q4: Testing a BSK Controller

```csharp
public class CaseControllerTests
{
    private readonly Mock<ICaseAsyncRepository> _mockRepo;
    private readonly CaseController _controller;

    public CaseControllerTests()
    {
        _mockRepo = new Mock<ICaseAsyncRepository>();
        _controller = new CaseController(_mockRepo.Object);
    }

    [Fact]
    public async Task GetCaseById_Returns200_WhenCaseFound()
    {
        // Arrange
        var caseId = "test-id-123";
        _mockRepo.Setup(r => r.GetCaseById(caseId))
                 .ReturnsAsync(new CaseModel { Id = Guid.NewGuid() });

        // Act
        var result = await _controller.GetCaseById(caseId);

        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result);
        var response = Assert.IsType<BaseResponseStatus>(okResult.Value);
        Assert.Equal("200", response.StatusCode);
    }

    [Fact]
    public async Task GetCaseById_Returns404_WhenCaseNotFound()
    {
        // Arrange
        _mockRepo.Setup(r => r.GetCaseById(It.IsAny<string>())).ReturnsAsync((CaseModel)null);

        // Act
        var result = await _controller.GetCaseById("nonexistent-id");

        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result);
        var response = Assert.IsType<BaseResponseStatus>(okResult.Value);
        Assert.Equal("404", response.StatusCode);
    }

    [Theory] // Data-driven test — runs once per InlineData
    [InlineData("")]
    [InlineData(null)]
    [InlineData("  ")]
    public async Task GetCaseById_ReturnsBadRequest_WhenIdIsInvalid(string invalidId)
    {
        // Act
        var result = await _controller.GetCaseById(invalidId);

        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result);
        var response = Assert.IsType<BaseResponseStatus>(okResult.Value);
        Assert.Equal("400", response.StatusCode);
    }
}
```

---

### Q5: Testing asynchronous code with xUnit

```csharp
// xUnit supports async Task test methods natively
[Fact]
public async Task SavePaymentAsync_ShouldCommitTransaction_WhenSuccessful()
{
    // Arrange
    var mockContext = new Mock<ApplicationDBContext>();
    var mockDbSet = new Mock<DbSet<Payment>>();
    var mockTransaction = new Mock<IDbContextTransaction>();

    mockContext.Setup(c => c.Database.BeginTransactionAsync(It.IsAny<CancellationToken>()))
               .ReturnsAsync(mockTransaction.Object);
    mockContext.Setup(c => c.Payments).Returns(mockDbSet.Object);

    var service = new PaymentService(mockContext.Object);

    // Act
    var result = await service.SaveTransactionAsync(new PaymentModel { Amount = 5000 });

    // Assert
    Assert.True(result);
    mockTransaction.Verify(t => t.CommitAsync(It.IsAny<CancellationToken>()), Times.Once);
    mockTransaction.Verify(t => t.RollbackAsync(It.IsAny<CancellationToken>()), Times.Never);
}
```

---

### Q6: FluentAssertions — More readable assertions

```csharp
using FluentAssertions;

// Instead of:
Assert.NotNull(result);
Assert.Equal("Omkar", result.ClaimantName);
Assert.True(result.ClaimAmount > 0);

// FluentAssertions (more readable, better failure messages):
result.Should().NotBeNull();
result.ClaimantName.Should().Be("Omkar");
result.ClaimAmount.Should().BePositive();
result.CreatedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));

// Collections
var cases = await service.GetActiveCasesAsync();
cases.Should().NotBeEmpty()
     .And.HaveCount(3)
     .And.AllSatisfy(c => c.IsActive.Should().BeTrue());
```

---

## 2. Integration Testing with WebApplicationFactory

### Q7: Integration Testing vs Unit Testing

| Feature | Unit Testing | Integration Testing |
|---|---|---|
| **Scope** | Single class/method | Entire HTTP stack |
| **Dependencies** | All mocked | Real DI, middleware, routing |
| **Speed** | Very fast (ms) | Slower (seconds) |
| **Database** | No real DB | SQLite in-memory or TestContainers |
| **Purpose** | Logic correctness | End-to-end flow correctness |

---

### Q8: WebApplicationFactory Test Setup

```csharp
// NuGet: Microsoft.AspNetCore.Mvc.Testing

public class CustomWebAppFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove real DB registration
            var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(DbContextOptions<ApplicationDBContext>));
            if (descriptor != null) services.Remove(descriptor);

            // Replace with in-memory SQLite (fast, no external deps)
            services.AddDbContext<ApplicationDBContext>(options =>
                options.UseSqlite("DataSource=:memory:"));

            // Build provider and ensure schema created
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ApplicationDBContext>();
            db.Database.EnsureCreated();
        });
    }
}

public class CaseApiIntegrationTests : IClassFixture<CustomWebAppFactory>
{
    private readonly HttpClient _client;

    public CaseApiIntegrationTests(CustomWebAppFactory factory)
    {
        _client = factory.CreateClient();
        // Add JWT token for authenticated endpoints
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", TestJwtHelper.GenerateValidToken());
    }

    [Fact]
    public async Task GetCaseById_Returns404_ForNonExistentCase()
    {
        // Act — real HTTP call to local test server
        var response = await _client.GetAsync("bsk/api/Case/GetCaseById?Id=nonexistent");

        // Assert — real HTTP response
        var content = await response.Content.ReadFromJsonAsync<BaseResponseStatus>();
        Assert.Equal("404", content.StatusCode);
    }
}
```

---

## 3. Test-Driven Development (TDD) Basics

### Q9: What is TDD? What is the Red-Green-Refactor cycle?

**TDD (Test-Driven Development)**: Write the **test first**, then write the minimum code to make it pass, then refactor.

```
RED    → Write a failing test (test doesn't even compile — method doesn't exist yet)
GREEN  → Write the minimum code to make the test pass
REFACTOR → Improve code quality without breaking the test
```

**Benefits**:
- Forces you to think about the API before implementation
- 100% test coverage by definition
- Tests act as executable documentation

**Realistic Use**: Full strict TDD is rare in enterprise. More common: write tests **alongside** or **just after** writing a feature, especially for complex business logic.

---

### Q10: What to test vs What NOT to test

**Test**:
- ✅ Business logic and conditional branches
- ✅ Edge cases (null input, empty list, boundary values)
- ✅ Error handling (exception thrown when expected)
- ✅ Data transformation logic (DTO mapping, calculations)

**Don't test**:
- ❌ Framework code (EF Core itself, ASP.NET Core routing)
- ❌ Simple property getters/setters
- ❌ Third-party library internals
- ❌ Configuration binding (appsettings.json reading)

---

## 4. OAuth 2.0 & OpenID Connect

### Q11: OAuth 2.0 vs OpenID Connect (OIDC) — Key difference

```
OAuth 2.0  → AUTHORIZATION framework (what can you access?)
             Grants "access tokens" to call API resources
             Answers: Can this app read my calendar?

OIDC       → AUTHENTICATION layer built on top of OAuth 2.0
             Adds "ID token" with user identity claims
             Answers: WHO is this user?
```

**BSK uses JWT** — which combines both: JWT contains identity claims (user ID, role) + is used to authorize API access.

---

### Q12: Authorization Code Flow with PKCE (for SPA/Mobile apps)

```
1. App generates: CodeVerifier (random) → SHA256 → CodeChallenge

2. Redirect user to Identity Provider:
   /authorize?response_type=code&client_id=...&code_challenge=...&code_challenge_method=S256

3. User authenticates at Identity Provider

4. IdP redirects back with: ?code=TEMP_AUTH_CODE

5. App exchanges: POST /token
   { code: TEMP_AUTH_CODE, code_verifier: ORIGINAL_RANDOM_STRING }
   ↑ IdP re-hashes verifier and compares with challenge — prevents interception!

6. IdP returns: access_token + id_token + refresh_token
```

**Why PKCE?** Public clients (SPAs, mobile apps) can't safely store a `client_secret`. PKCE proves the token request comes from the same app that initiated the auth flow, even without a secret.

---

## 5. OWASP Top 10 — Full Reference

### Q13: OWASP Top 10 with ASP.NET Core protections

| # | Vulnerability | ASP.NET Core Protection |
|---|---|---|
| A01 | **Broken Access Control** | `[Authorize]` + Role policies + resource-based auth |
| A02 | **Cryptographic Failures** | AES-256 for data at rest, HTTPS enforced (`UseHttpsRedirection`, `UseHsts`) |
| A03 | **Injection (SQL)** | EF Core parameterized queries; never string-concat user input |
| A04 | **Insecure Design** | Threat modeling, principle of least privilege, fail-secure defaults |
| A05 | **Security Misconfiguration** | No default credentials, dev exception page disabled in prod, CORS restricted |
| A06 | **Vulnerable Components** | `dotnet list package --vulnerable`, Dependabot alerts on GitHub |
| A07 | **Auth Failures** | JWT signature validation, short expiry, `ClockSkew = TimeSpan.Zero` |
| A08 | **Software/Data Integrity** | NuGet package signing, no `http://` CDN links |
| A09 | **Logging/Monitoring Failures** | Serilog structured logs, alert on 5xx rate spikes |
| A10 | **SSRF** | Validate/whitelist outgoing URLs, don't make server-side HTTP calls based on user-supplied URLs |

---

### Q14: Secrets Management in Production (BSK Pattern)

```
NEVER:   Hardcode secrets in appsettings.json (committed to Git!)
DEV:     dotnet user-secrets (local only, not in project files)
STAGING: Environment variables in Docker Compose / CI pipeline
PROD:    Azure Key Vault + DefaultAzureCredential (Managed Identity)
```

```csharp
// Azure Key Vault with Managed Identity (no credentials in code!)
builder.Configuration.AddAzureKeyVault(
    new Uri("https://bsk-prod-vault.vault.azure.net/"),
    new DefaultAzureCredential()); // Uses Managed Identity — no keys in code!

// Then read normally:
var jwtSecret = builder.Configuration["JwtSecret"]; // Fetched from Key Vault
```

---

## 6. OpenTelemetry & Distributed Tracing

### Q15: What is OpenTelemetry? Why use it?

**The three pillars of observability**:
- **Logs** — What happened (Serilog, Console)
- **Metrics** — How often / How fast (Prometheus, `dotnet-counters`)
- **Traces** — Where time was spent across services (Jaeger, Zipkin, AWS X-Ray)

**OpenTelemetry (OTel)** = vendor-neutral standard to collect all three. Export to any backend.

```csharp
// Program.cs — OpenTelemetry setup for BSK
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("BSK_API")
        .AddAspNetCoreInstrumentation()  // Traces incoming HTTP requests
        .AddHttpClientInstrumentation()  // Traces outgoing vendor API calls (Surepass, Razorpay)
        .AddSqlClientInstrumentation()   // Traces SQL query executions
        .AddOtlpExporter(opt =>
            opt.Endpoint = new Uri("http://jaeger-collector:4317")));

// Custom activity (manual span)
private static readonly ActivitySource _activitySource = new("BSK_API");

public async Task<CaseModel> ProcessKycVerificationAsync(string panNumber)
{
    using var activity = _activitySource.StartActivity("KycVerification.Pan");
    activity?.SetTag("pan.last4", panNumber[^4..]); // Tag for searchability
    // ... business logic
}
```

**Distributed Trace ID**: When a request enters BSK API, a `TraceId` is generated and propagated in HTTP headers (`traceparent`) to all downstream services. You can see the entire journey (BSK API → Accounting Service → SMS Service) in one trace in Jaeger.

---

## 7. Prometheus & Grafana

### Q16: Setting up metrics in ASP.NET Core

```csharp
// NuGet: prometheus-net.AspNetCore
app.UseMetricServer();    // Exposes /metrics endpoint for Prometheus to scrape
app.UseHttpMetrics();     // Auto-captures: request count, duration, status codes

// Custom business metric (e.g., track claim registrations)
private static readonly Counter ClaimsRegistered = Metrics
    .CreateCounter("bsk_claims_registered_total", "Total number of claims registered",
                   new CounterConfiguration { LabelNames = new[] { "policy_type" } });

// In the claim registration method:
ClaimsRegistered.WithLabels(claim.PolicyType).Inc();
```

**Grafana Dashboard queries**:
- Request rate: `rate(http_requests_received_total[5m])`
- Error rate: `rate(http_requests_received_total{code=~"5.."}[5m])`
- P95 latency: `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`
- Thread pool: `dotnet_threadpool_active_threads_count`

---

## 8. Event-Driven Architecture & RabbitMQ

### Q17: Event-Driven Architecture — Why and When?

**Synchronous (HTTP) problems for notifications**:
```
BSK API → POST /send-email (SendGrid)
         → If SendGrid is down: REQUEST FAILS! Claim data still saved, email never sent.
         → Thread blocks during email sending: slower API response
```

**Async Event-Driven solution**:
```
BSK API → Save Claim to DB + Publish ClaimRegisteredEvent to RabbitMQ → Return 200 immediately
                                         ↓ (decoupled)
NotificationService subscribes → SendGrid call here → Retry if fails (no impact on claim API)
```

---

### Q18: MassTransit + RabbitMQ in .NET

```csharp
// 1. Define Event Contract
public record ClaimRegisteredEvent(Guid ClaimId, string ClaimantName, string PolicyType);

// 2. Register in Program.cs
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<ClaimRegisteredEventConsumer>(); // Register consumer

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq://localhost", h =>
        {
            h.Username("bsk_user");
            h.Password("bsk_pass");
        });
        cfg.ConfigureEndpoints(ctx);
    });
});

// 3. Publish (in Case Repository after saving)
public class CaseAsyncRepository : ICaseAsyncRepository
{
    private readonly IPublishEndpoint _publisher;

    public async Task<bool> RegisterCaseAsync(CaseModel model)
    {
        _context.Cases.Add(model);
        await _context.SaveChangesAsync();

        // Publish event after successful save
        await _publisher.Publish(new ClaimRegisteredEvent(model.Id, model.ClaimantName, model.PolicyType));
        return true;
    }
}

// 4. Consume (in Notification Service)
public class ClaimRegisteredEventConsumer : IConsumer<ClaimRegisteredEvent>
{
    private readonly IEmailService _emailService;

    public async Task Consume(ConsumeContext<ClaimRegisteredEvent> context)
    {
        var ev = context.Message;
        await _emailService.SendClaimConfirmationAsync(ev.ClaimantName, ev.ClaimId);
    }
}
```

---

### Q19: Outbox Pattern — Guarantee Event Delivery

**Problem**: Between `SaveChangesAsync()` and `_publisher.Publish()`, the app could crash:
- Claim saved ✅, but event NOT published ❌ → Email never sent, silent inconsistency

**Solution — Transactional Outbox**:
```csharp
// Save claim + outbox message in ONE atomic transaction
using var transaction = await _context.Database.BeginTransactionAsync();

_context.Cases.Add(newCase);
_context.OutboxMessages.Add(new OutboxMessage
{
    MessageType = nameof(ClaimRegisteredEvent),
    Payload = JsonSerializer.Serialize(new ClaimRegisteredEvent(newCase.Id, newCase.ClaimantName)),
    CreatedAt = DateTime.UtcNow,
    IsProcessed = false
});

await _context.SaveChangesAsync();
await transaction.CommitAsync(); // Both saved atomically

// Background worker (separate HostedService) polls outbox:
// Reads unprocessed → publishes to RabbitMQ → marks processed
// Even if the app crashed after commit, the worker picks it up on next run
```

---

## 9. Quick-Fire Checklist

| Question | Short Answer |
|---|---|
| xUnit `[Fact]` vs `[Theory]` | `[Fact]` = single test; `[Theory]` = data-driven (with `[InlineData]`) |
| `It.IsAny<T>()` in Moq | Matches any argument of type T |
| `Mock<T>.Object` | Gets the actual mocked object to inject |
| `Verify(Times.Once)` | Asserts method was called exactly once |
| What is AAA? | Arrange (setup) → Act (execute) → Assert (verify) |
| What is test isolation? | Each test should be independent — no shared mutable state |
| FluentAssertions benefit? | More readable assertions with better failure messages |
| What is a test fixture? | Shared setup code for a group of tests (implements `IClassFixture`) |
| `IClassFixture` vs `ICollectionFixture` | Class: shared within one test class; Collection: shared across multiple |
| OAuth 2.0 vs OIDC | OAuth = authorization (what); OIDC = authentication (who) |
| PKCE purpose | Prevent auth code interception in public clients (SPA, mobile) |
| OpenTelemetry pillars | Logs + Metrics + Traces |
| TraceId purpose | Correlate logs across multiple microservices for one request |
| Prometheus pull vs push | Prometheus PULLS metrics from `/metrics` endpoint |
| RabbitMQ exchange types | Direct (routing key), Fanout (broadcast), Topic (wildcard), Headers |
| At-most-once vs At-least-once | At-most-once: may drop messages; At-least-once: may duplicate (MassTransit default) |
| Idempotency key | Unique ID per operation — prevents duplicate processing of same message |

---

## 🏁 BSK Talking Points — Testing Section

> *"In BSK, we use xUnit with Moq for unit testing. Our controller tests mock the repository interfaces, so tests run without any database. We verify both the response data and that the repository was called the expected number of times using `mock.Verify()`. For integration tests, we use `WebApplicationFactory` with SQLite in-memory to test the full HTTP pipeline — routing, DI, middleware, and response format — without needing a real SQL Server instance. We also added `FluentAssertions` to make test failure messages more readable."*

---
*Next: See [Section_12_Master_Index.md](./Section_12_Master_Index.md)*
