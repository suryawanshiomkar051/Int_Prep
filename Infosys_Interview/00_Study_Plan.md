# 📅 Infosys Interview – Complete 6-Day Study Plan
## Interview Date: 17-October-2026 | Preparation Window: 11-Oct to 16-Oct

---

> [!IMPORTANT]
> You have **6 days** left until your Infosys interview on **17-October-2026**.  
> Every day has a dedicated technical focus, code practice, and review targets. Stick to the daily schedule and you will enter the interview fully prepared!

---

## 📁 Preparation Resources Map

| Section File | Focus Area | Questions Covered |
|--------------|------------|-------------------|
| [00_Study_Plan.md](./00_Study_Plan.md) | Day-by-Day Roadmap & High-Yield Strategy | Roadmap & Checklist |
| [01_SQL.md](./01_SQL.md) | SQL Queries, Indexes, Execution Plans, Triggers, CTEs | **27 Questions** (17 Friend + 10 Bonus) |
| [02_DotNet_Core.md](./02_DotNet_Core.md) | C#, .NET Core, DI, Async, Middleware, GC, Memory | **37 Questions** (27 Friend + 10 Bonus) |
| [03_DotNet_API.md](./03_DotNet_API.md) | Web API, JWT, Filters, CORS, Minimal APIs, Polly | **19 Questions** (11 Friend + 8 Bonus) |
| [04_Generic_General.md](./04_Generic_General.md) | OOP, Design Patterns, Cloud/Azure, EF Core, Unit Testing | **13 Questions** (7 Friend + 6 Bonus) |
| [05_Coding_Challenges.md](./05_Coding_Challenges.md) | Top 15 Live C# Screen-Share / Whiteboard Problems | **15 Coding Problems** |
| [06_HR_And_Scenario_Questions.md](./06_HR_And_Scenario_Questions.md) | HR, Self-Intro, Architecture, Production Bug Story | **7 Core Scenarios** |

**Grand Total: 96 Technical Theory & Code Questions + 15 Live Coding Challenges + Complete HR Pack!**

---

## 🗓️ Day-by-Day Study Schedule

---

### 📌 Day 1 – 11 Oct (Today) → SQL Mastery
**Goal:** Master all SQL queries, indexes, and execution concepts.

