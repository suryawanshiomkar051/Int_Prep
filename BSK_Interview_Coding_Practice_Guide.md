# BSK Enterprise System — Coding Interview Practice Guide (C# & Algorithmic Rounds)

Welcome to the **BSK Coding Interview Practice Guide**. This document contains **20 high-frequency programming questions** frequently asked in live coding rounds, technical screens, and coding challenges for C# / .NET backend developer roles.

Every question features an optimized, idiomatic C# solution, complexity analysis, and preparation tips to help you succeed in live coding environments.

---

## 📋 Table of Contents

1. [Strings & Arrays (Q1 – Q8)](#-strings--arrays)
2. [Linked Lists & Data Structures (Q9 – Q10)](#-linked-lists--data-structures)
3. [LINQ, Collections & Logic (Q11 – Q16)](#-linq-collections--logic)
4. [Concurrency, Design Patterns & Advanced (Q17 – Q20)](#-concurrency-design-patterns--advanced)

---

## 🔤 Strings & Arrays

### Q1: Reverse a String (In-Place & Allocation-Free)
**Problem**: Write a function that reverses a string in-place. If allocations are a concern, optimize it using modern .NET primitives.

**Optimized C# Solution (using `Span<char>`)**:
```csharp
public static class StringAlgorithms
{
    public static string Reverse(string input)
    {
        if (string.IsNullOrEmpty(input)) return input;

        // Create a Span pointing to the stack-allocated string buffer
        // This avoids copying or creating temporary character arrays
        char[] charArray = input.ToCharArray();
        Span<char> charSpan = charArray.AsSpan();

        int left = 0;
        int right = charSpan.Length - 1;

        while (left < right)
        {
            // Swap values using tuple deconstruction
            (charSpan[left], charSpan[right]) = (charSpan[right], charSpan[left]);
            left++;
            right--;
        }

        return new string(charSpan);
    }
}
```
*   **Time Complexity**: O(N) — Passes through half of the string length.
*   **Space Complexity**: O(1) Extra Space — Swaps occur inline within the allocated character buffer.
*   *Interview Tip*: Standard `string` is immutable in C#. Reversing a string directly by concatenation (`+=`) inside a loop allocates O(N²) memory. Swapping inside a `Span` or `char[]` shows intermediate-to-advanced C# memory optimization knowledge.

---

### Q2: First Non-Repeated Character in a String
**Problem**: Find the first character in a string that does not repeat. If all characters repeat, return a null indicator.

**C# Solution (O(N) HashMap Approach)**:
```csharp
public static char? FindFirstNonRepeatedChar(string input)
{
    if (string.IsNullOrEmpty(input)) return null;

    // Dictionary to hold character frequency
    var charCount = new Dictionary<char, int>();

    // Pass 1: Build frequency map
    foreach (char c in input)
    {
        charCount[c] = charCount.GetValueOrDefault(c, 0) + 1;
    }

    // Pass 2: Find the first character with frequency = 1
    foreach (char c in input)
    {
        if (charCount[c] == 1) return c;
    }

    return null;
}
```
*   **Time Complexity**: O(N) — Requires two linear passes.
*   **Space Complexity**: O(K) — Where K is the number of unique characters (at most O(1) if the alphabet size is constant, e.g., 256 for ASCII).
*   *Interview Tip*: Do not use nested loops (O(N²)) to check duplicates. The HashMap/Dictionary pattern solves duplicate counting challenges in linear time.

---

### Q3: Two Sum (Find Index Pair Matching Target Sum)
**Problem**: Given an array of integers `nums` and an integer `target`, return indices of the two numbers such that they add up to `target`. Assume each input has exactly one solution.

**C# Solution (Optimal O(N) HashMap)**:
```csharp
public static int[] TwoSum(int[] nums, int target)
{
    // Dictionary maps: Value ➔ Index
    var numMap = new Dictionary<int, int>();

    for (int i = 0; i < nums.Length; i++)
    {
        int complement = target - nums[i];

        // If complement already exists, we found our pair
        if (numMap.ContainsKey(complement))
        {
            return new int[] { numMap[complement], i };
        }

        // Add current value and index to hash map
        numMap[nums[i]] = i;
    }

    throw new ArgumentException("No two sum solution exists.");
}
```
*   **Time Complexity**: O(N) — Scans the array exactly once.
*   **Space Complexity**: O(N) — Stores elements inside the dictionary.
*   *Interview Tip*: The brute-force solution uses two nested loops, taking O(N²) time. Storing the "complement" dynamically inside a Hash Map reduces the lookup speed to O(1) per element, bringing the total time down to linear.

---

### Q4: Palindrome Check (Ignoring Case and Special Characters)
**Problem**: Determine if a string is a palindrome, considering only alphanumeric characters and ignoring casing.

**C# Solution (Two-Pointer Method)**:
```csharp
public static bool IsPalindrome(string input)
{
    if (string.IsNullOrEmpty(input)) return true;

    int left = 0;
    int right = input.Length - 1;

    while (left < right)
    {
        // Skip non-alphanumeric characters on the left
        if (!char.IsLetterOrDigit(input[left]))
        {
            left++;
        }
        // Skip non-alphanumeric characters on the right
        else if (!char.IsLetterOrDigit(input[right]))
        {
            right--;
        }
        else
        {
            // Compare lowercase representations
            if (char.ToLower(input[left]) != char.ToLower(input[right]))
            {
                return false;
            }
            left++;
            right--;
        }
    }

    return true;
}
```
*   **Time Complexity**: O(N) — Traverses the string once.
*   **Space Complexity**: O(1) — Requires zero extra memory allocations.
*   *Interview Tip*: Avoid modifying or duplicating the entire string to strip special characters first (e.g., regex calls), which allocates unnecessary memory heap objects. The two-pointer approach checks values on the fly without allocations.

---

### Q5: Merge Two Sorted Arrays
**Problem**: Given two sorted integer arrays `arr1` and `arr2`, merge them into a third sorted array.

**C# Solution (O(N+M) Two-Pointer approach)**:
```csharp
public static int[] MergeSortedArrays(int[] arr1, int[] arr2)
{
    int[] merged = new int[arr1.Length + arr2.Length];
    
    int i = 0, j = 0, k = 0;

    // Traverse both arrays simultaneously
    while (i < arr1.Length && j < arr2.Length)
    {
        if (arr1[i] <= arr2[j])
        {
            merged[k++] = arr1[i++];
        }
        else
        {
            merged[k++] = arr2[j++];
        }
    }

    // Copy remaining elements from arr1 if any
    while (i < arr1.Length)
    {
        merged[k++] = arr1[i++];
    }

    // Copy remaining elements from arr2 if any
    while (j < arr2.Length)
    {
        merged[k++] = arr2[j++];
    }

    return merged;
}
```
*   **Time Complexity**: O(N + M) — Where N and M are the sizes of the two input arrays.
*   **Space Complexity**: O(N + M) — To store the final merged array.
*   *Interview Tip*: Do not concatenate the arrays and call `Array.Sort()`, which has a time complexity of O((N+M) log(N+M)). The two-pointer comparison technique runs in linear time.

---

### Q6: Binary Search (Iterative & Recursive)
**Problem**: Implement binary search to find the index of a target element in a sorted array. If it doesn't exist, return -1.

**C# Solution**:
```csharp
public static class BinarySearcher
{
    // Approach A: Iterative (Highly preferred for memory stability)
    public static int SearchIterative(int[] sortedArray, int target)
    {
        int left = 0;
        int right = sortedArray.Length - 1;

        while (left <= right)
        {
            // Avoid overflow: (left + right) / 2 can overflow int boundaries
            int mid = left + (right - left) / 2;

            if (sortedArray[mid] == target) return mid;

            if (sortedArray[mid] < target)
            {
                left = mid + 1;
            }
            else
            {
                right = mid - 1;
            }
        }

        return -1;
    }

    // Approach B: Recursive
    public static int SearchRecursive(int[] array, int target, int left, int right)
    {
        if (left > right) return -1;

        int mid = left + (right - left) / 2;

        if (array[mid] == target) return mid;

        if (array[mid] > target)
        {
            return SearchRecursive(array, target, left, mid - 1);
        }
        
        return SearchRecursive(array, target, mid + 1, right);
    }
}
```
*   **Time Complexity**: O(log N) — Halves the search space on each step.
*   **Space Complexity**:
    *   Iterative: O(1) — Constant memory.
    *   Recursive: O(log N) — Execution call stack frames.
*   *Interview Tip*: When calculating the midpoint, writing `int mid = left + (right - left) / 2;` instead of `(left + right) / 2` displays awareness of integer boundary overflow errors, which is a major positive point.

---

### Q7: Find the Kth Largest Element in an Array
**Problem**: Find the Kth largest element in an unsorted array. E.g. given `[3,2,1,5,6,4]` and `k = 2`, return `5`.

**C# Solution (using native .NET 6+ `PriorityQueue`)**:
```csharp
public static int FindKthLargest(int[] nums, int k)
{
    // Max-Heap or Min-Heap lookup.
    // We use a Min-Heap (PriorityQueue in C# tracks lower values with higher priority)
    // We maintain a heap of size K
    var minHeap = new PriorityQueue<int, int>();

    foreach (int num in nums)
    {
        // Enqueue: value and priority are both 'num'
        minHeap.Enqueue(num, num);

        // If size exceeds K, dequeue the smallest element
        if (minHeap.Count > k)
        {
            minHeap.Dequeue();
        }
    }

    // The root of the Min-Heap contains the Kth largest element
    return minHeap.Peek();
}
```
*   **Time Complexity**: O(N log K) — Enqueue and dequeue operations on a heap of size K take log K time.
*   **Space Complexity**: O(K) — Stores at most K elements inside the heap.
*   *Interview Tip*: Standard sorting takes O(N log N). Utilizing a **Min-Heap** of size K is the optimal solution because it reduces the time complexity and keeps the memory footprint bound to K elements rather than the whole array.

---

### Q8: Group Anagrams
**Problem**: Given an array of strings, group the anagrams together (words containing identical characters in different sequences).

**C# Solution (Sorting Key Approach)**:
```csharp
public static IList<IList<string>> GroupAnagrams(string[] strs)
{
    // Map: Sorted character key ➔ List of matching anagrams
    var groupMap = new Dictionary<string, List<string>>();

    foreach (string s in strs)
    {
        // Sort string characters to form the unique grouping key
        char[] chars = s.ToCharArray();
        Array.Sort(chars);
        string sortedKey = new string(chars);

        if (!groupMap.ContainsKey(sortedKey))
        {
            groupMap[sortedKey] = new List<string>();
        }

        groupMap[sortedKey].Add(s);
    }

    return groupMap.Values.Cast<IList<string>>().ToList();
}
```
*   **Time Complexity**: O(N * K log K) — Where N is the number of strings and K is the maximum length of a string (due to sorting characters).
*   **Space Complexity**: O(N * K) — To store groupings inside the dictionary.
*   *Interview Tip*: An alternative is counting character frequencies (e.g. `a1b2c1`) as keys to avoid the `K log K` sorting cost, resulting in a time complexity of O(N * K).

---

## 🔗 Linked Lists & Data Structures

### Q9: Detect a Cycle in a Single Linked List
**Problem**: Determine if a linked list contains a cycle (loop).

**C# Solution (Floyd's Tortoise and Hare Algorithm)**:
```csharp
public class ListNode
{
    public int Value { get; set; }
    public ListNode Next { get; set; }
    public ListNode(int val) => Value = val;
}

public static class LinkedListAlgorithms
{
    public static bool HasCycle(ListNode head)
    {
        if (head == null || head.Next == null) return false;

        ListNode slow = head; // Moves 1 step
        ListNode fast = head; // Moves 2 steps

        while (fast != null && fast.Next != null)
        {
            slow = slow.Next;
            fast = fast.Next.Next;

            // If they meet, a cycle exists
            if (slow == fast) return true;
        }

        return false;
    }
}
```
*   **Time Complexity**: O(N) — Traversing the nodes.
*   **Space Complexity**: O(1) — No extra data structures are allocated.
*   *Interview Tip*: Explaining **Floyd's Cycle Detection** (using slow and fast pointers) is the expected optimal solution in FAANG-style interviews. Using a HashSet to track visited nodes also works but carries an O(N) space complexity.

---

### Q10: Reverse a Single Linked List (Iterative)
**Problem**: Reverse a singly linked list in-place and return the new head node.

**C# Solution**:
```csharp
public static ListNode ReverseLinkedList(ListNode head)
{
    ListNode previous = null;
    ListNode current = head;
    ListNode next = null;

    while (current != null)
    {
        next = current.Next;     // 1. Temporarily store next node
        current.Next = previous; // 2. Reverse pointer direction (actual swap)
        previous = current;      // 3. Move previous pointer forward
        current = next;          // 4. Move current pointer forward
    }

    return previous; // New head of reversed list
}
```
*   **Time Complexity**: O(N) — Single pass over the list.
*   **Space Complexity**: O(1) — Constant memory space.
*   *Interview Tip*: Be careful to store the `current.Next` node *before* rewriting it, otherwise you will break the list link chain, preventing you from traversing to downstream elements.

---

## 🔁 LINQ, Collections & Logic

### Q11: Real-World LINQ Business Queries
**Problem**: Given a list of BSK `Case` objects, write LINQ queries to:
1.  Filter cases with Claim Amount > 10,000, active status, grouped by `PolicyType`, and sum the total claim value.
2.  Find the claimant with the highest total combined claim amount.

**C# Solution**:
```csharp
public record Case(Guid Id, string ClaimantName, string PolicyType, decimal ClaimAmount, bool IsActive);

public static class BskReportGenerator
{
    // Query 1: Filter, Group, and Aggregate
    public static Dictionary<string, decimal> GetTotalActiveClaimsByPolicyType(List<Case> cases)
    {
        return cases
            .Where(c => c.IsActive && c.ClaimAmount > 10000)
            .GroupBy(c => c.PolicyType)
            .ToDictionary(
                group => group.Key,
                group => group.Sum(c => c.ClaimAmount)
            );
    }

    // Query 2: Find Maximum aggregated claimant
    public static string GetClaimantWithHighestTotalClaims(List<Case> cases)
    {
        return cases
            .GroupBy(c => c.ClaimantName)
            .Select(g => new { Name = g.Key, Total = g.Sum(c => c.ClaimAmount) })
            .OrderByDescending(x => x.Total)
            .Select(x => x.Name)
            .FirstOrDefault(); // Returns null if collection is empty
    }
}
```
*   **Time Complexity**: O(N log N) — Due to ordering calculations (optimized to O(N) if utilizing single-pass tracking helpers).
*   **Space Complexity**: O(N) — To store collection transformations.
*   *Interview Tip*: In backend C# interviews, writing fluent method-syntax LINQ queries (`.Where().GroupBy()`) is preferred as it is the standard style in modern production repositories.

---

### Q12: Fibonacci Sequence (DP Memoization vs. Iterative O(1))
**Problem**: Write a method to return the Nth Fibonacci number. Compare recursively call methods with optimized options.

**C# Solution**:
```csharp
public static class Fibonacci
{
    // Approach A: Memoization (Top-down Dynamic Programming) - O(N) Space
    private static readonly Dictionary<int, long> Memo = new();

    public static long GetFibonacciMemoized(int n)
    {
        if (n <= 1) return n;

        if (Memo.ContainsKey(n)) return Memo[n];

        Memo[n] = GetFibonacciMemoized(n - 1) + GetFibonacciMemoized(n - 2);
        return Memo[n];
    }

    // Approach B: Iterative (Optimal O(1) Space)
    public static long GetFibonacciIterative(int n)
    {
        if (n <= 1) return n;

        long prev2 = 0;
        long prev1 = 1;
        long current = 0;

        for (int i = 2; i <= n; i++)
        {
            current = prev1 + prev2;
            prev2 = prev1;
            prev1 = current;
        }

        return current;
    }
}
```
*   **Time Complexity**: O(N) — Avoids exponential calculation trees.
*   **Space Complexity**:
    *   Memoization: O(N) — Call stack + dictionary capacity.
    *   Iterative: O(1) — Keeps only three active variables in memory.
*   *Interview Tip*: Standard recursion `Fib(n-1) + Fib(n-2)` without memoization runs in exponential **O(2^N)** time due to duplicated branch calculations. Converting it to an iterative O(1) space loop shows strong algorithm optimization capabilities.

---

### Q13: Balanced Parentheses Check
**Problem**: Given a string containing brackets `(`, `)`, `[`, `]`, `{`, and `}`, check if the brackets are closed in the correct order.

**C# Solution (using Stack)**:
```csharp
public static bool IsBalancedBrackets(string input)
{
    var stack = new Stack<char>();

    var bracketMap = new Dictionary<char, char>
    {
        { ')', '(' },
        { ']', '[' },
        { '}', '{' }
    };

    foreach (char c in input)
    {
        // If it's an opening bracket, push to stack
        if (bracketMap.ContainsValue(c))
        {
            stack.Push(c);
        }
        // If it's a closing bracket
        else if (bracketMap.ContainsKey(c))
        {
            // If stack is empty or top doesn't match matching opening bracket, fail
            if (stack.Count == 0 || stack.Pop() != bracketMap[c])
            {
                return false;
            }
        }
    }

    // Return true if all opening brackets were popped and matched
    return stack.Count == 0;
}
```
*   **Time Complexity**: O(N) — Single pass.
*   **Space Complexity**: O(N) — Stack storage in the worst-case scenario.
*   *Interview Tip*: Stacks are the standard data structure choice for nested or hierarchical matching patterns (like brackets, HTML parser tags, or compiler expressions).

---

### Q14: Longest Substring Without Repeating Characters
**Problem**: Find the length of the longest substring without repeating characters. E.g., given `"abcabcbb"`, the answer is `"abc"`, with a length of `3`.

**C# Solution (Sliding Window)**:
```csharp
public static int LengthOfLongestSubstring(string s)
{
    if (string.IsNullOrEmpty(s)) return 0;

    int maxLength = 0;
    int left = 0;
    
    // Hash map to store character and its latest index
    var charMap = new Dictionary<char, int>();

    for (int right = 0; right < s.Length; right++)
    {
        char currentChar = s[right];

        // If duplicate character is found, shift the left boundary of window
        if (charMap.ContainsKey(currentChar))
        {
            left = Math.Max(left, charMap[currentChar] + 1);
        }

        // Update latest position of character
        charMap[currentChar] = right;

        // Calculate window size
        maxLength = Math.Max(maxLength, right - left + 1);
    }

    return maxLength;
}
```
*   **Time Complexity**: O(N) — Linear scan over the string.
*   **Space Complexity**: O(K) — Size of unique character set.
*   *Interview Tip*: The Sliding Window technique is the standard pattern for subarray or substring search challenges where you need to track boundaries dynamically without nested loops.

---

### Q15: Extensible FizzBuzz (No If-Else Bloat)
**Problem**: Write a program that prints numbers 1 to 100. Print "Fizz" for multiples of 3, "Buzz" for multiples of 5, and "FizzBuzz" for multiples of both. Make it extensible so developers can easily add new rules.

**Extensible C# Solution (Dictionary Mapping)**:
```csharp
public static class ExtensibleFizzBuzz
{
    public static List<string> GenerateList(int limit)
    {
        var result = new List<string>();

        // Dynamically add new rules without altering loop code
        var rules = new Dictionary<int, string>
        {
            { 3, "Fizz" },
            { 5, "Buzz" },
            { 7, "Jazz" } // Extensibility test
        };

        for (int i = 1; i <= limit; i++)
        {
            var output = new StringBuilder();

            foreach (var rule in rules)
            {
                if (i % rule.Key == 0)
                {
                    output.Append(rule.Value);
                }
            }

            // Fallback to number if no rules matched
            if (output.Length == 0)
            {
                output.Append(i.ToString());
            }

            result.Add(output.ToString());
        }

        return result;
    }
}
```
*   **Time Complexity**: O(N * R) — Where N is 100 and R is the number of mapping rules.
*   **Space Complexity**: O(N) — Result array.
*   *Interview Tip*: Basic implementations have hardcoded `if (i % 15 == 0) ... else if (i % 3 == 0)`. Presenting an **extensible** design using dynamic dictionaries showcases adherence to the **Open/Closed Principle (SOLID)**, which technical interviewers appreciate.

---

### Q16: Intersection of Two Arrays
**Problem**: Write a function that returns an array containing the common elements present in both input arrays. Ensure each element in the result is unique.

**C# Solution (using Set)**:
```csharp
public static int[] Intersection(int[] nums1, int[] nums2)
{
    var set1 = new HashSet<int>(nums1);
    var intersection = new HashSet<int>();

    foreach (int num in nums2)
    {
        if (set1.Contains(num))
        {
            intersection.Add(num);
        }
    }

    return intersection.ToArray();
}
```
*   **Time Complexity**: O(N + M) — To populate the set and check values.
*   **Space Complexity**: O(N) — To store set items.
*   *Interview Tip*: HashSets provide O(1) constant time checks for item existence (`Contains`), making them ideal for set operations like intersections and differences.

---

## 🔒 Concurrency, Design Patterns & Advanced

### Q17: Thread-Safe Singleton Pattern
**Problem**: Implement the Singleton pattern in C# ensuring it is fully thread-safe and utilizes lazy initialization.

**Optimal C# Solution (Double-Check Locking & Lazy Initialization)**:
```csharp
public sealed class ThreadSafeSingleton
{
    private static ThreadSafeSingleton _instance = null;
    private static readonly object _padlock = new();

    // Private constructor prevents external instantiation
    private ThreadSafeSingleton() { }

    public static ThreadSafeSingleton Instance
    {
        get
        {
            // First check (Avoids lock overhead if instance is already created)
            if (_instance == null)
            {
                lock (_padlock)
                {
                    // Second check (Ensures only one thread can instantiate)
                    if (_instance == null)
                    {
                        _instance = new ThreadSafeSingleton();
                    }
                }
            }
            return _instance;
        }
    }
}

// Alternative (Even cleaner using .NET Lazy<T> wrapper):
public sealed class LazySingleton
{
    private static readonly Lazy<LazySingleton> _lazy = 
        new(() => new LazySingleton());

    private LazySingleton() { }

    public static LazySingleton Instance => _lazy.Value;
}
```
*   **Time Complexity**: O(1) — Instance retrieval.
*   **Space Complexity**: O(1) — A single shared object.
*   *Interview Tip*: Explaining the **Double-Check Locking** pattern (checking null before and inside the lock block) is a classic question. Mentioning `.NET's` native `Lazy<T>` wrapper shows modern, high-level C# framework capabilities.

---

### Q18: Asynchronous Custom Task Queue (Producer-Consumer)
**Problem**: Implement a thread-safe asynchronous queue where multiple publisher threads can enqueue tasks and a background consumer thread processes them sequentially.

**C# Solution (using `System.Threading.Channels`)**:
```csharp
public class AsyncJobQueue
{
    // System.Threading.Channels are high-performance lock-free queues introduced in modern .NET
    private readonly Channel<string> _queue;

    public AsyncJobQueue(int capacity = 100)
    {
        var options = new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait, // Wait if queue is full
            SingleReader = true // Optimized if we have only one consumer
        };
        _queue = Channel.CreateBounded<string>(options);
    }

    // Producer writes to Channel
    public async ValueTask PublishJobAsync(string jobId)
    {
        await _queue.Writer.WriteAsync(jobId);
    }

    // Consumer reads from Channel asynchronously
    public async Task StartConsumerAsync(CancellationToken cancellationToken)
    {
        while (await _queue.Reader.WaitToReadAsync(cancellationToken))
        {
            while (_queue.Reader.TryRead(out var job))
            {
                // Process job sequentially
                Console.WriteLine($"Processing Job: {job} on Thread: {Environment.CurrentManagedThreadId}");
                await Task.Delay(500, cancellationToken); // Simulating work
            }
        }
    }
}
```
*   **Time Complexity**: O(1) — Constant enqueue/dequeue times.
*   **Space Complexity**: O(C) — Bound by the capacity buffer of the Channel.
*   *Interview Tip*: While older implementations used `BlockingCollection<T>`, `System.Threading.Channels` are the modern, standard choice in .NET Core for building low-allocation, high-performance Producer-Consumer queues.

---

### Q19: Subsets / Power Set (Backtracking & Recursion)
**Problem**: Given an array of unique integers, return all possible subsets (the Power Set).

**C# Solution (Recursive Backtracking)**:
```csharp
public static class SubsetsGenerator
{
    public static IList<IList<int>> GenerateSubsets(int[] nums)
    {
        var result = new List<IList<int>>();
        Backtrack(result, new List<int>(), nums, 0);
        return result;
    }

    private static void Backtrack(IList<IList<int>> result, List<int> currentList, int[] nums, int startIndex)
    {
        // Add a copy of current list to result
        result.Add(new List<int>(currentList));

        for (int i = startIndex; i < nums.Length; i++)
        {
            // 1. Choose: Add element to current path
            currentList.Add(nums[i]);

            // 2. Explore: Move down recursion branch
            Backtrack(result, currentList, nums, i + 1);

            // 3. Un-choose: Backtrack by removing element
            currentList.RemoveAt(currentList.Count - 1);
        }
    }
}
```
*   **Time Complexity**: O(N * 2^N) — There are 2^N total subsets, and copying each list to results takes O(N) time.
*   **Space Complexity**: O(N) — Depth of the recursion call stack.
*   *Interview Tip*: Backtracking is a standard algorithm pattern for generating combinations, permutations, or resolving pathfinding algorithms. Remember the three phases: **Choose ➔ Explore ➔ Backtrack (Un-choose)**.

---

### Q20: Deep Copy of an Object
**Problem**: Write a function that creates a complete, independent Deep Copy of an object in C#. Contrast it with a Shallow Copy.

**C# Solution (using Serialization)**:
```csharp
public static class Cloner
{
    // Deep Copy using JSON Serialization
    // It serializes the object to a string, then deserializes it to create a completely new reference tree
    public static T DeepCopy<T>(T source)
    {
        if (source == null) return default;

        var json = JsonSerializer.Serialize(source);
        return JsonSerializer.Deserialize<T>(json);
    }
}
```
*   **Difference**:
    *   **Shallow Copy (`MemberwiseClone()`)**: Copies only the value type properties. For reference type properties, it copies *only* the memory address pointer. Therefore, both the original and copied objects will still share the same nested reference objects.
    *   **Deep Copy**: Creates a completely independent clone, duplicating both the parent object and all deeply nested reference objects in memory.
*   *Interview Tip*: Using JSON serialization is the easiest, safest way to implement deep copying in .NET without writing complex reflection scripts to traverse cyclic reference trees manually.

---

## 🌳 Data Structures & Algorithms (Trees, Graphs, Trie, Heap & DP)

### Q21: Binary Tree DFS Traversal (Inorder, Preorder, Postorder)
**Problem**: Implement Depth-First Search (DFS) traversals for a binary tree: Inorder, Preorder, and Postorder.

**C# Solution (Recursive Approaches)**:
```csharp
public class TreeNode
{
    public int Value { get; set; }
    public TreeNode Left { get; set; }
    public TreeNode Right { get; set; }
    public TreeNode(int val) => Value = val;
}

public static class BinaryTreeDFS
{
    // 1. Inorder (Left ➔ Root ➔ Right)
    public static void Inorder(TreeNode root, List<int> result)
    {
        if (root == null) return;
        Inorder(root.Left, result);
        result.Add(root.Value);
        Inorder(root.Right, result);
    }

    // 2. Preorder (Root ➔ Left ➔ Right)
    public static void Preorder(TreeNode root, List<int> result)
    {
        if (root == null) return;
        result.Add(root.Value);
        Preorder(root.Left, result);
        Preorder(root.Right, result);
    }

    // 3. Postorder (Left ➔ Right ➔ Root)
    public static void Postorder(TreeNode root, List<int> result)
    {
        if (root == null) return;
        Postorder(root.Left, result);
        Postorder(root.Right, result);
        result.Add(root.Value);
    }
}
```
*   **Time Complexity**: O(N) — Visits every node exactly once.
*   **Space Complexity**: O(H) — Height of the tree (execution stack frame size). O(N) in the worst-case for a skewed tree.
*   *Interview Tip*: Standard traversals are basic building blocks. Inorder traversal of a **Binary Search Tree (BST)** outputs values in **sorted ascending order**, which is a highly common interview question.

---

### Q22: Validate a Binary Search Tree (BST)
**Problem**: Given a binary tree, determine if it is a valid Binary Search Tree (BST).

**C# Solution (Min/Max Boundary Constraint Approach)**:
```csharp
public static class BstValidator
{
    public static bool IsValidBST(TreeNode root)
    {
        return Validate(root, null, null);
    }

    private static bool Validate(TreeNode node, int? min, int? max)
    {
        if (node == null) return true;

        // Node value must respect boundaries from parent levels
        if ((min != null && node.Value <= min) || (max != null && node.Value >= max))
        {
            return false;
        }

        // Left child must be less than current node value
        // Right child must be greater than current node value
        return Validate(node.Left, min, node.Value) && 
               Validate(node.Right, node.Value, max);
    }
}
```
*   **Time Complexity**: O(N) — Scans every node.
*   **Space Complexity**: O(H) — Recursion stack size.
*   *Interview Tip*: Simply checking `node.Left.Value < node.Value < node.Right.Value` locally is a trap. That fails for trees where a left descendant is larger than a grandparent. You must propagate dynamic Min/Max boundaries down the recursion stack.

---

### Q23: Binary Tree BFS Traversal (Level Order Traversal)
**Problem**: Implement Breadth-First Search (BFS) / Level Order Traversal of a binary tree, returning node values grouped level-by-level.

**C# Solution (using Queue)**:
```csharp
public static IList<IList<int>> LevelOrder(TreeNode root)
{
    var result = new List<IList<int>>();
    if (root == null) return result;

    var queue = new Queue<TreeNode>();
    queue.Enqueue(root);

    while (queue.Count > 0)
    {
        int levelSize = queue.Count; // Number of elements at current level
        var currentLevel = new List<int>();

        for (int i = 0; i < levelSize; i++)
        {
            TreeNode node = queue.Dequeue();
            currentLevel.Add(node.Value);

            // Enqueue left and right children for the next level iteration
            if (node.Left != null) queue.Enqueue(node.Left);
            if (node.Right != null) queue.Enqueue(node.Right);
        }

        result.Add(currentLevel);
    }

    return result;
}
```
*   **Time Complexity**: O(N) — Visits every node.
*   **Space Complexity**: O(W) — Maximum width of the tree (maximum number of elements at any level stored in the queue).
*   *Interview Tip*: The queue data structure is the standard mechanism to implement Breadth-First Search in trees and graphs. Always freeze the `queue.Count` at the start of the loop to process nodes level-by-level.

---

### Q24: Maximum Depth (Height) of a Binary Tree
**Problem**: Find the maximum depth (number of nodes along the longest path from the root node down to the farthest leaf node) of a binary tree.

**C# Solution (Recursive Divide and Conquer)**:
```csharp
public static int MaxDepth(TreeNode root)
{
    if (root == null) return 0;

    int leftDepth = MaxDepth(root.Left);
    int rightDepth = MaxDepth(root.Right);

    // Height of current node is 1 plus the height of the deeper subtree
    return Math.Max(leftDepth, rightDepth) + 1;
}
```
*   **Time Complexity**: O(N) — Traverses every node once.
*   **Space Complexity**: O(H) — Height of the tree.
*   *Interview Tip*: This is a classic recursive tree problem that showcases understanding of the **Divide and Conquer** design pattern.

---

### Q25: Invert a Binary Tree (Mirror Image)
**Problem**: Invert a binary tree (flip it so all left children become right children and vice versa).

**C# Solution (Recursive DFS)**:
```csharp
public static TreeNode InvertTree(TreeNode root)
{
    if (root == null) return null;

    // 1. Temporarily store subtrees
    TreeNode leftSubtree = InvertTree(root.Left);
    TreeNode rightSubtree = InvertTree(root.Right);

    // 2. Swap left and right children
    root.Left = rightSubtree;
    root.Right = leftSubtree;

    return root;
}
```
*   **Time Complexity**: O(N) — Visits every node once.
*   **Space Complexity**: O(H) — Call stack size.
*   *Interview Tip*: Mirroring trees is a highly common interview question that verifies node link manipulation. It is easily solved using post-order tree recursion.

---

### Q26: Graph Representation and Traversals (BFS & DFS)
**Problem**: Represent an undirected graph and implement basic Breadth-First Search (BFS) and Depth-First Search (DFS) traversals using an Adjacency List.

**C# Solution**:
```csharp
public class Graph
{
    // Adjacency List representation
    private readonly Dictionary<int, List<int>> _adjacencyList = new();

    public void AddVertex(int vertex)
    {
        if (!_adjacencyList.ContainsKey(vertex))
        {
            _adjacencyList[vertex] = new List<int>();
        }
    }

    public void AddEdge(int source, int destination)
    {
        AddVertex(source);
        AddVertex(destination);
        _adjacencyList[source].Add(destination);
        _adjacencyList[destination].Add(source); // Undirected graph
    }

    // 1. BFS (Queue-based iterative search)
    public List<int> TraverseBFS(int startVertex)
    {
        var visited = new HashSet<int>();
        var result = new List<int>();
        var queue = new Queue<int>();

        queue.Enqueue(startVertex);
        visited.Add(startVertex);

        while (queue.Count > 0)
        {
            int vertex = queue.Dequeue();
            result.Add(vertex);

            foreach (int neighbor in _adjacencyList[vertex])
            {
                if (!visited.Contains(neighbor))
                {
                    visited.Add(neighbor);
                    queue.Enqueue(neighbor);
                }
            }
        }
        return result;
    }

    // 2. DFS Wrapper
    public List<int> TraverseDFS(int startVertex)
    {
        var visited = new HashSet<int>();
        var result = new List<int>();
        DfsHelper(startVertex, visited, result);
        return result;
    }

    private void DfsHelper(int vertex, HashSet<int> visited, List<int> result)
    {
        visited.Add(vertex);
        result.Add(vertex);

        foreach (int neighbor in _adjacencyList[vertex])
        {
            if (!visited.Contains(neighbor))
            {
                DfsHelper(neighbor, visited, result);
            }
        }
    }
}
```
*   **Time Complexity**: O(V + E) — Where V is the number of Vertices and E is the number of Edges.
*   **Space Complexity**: O(V) — To track visited states.
*   *Interview Tip*: Always maintain a `visited` HashSet when traversing graphs to prevent falling into infinite loops due to graph cycles.

---

### Q27: Number of Islands (2D Grid Navigation / DFS)
**Problem**: Given an `m x n` 2D binary grid `grid` representing a map of '1's (land) and '0's (water), return the number of islands. An island is surrounded by water and is formed by connecting adjacent lands horizontally or vertically.

**C# Solution (Flood Fill using DFS)**:
```csharp
public static class IslandsCounter
{
    public static int NumIslands(char[][] grid)
    {
        if (grid == null || grid.Length == 0) return 0;

        int islandCount = 0;
        int rows = grid.Length;
        int cols = grid[0].Length;

        for (int r = 0; r < rows; r++)
        {
            for (int c = 0; c < cols; c++)
            {
                if (grid[r][c] == '1')
                {
                    islandCount++;
                    // Trigger DFS to sink the entire island (turn adjacent '1's to '0's)
                    DfsSink(grid, r, c);
                }
            }
        }

        return islandCount;
    }

    private static void DfsSink(char[][] grid, int r, int c)
    {
        int rows = grid.Length;
        int cols = grid[0].Length;

        // Boundary safety check and water skip
        if (r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] == '0')
        {
            return;
        }

        // Sink current cell
        grid[r][c] = '0';

        // Recurse in all 4 cardinal directions
        DfsSink(grid, r + 1, c); // Down
        DfsSink(grid, r - 1, c); // Up
        DfsSink(grid, r, c + 1); // Right
        DfsSink(grid, r, c - 1); // Left
    }
}
```
*   **Time Complexity**: O(M * N) — Where M and N are rows and columns respectively.
*   **Space Complexity**: O(M * N) — Recursion stack depth on massive single island grids.
*   *Interview Tip*: The "sinking" trick (rewriting visited '1's to '0's directly inside the input grid) is an elegant way to avoid allocating a separate `visited[][]` boolean array, saving O(M * N) auxiliary memory.

---

### Q28: Implement a Trie (Prefix Tree)
**Problem**: Implement a Trie (Prefix Tree) containing `Insert`, `Search`, and `StartsWith` methods.

**C# Solution**:
```csharp
public class TrieNode
{
    public Dictionary<char, TrieNode> Children { get; } = new();
    public bool IsEndOfWord { get; set; } = false;
}

public class Trie
{
    private readonly TrieNode _root = new();

    // Inserts a word into the trie
    public void Insert(string word)
    {
        TrieNode current = _root;
        foreach (char c in word)
        {
            if (!current.Children.ContainsKey(c))
            {
                current.Children[c] = new TrieNode();
            }
            current = current.Children[c];
        }
        current.IsEndOfWord = true;
    }

    // Returns true if the word is in the trie
    public bool Search(string word)
    {
        TrieNode node = GetNode(word);
        return node != null && node.IsEndOfWord;
    }

    // Returns true if there is any word in the trie that starts with the given prefix
    public bool StartsWith(string prefix)
    {
        return GetNode(prefix) != null;
    }

    private TrieNode GetNode(string prefix)
    {
        TrieNode current = _root;
        foreach (char c in prefix)
        {
            if (!current.Children.ContainsKey(c)) return null;
            current = current.Children[c];
        }
        return current;
    }
}
```
*   **Time Complexity**:
    *   `Insert`: O(L) — Where L is the word length.
    *   `Search`: O(L)
    *   `StartsWith`: O(L)
*   **Space Complexity**: O(N * L) — Worst case to store words.
*   *Interview Tip*: Tries are the foundation of autocomplete suggestions, dictionary lookups, and routing tables because search speed depends on word length rather than the number of words stored (O(L) vs O(log N)).

---

### Q29: Merge K Sorted Lists
**Problem**: Merge `k` sorted linked lists and return it as one sorted list.

**C# Solution (using PriorityQueue / Min-Heap)**:
```csharp
public static ListNode MergeKLists(ListNode[] lists)
{
    if (lists == null || lists.Length == 0) return null;

    // Priority Queue holds node values, priority is node.Value
    var minHeap = new PriorityQueue<ListNode, int>();

    // 1. Enqueue the head node of each non-empty list
    foreach (var list in lists)
    {
        if (list != null)
        {
            minHeap.Enqueue(list, list.Value);
        }
    }

    ListNode dummy = new ListNode(0);
    ListNode tail = dummy;

    // 2. Dequeue minimum and enqueue next node of the dequeued list
    while (minHeap.Count > 0)
    {
        ListNode node = minHeap.Dequeue();
        tail.Next = node;
        tail = tail.Next;

        // If this list has subsequent elements, add the next node to the heap
        if (node.Next != null)
        {
            minHeap.Enqueue(node.Next, node.Next.Value);
        }
    }

    return dummy.Next;
}
```
*   **Time Complexity**: O(N log K) — Where N is the total number of nodes and K is the number of linked lists.
*   **Space Complexity**: O(K) — Maximum elements in the PriorityQueue.
*   *Interview Tip*: This is a classic hard-level algorithm question. Presenting the **Min-Heap** solution using C# `PriorityQueue` demonstrates excellent tool knowledge and optimization capacity.

---

### Q30: Lowest Common Ancestor (LCA) in a Binary Tree
**Problem**: Find the lowest common ancestor (LCA) of two given nodes `p` and `q` in a binary tree.

**C# Solution (DFS Recursion)**:
```csharp
public static TreeNode LowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q)
{
    // Base Case
    if (root == null || root == p || root == q) return root;

    // Look for nodes in left and right subtrees
    TreeNode left = LowestCommonAncestor(root.Left, p, q);
    TreeNode right = LowestCommonAncestor(root.Right, p, q);

    // If both left and right return non-null, current node is the LCA
    if (left != null && right != null) return root;

    // Otherwise, return the non-null subtree result
    return left ?? right;
}
```
*   **Time Complexity**: O(N) — Scans the tree.
*   **Space Complexity**: O(H) — Height of call stack.
*   *Interview Tip*: This elegant recursion returns a node if it is either `p` or `q`. If a node receives values from both left and right, it is the splitting ancestor (LCA).

---

### Q31: Custom Min-Heap Implementation
**Problem**: Implement a basic custom Min-Heap data structure from scratch.

**C# Solution (Array-based Binary Heap)**:
```csharp
public class MinHeap
{
    private readonly List<int> _elements = new();

    public int Count => _elements.Count;

    public void Insert(int value)
    {
        _elements.Add(value);
        HeapifyUp(_elements.Count - 1);
    }

    public int ExtractMin()
    {
        if (_elements.Count == 0) throw new InvalidOperationException("Heap is empty");

        int min = _elements[0];
        int lastIndex = _elements.Count - 1;

        _elements[0] = _elements[lastIndex];
        _elements.RemoveAt(lastIndex);

        if (_elements.Count > 0)
        {
            HeapifyDown(0);
        }

        return min;
    }

    private void HeapifyUp(int index)
    {
        while (index > 0)
        {
            int parentIndex = (index - 1) / 2;
            if (_elements[index] >= _elements[parentIndex]) break;

            Swap(index, parentIndex);
            index = parentIndex;
        }
    }

    private void HeapifyDown(int index)
    {
        int lastIndex = _elements.Count - 1;
        while (true)
        {
            int leftChild = 2 * index + 1;
            int rightChild = 2 * index + 2;
            int smallest = index;

            if (leftChild <= lastIndex && _elements[leftChild] < _elements[smallest])
            {
                smallest = leftChild;
            }

            if (rightChild <= lastIndex && _elements[rightChild] < _elements[smallest])
            {
                smallest = rightChild;
            }

            if (smallest == index) break;

            Swap(index, smallest);
            index = smallest;
        }
    }

    private void Swap(int i, int j)
    {
        (_elements[i], _elements[j]) = (_elements[j], _elements[i]);
    }
}
```
*   **Time Complexity**:
    *   `Insert`: O(log N)
    *   `ExtractMin`: O(log N)
*   **Space Complexity**: O(N) — Array elements.
*   *Interview Tip*: Writing custom heap operations demonstrates full structural understanding of parent-child math arrays (`Parent = (i-1)/2`, `Left = 2i+1`, `Right = 2i+2`).

---

### Q32: Coin Change (Dynamic Programming)
**Problem**: Given an integer array `coins` representing coins of different denominations and an integer `amount`, return the fewest number of coins needed to make up that amount. If amount cannot be met, return -1.

**C# Solution (Bottom-Up 1D DP)**:
```csharp
public static int CoinChange(int[] coins, int amount)
{
    // dp[i] will store the minimum coins needed for amount i
    int[] dp = new int[amount + 1];
    
    // Fill array with a dummy high value (amount + 1 is mathematically impossible to reach)
    Array.Fill(dp, amount + 1);
    
    // Base Case
    dp[0] = 0;

    for (int i = 1; i <= amount; i++)
    {
        foreach (int coin in coins)
        {
            if (i - coin >= 0)
            {
                // Core State Transition: 1 + min coins for (amount - coin)
                dp[i] = Math.Min(dp[i], 1 + dp[i - coin]);
            }
        }
    }

    // If target amount remains at dummy high value, it means it's unreachable
    return dp[amount] > amount ? -1 : dp[amount];
}
```
*   **Time Complexity**: O(A * C) — Where A is the target amount and C is the number of coin denominations.
*   **Space Complexity**: O(A) — To store the 1D DP array table.
*   *Interview Tip*: This is a classic knapsack-style **Dynamic Programming** question. Presenting the O(A) bottom-up tabulation approach is a massive validation point for senior developer algorithmic interviews.

---

### 💡 Practice & Preparation Advice
During live coding rounds:
1.  **Talk Out Loud**: Explain your logic and thought process before typing a single line of code.
2.  **Verify Constraints**: Clarify input boundaries (Can values be negative? Are duplicates allowed? Is the array sorted?).
3.  **Discuss Trade-offs**: Explain the space/time complexity of your proposed approach and discuss how you could optimize it (e.g. converting nested loops to HashMap lookups).

