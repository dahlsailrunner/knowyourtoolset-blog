---
title: "Using Serilog Logging for LaunchDarkly" # Title of the blog post.
date: 2023-04-08T05:51:55-05:00 # Date of post creation.
summary: "Configuring LaunchDarkly to use Serilog (via Microsoft.Extensions.Logging) for its Logging. Uses an IServiceProvider created from an IServiceCollection." # Description used for search engine.
codeMaxLines: 30 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: true # Override global value for showing of line numbers within code block.
thumbnail: "/images/ld-log-entry.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/ld-log-entry.png" # Designate a separate image for social media sharing.
toc: true
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - C#
  - ASP.NET  
  - LaunchDarkly
  - Serilog
---

## Background

I was recently trying to get [LaunchDarkly](https://launchdarkly.com/) to write its logs using our standard
logging configuration - which is [Serilog](https://serilog.net/) on
top of `Microsoft.Extensions.Logging`. According to the [LaunchDarkly docs](https://docs.launchdarkly.com/sdk/features/logging#net-server-side), this was pretty straight-forward but it turned out to be trickier than I thought.

The solution involved getting services from an `IServiceCollection` ***before*** the standard `IServiceProvider`
is built with the `builder.Build()` call in ASP.NET applications.

**tl;dr:** Just wanna see the working code?  It's here: [https://github.com/dahlsailrunner/launch-darkly-logging](https://github.com/dahlsailrunner/launch-darkly-logging)

## The Documentation

The issue was in the code being suggested:

```c#
using LaunchDarkly.Logging;
using LaunchDarkly.Sdk.Server;

var config = Configuration.Builder("sdk-key-123abc")
    .Logging(Logs.CoreLogging)
    .Build();
```

All of the above looks simple enough - but what is `Logs.CoreLogging`? There are two options for this argument:

* `ILogAdapter`: Defined by LaunchDarkly as a hook into things like Microsoft.Extensions.Logging
* `IComponentConfigurer<LoggingConfiguration>`: Also defined by LaunchDarkly for lower-level control

The `ILogAdapter` was the route to pursue - but how to get one of those?

There's another NuGet package (`LaunchDarkly.Logging.Microsoft`) which defined a static method that can
return an `ILogAdapter` if you provide an instance of the standard `ILoggerFactory` interface.

We're getting close now!!

With the `config` object from the above snippet of code, we need to register a singleton instance for
LaunchDarkly into the `IServiceCollection` as we're starting the application:

```c#
builder.Services.AddSingleton<ILdClient>(_ => new LdClient(ldConfig));
```

## The Problem

During application startup, you have code something like this:

```c#
var builder = WebApplication.CreateBuilder(args);
// register services with builder.Services.AddXXXXX() calls

builder.Services.AddSingleton<ILdClient>(_ => new LdClient(ldConfig));

var app = builder.Build(); // builds IServiceProvider from IServiceCollection (builder.Services)
```

We need to be able to use an `ILoggerFactory` instance to create the `ldConfig` variable above - but we
won't be building the `IServiceProvider` until later.

## The Solution

We can use an intermediate building of an `IServiceProvider` to do what we need. A simplified version of the
code above looks like this:

```c#
var builder = WebApplication.CreateBuilder(args);
// register services with builder.Services.AddXXXXX() calls

var sp = builder.Services.BuildServiceProvider(); 
var loggerFactory = sp.GetService<ILoggerFactory>();
// .. use loggerFactory when building ldConfig variable
builder.Services.AddSingleton<ILdClient>(_ => new LdClient(ldConfig));

var app = builder.Build(); // builds IServiceProvider from IServiceCollection (builder.Services)
```

{{% notice warning Caution %}}
The above code will work fine -- GREAT! -- but there's also a caution here. Calling `BuildServiceProvider`
will create instances of singletons that you've registered - and then the `app = builder.Build()` line
will do that **again**.  So definitely avoid registering singleton services before that added call.
{{% /notice %}}

## The Result

With the above code in place, I started getting great logs from the standard logging pipeline I had
set up with Serilog.  

Here's a sample entry shown in Seq:

![ld-log-entry](/images/ld-log-entry.png)

And the configuration of filtering by level was just as simple as it should be:

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Debug",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "System": "Warning",
        "LaunchDarkly": "Debug"
      }
    }
  }
}
```

The Github repo listed above has a fully working minimal API that shows all of this in action.
It's here: [https://github.com/dahlsailrunner/launch-darkly-logging](https://github.com/dahlsailrunner/launch-darkly-logging)

Happy coding!
