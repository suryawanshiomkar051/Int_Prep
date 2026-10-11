# 📦 SQL – Infosys Interview Questions & Answers

---

## Q1. Difference between OLTP and OLAP Database Concepts

| Feature | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|--------|--------------------------------------|--------------------------------------|
| Purpose | Day-to-day transactions | Data analysis & reporting |
| Operations | INSERT, UPDATE, DELETE | SELECT (complex queries) |
| Data | Current, up-to-date | Historical, aggregated |
| Speed | Fast for short queries | Slower, complex queries |
| Design | Normalized (3NF) | Denormalized (Star/Snowflake schema) |
| Users | Clerks, front-end apps | Data analysts, BI tools |
| Example | Banking system, POS | Data warehouses, Power BI |

---

## Q2. Usage of SQL and No-SQL Databases

### SQL (Relational Databases) – MySQL, SQL Server, PostgreSQL
- Structured data with fixed schema
- ACID compliant (Atomicity, Consistency, Isolation, Durability)
- Use when: financial apps, ERP systems, where data integrity is critical
- Supports JOINs, complex queries

### No-SQL Databases – MongoDB, Redis, Cassandra
- Unstructured/semi-structured data
- Flexible schema (JSON, key-value, graph, column-family)
- Use when: real-time data, big data, social media, IoT
- Horizontally scalable

**When to choose SQL:** Relationships between entities, transactional integrity  
**When to choose NoSQL:** High velocity data, flexible schema, massive scale

---

## Q3. Why Do We Need Normalization?

Normalization organizes database to **reduce redundancy** and **improve data integrity**.

### Normal Forms:
- **1NF:** Each column has atomic values, no repeating groups
- **2NF:** 1NF + no partial dependency (non-key columns depend on full PK)
- **3NF:** 2NF + no transitive dependency

### Benefits:
- Eliminates data redundancy
- Prevents update/delete/insert anomalies
- Saves storage space
- Improves query consistency

```sql
-- Unnormalized: Orders(OrderID, CustomerName, CustomerPhone, Product)
-- Normalized: Customers(CustomerID, Name, Phone) + Orders(OrderID, CustomerID, Product)
```

---

## Q4. How to Ensure/Check the Performance of SQL Queries?

1. **Use Execution Plan** (`SET STATISTICS IO ON`, `CTRL+M` in SSMS)
2. **Check for Index Usage** – avoid table scans
3. **Avoid SELECT \*** – select only needed columns
4. **Use proper JOINs** instead of subqueries where possible
5. **Use WHERE clause effectively** – filter early
6. **Avoid functions on indexed columns** in WHERE clause
7. **Use `SET STATISTICS TIME ON`** to see CPU/elapsed time
8. **Check for missing indexes** via DMVs:
   ```sql
   SELECT * FROM sys.dm_db_missing_index_details
   ```
9. **Avoid N+1 queries** – use JOINs or CTEs
10. **Use Query Store** in SQL Server 2016+ to track query performance over time

---

## Q5. Usage of Different Types of JOINs

```sql
-- INNER JOIN: Only matching rows from both tables
SELECT e.Name, d.DeptName
FROM Employees e INNER JOIN Departments d ON e.DeptID = d.DeptID;

-- LEFT JOIN: All rows from left + matching from right
SELECT e.Name, d.DeptName
FROM Employees e LEFT JOIN Departments d ON e.DeptID = d.DeptID;

-- RIGHT JOIN: All rows from right + matching from left
SELECT e.Name, d.DeptName
FROM Employees e RIGHT JOIN Departments d ON e.DeptID = d.DeptID;

-- FULL OUTER JOIN: All rows from both tables
SELECT e.Name, d.DeptName
FROM Employees e FULL OUTER JOIN Departments d ON e.DeptID = d.DeptID;

-- CROSS JOIN: Cartesian product
SELECT e.Name, d.DeptName FROM Employees e CROSS JOIN Departments d;

-- SELF JOIN: Join table with itself
SELECT e1.Name AS Employee, e2.Name AS Manager
FROM Employees e1 JOIN Employees e2 ON e1.ManagerID = e2.EmployeeID;
```

---

## Q6. Usage of Different Types of SQL Functions

### Aggregate Functions
```sql
SELECT COUNT(*), SUM(Salary), AVG(Salary), MAX(Salary), MIN(Salary) FROM Employees;
```

