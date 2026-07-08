---
layout: post
title: "Decorator Pattern"
date: 2026-07-07 15:05:31 +0200
tags: c# design-patterns software-development
---

One of the biggest challenges in our daily job is keeping our code clean, easy to read, extendable, and understandable. It's a difficult task, but it becomes easier when following design patterns.

In software development, design patterns are concepts that define how to write code. They date back to as early as the 1970s. Design patterns can be divided into at least four different categories:

- Creational patterns
- Structural patterns
- Behavioral patterns
- Concurrency patterns

Today's specific topic is the **decorator pattern**, which is one of the structural design patterns. Let's take a closer look together.

<!--more-->

# What is a Decorator Design Pattern?

The decorator pattern allows you to add new behavior to existing objects, extending their functionalities without affecting the behavior of other objects of the same type. It's done by creating a set of **decorator** classes to wrap the concrete component. This pattern follows the open-closed principle and the single responsibility principle.

# Analogy

Before diving into the technical details, let's use an analogy to illustrate the decorator pattern.

Consider going out on a cold day. To stay warm, you put on a sweater. When you step outside, you feel even colder, so you add a jacket. Later, you wrap yourself with a scarf for extra warmth.

The sweater, jacket, and scarf are decorating you and extending your behavior, but they aren't part of you. They can be easily added or removed without affecting each other.

# Concept

Here's a conceptual structure of the decorator pattern:

![decorator diagram](/assets/decorator-diagram.png)

- IInterface: The entry point for the caller.
- Concrete Component: Defines the default behavior of the interface, which can be extended by the decorators.
- Abstract Decorator: The base decorator that contains the wrapped interface. It helps make the code cleaner with fewer repetitive declarations.
- Concrete decorator: Extends the default behavior and delegate the job to the wrapped decorator.
- The caller can wrap the component with different decorators at runtime.

### Why use the abstract class?

You might be asking, why use an abstract class? Can we skip it? The answer is simple: yes, but it's better not to. The abstract class defines how your decorators should minimally behave and reduces the repetition of creating the wrapped field inside each of your decorators.

# Pragmatic example using the decorator pattern

Let's assume that we're creating a logging library. Our first task is to take a message and log it. For simplicity, we'll be logging to the console instead of a file or other sinks.

## Our First Implementation

We first start by tackling the task directly.

```csharp
public interface ILogger
{
    Task Log(string message);
}
```

```csharp
public class Logger(LoggerConfiguration configuration) : ILogger
{
    public Task Log(string message)
    {
        Console.WriteLine(message);
        return Task.CompletedTask;
    }
}
```

A simple interface implemented by a concrete component, which can be injected directly into our services.  
Consider we also chose to create an extension to help the user inject our service without too much hassle:

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddLogger(this IServiceCollection services, LoggerConfiguration configuration)
    {
        services.AddSingleton<ILogger, Logger>((_) => new Logger(configuration));
        return services;
    }
}
```

Now, let's test this simple logger:

```csharp
using AllPhiLogger.Configuration;
using AllPhiLogger.Core;
using AllPhiLogger.Extensions;
using Microsoft.Extensions.DependencyInjection;

var serviceProvider = BuildServiceProvider();
var logger = serviceProvider.GetRequiredService<ILogger>();
await logger.Log("Important message here!");

static IServiceProvider BuildServiceProvider()
{
    return new ServiceCollection()
        .AddLogger(new LoggerConfiguration
        {
            Filename = "placeholder",
            Path = "path"
        }).BuildServiceProvider();
}
```

Result:

> Important message here!

## Extra tasks

We then chose to add a timestamp to the log. However, we'd like to keep the configuration in our user's hands, and whether they need a timestamp or not is up to them!

To keep the code clean and follow coding principles, we decide to implement the extra tasks using the **decorator pattern**.

First, we declare the abstract decorator:

```csharp
public abstract class BaseLoggerDecorator(ILogger logger) : ILogger
{
    public virtual async Task Log(string message)
    {
        await logger.Log(message);
    }
}
```

This defines our base decorator that wraps the existing component.

We extend our configuration class to include a date format:

```csharp
public string? DateFormat { get; set; }
```

Next, we create the concrete decorator that will handle the timestamp addition:

```csharp
public class TimestampLoggerDecorator(ILogger logger, string dateFormat) : BaseLoggerDecorator(logger)
{
    public override async Task Log(string message)
    {
        message = $"[{DateTime.UtcNow.ToString(dateFormat)}] - {message}";
        await base.Log(message);
    }
}
```

We also update our services extension to decorate our logger in case a date format was provided. But before we do that, let's take a quick look on the two different options we have. We can either directly add our decorator using an anonymous function, like the following:

```csharp
services.AddSingleton<ILogger>((_) =>
    new TimestampLoggerDecorator(new Logger(configuration), configuration.DateFormat));
