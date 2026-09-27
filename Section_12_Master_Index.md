# 🗺️ Master Index — Complete .NET Developer Interview Preparation Package
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Switching Before December 2026

> **Welcome to your complete interview preparation system.** This index links all 11 sections, gives you a prioritized study plan, and tracks your preparation progress. Start here every time you sit down to study.

---

## 📦 Complete File Structure

```
d:\Int_Prep\
├── 📚 SOURCE FILES (Original BSK Documentation)
│   ├── BSK_Interview_QA_Guide.md                  (91KB — C# & .NET deep Q&A)
│   ├── BSK_Interview_SQL_Database_Guide.md         (26KB — SQL queries & patterns)
│   ├── BSK_Interview_Advanced_Systems_Guide.md     (44KB — Architecture & DevOps)
│   ├── BSK_Interview_Coding_Practice_Guide.md      (47KB — Algorithms)
│   └── BSK_Interview_AI_Native_Developer_Guide.md  (20KB — AI/LLM patterns)
│
└── 📖 STUDY SECTIONS (Your Personalized Prep Files)
    ├── Section_01_CSharp_Fundamentals.md           ← OOP, Generics, LINQ, async/await
    ├── Section_02_SQL_Database.md                  ← Joins, Indexes, CTEs, ACID
    ├── Section_03_DotNet_Core_WebAPI.md             ← JWT, Middleware, DI, Polly
    ├── Section_04_EF_Core_ORM.md                   ← DbContext, Migrations, N+1 Problem
    ├── Section_05_System_Design_Advanced.md         ← Architecture, CQRS, Docker
    ├── Section_06_Coding_Practice.md               ← Algorithms + Master Index
    ├── Section_07_HR_Behavioral_Round.md           ← STAR stories, Salary Negotiation
    ├── Section_08_Mock_Interview_Full.md           ← 60 Q&A mock interview rounds
    ├── Section_09_Quick_Revision_Cheatsheet.md     ← Night-before exam-style reference
    ├── Section_10_Resume_Talking_Points.md         ← Resume bullets, LinkedIn, GitHub
    ├── Section_11_Testing_Security_Observability.md← xUnit, Moq, OAuth, OpenTelemetry
    ├── Section_12_Master_Index.md                  ← THIS FILE
    └── Section_13_Advanced_Architecture.md         ← K8s Health Probes, gRPC, SignalR, Zero-Trust
```

---

## 📊 Section Overview — Priority & Time Estimate

| Section | Topic | Priority | Study Time | Interview Frequency |
|---|---|---|---|---|
| [Section 01](./Section_01_CSharp_Fundamentals.md) | C# Fundamentals | 🔴 CRITICAL | 3-4 hours | Asked in 95% of interviews |
| [Section 02](./Section_02_SQL_Database.md) | SQL & Database | 🔴 CRITICAL | 3-4 hours | Asked in 90% of interviews |
| [Section 03](./Section_03_DotNet_Core_WebAPI.md) | .NET Core & Web API | 🔴 CRITICAL | 4-5 hours | Asked in 95% of interviews |
| [Section 04](./Section_04_EF_Core_ORM.md) | EF Core & ORM | 🔴 HIGH | 2-3 hours | Asked in 80% of interviews |
| [Section 05](./Section_05_System_Design_Advanced.md) | System Design | 🟡 HIGH | 3-4 hours | Asked in 60% (higher at Sr level) |
| [Section 06](./Section_06_Coding_Practice.md) | Coding Algorithms | 🟡 MEDIUM | 2-3 hours | Asked in 70% |
| [Section 07](./Section_07_HR_Behavioral_Round.md) | HR & Behavioral | 🔴 CRITICAL | 2 hours | Asked in 100% |
| [Section 08](./Section_08_Mock_Interview_Full.md) | Mock Interviews | 🔴 CRITICAL | 2-3 hours/week | Practice tool |
| [Section 09](./Section_09_Quick_Revision_Cheatsheet.md) | Quick Revision | 🔴 CRITICAL | 30 min/night | Pre-interview ritual |
| [Section 10](./Section_10_Resume_Talking_Points.md) | Resume & LinkedIn | 🔴 CRITICAL | 2-3 hours (once) | Prerequisite |
| [Section 11](./Section_11_Testing_Security_Observability.md) | Testing & Security | 🟡 MEDIUM | 2-3 hours | Asked in 50-60% |
| [Section 13](./Section_13_Advanced_Architecture.md) | Advanced Architecture (K8s, gRPC, Zero-Trust) | 🟢 BONUS | 1-2 hours | Asked in 30-40% (Sr level) |