### String Functions
```sql
SELECT LEN('Hello'), UPPER('hello'), LOWER('HELLO'), LTRIM('  hi'), RTRIM('hi  ');
SELECT SUBSTRING('Hello World', 1, 5);  -- 'Hello'
SELECT CHARINDEX('o', 'Hello');         -- 5
SELECT REPLACE('Hello', 'l', 'r');      -- 'Herro'
```

### Date Functions
```sql
SELECT GETDATE(), DATEADD(DAY, 7, GETDATE()), DATEDIFF(DAY, '2026-01-01', GETDATE());
SELECT YEAR(GETDATE()), MONTH(GETDATE()), DAY(GETDATE());
SELECT FORMAT(GETDATE(), 'dd-MM-yyyy');
```

### Window Functions
```sql
SELECT Name, Salary, ROW_NUMBER() OVER (ORDER BY Salary DESC) AS RowNum,
       RANK() OVER (ORDER BY Salary DESC) AS Rank,
       DENSE_RANK() OVER (ORDER BY Salary DESC) AS DenseRank,
       NTILE(4) OVER (ORDER BY Salary) AS Quartile
FROM Employees;
```

### Conversion Functions
```sql
SELECT CAST('123' AS INT), CONVERT(VARCHAR, GETDATE(), 103), ISNULL(NULL, 'Default');
```

---

## Q7. Usage of Indexes – Difference Between Clustered and Non-Clustered Index

### Clustered Index
- **Physically sorts** the table data based on the key
- Only **ONE per table** (data pages are the index)
- Automatically created on Primary Key
- Faster for range queries

### Non-Clustered Index (NC Index)
- Separate structure with **pointer** to actual data row
- Up to **999 per table** in SQL Server
- Contains index key + row locator (RID or clustered key)
- Good for columns used in WHERE, JOIN, ORDER BY

```sql
-- Create Clustered Index
CREATE CLUSTERED INDEX IX_Emp_ID ON Employees(EmployeeID);

-- Create Non-Clustered Index
CREATE NONCLUSTERED INDEX IX_Emp_Name ON Employees(Name);

-- Non-clustered with included columns (covering index)
CREATE NONCLUSTERED INDEX IX_Emp_Dept ON Employees(DeptID) INCLUDE (Name, Salary);
```

| Feature | Clustered | Non-Clustered |
|---------|-----------|---------------|
| Data storage | Sorted physically | Separate structure |
| Count per table | 1 | Up to 999 |
| Speed | Fast for range | Fast for lookups |
| Extra storage | No | Yes |

---

## Q8. How to Check Estimated and Actual Execution Plan?

```sql
-- Estimated Execution Plan (doesn't run query)
-- Press Ctrl+L in SSMS or:
SET SHOWPLAN_XML ON;
SELECT * FROM Employees WHERE DeptID = 5;
SET SHOWPLAN_XML OFF;

-- Actual Execution Plan (runs query, shows real stats)
-- Press Ctrl+M in SSMS or:
SET STATISTICS PROFILE ON;
SELECT * FROM Employees WHERE DeptID = 5;
SET STATISTICS PROFILE OFF;

-- Enable IO and Time statistics
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
```

---

## Q9. How to Read the Execution Plan?

Reading a SQL execution plan (right to left, top to bottom):

1. **Table Scan** – reads entire table (bad, means no index used)
2. **Index Scan** – reads entire index (better than table scan)
3. **Index Seek** – goes directly to relevant rows (best, efficient)
4. **Key Lookup** – found in non-clustered, then fetches remaining from clustered
5. **Nested Loops / Hash Match / Merge Join** – join algorithms
6. **Sort** – expensive operation, indicates missing index
7. **Compute Scalar** – calculates expression

**Key metrics to check:**
- **Cost %** – higher % = expensive operation
- **Estimated vs Actual rows** – large difference = outdated statistics
- **Thick arrows** – large data flow between operations (potential bottleneck)
- **Yellow exclamation** – warning (missing index, implicit conversion)

---

## Q10. What is a Subquery?

A subquery is a query **nested inside another query**.

```sql
-- Subquery in WHERE clause
SELECT Name, Salary FROM Employees
WHERE Salary > (SELECT AVG(Salary) FROM Employees);

-- Correlated Subquery (references outer query)
SELECT Name, Salary FROM Employees e1
WHERE Salary = (SELECT MAX(Salary) FROM Employees e2 WHERE e1.DeptID = e2.DeptID);

-- Subquery in FROM (derived table)
SELECT DeptID, AvgSal FROM
  (SELECT DeptID, AVG(Salary) AS AvgSal FROM Employees GROUP BY DeptID) AS DeptAvg
WHERE AvgSal > 50000;

-- Subquery with IN
SELECT Name FROM Employees
WHERE DeptID IN (SELECT DeptID FROM Departments WHERE Location = 'Mumbai');
```

