---
title: "Secure Web APIs With Swagger, Swashbuckle, and OAuth2 (Part 1)" 
date: 2015-08-12T05:45:40-05:00 # Date of post creation.
description: "Writing an ASP.NET Framework Web API that uses Swashbuckle for a Swagger UI and OAuth2 for security. - setting up the API with a Swagger UI " # Description used for search engine.
thumbnail: "/images/SwaggerSample.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/SwaggerSample.png" # Designate a separate image for social media sharing.
codeMaxLines: 25 # Override global value for how many lines within a code block before auto-collapsing.
tags:
  - ASP.NET 
  - API
  - OAuth2
---

I wanted to go down the path of creating a shiny new custom enterprise-grade API framework that includes the following features:

* Easy to navigate documentation and testability via a Swagger interface that can be tested/tried right on the site hosting the API
* Authenticated access for customers and clients using an OAuth2 endpoint (a custom OAuth2 endpoint using [Thinktecture’s IdentityServer3](https://github.com/IdentityServer/IdentityServer3))
* The Swagger interface should “fit well” in the rest of the API site — meaning a consistent look and feel and navigation options

{{% notice info "A Quick Aside" %}}
None of the above would have been possible without a couple a “prerequisites” that I first reviewed:

* Two excellent Pluralsight courses from Dominick Baier regarding [Web API security](http://www.pluralsight.com/courses/webapi-v2-security) and [OAuth2/OpenId-Connect](http://www.pluralsight.com/courses/oauth2-json-web-tokens-openid-connect-introduction)
* The [GitHub documentation for the Swashbuckle package](https://github.com/domaindrivendev/Swashbuckle#swashbuckle-50)

I can’t recommend these resources strongly enough. They’re great — especially Dominick’s courses and the Thinktecture IdentityServer3 documentation and samples.
{{% /notice %}}

## Create your project
I got started by creating a brand new Web API project and selecting **"No Authentication"** for the initial project — I would be adding that manually to support the correct, modern token-based authentication favored by modern applications.

## NuGet Packages
Right after the creation of the project, there were a couple of NuGet packages that I installed using the NuGet package manager:

* Microsoft.Owin.Host.SystemWeb
* Microsoft.Owin.Security.Jwt
* Microsoft.AspNet.WebApi.Owin
* Swashbuckle
* Thinktecture.IdentityServer3.AccessTokenValidation

## Setting up the authentication
Authentication is pretty easy to setup, *assuming you already have your OAUth server configured and ready*. This assumption turns out to be non-trivial, but setting 
it up is not the subject of this post.

So, given an OAuth endpoint (especially IdentityServer3), configuring the API project to use it is pretty straightforward. You already have the required NuGet packages.

The next step is setting up the Owin Middleware to use this authentication for API calls. To do this, do a few things:
Create a Startup class in your API project and make it look like this:

```csharp
using Owin;
using Thinktecture.IdentityServer.AccessTokenValidation;
 
namespace YourApiProject
{    
    public partial class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            ConfigureAuth(app);
        }
 
        public void ConfigureAuth(IAppBuilder app)
        {
            app.UseIdentityServerBearerTokenAuthentication(new IdentityServerBearerTokenAuthenticationOptions
            {
                Authority = SomeStaticHelperClass.GetIssuerUri(),
                RequiredScopes = new[] { "sampleapi" }
            });
        }
    }
}
```
### Recommendation: Create a `GetIssuerUri` method in some helper class
You won’t have a `SomeStaticHelperClass.GetIssuerUri` method defined but you should put this into a standalone class (we’ll see why 
when we hit Swagger below), and it should look something like the method below. It’s basically returning the URL of your identity issuer, and 
this MAY be conditional based on your environments or whether you are still developing it, etc. You could just as easily use a hard-coded 
string or constant value in the above code if that suits your purpose (or read the value from a web.config file or whatever). As noted, though, 
this same value will come into play when we configure Swagger, so I recommend not simply hard-coding it in the above code (or you’ll end up doing it twice).

```csharp
public static string GetIssuerUri()
{
    switch (MyEnvironment)
    {
        case "PRD":
            return "https://id.mydomain.com/identity";
        case "QA":
            return "https://idqa.mydomain.com/identity";
        default:            
            return "https://id.local/identity";  // probably locally-hosted within IIS on developer machine
    }
}
```

## Update `WebApiConfig.cs` to support your token-based authentication
The code in your `WebApiConfig.cs` should look something like the code below. The operative lines for the authentication filters are the first two code lines – `SuppressDefaultHostAuthentication`, and add the new `HostAuthenticationFilter`.

```csharp
using System.Web.Http;
using System.Web.Http.ExceptionHandling;
using Microsoft.Owin.Security.OAuth;
 
 
namespace YourApiProject
{
    public static class WebApiConfig
    {
        public static void Register(HttpConfiguration config)
        {            
            config.SuppressDefaultHostAuthentication();
            config.Filters.Add(new HostAuthenticationFilter(OAuthDefaults.AuthenticationType));
 
            // Web API routes
            config.MapHttpAttributeRoutes();
            config.Routes.MapHttpRoute(
                name: "DefaultApi",
                routeTemplate: "api/{controller}/{id}",
                defaults: new { id = RouteParameter.Optional }
            );
 
            config.Services.Replace(typeof(IExceptionHandler), new CustomApiExceptionHandler());
            config.Services.Add(typeof(IExceptionLogger), new CustomApiExceptionLogger());
        }
    }
}
```
{{% notice note "Note on the Above" %}}
I’d like to specifically call out the LAST two lines as not necessarily required to support the authentication, but they are very valuable for 
logging and handling errors that occur in your API. See my [On exception handling post]({{< ref "/2015/05/on-exception-handling#web-api-apps" >}}) and 
refer to the global exception handlers/loggers for Web API. By adding some code as shown below to the dictionary that is getting logged, you can 
get the entire claims principal logged whenever you encounter an error — which can be very useful.

```csharp
var cp = context.RequestContext.Principal as ClaimsPrincipal;
if (cp != null)
{
    foreach (var claim in cp.Claims)
    {
         dict.Add(claim.Type, claim.Value);
    }
}
```
{{% /notice %}}

## Creating an API controller with some APIs requiring authentication
Here are the details (quite simple, actually) to create APIs that require authentication in order to invoke. Basically all you need 
to do is create a new `ApiController` and add the `[Authorize]` attribute to it.

Here is some code — it’s very simple for now. The controller itself has the Authorize attribute, which means that every 
method in it will require authentication — and is not role or user specific. As long as the caller is an authenticated caller they 
will be able to make the call. There are two actions in the controller: getstuff and getanonymous. The getanonymous action is 
just what it sounds like– you can call that method without being authenticated, but the getstuff method requires authentication.

```csharp
[RoutePrefix("api/keennewstuff")]
[Authorize]
public class KeenNewApiController : ApiController
{
    [Route("getstuff")]              
    public string GetStuff()
    {
        return "Here is some stuff";
    }
 
    [OverrideAuthorization]
    [AllowAnonymous]
    [Route("getanonymous")]
    public string GetAnonymous()
    {
        return "yay!";
    }
}
```
So now you have an API that should be ready to accept calls, but no real way to test it without creating a client application of 
some sort. That’s where [Swagger](http://swagger.io/) and [Swashbuckle](https://github.com/domaindrivendev/Swashbuckle) come in. Swagger 
provides a form of web-based UI on top of a web API that will provide information about the API methods, their http methods, their inputs, 
and their outputs and response types. Additionally it allows you to “test” the calls right on the page so you can see how the API behaves.

## Configure Swagger and add a link to your navigation bar to it
When we added our NuGet packages above, one of those that we added was Swashbuckle, which is a .Net implementation 
of Swagger. Adding this package will create a `SwaggerConfig.cs` in your `App_Start` folder. We now need to go into that folder and 
configure it to actually expose a Swagger page for our API.

To do this you need to add the "Swagger" line into your `_Layout.cshtml`.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width" />
    <title>@ViewBag.Title</title>
    @Styles.Render("~/Content/css")
    @Scripts.Render("~/bundles/modernizr")
</head>
<body>
    <div class="navbar navbar-inverse navbar-fixed-top">
        <div class="container">
            <div class="navbar-header">
                <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
                    <span class="icon-bar"></span>
                    <span class="icon-bar"></span>
                    <span class="icon-bar"></span>
                </button>
                @Html.ActionLink("Application name", "Index", "Home", new { area = "" }, new { @class = "navbar-brand" })
            </div>
            <div class="navbar-collapse collapse">
                <ul class="nav navbar-nav">
                    <li>@Html.ActionLink("Home", "Index", "Home", new {area = ""}, null)</li>
                    <li><a href="~/swagger">Swagger</a></li>
                    <li>@Html.ActionLink("API", "Index", "Help", new { area = "" }, null)</li>
                </ul>
            </div>
        </div>
    </div>
    <div class="container body-content">
        @RenderBody()
        <hr />
        <footer>
            <p>&copy; @DateTime.Now.Year - My ASP.NET Application</p>
        </footer>
    </div>
 
    @Scripts.Render("~/bundles/jquery")
    @Scripts.Render("~/bundles/bootstrap")
    @RenderSection("scripts", required: false)
</body>
</html>

```

## Testing things out so far….
I strongly recommend that at this point you set yourself up to run this within IIS locally (not IIS Express) and to use SSL to do it. This is made 
easier by some of the information in [Dominick Baier’s Web API Security Pluralsight course](http://www.pluralsight.com/courses/webapi-v2-security) — check out the 
part about Developers and SSL.

That aside, you can run with SSL inside of IIS Express as well and you can test this out.

When you click the Swagger link at the top of the page, you should see your API controller as a heading, and if you click on it the 
operations appear, as shown in the image below.

![::img-med img-center](/images/SwaggerSample.png)

You’re probably noticing two things about the screenshot (among others):

* **The page doesn’t have the navbar or look like the rest of the site.** This is one of the things we’ll be addressing later. Hang tight. 🙂
* **There is an “Error” reported at the bottom of the page.** This is nothing to worry about. What’s happening is that the URL for your swagger data is being sent to Swagger.io for validation — and in my case that website’s servers cannot resolve my “sampleapi.local” url and so it shows an error. You can resolve this error in two ways: 1) make your site available on the internet, or 2) eliminate the validation process and badge entirely (I’ll come back to this later).

## Trying the API from Swagger
If you click on the methods in the listing on the Swagger page, you will see more details about the method, including a button to “Try it out”.
When you try out the anonymous method, you should see a response similar to the one shown here.

![::img-med img-center](/images/GetAnonymous.png)

The regular method should give you a `401` unauthorized response, which shouldn’t surprise you. We require authentication, and this “try it out” 
button does not send any authentication information when it tries to invoke the method.

![::img-med img-center](/images/Get401.png)

So where are we headed from here? Three places:

* [Get authentication wired into the Swagger “try it out” buttons]({{< ref "/2015/08/secure-web-apis-with-swagger-swashbuckle-and-oauth2-part-2" >}})
* [Customize the look of the Swagger page to make it more “ours”]({{< ref "/2015/09/secure-web-apis-with-swagger-swashbuckle-and-oauth2-part-3" >}})
* [Lock down the Swagger UI so that only authenticated users can see it]({{< ref "/2015/09/secure-web-apis-with-swagger-swashbuckle-and-oauth2-part-4" >}})