---

## 📅 Recommended Study Plan — 8 Weeks to December

### Phase 1: Foundation (Week 1-2)

| Day | Morning (20 min) | Lunch (10 min) | Evening (25 min) |
|---|---|---|---|
| Mon | Section 01: OOP + Types | LeetCode Easy (C#) | Section 10: Update Resume |
| Tue | Section 01: LINQ + async | LeetCode Easy (C#) | Section 10: Update LinkedIn |
| Wed | Section 01: Patterns + SOLID | LeetCode Easy (C#) | Mock: C# Q1-Q15 (Section 08) |
| Thu | Section 02: Joins + Indexes | SQL Fiddle practice | Section 07: Practice intro |
| Fri | Section 02: CTEs + Window Fns | SQL practice | Mock: SQL Q26-Q32 (Section 08) |
| Sat | Section 03: JWT + Middleware | Rest | Full section review |
| Sun | Rest | — | Read Section 09 cheatsheet |

---

### Phase 2: Deepening (Week 3-4)

| Day | Morning (20 min) | Lunch (10 min) | Evening (25 min) |
|---|---|---|---|
| Mon | Section 03: DI Lifetimes | LeetCode Medium | Mock: .NET Round (Q16-Q25) |
| Tue | Section 03: Polly + Background Svc | Apply to 3 companies | Section 07: STAR stories |
| Wed | Section 04: EF Core + Migrations | LeetCode Medium | Mock: EF questions |
| Thu | Section 04: N+1 + Dapper | SQL practice | Section 11: Unit testing |
| Fri | Section 05: BSK Architecture | Apply to 3 companies | System Design Q33-Q37 |
| Sat | Section 11: Testing + OAuth | LeetCode practice | Mock: Full round 1 |
| Sun | Rest | — | Section 09 cheatsheet |

---

### Phase 3: Active Interviews (Week 5-8)

- **Apply daily**: 2-5 companies per day (LinkedIn Easy Apply + Company websites)
- **Morning routine**: Read 2-3 Q&As from weak areas
- **After each interview**: Write down questions that surprised you → add to notes
- **Weekly mock**: 1 full mock interview round (all 60 questions)
- **Night before interview**: Read Section 09 in full

---

## 🎯 Top 20 Questions to Master Absolutely

These are the questions that appear in virtually every .NET backend interview:

| # | Question | Section |
|---|---|---|
| 1 | Explain DI Lifetimes with Captive Dependency example | [Section 03](./Section_03_DotNet_Core_WebAPI.md) |
| 2 | How does async/await work? What is thread starvation? | [Section 01](./Section_01_CSharp_Fundamentals.md) |
| 3 | Walk me through your BSK project architecture | [Section 10](./Section_10_Resume_Talking_Points.md) |
| 4 | Clustered vs Non-Clustered Index | [Section 02](./Section_02_SQL_Database.md) |
| 5 | SOLID Principles with code examples | [Section 01](./Section_01_CSharp_Fundamentals.md) |
| 6 | How does JWT authentication work? | [Section 03](./Section_03_DotNet_Core_WebAPI.md) |
| 7 | N+1 Problem — how to detect and fix | [Section 04](./Section_04_EF_Core_ORM.md) |
| 8 | Middleware vs Action Filter | [Section 03](./Section_03_DotNet_Core_WebAPI.md) |
| 9 | What is IHttpClientFactory and why? | [Section 03](./Section_03_DotNet_Core_WebAPI.md) |
| 10 | Explain your biggest technical achievement | [Section 07](./Section_07_HR_Behavioral_Round.md) |
| 11 | Write a SQL query with INNER JOIN, LEFT JOIN | [Section 02](./Section_02_SQL_Database.md) |
| 12 | Value vs Reference types | [Section 01](./Section_01_CSharp_Fundamentals.md) |
| 13 | Stored Procedure vs Function in SQL | [Section 02](./Section_02_SQL_Database.md) |
| 14 | What is Repository Pattern? | [Section 01](./Section_01_CSharp_Fundamentals.md) |
| 15 | EF Core .AsNoTracking() — when to use | [Section 04](./Section_04_EF_Core_ORM.md) |
| 16 | `throw` vs `throw ex` | [Section 01](./Section_01_CSharp_Fundamentals.md) |
| 17 | Task.WhenAll vs Task.WhenAny | [Section 01](./Section_01_CSharp_Fundamentals.md) |
| 18 | Circuit Breaker Pattern (Polly) | [Section 03](./Section_03_DotNet_Core_WebAPI.md) |
| 19 | Two Sum algorithm (HashMap) | [Section 06](./Section_06_Coding_Practice.md) |
| 20 | Why are you switching jobs? | [Section 07](./Section_07_HR_Behavioral_Round.md) |

---

## 📈 Progress Tracker

Use this as a checklist. Mark each section when you've read it AND practiced the mock questions:

### Sections Read

- [ ] Section 01: C# Fundamentals
- [ ] Section 02: SQL & Database
- [ ] Section 03: .NET Core & Web API
- [ ] Section 04: EF Core & ORM
- [ ] Section 05: System Design & Advanced
- [ ] Section 06: Coding Practice
- [ ] Section 07: HR & Behavioral
- [ ] Section 08: Mock Interview (Round 1)
- [ ] Section 08: Mock Interview (Round 2 — repeat)
- [ ] Section 09: Quick Revision Cheatsheet
- [ ] Section 10: Resume Updated + LinkedIn Optimized
- [ ] Section 11: Testing, Security & Observability
- [ ] Section 13: Advanced Architecture (K8s, gRPC, Zero-Trust)

### Practical Tasks

- [ ] Resume updated with BSK bullet points from Section 10
- [ ] LinkedIn headline + About section updated
- [ ] GitHub profile README created
- [ ] AZ-900 certification started
- [ ] Applied to 10+ companies
- [ ] Done 1 full mock interview round
- [ ] Done 2+ full mock interview rounds
- [ ] 5+ real interviews completed

---

## 🔍 Quick-Find Index — By Topic

| If asked about... | Go to... |
|---|---|
| Kubernetes health probes | Section 13, Q1-Q2 |
| Clean Architecture | Section 13, Q3-Q4 |
| Correlation ID / Request Tracing | Section 13, Q5 |
| Zero-Trust Security | Section 13, Q6 |
| SSO & External IdP (Entra ID) | Section 13, Q7 |
| Memory Leak detection | Section 13, Q8-Q9 |
| gRPC vs REST | Section 13, Q11 |
| SignalR real-time | Section 13, Q12 |
|---|---|
| OOP Pillars | Section 01, Q1 |
| async/await internals | Section 01, Q12-Q14 |
| Records, Pattern Matching, Nullable | Section 01, Q17-Q19 |
| SOLID | Section 01, Q20 |
| Repository Pattern | Section 01, Q22 |
| Singleton Lazy<T> | Section 01, Q21 |
| SQL Joins | Section 02, Q3 |
| SQL Indexes | Section 02, Q6-Q8 |
| CTEs | Section 02, Q11 |
| Window Functions | Section 02, Q12 |
| ACID | Section 02, Q19 |
| Dapper vs EF Core | Section 02, Q21 |
| JWT Auth full flow | Section 03, Q9 |
| Middleware order | Section 03, Q11 |
| Custom Middleware | Section 03, Q12 |
| DI Lifetimes + Captive Dep | Section 03, Q14-Q15 |
| IHostedService / BackgroundService | Section 03, Q17 |
| IHttpClientFactory | Section 03, Q18 |
| Polly patterns | Section 03, Q19 |
| Caching (Cache-Aside) | Section 03, Q20 |
| EF Core Migrations | Section 04, Q3 |
| N+1 Problem | Section 04, Q4 |
| EF Core Transactions | Section 04, Q5-Q6 |
| .AsNoTracking() | Section 04, Q7 |
| Dapper multi-mapping | Section 04, Q14 |
| BSK Architecture | Section 05, Q1-Q2 |
| CQRS | Section 05, Q3 |
| Outbox Pattern | Section 05, Q4 |
| Saga Pattern | Section 05, Q5 |
| Docker multi-stage | Section 05, Q10 |
| OWASP protection | Section 05, Q8 + Section 11, Q13 |
| Coding algorithms | Section 06, Q1-Q21 |
| Time complexity | Section 06, Table |
| HR self-intro | Section 07, Q1 |
| Why switching | Section 07, Q2 |
| STAR stories | Section 07, Q4-Q8 |
| Salary negotiation | Section 07, Q6 |
| Unit Testing (xUnit+Moq) | Section 11, Q1-Q6 |
| Integration Testing | Section 11, Q7-Q8 |
| OAuth/OIDC | Section 11, Q11-Q12 |
| OpenTelemetry | Section 11, Q15 |
| Event-Driven + RabbitMQ | Section 11, Q17-Q19 |
| Resume bullets | Section 10, Q1 |
| 2-min project walkthrough | Section 10, Q2 |
| Interview TRAP questions | Section 09, bottom table |
| Mock Q&A full set | Section 08, all |
| Night-before cheatsheet | Section 09 (entire file) |

---

## 🚀 BSK Project Elevator Pitch (Memorize This)

**30-second version**:
> *"I work on BSK — an insurance claims management platform built on ASP.NET Core 8. I've built features like JWT auth with distributed token blacklisting, vendor resilience using Polly circuit breakers, and a proactive health monitoring background service that eliminated 30-second API timeouts during vendor outages. We use EF Core for CRUD and Dapper for complex reporting queries."*

**2-minute version**: See [Section 10, Q2](./Section_10_Resume_Talking_Points.md)

---

## 📌 Key Numbers to Remember for BSK

| Metric | Value |
|---|---|
| Number of external vendor APIs | **7** (Surepass, Razorpay, SMS, AWS S3, Accounting, etc.) |
| Dashboard optimization | **8-12 seconds → under 1.5 seconds** (7x improvement) |
| Vendor timeout before fix | **30 seconds** blocking |
| Vendor fail-fast after fix | **<10ms** response |
| DB CPU reduction | **~40%** after index optimization |
| Health monitor interval | Every **60 seconds** (PeriodicTimer) |
| Polly retry count | **3 retries** with exponential backoff |
| Circuit breaker failure threshold | **50%** failure rate over 30s window |
| Circuit breaker break duration | **15 seconds** before half-open state |

---

## 💡 Final Tips

### When You Don't Know an Answer
> *"That's a great question. I haven't worked with that specific technology, but here's how I'd approach thinking about it based on similar patterns I've used..."*

### When You Make a Mistake
> *"Actually, let me correct that — I misspoke. The correct answer is..."*
> (Correcting yourself shows intellectual honesty and attention to detail)

### When the Question Is Ambiguous
> *"Just to make sure I'm answering the right thing — are you asking about X or Y?"*
> (Clarifying is better than answering the wrong question perfectly)

### The Golden Rule
> **Always relate your answer back to BSK.** Even if the question is general, say "In the BSK project, we handle this by..." — it transforms theoretical knowledge into demonstrated experience.

---

## 🎉 You've Got This!

You have **2.5 years of real production experience** with a complex, multi-vendor insurance platform. That's genuinely impressive. Most candidates at your level have only done CRUD apps or tutorial projects.

Your BSK experience gives you stories about:
- **Real scale**: Multiple concurrent users, production insurance data
- **Real integrations**: 7 external APIs with resilience patterns
- **Real performance optimization**: Measurable query improvements
- **Real security**: JWT + AES encryption + distributed caching
- **Real architecture**: Clean N-tier with background services

Go into every interview with confidence. You've built production software. Now you just need to **articulate** it well. That's exactly what these 11 sections help you do.

**Good luck with your job switch! December is achievable. 🚀**

---

*Created: September 2026 | Sections: 12 | Based on: 5 BSK project documentation files*