---

## Q11. What is the Use of Writing the Query in CTE?

**CTE (Common Table Expression)** is a temporary named result set defined within the execution scope of a SELECT, INSERT, UPDATE, or DELETE statement.

```sql
-- Basic CTE
WITH EmployeeCTE AS (
    SELECT EmployeeID, Name, DeptID, Salary,
           ROW_NUMBER() OVER (PARTITION BY DeptID ORDER BY Salary DESC) AS Rank
    FROM Employees
)
SELECT * FROM EmployeeCTE WHERE Rank = 1;  -- Top earner per department

-- Recursive CTE (Organizational Hierarchy)
WITH OrgHierarchy AS (
    -- Anchor member
    SELECT EmployeeID, Name, ManagerID, 0 AS Level
    FROM Employees WHERE ManagerID IS NULL
    UNION ALL
    -- Recursive member
    SELECT e.EmployeeID, e.Name, e.ManagerID, oh.Level + 1
    FROM Employees e
    INNER JOIN OrgHierarchy oh ON e.ManagerID = oh.EmployeeID
)
SELECT * FROM OrgHierarchy;
```

### Benefits of CTE over Subqueries:
- Improves **readability**
- Can be **referenced multiple times**
- Supports **recursion**
- Easier to **debug** step by step

---

## Q12. Scenario: How to Implement Audit Capturing?

**Approach 1: Triggers (most common)**
```sql
-- Audit table
CREATE TABLE AuditLog (
    AuditID INT IDENTITY PRIMARY KEY,
    TableName VARCHAR(100),
    Action VARCHAR(10),        -- INSERT, UPDATE, DELETE
    PrimaryKeyValue INT,
    OldValue NVARCHAR(MAX),   -- JSON
    NewValue NVARCHAR(MAX),   -- JSON
    ChangedBy VARCHAR(100) DEFAULT SUSER_NAME(),
    ChangedAt DATETIME DEFAULT GETDATE()
);

-- Trigger for Employees table
CREATE TRIGGER trg_Employees_Audit
ON Employees
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    IF EXISTS (SELECT 1 FROM inserted) AND EXISTS (SELECT 1 FROM deleted)
    BEGIN
        -- UPDATE
        INSERT INTO AuditLog (TableName, Action, PrimaryKeyValue, OldValue, NewValue)
        SELECT 'Employees', 'UPDATE', i.EmployeeID,
               (SELECT * FROM deleted d WHERE d.EmployeeID = i.EmployeeID FOR JSON PATH),
               (SELECT * FROM inserted i2 WHERE i2.EmployeeID = i.EmployeeID FOR JSON PATH)
        FROM inserted i;
    END
    ELSE IF EXISTS (SELECT 1 FROM inserted)
    BEGIN
        -- INSERT
        INSERT INTO AuditLog (TableName, Action, PrimaryKeyValue, NewValue)
        SELECT 'Employees', 'INSERT', EmployeeID,
               (SELECT * FROM inserted FOR JSON PATH)
        FROM inserted;
    END
    ELSE
    BEGIN
        -- DELETE
        INSERT INTO AuditLog (TableName, Action, PrimaryKeyValue, OldValue)
        SELECT 'Employees', 'DELETE', EmployeeID,
               (SELECT * FROM deleted FOR JSON PATH)
        FROM deleted;
    END
END;
```

**Approach 2: Temporal Tables (SQL Server 2016+)**
```sql
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY,
    Name VARCHAR(100),
    Salary DECIMAL(18,2),
    ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START,
    ValidTo DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.EmployeesHistory));
```

---

## Q13. Scenario: When to Use UNION vs UNION ALL?

### UNION
- Removes **duplicate** rows (performs sort/distinct)
- **Slower** due to overhead of deduplication
- Use when: you want unique records across result sets

### UNION ALL
- Keeps **all rows** including duplicates
- **Faster** (no deduplication overhead)
- Use when: you know there are no duplicates, or duplicates are acceptable

```sql
-- UNION (removes duplicates)
SELECT Name FROM Employees WHERE DeptID = 1
UNION
SELECT Name FROM Employees WHERE Salary > 50000;

-- UNION ALL (keeps duplicates, faster)
SELECT Name FROM Employees WHERE DeptID = 1
UNION ALL
SELECT Name FROM Employees WHERE Salary > 50000;

-- Practical use: combining sales from multiple years
SELECT ProductID, Sales, 'Q1' AS Quarter FROM Sales_Q1
UNION ALL
SELECT ProductID, Sales, 'Q2' AS Quarter FROM Sales_Q2;
```

