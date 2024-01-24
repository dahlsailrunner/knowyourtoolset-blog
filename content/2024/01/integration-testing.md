---
title: "Integration Testing for ASP.NET APIs (1/3) - Basics" 
date: 2024-01-18T05:51:55-05:00 
description: "Using WebApplicationFactory, XUnit fixtures, and Bogus data to perform automated integration tests against an ASP.NET Core API" 
codeMaxLines: 30 .
codeLineNumbers: true 
thumbnail: "/images/test-output.png" 
shareImage: "/images/test-output.png" 
toc: true
tags:
  - C#
  - API
  - ASP.NET
  - Testing
  - XUnit
  - Bogus  
---

{{% notice note "Where's the code?" %}}
A fully working example for this code is in the `01-simple-api`
folder of the GitHub repo with the examples for this series
of posts:  [https://github.com/dahlsailrunner/testing-examples/tree/main/01-simple-api](https://github.com/dahlsailrunner/testing-examples/tree/main/01-simple-api).
The tests are in the `SimpleApi.Tests` project (surprise!).
{{% /notice %}}

Having automated tests to ensure that your API is working
(and continues to work) as expected and designed is a huge
productivity boost and safety net for rapid change.

This article will be the first of a series of posts
that show different techniques that can be used that go
beyond simple unit tests where you test a single method
or class.

## Key Ingredients

Since the topic here is *integration tests,* we want to be
able to make API calls to our API methods and use a real
data store for the data that it will return.

Three (possibly four, depending on your situation) key ingredients
make for a super potent mix to make this type of testing easy,
predictable, and fast:

- `WebApplicationFactory` - a class provided by [Microsoft.AspNetCore.Mvc.Testing](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-8.0)
that lets us host an in-memory version of our API as well as swap out certain dependencies
when needed
- [XUnit `Fixture`](https://xunit.net/docs/shared-context) functionality - ability
to provide a way to share context among our test cases and a place where
we can set up test data
- [Bogus](https://github.com/bchavez/Bogus) - a library that we can use to generate
test data based on rules we provide.
- [TestContainers](https://dotnet.testcontainers.org/) - a way to create SQL Server
or Postgres (or other databases) that we need for our API.  Note that if you
use SQLite you can use an in-memory version without needing a TestContainer instance.

## Testing Mindset

A key concept regarding the use of automated tests to verify
the behavior of our API is ***focusing on the results of the API***.  Here are
some things that we should likely want to test (depending on whether the
concepts are present in your API):

- anonymous call behavior
- `GET` requests with and without parameters
- Routes that don't exist
- Error responses
- `POST` or `PUT` requests - successful and validation errors
- authentication and authorization logic
- pagination logic
- existence (or not) of swagger endpoint(s)
- cascading object behavior (e.g. proper handling of parent-child relationships)
- handling of different responses / performance / errors from calls to other APIs
- health check behavior

## Step One: Use XUnit and WebApplicationFactory

We'll start the journey with a super-simple API that expands only a
little on the "weather forecast" API that you get as a sample when
you create a new .NET 8 API.

I've added the following features that we'll want to test:

- `GET` method now requires a `postalCode` string parameter that is 5 digits
- Use of `ProblemDetails` for error responses
- Inclusion of the ASP.NET Core environment name in the results
- An invalid value for the `postalCode` parameter will return a `BadRequest (400)` with `ProblemDetails`
- The value of "error" for the `postalCode` will return an `InternalServerError (500)` with `ProblemDetails`

Our goal with the above is to establish a foundation where we can:

- Write tests that will call the API
- Overwrite the ASP.NET Core Environment variable to see something other
than "Development"
- Verify the returned content and response codes - both for successful and failed calls

### Create the Test Project

I created the test project by using the "XUnit Test Project (C#)"
in the standard .NET project templates.

Then you need to add a reference to the [Microsoft.AspNetCore.Mvc.Testing NuGet package](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.Testing).

Lastly you need to edit the `.csproj` file for the test project
and make the top-level project node use the "Web" project
type:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
```

{{% notice tip "Create \"Utilities\" Folder" %}}
A key focus for any test project should be ***readability
of the tests***.  That means any "plumbing" code we have
to set up tests or the project should be away from the
test code.  I choose to put that in a `Utilities` folder.
By doing so, all of the top-level files in the test
project contain actual tests.

I moved the `GlobalUsings.cs` file into this folder.
{{% /notice %}}

### Create Tests Using WebApplicationFactory

Once you've got the test project created, it's time to
create tests.

{{% notice info "Code for First Tests" %}}
The code for the first tests is in [the WeatherForecast.cs file](https://github.com/dahlsailrunner/testing-examples/blob/main/01-simple-api/SimpleApi.Tests/WeatherForecast.cs).
{{% /notice %}}

Here's a starter class with a single test for the "happy
path" of when the `GET` method is called with a valid `postalCode`
parameter:

```c#
public class WeatherForecast(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly List<string> _possibleSummaries =
        ["Freezing", "Bracing", "Chilly", "Cool", "Mild",
         "Warm", "Balmy", "Hot", "Sweltering", "Scorching"];

    [Fact]
    public async Task HappyPathReturnsGoodData()
    {
        var client = factory.CreateClient();

        var forecastResult = await client.GetFromJsonAsync<ForecastAndEnv>(
            "/weatherForecast?postalCode=12345");

        Assert.Equal("Development", forecastResult?.Environment);
        Assert.Equal(5, forecastResult?.Forecast.Length);
        foreach (var forecast in forecastResult?.Forecast!)
        {
            Assert.Equal(forecast.TemperatureF, 32 + (int)(forecast.TemperatureC / 0.5556));
            Assert.Contains(forecast.Summary, _possibleSummaries);
        }
    }
}
```

Here are some notes about the code above.

The class declaration is this:

```c#
public class WeatherForecast(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
```

It uses the new primary constructor syntax to get a `WebApplicationFactory` of
type `Program` via dependency injection and is using the `IClassFixture` interface
from XUnit.

To enable this to work properly, I needed to add the following line to `Program.cs`
of the API project itself:

```c#
public partial class Program { } // needed for integration tests
```

With those elements in place, you can create a client that can
call your API with the following line:

```c#
var client = factory.CreateClient();
```

To actually make an API call, we can do a line like the following (from the code above):

```c#
var forecastResult = await client.GetFromJsonAsync<ForecastAndEnv>(
            "/weatherForecast?postalCode=12345");
```

This uses the built-in extenstion method from `System.Net.Http` to
deserialize the response and verify a successful status.

Note that we can't actually see what status is being verified and if
the deserialization doesn't work we'll simply get an exception. We'll
address these shortcomings soon.  :)

Then we can do whatever assertions we want against the results. In the
code above, I'm checking the environment value, and the contents of
the 5-element array.

Here's another test:

```c#
[Fact]
public async Task MissingPostalCodeReturnsBadRequest()
{
    var client = factory.CreateClient();

    var response = await client.GetAsync("/weatherForecast");

    var problemDetails = await response.Content.ReadFromJsonAsync<ProblemDetails>();

    Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);

    Assert.NotNull(problemDetails);
    Assert.Equal("Postal Code is required.", problemDetails.Detail);
    Assert.Equal(400, problemDetails.Status);
}
```

Note that in this test - we're *expecting* a status of `400 Bad Request` and
so we can't use the handy deserialization method from the previous test. We
get the raw response and then have to evaluate status and handle deserialization
inside the test code.

## Step Two: Add HttpClient Extensions and a Base Class

To simplify our the calls to our API, where we want to both check various
HTTP status codes as well as the returned JSON content, we'd like to have
a simple helper method that would enable calls like these:

```c#
var forecastResult = await Client.GetJsonResultAsync<ForecastAndEnv>(
    "/weatherForecast?postalCode=12345", HttpStatusCode.OK);

var problemDetails = await Client.GetJsonResultAsync<ProblemDetails>("/weatherForecast",
    HttpStatusCode.BadRequest);
```

Note in both cases we're passing arguments that clarify the `HttpStatusCode`
we're expecting from a given call.

Here's a method that does what we need and also supports outputing
the JSON content to the test output if something was amiss with deserialization
(check [the full source code for the class on GitHub](https://github.com/dahlsailrunner/testing-examples/blob/main/01-simple-api/SimpleApi.Tests/Utilities/HttpClientExtensions.cs)):

```c#
public static async Task<T> GetJsonResultAsync<T>(this HttpClient client, string uri,
    HttpStatusCode expectedHttpStatus, ITestOutputHelper? output = null)
{
    var response = await client.GetAsync(uri);
    Assert.Equal(expectedHttpStatus, response.StatusCode);
    var stringContent = await response.Content.ReadAsStringAsync();
    try
    {
        var result = JsonSerializer.Deserialize<T>(stringContent, JsonWebOptions);
        Assert.NotNull(result);
        return result;
    }
    catch (Exception)
    {
        if (output != null) WriteJsonMessage(stringContent, output);
        throw;
    }
}
```

We can put the above code into a static class in the `Utilities` folder
to avoid distracting it with our tests.

Additionally, we can create a simple base class to avoid the `factory.CreateClient()` call
repetitions.

That class is as simple as this:

```c#
public class BaseTest : IClassFixture<WebApplicationFactory<Program>>
{
    protected HttpClient Client { get; }

    protected BaseTest(WebApplicationFactory<Program> factory)
    {
        Client = factory.CreateClient();
    }
}
```

And now the class declaration for the `BetterWeatherForecast` looks like this:

```c#
public class BetterWeatherForecast(WebApplicationFactory<Program> factory, ITestOutputHelper output)
    : BaseTest(factory)
```

Even better - the test that we need to write for a `BadRequest` response now
becomes much more terse and readable (compare these **7** lines to the **13** in the
other `MissingPostalCodeReturnsBadRequest` test above):

```c#
[Fact]
public async Task MissingPostalCodeReturnsBadRequest()
{
    var problemDetails = await Client.GetJsonResultAsync<ProblemDetails>("/weatherForecast",
        HttpStatusCode.BadRequest);

    Assert.Equal("Postal Code is required.", problemDetails.Detail);
    Assert.Equal(400, problemDetails.Status);
}
```

Getting better!

### Using ITestOutputHelper

There is a skipped test in the `BetterWeatherForecast` set of tests:

```c#
//[Fact]
[Fact(Skip = "Run this when you want to see the output helper in action")]
public async Task ShowOutputHelperWhenTestFails()
{
    var problemDetails = await Client.GetJsonResultAsync<WeatherForecast>("/weatherForecast",
        HttpStatusCode.BadRequest, output);

    // this test fails since I'm trying to deserialize a WeatherForecast instead of
    // a ProblemDetails but if you look at the test output you'll see the JSON response
    // at the bottom of the output (under the exception)
}
```

The comment explains the reason it's skipped (it would be a failing test), but
running it shows how you can use the `ITestOutputHelper` to see what actually
was returned by the API in the event that it didn't match your expectation
(just by using its `WriteLine` method).

Here's a screenshot of the results when the test is included / run:
![Using ITestOutputHelper](/images/test-output.png)

Code for the method that created the above output is in the [HttpClientExensions](https://github.com/dahlsailrunner/testing-examples/blob/5056a5464ec696572bc6246372b93548fe4138c7/01-simple-api/SimpleApi.Tests/Utilities/HttpClientExtensions.cs#L34).

## Step Three: Use a Custom WebApplicationFactory

In many of our tests we will want to adjust the behavior or services of the
API to support our tests. Using a custom (inherited) version of the `WebApplicationFactory`
class can let us do that.

Here's a simple one:

```c#
public class CustomApiFactory<TProgram> : WebApplicationFactory<TProgram>
    where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("test");
    }
}
```

This class provides an override for the `ConfigureWebHost` method, and
inside it updates the `Environment` to be "test" instead of the "Development"
value that is set by the existing `launchSettings.json` file.

Note this line from `BetterWeatherForecast.cs`:

```c#
Assert.Equal("Development", forecastResult.Environment);
```

This is part of a passing test that uses `WebApplicationFactory`.

A new test class called `CustomWeatherForecast` can be created that has
the following declaration:

```c#
public class CustomWeatherForecast(CustomApiFactory<Program> factory, ITestOutputHelper output)
    : BaseTest(factory), IClassFixture<CustomApiFactory<Program>>
```

Note that it can use the same (unchanged!) base class introduced with the
`BetterWeatherForecast`.  But this is using our new `CustomApiFactory<Program>`,
so when we get a client that will call this version of the API, it should
be running with an environment of "test" (note line 7 of the following code):

```c#
[Fact]
public async Task HappyPathReturnsGoodData()
{
    var forecastResult = await Client.GetJsonResultAsync<ForecastAndEnv>(
        "/weatherForecast?postalCode=12345", HttpStatusCode.OK);

    Assert.Equal("test", forecastResult.Environment);
    Assert.Equal(5, forecastResult.Forecast.Length);
    foreach (var forecast in forecastResult.Forecast)
    {
        Assert.Equal(forecast.TemperatureF, 32 + (int)(forecast.TemperatureC / 0.5556));
        Assert.Contains(forecast.Summary, _possibleSummaries);
    }
}
```

## Next Up... Data

This post established some basic foundations for integration tests in ASP.NET Core
APIs.  In the [next post](../integration-testing-data), we'll build on top of these concepts and add some data
to our APIs with both SQLite and Postgres and see how we can create integration
tests for that. The exact same techniques as Postgres could be used for SQL Server.

Stay tuned!
