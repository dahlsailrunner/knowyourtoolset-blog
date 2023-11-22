---
title: "Logging Updates in .NET 8 and ASP.NET 8" # Title of the blog post.
date: 2023-11-21T05:51:55-05:00 # Date of post creation.
description: "Some nice updates to logging within ASP.NET 8." # Description used for search engine.
codeMaxLines: 30 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: true # Override global value for showing of line numbers within code block.
toc: true
tags:
  - C#
  - API
  - ASP.NET
  - Logging  
---

The [recent release of .NET 8](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-8?WT.mc_id=MVP_324326)
has occurred and as I was updating [my Pluralsight course on Logging and Monitoring in ASP.NET Core](https://www.pluralsight.com/courses/logging-monitoring-aspdotnet-core-6),
I thought it would also be useful to write a post about some of the nice new goodness related to
logging and monitoring.

Everything that you had in place for recent versions of .NET and ASP.NET Core apps and APIs will continue
to work fine, but there are a couple of nice new things specifically worth mentioning that can make things
easy for you.

If your curiosity is piqued by the content below, I elaborate more completely in my updated course under the link above.

Check it out!

## `ProblemDetails` Support is Easily Available

For API projects it's often very useful to return a `ProblemDetails` response when errors occur rather
than a custom object of yours or using the default exception handler (which returns an HTML page).

Doing this with prior releases of ASP.NET Core required either writing your own middleware or else
using a NuGet package like the very helpful Hellang.Middleware.ProblemDetails one.

Now with ASP.NET Core 8 there is built-in functionality just waiting for you to enable. The code
from a sample `Program.cs` that follows will respond to API errors with a `ProblemDetails` object
using sensible defaults, while logging the full details of the error to whatever logging provider
you are using. No extra code or NuGet packages required!

```c# 
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddProblemDetails();  // THIS LINE

var app = builder.Build();

app.UseExceptionHandler();  // AND THIS LINE

app.MapControllers();
app.Run();
```

Here is a sample response (the `traceId` is included in log entries for easy lookup):

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.6.1",
  "title": "An error occurred while processing your request.",
  "status": 500,
  "traceId": "00-f8cd643acbbc4d744e635200c3d9a098-451359bfdcf43482-01"
}
```

Customization of the `ProblemDetails` object and the actual exception handling is also possible.  See
[the official Microsoft docs on the topic](https://bit.ly/aspnetcore-problemdetails) for more information.

## New `LoggerMessage` Source Generator Overloads

Another nice new feature - yes, it's small, but also handy - is new overloads for the `[LoggerMessage]`
source generator which offer more options and require less parameters.

In previous versions of .NET, you would have to include an `EventId`, a `LogLevel`, and the `MessageTemplate`.

You would have code that looks like this:

```C#
[LoggerMessage(0, LogLevel.Warning, "API failure: {fullPath} Response: {statusCode}, Trace: {traceId}")]
partial void LogApiFailure(string fullPath, int statusCode, string traceId);
```

Note that in the above example, I'm using `0` as the EventId - it would get logged as such but I'm actually
not using that value at all.

Now with .NET 8, there are more overloads and `EventId` is no longer required.  So the code is slightly
simpler:

```C#
[LoggerMessage(LogLevel.Warning, "API failure: {fullPath} Response: {statusCode}, Trace: {traceId}")]
partial void LogApiFailure(string fullPath, int statusCode, string traceId);
```

Subtle, and maybe not a huge deal for you, but eliminating a parameter that is not used and can
possibly raise questions is always a good thing in my book.

Other possibilities exist where you can pass the LogLevel as a method parameter instead of
in the attribute contstructor as well.  

Once again, explore
[the official docs](https://learn.microsoft.com/en-us/dotnet/core/extensions/logger-message-generator?WT.mc_id=MVP_324326)
if you'd like to see more of this.

## Redaction of Sensitive Data

Regarding the redaction of sensitive data when writing log entries, there's never been a super easy answer. It's mostly
been on us as developers to make sure that sensitive data isn't getting in to log entries and there hasn't been any
consistent way of making that happen.  

> When logging an object that includes a date of birth, social security number, or other personally identifiable
> information (PII), we would want to mask, completely erase, or hash the value for security purposes.

It looks like this issue is getting some real attention within Microsoft, and they've published a NuGet package
called [Microsoft.Extensions.Compliance.Redaction](https://github.com/dotnet/extensions/tree/main/src/Libraries/Microsoft.Extensions.Compliance.Redaction)
to help with this.

It's early even for real docs on this, but there's a sample program that shows it in action here:

[https://github.com/bitbonk/LogRedactionDemo/blob/main/LogRedactionDemo.SimpleWorker/Program.cs](https://github.com/bitbonk/LogRedactionDemo/blob/main/LogRedactionDemo.SimpleWorker/Program.cs)

## Aspire Dashboard for Composite Applications

A lot was discussed regarding [Aspire and the new dashboard that it provides](https://devblogs.microsoft.com/dotnet/introducing-dotnet-aspire-simplifying-cloud-native-development-with-dotnet-8/?WT.mc_id=MVP_324326)
to simplify local development for composite applications targeting the cloud - often micro-service-style applications,
with different projects for the UI, API (maybe more than one API), and possibly other services.

This new dashboard leverages logs and Open Telemetry messages and looks really promising!

Definitely worth checking out for your local development and trying this out.