**Rule:** Always prefer **UNION ALL** unless you specifically need deduplication.

---

## Q14. What is View and Materialized View?

### View
- **Virtual table** – stores the SQL query, not data
- Data is always **fresh** (executes query every time)
- Cannot be indexed directly (except indexed views)

```sql
CREATE VIEW vw_EmployeeDetails AS
SELECT e.Name, d.DeptName, e.Salary
FROM Employees e JOIN Departments d ON e.DeptID = d.DeptID;

SELECT * FROM vw_EmployeeDetails;
```

### Materialized View (Indexed View in SQL Server)
- **Physically stores** the result of the query
- **Pre-computed** data – faster reads
- Needs to be **refreshed** when underlying data changes
- SQL Server calls it **Indexed View**

```sql
CREATE VIEW vw_DeptSalary WITH SCHEMABINDING AS
SELECT DeptID, COUNT_BIG(*) AS EmpCount, SUM(Salary) AS TotalSalary
FROM dbo.Employees GROUP BY DeptID;

-- Create clustered index to make it materialized
CREATE UNIQUE CLUSTERED INDEX IX_vw_DeptSalary ON vw_DeptSalary(DeptID);
```

| Feature | View | Materialized View |
|---------|------|-------------------|
| Data storage | No (query only) | Yes (physical) |
| Performance | Slower | Faster |
| Data freshness | Always current | May be stale |
| SQL Server term | View | Indexed View |

---

## Q15. How Can We Delete Duplicate Rows in a Table?

```sql
-- Method 1: Using CTE with ROW_NUMBER (Most Common)
WITH DuplicateCTE AS (
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY Name, DeptID, Salary  -- columns that define uniqueness
        ORDER BY EmployeeID
    ) AS RowNum
    FROM Employees
)
DELETE FROM DuplicateCTE WHERE RowNum > 1;

-- Method 2: Using MIN/MAX
DELETE FROM Employees
WHERE EmployeeID NOT IN (
    SELECT MIN(EmployeeID)
    FROM Employees
    GROUP BY Name, DeptID, Salary  -- duplicate-defining columns
);

-- Method 3: INTO temp table approach
SELECT DISTINCT * INTO #TempTable FROM Employees;
TRUNCATE TABLE Employees;
INSERT INTO Employees SELECT * FROM #TempTable;
DROP TABLE #TempTable;
```

---

## Q16. How Can We View the Last Record Added to a Table?

```sql
-- Method 1: Using IDENTITY column (most reliable)
SELECT TOP 1 * FROM Employees ORDER BY EmployeeID DESC;

-- Method 2: Using SCOPE_IDENTITY() after insert
INSERT INTO Employees (Name, DeptID) VALUES ('John', 2);
SELECT SCOPE_IDENTITY();  -- Returns the last inserted IDENTITY value

-- Method 3: Using @@IDENTITY
SELECT @@IDENTITY;  -- Returns last identity in current session (includes triggers)

-- Method 4: Using OUTPUT clause
INSERT INTO Employees (Name, DeptID)
OUTPUT INSERTED.*
VALUES ('Jane', 3);

-- Method 5: Using CreatedAt timestamp column (best practice)
SELECT TOP 1 * FROM Employees ORDER BY CreatedAt DESC;
```

---

## Q17. What is USING Clause and Give Example?

The **USING clause** is used in a JOIN when both tables have a column with the **same name** and you want to join on that column. It's a shorthand for `ON t1.col = t2.col`.

> **Note:** `USING` is standard SQL supported in MySQL, PostgreSQL, Oracle. SQL Server uses `ON` syntax instead.

```sql
-- Standard SQL / MySQL / PostgreSQL
SELECT e.Name, d.DeptName
FROM Employees e JOIN Departments d USING (DeptID);

-- Equivalent using ON (works in all databases including SQL Server):
SELECT e.Name, d.DeptName
FROM Employees e JOIN Departments d ON e.DeptID = d.DeptID;

-- USING with multiple columns
SELECT * FROM Orders o JOIN OrderDetails od USING (OrderID, ProductID);
```

### USING vs ON:
| USING | ON |
|-------|----|
| Shorter syntax | Verbose but flexible |
| Columns must have same name | Columns can have different names |
| Not supported in SQL Server | Supported everywhere |

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep*

