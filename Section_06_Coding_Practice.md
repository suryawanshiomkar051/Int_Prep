# 💻 Section 6: Coding Practice & Interview Master Index
### 🎯 Target: .NET Backend Developer (2.5 Years Exp) | Job Switch Before December

> This file serves as both the **Coding Practice Guide** (algorithms) and the **Master Index** linking all 6 sections.

---

## 📋 Table of Contents

1. [Master Interview Prep Index](#1-master-interview-prep-index)
2. [Strings & Arrays](#2-strings--arrays)
3. [Linked Lists](#3-linked-lists)
4. [LINQ Business Queries](#4-linq-business-queries)
5. [Trees & Graphs](#5-trees--graphs)
6. [Dynamic Programming](#6-dynamic-programming)
7. [Concurrency & Design Patterns](#7-concurrency--design-patterns)
8. [Time & Space Complexity Quick Reference](#8-time--space-complexity-quick-reference)
9. [December Job Switch Plan](#9-december-job-switch-plan)

---

## 1. Master Interview Prep Index

| Section | File | Key Topics | Priority |
|---|---|---|---|
| **1. C# Fundamentals** | `Section_01_CSharp_Fundamentals.md` | OOP, Generics, LINQ, async/await, GC, Records, Pattern Matching, SOLID | 🔴 HIGH |
| **2. SQL & Database** | `Section_02_SQL_Database.md` | Joins, Indexes, CTEs, Window Functions, Normalization, Transactions, ACID | 🔴 HIGH |
| **3. .NET Core & Web API** | `Section_03_DotNet_Core_WebAPI.md` | JWT Auth, Middleware, DI Lifetimes, HttpClient, Polly, Caching, Background Services | 🔴 HIGH |
| **4. EF Core & ORM** | `Section_04_EF_Core_ORM.md` | DbContext, Migrations, Loading Strategies, N+1 Problem, Dapper | 🟡 MEDIUM |
| **5. System Design** | `Section_05_System_Design_Advanced.md` | BSK Architecture, CQRS, Docker, Security, OWASP, Coding Patterns | 🟡 MEDIUM |
| **6. Coding Practice** | `Section_06_Coding_Practice.md` (THIS FILE) | Algorithms, Data Structures, Time Complexity | 🟢 LOW-MED |

---

## 2. Strings & Arrays

### Q1: Reverse a String — O(N), O(1) space using Span<char>
```csharp
public static string Reverse(string input)
{
    if (string.IsNullOrEmpty(input)) return input;
    char[] chars = input.ToCharArray();
    Span<char> span = chars.AsSpan();
    int left = 0, right = span.Length - 1;
    while (left < right)
    {
        (span[left], span[right]) = (span[right], span[left]); // Tuple swap
        left++; right--;
    }
    return new string(span);
}
// Interview tip: string is immutable in C#, so ToCharArray() is needed.
// Span<char> avoids extra allocation. Swap in-place = O(1) extra space.
```

---

### Q2: First Non-Repeated Character — O(N), O(K) space
```csharp
public static char? FindFirstNonRepeated(string input)
{
    var freq = new Dictionary<char, int>();
    foreach (char c in input) freq[c] = freq.GetValueOrDefault(c, 0) + 1;
    foreach (char c in input) if (freq[c] == 1) return c;
    return null;
}
// Tip: Two passes. Pass 1 = count frequency. Pass 2 = find first with count 1.
// Nested loop approach is O(N²) — always use HashMap pattern.
```

---

### Q3: Two Sum — O(N), O(N) space
```csharp
public static int[] TwoSum(int[] nums, int target)
{
    var map = new Dictionary<int, int>(); // value → index
    for (int i = 0; i < nums.Length; i++)
    {
        int complement = target - nums[i];
        if (map.ContainsKey(complement)) return new[] { map[complement], i };
        map[nums[i]] = i;
    }
    throw new ArgumentException("No solution");
}
// Tip: Store "complement" not "current value" logic. Single pass HashMap.
// Brute force O(N²) with nested loops — show HashMap solution in interviews.
```

---

### Q4: Palindrome Check (alphanumeric only) — O(N), O(1)
```csharp
public static bool IsPalindrome(string input)
{
    int left = 0, right = input.Length - 1;
    while (left < right)
    {
        if (!char.IsLetterOrDigit(input[left])) { left++; continue; }
        if (!char.IsLetterOrDigit(input[right])) { right--; continue; }
        if (char.ToLower(input[left]) != char.ToLower(input[right])) return false;
        left++; right--;
    }
    return true;
}
// Tip: Two pointers. Skip non-alphanumeric on-the-fly. No extra string allocation.
```

---

### Q5: Merge Two Sorted Arrays — O(N+M), O(N+M)
```csharp
public static int[] MergeSortedArrays(int[] arr1, int[] arr2)
{
    int[] merged = new int[arr1.Length + arr2.Length];
    int i = 0, j = 0, k = 0;
    while (i < arr1.Length && j < arr2.Length)
        merged[k++] = arr1[i] <= arr2[j] ? arr1[i++] : arr2[j++];
    while (i < arr1.Length) merged[k++] = arr1[i++];
    while (j < arr2.Length) merged[k++] = arr2[j++];
    return merged;
}
// Tip: Two-pointer comparison. Linear time. Don't concatenate + sort (O(N log N)).
```

---

### Q6: Binary Search — O(log N), O(1)
```csharp
public static int BinarySearch(int[] sorted, int target)
{
    int left = 0, right = sorted.Length - 1;
    while (left <= right)
    {
        int mid = left + (right - left) / 2; // NOT (left+right)/2 — avoids overflow!
        if (sorted[mid] == target) return mid;
        if (sorted[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
// Key interview points:
// 1. Use left + (right - left) / 2 to prevent integer overflow
// 2. Condition: left <= right (not left < right, missing single element case)
```

---

### Q7: Kth Largest Element — O(N log K), O(K) using MinHeap
```csharp
public static int FindKthLargest(int[] nums, int k)
{
    var minHeap = new PriorityQueue<int, int>();
    foreach (int num in nums)
    {
        minHeap.Enqueue(num, num);
        if (minHeap.Count > k) minHeap.Dequeue(); // Remove smallest
    }
    return minHeap.Peek(); // Root = Kth largest
}
// Better than sort O(N log N). MinHeap of size K = O(N log K).
// PriorityQueue<T, TPriority> is the .NET 6+ built-in min-heap.
```

---

### Q8: Longest Substring Without Repeating Chars — O(N), Sliding Window
```csharp
public static int LengthOfLongestSubstring(string s)
{
    int maxLen = 0, left = 0;
    var charMap = new Dictionary<char, int>(); // char → latest index
    for (int right = 0; right < s.Length; right++)
    {
        if (charMap.ContainsKey(s[right]))
            left = Math.Max(left, charMap[s[right]] + 1); // Slide window past duplicate
        charMap[s[right]] = right;
        maxLen = Math.Max(maxLen, right - left + 1);
    }
    return maxLen;
}
// Sliding Window pattern: dynamic left-right boundary without nested loops.
```

---

### Q9: Group Anagrams — O(N * K log K)
```csharp
public static IList<IList<string>> GroupAnagrams(string[] strs)
{
    var groupMap = new Dictionary<string, List<string>>();
    foreach (string s in strs)
    {
        char[] chars = s.ToCharArray();
        Array.Sort(chars);
        string key = new string(chars); // Sorted chars = grouping key
        if (!groupMap.ContainsKey(key)) groupMap[key] = new List<string>();
        groupMap[key].Add(s);
    }
    return groupMap.Values.Cast<IList<string>>().ToList();
}
// Sorted string as key: "eat" → "aet", "tea" → "aet" → same group
```

---

## 3. Linked Lists

### Q10: Detect Cycle — Floyd's Algorithm O(N), O(1)
```csharp
public static bool HasCycle(ListNode head)
{
    var slow = head; var fast = head;
    while (fast != null && fast.Next != null)
    {
        slow = slow.Next;       // 1 step
        fast = fast.Next.Next;  // 2 steps
        if (slow == fast) return true; // They meet → cycle!
    }
    return false;
}
// Floyd's Tortoise & Hare: if cycle exists, fast laps slow and they meet.
// Alternative: HashSet of visited nodes — O(N) space (not optimal).
```

---

### Q11: Reverse Linked List — O(N), O(1)
```csharp
public static ListNode ReverseList(ListNode head)
{
    ListNode prev = null, curr = head, next = null;
    while (curr != null)
    {
        next = curr.Next;  // 1. Save next
        curr.Next = prev;  // 2. Reverse pointer
        prev = curr;       // 3. Move prev forward
        curr = next;       // 4. Move curr forward
    }
    return prev; // New head
}
// Critical: save curr.Next BEFORE reversing, otherwise you lose the chain.
```

---

## 4. LINQ Business Queries

### Q12: Real-World LINQ — BSK Claim Analysis
```csharp
public record Case(Guid Id, string ClaimantName, string PolicyType, decimal ClaimAmount, bool IsActive);

// Total active claims by policy type
var totalByPolicy = cases
    .Where(c => c.IsActive && c.ClaimAmount > 10000)
    .GroupBy(c => c.PolicyType)
    .ToDictionary(g => g.Key, g => g.Sum(c => c.ClaimAmount));

// Claimant with highest total claims
var topClaimant = cases
    .GroupBy(c => c.ClaimantName)
    .Select(g => new { Name = g.Key, Total = g.Sum(c => c.ClaimAmount) })
    .OrderByDescending(x => x.Total)
    .Select(x => x.Name)
    .FirstOrDefault();

// LINQ Left Join with DefaultIfEmpty
var caseInvoiceReport = from c in cases
                        join i in invoices on c.Id equals i.CaseId into invoiceGroup
                        from inv in invoiceGroup.DefaultIfEmpty()
                        select new
                        {
                            c.ClaimantName,
                            InvoiceAmount = inv?.Amount ?? 0m
                        };
```

---

### Q13: FizzBuzz — Extensible (Open/Closed Principle)
```csharp
public static List<string> ExtensibleFizzBuzz(int limit)
{
    var rules = new Dictionary<int, string>
    {
        { 3, "Fizz" },
        { 5, "Buzz" },
        { 7, "Jazz" }  // Adding new rule requires NO code change in the loop
    };

    var result = new List<string>();
    for (int i = 1; i <= limit; i++)
    {
        var sb = new StringBuilder();
        foreach (var rule in rules)
            if (i % rule.Key == 0) sb.Append(rule.Value);

        result.Add(sb.Length == 0 ? i.ToString() : sb.ToString());
    }
    return result;
}
// Show Open/Closed Principle: add new divisor rule WITHOUT modifying loop.
```

---

## 5. Trees & Graphs

### Q14: Binary Tree DFS — Inorder, Preorder, Postorder
```csharp
// Inorder: Left → Root → Right (outputs sorted BST values!)
void Inorder(TreeNode node, List<int> result)
{
    if (node == null) return;
    Inorder(node.Left, result);
    result.Add(node.Value);
    Inorder(node.Right, result);
}

// Preorder: Root → Left → Right (copy tree structure)
void Preorder(TreeNode node, List<int> result)
{
    if (node == null) return;
    result.Add(node.Value);
    Preorder(node.Left, result);
    Preorder(node.Right, result);
}

// Postorder: Left → Right → Root (delete tree safely)
void Postorder(TreeNode node, List<int> result)
{
    if (node == null) return;
    Postorder(node.Left, result);
    Postorder(node.Right, result);
    result.Add(node.Value);
}
```

---

### Q15: Level Order BFS — Queue-based
```csharp
public static IList<IList<int>> LevelOrder(TreeNode root)
{
    var result = new List<IList<int>>();
    if (root == null) return result;

    var queue = new Queue<TreeNode>();
    queue.Enqueue(root);

    while (queue.Count > 0)
    {
        int levelSize = queue.Count; // Snapshot current level count!
        var level = new List<int>();
        for (int i = 0; i < levelSize; i++)
        {
            var node = queue.Dequeue();
            level.Add(node.Value);
            if (node.Left != null) queue.Enqueue(node.Left);
            if (node.Right != null) queue.Enqueue(node.Right);
        }
        result.Add(level);
    }
    return result;
}
// Key: Snapshot queue.Count BEFORE the inner loop to process one level at a time.
```

---

### Q16: Validate BST — Min/Max Boundary Approach
```csharp
public static bool IsValidBST(TreeNode root) => Validate(root, null, null);

private static bool Validate(TreeNode node, int? min, int? max)
{
    if (node == null) return true;
    if (min != null && node.Value <= min) return false; // Must be > min
    if (max != null && node.Value >= max) return false; // Must be < max
    return Validate(node.Left, min, node.Value) &&      // Left: max = parent value
           Validate(node.Right, node.Value, max);        // Right: min = parent value
}
// TRAP: Just checking node.Left < node < node.Right is wrong for deep nodes!
// Must propagate min/max boundaries DOWN through recursion.
```

---

### Q17: Maximum Depth of Binary Tree — O(N)
```csharp
public static int MaxDepth(TreeNode root)
{
    if (root == null) return 0;
    return Math.Max(MaxDepth(root.Left), MaxDepth(root.Right)) + 1;
}
// Classic divide-and-conquer. Post-order traversal.
```

---

## 6. Dynamic Programming

### Q18: Fibonacci — O(N), O(1) Iterative (Optimal)
```csharp
// Approach A: Memoization (O(N) space)
private static Dictionary<int, long> memo = new();
public static long FibMemo(int n)
{
    if (n <= 1) return n;
    if (memo.ContainsKey(n)) return memo[n];
    memo[n] = FibMemo(n - 1) + FibMemo(n - 2);
    return memo[n];
}

// Approach B: Iterative (O(1) space — OPTIMAL)
public static long FibIterative(int n)
{
    if (n <= 1) return n;
    long prev2 = 0, prev1 = 1, curr = 0;
    for (int i = 2; i <= n; i++)
    {
        curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return curr;
}
// Naive recursion = O(2^N) exponential. Iterative = O(N) time, O(1) space.
```

---

## 7. Concurrency & Design Patterns

### Q19: Thread-Safe Singleton — Double Check + Lazy<T>
```csharp
// Option A: Double-check locking
public sealed class ThreadSafeSingleton
{
    private static ThreadSafeSingleton _instance = null;
    private static readonly object _lock = new();
    private ThreadSafeSingleton() { }

    public static ThreadSafeSingleton Instance
    {
        get
        {
            if (_instance == null)           // First check (no lock overhead)
            {
                lock (_lock)
                {
                    if (_instance == null)   // Second check (inside lock)
                        _instance = new ThreadSafeSingleton();
                }
            }
            return _instance;
        }
    }
}

// Option B: Lazy<T> — PREFERRED (cleaner, thread-safe by default)
public sealed class LazySingleton
{
    private static readonly Lazy<LazySingleton> _lazy = new(() => new LazySingleton());
    private LazySingleton() { }
    public static LazySingleton Instance => _lazy.Value;
}
```

---

### Q20: Producer-Consumer with System.Threading.Channels
```csharp
public class AsyncJobQueue
{
    private readonly Channel<string> _queue;

    public AsyncJobQueue(int capacity = 100)
    {
        _queue = Channel.CreateBounded<string>(new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = true
        });
    }

    // Producer
    public async ValueTask EnqueueAsync(string jobId) =>
        await _queue.Writer.WriteAsync(jobId);

    // Consumer
    public async Task StartConsumerAsync(CancellationToken ct)
    {
        while (await _queue.Reader.WaitToReadAsync(ct))
        {
            while (_queue.Reader.TryRead(out var job))
            {
                Console.WriteLine($"Processing: {job}");
                await Task.Delay(100, ct); // Simulate work
            }
        }
    }
}
// System.Threading.Channels = modern, lock-free, high-performance queues
// Replaces older BlockingCollection<T> approach
```

---

### Q21: Deep Copy using JSON Serialization
```csharp
public static T DeepCopy<T>(T source)
{
    if (source == null) return default;
    var json = JsonSerializer.Serialize(source);
    return JsonSerializer.Deserialize<T>(json);
}

// Shallow Copy (MemberwiseClone) vs Deep Copy:
// Shallow: copies value types, but reference types point to SAME nested objects
// Deep:    creates completely independent clone — all nested objects are new
```

---

## 8. Time & Space Complexity Quick Reference

| Algorithm | Time | Space | Notes |
|---|---|---|---|
| Binary Search | O(log N) | O(1) | Array must be sorted |
| Linear Search | O(N) | O(1) | Unsorted array |
| Bubble/Selection Sort | O(N²) | O(1) | Never use in production |
| Merge Sort | O(N log N) | O(N) | Stable sort |
| Quick Sort | O(N log N) avg | O(log N) | In-place, unstable |
| HashMap Lookup | O(1) avg | O(N) | Hash collisions degrade to O(N) |
| HashSet Contains | O(1) avg | O(N) | Fast duplicate detection |
| Heap Insert/Delete | O(log N) | O(N) | PriorityQueue<T,P> in .NET |
| DFS Tree | O(N) | O(H) | H = height of tree |
| BFS Tree/Graph | O(N + E) | O(W) | W = max width |
| Two Sum (HashMap) | O(N) | O(N) | vs O(N²) brute force |
| Sliding Window | O(N) | O(K) | K = window size |
| Fibonacci DP | O(N) | O(1) | vs O(2^N) naive recursion |

---

## 9. December Job Switch Plan

### 📅 Week-by-Week Schedule

| Week | Focus | Files to Study |
|---|---|---|
| **Week 1 (Now)** | C# Fundamentals + SQL deep dive | Section_01 + Section_02 |
| **Week 2** | .NET Core + Web API + EF Core | Section_03 + Section_04 |
| **Week 3** | System Design + BSK project walkthroughs | Section_05 |
| **Week 4** | Coding practice + Mock interviews | Section_06 |
| **Week 5-6** | Active applications + practice each day | All sections rotating |
| **Week 7-8** | Company-specific preparation + interviews | BSK project deep dive |
| **December** | Close offers | — |

---

### 📋 Daily Practice Routine (45 min/day)

```
Morning (20 min): Read 2-3 Q&A from current section
Lunch  (10 min):  1 LeetCode Easy/Medium in C#
Evening (15 min): Review previous day's mistakes + BSK STAR stories
```

---

### 🎯 Target Companies for .NET Backend Role (2.5 yr exp)

**Tier 1 (Product Companies)**:
- Microsoft, Thoughtworks, Publicis Sapient, Nagarro

**Tier 2 (Service Companies — Easier to get in)**:
- Wipro, Infosys, Cognizant, TCS (Digital), HCL

**Tier 3 (Startups — Fast growth)**:
- InsurTech startups (BSK experience directly applies!)
- FinTech companies (payment gateway, KYC experience)

---

### 💡 Salary Negotiation Tips for 2.5 yr exp

- Target: **8-15 LPA** (varies by company tier and city)
- Never give exact current CTC — give range or say "as per industry standards"
- Always negotiate — first offer is rarely the best offer
- Factor in: ESOP/stock, health insurance, WFH flexibility

---

### 🗣️ BSK Project Intro (30-second elevator pitch)

> *"I'm currently working on Bima Sevak Kendra (BSK), an insurance claims management platform built with ASP.NET Core 8 Web API and SQL Server. The system handles end-to-end insurance case management — from KYC verification via third-party APIs like Surepass, to payment processing via Razorpay, to document storage on AWS S3. I've worked on features like JWT authentication, Polly-powered resilient vendor integration, background health monitoring services, and performance-optimized dashboard queries using Dapper with complex SQL CTEs and window functions."*

---

### ✅ Top 10 Most Asked Questions for .NET Backend Dev (2.5 yr)

Rank by frequency based on BSK preparation:

1. **DI Lifetimes** — Transient, Scoped, Singleton (with examples)
2. **Async/Await** — Why, thread starvation, Task.WhenAll
3. **JWT Authentication** — How it works, claims, token validation
4. **SOLID Principles** — With code examples
5. **SQL Joins** — INNER vs LEFT vs RIGHT with examples
6. **EF Core Loading** — Eager vs Lazy, N+1 Problem
7. **Middleware vs Action Filter** — Difference and use cases
8. **Repository Pattern** — Why, benefits, interface
9. **Index types** — Clustered vs Non-Clustered, when to use
10. **Exception handling** — Global middleware, try-catch best practices

---

*Good luck with your job switch! 🚀*

*All 6 section files:*
- [Section_01_CSharp_Fundamentals.md](./Section_01_CSharp_Fundamentals.md)
- [Section_02_SQL_Database.md](./Section_02_SQL_Database.md)
- [Section_03_DotNet_Core_WebAPI.md](./Section_03_DotNet_Core_WebAPI.md)
- [Section_04_EF_Core_ORM.md](./Section_04_EF_Core_ORM.md)
- [Section_05_System_Design_Advanced.md](./Section_05_System_Design_Advanced.md)
- [Section_06_Coding_Practice.md](./Section_06_Coding_Practice.md) ← You are here
