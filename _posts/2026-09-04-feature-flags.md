---
layout: post
title: "Feature Management in .NET: Simple Flags to Advanced Targeting"
date: 2026-09-04
tags: csharp dotnet feature-management feature-flags aspnet-core a-b-testing configuration-management feature-targeting tutorial
---

Have you ever had to deploy code to production, but want to keep it inactive for the time being? I certainly had to do so for many reasons. Sometimes it was simply a feature that has to be enabled at a certain time, or a feature that is waiting for another team deployment that is scheduled for a few days later, but the idea remains the same, deploy the feature and keep it inactive until it's time.

The most common way to do so is by using feature flags. A simple configuration in the app can accomplish this, but in `.NET` we have a more powerful tool that can do more than enabling/disabling a feature.

Let's take a look together at what [Feature Management](https://github.com/microsoft/FeatureManagement-Dotnet) offers to make it easier to handle this.

<!--more-->

# What is .NET feature management?

It's an open source library that provides you with the abilities to develop and expose functionalities based on feature flags. It is built on top of the .NET Core configuration system, which means that any .NET core configuration provider is capable of providing the capabilities the library needs. .NET feature management can handle simple scenarios as turning a feature on or off dynamically, but it can also handle more complex scenarios such as:

- Activating a feature based on random percentage.
- Activating a feature based on a time window, which can also be configured to recur periodically.
- Activating a feature for targeted audiences.

# How to use?

You first need to install the `Microsoft.FeatureManagement` library in your project:

```shell
Install-Package Microsoft.FeatureManagement
```

Next, you'll have to register the required services with dependency injection:

```csharp
builder.Services.AddFeatureManagement();
```

