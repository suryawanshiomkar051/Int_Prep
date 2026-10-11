# 🏢 Infosys Technical Interview Preparation Kit
## 📅 Interview Date: 17-October-2026 | Role: .NET / C# / SQL Backend Engineer

---

## 📁 Repository Overview

This repository is organized into dedicated modules containing **96+ in-depth technical questions with code examples**, **15 live coding whiteboard challenges**, an **HR/Managerial scenario guide**, and a structured **6-day study plan**:

```
Infosys_Interview/
├── 00_Study_Plan.md                 ← 📅 START HERE: Complete 6-Day Study Plan
├── 01_SQL.md                        ← 📦 SQL (27 Qs: 17 Friend + 10 Bonus with Queries)
├── 02_DotNet_Core.md                ← ⚙️ .NET Core & C# (37 Qs: 27 Friend + 10 Bonus with Code)
├── 03_DotNet_API.md                 ← 🌐 ASP.NET Core Web API (19 Qs: 11 Friend + 8 Bonus)
├── 04_Generic_General.md            ← 🧩 OOP, Design Patterns, Cloud & Testing (13 Qs: 7 Friend + 6 Bonus)
├── 05_Coding_Challenges.md          ← 💻 Top 15 Live C# Screen-Share Coding Challenges
├── 06_HR_And_Scenario_Questions.md  ← 🤝 Self-Intro, Project Architecture & Behavioral Answers
└── README.md                        ← 📌 Master Index & Navigation (This file)
```

---

## 📋 Comprehensive Question Index

### 1. SQL Mastery (`01_SQL.md`) – 27 Questions
| # | Topic | Key Concepts |
|---|-------|--------------|
| Q1 | OLTP vs OLAP | Transactional vs Analytical schemas, 3NF vs Star schema |
| Q2 | SQL vs NoSQL | Relational ACID vs Document/Key-Value scalability |
| Q3 | Normalization | 1NF, 2NF, 3NF, Redundancy reduction, Anomaly prevention |
| Q4 | SQL Performance Tuning | Indexes, Execution plans, Statistics, SARGable queries |
| Q5 | Types of JOINs | INNER, LEFT, RIGHT, FULL OUTER, CROSS, SELF JOIN |
| Q6 | SQL Functions | Aggregate, String, Date, Window functions |
| Q7 | Clustered vs Non-Clustered Index | B-Tree structure, Data order, Limits (1 Clustered, 999 NC) |
| Q8 | Estimated vs Actual Execution Plan | Compile-time estimates vs Runtime actuals & statistics |
| Q9 | How to Read Execution Plan | Table Scan, Index Seek, Key Lookup, Cost percentages |
| Q10 | Subqueries | Correlated vs Non-correlated, IN vs EXISTS |
| Q11 | CTE (Common Table Expression) | Recursive CTE, Hierarchy queries, Temporary named results |
| Q12 | Audit Capturing | Triggers with `inserted`/`deleted`, Temporal Tables |
| Q13 | UNION vs UNION ALL | Duplicate elimination, Sorting overhead, Performance |
| Q14 | View vs Materialized View | Virtual query vs Physically indexed view with cluster |
| Q15 | Delete Duplicate Rows | `ROW_NUMBER() OVER (PARTITION BY ...)` technique |
| Q16 | View Last Inserted Record | `SCOPE_IDENTITY()`, `@@IDENTITY`, `IDENT_CURRENT()` |
| Q17 | USING Clause | Difference from `ON` clause, MySQL/Postgres vs T-SQL |
| **BQ1** | Transactions & ACID Properties | `BEGIN TRAN`, `COMMIT`, `ROLLBACK`, Isolation Levels |
| **BQ2** | DELETE vs TRUNCATE vs DROP | DML vs DDL, Logging, Identity reset, Rollback capability |
| **BQ3** | Stored Procedure vs Stored Function | Return types, SELECT usage, DML allowances |
| **BQ4** | Triggers (AFTER vs INSTEAD OF) | Magic tables (`inserted`/`deleted`), Soft deletes |
| **BQ5** | WHERE vs HAVING | Row filtering vs Aggregate group filtering |
| **BQ6** | Advanced Window Functions | `LEAD`, `LAG`, `FIRST_VALUE`, `LAST_VALUE`, `NTILE` |
| **BQ7** | Deadlocks & Prevention | Detection victim, Table ordering, `NOLOCK`, Short transactions |
| **BQ8** | PIVOT and UNPIVOT | Row-to-column and Column-to-row transformation |
| **BQ9** | Temp Tables vs Table Variables vs CTEs | `#Temp`, `@TableVar`, memory vs tempdb behavior |
| **BQ10** | Composite & Covering Indexes | Leading columns, `INCLUDE` clause, Fragmentation & Rebuild |