---

---

# ⭐ BONUS – Important Questions (Added by Interviewer Pattern Analysis)

---

## BQ1. What is a Transaction? Explain ACID Properties.

A **transaction** is a sequence of SQL operations treated as a single unit — either ALL succeed or ALL fail.

```sql
-- Transaction Example: Transfer money between accounts
BEGIN TRANSACTION;

BEGIN TRY
    -- Deduct from sender
    UPDATE Accounts SET Balance = Balance - 5000 WHERE AccountID = 101;
    
    -- Add to receiver
    UPDATE Accounts SET Balance = Balance + 5000 WHERE AccountID = 202;
    
    -- Both succeeded → commit
    COMMIT TRANSACTION;
    PRINT 'Transfer successful';
END TRY
BEGIN CATCH
    -- Something failed → rollback everything
    ROLLBACK TRANSACTION;
    PRINT 'Transfer failed: ' + ERROR_MESSAGE();
END CATCH;
```

### ACID Properties:
| Property | Meaning | Example |
|----------|---------|---------|
| **Atomicity** | All or nothing | Money deducted → must also be credited |
| **Consistency** | DB stays valid | Total money before = total money after |
| **Isolation** | Transactions don't interfere | Two users booking same seat simultaneously |
| **Durability** | Committed data persists | Even after crash, committed data is safe |

```sql
-- Isolation Levels (controls how transactions see each other's data)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;  -- Dirty reads allowed (fastest)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;    -- Default: no dirty reads
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;   -- No dirty/non-repeatable reads
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;      -- Strictest, no phantom reads
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;          -- Row versioning (no locks)

-- SAVEPOINT – partial rollback
BEGIN TRANSACTION;
    INSERT INTO Orders VALUES (1, 'Laptop', 50000);
    SAVE TRANSACTION SavePoint1;
    INSERT INTO Orders VALUES (2, 'Mouse', 500);
    ROLLBACK TRANSACTION SavePoint1;  -- Only undoes the second insert
COMMIT TRANSACTION;
```

---

## BQ2. Difference Between DELETE, TRUNCATE, and DROP

```sql
-- DELETE: Removes specific rows, can be rolled back, fires triggers
BEGIN TRANSACTION;
DELETE FROM Employees WHERE DeptID = 5;
-- Can ROLLBACK here
ROLLBACK;

-- TRUNCATE: Removes ALL rows, faster, resets identity, cannot be filtered
BEGIN TRANSACTION;
TRUNCATE TABLE TempLogs;
-- Can ROLLBACK (inside explicit transaction)
COMMIT;

-- DROP: Removes the entire table (structure + data + indexes)
DROP TABLE TempLogs;  -- Permanent! No rollback in most cases
```

| Feature | DELETE | TRUNCATE | DROP |
|---------|--------|----------|------|
| Removes | Specific rows | All rows | Entire table |
| WHERE clause | ✅ Yes | ❌ No | ❌ No |
| Rollback | ✅ Yes | ✅ (in explicit tx) | ❌ No |
| Triggers fired | ✅ Yes | ❌ No | ❌ No |
| IDENTITY reset | ❌ No | ✅ Yes | ❌ (table gone) |
| Speed | Slower | Faster | Fastest |
| Logged | Fully logged | Minimally logged | DDL logged |

---

## BQ3. Stored Procedure vs Stored Function

```sql
-- STORED PROCEDURE: Can do everything, returns via OUTPUT or result set
CREATE PROCEDURE sp_GetEmployeesByDept
    @DeptID INT,
    @TotalCount INT OUTPUT
AS
BEGIN
    SELECT * FROM Employees WHERE DeptID = @DeptID;
    SET @TotalCount = @@ROWCOUNT;
END;

-- Execute SP
DECLARE @count INT;
EXEC sp_GetEmployeesByDept @DeptID = 2, @TotalCount = @count OUTPUT;
PRINT 'Total: ' + CAST(@count AS VARCHAR);

-- STORED FUNCTION: Returns a value, used inside SELECT/WHERE
CREATE FUNCTION fn_GetFullName(@FirstName VARCHAR(50), @LastName VARCHAR(50))
RETURNS VARCHAR(101)
AS
BEGIN
    RETURN @FirstName + ' ' + @LastName;
END;

-- Use function inside a query (cannot do this with SP)
SELECT dbo.fn_GetFullName(FirstName, LastName) AS FullName FROM Employees;

-- TABLE-VALUED FUNCTION (returns a table)
CREATE FUNCTION fn_GetEmployeesByDept(@DeptID INT)
RETURNS TABLE
AS
RETURN (
    SELECT EmployeeID, Name, Salary FROM Employees WHERE DeptID = @DeptID
);

-- Use like a table
SELECT * FROM dbo.fn_GetEmployeesByDept(3) WHERE Salary > 50000;
```

