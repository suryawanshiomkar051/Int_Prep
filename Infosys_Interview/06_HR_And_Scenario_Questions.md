# 🤝 Infosys Interview – HR, Managerial & Project Scenario Guide

> **Make a Lasting Impression!**  
> Technical knowledge gets you through the screening, but clear communication, project articulation, and structured behavioral answers secure the job offer.  
> Use the **STAR Method** (Situation, Task, Action, Result) for all scenario-based questions.

---

## 1. "Tell Me About Yourself" (Structured 90-Second Pitch)

### 🎯 Formula:
1. **Who you are + Current role** (20 seconds)
2. **Key Technical Strengths** (30 seconds)
3. **Notable Project / Accomplishment** (25 seconds)
4. **Why you are excited about this opportunity at Infosys** (15 seconds)

### Sample Answer:
> *"Hello, thank you for this opportunity. I am a .NET Developer with [X] years of experience designing and developing enterprise web applications, RESTful APIs, and relational databases.*  
>  
> *My core technical stack centers around C#, .NET 8 / .NET Core, ASP.NET Core Web API, Entity Framework Core, and SQL Server. In my current role at [Current Company], I am part of the core backend team building [e.g., a scalable financial reporting / e-commerce / order management system]. I actively work on implementing microservices, optimizing SQL queries, creating secure JWT-authenticated endpoints, and maintaining clean code following SOLID principles and design patterns like Repository and Unit of Work.*  
>  
> *One of my recent achievements was optimizing a heavy reporting endpoint by replacing unindexed table scans with covering indexes and AsNoTracking queries, reducing latency by over 60%.*  
>  
> *I am looking forward to joining Infosys because of its scale, global clientele, and the opportunity to work on mission-critical enterprise digital transformation projects where I can contribute my backend skills and continue growing."*

---

## 2. "Explain Your Current Project Architecture"

### 🎯 How to Explain Clearly:
Draw a mental diagram from client to database:

1. **Client / Front-End:**
   - Single Page Application (Angular / React) or mobile client making HTTPS calls.
2. **API Gateway / Ingress:**
   - Handles SSL termination, routing, CORS, rate limiting.
3. **Application Layer (ASP.NET Core Web API):**
   - Controllers / Endpoints for REST contracts.
   - Middleware Pipeline for Global Exception Handling, JWT validation, and Request Logging (Serilog).
4. **Business / Service Layer:**
   - Core domain logic, validation (FluentValidation), mapping (AutoMapper / Mapperly).
   - Loose coupling through Dependency Injection (`AddScoped`).
5. **Data Access Layer:**
   - Entity Framework Core / Dapper using Repository and Unit of Work pattern.
   - SQL Server database with clustered and non-clustered indexes.
   - Redis distributed cache for frequently accessed static master data (product catalogs, dropdowns).
6. **DevOps & CI/CD:**
   - Azure DevOps / GitHub Actions running automated xUnit tests and deploying to Azure App Service / Docker Kubernetes.

---

## 3. "Tell Me About a Challenging Production Bug You Solved"

### Sample Story (STAR Method):

- **Situation:**  
  *"During peak business hours, our order checkout API began returning HTTP 504 Gateway Timeouts, causing customers to face payment verification failures."*
- **Task:**  
  *"As the on-call backend engineer, I needed to identify the root cause without taking the system offline and restore checkout response times to under 300ms."*
- **Action:**  
  *"I inspected application logs in Application Insights and noticed thread pool starvation warnings alongside SQL connection pool exhaustion. Drilling down into SQL Server Dynamic Management Views (`sys.dm_exec_requests`), I discovered two issues:*  
  1. *A newly added notification call was blocking synchronously (`.Result` on an async Task), causing a thread sync deadlock.*  
  2. *An unindexed foreign key in the Orders table was causing clustered index table scans during concurrent transactions.*  
  *I converted the blocking call to pure `await`, added a non-clustered covering index on `CustomerID` and `OrderDate`, and introduced `AsNoTracking()` on read queries."*
- **Result:**  
  *"Response latency dropped from 8 seconds down to 180 milliseconds, CPU utilization normalized to 25%, and zero timeouts occurred thereafter. I also documented the findings in our post-mortem and added automated code analysis rules to prevent synchronous blocking."*

---

## 4. "How Do You Handle Disagreements with a Teammate or Lead?"

### Key Message: Objective data > Ego.
> *"Whenever there is a disagreement on technical approach—for example, choosing between Dapper vs EF Core for a new microservice—I always ground the discussion in objective project requirements and measurable benchmarks rather than personal preference.*  
>  
> *I listen to the other person's reasoning first to understand their perspective on maintainability, developer productivity, or performance. Then, I prepare a small prototype or benchmark test showing memory footprint and execution speed.*  
>  
> *If the lead makes a decision that differs from my preference, I respect the lead's final call, fully commit to delivering the solution, and ensure it is tested thoroughly."*

---

## 5. "How Do You Handle Requirement Changes Mid-Sprint?"

### Key Points:
1. Evaluate impact on the current sprint backlog and sprint goal.
2. Communicate transparently with the Product Owner / Scrum Master.
3. Propose trade-offs: if a new critical story is brought in, a story of equal effort must be pushed to the next sprint.
4. Assess architectural impact to ensure the change doesn't introduce technical debt.

---

## 6. "Why Infosys?"

> *"Infosys is recognized globally for its engineering excellence, continuous learning culture, and large-scale digital transformation initiatives across Fortune 500 clients.  
> What appeals most to me is the opportunity to work in a mature Agile delivery ecosystem with modern cloud and .NET architectures. Given my experience in building scalable .NET backend APIs and SQL optimization, I know I can hit the ground running, deliver value to your clients, and continue expanding my cloud and distributed systems capabilities alongside experienced architects."*

---

## 7. Intelligent Questions to Ask the Interviewer (At the End of the Interview)

Never say *"No questions, sir"*. Asking thoughtful questions shows enthusiasm and seniority:

1. *"What does the typical tech stack and microservices architecture look like for the project this role is aligned with?"*
2. *"How does the team balance delivering new business features with addressing technical debt and upgrading .NET versions?"*
3. *"What are the key qualities that differentiate an average developer from a top performer on your team at Infosys?"*

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep | HR & Managerial Module*