---

### 2. .NET Core & C# (`02_DotNet_Core.md`) – 37 Questions
| # | Topic | Key Concepts |
|---|-------|--------------|
| Q1 | Sealed Class | Preventing inheritance, JIT inlining, Virtual method sealing |
| Q2 | Exception Handling | `try`/`catch`/`finally`, `throw` vs `throw ex` callstack preservation |
| Q3 | SOLID Principles | Single Responsibility, Open/Closed, Liskov, Interface Segregation, DIP |
| Q4 | Sync vs Async Programming | `async`/`await`, SynchronizationContext, Thread pool, Avoid `.Result` |
| Q5 | DI: .NET Framework vs .NET Core | Third-party containers (Autofac/Unity) vs Built-in `IServiceCollection` |
| Q6 | Code Changes in DI Migration | Converting static factories to constructor injection |
| Q7 | Multi-Targeting Framework & Core | `.csproj` `<TargetFrameworks>net48;net8.0</TargetFrameworks>`, `#if NETCOREAPP` |
| Q8 | Application Architecture | Clean / Onion / N-Tier Architecture, Separation of Concerns |
| Q9 | Technical Feature Implementation | End-to-end flow: Controller → Service → Repository → DB |
| Q10 | Code for DIP (Dependency Inversion) | High & low modules depending on abstractions with DI |
| Q11 | Performance via Middleware | Response caching, Compression, Short-circuiting pipelines |
| Q12 | Exception Handling (Deep Dive) | Custom exception hierarchies, InnerException handling |
| Q13 | Exception Filters | `IExceptionFilter`, Controller-scoped vs Global error filters |
| Q14 | Is try/catch a good practice? | Cost of exceptions, Guard clauses, Exception-free validation |
| Q15 | Centralized Exception Handling | `UseExceptionHandler`, Custom error handling middleware |
| Q16 | Code Review Process | Pull request workflows, Automated linting, Static analysis |
| Q17 | PR Review Checklist | Null safety, Async best practices, SQL injection, Unit test coverage |
| Q18 | Design Patterns in .NET Core | Repository, Unit of Work, Factory, Strategy, Singleton (`Lazy<T>`) |
| Q19 | .NET Framework vs .NET Core | Cross-platform, Performance, Kestrel, Unified SDK |
| Q20 | Asynchronous Deep Dive | `Task.WhenAll`, `Task.WhenAny`, `ConfigureAwait(false)`, `CancellationToken` |
| Q21 | Generics in C# | Type safety, Avoiding boxing/unboxing, Generic constraints (`where T : class`) |
| Q22 | Microservices Architecture | API Gateway, Event-driven communication, Database-per-service |
| Q23 | Dependency Injection Fundamentals | IoC container, Inversion of Control, Loosely coupled components |
| Q24 | DI Service Lifetimes | Transient (per call), Scoped (per HTTP request), Singleton (app-lifetime) |
| Q25 | Middleware Pipeline | `RequestDelegate`, `Use`, `Run`, `Map`, Execution ordering |
| Q26 | .NET Core Key Advantages | Cross-platform, High throughput, CLI tooling, Built-in DI/logging |
| Q27 | LINQ & Left Joins | Method vs Query syntax, `GroupJoin` + `DefaultIfEmpty()` |
| **BQ1** | Value Types vs Reference Types | Stack vs Heap, Pass-by-value vs Reference, `ref` / `out` / `in` |
| **BQ2** | Garbage Collection (GC) | Generations 0/1/2, `IDisposable`, Finalizers, `using` declarations |
| **BQ3** | Delegates, Func, Action & Predicate | Type-safe function pointers, Lambda expressions, Event hooks |
| **BQ4** | Events and EventHandlers | Publisher-Subscriber pattern, Custom EventArgs, Thread safety |
| **BQ5** | Struct vs Class vs Record | Value semantics, Mutability, Value equality with `record` |
| **BQ6** | IEnumerable vs IQueryable vs IList | In-memory evaluation vs Server-side SQL translation |
| **BQ7** | Extension Methods | Static class with `this` modifier, Fluent API construction |
| **BQ8** | yield return & Custom Iterators | State machine generation, Deferred sequence evaluation |
| **BQ9** | Null Safety (C# 8+) | Nullable references `?`, Null-coalescing `??` and `??=`, Guard clauses |
| **BQ10** | String Handling & StringBuilder | Immutability, Memory allocation in loops, String interpolation |

---

### 3. ASP.NET Core Web API (`03_DotNet_API.md`) – 19 Questions
| # | Topic | Key Concepts |
|---|-------|--------------|
| Q1 | Authentication & Authorization | JWT tokens, Symmetric key validation, Claims & Roles, `[Authorize]` |
| Q2 | Controller Attributes / Decorators | `[ApiController]`, `[Route]`, `[HttpGet]`, `[ProducesResponseType]` |
| Q3 | Action Filters | `IActionFilter`, `OnActionExecuting`, Execution pipeline |
| Q4 | Global Error Handling | `ProblemDetails` standard (RFC 7807), Middleware interceptor |
| Q5 | Request & Response Logging | Reading streams without breaking pipeline, Serilog structured logs |
| Q6 | API Routing | Attribute routing, Route constraints (`{id:int}`), Route tokens |
| Q7 | HTTP Verbs | GET, POST, PUT (Idempotent update), PATCH (Partial), DELETE |
| Q8 | Model Binding & Validation | `[FromBody]`, `[FromQuery]`, `[FromRoute]`, DataAnnotations |
| Q9 | HTTP Response Codes | 200 OK, 201 Created, 204 NoContent, 400 BadReq, 401 Unauth, 403 Forbidden |
| Q10 | API Versioning | Query string, URL path, Header versioning strategies |
| Q11 | Steps to Create Web API | `Program.cs` setup, DI registration, Controller routing, Swagger |
| **BQ1** | CORS Configuration | Allowed origins, methods, headers, Preflight OPTIONS request |
| **BQ2** | Rate Limiting (.NET 7+) | Fixed Window, Sliding Window, Token Bucket, Concurrency Limiter |
| **BQ3** | Swagger / OpenAPI | XML docs, JWT Security definition, Swashbuckle configuration |
| **BQ4** | REST vs SOAP | JSON vs XML, Statelessness vs WS-Security, Lightweight contracts |
| **BQ5** | API Pagination & Filtering | `PageNumber`, `PageSize`, Total count headers, Metadata wrapper |
| **BQ6** | API Idempotency | `Idempotency-Key` header, Cache verification, Preventing double charges |
| **BQ7** | HttpClient Best Practices | `IHttpClientFactory`, Socket exhaustion prevention, Polly Retry & Circuit Breaker |
| **BQ8** | Minimal APIs vs Controllers | Endpoint mapping, Lambda handlers, Startup time, Microservice fit |

---

### 4. Generic & General Topics (`04_Generic_General.md`) – 13 Questions
| # | Topic | Key Concepts |
|---|-------|--------------|
| Q1 | jQuery Basics & API Calls | `$.ajax()`, `$.get()`, `$.post()`, Success and Error callbacks |
| Q2 | Calling Backend API | HTTP methods, Headers, Content-Type, CORS credentials |
| Q3 | Clustered vs Non-Clustered Summary | Fast lookup comparison, Leaf nodes with data vs pointers |
| Q4 | Number of Indexes Per Table | 1 Clustered, up to 999 Non-Clustered per table |
| Q5 | Can Stored Proc Be Called from Function?| No (Side-effect restriction in SQL scalar functions) |
| Q6 | SOLID Principles Quick Reference | Acronym breakdown with 1-line definitions |
| Q7 | Interface vs Abstract Class | Multiple inheritance, Default implementations, IS-A vs CAN-DO |
| **BQ1** | Four Pillars of OOP | Encapsulation, Abstraction, Inheritance, Polymorphism (with C# code) |
| **BQ2** | Overloading vs Overriding vs Hiding | Compile-time vs Runtime dispatch, `virtual`/`override` vs `new` |
| **BQ3** | Cloud Concepts for Interviews | IaaS, PaaS, SaaS, Azure Blob, Service Bus, Key Vault, Auto-scaling |
| **BQ4** | Entity Framework Approaches | Code-First migrations vs Database-First scaffolding, `AsNoTracking` |
| **BQ5** | Unit Testing in .NET | xUnit, Moq, AAA (Arrange-Act-Assert) pattern |
| **BQ6** | Caching in .NET | In-Memory cache, Distributed Redis, Cache-Aside pattern |

---

### 5. Live Coding Challenges (`05_Coding_Challenges.md`) – 15 Problems
- Reverse a String / Words in a Sentence (Two-pointer technique)
- Check Palindrome (String & Integer without string conversion)
- Find 2nd Highest Number in an Array (Single-pass $O(N)$)
- Character & Word Frequency Counting (`Dictionary` + LINQ)
- Remove Duplicates from Array / String (Order-preserving `HashSet`)
- Two Sum Problem ($O(N)$ with HashMap)
- Check Anagrams (Character frequency array)
- Balanced Parentheses & Brackets (`Stack<char>`)
- Fibonacci Series (Iterative vs Recursive)
- Prime Number Check ($O(\sqrt{N})$)
- Swap Numbers Without 3rd Variable (Arithmetic & Tuple syntax)
- Find Missing Number in Array $[1..N]$
- FizzBuzz
- Merge Two Sorted Arrays ($O(M+N)$)
- Top 5 Practical LINQ Interview Snippets

---

### 6. HR & Scenario Questions (`06_HR_And_Scenario_Questions.md`) – 7 Scenarios
- **"Tell Me About Yourself"**: High-impact 90-second pitch tailored for Infosys.
- **"Explain Your Project Architecture"**: Clean multi-layer explanation from UI to DB.
- **Challenging Production Bug**: STAR method story about thread starvation & SQL index tuning.
- **Disagreements with Leads / Peers**: Objective benchmark-driven resolution.
- **Handling Changing Requirements**: Agile sprint scope negotiation.
- **"Why Infosys?"**: Company-specific value proposition.
- **Intelligent Questions to Ask the Interviewer**: Leaving a confident final impression.

---

## ⚡ Quick Start Instructions

1. Start by reviewing the schedule in [00_Study_Plan.md](./00_Study_Plan.md).
2. Begin Day 1 with SQL queries in [01_SQL.md](./01_SQL.md).
3. Open Visual Studio / VS Code or Notepad to practice the live coding problems in [05_Coding_Challenges.md](./05_Coding_Challenges.md).
4. Rehearse your self-introduction from [06_HR_And_Scenario_Questions.md](./06_HR_And_Scenario_Questions.md).

*All files prepared specifically for your Infosys interview on 17-Oct-2026. Best of luck! 🚀*