| Feature | Stored Procedure | Stored Function |
|---------|-----------------|-----------------|
| Return | Multiple result sets / OUTPUT params | Single value or table |
| Use in SELECT | ❌ No | ✅ Yes |
| DML (INSERT/UPDATE) | ✅ Yes | ❌ No (Scalar) |
| Transaction control | ✅ Yes | ❌ No |
| Error handling (TRY/CATCH) | ✅ Yes | ❌ No |
| Can call from function | ❌ No | N/A |

---

## BQ4. What is a Trigger and When to Use It?

A **trigger** is a special stored procedure that automatically executes in response to DML events (INSERT, UPDATE, DELETE).

```sql
-- AFTER Trigger (most common): fires after DML
CREATE TRIGGER trg_AfterInsertEmployee
ON Employees
AFTER INSERT
AS
BEGIN
    DECLARE @empName VARCHAR(100);
    SELECT @empName = Name FROM inserted;  -- 'inserted' = new rows
    
    INSERT INTO ActivityLog (Action, Details, CreatedAt)
    VALUES ('Employee Added', 'New employee: ' + @empName, GETDATE());
END;

-- INSTEAD OF Trigger: replaces the DML operation
CREATE TRIGGER trg_InsteadOfDeleteEmployee
ON Employees
INSTEAD OF DELETE
AS
BEGIN
    -- Soft delete instead of hard delete
    UPDATE Employees SET IsActive = 0, DeletedAt = GETDATE()
    WHERE EmployeeID IN (SELECT EmployeeID FROM deleted);  -- 'deleted' = rows being removed
END;

-- Trigger to prevent salary decrease
CREATE TRIGGER trg_PreventSalaryDecrease
ON Employees
AFTER UPDATE
AS
BEGIN
    IF UPDATE(Salary)  -- Only fires if Salary column was updated
    BEGIN
        IF EXISTS (
            SELECT 1 FROM inserted i JOIN deleted d ON i.EmployeeID = d.EmployeeID
            WHERE i.Salary < d.Salary  -- New salary < old salary
        )
        BEGIN
            RAISERROR('Salary cannot be decreased!', 16, 1);
            ROLLBACK TRANSACTION;
        END
    END
END;

-- Magic tables: inserted & deleted
-- INSERT trigger: 'inserted' has new rows, 'deleted' is empty
-- DELETE trigger: 'deleted' has old rows, 'inserted' is empty
-- UPDATE trigger: 'inserted' has new values, 'deleted' has old values
```

### When to Use Triggers:
- Audit logging (track all changes)
- Enforcing complex business rules
- Maintaining denormalized summary tables
- Soft delete implementation

---

## BQ5. Difference Between WHERE and HAVING

```sql
-- WHERE: Filters ROWS before grouping (works on individual rows)
-- HAVING: Filters GROUPS after GROUP BY (works on aggregated data)

-- Find departments where average salary > 60000
SELECT DeptID, AVG(Salary) AS AvgSalary
FROM Employees
WHERE IsActive = 1             -- ← WHERE filters individual rows FIRST
GROUP BY DeptID
HAVING AVG(Salary) > 60000;   -- ← HAVING filters groups AFTER aggregation

-- WHERE can use column values; HAVING can use aggregate functions
-- Cannot use aggregate in WHERE:
-- ❌ Wrong: WHERE AVG(Salary) > 60000
-- ✅ Right: HAVING AVG(Salary) > 60000

-- Practical example:
SELECT 
    DeptID,
    COUNT(*) AS EmpCount,
    SUM(Salary) AS TotalSalary,
    MAX(Salary) AS MaxSalary
FROM Employees
WHERE HireDate >= '2020-01-01'    -- Only employees hired after 2020
GROUP BY DeptID
HAVING COUNT(*) >= 5              -- Only depts with 5+ such employees
ORDER BY TotalSalary DESC;
```

| Feature | WHERE | HAVING |
|---------|-------|--------|
| When it runs | Before GROUP BY | After GROUP BY |
| Works on | Individual rows | Groups / Aggregates |
| Can use aggregate | ❌ No | ✅ Yes |
| Can filter rows | ✅ Yes | ❌ No (only groups) |

---

