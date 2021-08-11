---
title: "Updating an ASP.NET Core 2.2 Solution That Integrates With Identityserver4" # Title of the blog post.
date: 2020-07-17T14:25:02-05:00 # Date of post creation.
description: "Article description." # Description used for search engine.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - IdentityServer
  - ASP.NET
# comment: false # Disable comment if false.
---

# Background
The very useful [public demo of IdentityServer4](https://demo.identityserver.io/) recently changed to limit the different flows that it supports based on updated recommendations: https://leastprivilege.com/2019/09/09/two-is-the-magic-number/

I had an existing solution written in .NET Core 2.2 that needed to get updated to support these new flows, and that implied two sets of changes – one that would get the applications updated to .NET Core 3.1, and another that would have the updates required for the new OAuth2 flows.

# Solution Overview
The solution is a multi-project solution containing two web apps – one is a Razor Page UI project (`BookClub.UI`), and the other is an API project that serves as its back end (`BookClub.API`). The API does data access and uses the [Google Books API](https://developers.google.com/books/) to get some information about books for display. There are also a few class libraries that contain shared class definitions and such.

It is a simple sample application that demonstrates logging in a bit more than just “Hello, world” applications. The solution uses the public demo IdentityServer4 instance for authentication in the UI and for bearer token authentication in the API.

The current source code is here: https://github.com/dahlsailrunner/aspnetcore-effective-logging

A picture of the Book List page is shown below:

![::img-med img-shadow](/images/BookClub-UI.png)

## Getting from .NET Core 2.2 to 3.1
The commit for the the updates I made to get from .NET Core 2.2 to 3.1 is here: https://github.com/dahlsailrunner/aspnetcore-effective-logging/commit/238ea8035a7852335c1cf320879eecffc1f9d39e

### .NET Core Updates
There is [good documentation on the MS Docs website](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30?view=aspnetcore-3.1&tabs=visual-studio) for doing this upgrade, and I won’t repeat that information here. That documentation points out the changes to the csproj files for framework-related stuff and directs you to remove a couple of package references.

Additionally, `Startup.cs` in the two web projects has some changes regarding the wire-up of endpoints, routing, and such.

### Swashbuckle Updates
When I updated Swashbuckle from `4.0.1` to `5.5.1`, a few more changes needed to be made to the `AddSwaggerGen()` method, as shown in the commit changes screenshot (from **Startup.cs->ConfigureServices**):

![::img-shadow](/images/Swashbuckle-Updates.png)

Keep in mind that this does NOTHING to change the flows as mentioned to start — this is a simple update of the packages and code so that it does the same thing in the new version. The flow updates will be discussed / shown further below.

### Removing Newtonsoft.JSON in favor of System.Text.Json and System.Net.Http.Json
I also wanted to use the newer / more preferred System.Text.Json package in the updated application as well as making my JSON-based API calls as simple as possible (serialize POST content and deserialize response content).

One of the handiest changes here was to turn this code:
```csharp
var bookResponse = await httpClient.GetFromJsonAsync(uri);
if (!response.IsSuccessStatusCode)
{
   throw new Exception($"Failed in Google API for ISBN: {book.Isbn} -   responseCode = " + $"{response.StatusCode}");
}
var content = await response.Content.ReadAsStringAsync();
var bookResponse = JsonConvert.DeserializeObject(content);
```

into this:
```csharp
var bookResponse = await httpClient.GetFromJsonAsync&lt;GoogleBookResponse>(uri);
```

More good information about switching from `Newtonsoft.JSON` to `System.Text.Json` is here: https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-migrate-from-newtonsoft-how-to

And Steve Gordon has a great blog post about using `System.Net.Http.Json` here: https://www.stevejgordon.co.uk/sending-and-receiving-json-using-httpclient-with-system-net-http-json

### Updating for new IdentityServer4 flows
The commit that shows the updates for IdentityServer4 demo flows is here: https://github.com/dahlsailrunner/aspnetcore-effective-logging/commit/6aa84a6a1cbdd6012cbab559a81ad20bab73b237

#### UI Updates
The UI updates were very simple. Just the following lines for the new settings based on new setting values in the demo identity server.

```csharp
options.ClientId = "interactive.confidential";
options.ClientSecret = "secret";
options.ResponseType = "code";
options.Scope.Add("email");
options.Scope.Add("api");
options.Scope.Add("offline_access");
```

### API Updates
The API updates were a little more involved, because the implicit flow that I was using is no longer supported on the demo server ***and*** the `IdentityServer4.AccessValidationToken` package used for the bearer token validation is no longer being maintained, in favor of the standard `Microsoft.AspNetCore.Authentication.JwtBearer` package.

### Swagger Updates

Relative to the `AddSwaggerGen()` method, I added a transient `IConfigureOptions<SwaggerGenOptions>` implmentation that is injected to isolate the Swagger options into its own simple class file.

You can certainly inspect and review the source code for the whole class, but the cogent parts of the code that update the flow from implicit to code is here:

```csharp
options.AddSecurityDefinition("oauth2", new OpenApiSecurityScheme
{
  Type = SecuritySchemeType.OAuth2,
  Flows = new OpenApiOAuthFlows
  {
    AuthorizationCode = new OpenApiOAuthFlow
    {
      AuthorizationUrl = new Uri(disco.AuthorizeEndpoint),
      TokenUrl = new Uri(disco.TokenEndpoint),
      Scopes = oauthScopeDic
    }
  }
});
```
In the code above the disco variable is set with values from the discovery endpoint of the identity server, which I’m reading with the help of the `IdentityModel` package.

Then in the **Startup.cs->Configure** method, the `UseSwagger()` options also need to take some new values:
```csharp
options.OAuthClientId(Configuration.GetValue("Security:ClientId"));
options.OAuthClientSecret(Configuration.GetValue("Security:ClientSecret"));
options.OAuthAppName("Book Club API");
options.OAuthUsePkce();
```

### Token Authentication Updates
Finally, to update the code to use the `Microsoft.AspNetCore.Authentication.JwtBearer` package, the following existing code:

```csharp
.AddIdentityServerAuthentication(options =>
{
  options.Authority = "https://demo.identityserver.io";
  options.ApiName = "api";
});
```

needed to change into this:
```csharp
.AddJwtBearer(options =>
{
  options.Authority = Configuration.GetValue("Security:Authority");
  options.Audience = Configuration.GetValue("Security:Audience");
}
```

At the end of the day, all of the details described in this post are easily viewed in the commits on the GitHub repo — this post hopefully just drives a little bit toward understanding and categorizing the updates involved in the transition I needed to do, which likely is not that different from what at least a few others will encounter. 🙂