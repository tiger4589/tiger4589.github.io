---
layout: post
title: "Union Types in C#"
date: 2026-08-15
tags: csharp dotnet preview union-types
---

For a long time in `C#` we got really good at modeling what things are, but we're still not that good at modeling what things can't be.

This will change, as a long awaited feature in `C#` is arriving with `.NET 11` and `C# 15`.

This feature isn't just syntactic sugar, it actually changes how we model data, and will let the compiler enforces more correctness.

This is going to be an introduction to look at what union types solve, and what they help with. Note that the article reflects the current preview syntax and behavior.

<!--more-->

# The problem

One of the very common examples is representing a result:

```csharp
public class Result<T>
{
    public bool IsSuccess { get; set; }
    public string? Error { get; set; }
    public T? Value { get; set; }
}
```

This compiles and works, however:

- You might end up with `IsSuccess` true and `Error` not null.
- You might end up with `IsSuccess` false and `Value` not null.

Developers end up relying on discipline, and not the type system. The compiler can't really prevent these invalid states.

### Common Workarounds

We have tried to solve this with:

- Inheritance
- Interfaces
- Tuples
- Libraries (OneOf, Result patterns, etc...)

# Union Types

### What are union types?

A quote from Microsoft documentation:

> A union type represents a value that can be one of several case types. Unions provide implicit conversions from each case type, exhaustive pattern matching, and enhanced nullability tracking.

In other words, it lets a value be **exactly one of several defined types**.

### Syntax - Example

Let's consider we have the following three different results for our calls:

```csharp
public record Success<T>
{
    public required T Value { get; init; }
}

public record SuccessWithWarning<T>
{
    public required T Value { get; init; }
    public required string Warning { get; init; }
}

public record Failure
{
    public required string Error { get; init; }
}
```

You can either get a success with a value, a success with a value and a warning, or a failure.

You can define them as a union like this:

```csharp
public union Result<T>(Success<T>, SuccessWithWarning<T>, Failure);
```

Let's consider we have a library that returns a `Result<int>`, with the following signature:

`public Result<int> GetResult()`

The caller would use pattern matching to handle each case differently if needed:

```csharp
var result = GetResult();

var description = result switch
{
    Success<int> success => $"Success: {success.Value}",
    SuccessWithWarning<int> successWithWarning =>
        $"Success with Warning: {successWithWarning.Value}, Warning: {successWithWarning.Warning}",
    Failure failure => $"Failure: {failure.Error}",
};
```

What's the main difference between what we currently have and union types?

- You cannot construct an invalid state.
- The compiler ensures all cases are handled.

In case you forget one case, you'd get a compile-time warning or error!

# Real-world use cases

I'll list some of the examples where union types can be used.

### Domain Modeling

A payment system where it can take different payments methods:

```csharp
public union PaymentMethod(CreditCard, PayPal, Crypto, Cash);
```

You won't accidentally mix fields.

### State Machines

In a state machine, impossible states will be literally impossible:

```csharp
public union OrderState(Pending, Paid, Shipped, Cancelled);
```

### API Response

```csharp
public union ApiResponse(Ok, NotFound, Unauthorized);
```

# Final Thoughts

Union types will push us toward:

- Better modeling
- Safer Refactoring (New case? Compiler will show you where to handle it)
- Clearer APIs

We should be careful though not to overuse it everywhere.  
Not everything will need a union.

In case of simple data, use classes. If you have a complex branching logic, use unions.  
Too many use cases can become hard to follow.

### When to use?

Use unions when

- You have distinct **states**.
- You want compile-time guarantees.
- You're modeling business rules.

Avoid when

- Data is simple and uniform.
- Flexibility matters more than strictness.

# How to try it yourself currently?

- Install .NET 11 Preview (it's currently preview 7).
- Use Visual Studio Insiders or the latest C# DevKit Insiders build.
- Create a new project with .NET 11.
- Add <LangVersion>preview</LangVersion> to your project file.

# References

[Microsoft Learn Documentation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/union)  
[Dev Blogs](https://devblogs.microsoft.com/dotnet/csharp-15-union-types/)
