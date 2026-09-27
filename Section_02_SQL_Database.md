# 🗄️ Section 2: SQL & Database — Interview Preparation Guide
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> **BSK uses SQL Server 2022.** All examples below are real SQL Server syntax. Reference BSK queries whenever possible in interviews.

---

## 📋 Table of Contents

1. [Database Fundamentals](#1-database-fundamentals)
2. [SQL Joins](#2-sql-joins)
3. [SQL Functions](#3-sql-functions)
4. [Indexes & Performance](#4-indexes--performance)
5. [Stored Procedures vs Functions](#5-stored-procedures-vs-functions)
6. [CTEs & Window Functions](#6-ctes--window-functions)
7. [Normalization & Design](#7-normalization--design)
8. [Performance Optimization](#8-performance-optimization)
9. [Transaction Management & ACID](#9-transaction-management--acid)
10. [Audit & Advanced Scenarios](#10-audit--advanced-scenarios)
11. [Quick-Fire SQL Checklist](#11-quick-fire-sql-checklist)

---

## 1. Database Fundamentals

### Q1: OLTP vs OLAP — What is BSK?

| Feature | OLTP (BSK's DB) | OLAP (Analytics) |
|---|---|---|
| **Purpose** | Day-to-day transactions (CRUD) | Analyze historical data, trends |
| **Data Structure** | Highly normalized (3NF) | Denormalized (Star/Snowflake schema) |
| **Query Type** | Simple fast queries — single records | Complex aggregations over millions of rows |
| **Response Time** | Milliseconds | Seconds to hours |
| **Examples** | SQL Server, MySQL, PostgreSQL | Snowflake, AWS Redshift, BigQuery |

**BSK Context**: BSK API database is strictly **OLTP** — handles real-time claim registration, KYC uploads, payment webhooks via SQL Server. ACID compliance and fast response times are vital.

---

### Q2: SQL vs NoSQL — when to use each?

**SQL (Relational) — BSK uses this**:
- Structured data with strict schemas
- Complex relationships enforced via foreign keys
- ACID compliance for transactions
- Use when: data integrity is paramount

**NoSQL (Non-Relational)**:
- Schema-less (MongoDB = JSON documents)
- High-throughput writes, rapidly changing structure
- **BSK Scenario**: If BSK wants to store raw vendor API request/response payloads (Surepass, Razorpay JSON logs vary per vendor), MongoDB would be ideal — SQL Server struggles with unstructured varying JSON schemas.

---

## 2. SQL Joins

### Q3: All Join Types with real BSK examples

```
[INNER JOIN]  — Rows present in BOTH tables
[LEFT JOIN]   — All left table rows + matched right (NULL if no match)
[RIGHT JOIN]  — All right table rows + matched left (rarely used)
[FULL JOIN]   — All rows from BOTH, NULL where no match
[CROSS JOIN]  — Cartesian product (every row A × every row B)
```

**Real BSK Production Query (INNER + LEFT JOIN)**:
```sql
-- From BSK's Q000_ExecutiveDashboard.sql
SELECT
    l.Id,
    per.Name,
    CASE
        WHEN l.SourceTypeId IN (2, 3) AND TRY_CAST(l.SourceId AS UNIQUEIDENTIFIER) IS NOT NULL
             THEN ISNULL(pr.Name, o.OrganizationName)
        ELSE ISNULL(l.SourceId, 'Direct')
    END AS Source
FROM Lead l WITH (NOLOCK)
-- INNER JOIN: Lead MUST have a matching Person
INNER JOIN Person per WITH (NOLOCK) ON l.PersonId = per.Id
-- LEFT JOIN: Optional Partner details (null if direct lead)
LEFT JOIN Partner p WITH (NOLOCK) ON TRY_CAST(l.SourceId AS UNIQUEIDENTIFIER) = p.Id
LEFT JOIN Person pr WITH (NOLOCK) ON p.PersonId = pr.Id
LEFT JOIN Organization o WITH (NOLOCK) ON p.OrganizationId = o.Id;
```

**CROSS JOIN example (BSK Dashboard — compare current vs previous period)**:
```sql
SELECT curr.TotalLeads, prev.TotalLeads, curr.TotalLeads - prev.TotalLeads AS Diff
FROM Metrics curr, Metrics prev  -- Implicit CROSS JOIN (single-row CTEs)
WHERE curr.Period = 'Current' AND prev.Period = 'Previous';
```

---

## 3. SQL Functions

### Q4: Aggregate vs Scalar Functions — BSK examples

**Aggregate Functions** (operate on multiple rows → return one value):
```sql
COUNT(Id) AS TotalCases           -- Count rows
SUM(pay.Amount) AS TotalPayout    -- From Q031_TopPerformingPartners.sql
AVG(DATEDIFF(DAY, CreatedAt, GETDATE())) AS AvgDaysOpen
MAX(ClaimAmount) AS HighestClaim
MIN(CreatedAt) AS EarliestCase
```

**Scalar Functions** (operate on single row → return single value):
```sql
COALESCE(per.Name, org.OrganizationName)  -- First non-NULL value
ISNULL(p.PanNo, '')                        -- Replace NULL with default
FORMAT(l.CreatedAt, 'dd MMM yyyy')         -- "27 Sep 2026"
TRY_CAST(l.SourceId AS UNIQUEIDENTIFIER)   -- Safe cast, returns NULL on failure
DATEADD(MONTH, -1, @StartDate)             -- Date arithmetic
DATEDIFF(DAY, CreatedAt, GETDATE())        -- Difference between dates
```

---

### Q5: Subqueries — Correlated vs Non-Correlated

**Non-Correlated** (independent — executes once):
```sql
-- From BSK Executive Dashboard
SELECT
    'Current' AS Period,
    (SELECT COUNT(Id) FROM Lead WITH (NOLOCK)
     WHERE CreatedAt BETWEEN @StartDate AND @EndDate) AS TotalLeads
```

**Correlated** (references outer query — executes once per outer row):
```sql
-- Cases with claim amount above average for their policy type
SELECT c.Id, c.ClaimantId, c.ClaimAmount, c.PolicyTypeId
FROM [Case] c
WHERE c.ClaimAmount > (
    SELECT AVG(sub.ClaimAmount)
    FROM [Case] sub
    WHERE sub.PolicyTypeId = c.PolicyTypeId  -- References outer c
);
```

---

## 4. Indexes & Performance

### Q6: Clustered vs Non-Clustered Index

```
Clustered Index    → Stores actual table DATA rows at leaf level
Non-Clustered Index → Stores pointers (bookmarks) back to clustered index rows
```

| Feature | Clustered | Non-Clustered |
|---|---|---|
| **Physical Storage** | Reorganizes physical row order | Separate from table data |
| **Max Per Table** | **EXACTLY 1** | Up to 999 |
| **Default Creation** | Created with PRIMARY KEY | Created manually |
| **Leaf Level** | Contains actual data rows | Contains index key + pointer to data |

**Why only 1 clustered index?** Table rows can only be physically sorted ONE way (like a dictionary — sorted by word). A table without a clustered index is called a **Heap**.

**Covering Index** (fixes Key Lookup problem):
```sql
-- Key Lookup problem: Non-clustered index found the key, but must
-- jump back to clustered index to get other columns = expensive!

-- Solution: INCLUDE columns in the non-clustered index
CREATE NONCLUSTERED INDEX IX_Case_ClaimantId
ON [Case] (ClaimantId)
INCLUDE (ClaimAmount, StatusId, CreatedAt); -- These columns stored IN the index leaf
-- Now SQL can serve the query WITHOUT a key lookup!
```

---

### Q7: Execution Plans — How to read and optimize

**How to check in SSMS**:
- **Estimated Plan** (Ctrl+L) — Shows plan without running. Safe for production.
- **Actual Plan** (Ctrl+M) — Runs the query and shows real resource usage.

**How to read**: Right-to-Left, Top-to-Bottom. Thicker arrow = more data flowing.

**Key operators to identify**:

| Operator | Meaning | Action |
|---|---|---|
| **Table Scan** | Full scan of entire table — no index | Create an index! |
| **Clustered Index Scan** | Scanning all rows via clustered index | Narrow query or add index |
| **Index Seek** | ✅ GOOD — B-Tree jump to exact rows | Already optimized |
| **Key Lookup** | Found key in non-clustered, jumped to clustered for other cols | Add INCLUDE columns |
| **Hash Match** | Expensive join — large unindexed tables | Index the join column |

---

### Q8: SARGability — Making queries index-friendly

```sql
-- NON-SARGable (forces index scan — bad!)
WHERE YEAR(CreatedAt) = 2025           -- Function on column prevents seek
WHERE LEFT(ClaimantName, 3) = 'OmK'   -- Function on column

-- SARGable (index seek — good!)
WHERE CreatedAt BETWEEN '2025-01-01' AND '2025-12-31'
WHERE ClaimantName LIKE 'OmK%'        -- Prefix LIKE is SARGable

-- BSK reference: Dashboard uses WITH (NOLOCK) to avoid read locks
FROM Lead l WITH (NOLOCK)  -- Dirty read, no shared locks, faster for dashboards
```

---

## 5. Stored Procedures vs Functions

### Q9: Stored Procedure vs User-Defined Function

| Feature | Stored Procedure | User-Defined Function (UDF) |
|---|---|---|
| **Can modify data?** | YES (INSERT, UPDATE, DELETE) | NO — side-effect free |
| **Returns** | Result set, output params, or nothing | Scalar value or table |
| **Can call SP?** | Can call functions | CANNOT call stored procedures |
| **Transaction control** | YES | NO |
| **Called in SELECT?** | NO | YES (scalar UDFs) |

**Why can't UDF call SP?**
SQL Server requires functions to be **deterministic and side-effect free**. Stored procedures can modify data — this would violate function purity contracts.

---

### Q10: View vs Materialized View (Indexed View in SQL Server)

| Feature | Standard View | Materialized View (Indexed View) |
|---|---|---|
| **Data Storage** | No physical storage — virtual table | Physically stores result on disk |
| **Always up-to-date?** | YES — executes query each time | Maintained automatically by SQL Server |
| **Read Performance** | Can be slow for complex joins | Very fast — pre-computed data |
| **Write Performance** | No overhead | Slight overhead — SQL Server updates indices |

```sql
-- Standard View
CREATE VIEW ActiveCaseSummary AS
SELECT c.Id, cl.Name, c.ClaimAmount, c.StatusId
FROM [Case] c
INNER JOIN Claimant cl ON c.ClaimantId = cl.Id
WHERE c.IsActive = 1;

-- Query it like a table
SELECT * FROM ActiveCaseSummary WHERE ClaimAmount > 100000;
```

---

## 6. CTEs & Window Functions

### Q11: CTE (Common Table Expression) — BSK Dashboard Usage

A **CTE** is a temporary named result set within a single SQL statement.

```sql
-- BSK's Q000_ExecutiveDashboard.sql pattern:
-- Get the LATEST stage log for each case (row_number trick)
;WITH LatestStages AS (
    SELECT
        CaseId,
        LogId,
        ROW_NUMBER() OVER(PARTITION BY CaseId ORDER BY CreatedAt DESC) AS rn
    FROM CaseLog WITH (NOLOCK)
)
-- Use CTE in the main dashboard query
SELECT
    (SELECT COUNT(c.Id)
     FROM [Case] c WITH (NOLOCK)
     LEFT JOIN LatestStages ls ON c.Id = ls.CaseId AND ls.rn = 1
     WHERE c.IsCaseReached != 1 AND (ls.LogId IS NULL OR ls.LogId >= 4)
    ) AS OngoingCases
```

**Benefits of CTE**:
1. **Readability** — Break massive queries into named steps (like C# variables)
2. **Recursive Queries** — Traverse hierarchical data (org charts, categories)

---

### Q12: Window Functions

```sql
-- ROW_NUMBER() — Unique sequential number within partition
SELECT
    CaseId,
    CreatedAt,
    ROW_NUMBER() OVER(PARTITION BY CaseId ORDER BY CreatedAt DESC) AS rn
FROM CaseLog;

-- RANK() — Allows gaps when tied; DENSE_RANK() — No gaps when tied
SELECT ClaimantName, ClaimAmount,
    RANK() OVER(ORDER BY ClaimAmount DESC) AS ClaimRank
FROM [Case];

-- LEAD/LAG — Access next/previous row value
SELECT CaseId, CreatedAt,
    LAG(CreatedAt) OVER(PARTITION BY ClaimantId ORDER BY CreatedAt) AS PrevCaseDate
FROM [Case];
```

---

### Q13: UNION vs UNION ALL

```sql
-- UNION — removes duplicates (sorts + compares = SLOW)
SELECT Name FROM Claimant
UNION
SELECT Name FROM Partner;

-- UNION ALL — keeps duplicates (NO sorting = FAST)
SELECT Name FROM Claimant
UNION ALL
SELECT Name FROM Partner;

-- BSK Dashboard uses UNION ALL for speed:
SELECT 'Follow-ups Due Today' AS Label, COUNT(Id) AS Count, 'warning' AS Type
FROM Activity WITH (NOLOCK) WHERE CAST(ScheduledDate AS DATE) = CAST(GETDATE() AS DATE)
UNION ALL
SELECT 'KYC Overdue (>7d)', COUNT(Id), 'danger'
FROM [Case] WITH (NOLOCK) WHERE IsCaseReached != 1 AND CreatedAt < DATEADD(DAY, -7, GETDATE())
UNION ALL
SELECT 'High Value Claims (>5L)', COUNT(Id), 'primary'
FROM [Case] WITH (NOLOCK) WHERE ClaimAmount > 500000;
```

> **Rule**: Always use `UNION ALL` unless you explicitly need deduplication.

---

## 7. Normalization & Design

### Q14: Database Normalization — Why and How?

**Normalization** = Organizing data to reduce redundancy and eliminate anomalies.

| Normal Form | Rule |
|---|---|
| **1NF** | Atomic values in each cell, no repeating column groups |
| **2NF** | 1NF + all non-key attributes fully depend on PRIMARY KEY |
| **3NF** | 2NF + no transitive dependencies (non-key depending on non-key) |

**BSK Example — Why normalize?**

Without normalization, storing claimant name+mobile directly in Case table:
- **Redundancy**: Claimant with 3 cases → name/mobile duplicated 3 times
- **Update Anomaly**: Mobile number changes → must update all 3 rows. Miss one → inconsistent data!

**BSK Solution (normalized)**:
```
[Case] → ClaimantId (FK) → [Claimant] → PersonId (FK) → [Person] → AddressId (FK) → [Address]
```
Update address in ONE place — all cases see the correct data automatically.

---

### Q15: Delete Duplicate Rows (CTE + ROW_NUMBER trick)

```sql
WITH CTE_Duplicates AS (
    SELECT
        Id,
        PersonId,
        CreatedAt,
        ROW_NUMBER() OVER (
            PARTITION BY PersonId        -- Define "duplicate" = same PersonId
            ORDER BY CreatedAt ASC       -- Keep the OLDEST record (rn = 1)
        ) AS RowNumber
    FROM Lead
)
DELETE FROM CTE_Duplicates
WHERE RowNumber > 1;  -- Delete all but the first occurrence
```

---

### Q16: Get last record added to a table

```sql
-- Method 1: ORDER BY DESC + TOP 1
SELECT TOP 1 Id, CreatedAt, Name
FROM Person WITH (NOLOCK)
ORDER BY CreatedAt DESC;

-- Method 2: Inside stored procedure after INSERT
DECLARE @NewId INT;
INSERT INTO Lead (...) VALUES (...);
SET @NewId = SCOPE_IDENTITY();   -- Safe — current scope only
-- @@IDENTITY is UNSAFE if triggers exist (crosses scope)
-- IDENT_CURRENT('Lead') returns last for table across ALL sessions
```

---

## 8. Performance Optimization

### Q17: Top SQL Performance Techniques

1. **Execution Plans** — Identify Table Scans, Key Lookups, Hash Joins.
2. **WITH (NOLOCK)** — Dirty reads on dashboards/reports (BSK uses extensively).
3. **SARGable predicates** — Avoid functions on indexed columns.
4. **Covering Indexes** — INCLUDE columns in non-clustered indexes.
5. **Parameterized Queries** — Avoid plan recompilations; EF Core does this automatically.
6. **`.AsNoTracking()`** — EF Core: disables change tracking for read-only queries.

```sql
-- With NOLOCK: Prevents read queries from blocking write operations
-- Used throughout BSK Dashboard queries
SELECT COUNT(Id) FROM Lead WITH (NOLOCK) WHERE CreatedAt > '2025-01-01'

-- Parameterized query (EF Core generates this automatically)
SELECT * FROM [Case] WHERE Id = @p0  -- @p0 is parameterized, plan is cached
```

---

### Q18: USING clause — SQL Server caveat

> ⚠️ **SQL Server does NOT support the `USING` clause** (it's ANSI SQL, supported in PostgreSQL/MySQL/Oracle).

```sql
-- ANSI SQL USING syntax (NOT in SQL Server)
SELECT * FROM Case JOIN Claimant USING (ClaimantId);  -- ERROR in SQL Server!

-- SQL Server: Must use explicit ON clause
SELECT c.Id, cl.Name
FROM [Case] c
JOIN Claimant cl ON c.ClaimantId = cl.Id;
```

---

## 9. Transaction Management & ACID

### Q19: What is ACID? With BSK Example

| Property | Meaning | BSK Example |
|---|---|---|
| **Atomicity** | All or nothing — whole transaction succeeds or fails | Claimant created + Case created, or BOTH rollback |
| **Consistency** | DB remains in valid state before and after | FK constraints — Case.ClaimantId must exist in Claimant |
| **Isolation** | Concurrent transactions don't interfere | NOLOCK for reads, locks for writes |
| **Durability** | Committed data survives system failure | SQL Server transaction log ensures persistence |

**Explicit Transaction Example (Multi-repository)**:
```sql
BEGIN TRANSACTION;

INSERT INTO Claimant (Id, Name) VALUES (@ClaimantId, @Name);
INSERT INTO [Case] (Id, ClaimantId, ClaimAmount) VALUES (@CaseId, @ClaimantId, @Amount);

-- If all succeed:
COMMIT TRANSACTION;

-- If anything fails:
ROLLBACK TRANSACTION;
```

---

## 10. Audit & Advanced Scenarios

### Q20: Implement Audit Capturing

**Option A: SQL Trigger (Database-Level — catches ALL changes)**:
```sql
-- Audit table
CREATE TABLE CaseAuditLog (
    AuditId INT IDENTITY(1,1) PRIMARY KEY,
    CaseId UNIQUEIDENTIFIER,
    OldClaimAmount DECIMAL(18,2),
    NewClaimAmount DECIMAL(18,2),
    ModifiedBy VARCHAR(100) DEFAULT SUSER_SNAME(),
    ModifiedOn DATETIME DEFAULT GETDATE(),
    ActionType VARCHAR(10)
);

-- Trigger on Case table
CREATE TRIGGER trg_Case_Audit
ON [Case]
AFTER UPDATE, INSERT, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- Detect UPDATEs (rows in both inserted and deleted)
    IF EXISTS(SELECT * FROM inserted) AND EXISTS(SELECT * FROM deleted)
    BEGIN
        INSERT INTO CaseAuditLog (CaseId, OldClaimAmount, NewClaimAmount, ActionType)
        SELECT i.Id, d.ClaimAmount, i.ClaimAmount, 'UPDATE'
        FROM inserted i
        INNER JOIN deleted d ON i.Id = d.Id;
    END
    -- Detect INSERTs (only in inserted)
    ELSE IF EXISTS(SELECT * FROM inserted)
    BEGIN
        INSERT INTO CaseAuditLog (CaseId, OldClaimAmount, NewClaimAmount, ActionType)
        SELECT i.Id, NULL, i.ClaimAmount, 'INSERT' FROM inserted i;
    END
END;
```

**Option B: EF Core SaveChanges Override (Application-Level)**:
```csharp
// In ApplicationDBContext.cs
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    var auditEntries = ChangeTracker.Entries()
        .Where(e => e.State == EntityState.Added || e.State == EntityState.Modified)
        .Select(e => new AuditLog
        {
            EntityName = e.Entity.GetType().Name,
            ActionType = e.State.ToString(),
            Username = _currentUserService.Username,
            Timestamp = DateTime.UtcNow,
            ChangedValues = JsonSerializer.Serialize(e.CurrentValues.ToObject())
        }).ToList();

    var result = await base.SaveChangesAsync(cancellationToken);

    if (auditEntries.Any())
    {
        AuditLogs.AddRange(auditEntries);
        await base.SaveChangesAsync(cancellationToken);
    }
    return result;
}
```

---

### Q21: Dapper vs Entity Framework Core — When to use each?

| Feature | EF Core | Dapper |
|---|---|---|
| **Design** | Full ORM — LINQ to SQL, change tracking | Micro-ORM — raw SQL with mapping |
| **Performance** | Good (tracking overhead) | Excellent — near raw ADO.NET speed |
| **SQL Control** | Abstracted — generates SQL | Explicit — you write raw SQL |
| **Use Case** | CRUD, transactional operations | Complex reports, dashboard queries |

**BSK uses BOTH**:
- **EF Core** for claims processing, document ops, status changes
- **Dapper** for `Q000_ExecutiveDashboard` — complex window functions, CTEs, UNION ALL — zero tracking overhead

```csharp
// Dapper usage in BSK (QueryDbHandle)
var result = await _connection.QueryAsync<DashboardDto>(
    "SELECT ... FROM [Case] WITH (NOLOCK) JOIN ...",
    new { StartDate = start, EndDate = end });
```

---

## 11. Quick-Fire SQL Checklist

| Question | Short Answer |
|---|---|
| `WHERE` vs `HAVING` | `WHERE` filters before grouping; `HAVING` filters after GROUP BY |
| `DELETE` vs `TRUNCATE` | `DELETE` is logged, supports WHERE; `TRUNCATE` is faster, no WHERE, resets identity |
| `DROP` vs `TRUNCATE` | `DROP` removes table + structure; `TRUNCATE` removes only data |
| Primary Key vs Unique Key | PK — no NULL, 1 per table; UK — allows 1 NULL, multiple per table |
| `NULL` in comparisons | `NULL = NULL` is FALSE — use `IS NULL` or `IS NOT NULL` |
| `IN` vs `EXISTS` | `EXISTS` is faster when inner query returns many rows |
| `CHAR` vs `VARCHAR` | `CHAR` fixed length, `VARCHAR` variable — use `VARCHAR` |
| `SCOPE_IDENTITY()` | Returns last identity in current scope — safest method |
| Index on boolean? | Low cardinality — rarely useful; SQL may ignore it |
| Heap table | Table WITHOUT a clustered index |

---

## 🏁 BSK Talking Points for SQL Section

> *"In BSK, I worked with SQL Server 2022 extensively. Our executive dashboard uses complex queries with CTEs, ROW_NUMBER() window functions, and UNION ALL for fast metrics aggregation. We use WITH (NOLOCK) hints on read-heavy dashboard queries to prevent blocking write operations on production tables. For ORM, we use EF Core for transactional CRUD and Dapper for performance-critical reporting queries."*

---
*Next: See [Section_03_DotNet_Core_WebAPI.md](./Section_03_DotNet_Core_WebAPI.md)*
