---
title: "Integration Testing for ASP.NET APIs (3/3) - Auth" 
date: 2024-01-25T05:51:55-05:00 
description: "Using some custom authentication middleware to perform automated integration tests against an ASP.NET Core API" 
codeMaxLines: 30 .
codeLineNumbers: true 
toc: true
tags:
  - C#
  - API
  - ASP.NET
  - Testing
  - XUnit
  - OAuth2  
  - OIDC
---

{{% notice note "Where's the code?" %}}
Fully working examples for this code are in the [04-api-with-postgres-and-auth](https://github.com/dahlsailrunner/testing-examples/tree/main/04-api-with-postgres-and-auth)
folder of the GitHub repo with the examples for this series
of posts.
{{% /notice %}}

In the [previous post](../integration-testing-data) we got started
with some basic integration tests.  In this one we'll require
authentication in the API and write tests to ensure it's working.

## API Changes

In the API I added JWT bearer token (OAuth2) authentication
and am requiring authenticated callers for both the
`WeatherForecast` and `Products` routes.

Most of that code is right in `Program.cs` for the API:

```c#
JwtSecurityTokenHandler.DefaultMapInboundClaims = false;
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://demo.duendesoftware.com";
        options.Audience = "api";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            NameClaimType = "email",
            RoleClaimType = "role"
        };
        options.SaveToken = true;
    });
    
    //...
    app.UseAuthentication();
    app.UseSerilogRequestLogging();
    app.UseAuthorization();

    app.MapControllers().RequireAuthorization();

    app.MapGet("/weatherforecast", [Authorize]
    //...
```

I'm accepting JWT bearer tokens from the [demo Duende Identity
Server](https://demo.duendesoftware.com/) (super helpful for demos and simple testing btw).

Then note the `RequireAuthorization()` method on the end of
`MapControllers()` as well as the `[Authorize]` attribute
on the `MapGet("/weatherforecast")` call.  Those lines
will require us to have an authenticated caller for all
of the API methods.

To verify this, try the `.http` file when the API is running.

It has a `POST` call at the top to get an access (bearer)
token from the identity server, and then you can paste
the result into the value for the `token` variable.

Make the calls with and without this bearer token - if you
don't have a valid one you should get `401 Unauthorized`
responses and the calls should work normally if you provide
a valid token.

One other nuance is that I added the following attribute
to the `POST Products` method:

```c#
[Authorize(Roles = "admin")]
```

This goes further than simply requiring an authenticated
caller and ALSO requires that they belong to the `admin`
role in order to create a product.

If you wanted to test this in real code from the identity
server you would need to implement some "claims transformation"
logic.

Fortunately, we can use another technique to perform
integration tests against our simple auth logic that
completely bypasses the identity server (keeping
these tests fast).

## `IStartupFilter` with Custom Middleware

We can customize the `WebApplicationFactory` using
the techniques we described in the previous posts
of this series.

To add some custom "auto-authentication" logic, you
can have code something like this:

```c# {hl_lines=[10]}
public class CustomApiFactory<TProgram>(DatabaseFixture dbFixture) 
    : WebApplicationFactory<TProgram> where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        //... other logic 

        builder.ConfigureServices(services => 
        {
            services.AddSingleton<IStartupFilter>(new AutoAuthorizeStartupFilter());
        });
    }
}
```

Note the highlighted addition of the `IStartupFilter`.

The `AutoAuthorizeStartupFilter` is something that we
need to provide code for - and this is where we can include
some custom logic.

{{% notice info "Credit Where Credit is Due" %}}
The `AutoAuthorizeStartupFilter` described here was something
I discovered while exploring the new [Aspire version of the
eShop reference architecture](https://github.com/dotnet/eShop) - in the `tests/Ordering.FunctionalTests` project.  Thanks, eShop team!  :)
{{% /notice %}}

All of the code for the authorization logic is in the
`Utilities/AuthorizationHelper.cs` file in the code repo.

```c#
internal class AutoAuthorizeStartupFilter : IStartupFilter
{
    public Action<IApplicationBuilder> Configure(Action<IApplicationBuilder> next)
    {
        return builder =>
        {
            builder.UseMiddleware<AutoAuthorizeMiddleware>();
            next(builder);
        };
    }
}
internal class AutoAuthorizeMiddleware(RequestDelegate rd)
{
    public async Task Invoke(HttpContext httpContext)
    {
        var identity = new ClaimsIdentity("Bearer");

        identity.AddClaim(new Claim("sub", "1234567"));
        identity.AddClaim(new Claim(ClaimTypes.Name, "test-name"));

        httpContext.User.AddIdentity(identity);
        await rd.Invoke(httpContext);
    }
}
```

The `AutoAuthorizeStartupFilter` is what we referenced
in the `CustomApiFactory` code -- that adds some middleware
called `AutoAuthorizeMiddleware` which is where the real
logic comes into play.

I think you'll appreciate that this code is pretty simple
and can be what you want it to be.  It creates a `ClaimsIdentity`,
adds some claims to it, and then adds the created identity to
the `User` of the `HttpContext`.  Voila!

Any tests that uses the `CustomApiFactory` will have
an authenticated user with the claims specified in the code
above - you don't even need to adjust the tests!

### Testing Anonymous Users

When you add authentication requirements to your app, you
should make sure that if you don't have an authenticated
user that the caller is denied access to the resource in
question.

That means we need to have a different `CustomApiFactory`
that ***doesn't*** include this auto-authorize middleware.

To achieve this, I created a new class called
`AnonymousApiFactory` and also a static class
called `ApiFactoryExtensions`.

Here's the `AnonymousApiFactory` code:

```c#
public class AnonymousApiFactory<TProgram>(DatabaseFixture dbFixture) : WebApplicationFactory<TProgram>
    where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("test");
        builder.SwapDatabase(dbFixture.TestConnectionString);
    }
}
```

The `SwapDatabase` method is something I put in the
`ApiFactoryExtensions` class and the code is something I
described in the previous post to use a TestContainer for
Postgres instead of the normal Postgres instance used by
the API.

Of note is the ***absence*** of the addition of the
`IStartupFilter` here.  But the rest of this code
is the same.

Then with this `AnonymousApiFactory`, we can use that
in some tests to make sure that anonymous access to the
API methods is not allowed:

```c#
[Collection("IntegrationTests")]
public class AnonymousTests(AnonymousApiFactory<Program> factory, ITestOutputHelper output)
    : BaseTest(factory, output), IClassFixture<AnonymousApiFactory<Program>>
{
    [Fact]
    public async Task AnonymousGetWeatherShouldReturnUnauthorized()
    {
        var response = await Client.GetAsync("/weatherForecast?postalCode=12345");
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
```

Pretty straight-forward - note the primary constructor
parameter for the `AnonymousApiFactory<Program>`.

## Some Code Cleanup

Since we have an `AnonymousApiFactory` that has most of
the code we want, we can update our `CustomApiFactory` to
inherit from that, and then include the custom `IStartupFilter`:

```c# {hl_lines=[6]}
public class CustomApiFactory<TProgram>(DatabaseFixture dbFixture) : AnonymousApiFactory<TProgram>(dbFixture)
    where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        base.ConfigureWebHost(builder);

        builder.ConfigureServices(services => 
        {
            services.AddSingleton<IStartupFilter>(new AutoAuthorizeStartupFilter());
        });
    }
}
```

The important line here is the one that calls
`base.ConfigureWebHost` - that will include the logic from
the `AnonymousApiFactory` here and then after that adds
in the `IStartupFilter`.

## Testing Authorization (Roles, etc)

The `POST Products` method has that `[Authorize(Roles="admin")]`
attribute that we should test.

And we would want to test both positive and negative cases here.

To accommodate these tests, I updated the
`AutoAuthorizeMiddleware` to look for some specific HTTP
headers in the request (remember that this is for our
*tests*, not in the real API code) and creating claims
based on those http headers.

Here's the code (in `AuthorizationHelper.cs`):

```c#
private static IEnumerable<Claim> GetClaimsBasedOnHttpHeaders(HttpContext context)
{
    const string headerPrefix = "X-Test-";

    var claims = new List<Claim>();

    var claimHeaders = context.Request.Headers.Keys.Where(k => k.StartsWith(headerPrefix));
    foreach (var header in claimHeaders)
    {
        var value = context.Request.Headers[header];
        var claimType = header[headerPrefix.Length..];
        if (!string.IsNullOrEmpty(value))
        {
            claims.Add(new Claim(claimType == "role" ? ClaimTypes.Role : claimType, value!));
        }
    }
    return claims;
}
```

The code looks for header that look like `X-Test-**` and
then creates claims based on those headers with some
special handling for `X-Test-role` which uses
`ClaimTypes.Role` for the claim type.

In the `BaseTest` class, I created a new `HttpClient`
called `AdminCLient`, and in the constructor:

```c#
AdminClient = factory.CreateClient();
AdminClient.DefaultRequestHeaders.Add("X-Test-role", "admin");
```

Now any time I want to make an API call with an `admin` role
I can simply reference the `AdminClient` - see this example:

```c# {hl_lines=[8]}
[Fact(DisplayName = "Create Product multiple validation errors")]
public async Task CreateProductMultipleValidations()
{
    var newProduct = dbFixture.ProductFaker.Generate();
    newProduct.Name = "";
    newProduct.Category = "";

    var problem = await AdminClient.PostJsonForResultAsync<ProblemDetails>($"/v1/products", newProduct,
        HttpStatusCode.BadRequest);

    Assert.Equal("Validation error(s) occurred.", problem.Title);
    Assert.Equal("Name is required.", problem.Extensions["Name"]!.ToString());
    Assert.Equal("Category is required.", problem.Extensions["Category"]!.ToString());
}
```

And to test non-admins is pretty simple, too - the following code
uses the regular `Client` (no admin header) with and without
role headers:

```c#
[Theory(DisplayName = "Non-Admin role cannot create product")]
[InlineData("")]
[InlineData("non-admin")]
public async Task NonAdminCannotCreateProduct(string role)
{
    var newProduct = dbFixture.ProductFaker.Generate();

    if (!string.IsNullOrEmpty(role))
    {
        Client.DefaultRequestHeaders.Add("X-Test-role", role);
    }
    var response = await Client.PostAsJsonAsync($"/v1/products", newProduct);
    Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
}
```

## Write Some Tests!

This series of posts offered up some guidance and suggestions
about how to create some effective and easy-to-read integration
tests for ASP.NET Core APIs.

Try them out, and your feedback is welcome.

I'll be adding one more (bonus!!) post to this series that
will show how to review code coverage for the tests
we've got.

Happy coding and testing!
