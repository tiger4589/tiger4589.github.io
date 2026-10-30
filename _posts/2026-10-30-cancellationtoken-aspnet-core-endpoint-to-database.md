---
layout: post
title: "CancellationToken in ASP.NET Core: From Endpoint to Database"
date: 2026-10-30
tags: dotnet aspnet-core ef-core cancellation csharp aspnetcore minimal-apis
---

Accepting a `CancellationToken` at the endpoint is just the beginning.  
The real value comes from passing that token through every layer, so your application can stop downstream work as soon as the client disconnects.

This article shows how to do that end-to-end in ASP.NET Core with minimal APIs, and when you'd have a valid reason _not_ to do it.

<!--more-->

# Why cancellation matters

ASP.NET Core signals cancellation when the client disconnects: either they close the browser tab, or their network drops, or a reverse proxy in front of your app gives up and closes the connection. But what happens if your code ignores that signal?

- EF Core queries keep running.
- Outgoing HTTP calls keep running and holding sockets.
- Database connections stay checked out of the pool.
- Throughput under load degrades, because the server is busy doing work nobody will ever see.

Your application doesn't propagate cancellation out of the box. You have to pass the token explicitly.

# The flow

Here's the pipeline we'll build:

![Pipeline We'll Build](/assets/cancellation-token/cancellation-pipeline.png)

## Step 1 – The endpoint

Minimal APIs bind the request's cancellation token automatically. Simply add it to your parameters:

```csharp
app.MapGet("/get-dad-joke", async (DadJokeService dadJokeService, CancellationToken cancellationToken) =>
{
    var dadJoke = await dadJokeService.FetchAndStoreRandomDadJokeAsync(cancellationToken);
    return Results.Ok(dadJoke);
})
.WithName("GetDadJoke");
```

This token is `HttpContext.RequestAborted`. By default, it's canceled when the connection (or HTTP/2 stream) is aborted. That includes a client disconnecting and a reverse proxy dropping the connection after its own timeout.

Two things it does _not_ cover out of the box:

- **Request timeouts.** The token only fires on a timeout if you opt into the request timeouts middleware (.NET 8+). Note that the middleware doesn't enforce timeouts while a debugger is attached.
- **Server shutdown.** During a graceful shutdown, Kestrel lets in-flight requests finish. The token only fires for requests still running when the shutdown timeout expires and their connections are aborted.

If you want timeouts, add them explicitly:

```csharp
builder.Services.AddRequestTimeouts();

app.UseRequestTimeouts();

app.MapGet("/get-dad-joke", /* ... */)
   .WithRequestTimeout(TimeSpan.FromSeconds(10));
```

MVC controller actions work the same way: add a `CancellationToken` parameter to the action and it's bound automatically.

## Step 2 – The service layer

Your service should forward the token it received, not create its own.

This is where most teams accidentally drop the token. It compiles fine, because the parameter is optional:

```csharp
// The token stops here. The HTTP call runs to completion no matter what.
var response = await client.GetFromJsonAsync<DadJokeResponse>("/random/joke");
```

Pass it along instead:

```csharp
// The HTTP call is canceled if the client disconnects.
var response = await client.GetFromJsonAsync<DadJokeResponse>("/random/joke", cancellationToken);
```

## Step 3 – Calling the repository

Now imagine we want to store the joke in our own database so users can vote on it. The service fetches the joke, then hands it to the repository, passing the same token:

```csharp
public async Task<DadJokeResponse?> FetchAndStoreRandomDadJokeAsync(CancellationToken cancellationToken)
{
    var response = await client.GetFromJsonAsync<DadJokeResponse>("/random/joke", cancellationToken);

    var dadJoke = response?.Body?.FirstOrDefault();
    if (dadJoke is null)
    {
        return response;
    }

    var joke = new Joke
    {
        Id = dadJoke.Id,
        Setup = dadJoke.Setup,
        Punchline = dadJoke.Punchline
    };

    await jokeRepository.AddJokeAsync(joke, cancellationToken);

    return response;
}
```

## Step 4 – The repository

This repository uses EF Core:

```csharp
public async Task AddJokeAsync(Joke joke, CancellationToken cancellationToken)
{
    var alreadyExists = await dbContext.Jokes.AnyAsync(j => j.Id == joke.Id, cancellationToken);
    if (alreadyExists)
    {
        return;
    }

    dbContext.Jokes.Add(joke);
    await dbContext.SaveChangesAsync(cancellationToken);
}
```

When the token is canceled, EF Core asks the database driver to cancel the running command. However, keep in mind: cancellation is cooperative and best-effort, not instant. SQL Server's driver sends an attention signal, and Npgsql sends a cancel request over a separate connection. Both take time, and the server may finish the statement anyway.

For writes, there's one more nuance: if cancellation arrives after the commit has already been sent, the data can be persisted even though your code sees an `OperationCanceledException`.

### What about drivers that don't support cancellation?

Not every database provider implements real cancellation. Some older providers implement their async methods synchronously, which means the token has nothing to cancel.

When the driver doesn't respect the token:

- Your awaiting code may stop waiting, but the database operation keeps going.
- You can end up with "zombie queries" running long after the request is gone.

To limit the damage, add timeouts on the database side. The database enforces the timeout, regardless of what the client or driver does:

- Set a sensible `CommandTimeout` on your commands or `DbContext`.
- Use server-side limits, such as PostgreSQL's `statement_timeout` or MySQL's `max_execution_time`.
- If possible, switch to a provider that supports cancellation properly.

# What happens when cancellation occurs?

When the client disconnects:

- ASP.NET Core cancels the request token.
- Any awaited operation that respects the token throws an `OperationCanceledException` (sometimes the subclass, `TaskCanceledException`).
- The exception bubbles back up through your code and the middleware pipeline, just like any other exception. Nobody is waiting for the response, so nothing gets sent.
- Since .NET 8, the exception handler middleware recognizes this case: it logs it at Debug level instead of an error and sets the status code to 499 (Client Closed Request). This applies if you've registered `UseExceptionHandler`, and only if the response hasn't started yet.

This is exactly what you want: the work stops, and your error logs stay clean.

# Handling cancellation correctly

## Don't swallow `OperationCanceledException`

```csharp
try
{
    await jokeRepository.AddJokeAsync(joke, cancellationToken);
}
catch (OperationCanceledException)
{
    logger.LogInformation("Request was canceled.");
    // The caller now thinks the save succeeded and carries on with the next step(s).
}
```

Catching and discarding the exception in a lower layer is the dangerous part: the caller continues as if the work completed, and may even start more work for a request that no longer exists, and no one is waiting for its result.

## Rethrow it instead

```csharp
try
{
    await jokeRepository.AddJokeAsync(joke, cancellationToken);
}
catch (OperationCanceledException)
{
    logger.LogInformation("Request was canceled.");
    throw; // Let the cancellation propagate.
}
```

## Differentiate between cancellation and timeouts

When an `HttpClient` call exceeds `HttpClient.Timeout`, it throws a `TaskCanceledException`, which _is_ an `OperationCanceledException`, even though your token was never canceled. The log message would make it look as if the client disconnected, for what was really an `HttpClient` timeout.

An exception filter enforces the difference:

```csharp
catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
{
    logger.LogInformation("Request was canceled by the client.");
    throw;
}
```

Timeouts fall through to your normal error handling, where they belong.

# Adding your own timeouts

There's an important exception of the "don't create new tokens" rule: putting a time limit on a specific operation. In this case, create a linked token source so the operation is canceled by _either_ the client disconnecting _or_ the timeout:

```csharp
using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
cts.CancelAfter(TimeSpan.FromSeconds(5));

var response = await client.GetFromJsonAsync<DadJokeResponse>("/random/joke", cts.Token);
```

Always dispose the linked source (in this case, the `using` takes care of it), or it stays registered with the parent token.

# When _not_ to use the request token

Propagating the token everywhere is the right default, but there are some exceptions.

**Work that must complete.** For our dad joke, abandoning the save when the client leaves is fine. For a payment, it isn't:

```csharp
// Simplified Illustration - Not for production!

await paymentGateway.ChargeAsync(order, cancellationToken);

// The charge went through. Recording it must not be abandoned just because the client left.
await orderRepository.MarkAsPaidAsync(order.Id, CancellationToken.None);

// Simplified Illustration - Not for production!
```

For critical operations such as payments, cancellation requires more careful handling. An external system might process an operation successfully, even if your application receives a cancellation. You need to take that into account through idempotency, and reconciliation mechanisms. Once a side effect has been confirmed, subsequent operations required to maintain consistency should not be abandoned simply because the original HTTP request was canceled.

**Work that outlives the request.** If you queue background work or fire off a task that continues after the response is sent, never hand it the request token. That token belongs to a request that's about to end. Use the `stoppingToken` of your hosted service, or `IHostApplicationLifetime.ApplicationStopping`, so the work stops when the _application_ shuts down.

# Let the compiler help

You don't have to wait for a code review to catch dropped tokens. The built-in .NET analyzers do it for you:

- **CA2016** flags calls where you have a token available but don't forward it to a method that accepts one.
- **CA1068** enforces that `CancellationToken` parameters come last.

Raise them to warnings in your `.editorconfig`:

```ini
dotnet_diagnostic.CA2016.severity = warning
dotnet_diagnostic.CA1068.severity = warning
```

In your application code, also consider making `CancellationToken` a required parameter rather than making it optional by using `= default`. Then the compiler forces every caller to make an explicit choice.

**Important to know**: CA2016 has a limitation. It only analyzes methods that take a `CancellationToken` as their last parameter. CA1068 is more than a style recommendation here: following it allows CA2016 to recognize methods that should propagate cancellation.

# Testing cancellation

Test that your code respects the token. A pre-canceled token makes your test deterministic. A short timeout, even just for 1 ms, creates a race that will make the test flaky.

```csharp
[Fact]
public async Task AddJokeAsync_Throws_When_Token_Is_Canceled()
{
    var joke = new Joke { Id = "test-joke", Setup = "Setup", Punchline = "Punchline" };
    var canceledToken = new CancellationToken(canceled: true);

    await Assert.ThrowsAnyAsync<OperationCanceledException>(() =>
        _repository.AddJokeAsync(joke, canceledToken));
}
```

**Note**: `ThrowsAnyAsync` instead of `ThrowsAsync` is important. xUnit's `ThrowsAsync<T>` requires the _exact_ type, and EF Core and `HttpClient` might throw the `TaskCanceledException` subclass, so `ThrowsAsync<OperationCanceledException>` would fail even when cancellation works perfectly.

Keep in mind what this test proves: the repository respects the token.  
Proving that the database actually stops working requires an integration test against your real provider.

# Checklist: proper cancellation propagation

- Accept a `CancellationToken` in your endpoint.
- Pass it to every service and repository method.
- Pass it to EF Core, `HttpClient`, and similar.
- Don't swallow `OperationCanceledException`, rethrow it instead.
- Distinguish client cancellation from timeouts with an exception filter.
- Create new tokens only for a reason, like a linked token with a timeout.
- Use a separate token for work that must complete or that outlives the request.
- Enable CA2016 so the compiler catches dropped tokens.
- Test cancellation behavior, with a pre-canceled token and `ThrowsAnyAsync`.
- Check whether your database driver supports cancellation, and enforce server-side timeouts either way.

# Final thoughts

Cancellation is not just a nice-to-have. It's a performance feature, a resource-protection feature, and a correctness feature.

Minimal APIs make the token easy to get, but the responsibility stays the same: pass it all the way down to the database and every outgoing HTTP call. If you do that, your system stops wasting work on requests nobody is waiting for, stays predictable under load, and keeps its resources for the clients who are still there. Just remember the exceptions: some work has to finish, whether the client disconnected or not.
