---
title: "ASP.NET Core APIs: Getting Swashbuckle to work with Auth0" # Title of the blog post.
date: 2022-07-04T05:51:55-05:00 # Date of post creation.
description: "Updating ASP.NET Core Swashbuckle OAuth2 options to support authentcation with Auth0 and prevent invalid_token JWT errors" # Description used for search engine.
codeMaxLines: 15 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
thumbnail: "/images/execute-api.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/execute-api.png" # Designate a separate image for social media sharing.
toc: true
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - ASP.NET Core
  - API
  - JWT
  - OAuth2
  - Swagger
  - Swashbuckle
  - Auth0
---

## Background

Using JWT bearer token authentication for your ASP.NET Core API projects is pretty standard stuff these days, and I've
done it many times - mostly with tokens from [Duende IdentityServer](https://duendesoftware.com/products/identityserver).
When I tried to update the code and configuration to use [Auth0](https://auth0.com/) though, I kept getting a 401 response
after authenticating properly, with a message about an `invalid_token`
and the token itself was a JWT but it had an empty payload -- the part of the JWT that contains the claims.

### tl;dr

I'll explain the full setup below, but **make sure you send the `audience` query string parameter with your OAuth2
authentication request to the `authorize` endpoint** (as shown in the `OAuthAdditionalQueryStringParams` method
below).  A code repo that demonstrates this is here: [https://github.com/dahlsailrunner/auth0-swashbuckle-api](https://github.com/dahlsailrunner/auth0-swashbuckle-api) (but you need to provide your own Auth0 tenant and client id).

```csharp
app.UseSwagger()
   .UseSwaggerUI(options =>
    {        
        options.OAuthClientId(clientId);
        options.OAuthAppName("QuickDemo");
        options.OAuthUsePkce();
        options.OAuthAdditionalQueryStringParams(new Dictionary<string, string>
        {
            { "audience", "https://yourapi-identifier-in-auth0" }
        });
    });
```

## Setting Up an ASP.NET Core API for JWT Bearer Authentication

If you've created a new web api, setting it up to support bearer token authentication is pretty simple.

You need to add a NuGet package reference for `Microsoft.AspNetCore.Authentication.JwtBearer` as a first step.

Then in `Program.cs` add code that looks like the following:

```csharp
builder.Services
  .AddAuthentication("Bearer")
  .AddJwtBearer(options =>
  {
      options.Authority = "YOUR-AUTH0-TENANT";
      options.Audience = "PLACEHOLDER FOR API IDENTIFIER";
  });
```

You should probably also have the following in your `Program.cs` file a little further down for the HTTP
request pipeline which will require an authenticated user on calls to your API:

```csharp
 app        
  .UseAuthentication()  
  .UseRouting()
  .UseAuthorization()
  .UseEndpoints(endpoints =>
  {
      endpoints.MapControllers().RequireAuthorization();      
  });
```

## Configuring the API within Auth0

Within the management area of Auth0, there is an **Applications** section, and that has an item called **APIs**.

If you go there and **Create API** you will be presented with a dialog that looks something like this:

![::img-center img-shadow img-med](/images/create-api.png)

In the `Identifier` section you can specify anything you like, but the value here will need to be
what you provide in the `audience` querystring parameter.

Once you have done this, make sure to update the `Authority` and `Audience` values of the `AddJwtBearer` call
above.

### Testing the authentication to your API

If you have created an API within Auth0 and updated the `Authority` and `Audience` values in your code/configuration,
you can test the authorization in your API.

Make a simple `GET` request to your API and it should return a 401 since you are not passing a JWT bearer token
in the HTTP headers yet.

Then go to the Test page in the Auth0 API settings for your API as shown below:
![::img-center img-shadow img-med](/images/test-api.png)

The **Response** section will show an `access_token` that you can use.

Using a tool like [Postman] or [Insomnia] or anything else, send the same `GET` request to your
API method but this time add an HTTP header called `Authorization` and set the value to `Bearer ACCESS_TOKEN_VALUE`
where you replace `ACCESS_TOKEN_VALUE` with the `access_token` from the test page.  In the screen shot this
would be `eyJh...` (rest of token omitted).

This should return a successful response from your API, which means it is properly requiring authenticated
users.

## Configuring a Swagger Client Application in Auth0

If you want to test your API from the Swagger UI, you need to set up an **Application** within
Auth0 to do that.  

You should set up a **Single Page Application** for this.  In the **Applications** section of Auth0
choose "Create Application" and choose the "Single Page Web Applications" option.

The **Settings** page should look something like this:

![::img-center img-shadow img-med](/images/swagger-application.png)

Note the `Domain` - this will be the `Authority` - it should be the same value you used for the
`AddJwtBearer` authority above.

Also note the `Client ID` - this will be the **OAuth2 Client Id** that you will need to provide
to some Swagger setup below.

In the **Application URIs** section, provide the Callback URL for your API. It will be the base address
of your API with `/oauth2-redirect.html` appended, so something like this:

```bash
  https://localhost:44380/oauth2-redirect.html
```

## Connecting Swagger UI Authentication to Auth0

Using the Swagger UI to test your API is very handy, and relatively easy to set up.

{{% notice warning Caution %}}
You may want your Swagger UI (and even the Swagger doc itself) available in non-production
environments.  If this is the case just make sure to wrap your `UseSwagger` method calls with
an environment check.
{{% /notice %}}

For my purposes, I find that having the code related to Swagger in its own folder can be
helpful, and I've created two extension methods (`AddSwaggerFeatures` and `UseSwaggerFeatures`)
that I call from `Program.cs` - and they are shown in the listing below (which you can expand with
the ellipsis "...") - note that they support both authentication and versioned APIs:

```csharp
public static class SwaggerExtensions
{
  public static IServiceCollection AddSwaggerFeatures(this IServiceCollection services)
  {
      services.AddTransient<IConfigureOptions<SwaggerGenOptions>, ConfigureSwaggerOptions>();
      services.AddSwaggerGen();

      return services;
  }

  public static IApplicationBuilder UseSwaggerFeatures(this IApplicationBuilder app, IConfiguration config,
      IApiVersionDescriptionProvider provider, IWebHostEnvironment env)
  {
    if (!env.IsDevelopment())
    {
        return app;
    }

    var clientId = config.GetValue<string>("Authentication:SwaggerClientId");
    app
      .UseSwagger()
      .UseSwaggerUI(options =>
      {
        foreach (var description in provider.ApiVersionDescriptions)
        {
          options.SwaggerEndpoint($"/swagger/{description.GroupName}/swagger.json",
              $"QuickDemo API {description.GroupName.ToUpperInvariant()}");
          options.RoutePrefix = string.Empty;
        }

        options.DocumentTitle = "QuickDemo Documentation";
        options.OAuthClientId(clientId);
        options.OAuthAppName("QuickDemo");
        options.OAuthUsePkce();
        options.OAuthAdditionalQueryStringParams(new Dictionary<string, string>
        {
          { "audience", config.GetValue<string>("Authentication:ApiName") }
        });
      });

    return app;
  }
}
```

A key ingredient of the above code (you need to expand the listing to see it) is the inclusion
of the `audience` query string parameter when making the `authorize` request to Auth0. It's these
three lines:

```csharp
options.OAuthAdditionalQueryStringParams(new Dictionary<string, string>
{
  { "audience", config.GetValue<string>("Authentication:ApiName") }
});
```

The `ConfigureSwaggerOptions` class is a second file that I generally use and it has the
security definitions that will be used - and it reads configuration to drive it.  The
listing is probably [easier to see on GitHub](https://github.com/dahlsailrunner/auth0-swashbuckle-api/blob/main/QuickDemo.Api/Swagger/ConfigureSwaggerOptions.cs).

The configuration that both of the Swagger-related files read is here:

```json
 "Authentication": {
    "Authority": "YOUR_AUTH0_TENANT - https://tenant.us.auth0.com",
    "ApiName": "https://dsr-sampleapi",
    "SwaggerClientId": "AUTH0_SINGLE_PAGE_CLIENTID",
    "AdditionalScopes": "openid profile email read:everything"
  }
```

Once you've got some code at least somewhat like this, you should be able to run your API and see
a Swagger UI that includes an `Authorize` button, and when you click it the dialog you see
should look something like this:

![::img-center img-shadow img-med](/images/swagger-authorize.png)

The `client_secret` ***can be left blank*** as it is not used in this particular
authorization flow (Code with PKCE).  You should choose `select all` to choose all of the
scopes.

Once you complete the authorization, you should be back on the Swagger page and you
can execute API methods. When you do that, you should see a `curl` command that includes
the bearer token that was issued when you logged in:

![::img-center img-shadow img-med](/images/execute-api.png)

You can copy that bearer token value - the whole thing, it's long - into your clipboard and
paste the value into [JWT.io](https://jwt.io)

![::img-center img-shadow img-med](/images/claims.png)

Your API call should have been successful, and if you put a breakpoint in the controller
and look at the `User` property, you should see claims that match the payload section shown
above.

Awesome!

## Going Further

There is an additional `scope` in the token that I was showing above called `read:everything` and
this was something I configured both in the API defined in Auth0 as a "permission" that can
be associated with it, and in the requested scope from the application. You could wire up
some permissions in the ASP.NET Core API to make sure that this scope is allowed on the `GET`
methods of your API.

Hope this was helpful!