## BQ6. Window Functions – LEAD, LAG, FIRST_VALUE, LAST_VALUE

Window functions perform calculations **across a set of rows related to the current row**, without collapsing them into groups.

```sql
-- Sample: Monthly sales data
CREATE TABLE MonthlySales (Month INT, Year INT, Sales DECIMAL(18,2));

-- ROW_NUMBER: Unique sequential number
SELECT Month, Sales,
    ROW_NUMBER() OVER (ORDER BY Month) AS RowNum
FROM MonthlySales;

-- RANK vs DENSE_RANK: Handle ties differently
SELECT Name, Score,
    RANK() OVER (ORDER BY Score DESC) AS Rank,         -- 1,2,2,4  (skips 3)
    DENSE_RANK() OVER (ORDER BY Score DESC) AS DRank   -- 1,2,2,3  (no skip)
FROM Leaderboard;

-- LAG: Access PREVIOUS row value
-- LEAD: Access NEXT row value
SELECT 
    Month,
    Sales,
    LAG(Sales, 1, 0) OVER (ORDER BY Month) AS PrevMonthSales,
    LEAD(Sales, 1, 0) OVER (ORDER BY Month) AS NextMonthSales,
    Sales - LAG(Sales, 1, 0) OVER (ORDER BY Month) AS MoM_Change,
    -- % change
    ROUND(
        (Sales - LAG(Sales, 1) OVER (ORDER BY Month)) / 
        LAG(Sales, 1) OVER (ORDER BY Month) * 100, 2
    ) AS MoM_Pct_Change
FROM MonthlySales;

-- FIRST_VALUE / LAST_VALUE
SELECT 
    Name, DeptID, Salary,
    FIRST_VALUE(Name) OVER (PARTITION BY DeptID ORDER BY Salary DESC) AS TopEarner,
    LAST_VALUE(Name) OVER (
        PARTITION BY DeptID ORDER BY Salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS BottomEarner
FROM Employees;

-- Running total (ROWS BETWEEN)
SELECT Month, Sales,
    SUM(Sales) OVER (ORDER BY Month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RunningTotal,
    AVG(Sales) OVER (ORDER BY Month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS Rolling3MonthAvg
FROM MonthlySales;

-- NTILE: Divide into N buckets
SELECT Name, Salary,
    NTILE(4) OVER (ORDER BY Salary) AS SalaryQuartile  -- Q1, Q2, Q3, Q4
FROM Employees;
```

---

## BQ7. What is a Deadlock and How to Prevent It?

A **deadlock** occurs when two or more transactions are waiting for each other to release locks, creating a circular dependency.

```sql
-- Deadlock scenario:
-- Session 1:
BEGIN TRAN;
UPDATE TableA SET Col1 = 1 WHERE ID = 1;  -- Locks TableA row 1
-- Waiting for TableB row 1...

-- Session 2 (simultaneously):
BEGIN TRAN;
UPDATE TableB SET Col1 = 1 WHERE ID = 1;  -- Locks TableB row 1
-- Waiting for TableA row 1... → DEADLOCK!

-- SQL Server automatically detects and kills the "victim" process

-- Prevention Strategies:
-- 1. Always access tables in the SAME ORDER
-- Session 1 and Session 2 both: TableA first, then TableB

-- 2. Keep transactions SHORT
BEGIN TRAN;
UPDATE Accounts SET Balance -= 100 WHERE ID = 1;
UPDATE Accounts SET Balance += 100 WHERE ID = 2;
COMMIT;  -- Commit immediately, don't hold locks

-- 3. Use NOLOCK hint (allows dirty reads, use carefully)
SELECT * FROM Orders WITH (NOLOCK);

-- 4. Use SNAPSHOT isolation level (no shared locks on read)
ALTER DATABASE MyDB SET ALLOW_SNAPSHOT_ISOLATION ON;
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;

-- 5. Add indexes to reduce lock time
-- The faster the query, the shorter the lock duration

-- Monitor deadlocks
SELECT * FROM sys.dm_exec_requests WHERE blocking_session_id > 0;
```

---

## BQ8. PIVOT and UNPIVOT in SQL