To add a feature flag in your configuration, you can follow the fixed convention by using the [Microsoft Feature Management Schema](https://github.com/microsoft/FeatureManagement/blob/main/Schema/FeatureManagement.v2.0.0.schema.json):

```json
"feature_management": {
  "feature_flags": [
    {
      "id": "FeatureA",
      "enabled": false
    },
    {
      "id": "FeatureB",
      "enabled": true
    }
  ]
}
```

In your code, to access the feature manager API and check if a feature is enabled or not, you'll have to inject the `IFeatureManager` interface, and then you call the `IsEnabledAsync` method by providing the id. A good rule of thumb is to have your ids saved as constants. For example:

```csharp
public static class FeatureFlagsNames
{
    public const string FeatureA = "FeatureA";
    public const string FeatureB = "FeatureB";
}
```

and where you have to access the feature manager:

```csharp
//feature A:
var isFeatureAEnabled = await featureManager.IsEnabledAsync(FeatureFlagsNames.FeatureA);

//feature B:
var isFeatureBEnabled = await featureManager.IsEnabledAsync(FeatureFlagsNames.FeatureB);
```

This is all cool and everything, but let's see it in action. We'll be using the default weather controller endpoint that is added when you create a new API project with slight modifications.

## Scenario

The default API project comes with a weather controller and one get endpoint.

What we'll be doing is expanding the result object a little bit, assume that we'll introduce another "weather provider", look at how a random percentage-based works, add a time window feature, and finally check how a targeted feature works.

### WeatherForecast modification

We'll start by modifying `WeatherForecast` and add two new properties:

```csharp
public class WeatherForecast
{
    public DateOnly Date { get; set; }

    public int TemperatureC { get; set; }

    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);

    public string? Summary { get; set; }

    public string? Alert { get; set; }

    public required string Provider { get; set; }
}
```

and we'll add a feature flag in our `appsettings.json`:

```json
"feature_management": {
  "feature_flags": [
    {
      "id": "NewWeatherProvider",
      "enabled": false
    }
  ]
}
```

Then we're going to inject `IFeatureManager` in our `WeatherForecastController` and modify our `Get` method into:

```csharp
public async Task<IEnumerable<WeatherForecast>> Get()
{
    if (await featureManager.IsEnabledAsync(FeatureFlagsNames.NewWeatherProvider))
    {
        return GetFromNewWeatherProvider();
    }

    return GetFromDefaultWeatherProvider();
}
```

Honestly, for demonstration purposes, all I did with the new provider is change the `Provider` value, but the code remains the same. The objective here is to demonstrate how to use feature management.

### Simple scenario

After all these changes, if we keep our feature flag enabled property to false, no matter how many times we call the API, our result will remain the "same":

```json
{
  "date": "2026-09-05",
  "temperatureC": 31,
  "temperatureF": 87,
  "summary": "Bracing",
  "provider": "Weather API Default Provider"
}
```

If we change the enabled value to true, our new result would be:

```json
{
  "date": "2026-09-05",
  "temperatureC": 43,
  "temperatureF": 109,
  "summary": "Sweltering",
  "provider": "Amazing New Provider"
}
```

### Percentage based feature

Before we get into how to use percentage based feature, you might be wondering why would we want to randomly return data from different providers. The first reason that comes to mind is A/B testing. You can ask your users to rate the different providers to end up using one or the other. You can also use metrics such as conversion rate to choose which provider to keep.

We have to add the filter to our configuration:

```json
"feature_management": {
  "feature_flags": [
    {
      "id": "NewWeatherProvider",
      "enabled": true,
      "conditions": {
        "client_filters": [
          {
            "name": "Microsoft.Percentage",
            "parameters": {
              "Value": 50
            }
          }
        ]
      }
    }
  ]
}
```

What we're configuring here is that our feature will be enabled for 50% of the requests. This doesn't guarantee that every second request will be enabled, but it guarantees that 50% of all of our requests will be using the new weather provider.

And that's it, our core code doesn't need to change, and if we test this out, at least for 100 requests, we will probably end up with 50 results using default provider, and 50 results using the new provider.

### Time window based feature

Remember the `Alert` property we added? Let's assume we're going to use it when we know there's a huge storm coming in. Yes, I know, this isn't the best way to do it, but it kinda fits the scenario and explanation here!

Feature management allows you to enable a behavior during a specific time window.
Let's add a new feature to our configuration called `StormAlert`:

```json
{
  "id": "StormAlert",
  "enabled": true,
  "conditions": {
    "client_filters": [
      {
        "name": "Microsoft.TimeWindow",
        "parameters": {
          "Start": "Fri, 04 Sep 2026 00:00:00 GMT",
          "End": "Fri, 11 Sep 2026 00:00:00 GMT"
        }
      }
    ]
  }
}
```

If I call the endpoint between the start and the end, the alert will be added to the result:

```json
{
  "date": "2026-09-05",
  "temperatureC": 35,
  "temperatureF": 94,
  "summary": "Hot",
  "alert": "A severe storm is expected in your area. Please take necessary precautions.",
  "provider": "Weather API Default Provider"
}
```

### Targeted feature

The percentage and time window filters are added automatically when you use the `AddFeatureManagement`. However, if you want to use the `TargetingFilter`, you'd have to specifically add the filter with the `WithTargeting` method.

The reason behind this is because the targeting filter needs a context accessor. You can install The `Microsoft.FeatureManagement.AspNetCore` library which provides a default implementation of `ITargetingContextAccessor` that would extract the information from the request `HttpContext` value. It extracts the following:

- Targeting information from the `HttpContext.User` property
- `UserId` from the `Identity.Name`
- `Groups` from role claims

If you don't want to use the default implementation, then you'd have to provide a class that implements the interface `ITargetingContextAccessor` when calling `WithTargeting<>`.

In our case, we'll just go with the default implementation.

First, we need to install the library:

```shell
Install-Package Microsoft.FeatureManagement.AspNetCore
```

Then we have to update our service registrations:

```csharp
builder.Services.AddFeatureManagement().WithTargeting();
```

Then we'll have to update our configuration to change the new weather provider feature from a percentage filter to a target audience filter:

```json
{
  "id": "NewWeatherProvider",
  "enabled": true,
  "conditions": {
    "client_filters": [
      {
        "name": "Microsoft.Targeting",
        "parameters": {
          "Audience": {
            "Users": ["Paul"],
            "Groups": [
              {
                "Name": "Beta",
                "RolloutPercentage": 100
              },
              {
                "Name": "Premium",
                "RolloutPercentage": 50
              }
            ]
          }
        }
      }
    ]
  }
}
```

This configuration means that the user `Paul` will have the feature enabled for him.  
The group `Beta` will have 100% access to the feature.  
The group `Premium` would have 50% access.

For the demo, I added a middleware that would read the user id from the headers I send, and the role if needed.

```csharp
app.Use(async (context, next) =>
{
    var userId = context.Request.Headers["X-User-Id"].FirstOrDefault();
    var roles = context.Request.Headers["X-Roles"]
                    .FirstOrDefault()?
                    .Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries)
                ?? [];

    if (!string.IsNullOrEmpty(userId))
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.Name, userId)
        };

        claims.AddRange(
            roles.Select(role => new Claim(ClaimTypes.Role, role)));

        context.User = new ClaimsPrincipal(
            new ClaimsIdentity(claims, "Demo"));
    }

    await next();
});
```

**Don't use that for production! It's only for demo use!**

Now if I call the endpoint without adding anything to the header, it will always return the default provider.
But when I add `X-User-Id` header with the value `Paul`, I will get the new provider:

```json
{
  "date": "2026-09-05",
  "temperatureC": 27,
  "temperatureF": 80,
  "summary": "Balmy",
  "alert": null,
  "provider": "Amazing New Provider"
}
```

Any other user id would always return the default one without roles added.

If we add the `Beta` role to any different user, we'll always get the new provider, and if we use another user with the `Premium` role then we have a chance to be entered in the pool of the 50% users who have access to the new provider.

# Final thoughts

The Feature Management library is a powerful library to turn features on or off without deploying the code. It offers a lot of options which we covered, and some that we didn't cover.

For example you can add exclusions to the targeted audience which allows the users to opt-out of some features, the time window can add a recurring pattern, and you can create custom filters.

For diving more deeply check the references below.

# References

[.NET feature management docs](https://learn.microsoft.com/en-us/azure/azure-app-configuration/feature-management-dotnet-reference)  
[Feature Management GitHub repo](https://github.com/microsoft/FeatureManagement-Dotnet)
