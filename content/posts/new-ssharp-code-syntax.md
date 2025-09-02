---

title: What's New in C# 12 and 13?  
date: 2024-06-14  
author: Nguyen Thanh Tuyen  
tags: [".NET", "language features"]  
summary: "C# 12 is out and C# 13 is coming—see what’s new"  

---

# What’s New in C# 12 and 13? A Quick Guide for Modern Backend Developers

C# is evolving fast—not just for desktop or mobile, but to help us write more elegant, high-performance cloud and microservice backends. Here’s a focused summary on what really matters in **C# 12** (now released) and the most exciting proposals in **C# 13** (coming soon). I’ll highlight features with direct impact for backend work, along with bite-size code samples.

---

## C# 12: What’s New (Released with .NET 8)

### 1. **Primary Constructors for Non-Records**

One of the most impactful: primary constructors are now allowed in **classes and structs**, not just records. Makes DTOs, services, and many microservice models cleaner.

```csharp
public class Invoice(string id, decimal amount)
{
    public string Id { get; } = id;
    public decimal Amount { get; } = amount;
}
```
**Why it matters:**  
Less boilerplate, better immutability for models across APIs.

---

### 2. **Collection Expressions**

Quickly create arrays and lists without explicit constructors or `new` keyword (inspired by JavaScript, Python).

```csharp
int[] a = [1, 2, 3];
List<string> b = ["A", "B", "C"];
```
**Why it matters:**  
Faster and cleaner initialization of configs, DTOs, test data.

---

### 3. **Default Lambda Parameters**

Set default values right in your lambda arguments.

```csharp
Func<int, int, int> adder = (a, b = 1) => a + b;
Console.WriteLine(adder(10)); // 11
```
**Why it matters:**  
Lambdas are more flexible for utilities, especially in LINQ or functional code.

---

### 4. **Alias Any Type (using Aliases for Tuples, Arrays, Spans, etc.)**

Use the `using` keyword as an alias for complex types, not just classes.

```csharp
using Person = (string Name, int Age);

Person p = ("Alice", 30);
```
**Why it matters:**  
Makes working with complex tuple or generic types readable and maintainable.

---

### 5. **Optional Parameters in Struct Constructors**

You can now use optional and default values in struct constructors.

```csharp
public struct Point(int x = 0, int y = 0)
{
    public int X = x;
    public int Y = y;
}
```

---

### 6. **Inline Arrays**

Better support for fixed-length buffers and serialization—useful in high-perf and interop code.

```csharp
[System.Runtime.CompilerServices.InlineArray(10)]
public struct InlineByteArray
{
    private byte _element0;
}
```

---

## C# 13: What’s Coming? (Preview/Proposal)

C# 13 features are still being discussed, but several are hot topics and some are appearing in early previews. Here’s what backend engineers should keep an eye on:

---

### 1. **Params Collections in Method Signatures**

Write APIs that take variable-sized arrays or lists but remain type-safe and concise.

```csharp
void PrintNames(params string[] names) { }
void PrintNumbers(params List<int> nums) { } // Collection params in broader usage
```

---

### 2. **Lambda Expressions with Attributes**

Soon you’ll be able to decorate lambdas with attributes, enhancing diagnostics and policies for advanced function pipelining or middleware.

```csharp
[MyLog]
var add = ([LogArg] int a, int b) => a + b;
```

---

### 3. **Primary Constructors for Abstract and Static Classes**

Primary constructors are expected to be extended for even more class types, such as abstract/service-oriented classes.

```csharp
public abstract class BaseHandler(string type) { }
```

---

### 4. **Pattern Matching Enhancements**

Pattern matching keeps improving (properties, collections, discriminated unions) to write safer, more concise microservice logic.

```csharp
if (obj is Order { Status: "Pending", Amount: > 100 }) { ... }
```

---

### 5. **Improved Intellisense/Diagnostics**

Not a code feature, but C# 13 + tooling is expected to deliver enhanced code analyzers, error messages, and fix suggestions, reducing daily friction for all engineers.

---

## Should You Upgrade?

**If you’re on .NET 8:**  
Start using C# 12 features today—especially primary constructors, collection expressions, and type aliasing. They pay off in model clarity and fewer bugs.

**For C# 13:**  
Follow its preview if you love cutting-edge—big gains are coming for API ergonomics and pattern-based architecture.

---

## Final Thoughts

C# remains a modern, rapidly evolving language. Features like primary constructors, collection expressions, and smarter defaults matter a lot as we build bigger, more reliable distributed systems. Adopt them early—you’ll save code, cut bugs, and keep your toolkit sharp.

**Have a favorite new feature or one you’re waiting for? Share below!**

---

*Nguyen Thanh Tuyen – Technical Lead, practical cloud architecture and modern .NET from the trenches*