```sql
-- PIVOT: Converts rows to columns
-- Problem: Show sales per product per quarter (rows) → columns

-- Source data: ProductID, Quarter, Sales
-- Output: ProductID | Q1 | Q2 | Q3 | Q4

SELECT ProductID, [Q1], [Q2], [Q3], [Q4]
FROM (
    SELECT ProductID, Quarter, Sales FROM QuarterlySales
) AS SourceData
PIVOT (
    SUM(Sales)
    FOR Quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS PivotTable;

-- UNPIVOT: Converts columns back to rows
SELECT ProductID, Quarter, Sales
FROM (
    SELECT ProductID, Q1, Q2, Q3, Q4 FROM PivotedSales
) AS PivotData
UNPIVOT (
    Sales FOR Quarter IN (Q1, Q2, Q3, Q4)
) AS UnpivotTable;

-- Dynamic PIVOT (when column values are not known in advance)
DECLARE @cols NVARCHAR(MAX), @sql NVARCHAR(MAX);

SELECT @cols = STRING_AGG(QUOTENAME(Quarter), ',') 
FROM (SELECT DISTINCT Quarter FROM QuarterlySales) AS Q;

SET @sql = '
SELECT ProductID, ' + @cols + '
FROM (SELECT ProductID, Quarter, Sales FROM QuarterlySales) AS Src
PIVOT (SUM(Sales) FOR Quarter IN (' + @cols + ')) AS Pvt';

EXEC sp_executesql @sql;
```

---

## BQ9. Temporary Tables vs Table Variables vs CTEs

```sql
-- 1. TEMPORARY TABLE: Stored in tempdb, survives across batches in same session
CREATE TABLE #TempEmployees (
    EmployeeID INT, Name VARCHAR(100), Salary DECIMAL(18,2)
);
INSERT INTO #TempEmployees SELECT * FROM Employees WHERE DeptID = 3;
SELECT * FROM #TempEmployees WHERE Salary > 50000;
DROP TABLE #TempEmployees;  -- Always clean up

-- Global temp table (visible to all sessions)
CREATE TABLE ##GlobalTemp (ID INT, Name VARCHAR(100));

-- 2. TABLE VARIABLE: Stored in memory (small data), limited scope
DECLARE @EmpTable TABLE (
    EmployeeID INT, Name VARCHAR(100)
);
INSERT INTO @EmpTable SELECT EmployeeID, Name FROM Employees WHERE DeptID = 3;
SELECT * FROM @EmpTable;  -- Auto-cleaned when batch ends, no DROP needed

-- 3. CTE: Named temporary result, only lives for ONE SELECT statement
WITH HighEarners AS (
    SELECT * FROM Employees WHERE Salary > 80000
)
SELECT * FROM HighEarners WHERE DeptID = 2;
-- HighEarners doesn't exist after this query

```

| Feature | Temp Table (#) | Table Variable (@) | CTE |
|---------|---------------|-------------------|-----|
| Scope | Session | Batch/procedure | Single statement |
| Storage | TempDB | Memory (usually) | None (virtual) |
| Statistics | ✅ Yes | ❌ Limited | N/A |
| Indexes | ✅ Yes | Limited | N/A |
| Transactions | ✅ Yes | ❌ No | N/A |
| Best for | Large data, joins | Small data | Readability, recursion |

---

## BQ10. Index Covering and Composite Index Best Practices

```sql
-- COMPOSITE INDEX: Index on multiple columns
-- Order matters! Leading column must be in WHERE clause
CREATE NONCLUSTERED INDEX IX_Orders_CustDate 
ON Orders(CustomerID, OrderDate);  -- CustomerID is LEADING column

-- This query USES the index:
SELECT * FROM Orders WHERE CustomerID = 5 AND OrderDate > '2026-01-01';
SELECT * FROM Orders WHERE CustomerID = 5;  -- Uses index (leading column)

-- This query DOES NOT use the index:
SELECT * FROM Orders WHERE OrderDate > '2026-01-01';  -- Missing leading column!

-- COVERING INDEX: Include all columns needed by query (avoid Key Lookup)
CREATE NONCLUSTERED INDEX IX_Orders_Covering
ON Orders(CustomerID, OrderDate)       -- Search columns
INCLUDE (TotalAmount, Status, Notes);  -- Extra columns (no search, just read)

-- Now this query needs NO Key Lookup:
SELECT CustomerID, OrderDate, TotalAmount, Status
FROM Orders WHERE CustomerID = 5;

-- Check index fragmentation
SELECT 
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent AS Fragmentation
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10
ORDER BY Fragmentation DESC;

-- Rebuild vs Reorganize
ALTER INDEX IX_Orders_CustDate ON Orders REBUILD;     -- >30% fragmentation
ALTER INDEX IX_Orders_CustDate ON Orders REORGANIZE;  -- 10-30% fragmentation
```

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep | ⭐ = Extra Important*
