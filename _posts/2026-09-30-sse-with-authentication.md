---
layout: post
title: "SSE And Authentication in .NET"
date: 2026-09-30
tags: server-sent-events sse real-time authentication slt short-lived-tokens cookies http-client auth-headers
---

After a talk I gave about the difference between `SSE` and `SignalR`, and how to implement user targeted messages in both of the tools I was asked a very important question that I haven't explained during the session. "Can a SSE be authenticated? (does it have Authorization headers)". Since `SSE` is simply a way to stream events to the clients, via a normal HTTP request, the short and quick answer is yes. The longer answer is dependant on what client/API you're using. In this article, I will be explaining the different ways to use `SSE` with authenticated users, what are the pitfalls that you might encounter, and how to avoid them.

<!--more-->

# EventSource

In the example that I shared during the session, I was using `EventSource` to connect to my `SSE` endpoint. But here's the issue: Event Source doesn't support sending headers. So what's the alternatives? How can we authenticate our client using `EventSource`?

## Cookies

The good thing is that `EventSource` automatically sends cookies when connecting to an SSE endpoint, provided the browser's cookie and security rule allow them.

### Same-Origin

If your client application and API are on the same origin, and the browser already has an authentication cookie, then it will automatically include it in the SSE request.  
Meanwhile, your `ASP.NET` Core endpoint can then use the normal authentication middleware. The important part is that SSE does't bypass the authentication middleware. As long as you have configured the endpoint to have an authorized client, for example `.RequireAuthorization` in minimal API, that should be enough.

### Cross-Origin

If your client application and API are't on the same origin, such as:

- Front-End: https://app.example.com
- Back-End: https://api.example.com

Then you need to explicitly tell `EventSource` to include credentials.

This can be done using `withCredentials: true`:

```javascript
const events = new EventSource("https://api.example.com/api/events", {
  withCredentials: true,
});
```

And of course, the cookie must be valid for the API domain.

In this case, you also need to allow credentials on your server's CORS configuration:

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("frontend", policy =>
    {
        policy
            .WithOrigins("https://app.example.com")
            .AllowCredentials()
            .AllowAnyHeader()
            .AllowAnyMethod();
    });
});
```

Important note here: **Do NOT use `AllowAnyOrigin` together with `AllowCredentials()`.**

## Short-Lived Tokens

### Scenario

Before I explain what are short-lived tokens, and why to use them. Let me explain the demo that we're going to use.

We will have a newly created `Aspire` app with two projects and one component:

- Backend project
- Frontend project
- Keycloak component

Since we'll be using keycloak for authentication, then we're not going to have any cookies saved regarding the API.

### Then what?

What if I am not using cookies? What can I do?

In this case, we can fall-back to short-lived tokens. A short-lived token is a token that you can request from the server, that will only be valid for a very short time. But why? Even though `EventSource` doesn't support headers, we can still pass query parameters.

Passing our access token is not safe, it can be picked up by intercepting it, or it might even get logged somewhere. We don't want that to happen.

We'll implement an endpoint that an authenticated user can reach to request a short-lived token. Then we'll pass that `SLT` as a query parameter to our SSE endpoint. Even if it got intercepted, or logged, this token won't be active after a configurable amount of time.

First, let's start by implementing an in-memory store for the tokens:

Then we'll use an endpoint so authenticated users can request a short lived token:

Then we'll expose an endpoint that accepts an slt for authentication and stream events:

# Final thoughts
