# BSK Enterprise System — Comprehensive SQL & Database Study Guide

Welcome to the **BSK Database & SQL Study Guide**. This document contains exhaustive, enterprise-grade answers to all key database and SQL questions, referencing the actual database architecture and production queries of the **Bima Sevak Kendra (BSK) API** project.

---

## 🗄️ SQL & Database Q&A

### Q1: Difference between OLTP and OLAP Database Concepts
**Question**: What is the difference between OLTP and OLAP? How do they apply to enterprise applications like BSK?
**Answer**:

| Feature | OLTP (Online Transactional Processing) | OLAP (Online Analytical Processing) |
|---|---|---|
| **Primary Purpose** | To manage immediate, day-to-day transaction records (CRUD operations). | To analyze historical data, trends, and compile business intelligence reports. |
| **Data Structure** | Highly normalized (3NF) to prevent redundancy and ensure rapid writes. | Denormalized (Star/Snowflake schema) to optimize complex read queries. |
| **Query Complexity** | Simple, fast queries inserting, updating, or deleting single records. | Complex, long-running queries aggregate millions of rows (aggregates, joins). |
| **Response Time** | Milliseconds. | Seconds to hours depending on data volume. |
| **Storage Engine** | SQL Server (BSK's current engine), PostgreSQL, MySQL. | Snowflake, AWS Redshift, Google BigQuery, SQL Server Analysis Services (SSAS). |

**BSK Application Context**:
*   The primary **BSK API** database is strictly an **OLTP system**. It handles real-time claim registration, KYC uploads, and payment webhooks via SQL Server. Fast response times and transactional integrity (ACID) are vital here.
*   If BSK decides to build an advanced ML claims-fraud model or an annual performance analytics suite, they would export historical BSK database backups into an **OLAP Data Warehouse** (e.g., Snowflake) to run heavy analytical models without locking the production OLTP tables.

---

### Q2: Usage of SQL and NoSQL Databases
**Question**: When do we use relational SQL databases vs. NoSQL databases? How does this apply to BSK?
**Answer**:

**SQL (Relational Databases)**:
*   *Characteristics*: Structured data with strict schemas, relationships enforced via foreign keys, ACID compliance for transactions.
*   *When to use*: When data integrity is paramount, entities have complex, tightly coupled relationships, and multi-table transactions must succeed or fail as a single unit (All-or-Nothing).
*   *BSK Reference*: BSK uses a relational SQL database. Entities like `Case`, `Claimant`, `Invoice`, and `Receipt` are highly structured and bound by business logic. A payment receipt cannot exist without a valid case ID.

**NoSQL (Non-Relational Databases)**:
*   *Characteristics*: Schema-less, key-value, document-based (JSON), column-family, or graph databases.
*   *When to use*: High-throughput write environments, unstructured or rapidly changing data formats, horizontal scaling across multiple servers.
*   *BSK Scenario*: If BSK wants to store raw JSON request/response payloads from the 7 external vendors (Surepass, Razorpay, etc.) for auditing, a **NoSQL Document Database** like **MongoDB** is ideal. Payloads vary significantly between vendors, and MongoDB excels at storing unstructured JSON logs.

---

### Q3: Why Do We Need Normalization?
**Question**: What is Database Normalization? Why do we normalize schemas? Provide a BSK reference.
**Answer**:
**Normalization** is the process of organizing data in a database to reduce data redundancy (duplicate values) and eliminate **Update, Insertion, and Deletion anomalies**.

**Normal Forms Summary**:
1.  **First Normal Form (1NF)**: Eliminate duplicate columns. Ensure every cell contains atomic (indivisible) values.
2.  **Second Normal Form (2NF)**: Must be in 1NF. All non-key attributes must be fully dependent on the primary key (no partial dependencies).
3.  **Third Normal Form (3NF)**: Must be in 2NF. Eliminate transitive dependencies (non-key columns depending on other non-key columns).

**Why We Need It (Using BSK as Reference)**:
If BSK did not normalize, we might store the claimant's name, mobile number, and address directly inside the `Case` table.
*   *Redundancy*: If a claimant has three active insurance cases, their name, mobile, and address are duplicated three times.
*   *Update Anomaly*: If the claimant changes their mobile number, we must update all three case records. If we miss one, the database is in an inconsistent state.
*   *BSK Solution*: BSK separates these concerns into normalized tables:
    `Case` has a foreign key to `Claimant`. `Claimant` references `Person`. `Person` references `Address`. An address is updated in *one* place, and all active cases automatically read the correct data.

---

### Q4: Usage of Different Types of Joins
**Question**: Explain INNER, LEFT, RIGHT, FULL, and CROSS joins, and provide actual join queries from the BSK project.
**Answer**:

```
[INNER JOIN]  ──► Matches rows present in BOTH tables.
[LEFT JOIN]   ──► All rows from left table + matched rows from right table (NULL if no match).
[RIGHT JOIN]  ──► All rows from right table + matched rows from left table (rarely used).
[FULL JOIN]   ──► All rows from BOTH tables (NULL if no match exists on either side).
[CROSS JOIN]  ──► Cartesian product (every row in A joined with every row in B).
```

**Real Examples from BSK Production Queries**:
1.  **INNER JOIN & LEFT JOIN combined** (From `Q000_ExecutiveDashboard.sql`):
    ```sql
    SELECT 
        l.Id, 
        per.Name, 
        CASE 
            WHEN l.SourceTypeId IN (2, 3) AND TRY_CAST(l.SourceId AS UNIQUEIDENTIFIER) IS NOT NULL 
                 THEN ISNULL(pr.Name, o.OrganizationName)
            ELSE ISNULL(l.SourceId, 'Direct') 
        END AS Source
    FROM Lead l WITH (NOLOCK)
    -- INNER JOIN ensures a Lead must have a matching Person record:
    INNER JOIN Person per WITH (NOLOCK) 
        ON l.PersonId = per.Id
    -- LEFT JOIN allows the Lead to load optional Partner details (null if direct lead):
    LEFT JOIN Partner p WITH (NOLOCK) 
        ON TRY_CAST(l.SourceId AS UNIQUEIDENTIFIER) = p.Id
    LEFT JOIN Person pr WITH (NOLOCK) 
        ON p.PersonId = pr.Id
    LEFT JOIN Organization o WITH (NOLOCK) 
        ON p.OrganizationId = o.Id;
    ```
2.  **CROSS JOIN (Implicit via comma syntax)** (From `Q000_ExecutiveDashboard.sql`):
    Used to join the current and previous period KPI summaries to compare trends directly in a single row:
    ```sql
    SELECT curr.TotalLeads, prev.TotalLeads, curr.TotalLeads - prev.TotalLeads AS LeadsDiff
    FROM Metrics curr, Metrics prev -- Cartesian product matching single-row results
    WHERE curr.Period = 'Current' AND prev.Period = 'Previous';
    ```

---

### Q5: Usage of Different Types of SQL Functions
**Question**: What are the main categories of SQL functions? Provide real examples of their usage in BSK database queries.
**Answer**:
SQL functions are categorized into **Aggregate Functions** (operating on multiple rows to return one value) and **Scalar Functions** (operating on a single input row to return a single value).

**BSK Real-World Examples**:
1.  **Aggregate Functions**:
    *   `COUNT()`: Counts rows. E.g., `COUNT(Id) AS [Count]` to bucket lead aging ranges.
    *   `SUM()`: Totals a column. E.g., `SUM(pay.Amount)` in `Q031_TopPerformingPartners.sql` to calculate total partner payouts.
    *   `AVG()`: Calculates average values. E.g., `AVG(DATEDIFF(DAY, CreatedAt, GETDATE()))` to track the average days cases remain open.
2.  **Scalar Functions**:
    *   `COALESCE(val1, val2, ...)`: Returns the first non-NULL value. E.g., `COALESCE(per.Name, org.OrganizationName)` in `Q031_TopPerformingPartners.sql` to fetch a partner's name whether they are registered as an individual person or a corporation.
    *   `ISNULL(val, default)`: Replaces NULL with a default. E.g., `ISNULL(p.PanNo, '') <> ''` to ensure a KYC field is populated.
    *   `FORMAT(date, format)`: Custom string formatting. E.g., `FORMAT(l.CreatedAt, 'dd MMM yyyy') AS AddedOn` to format dates beautifully for front-end dashboards.
    *   `TRY_CAST(val AS type)` / `TRY_CONVERT(type, val)`: Safe data type conversions. Returns NULL on conversion failure instead of crashing the query. E.g., `TRY_CAST(l.SourceId AS UNIQUEIDENTIFIER)`.
    *   `DATEADD(unit, value, date)` / `DATEDIFF(unit, date1, date2)`: Date manipulation. E.g., `DATEADD(MONTH, -1, @StartDate)` to calculate previous month ranges dynamically.

---

### Q6: What is a Subquery?
**Question**: What is a subquery? Contrast correlated vs. non-correlated subqueries with BSK examples.
**Answer**:
A **Subquery** (or nested query) is a query query embedded inside another SQL statement (SELECT, INSERT, UPDATE, or DELETE).

1.  **Non-Correlated Subquery** (Independent):
    The subquery executes once, independently of the outer query. The outer query uses its result.
    *   *BSK Reference (`Q000_ExecutiveDashboard.sql`)*:
        ```sql
        SELECT 
            'Current' AS Period,
            -- Subquery runs independently to calculate count, which is projected as a static column value:
            (SELECT COUNT(Id) FROM Lead WITH (NOLOCK) WHERE CreatedAt BETWEEN @StartDate AND @EndDate) AS TotalLeads
        ```
2.  **Correlated Subquery** (Dependent):
    The subquery references a column from the outer query, meaning it executes once *for every row* processed by the outer query.
    *   *Example (Finding cases that have high-value claims above their average insurance policy type)*:
        ```sql
        SELECT c.Id, c.ClaimantId, c.ClaimAmount, c.PolicyTypeId
        FROM [Case] c
        WHERE c.ClaimAmount > (
            -- Subquery dynamically references the outer query's c.PolicyTypeId
            SELECT AVG(sub.ClaimAmount) 
            FROM [Case] sub 
            WHERE sub.PolicyTypeId = c.PolicyTypeId
        );
        ```

---

### Q7: View vs. Materialized View
**Question**: What is the difference between a View and a Materialized View?
**Answer**:

**Standard View**:
*   A View is a **virtual table** representing the output of a stored SELECT query.
*   It **does not store data physically** on the disk. Instead, every time a query references the View, SQL Server merges the View definition and executes it against the underlying base tables.
*   *Pros*: Always up-to-date, zero storage overhead.
*   *Cons*: Slow if it wraps complex joins or aggregations because they execute repeatedly.

**Materialized View (Called "Indexed View" in SQL Server)**:
*   A Materialized View **physically stores the query results on the disk**.
*   A clustered index is created on the view, binding its schema and persisting the data. When the base tables are updated, SQL Server automatically maintains the view data.
*   *Pros*: Extremely fast read times because the data is pre-computed and stored.
*   *Cons*: Write operations on base tables become slightly slower because SQL Server must update the materialized indices immediately.

---

### Q8: How to Ensure/Check the Performance of SQL Queries
**Question**: What are the top strategies for identifying and optimizing slow SQL queries in a production system?
**Answer**:

1.  **Analyze Execution Plans**: Run the query in SSMS with "Actual Execution Plan" enabled (Ctrl+M). Look for costly operators like **Table Scans** (full database scans due to missing indices) or **Key Lookups** (jumping to clustered index to fetch non-indexed columns).
2.  **Use Database Engine Tuning Advisor**: Input a slow query execution workload, and SQL Server will suggest exactly what indices or statistics to create.
3.  **Implement Dirty Reads with `NOLOCK`**:
    *   *BSK Reference*: BSK uses `WITH (NOLOCK)` on almost all dashboard read operations (e.g., `FROM Lead l WITH (NOLOCK)`). This tells SQL Server to read data without acquiring shared locks. This prevents read queries from blocking incoming API writes (deadlocks) and increases read performance.
4.  **Use Parameterized Queries**: Prevents plan recompilations. The database caches the query execution plan for reuse.
5.  **Avoid Functions on Indexed Columns (SARGability)**:
    *   *Bad*: `WHERE YEAR(CreatedAt) = 2025` (forces index scan because function modifies the column).
    *   *Good*: `WHERE CreatedAt BETWEEN '2025-01-01' AND '2025-12-31'` (utilizes Index Seek).

---

### Q9, Q10 & Q11: Clustered vs. Non-Clustered Indexes
**Question**: Detail Clustered and Non-Clustered indexes. What are the structural differences, limits, and rules in SQL Server?
**Answer**:

An index is a B-Tree structure on the database server used to locate table records quickly without scanning the entire database.

```
[Clustered Index]      ──► Stores the actual physical table data rows at the leaf level.
[Non-Clustered Index]  ──► Stores pointers (bookmarks) back to the clustered index rows.
```

**Key Differences**:

| Feature | Clustered Index | Non-Clustered Index |
|---|---|---|
| **Physical Storage** | Reorganizes the physical order of rows in the table. | Maintained in a separate physical location away from table data. |
| **Leaf Level Content** | Contains the actual data rows. | Contains index keys + pointers (rid or clustered key) to data rows. |
| **Max Limit per Table** | **Exactly 1** (A table can only be physically sorted one way). | **999** in modern SQL Server versions. |
| **Default Creation** | Automatically created when you define a Primary Key. | Manually created to optimize `WHERE`, `JOIN`, or `ORDER BY` fields. |

**Why the limit?**:
*   *Clustered*: Since data rows can only be physically stored in one sorted sequence (e.g., alphabetically or numerically by ID), you are physically limited to **one clustered index**. A table without a clustered index is called a **Heap**.
*   *Non-Clustered*: You can build up to 999 indices because they are simply separate lookup structures (like an index at the back of a textbook). However, creating too many non-clustered indexes will slow down `INSERT`, `UPDATE`, and `DELETE` operations as every index must update dynamically.

---

### Q12 & Q13: Execution Plans in SQL Server
**Question**: How do we view and read Estimated and Actual Execution Plans? What costly operations should we look out for?
**Answer**:

An execution plan visually represents how the SQL Server Query Optimizer executes a query.

**How to check in SSMS (SQL Server Management Studio)**:
*   **Estimated Execution Plan (Ctrl+L)**: Shows the plan SQL Server *intends* to use without executing the query. Fast and safe to run on large production queries.
*   **Actual Execution Plan (Ctrl+M)**: Runs the query, retrieves the data, and displays the exact plan along with actual resource utilization (CPU, I/O).

**How to Read the Plan**:
1.  **Right-to-Left, Top-to-Bottom**: The flow of data begins on the right side with retrievals (Scans/Seeks) and moves left towards the SELECT operator.
2.  **Look at the Operator Cost %**: Each icon has a cost percentage relative to the entire query. Focus optimization on the most expensive icons.
3.  **Key Costly Operators to Identify**:
    *   **Table Scan / Clustered Index Scan**: (Bad) SQL Server is scanning every single row in the table. Fix by creating an index.
    *   **Index Seek**: (Good) SQL Server jumps directly to the matching data rows using a B-Tree index lookup.
    *   **Key Lookup / RID Lookup**: (Costly) SQL Server found the matching keys in a non-clustered index, but had to jump back to the clustered index to retrieve other columns. Fix by creating a **Covering Index** (adding those extra columns using the `INCLUDE` clause).
    *   **Hash Match**: Represents expensive hashing joins, typically occurring when joining large, unindexed tables.

---

### Q14: Common Table Expressions (CTEs)
**Question**: What is a CTE? Why do we use it, and how does BSK leverage it for dashboard reports?
**Answer**:
A **Common Table Expression (CTE)** is a temporary, named result set that exists only within the execution scope of a single SELECT, INSERT, UPDATE, or DELETE statement.

**Key Benefits**:
1.  **Readability**: Breaks massive query calculations into structured, step-by-step blocks (similar to declaring variables in C#).
2.  **Recursive Queries**: Allows a query to reference itself to traverse hierarchical data (e.g., employee-manager charts or organization trees).

**BSK Implementation Reference (`Q000_ExecutiveDashboard.sql`)**:
BSK uses a CTE to isolate and retrieve the latest case status logs first. This keeps the primary aggregation query clean:
```sql
-- Define the CTE named 'LatestStages'
;WITH LatestStages AS (
    SELECT CaseId, LogId, ROW_NUMBER() OVER(PARTITION BY CaseId ORDER BY CreatedAt DESC) as rn
    FROM CaseLog WITH (NOLOCK)
)
-- Use the CTE inside the main dashboard metric query:
SELECT 
    (SELECT COUNT(c.Id) 
     FROM [Case] c WITH (NOLOCK) 
     LEFT JOIN LatestStages ls 
        ON c.Id = ls.CaseId AND ls.rn = 1 -- Retrieve the latest stage partition
     WHERE c.IsCaseReached != 1 AND (ls.LogId IS NULL OR ls.LogId >= 4)
    ) AS OngoingCases
```

---

### Q15: UNION vs. UNION ALL
**Question**: What is the difference between UNION and UNION ALL? When should we choose one over the other?
**Answer**:

*   **`UNION`**: Combines the results of two SELECT statements and **removes duplicate rows**. To do this, SQL Server must sort and compare the entire combined result set on the server.
*   **`UNION ALL`**: Combines the results of two SELECT statements and **includes all duplicates**. No sorting or deduplication occurs.

**Performance & Best Practices**:
*   `UNION ALL` is significantly faster because it bypasses the memory-intensive sorting and duplicate checking phase.
*   **Always use `UNION ALL`** unless you explicitly require duplicate rows to be filtered out.

**BSK Reference (`Q000_ExecutiveDashboard.sql`)**:
BSK uses `UNION ALL` to aggregate metrics quickly across distinct categories:
```sql
SELECT 'Follow-ups Due Today' AS Label, COUNT(Id) AS [Count], 'warning' AS [Type] FROM Activity WITH (NOLOCK) WHERE CAST(ScheduledDate AS DATE) = CAST(GETDATE() AS DATE)
UNION ALL
SELECT 'KYC Overdue (>7d)' AS Label, COUNT(Id) AS [Count], 'danger' AS [Type] FROM [Case] WITH (NOLOCK) WHERE IsCaseReached != 1 AND CreatedAt < DATEADD(DAY, -7, GETDATE())
UNION ALL
SELECT 'High Value Claims (>5L)' AS Label, COUNT(Id) AS [Count], 'primary' AS [Type] FROM [Case] WITH (NOLOCK) WHERE ClaimAmount > 500000;
```

---

### Q16: How to Delete Duplicate Rows in a Table
**Question**: How can we safely delete duplicate rows in a SQL table? Provide a professional query template.
**Answer**:
The industry-standard method to safely delete duplicate records while preserving one original row is using a **CTE** combined with the `ROW_NUMBER()` window function.

**Step-by-step query template**:
```sql
-- Step 1: Wrap duplicate rows in a CTE partition
WITH CTE_Duplicates AS (
    SELECT 
        Id,
        PersonId,
        CreatedAt,
        -- Partition by columns that define a "duplicate" (e.g. same claimant with identical mobile)
        ROW_NUMBER() OVER (
            PARTITION BY PersonId 
            ORDER BY CreatedAt ASC -- Keep the oldest record (rn = 1)
        ) AS RowNumber
    FROM Lead
)
-- Step 2: Delete duplicate records where RowNumber is greater than 1
DELETE FROM CTE_Duplicates
WHERE RowNumber > 1;
```
**How it works**:
*   `PARTITION BY PersonId` groups rows with identical data.
*   `ORDER BY CreatedAt ASC` assigns a sequential index starting from 1, ordering from oldest to newest.
*   The CTE acts as an editable target, letting you safely delete all duplicate rows (`RowNumber > 1`) while keeping the original (`RowNumber = 1`).

---

### Q17: How to View the Last Record Added to a Table
**Question**: What are the best ways to retrieve the last record added to a database table?
**Answer**:

1.  **Using `ORDER BY` with `TOP 1`** (Standard & Safe):
    Reads the index in reverse order to find the latest timestamp or auto-incremented primary key:
    ```sql
    SELECT TOP 1 Id, CreatedAt, Name 
    FROM Person WITH (NOLOCK)
    ORDER BY CreatedAt DESC;
    ```
2.  **Using Identity Functions** (Retrieving database session additions):
    If you just inserted a row inside a stored procedure and need the newly generated identity key immediately:
    *   `SCOPE_IDENTITY()`: Returns the last identity value generated in the current execution scope (safest, most common).
    *   `@@IDENTITY`: Returns the last identity value across any session (unsafe if triggers exist on the target table).
    *   `IDENT_CURRENT('TableName')`: Returns the last identity value generated for a specific table across all sessions.

---

### Q18: What is the USING Clause?
**Question**: What is the `USING` clause in joins, and is it supported in SQL Server?
**Answer**:
The `USING` clause is a shorthand join syntax defined in the ANSI SQL standard. It is used when joining two tables that share identical column names.

*Standard ANSI SQL syntax (Supported in PostgreSQL, MySQL, Oracle)*:
```sql
SELECT Case.Id, Claimant.Name
FROM Case
JOIN Claimant USING (ClaimantId); -- Joins on Case.ClaimantId = Claimant.ClaimantId
```

**⚠️ SQL Server Limitation**:
**Microsoft SQL Server does not support the `USING` clause.** Instead, you must use the standard explicit `ON` clause:
```sql
SELECT c.Id, cl.Name
FROM [Case] c
JOIN Claimant cl ON c.ClaimantId = cl.Id;
```

---

### Q19: Can We Call a Stored Procedure from a Stored Function?
**Question**: Can a user-defined function (UDF) call a stored procedure in SQL Server?
**Answer**:
**No, SQL Server does not allow calling a Stored Procedure from a User-Defined Function (UDF).**

**Technical Reason**:
*   By design, SQL Server Stored Functions must be **deterministic and side-effect free**. They cannot modify the database state.
*   Stored Procedures are considered non-deterministic because they can execute data modification statements (INSERT, UPDATE, DELETE) or change transaction isolation levels.
*   To prevent functions from causing unexpected data side-effects, SQL Server throws a compilation error if a function attempts to execute a stored procedure.

**Exceptions/Workarounds**:
1.  **Extended Stored Procedures**: A function *can* call certain system extended stored procedures (e.g., `xp_cmdshell` - though usually disabled for security).
2.  **Refactor**: Convert the Stored Function into a lightweight Stored Procedure that returns an output parameter or result set.

---

### Q20: Scenario-Based: How to Implement Audit Capturing
**Question**: How would you implement audit capturing in a production database like BSK?
**Answer**:
There are two production-grade patterns for capturing database audit logs (tracking who changed what, and when):

#### Option A: Application-Level Audit Capturing (EF Core Interceptor)
If your system uses Entity Framework Core, you can intercept all database updates inside the application code before they are sent to SQL Server.

1.  **Define the Audit Table**:
    ```csharp
    public class AuditLog
    {
        public Guid Id { get; set; }
        public string EntityName { get; set; } // e.g. "Case"
        public string ActionType { get; set; } // e.g. "INSERT", "UPDATE"
        public string Username { get; set; }
        public DateTime Timestamp { get; set; }
        public string ChangedValues { get; set; } // JSON representation of changes
    }
    ```
2.  **Override `SaveChangesAsync` in `ApplicationDBContext.cs`**:
    ```csharp
    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var auditEntries = new List<AuditEntry>();
        foreach (var entry in ChangeTracker.Entries())
        {
            if (entry.Entity is AuditLog || entry.State == EntityState.Detached || entry.State == EntityState.Unchanged)
                continue;

            var auditEntry = new AuditEntry(entry)
            {
                EntityName = entry.Entity.GetType().Name,
                ActionType = entry.State.ToString(),
                Timestamp = DateTime.UtcNow,
                Username = _currentUserService.Username // Resolve current user from context
            };
            auditEntries.Add(auditEntry);
        }

        var result = await base.SaveChangesAsync(cancellationToken);
        
        if (auditEntries.Any())
        {
            // Save audit logs to DB
            AuditLogs.AddRange(auditEntries.Select(e => e.ToAuditLog()));
            await base.SaveChangesAsync(cancellationToken);
        }
        return result;
    }
    ```

#### Option B: Database-Level Audit Capturing (SQL Server Triggers)
If the database is updated directly via SSMS, legacy procedures, or external microservices, database-level triggers ensure no audit logs are missed.

*Trigger SQL Example (Auditing changes to the `Case` table)*:
```sql
CREATE TABLE CaseAuditLog (
    AuditId INT IDENTITY(1,1) PRIMARY KEY,
    CaseId UNIQUEIDENTIFIER,
    OldClaimAmount DECIMAL(18,2),
    NewClaimAmount DECIMAL(18,2),
    ModifiedBy VARCHAR(100) DEFAULT SUSER_SNAME(),
    ModifiedOn DATETIME DEFAULT GETDATE(),
    ActionType VARCHAR(10)
);
GO

CREATE TRIGGER trg_Case_Audit
ON [Case]
AFTER UPDATE, INSERT, DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Detect Updates
    IF EXISTS(SELECT * FROM inserted) AND EXISTS(SELECT * FROM deleted)
    BEGIN
        INSERT INTO CaseAuditLog (CaseId, OldClaimAmount, NewClaimAmount, ActionType)
        SELECT i.Id, d.ClaimAmount, i.ClaimAmount, 'UPDATE'
        FROM inserted i
        INNER JOIN deleted d ON i.Id = d.Id;
    END
    -- Detect Inserts
    ELSE IF EXISTS(SELECT * FROM inserted)
    BEGIN
        INSERT INTO CaseAuditLog (CaseId, OldClaimAmount, NewClaimAmount, ActionType)
        SELECT i.Id, NULL, i.ClaimAmount, 'INSERT'
        FROM inserted i;
    END
END;
```

---

### 💡 Preparation Tip
Keep this SQL reference alongside the .NET guide. Being able to explain query optimization with examples like `WITH (NOLOCK)` and window CTEs shows strong backend database capabilities.
