# 💻 Infosys Technical Interview – Top C# Live Coding Challenges

> **Must-Practice for Infosys Whiteboard / Screen-Sharing Round!**  
> In almost every Infosys technical round for .NET / C# developers, the interviewer asks candidates to write live code on Notepad, coderpad, or Visual Studio.  
> Below are the top 15 most frequently asked live coding questions with full working code, explanations, and edge-case handling.

---

## Table of Contents
1. [Reverse a String / Words in a Sentence](#1-reverse-a-string--words-in-a-sentence)
2. [Check Palindrome (String & Number)](#2-check-palindrome-string--number)
3. [Find Second Highest Number in an Array](#3-find-second-highest-number-in-an-array)
4. [Count Character Frequencies in a String](#4-count-character-frequencies-in-a-string)
5. [Remove Duplicates from an Array / String](#5-remove-duplicates-from-an-array--string)
6. [Two Sum Problem (Find Pair with Given Sum)](#6-two-sum-problem-find-pair-with-given-sum)
7. [Check if Two Strings are Anagrams](#7-check-if-two-strings-are-anagrams)
8. [Balanced Parentheses / Brackets Check](#8-balanced-parentheses--brackets-check)
9. [Fibonacci Series (Iterative vs Recursive)](#9-fibonacci-series-iterative-vs-recursive)
10. [Check if a Number is Prime](#10-check-if-a-number-is-prime)
11. [Swap Two Numbers Without Temporary Variable](#11-swap-two-numbers-without-temporary-variable)
12. [Find the Missing Number in an Array (1 to N)](#12-find-the-missing-number-in-an-array-1-to-n)
13. [FizzBuzz Problem](#13-fizzbuzz-problem)
14. [Merge Two Sorted Arrays](#14-merge-two-sorted-arrays)
15. [Top 5 LINQ Practical Coding Snippets](#15-top-5-linq-practical-coding-snippets)

---

## 1. Reverse a String / Words in a Sentence

### Question:
1. Reverse a string without using built-in `Array.Reverse()`
2. Reverse the words in a sentence (e.g., `"Infosys Interview Preparation"` → `"Preparation Interview Infosys"`)

### C# Solution:
```csharp
using System;
using System.Text;

public class StringReversal
{
    // Part A: Reverse a string in-place using two pointers
    public static string ReverseString(string input)
    {
        if (string.IsNullOrEmpty(input)) return input;

        char[] chars = input.ToCharArray();
        int left = 0;
        int right = chars.Length - 1;

        while (left < right)
        {
            // Swap characters
            char temp = chars[left];
            chars[left] = chars[right];
            chars[right] = temp;

            left++;
            right--;
        }

        return new string(chars);
    }

    // Part B: Reverse words in a sentence
    public static string ReverseWords(string sentence)
    {
        if (string.IsNullOrWhiteSpace(sentence)) return sentence;

        string[] words = sentence.Split(' ', StringSplitOptions.RemoveEmptyEntries);
        var sb = new StringBuilder();

        for (int i = words.Length - 1; i >= 0; i--)
        {
            sb.Append(words[i]);
            if (i > 0) sb.Append(" ");
        }

        return sb.ToString();
    }
}

// Example Run:
// ReverseString("Infosys") => "sysofnI"
// ReverseWords("I love C# programming") => "programming C# love I"
```

**Key Points to Tell Interviewer:**
- Time Complexity: $O(N)$
- Space Complexity: $O(N)$ for character array / StringBuilder.
- Using two pointers (`left` and `right`) avoids excessive string allocation.

---

## 2. Check Palindrome (String & Number)

### Question:
Determine whether a given string or number reads the same forwards and backwards (e.g., `"madam"`, `121`).

### C# Solution:
```csharp
public class PalindromeChecker
{
    // String Palindrome (case-insensitive, ignoring non-alphanumeric)
    public static bool IsPalindrome(string s)
    {
        if (string.IsNullOrEmpty(s)) return true;

        int left = 0;
        int right = s.Length - 1;

        while (left < right)
        {
            // Skip non-alphanumeric characters
            while (left < right && !char.IsLetterOrDigit(s[left])) left++;
            while (left < right && !char.IsLetterOrDigit(s[right])) right--;

            if (char.ToLower(s[left]) != char.ToLower(s[right]))
                return false;

            left++;
            right--;
        }

        return true;
    }

    // Integer Palindrome (without converting to string!)
    public static bool IsNumberPalindrome(int x)
    {
        // Negative numbers or numbers ending in 0 (except 0 itself) cannot be palindrome
        if (x < 0 || (x % 10 == 0 && x != 0)) return false;

        int reversed = 0;
        int original = x;

        while (x > 0)
        {
            int digit = x % 10;
            reversed = (reversed * 10) + digit;
            x /= 10;
        }

        return original == reversed;
    }
}

// Example Run:
// IsPalindrome("A man, a plan, a canal: Panama") => true
// IsNumberPalindrome(12321) => true
// IsNumberPalindrome(-121) => false
```

---

## 3. Find Second Highest Number in an Array

### Question:
Find the 2nd largest number in an unsorted integer array in a single pass ($O(N)$) without sorting.

### C# Solution:
```csharp
using System;
using System.Linq;

public class SecondHighest
{
    // Single-pass O(N) approach - BEST FOR INTERVIEWS
    public static int FindSecondHighest(int[] numbers)
    {
        if (numbers == null || numbers.Length < 2)
            throw new ArgumentException("Array must contain at least 2 numbers.");

        int highest = int.MinValue;
        int secondHighest = int.MinValue;

        foreach (int num in numbers)
        {
            if (num > highest)
            {
                secondHighest = highest;
                highest = num;
            }
            else if (num > secondHighest && num != highest)
            {
                secondHighest = num;
            }
        }

        if (secondHighest == int.MinValue)
            throw new InvalidOperationException("No distinct second highest found (all elements may be identical).");

        return secondHighest;
    }

    // LINQ approach (interviewer may ask for LINQ 1-liner):
    public static int FindSecondHighestLinq(int[] numbers)
    {
        return numbers
            .Distinct()
            .OrderByDescending(x => x)
            .Skip(1)
            .FirstOrDefault();
    }
}

// Example Run:
// FindSecondHighest(new int[] { 10, 45, 23, 89, 74, 89 }) => 74
```

---

## 4. Count Character Frequencies in a String

### Question:
Count occurrences of each character in a given string and print them.

### C# Solution:
```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class CharacterFrequency
{
    // Method 1: Using Dictionary<char, int>
    public static Dictionary<char, int> GetFrequencies(string input)
    {
        var freqMap = new Dictionary<char, int>();

        if (string.IsNullOrEmpty(input)) return freqMap;

        foreach (char c in input)
        {
            if (char.IsWhiteSpace(c)) continue; // skip spaces if needed

            if (freqMap.ContainsKey(c))
                freqMap[c]++;
            else
                freqMap[c] = 1;
        }

        return freqMap;
    }

    // Method 2: Using LINQ GroupBy
    public static void PrintFrequenciesLinq(string input)
    {
        var query = input
            .Where(c => !char.IsWhiteSpace(c))
            .GroupBy(c => c)
            .Select(g => new { Char = g.Key, Count = g.Count() })
            .OrderByDescending(x => x.Count);

        foreach (var item in query)
        {
            Console.WriteLine($"'{item.Char}' appears {item.Count} time(s)");
        }
    }
}
```

---

## 5. Remove Duplicates from an Array / String

### Question:
Remove duplicates from an array while preserving original order.

### C# Solution:
```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class RemoveDuplicates
{
    // Using HashSet for O(N) time preserving order
    public static T[] RemoveDuplicatesArray<T>(T[] array)
    {
        var seen = new HashSet<T>();
        var result = new List<T>();

        foreach (var item in array)
        {
            if (seen.Add(item)) // Add returns false if already present!
            {
                result.Add(item);
            }
        }

        return result.ToArray();
    }

    // Remove duplicates from string
    public static string RemoveDuplicateChars(string input)
    {
        if (string.IsNullOrEmpty(input)) return input;

        var seen = new HashSet<char>();
        var sb = new System.Text.StringBuilder();

        foreach (char c in input)
        {
            if (seen.Add(c))
                sb.Append(c);
        }

        return sb.ToString();
    }
}

// Example Run:
// RemoveDuplicateChars("google") => "gole"
// RemoveDuplicatesArray(new int[] { 1, 2, 2, 3, 4, 1 }) => [1, 2, 3, 4]
```

---

## 6. Two Sum Problem (Find Pair with Given Sum)

### Question:
Given an array of integers and a target sum, find the indices of the two numbers that add up to the target.

### C# Solution:
```csharp
using System;
using System.Collections.Generic;

public class TwoSumProblem
{
    // O(N) time and O(N) space using Dictionary
    public static int[] TwoSum(int[] nums, int target)
    {
        var map = new Dictionary<int, int>(); // Value -> Index

        for (int i = 0; i < nums.Length; i++)
        {
            int complement = target - nums[i];

            if (map.ContainsKey(complement))
            {
                return new int[] { map[complement], i };
            }

            // Store current value and index
            if (!map.ContainsKey(nums[i]))
            {
                map[nums[i]] = i;
            }
        }

        return Array.Empty<int>(); // No pair found
    }
}

// Example: nums = [2, 7, 11, 15], target = 9
// i = 0: complement = 7, map[2] = 0
// i = 1: complement = 2 (exists in map!) => returns [0, 1]
```

---

## 7. Check if Two Strings are Anagrams

### Question:
Determine if two strings contain the exact same characters in any order (e.g., `"silent"` and `"listen"`).

### C# Solution:
```csharp
using System;

public class AnagramChecker
{
    // Array frequency count (O(N) time, O(1) extra space)
    public static bool AreAnagrams(string s1, string s2)
    {
        if (s1.Length != s2.Length) return false;

        int[] charCount = new int[256]; // ASCII character array

        for (int i = 0; i < s1.Length; i++)
        {
            charCount[s1[i]]++;
            charCount[s2[i]]--;
        }

        // Check if all counts return to zero
        foreach (int count in charCount)
        {
            if (count != 0) return false;
        }

        return true;
    }
}
```

---

## 8. Balanced Parentheses / Brackets Check

### Question:
Check if string containing parentheses `()`, brackets `[]`, and braces `{}` is balanced.  
Example: `"{[()]}"` is valid, `"{[(])}"` is invalid.

### C# Solution:
```csharp
using System;
using System.Collections.Generic;

public class BracketValidator
{
    public static bool IsValid(string s)
    {
        if (string.IsNullOrEmpty(s)) return true;

        var stack = new Stack<char>();

        foreach (char c in s)
        {
            // Push opening brackets onto stack
            if (c == '(' || c == '{' || c == '[')
            {
                stack.Push(c);
            }
            // For closing brackets, check matching top of stack
            else if (c == ')' || c == '}' || c == ']')
            {
                if (stack.Count == 0) return false; // Closing without opening

                char top = stack.Pop();

                if ((c == ')' && top != '(') ||
                    (c == '}' && top != '{') ||
                    (c == ']' && top != '['))
                {
                    return false; // Mismatched bracket
                }
            }
        }

        // Must be empty if all brackets were closed
        return stack.Count == 0;
    }
}
```

---

## 9. Fibonacci Series (Iterative vs Recursive)

### Question:
Generate the Nth Fibonacci number or print the first N Fibonacci numbers.

### C# Solution:
```csharp
using System;

public class Fibonacci
{
    // Iterative: O(N) time, O(1) space - BEST FOR INTERVIEW
    public static int GetNthFibonacci(int n)
    {
        if (n <= 0) return 0;
        if (n == 1) return 1;

        int a = 0;
        int b = 1;

        for (int i = 2; i <= n; i++)
        {
            int c = a + b;
            a = b;
            b = c;
        }

        return b;
    }

    // Print first N terms
    public static void PrintFibonacciSeries(int count)
    {
        int a = 0, b = 1;
        Console.Write($"{a} {b} ");

        for (int i = 2; i < count; i++)
        {
            int c = a + b;
            Console.Write($"{c} ");
            a = b;
            b = c;
        }
        Console.WriteLine();
    }
}
```

---

## 10. Check if a Number is Prime

### Question:
Check if integer $N > 1$ is a prime number.

### C# Solution:
```csharp
using System;

public class PrimeChecker
{
    // O(sqrt(N)) time complexity
    public static bool IsPrime(int number)
    {
        if (number <= 1) return false;
        if (number == 2 || number == 3) return true;
        if (number % 2 == 0 || number % 3 == 0) return false;

        // Check factors up to square root of number, incrementing by 6
        for (int i = 5; i * i <= number; i += 6)
        {
            if (number % i == 0 || number % (i + 2) == 0)
                return false;
        }

        return true;
    }
}
```

---

## 11. Swap Two Numbers Without Temporary Variable

### Question:
Swap two variables `a` and `b` without introducing a 3rd helper variable.

### C# Solution:
```csharp
public class NumberSwap
{
    public static void SwapArithmetic(ref int a, ref int b)
    {
        a = a + b; // a now holds sum
        b = a - b; // b becomes original a
        a = a - b; // a becomes original b
    }

    public static void SwapBitwise(ref int a, ref int b)
    {
        a = a ^ b;
        b = a ^ b;
        a = a ^ b;
    }

    // Modern C# Tuple swap (cleanest in C# 7+)
    public static void SwapTuple(ref int a, ref int b)
    {
        (a, b) = (b, a);
    }
}
```

---

## 12. Find the Missing Number in an Array (1 to N)

### Question:
Given array containing $N-1$ distinct numbers in range $[1, N]$, find the one missing number.

### C# Solution:
```csharp
public class MissingNumberFinder
{
    // Using mathematical formula: Sum = N * (N + 1) / 2
    public static int FindMissingNumber(int[] nums, int n)
    {
        long expectedSum = (long)n * (n + 1) / 2;
        long actualSum = 0;

        foreach (int num in nums)
        {
            actualSum += num;
        }

        return (int)(expectedSum - actualSum);
    }
}

// Example: nums = [1, 2, 4, 5, 6], n = 6
// expectedSum = 6 * 7 / 2 = 21
// actualSum = 18
// missing = 21 - 18 = 3
```

---

## 13. FizzBuzz Problem

### Question:
Print numbers from 1 to $N$. Multiples of 3 print "Fizz", multiples of 5 print "Buzz", multiples of both print "FizzBuzz".

### C# Solution:
```csharp
using System;

public class FizzBuzz
{
    public static void Run(int n)
    {
        for (int i = 1; i <= n; i++)
        {
            bool divBy3 = (i % 3 == 0);
            bool divBy5 = (i % 5 == 0);

            if (divBy3 && divBy5)
                Console.WriteLine("FizzBuzz");
            else if (divBy3)
                Console.WriteLine("Fizz");
            else if (divBy5)
                Console.WriteLine("Buzz");
            else
                Console.WriteLine(i);
        }
    }
}
```

---

## 14. Merge Two Sorted Arrays

### Question:
Merge two sorted arrays `arr1` and `arr2` into a single sorted array without using `.Sort()`.

### C# Solution:
```csharp
public class SortedArrayMerge
{
    // Two pointer technique: O(M + N) time
    public static int[] Merge(int[] arr1, int[] arr2)
    {
        int i = 0, j = 0, k = 0;
        int[] result = new int[arr1.Length + arr2.Length];

        while (i < arr1.Length && j < arr2.Length)
        {
            if (arr1[i] <= arr2[j])
            {
                result[k++] = arr1[i++];
            }
            else
            {
                result[k++] = arr2[j++];
            }
        }

        // Copy remaining elements of arr1
        while (i < arr1.Length)
            result[k++] = arr1[i++];

        // Copy remaining elements of arr2
        while (j < arr2.Length)
            result[k++] = arr2[j++];

        return result;
    }
}
```

---

## 15. Top 5 LINQ Practical Coding Snippets

### Interview Snippets:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class LinqSnippets
{
    public class Employee
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Department { get; set; }
        public decimal Salary { get; set; }
    }

    public static void CommonQuestions(List<Employee> employees)
    {
        // 1. Find Highest Salary in each Department
        var highestPerDept = employees
            .GroupBy(e => e.Department)
            .Select(g => new
            {
                Dept = g.Key,
                TopEarner = g.OrderByDescending(e => e.Salary).FirstOrDefault()
            });

        // 2. Second Highest Salary across company
        var secondHighestSalary = employees
            .Select(e => e.Salary)
            .Distinct()
            .OrderByDescending(s => s)
            .Skip(1)
            .FirstOrDefault();

        // 3. Count employees in each department where salary > 50,000
        var deptStats = employees
            .Where(e => e.Salary > 50000)
            .GroupBy(e => e.Department)
            .Select(g => new { Department = g.Key, Count = g.Count() });

        // 4. Any vs All check
        bool allHighEarners = employees.All(e => e.Salary >= 30000);
        bool anyExecutive = employees.Any(e => e.Salary > 200000);

        // 5. Select vs SelectMany (Flattening nested lists)
        var departmentsWithTags = new List<(string Dept, List<string> Skills)>
        {
            ("IT", new List<string> { "C#", "SQL", "Azure" }),
            ("HR", new List<string> { "Recruitment", "Payroll" })
        };

        // Select gives List<List<string>>
        // SelectMany gives List<string> (flattened)
        var allSkillsDistinct = departmentsWithTags
            .SelectMany(d => d.Skills)
            .Distinct()
            .ToList();
    }
}
```

---

*Last Updated: 11-Oct-2026 | Infosys Interview Prep | Live Coding Module*