```

This doesn't look very nice but it's usable. However, it will get messy if we want to decorate our service according to certain criteria. If we need to configure our logger during registration in a cleaner way, I then suggest using [Scrutor Library](https://github.com/khellang/Scrutor), which allows you to use a `.Decorate<>` extension method. Our new extension looks like this:

```csharp
public static IServiceCollection AddLogger(this IServiceCollection services, LoggerConfiguration configuration)
{
    services.AddSingleton<ILogger, Logger>((_) => new Logger(configuration));

    if (!string.IsNullOrWhiteSpace(configuration.DateFormat))
    {
        services.Decorate<ILogger>((inner) =>
            new TimestampLoggerDecorator(inner, configuration.DateFormat));
    }

    return services;
}
```

This gives the user the option to include a date format or not, and we'll decorate our `ILogger` accordingly. Let's give this a test!

Our new service provider builder will change to:

```csharp
static IServiceProvider BuildServiceProvider()
{
    return new ServiceCollection()
        .AddLogger(new LoggerConfiguration
        {
            Filename = "placeholder",
            Path = "path",
            DateFormat = "dd/MM/yyyy HH:mm:ss"
        }).BuildServiceProvider();
}
```

And finally, the new result is:

> [17-11-2024 21:33:28] - Important message here!

You can notice that we didn't touch any of our initial code. Neither the concrete implementation nor the log call. We only had to extend our configuration model, and change our service builder.

For an additional task, some of our users wanted to have the ability to add a stack trace up to a configured frame number. So let's do it again - let's keep decorating our `ILogger`.

As before, let's take the same steps:

- Update our configuration model.
- Update our extension.
- Update the service provider builder.

```csharp
public int? StackFrames { get; set; }
```

```csharp
public static IServiceCollection AddLogger(this IServiceCollection services, LoggerConfiguration configuration)
{
    services.AddSingleton<ILogger, Logger>((_) => new Logger(configuration));

    if (!string.IsNullOrWhiteSpace(configuration.DateFormat))
    {
        services.Decorate<ILogger>((inner) =>
            new TimestampLoggerDecorator(inner, configuration.DateFormat));
    }

    if (configuration.StackFrames != null)
    {
        services.Decorate<ILogger>((inner) =>
            new StackTraceLoggerDecorator(inner, configuration.StackFrames.Value));
    }

    return services;
}
```

```csharp
public class StackTraceLoggerDecorator(ILogger logger, int framesNumber) : BaseLoggerDecorator(logger)
{
    public override async Task Log(string message)
    {
        var st = new StackTrace(framesNumber, false);
        message = $"{message} - Stack Trace: {st}";
        await base.Log(message);
    }
}
```

```csharp
static IServiceProvider BuildServiceProvider()
{
    return new ServiceCollection()
        .AddLogger(new LoggerConfiguration
        {
            Filename = "placeholder",
            Path = "path",
            DateFormat = "dd/MM/yyyy HH:mm:ss",
            StackFrames = 2
        }).BuildServiceProvider();
}
```

You can notice that, again, we didn't need to touch any of the previously implemented code. We just had to extend what we had.
Running our app now provides us with the following result:

> [17-11-2024 21:56:14] - Important message here! - Stack Trace: at AllPhiLogger.Decorators.StackTraceLoggerDecorator.Log(String message)
> at Program.<Main>$(String[] args)
>   at System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start[TStateMachine](TStateMachine& stateMachine)
>   at Program.<Main>$(String[] args)
> at Program.<Main>(String[] args)

### Runtime decorations

One of the advantages of the decorator pattern is the ability to add or remove decorators at runtime. As demonstrated above, the decorators can be added or removed according to a certain configuration, which allows you to dynamically change the behavior of your decorated service.

# Final Thoughts

## Advantages

- Extend behaviors without much trouble and without affecting previous behaviors.
- Dynamically add or remove decorators during runtime.
- Applies the Single Responsibility Principle.
- Applies the Open-Closed Principle.
- Doesn't break the code that already uses the decorated type.
- It has a clear structured design.
- Avoids creating large number of subclasses for every possiblity.

## Disadvantages

- It can become complex with an increased number of decorators, whether to follow the order of execution, or navigating the codebase to debug.
- Without a proper library, the way of configuring the wrapper stack can become quite messy.
- Every decorator adds an additional level of wrapping, which can result in increased memory usage.

---

There you have it—everything you need to know about the decorator pattern.

The demo code can be found [here](https://github.com/tiger4589/design-patterns/tree/main/Structural%20Patterns/Decorator%20Pattern).