| Time | Topic | Resource |
|------|-------|----------|
| Morning (2 hrs) | OLTP vs OLAP, Normalization (1NF–3NF), JOINs, UNION vs UNION ALL | [01_SQL.md](./01_SQL.md#q1-difference-between-oltp-and-olap-database-concepts) |
| Afternoon (2 hrs) | Indexes (Clustered vs Non-Clustered), Execution Plan reading, Covering Indexes | [01_SQL.md](./01_SQL.md#q7-difference-between-clustered-and-non-clustered-index) |
| Evening (2 hrs) | CTEs, Subqueries, Window Functions (ROW_NUMBER, DENSE_RANK, LEAD/LAG) | [01_SQL.md](./01_SQL.md#q11-what-is-cte-common-table-expression) |
| Night (1 hr) | Hands-on Practice: Delete Duplicates, Audit Triggers, Transactions (ACID) | [01_SQL.md](./01_SQL.md#q15-how-to-delete-duplicate-rows-in-sql) |

**Must-Practice Queries Today:**
- [ ] Delete duplicate rows using `ROW_NUMBER() OVER (PARTITION BY ...)`
- [ ] 2nd highest salary using `DENSE_RANK()` and `OFFSET-FETCH`
- [ ] Recursive CTE for employee manager hierarchy
- [ ] Create AFTER trigger for audit capturing with `inserted` / `deleted` tables
- [ ] Explain difference between DELETE, TRUNCATE, and DROP

---

### 📌 Day 2 – 12 Oct → C# & .NET Core Core Architecture
**Goal:** Master OOP pillars, SOLID principles, and Dependency Injection.

| Time | Topic | Resource |
|------|-------|----------|
| Morning (2 hrs) | SOLID Principles (Detailed code for SRP, OCP, LSP, ISP, DIP) | [02_DotNet_Core.md](./02_DotNet_Core.md#q3-explain-solid-principles-with-examples) |
| Afternoon (2 hrs) | Dependency Injection (Transient, Scoped, Singleton) + Service lifetimes | [02_DotNet_Core.md](./02_DotNet_Core.md#q23-dependency-injection-in-net-core) |
| Evening (2 hrs) | OOP Pillars, Interface vs Abstract Class, Sealed Class, Value vs Ref Types | [04_Generic_General.md](./04_Generic_General.md#bq1-four-pillars-of-oop-with-code-examples) |
| Night (1 hr) | Design Patterns: Repository Pattern, Unit of Work, Factory, Singleton | [02_DotNet_Core.md](./02_DotNet_Core.md#q18-design-patterns-used-in-net-core-projects) |

**Must-Practice Code Today:**
- [ ] Write a clean DIP implementation with interfaces and constructor injection
- [ ] Write custom Generic Repository `<T>`
- [ ] Implement Singleton Pattern using `Lazy<T>`

---

### 📌 Day 3 – 13 Oct → Async Programming, Middleware & Runtime
**Goal:** Master async/await, Garbage Collection, Middleware pipeline, and LINQ.

| Time | Topic | Resource |
|------|-------|----------|
| Morning (2 hrs) | Asynchronous programming (`async`/`await`, `Task.WhenAll`, avoiding deadlocks) | [02_DotNet_Core.md](./02_DotNet_Core.md#q4-sync-vs-async-programming-in-aspnet-core) |
| Afternoon (2 hrs) | Middleware Pipeline: Built-in order + writing custom logging middleware | [02_DotNet_Core.md](./02_DotNet_Core.md#q25-what-is-middleware-in-net-core) |
| Evening (2 hrs) | Garbage Collection (Gen 0/1/2, `IDisposable`, `using`), Null Safety | [02_DotNet_Core.md](./02_DotNet_Core.md#bq2-garbage-collection-gc-in-net) |
| Night (1 hr) | LINQ: Left Join, Deferred Execution (`IEnumerable` vs `IQueryable`) | [02_DotNet_Core.md](./02_DotNet_Core.md#q27-linq-basics--left-join-in-linq) |

**Must-Practice Code Today:**
- [ ] Asynchronous method with `CancellationToken`
- [ ] Custom Middleware with `InvokeAsync(HttpContext context, RequestDelegate next)`
- [ ] LINQ Left Join using `GroupJoin` and `DefaultIfEmpty()`

---

### 📌 Day 4 – 14 Oct → ASP.NET Core Web API, EF Core & Caching
**Goal:** Master REST API concepts, JWT Security, EF Core ORM, and Caching.

| Time | Topic | Resource |
|------|-------|----------|
| Morning (2 hrs) | Authentication & Authorization: JWT tokens, claims, roles, `[Authorize]` | [03_DotNet_API.md](./03_DotNet_API.md#q1-authentication-and-authorization-in-aspnet-core-web-api) |
| Afternoon (2 hrs) | HTTP Verbs, Status Codes (200, 201, 204, 400, 401, 403, 404, 500), Filters | [03_DotNet_API.md](./03_DotNet_API.md#q3-action-filters-in-aspnet-core) |
| Evening (2 hrs) | EF Core: Code-First vs DB-First, Migrations, `AsNoTracking()`, Eager loading | [04_Generic_General.md](./04_Generic_General.md#bq4-explain-the-code-first-and-database-first-approaches-in-entity-framework) |
| Night (1 hr) | CORS, Rate Limiting, Minimal APIs, In-Memory & Redis Caching | [03_DotNet_API.md](./03_DotNet_API.md#bq1-what-is-cors-and-how-do-you-configure-it-in-aspnet-core) |

**Must-Practice Code Today:**
- [ ] Program.cs setup for JWT Bearer authentication
- [ ] Global Exception Handling Middleware returning `ProblemDetails`
- [ ] Configure In-Memory Cache with `AbsoluteExpiration` and `SlidingExpiration`

---

### 📌 Day 5 – 15 Oct → Live C# Coding Problems & Mock Practice
**Goal:** Confidently solve live coding challenges on screen share / whiteboard.

| Time | Topic | Resource |
|------|-------|----------|
| Morning (2.5 hrs) | Top Coding Problems: Reverse string/words, Palindrome, 2nd Highest, Frequencies | [05_Coding_Challenges.md](./05_Coding_Challenges.md#1-reverse-a-string--words-in-a-sentence) |
| Afternoon (2 hrs) | Coding Problems: Two Sum, Balanced Parentheses, Fibonacci, Missing Number | [05_Coding_Challenges.md](./05_Coding_Challenges.md#6-two-sum-problem-find-pair-with-given-sum) |
| Evening (2 hrs) | **Mock Interview Simulation** (Answer technical questions verbally out loud) | All files |
| Night (1 hr) | Review weak areas identified during mock session | - |

**Must-Code on Notepad (Without Intellisense):**
- [ ] Reverse string with two pointers
- [ ] Second highest in array in single pass
- [ ] Balanced bracket validation using `Stack<char>`
- [ ] Count character frequencies with `Dictionary<char, int>`

---

### 📌 Day 6 – 16 Oct (Day Before Interview) → HR, Project Pitch & Final Polish
**Goal:** Polish project narrative, rehearse behavioral stories, and mental calm.

| Time | Topic | Resource |
|------|-------|----------|
| Morning (2 hrs) | Rehearse "Tell Me About Yourself" & Project Architecture Walkthrough | [06_HR_And_Scenario_Questions.md](./06_HR_And_Scenario_Questions.md#1-tell-me-about-yourself-structured-90-second-pitch) |
| Afternoon (2 hrs) | Production Bug story (STAR method) & Agile scenario questions | [06_HR_And_Scenario_Questions.md](./06_HR_And_Scenario_Questions.md#3-tell-me-about-a-challenging-production-bug-you-solved) |
| Evening (1.5 hrs) | Light review of cheatsheet index (Index limits, DI lifetimes, Status codes) | [README.md](./README.md) |
| Night | **Sleep early (by 10:30 PM)! No cramming tonight.** 😴 | Rest & Recharge |

---

### 🎯 Day 7 – 17-Oct-2026 → INTERVIEW DAY 🚀

- **Pre-Interview Checklist:**
  - [ ] Quiet room with good lighting and stable internet connection.
  - [ ] Laptop charged, webcam & microphone tested.
  - [ ] Notepad and pen next to you for sketching logic before coding.
  - [ ] Keep a glass of water nearby.
- **During the Interview:**
  - [ ] Listen carefully before answering; clarify requirements if a question is vague.
  - [ ] **Think Out Loud:** When coding, narrate your thinking: *"First I am checking for null inputs, then I'll use a two-pointer approach because it gives O(N) time complexity."*
  - [ ] If you don't know a specific syntax, explain the conceptual approach honestly.
  - [ ] End the interview by asking 1–2 thoughtful questions from [06_HR_And_Scenario_Questions.md](./06_HR_And_Scenario_Questions.md#7-intelligent-questions-to-ask-the-interviewer-at-the-end-of-the-interview).

---

*You have worked through all 96 questions and live coding scenarios. Walk in with full confidence!*
