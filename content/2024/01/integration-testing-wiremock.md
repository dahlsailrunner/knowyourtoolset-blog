---
title: "Using WireMock in Integration Tests for ASP.NET Core APIs" 
date: 2024-01-29T01:51:55-05:00 
description: "Use WireMock to simulate responses from external APIs you call from your own .NET API project instead of actually calling them." 
codeMaxLines: 30 .
codeLineNumbers: true 
toc: true
tags:
  - C#
  - API
  - ASP.NET
  - Testing
  - XUnit
  - WireMock
---

If you have a reasonably-complex API project in ASP.NET, chances are pretty good that
it needs to call ***other*** APIs for some operations.  Having integration tests to
accommodate different behavior for those external API calls - without having to call
the *real* version of the external API can be super helpful.  Turns out this is easy
to do using the handy [WireMock.net NuGet package](https://github.com/WireMock-Net/WireMock.Net).

{{% notice tip "Show me the Code!" %}}
The code I used as a reference for this article is in the same repo
as the previous posts I did about integration testing, and the external API call and the
WireMock feature has been added in the
[4-api-with-postgres-and-auth](https://github.com/dahlsailrunner/testing-examples/tree/main/04-api-with-postgres-and-auth) repo.
{{% /notice %}}

## Application Code Calls an External API

I updated the simple little API to make an external API call using a [typed HTTP client](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-factory#typed-clients):

```c# {hl_lines=[6]}
public class ProductsController(LocalContext context, ProductValidator validator, IExternalApiClient apiClient) : ControllerBase
{
    [HttpGet]
    public async Task<IEnumerable<Product>> GetProducts(string category = "all")
    {
        var sampleClaims = await apiClient.GetSampleResult(HttpContext);

        return await context.Products
            .Where(p => p.Category == category || category == "all")
            .ToListAsync();
    }
    //...
}
```

The typed HTTP client code uses a demo/test API method available on the
[demo Duende IdentityServer](https://demo.duendesoftware.com/)
that would use a bearer token for authentication and
return a list of claims.  The entire code for the client is below:

```c# {hl_lines=[14]}
public record InternalClaim(string Type, string Value);

public interface IExternalApiClient
{
    Task<List<InternalClaim>> GetSampleResult(HttpContext ctx);
}

public class ExternalApiClient : IExternalApiClient
{
    private HttpClient Client { get; }

    public ExternalApiClient(HttpClient client, IConfiguration config)
    {
        client.BaseAddress = new Uri(config.GetValue<string>("ExternalApiBaseUrl")!);
        Client = client;
    }

    public async Task<List<InternalClaim>> GetSampleResult(HttpContext ctx)
    {
        var token = await ctx.GetTokenAsync("access_token");
        Client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
        var claims = await Client.GetFromJsonAsync<List<InternalClaim>>("api/test");
        return claims!;
    }
}
```

Note that the `ExternalApiBaseUrl` is read from configuration (line 14) -- meaning `appsettings.json`
in this case.

If you debug the API and use the `.http` file to execute the `GET v1/products` route, you
can see the claims returned from the API method.

But we want to make sure this works properly in our integration tests - that's the
real point of this article.  So read on!

## Using WireMock

In its simplest use case, the WireMock.Net library creates a simple http server that listens
for requests that you specify and sends responses that you also specify.

If you have any kind of `IClassFixture` in your integration test project, you are
already well-prepped to be able to easily use WireMock in your integration tests.

In the sample project I've been working with, that class is `DatabaseFixture`.

First, I added a NuGet package reference to WireMock.Net.

Then I created a public string property which will have the base address of the
WireMock server:

```c#
public string ExternalApiBaseUrlOverride { get; private set; } = null!;
```

Then in the `InitializeAsync` method I added this code:

```c# {hl_lines=[10]}
var server = WireMockServer.Start();

var claims = new List<InternalClaim>
{
    new("email", "hi@there.com"),
    new("role", "admin"),
    new("sub", "1234567890")
};

ExternalApiBaseUrlOverride = server.Url!;
server
    .Given(
        Request.Create().WithPath("/api/test").UsingGet()
    )
    .RespondWith(
        Response.Create()
            .WithStatusCode(200)
            .WithHeader("Content-Type", "application/json")
            .WithBody(JsonSerializer.Serialize(claims, new JsonSerializerOptions(JsonSerializerDefaults.Web)))
    );
```

The above code sets up a WireMock server, then creates a list of claims that I want to
serialize as the response.

It sets the `ExternalApiBaseUrlOverride` property to the dynamic URL that was generated
when the WireMock server started (line 10).

The next big block of `server.Given(...).RespondWith(...)` is where we say that the
`GET api/test` route should respond with the serialized payload of those hard-coded
claims.

### Override the `appsettings.json` Configuration

A final change needed to actually put this WireMock server in play for the tests
is to make sure the `ExternalApiBaseUrl` defined in `appsettings.json` for the API
gets overriden with the value from our `DatabaseFixture` class.

This is a very simple addition to the `CustomApiFactory.ConfigureWebHost` method:

```c#
builder.ConfigureAppConfiguration((_, configBuilder) =>
{
    configBuilder.AddInMemoryCollection(new Dictionary<string, string>
    {
        ["ExternalApiBaseUrl"] = dbFixture.ExternalApiBaseUrlOverride
    }!);
});
```

The above code will do exactly what we want - note the use of the `ExternalApiBaseUrlOverride`
property from the `DatabaseFixture` class.

Run / debug the tests and they should work fine!

There are ***many*** additional features and options for the WireMock library,
including:

- Delays in responses (e.g. simulate an API taking 5 seconds to respond)
- Fault or error responses for some percentage of the time
- Regular expression support for complex route / request mapping
- "Proxy mode" to record actual requests and responses for uses in tests or other scenarios later
- Dynamic configuration
- More!

Check the [WireMock.Net Wiki](https://github.com/WireMock-Net/WireMock.Net/wiki/What-Is-WireMock.Net)
for more information.

Keep testing!
