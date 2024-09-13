---
title: "Using Shared Logging Levels with .NET Aspire" 
date: 2024-09-13T05:11:55-05:00 
description: "Setting logging levels during local development for your entire distributed application easily with .NET Aspire." 
codeMaxLines: 30 .
codeLineNumbers: true 
toc: true
tags:
  - Aspire  
  - ASP.NET
  - Logging
---

I recently was consolidating quite a few pieces of a microservices application
into a single Aspire solution (which I highly recommend, by the way, and if you
need guidance for sure check out [the official docs](https://learn.microsoft.com/en-us/dotnet/aspire/get-started/aspire-overview)
but I also have [a Pluralsight course](https://bit.ly/ps-aspire) that can help).

While doing the setup, I came up with a technique to define some shared logging
levels that can be applied to your ASP.NET Core projects that I found pretty helpful
and wanted to share it here in this short post.

{{% notice tip "Just Show Me the Code!" %}}
A sample of code that shows the approach in this article is in [the GitHub repo
tied to my Pluralsight course](https://github.com/dahlsailrunner/cloud-native-dotnet/tree/main/aspire)
-- it uses the Serilog-based approach for environment variables.
{{% /notice %}}

## Logging Levels in ASP.NET

For ASP.NET applications, you often configure log levels for applications with values
in an `appsettings.json` file with content that might look like this:

```json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "System": "Information",
    "Microsoft": "Warning",
    "Microsoft.Hosting": "Information",
    "Duende": "Warning"
  }
}
```

{{% notice info "What About Serilog?" %}}
In Serilog, the same settings are possible but the `appsettings.json` syntax is a little
different:  

```json
"Serilog": {
  "MinimumLevel": {
    "Default": "Information",
    "Override": {
      "System": "Warning",
      "Microsoft": "Warning",
      "Microsoft.Hosting": "Information",
      "Duende": "Warning"
    }
  }
}
```

{{% /notice %}}

This configuration exists for ***each*** of your ASP.NET applications - and if you
have a microservices-based one like I did, this might be 10 or more different
applications.

If you need to use different log levels, like more `Debug` levels, you would end
up needing to change content in many different `appsettings.json` files.

## AppHost Modification in .NET Aspire

Two .NET features come to our assistance here:

* the chained / override feature where environment variables override appsettings files
* the ability to easily set environment variables for ASP.NET projects in the Aspire `AppHost`

You can set an environment variable on a Project in the AppHost with code such as this:

```c#
var api = builder.AddProject<Projects.CatalogApi>("catalog")
    .WithEnvironment("SomeEnvVar", "some value");
```

The environment-variable based name for the logging level setup from above for the `Default` log level would
be:

`Logging__LogLevel__Default`

Note the double-underscores that separate the levels of the json tree.

### The Code - An Extension Method and Single-Line AppHost Updates Per Project

Based on the above, we can define an extension method that looks like this - and I've got this
defined in the `AppHost` project:

```c#
internal static class LoggingHelper
{
    internal static IResourceBuilder<T> WithSharedLoggingLevels<T>(this IResourceBuilder<T> builder) 
        where T : IResourceWithEnvironment
    {
        var dict = new Dictionary<string, string>
        {
            { "Default", "Information" },
            { "System", "Warning" },
            { "Microsoft", "Warning" },
            { "Microsoft.Hosting", "Information" },
            { "Duende": "Warning" }
        };

        foreach (var item in dict.Keys)
        {
            builder = builder.WithEnvironment($"Logging__LogLevel__{item}", dict[item]);
        }
        return builder;
    }
}
```

And then on any project we add to the `AppHost`, we can simply invoke the method:

```c#
var api = builder.AddProject<Projects.CatalogApi>("catalog")
    .WithSharedLoggingLevels();
```

## EF Core -- See Queries in Logs

If you add the following settings, you can see the queries that you're executing in the log entries:

```c#
{ "Microsoft.EntityFrameworkCore.Database.Command", "Information" },
{ "Microsoft.EntityFrameworkCore.Query", "Information" },
{ "Microsoft.EntityFrameworkCore.Update", "Information" },
```

These will enable you to see the queries that EF Core executes the database even if the trace
information does not include it.

## Implications & Limitations

### Implemented Approach: Only for Local Development

Because I implemented the extension method in the `AppHost` project, it ***only*** applies to
local development.  If you wanted this kind of behavior more generally in any deployed environment,
you could add a similar (but not the same) method in the `ServiceDefaults` project and reference
shared configuration in some other way -- shared file, hard-coded values, or whatever (even a database!).

### Overriding an Individual ASP.NET Project

The implemented approach here will use the shared logging level in the AppHost project
**instead of** any configured values in the local `appsettings.json` file for the projects due
to the fact that environment variables take a higher priority in the default configuration
hierarchy.

The easiest way to set a custom level of logging for an individual ASP.NET project in your
Aspire solution is to simply **comment out the `WithSharedLoggingLevels()` call** for that
project, and then update the `appsettings.json` file for the project with whatever settings
you need.
