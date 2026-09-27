# BSK Enterprise System — Advanced System Design, DevOps & Security Study Guide

Welcome to the **BSK Advanced System Design, DevOps & Security Study Guide**. This document contains production-grade, highly technical Q&As addressing modern enterprise architecture, cloud-native deployments, advanced testing pipelines, data security, and application observability.

---

## 📋 Table of Contents

1. [📐 System Design & Architecture (Questions 1 – 3)](#-system-design--architecture)
2. [🐳 Cloud Native, Docker & DevOps (Questions 4 – 6)](#-cloud-native-docker--devops)
3. [⚡ High-Performance Caching (Questions 7 – 8)](#-high-performance-caching)
4. [🧪 Advanced Testing Pipelines (Questions 9 – 10)](#-advanced-testing-pipelines)
5. [🔒 Advanced Security & Encryption (Questions 11 – 13)](#-advanced-security--encryption)
6. [📊 APM, Observability & OpenTelemetry (Questions 14 – 15)](#-apm-observability--opentelemetry)

---

## 📐 System Design & Architecture

### Q1: What is Domain-Driven Design (DDD)? Entities vs. Value Objects
**Question**: Explain Domain-Driven Design (DDD). What is the difference between an Entity and a Value Object? Provide C# examples.
**Answer**:
**Domain-Driven Design (DDD)** is an architectural approach to software development for complex systems. It focuses on aligning the software implementation with a changing business model (the "domain").

Key Tactical DDD Patterns:
1.  **Entity**:
    *   *Definition*: An object defined by its **unique identity** rather than its attributes. Two entities with different attributes are still the same object if their ID matches.
    *   *Lifecycle*: Mutable. Its properties change over time, but its identity remains constant.
2.  **Value Object**:
    *   *Definition*: An object that has **no conceptual identity**. It is defined entirely by its attributes. Two value objects are identical if all their properties are equal.
    *   *Lifecycle*: Immutable. If you need to change a value object, you replace it entirely.

**C# Implementation**:
```csharp
// 1. ENTITY (Defined by ID)
public class Claimant : Entity
{
    public Guid Id { get; private set; } // Identity
    public string Name { get; set; }     // Mutable attribute

    public Claimant(Guid id, string name)
    {
        Id = id;
        Name = name;
    }
}

// 2. VALUE OBJECT (Immutable, compared by structural values)
public class Address
{
    public string Street { get; }
    public string City { get; }
    public string ZipCode { get; }

    public Address(string street, string city, string zipCode)
    {
        Street = street;
        City = city;
        ZipCode = zipCode;
    }

    // Value Objects override Equality operators to compare values, not references
    public override bool Equals(object obj)
    {
        return obj is Address other &&
               Street == other.Street &&
               City == other.City &&
               ZipCode == other.ZipCode;
    }
    
    public override int GetHashCode() => HashCode.Combine(Street, City, ZipCode);
}
```

---

### Q2: Event-Driven Architecture (EDA) using RabbitMQ
**Question**: What is Event-Driven Architecture (EDA)? How do you register and publish events to RabbitMQ in .NET?
**Answer**:
**Event-Driven Architecture (EDA)** is a system design pattern where microservices communicate asynchronously by publishing and subscribing to **Events** via a centralized Message Broker (e.g., RabbitMQ, Apache Kafka) instead of making synchronous HTTP calls.

**Core Benefits**:
*   **Asynchronous Decoupling**: Service A does not wait for Service B.
*   **Temporal Availability**: If Service B is temporarily offline, events remain queued in RabbitMQ and will be processed once Service B recovers.

**C# Implementation using MassTransit (Enterprise standard wrapper for RabbitMQ)**:
1.  **Register services in `Program.cs`**:
    ```csharp
    builder.Services.AddMassTransit(x =>
    {
        x.UsingRabbitMq((context, cfg) =>
        {
            cfg.Host("rabbitmq://localhost", h =>
            {
                h.Username("guest");
                h.Password("guest");
            });
        });
    });
    ```
2.  **Define the Event (Interface/Record)**:
    ```csharp
    public record CaseRegisteredEvent(Guid CaseId, string ClaimantEmail, decimal ClaimAmount);
    ```
3.  **Publish the Event from Controller/Service**:
    ```csharp
    public class CaseController : ControllerBase
    {
        private readonly IPublishEndpoint _publishEndpoint;
        public CaseController(IPublishEndpoint publishEndpoint) => _publishEndpoint = publishEndpoint;

        [HttpPost("RegisterCase")]
        public async Task<IActionResult> RegisterCase([FromBody] CaseModel model)
        {
            // 1. Save case to primary SQL database...
            
            // 2. Publish event to RabbitMQ (Non-blocking asynchronous call)
            await _publishEndpoint.Publish(new CaseRegisteredEvent(model.Id, "claimant@email.com", model.ClaimAmount));
            
            return Ok();
        }
    }
    ```

---

### Q3: Transactional Outbox Pattern
**Question**: What is the Transactional Outbox Pattern, and how does it prevent data inconsistency in Event-Driven systems?
**Answer**:
**The Problem**:
In an event-driven system, when a controller registers a case, it must do two things: (1) Save the case to the SQL database, and (2) Publish a `CaseRegisteredEvent` to RabbitMQ.
If the database save succeeds, but the network connection to RabbitMQ drops right before publishing, the system enters an **inconsistent state**: the case is saved, but other services (like billing or messaging) never find out.

**The Solution (Transactional Outbox Pattern)**:
Instead of publishing the event directly to RabbitMQ, the application saves both the **Entity** and the **Event Payload (Outbox Message)** into the same SQL database under a single database transaction. This ensures that either *both* succeed or *both* fail (ACID).

```
[API Controller] ──► [Begin SQL Transaction]
                           │
                           ├──► Save Case record to 'Cases' table
                           ├──► Save Event Payload to 'OutboxMessages' table
                           │
                     [Commit SQL Transaction]
                           │
[Background Worker]  ◄─────┘ (Probes 'OutboxMessages' table periodically)
      │
      ├──► Read unpublished outbox messages
      ├──► Publish messages to RabbitMQ
      └──► Mark messages as 'Processed' in DB
```

**Outbox Worker C# Mockup**:
```csharp
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    public OutboxProcessor(IServiceProvider serviceProvider) => _serviceProvider = serviceProvider;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            using var scope = _serviceProvider.CreateScope();
            var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDBContext>();
            var publisher = scope.ServiceProvider.GetRequiredService<IPublishEndpoint>();

            // 1. Read unpublished events
            var messages = await dbContext.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .Take(20)
                .ToListAsync();

            foreach (var msg in messages)
            {
                // 2. Publish event safely to RabbitMQ
                await publisher.Publish(JsonSerializer.Deserialize<CaseRegisteredEvent>(msg.Payload));
                
                // 3. Mark as processed
                msg.ProcessedAt = DateTime.UtcNow;
            }
            await dbContext.SaveChangesAsync();
        }
    }
}
```

---

## 🐳 Cloud Native, Docker & DevOps

### Q4: Multi-Stage Docker Builds
**Question**: Explain how a multi-stage Docker build works. Provide a production-ready `Dockerfile` for an ASP.NET Core Web API.
**Answer**:
A **Multi-Stage Docker Build** uses multiple `FROM` instructions in a single Dockerfile. It allows developers to compile and build their application inside a heavy SDK container, and then copy *only* the compiled binaries into a lightweight runtime container.

**Why we use it**:
It dramatically reduces the final production container footprint (from ~800MB SDK image down to ~100MB runtime image) and enhances security by removing compiler tools from the production environment.

**Production-ready `Dockerfile` for BSK API**:
```dockerfile
# Stage 1: Build environment
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj files and restore dependencies to utilize Docker layer caching
COPY ["BSKAPI/BSKAPI.csproj", "BSKAPI/"]
RUN dotnet restore "BSKAPI/BSKAPI.csproj"

# Copy the remaining source files and compile
COPY . .
WORKDIR "/src/BSKAPI"
RUN dotnet build "BSKAPI.csproj" -c Release -o /app/build

# Stage 2: Publish the application
FROM build AS publish
RUN dotnet publish "BSKAPI.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Stage 3: Lightweight production container
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS final
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# Copy compiled binaries from Stage 2
COPY --from=publish /app/publish .

# Run container as a secure, non-root user (security best practice)
USER $APP_UID
ENTRYPOINT ["dotnet", "BSKAPI.dll"]
```

---

### Q5: CI/CD Pipelines (GitHub Actions)
**Question**: What is a CI/CD Pipeline? Provide a GitHub Actions workflow configuration (`.yml`) to build and test a .NET application automatically.
**Answer**:
*   **CI (Continuous Integration)**: Automatically builds, compiles, and runs unit/integration tests every time code is pushed to a Git branch, catching compile bugs instantly.
*   **CD (Continuous Deployment)**: Automatically builds the production Docker container and deploys it to servers (e.g., AWS, Azure) once changes are merged into the `main` branch.

**GitHub Actions Configuration (`.github/workflows/ci.yml`)**:
```yaml
name: .NET Core CI Pipeline

on:
  push:
    branches: [ "main", "develop" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Source Code
      uses: actions/checkout@v4

    - name: Setup .NET SDK 8.0
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'

    - name: Restore Dependencies
      run: dotnet restore

    - name: Build Solution
      run: dotnet build --no-restore --configuration Release

    - name: Run Unit Tests
      run: dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage"
```

---

### Q6: Kubernetes (K8s) Deployment
**Question**: What is Kubernetes (K8s)? Provide a standard deployment YAML file for hosting an ASP.NET Core API.
**Answer**:
**Kubernetes (K8s)** is an open-source container orchestration engine that automates the deployment, scaling, and management of containerized applications (Docker containers) across a cluster of servers.

**Core K8s Resources**:
*   **Pod**: The smallest deployable unit in K8s, wrapping one or more Docker containers.
*   **Deployment**: Manages Pod replicas, self-healing (replacing dead containers), and zero-downtime rolling updates.
*   **Service**: Provides a stable internal IP address and load balancer routing across pods.

**Standard Kubernetes Configuration (`deployment.yaml`)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bsk-api-deployment
  labels:
    app: bsk-api
spec:
  replicas: 3 # Keeps 3 identical instances running for high availability
  selector:
    matchLabels:
      app: bsk-api
  template:
    metadata:
      labels:
        app: bsk-api
    spec:
      containers:
      - name: bsk-api-container
        image: dockerregistry.yourcompany.com/bsk-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: bsk-api-service
spec:
  type: LoadBalancer # Exposes the service externally
  selector:
    app: bsk-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---

## ⚡ High-Performance Caching

### Q7: Cache-Aside Pattern using Redis
**Question**: What is the Cache-Aside Pattern? How do you implement Redis Caching in ASP.NET Core?
**Answer**:
**The Cache-Aside Pattern** is the most common distributed caching strategy:
1.  The application receives a request for a data resource.
2.  It checks the **Cache (Redis)**. E.g., *Cache Hit*: Returns data instantly.
3.  *Cache Miss*: Reads data from the slow **Database (SQL Server)**.
4.  The application writes the fetched data to the Cache for future requests and returns it to the client.

**C# Implementation using `StackExchange.Redis`**:
```csharp
public class CaseService
{
    private readonly ICaseAsyncRepository _dbRepo;
    private readonly IDistributedCache _cache; // Abstracted wrapper for Redis

    public CaseService(ICaseAsyncRepository dbRepo, IDistributedCache cache)
    {
        _dbRepo = dbRepo;
        _cache = cache;
    }

    public async Task<CaseDto> GetCaseDetailsAsync(Guid caseId)
    {
        string cacheKey = $"case:{caseId}";

        // 1. Check Redis Cache
        var cachedData = await _cache.GetStringAsync(cacheKey);
        if (!string.IsNullOrEmpty(cachedData))
        {
            // Cache Hit: Deserialize and return
            return JsonSerializer.Deserialize<CaseDto>(cachedData);
        }

        // 2. Cache Miss: Query SQL Database
        var dbData = await _dbRepo.GetCaseById(caseId.ToString());
        if (dbData == null) return null;

        var dto = new CaseDto { Id = dbData.Id, ClaimantName = dbData.ClaimantName };

        // 3. Write data back to Redis with a 15-minute sliding expiry
        var options = new DistributedCacheEntryOptions()
            .SetAbsoluteExpiration(TimeSpan.FromMinutes(15));
            
        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(dto), options);

        return dto;
    }
}
```

---

### Q8: Cache Stampede (Cache Thundering) Mitigation
**Question**: What is a Cache Stampede, and how do you prevent it using distributed locks?
**Answer**:
**Cache Stampede** occurs when a highly popular cache item (e.g., the primary executive dashboard metrics) expires.
If 1,000 concurrent API requests hit the server at that exact millisecond:
1.  All 1,000 requests experience a **Cache Miss** simultaneously.
2.  All 1,000 threads attempt to query the database, causing database thread pool starvation, high CPU spikes, and potential database service outages.

**Mitigation (Distributed Locking)**:
Only the *first* thread to experience the cache miss acquires a lock to query the database and update the cache. All other 999 threads wait, then read the newly populated cache value once the lock is released.

**C# Implementation using `SemaphoreSlim` (Local Lock)**:
```csharp
public class ResilientCacheService
{
    private readonly IDistributedCache _cache;
    private readonly ApplicationDBContext _context;
    private static readonly SemaphoreSlim _lock = new(1, 1); // Thread synchronization lock

    public async Task<string> GetDashboardDataAsync()
    {
        string cacheKey = "dashboard_data";
        
        // 1. Check Cache
        var data = await _cache.GetStringAsync(cacheKey);
        if (data != null) return data;

        // 2. Cache Miss: Attempt to acquire Lock
        await _lock.WaitAsync();
        try
        {
            // Double-check cache inside lock (in case a previous thread already populated it)
            data = await _cache.GetStringAsync(cacheKey);
            if (data != null) return data;

            // 3. Query database (Executed by only ONE thread)
            data = await QueryHeavyMetricsFromDbAsync();

            // 4. Update Cache
            await _cache.SetStringAsync(cacheKey, data, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            });
        }
        finally
        {
            _lock.Release(); // Always release lock in finally block
        }

        return data;
    }
}
```

---

## 🧪 Advanced Testing Pipelines

### Q9: Integration Testing with `WebApplicationFactory`
**Question**: What is an Integration Test? Show how to implement a test using `WebApplicationFactory` in xUnit to test a real API endpoint.
**Answer**:
While Unit Tests isolate a single method using mocked dependencies, **Integration Tests** verify that all application components (Routing, Controllers, Dependency Injection, and Database Drivers) work together seamlessly.

**C# Implementation**:
1.  Install NuGet Package: `Microsoft.AspNetCore.Mvc.Testing`
2.  **Define the Integration Test**:
    ```csharp
    public class CaseApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
    {
        private readonly HttpClient _client;

        public CaseApiIntegrationTests(WebApplicationFactory<Program> factory)
        {
            // Spins up an in-memory test instance of your real API using your Program.cs configurations
            _client = factory.CreateClient();
        }

        [Fact]
        public async Task GetCaseById_ShouldReturnNotFound_WhenCaseDoesNotExist()
        {
            // Act: Make a real HTTP call to the local test server
            var response = await _client.GetAsync("bsk/api/Case/GetCaseById?Id=non_existent_guid");

            // Assert: Verify HTTP Status Code and Response Contract
            Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
            
            var content = await response.Content.ReadAsStringAsync();
            Assert.Contains("Data Not Found", content);
        }
    }
    ```

---

### Q10: Mocking in Unit Tests (using Moq)
**Question**: How do you mock database or external services in unit tests? Provide a complete xUnit/Moq C# example.
**Answer**:
In Unit Testing, you must isolate the business logic of your service by mocking its dependencies. This ensures your tests are fast, reliable, and do not make real network calls or database writes.

**C# Implementation**:
```csharp
public class CaseServiceTests
{
    [Fact]
    public async Task GetCaseDetails_ShouldReturnValidDto_WhenCaseExists()
    {
        // 1. Create a Mock instance of the Repository Interface
        var mockRepo = new Mock<ICaseAsyncRepository>();

        // 2. Setup the Mock to return target data when called with any string
        var mockCase = new CaseModel { Id = Guid.NewGuid(), ClaimantName = "Omkar" };
        mockRepo
            .Setup(r => r.GetCaseById(It.IsAny<string>()))
            .ReturnsAsync(mockCase);

        // 3. Inject the Mocked repository into the real Service
        var service = new CaseService(mockRepo.Object, new Mock<IDistributedCache>().Object);

        // 4. Act
        var result = await service.GetCaseDetailsAsync(Guid.NewGuid());

        // 5. Assert
        Assert.NotNull(result);
        Assert.Equal("Omkar", result.ClaimantName);
        
        // 6. Verify that the repository method was executed exactly once
        mockRepo.Verify(r => r.GetCaseById(It.IsAny<string>()), Times.Once);
    }
}
```

---

## 🔒 Advanced Security & Encryption

### Q11: OAuth 2.0 vs. OpenID Connect (OIDC)
**Question**: What is the difference between OAuth 2.0 and OpenID Connect (OIDC)? Explain the Authorization Code Flow with PKCE.
**Answer**:
*   **OAuth 2.0**: An **Authorization framework** that grants third-party applications limited access to HTTP resources (API endpoints) using **Access Tokens** (typically JWTs) without exposing user passwords. (Focus: *Delegated Access*).
*   **OpenID Connect (OIDC)**: An **Authentication layer** built on top of OAuth 2.0. It introduces the **ID Token** containing user profile information (e.g., email, name) to verify the user's identity. (Focus: *Identity Verification*).

**Authorization Code Flow with PKCE**:
This is the industry-standard flow for securing Public Clients (Single Page Applications like React, Angular, or Mobile iOS/Android apps):
1.  **Authorize Request with PKCE**: The app generates a cryptographically secure random string called **Code Verifier**, hashes it using SHA-256 to create the **Code Challenge**, and redirects the user to the Identity Provider (IdP) sending the challenge.
2.  **User Authentication**: The user logs into the IdP.
3.  **Return Authorization Code**: The IdP redirects the user back to the app with a temporary **Authorization Code**.
4.  **Token Exchange with PKCE**: The app sends the Authorization Code and the plain-text **Code Verifier** to the IdP's token endpoint.
5.  **Validation**: The IdP hashes the Code Verifier and compares it with the Code Challenge sent in step 1. If they match, the IdP returns the Access, ID, and Refresh tokens. This prevents attackers from intercepting and utilizing intercepted authorization codes.

---

### Q12: OWASP Top 10 Protections in ASP.NET Core
**Question**: How does ASP.NET Core natively protect against SQL Injection, CSRF, and XSS?
**Answer**:

1.  **SQL Injection**:
    *   *Risk*: Attackers insert SQL commands into input strings to manipulate the database.
    *   *ASP.NET Core Protection*: **Entity Framework Core uses parameterized queries by default**. Input variables are treated as literal parameter values rather than executable SQL code.
2.  **CSRF (Cross-Site Request Forgery)**:
    *   *Risk*: A malicious website tricks a user's browser into sending state-changing HTTP requests to your secure API using the browser's stored credentials (cookies).
    *   *ASP.NET Core Protection*: Anti-forgery tokens (using `[ValidateAntiForgeryToken]` or native middleware configurations). The server sends a unique encrypted cryptographic token to the client. Any subsequent POST/PUT/DELETE request must return this exact token in the HTTP header for the request to succeed.
3.  **XSS (Cross-Site Scripting)**:
    *   *Risk*: Attackers inject malicious JavaScript scripts into database records which then execute when other users view those records.
    *   *ASP.NET Core Protection*: The **Razor view engine automatically HTML-encodes** all output rendering statements (e.g. `@Model.Name`), converting characters like `<script>` into safe characters (`&lt;script&gt;`). For APIs, standard JSON serialization automatically escapes scripting parameters.

---

### Q13: Data Encryption at Rest (C# Column Encryption)
**Question**: How do you implement secure column-level encryption for sensitive data (e.g., PAN, Aadhar, Bank Details) in your database?
**Answer**:
For highly regulated data, securing the database server is not enough. If an attacker gains access to a database backup, sensitive information must remain encrypted at rest.

**Implementation Strategy**:
We encrypt sensitive data in C# before writing to SQL Server using the **AES-256 (Advanced Encryption Standard)** symmetric encryption algorithm, and decrypt it upon database retrieval.

**C# Encryption Helper**:
```csharp
public static class EncryptionHelper
{
    private static readonly byte[] Key = Encoding.UTF8.GetBytes("32_byte_secret_key_for_aes_256!"); // Must be loaded from secure Key Vault
    private static readonly byte[] Iv = Encoding.UTF8.GetBytes("16_byte_iv_vector");

    public static string Encrypt(string plainText)
    {
        if (string.IsNullOrEmpty(plainText)) return plainText;

        using var aes = Aes.Create();
        aes.Key = Key;
        aes.IV = Iv;

        using var encryptor = aes.CreateEncryptor(aes.Key, aes.IV);
        using var ms = new MemoryStream();
        using (var cs = new CryptoStream(ms, encryptor, CryptoStreamMode.Write))
        using (var sw = new StreamWriter(cs))
        {
            sw.Write(plainText);
        }

        return Convert.ToBase64String(ms.ToArray());
    }

    public static string Decrypt(string cipherText)
    {
        if (string.IsNullOrEmpty(cipherText)) return cipherText;

        using var aes = Aes.Create();
        aes.Key = Key;
        aes.IV = Iv;

        using var decryptor = aes.CreateDecryptor(aes.Key, aes.IV);
        using var ms = new MemoryStream(Convert.FromBase64String(cipherText));
        using var cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Read);
        using var sr = new StreamReader(cs);

        return sr.ReadToEnd();
    }
}
```

**Auto-encryption in EF Core (`ApplicationDBContext.cs`)**:
Using Value Converters in EF Core, we can automatically encrypt and decrypt columns without modifying our controller logic:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    var encryptConverter = new ValueConverter<string, string>(
        v => EncryptionHelper.Encrypt(v), // Encrypt on write to database
        v => EncryptionHelper.Decrypt(v)  // Decrypt on read from database
    );

    modelBuilder.Entity<Person>()
        .Property(p => p.PanNo)
        .HasConversion(encryptConverter);
}
```

---

## 📊 APM, Observability & OpenTelemetry

### Q14: OpenTelemetry & Distributed Tracing
**Question**: What is OpenTelemetry? How do you configure and trace requests across different microservices in .NET?
**Answer**:
**OpenTelemetry (OTel)** is an vendor-neutral observability framework used to collect **Metrics, Logs, and Traces** from your applications.

*   **Distributed Tracing**: Traces the entire lifecycle of a request as it hops across multiple microservices.
*   **Trace ID**: A unique identifier generated when a request hits the system edge (e.g., API Gateway). This ID is passed in the HTTP request headers (using W3C Trace Context standard `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`) to all downstream microservices, letting you correlate logs across systems in tools like Jaeger, Zipkin, or AWS X-Ray.

**C# Registration in `Program.cs`**:
```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("BSK_API_Source")
        .AddAspNetCoreInstrumentation() // Automatically traces incoming controller requests
        .AddHttpClientInstrumentation() // Automatically traces outgoing HTTP calls (propagates Trace ID)
        .AddSqlClientInstrumentation()  // Automatically records SQL query executions
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://jaeger-collector:4317");
        }));
```

---

### Q15: APM Systems (Prometheus & Grafana)
**Question**: How do Prometheus and Grafana collect and visualize .NET runtime metrics?
**Answer**:
*   **Prometheus**: A time-series database and monitoring tool that uses a **Pull model** to scrape real-time metrics from your API over HTTP at configured intervals (e.g., every 15 seconds).
*   **Grafana**: A visualization dashboard tool that connects to Prometheus to render real-time graphs, charts, and trigger system health alerts.

**.NET Integration (Prometheus Exporter)**:
1.  Install NuGet Package: `prometheus-net.AspNetCore`
2.  **Enable Metric Exporting in `Program.cs`**:
    ```csharp
    // Exposes a secure '/metrics' endpoint that Prometheus scrapes
    app.UseMetricServer(); 
    app.UseHttpMetrics(); // Automatically captures request counts, durations, and status codes
    ```
3.  **Visualized Runtime Metrics**:
    Once configured, Prometheus scrapes standard metrics such as:
    *   `dotnet_gc_collections_count`: Garbage collection triggers by generation.
    *   `dotnet_threadpool_active_threads_count`: Current ThreadPool thread count (monitors Thread Starvation).
    *   `http_requests_received_total`: Request volumes categorized by endpoint and HTTP status codes.

---

### Q16: Microservices Distributed Transactions — Saga Pattern
**Question**: How do you manage transactional integrity across multiple independent microservices without distributed locks? Explain the Saga Pattern.
**Answer**:
In a microservices architecture, standard database transactions (`ACID`) are impossible because each microservice owns its private database. If a claimant initiates a claim registration:
1.  `Case Service` saves the case.
2.  `Accounting Service` issues a voucher ledger.
3.  `Notification Service` sends an email.
If the `Notification Service` crashes or payment validation fails, you cannot automatically rollback the previous steps since the transactions are already committed to their respective databases.

**The Solution (Saga Pattern)**:
A Saga is a sequence of local transactions. If a step fails, the Saga executes **Compensating Transactions** (undo actions) in reverse order to restore the system to a consistent state (Eventual Consistency).

**Saga Types**:
1.  **Choreography** (Decentralized): Each microservice publishes an event, and the next service listens and reacts. Simple to set up but difficult to track as the workflow grows.
2.  **Orchestration** (Centralized): A dedicated service (Saga Orchestrator) acts as the coordinator, explicitly telling each service what to do and when. Excellent for complex enterprise workflows.

```
[Choreography Saga Flow]:
Case Service ──► (CaseCreatedEvent) ──► Accounting Service ──► (VoucherIssuedEvent) ──► Notification Service
                                              │ (Payment Fails!)
                                              └──► (CancelCaseEvent) ──► Case Service (Compensating Rollback)
```

---

### Q17: API Gateway Pattern & BFF (Backend for Frontend)
**Question**: What is an API Gateway? What are the benefits of using YARP or Ocelot, and what is the BFF Pattern?
**Answer**:
An **API Gateway** acts as a single, centralized entry point for all incoming client requests (Web, Mobile) before routing them to the correct downstream microservices.

**Core Capabilities**:
*   **Routing & Load Balancing**: Redirects incoming calls dynamically.
*   **Authentication & Authorization**: Validates JWT tokens once at the gateway, avoiding repeating auth logic in every microservice.
*   **Rate Limiting & IP Whitelisting**: Protects internal services from DDoS.
*   **Protocol Translation**: E.g., translating client HTTPS requests to high-speed internal gRPC calls.

**YARP (Yet Another Reverse Proxy) in .NET**:
YARP is a highly performant reverse proxy library built by Microsoft using standard .NET middleware components.

**BFF (Backend for Frontend) Pattern**:
Instead of having a single API Gateway for all clients, you build **dedicated gateways for each user interface**:
*   *BFF Mobile*: Optimizes payloads, strips heavy response structures, and aggregates data to reduce mobile bandwidth.
*   *BFF Web*: Passes secure cookies, maps heavier JSON structures, and supports higher bandwidth queries.

---

### Q18: Zero-Downtime Deployments — Blue-Green vs. Canary
**Question**: What is the difference between Blue-Green and Canary deployment strategies? How do they ensure zero-downtime rollouts?
**Answer**:

```
[Blue-Green Deployment]:
Client Traffic ──► [Router/Load Balancer]
                         │
                         ├──► [Environment BLUE (Active v1.0)]
                         └──► [Environment GREEN (Idle v2.0)] ◄── Test new version here
(Switch Router when green is verified)

[Canary Deployment]:
Client Traffic ──► [Router] ──┬──► 95% Traffic ──► [Active v1.0]
                            └──►  5% Traffic ──► [Canary v2.0] (Test with small user base)
```

**Key Differences**:

| Feature | Blue-Green Deployment | Canary Deployment |
|---|---|---|
| **Strategy** | Runs two identical physical environments. | Gradually shifts traffic to the new version in a single cluster. |
| **Traffic Split** | 100% switch at once. | Incremental (e.g., 5% ➔ 20% ➔ 50% ➔ 100%). |
| **Resource Cost** | High (Requires doubling your server count/infrastructure). | Low (Uses existing cluster capacity, spinning up few pods). |
| **Risk Management** | Instant rollback (simply flip the router back to Blue). | Slow blast radius (if v2.0 has bugs, only 5% of users are impacted). |

---

### Q19: Rates & Concurrency Limiting (.NET 8.0 Middleware)
**Question**: How do you configure and enforce Rate Limiting in an ASP.NET Core API? Explain Token Bucket vs. Fixed Window algorithms.
**Answer**:
ASP.NET Core (from .NET 7) includes native rate-limiting middleware to protect endpoints from brute-force attacks and resource exhaustion.

**Algorithms**:
1.  **Fixed Window**: Divides time into fixed segments (e.g., 1 minute). Allows a set number of requests (e.g., 100) per window. If the limit is reached, requests are blocked until the next window resets.
2.  **Token Bucket**: A virtual bucket holds a fixed number of tokens. Each request consumes a token. Tokens are added back to the bucket at a constant rate. This allows for sudden bursts of traffic while maintaining a steady long-term limit.

**C# Configuration in `Program.cs`**:
```csharp
builder.Services.AddRateLimiter(options =>
{
    // Configure a Fixed Window policy
    options.AddFixedWindowLimiter("StrictPolicy", opt =>
    {
        opt.PermitLimit = 10; // Max 10 requests
        opt.Window = TimeSpan.FromSeconds(30); // per 30 seconds
        opt.QueueLimit = 2; // Queue up to 2 excess requests before rejecting
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
    
    // Set 429 Too Many Requests response code
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

// Register middleware early in pipeline
app.UseRateLimiter();

// Apply to specific controller or action
[HttpGet("GetCaseById")]
[EnableRateLimiting("StrictPolicy")]
public async Task<IActionResult> GetCaseById(string Id) { ... }
```

---

### Q20: API Health Probes (Live vs. Ready) in .NET and Kubernetes
**Question**: What are API Health Checks? Explain the difference between Liveness and Readiness probes in Kubernetes and how to configure them in .NET.
**Answer**:
Health Checks allow external cluster managers (like Kubernetes) to query your application's status to ensure it is healthy and ready to receive user traffic.

**Probes**:
1.  **Liveness Probe (`/health/live`)**:
    *   *Purpose*: Verifies if the API process is running. If this endpoint fails (e.g., app is deadlocked), Kubernetes kills the container and restarts it automatically (Self-healing).
2.  **Readiness Probe (`/health/ready`)**:
    *   *Purpose*: Verifies if the API is ready to accept user requests. It checks active dependencies (can it connect to SQL Server, Redis, and SMS gateways?). If this fails, Kubernetes stops sending network traffic to this pod, redirecting users to other healthy pods.

**C# Implementation in `Program.cs`**:
```csharp
// 1. Register Health Checks and dependencies
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"), name: "SQLServerDB")
    .AddRedis(builder.Configuration.GetConnectionString("RedisConnection"), name: "RedisCache");

// 2. Map endpoints in middleware
// Liveness: basic status check
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // Bypasses dependency checks, returns 200 OK instantly if process is up
});

// Readiness: full dependency check
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse // Returns detailed JSON status
});
```

---

### Q21: Clean Architecture vs. Onion Architecture
**Question**: Explain Clean Architecture / Onion Architecture. What are the dependency rules, and how does it benefit enterprise software?
**Answer**:
Clean Architecture and Onion Architecture are software design paradigms that organize code into concentric layers with a fundamental **Dependency Rule**: **All code dependencies must point inward.** Outer layers can depend on inner layers, but inner layers must never depend on (or know anything about) outer layers.

```
       [ Outer Layers: Infrastructure, UI, Database Drivers ]
                               │
                               ▼
            [ Application Layer: Services, Use Cases ]
                               │
                               ▼
             [ Core Layer: Domain Models, Entities ] (Center)
```

**Layers Breakdown**:
1.  **Core / Domain (Center)**: Contains business entities, specifications, and domain exceptions. It has **zero dependencies** on external libraries, ORMs, or frameworks.
2.  **Application (Use Cases)**: Contains service workflows and repository interfaces (abstractions).
3.  **Infrastructure (Outer)**: Contains the implementation of repository interfaces, database contexts (EF Core), loggers, and external network clients.

**Core Benefits**:
*   **Independent of Database/Frameworks**: If BSK decides to migrate from SQL Server to PostgreSQL, the Core domain and Application service logic remain completely untouched. You only swap the Outer Infrastructure driver layer.
*   **Testability**: Because the core business logic is completely insulated from databases and external APIs, you can unit-test all domain rules without mocking database adapters.

---

### Q22: Identity Management and Single Sign-On (SSO)
**Question**: How does Single Sign-On (SSO) work? How does a .NET backend validate tokens from external Identity Providers (e.g., Auth0, Microsoft Entra ID)?
**Answer**:
**Single Sign-On (SSO)** allows a user to log in once with a single set of credentials and access multiple independent applications within an enterprise ecosystem.

**How the Token Validation Works (SSO Backend Pipeline)**:
When a client sends a JWT to your secure `.NET` API, the API does not query the Identity Provider (IdP) for every request. That would cause massive network latency and overload the IdP. Instead, the API validates the token cryptographically locally:
1.  **Fetch Public Signing Keys (JWKS)**: On startup, the API calls the IdP's metadata endpoint (e.g. `/.well-known/openid-configuration`) to retrieve the public keys (**JSON Web Key Set** - JWKS). The API caches these keys.
2.  **Local Cryptographic Verification**: For every incoming request, the API's JWT Bearer middleware verifies:
    *   **Signature**: The token was signed by the IdP's matching private key.
    *   **Expiration (`exp`)**: The token is not expired.
    *   **Issuer (`iss`)**: The token was issued by the trusted IdP url.
    *   **Audience (`aud`)**: The token was issued specifically for this API client identifier.

**Configuration in `Program.cs`**:
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://identityprovider.yourcompany.com";
        options.Audience = "bsk_api_audience";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });
```

---

### Q23: Memory Leaks in C# & Profiling Tools
**Question**: Can memory leaks occur in a managed language like C#? What are the common causes, and how do you diagnose them in production?
**Answer**:
Yes! Although the .NET runtime has a Garbage Collector, **Memory Leaks occur when objects that are no longer needed remain referenced by active, long-lived root objects** (like Singleton classes or static variables), preventing the GC from collecting them.

**Common Causes in C#**:
1.  **Static References**: Storing items in static lists or dictionaries that grow continuously.
2.  **Unsubscribed Event Handlers**: If a short-lived class subscribes to an event in a long-lived Singleton service but fails to unsubscribe, the Singleton maintains a reference to the short-lived class, leaking it in memory.
3.  **Unmanaged Resources**: Failing to close database connections, file handles, or network sockets (omitting `Dispose()` calls).

**How to Diagnose in Production**:
1.  **`dotnet-dump`**: Captures memory dumps of running processes without stopping them.
    ```bash
    dotnet-dump collect -p <ProcessId>
    ```
2.  **`dotnet-gcdump`**: Lightweight memory dumps specifically for GC heap analysis.
3.  **Visual Studio Memory Profiler**: Open the captured `.dmp` file to analyze the **Retention Tree** to find exactly which root objects are holding onto dead objects in the heap.

---

### Q24: Dynamic Logging and Log Correlation (CorrelationID)
**Question**: What is a Correlation ID? How do you implement request correlation tracing inside a monolithic ASP.NET Core API?
**Answer**:
A **Correlation ID** is a unique GUID generated for every incoming HTTP request. This ID is injected into the logging context and appended to every log entry written during that request's execution thread.

**Why we need it**:
If your API receives thousands of concurrent requests, log lines from different users are interleaved in your log files. If a claim save fails, searching for the error in logs is useless without context. Searching by the **Correlation ID** retrieves *only* the log lines associated with that single execution trace, from the controller down to the database query.

**C# Implementation using Custom Middleware & Serilog**:
```csharp
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    private const string CorrelationHeaderKey = "X-Correlation-ID";

    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        // 1. Check if client sent an existing Correlation ID, otherwise generate a new one
        if (!context.Request.Headers.TryGetValue(CorrelationHeaderKey, out var correlationId))
        {
            correlationId = Guid.NewGuid().ToString();
        }

        // 2. Add to response headers so client knows the Trace ID
        context.Response.Headers[CorrelationHeaderKey] = correlationId;

        // 3. Push to Serilog Log Context (automatically appends to all log entries in this thread)
        using (LogContext.PushProperty("CorrelationId", correlationId.ToString()))
        {
            await _next(context);
        }
    }
}
```

---

### Q25: Zero-Trust Security Architecture in Backend APIs
**Question**: What is Zero-Trust Security? How do you implement the "Never Trust, Always Verify" principle in an API backend?
**Answer**:
**Zero-Trust** is a cybersecurity model based on the core principle: **"Never Trust, Always Verify."** It assumes that attackers are already inside the network perimeter, so traditional perimeter security (like IP firewalls) is insufficient.

**How Zero-Trust is implemented in a .NET API**:
1.  **Least Privilege Access (RBAC & CBAC)**:
    *   Do not just check if a user is authenticated. Check their specific scopes and claims.
    *   *BSK Scenario*: A lawyer can read cases assigned to them, but they cannot read cases belonging to other lawyers. Implement **Resource-Based Authorization** (`IAuthorizationService`) instead of relying solely on generic `[Authorize]` attributes.
2.  **Explicit Verification**:
    *   Never assume a request is safe because it came from the internal network (e.g., an internal service). Every microservice must validate the caller's JWT token explicitly.
3.  **Encrypted Communication**:
    *   Enforce **HTTPS/TLS 1.3** for all internal communication between backend microservices and databases (no plain HTTP inside the network).
4.  **Network Micro-segmentation**:
    *   Configure database servers to accept incoming traffic *only* from the specific API container IP address, blocking all other internal traffic.

---

### 💡 Interview Tip
Mastering these advanced concepts shows you can design robust, highly scalable, and secure enterprise systems. Highlight these architectural topics during technical design discussions!

