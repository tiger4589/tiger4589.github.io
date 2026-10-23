---
layout: post
title: "The Options Pattern in ASP.NET Core: Typed, Validated Configuration"
date: 2026-10-21
tags: dotnet aspnetcore csharp configuration options-pattern validation
---

One of the things that needs careful handling in any application is configuration. You don't want to use magic numbers and strings, or hardcoded values inside your application, and that's for a simple reason: changing them means a new deployment that has to go through the whole software development life cycle. What we usually do is put them in a configuration file and read them at runtime. With that comes a new challenge: how do we ensure they are valid and correctly set up? How do we make sure we haven't forgotten anything, rather than discovering it at runtime? That's where the options pattern comes in handy.

In this article I will walk through what happens if we're not using this pattern, and what it brings to the table in an ASP.NET Core application.

<!--more-->

# Demo scenario

As usual, I like to keep my demos simple and centered on the subject. Let's consider that we have an API endpoint that returns email settings: sender, subject, and the name of the template currently in use.  
We'll start without the options pattern, reading straight from `appsettings.json` using `IConfiguration`, highlight the issues we might run into, and then implement the pattern step by step, showing what it brings.

### The configuration

```json
{
  "EmailSettings": {
    "Sender": "",
    "Subject": "",
    "Template": ""
  }
}
```

This is the configuration that you can use if you want to test this exact demo. Fill in the values, or leave them empty to test out the validations.

## Using IConfiguration

By injecting `IConfiguration` in your endpoint, you can read the configuration saved in your `appsettings.json`.

For example:

```csharp
app.MapGet("/email-settings", (IConfiguration configuration) =>
    {
        var sender = configuration["EmailSettings:Sender"];
        if (string.IsNullOrWhiteSpace(sender))
        {
            return Results.InternalServerError("Sender not found");
        }

        var subject = configuration["EmailSettings:Subject"];
        if (string.IsNullOrWhiteSpace(subject))
        {
            return Results.InternalServerError("Subject not found");
        }

        var template = configuration["EmailSettings:Template"];
        if (string.IsNullOrWhiteSpace(template))
        {
            return Results.InternalServerError("Template not found");
        }

        return Results.Ok(new
        {
            Sender = sender,
            Subject = subject,
            Template = template
        });
    })
.WithName("GetEmailSettings");
```

This is a pretty straightforward endpoint, reading the values, and returning them. If something is missing, it will return an internal server error. However, this hides potential problems.

### Issues you might run into

- Magic Strings / Hardcoded sections

Now imagine this isn't a simple scenario and the configuration is only accessed several layers below the endpoint. If you've mistyped a section or key name, you'll only know about it during runtime.

- More than one place

If you need to access the configuration in more than one place, what is your strategy going to be? Rewrite the same code? Move it to a helper class? This will create more failure points in your codebase, which is best avoided.

- Validations

The validation in this example will only run when you fetch the configuration. As with the first point, you won't know something is wrong until the runtime. It's also similar to the second point where if you need the configuration in more than one place, what is your strategy going to be?

- Coupling

Even if you fix the first three points, you'll end up coupling your domain logic with the fetching and validating of the configuration.

## Using the Options Pattern

### What is the options pattern?

The options pattern uses a class to provide strongly typed access to your configuration. By isolating the configuration into a separate class, your application follows two principles:

- Encapsulation
- Separation of concerns

It also provides a way to validate the configuration, either at runtime or at startup, depending on how you configure it. This lets you fail fast (when validating on start) and catch issues before they surface at runtime.

### Options Class

To be able to use the options pattern, you first need to create an options class related to your configuration section. In our example:

```csharp
public class EmailSettingsOptions
{
    public const string EmailSettings = nameof(EmailSettings);

    public string? Sender { get; set; }
    public string? Subject { get; set; }
    public string? Template { get; set; }
}
```

The options class is a simple .NET class with public read-write properties that match your configuration entries.

The `EmailSettings` constant isn't bound (the binder ignores fields). It's used to avoid hardcoding the configuration section name when binding it to the configuration provider.

### Binding options to the DI service container

You can register your options in your dependency injection service container by calling the `.Configure<T>` method:

```csharp
builder.Services.Configure<EmailSettingsOptions>(
    builder.Configuration.GetSection(EmailSettingsOptions.EmailSettings));
```

### Injecting in the endpoint

You can now inject `IOptions<TOptions>` in your endpoint and test it out:

```csharp
app.MapGet("/email-settings", (IOptions<EmailSettingsOptions> emailSettingsOptions) => Results.Ok(new
    {
        emailSettingsOptions.Value.Sender,
        emailSettingsOptions.Value.Subject,
        emailSettingsOptions.Value.Template
    }))
.WithName("GetEmailSettings");
```

**Important note: `IOptions<TOptions>` reads the configuration once and never sees later changes, and is registered as a singleton service. The value is computed once, on first access, and cached for the app's lifetime.**

If you need the options to be recomputed on every request, you have to inject `IOptionsSnapshot<TOptions>` instead:

```csharp
app.MapGet("/email-settings", (IOptionsSnapshot<EmailSettingsOptions> emailSettingsOptions) => Results.Ok(new
    {
        emailSettingsOptions.Value.Sender,
        emailSettingsOptions.Value.Subject,
        emailSettingsOptions.Value.Template
    }))
.WithName("GetEmailSettings");
```

The problem that you might face with `IOptionsSnapshot<T>` is that it's scoped, so you can't inject it into singletons. But there's a solution! You can inject the `IOptionsMonitor<T>` instead.

`IOptionsMonitor<T>` is registered as a singleton and still picks up future changes without app restart.

It can be injected the same way as the others in your endpoint, but instead of the `Value` property, it uses `CurrentValue`:

```csharp
app.MapGet("/email-settings", (IOptionsMonitor<EmailSettingsOptions> emailSettingsOptions) => Results.Ok(new
    {
        emailSettingsOptions.CurrentValue.Sender,
        emailSettingsOptions.CurrentValue.Subject,
        emailSettingsOptions.CurrentValue.Template
    }))
.WithName("GetEmailSettings");
```

`IOptionsMonitor<T>` also exposes `OnChange` method. It takes a callback (`Action<T, string?>`) and returns an `IDisposable`. If you've got a background service that reads the configuration once and uses it during the app lifetime, then you can use the `OnChange` method to update the values when they are modified in the `appsettings.json`.

To compare the three:

| Interface                  | Lifetime  | Reloads  |
| -------------------------- | --------- | -------- |
| IOptions<TOptions>         | Singleton | &#10007; |
| IOptionsSnapshot<TOptions> | Scoped    | &#10003; |
| IOptionsMonitor<TOptions>  | Singleton | &#10003; |

Which one to choose is up to you and what your application needs.

If you've got a configuration that would almost never change, you probably only need `IOptions`, otherwise you will need either one of the two others. Choosing between the two depends on where your configuration is being accessed.

If it's used in a singleton service, then going with the `IOptionsMonitor` is certainly advised, but if it's being used in a scoped service, then using `IOptionsSnapshot` might fit your needs, but that doesn't mean that you can't or shouldn't use `IOptionsMonitor` in a scoped service.

One pitfall to avoid is that `IOptionsMonitor.CurrentValue` can change halfway through a request if the configuration file is edited at that moment, while `IOptionsSnapshot` gives you a consistent value for the whole request. On the other hand, `IOptionsSnapshot` rebinds on every request, which has a small performance cost.

It's also important to know that reloading depends on the provider. By default a `WebApplication` has the `appsettings.json` registered with `reloadOnChange: true`. This works in this case. However, using environment variables and command-line arguments never reload.

### Validation

So far, we've dropped the validation that existed in our first example. So let's add it back.

Using the options pattern in .NET you can add validation in multiple ways:

- Using data annotations
- Using a dedicated class with `IValidateOptions<TOptions>`
- Using a delegate

A small change to the service registration is needed to add validation. Instead of using `.Configure`, we will need to use `.AddOptions` to get back an `OptionsBuilder<TOptions>`.

**Using a delegate**:

```csharp
builder.Services.AddOptions<EmailSettingsOptions>().Bind(
    builder.Configuration.GetSection(EmailSettingsOptions.EmailSettings))
    .Validate(options => !string.IsNullOrWhiteSpace(options.Subject) && options.Subject.Length <= 100, "Subject must be at most 100 characters");
```

**Using data annotation**:

To use data annotation, we need to modify the `EmailSettingsOptions` class:

```csharp
public class EmailSettingsOptions
{
    public const string EmailSettings = nameof(EmailSettings);

    [Required]
    public string? Sender { get; set; }
    [Required]
    [MaxLength(100)]
    public string? Subject { get; set; }
    [Required]
    [MaxLength(20)]
    public string? Template { get; set; }
}
```

and instead of just calling `Validate` with a delegate in the service registration, we add a call to `ValidateDataAnnotations`:

```csharp
builder.Services.AddOptions<EmailSettingsOptions>().Bind(
    builder.Configuration.GetSection(EmailSettingsOptions.EmailSettings))
    .ValidateDataAnnotations();
```

**Using a dedicated class with `IValidateOptions<TOptions>`**:

First, remove the data annotations you previously added to the `EmailSettingsOptions` and create a new class:

```csharp
public class EmailSettingsOptionsValidation : IValidateOptions<EmailSettingsOptions>
{
    public ValidateOptionsResult Validate(string? name, EmailSettingsOptions options)
    {
        StringBuilder validationResult = new();

        if (string.IsNullOrWhiteSpace(options.Subject))
        {
            validationResult.AppendLine($"{nameof(options.Subject)} must not be empty");
        }

        // Implement additional validation logic here if needed - using a simple example for demonstration purposes

        if (validationResult.Length > 0)
        {
            return ValidateOptionsResult.Fail(validationResult.ToString());
        }

        return ValidateOptionsResult.Success;
    }
}
```

Finally, the service registration will change to:

```csharp
builder.Services.AddOptions<EmailSettingsOptions>().Bind(
    builder.Configuration.GetSection(EmailSettingsOptions.EmailSettings));

builder.Services.AddSingleton<IValidateOptions<EmailSettingsOptions>, EmailSettingsOptionsValidation>();
```

**Run validation when the app starts**:

In order to fail fast, and immediately catch any issues with your configuration, you can run the validations when the app starts.

In all three cases you just need to call the `ValidateOnStart` method in service registration:

```csharp
builder.Services.AddOptions<EmailSettingsOptions>().Bind(
    builder.Configuration.GetSection(EmailSettingsOptions.EmailSettings))
    .ValidateDataAnnotations() // or by adding an IValidateOptions
    .ValidateOnStart();
```

With `ValidateOnStart`, the app won't start if any validation fails. This way you will catch all the validation issues and fix them before any runtime surprises. It's important to know that this only covers startup. If someone edits the configuration, the options are recomputed when using `IOptionsSnapshot<T>` or `IOptionsMonitor<T>`. Accessing their value will still throw an `OptionsValidationException` during runtime.

**Note**: As with `IConfiguration`, a mistyped section name still fails silently: it binds to an object whose properties are all null. Validation combined with `ValidateOnStart` catches this as early as possible, but only if your validation rules reject null values.

**Using `ErrorOnUnknownConfiguration`**:

You can go one step further to keep your configuration clean by using `ErrorOnUnknownConfiguration` during service registration. For example:

```csharp
builder.Services.AddOptions<EmailSettingsOptions>().Bind(
    builder.Configuration.GetSection(EmailSettingsOptions.EmailSettings), binderOptions =>
    {
        binderOptions.ErrorOnUnknownConfiguration = true;
    })
    .ValidateOnStart();
```

How does that help?

Consider you added a new key in your email settings configuration:

```json
{
  "EmailSettings": {
    "Sender": "",
    "Subject": "",
    "Template": "",
    "Provider": ""
  }
}
```

If you don't add the `Provider` property in the `EmailSettingsOptions` class, an exception will be thrown on app start because there's a configuration mismatch. The same can be said in case you mistyped `Sender` as `Sendr`. However, the opposite doesn't throw. Adding a property that doesn't exist in the configuration is ignored.

Using `ErrorOnUnknownConfiguration` makes `ValidateOnStart` even more important. The binder will only throw when the options are actually created. Without validating on start, the exception would only surface on first access, not on startup.

## How did the pattern help

By using the options pattern, we've effectively addressed most of the issues listed earlier with `IConfiguration`:

We no longer introduce magic strings or hardcoded sections in our code. The section name is still a string, but it's now defined once as a constant field instead of being scattered through the code. Configuration is now accessed by injecting the interface we need. Writing validations for the configuration is now done in one place only, and can be run when the application starts.

Additionally, testing becomes much simpler by using `Options.Create(new EmailSettingsOptions());` instead of mocking `IConfiguration`. However, `Options.Create` only gives you an `IOptions<T>`. If you're using anything else, you'll still need a small fake or mock.

# Final thoughts

The options pattern in ASP.NET Core makes your code and configuration cleaner, and allows you to easily validate your configuration. My default setup at the beginning of a project is to use `AddOptions`, `ValidateDataAnnotations`, `ValidateOnStart`, and `IOptionsMonitor<TOptions>`. I revisit these choices as the project grows.

### References

[Microsoft Learn Documentation](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-10.0)
