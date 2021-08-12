---
title: "Secure Web APIs With Swagger, Swashbuckle, and OAuth2 (Part 4)" 
date: 2015-09-03T05:45:40-05:00 # Date of post creation.
description: "Writing an ASP.NET Framework Web API that uses Swashbuckle for a Swagger UI and OAuth2 for security - restricting access to the Swagger UI" # Description used for search engine.
codeMaxLines: 25 # Override global value for how many lines within a code block before auto-collapsing.
tags:
  - ASP.NET 
  - API
  - OAuth2
---

This article continues the process started in [part 1]({{< ref "2015/08/secure-web-apis-with-swagger-swashbuckle-and-oauth2-part-1" >}}) which concluded with 
us having an API that has both anonymous and secure methods that can be called, and a Swagger interface provided by Swashbuckle. For 
this post, I will be discussing how to secure the Swagger interface so that public / anonymous users cannot browse your API.

I’ll break the topic into two logical parts: disabling anonymous access, and then enabling authenticated access.

## Disabling anonymous access
As it turns out, this is pretty easy but a little more roundabout than you might think. In order to get started, I added a global authorization filter, and then made the HomeController accept anonymous requests.

### Adding a global authorization filter
If you modify the `FilterConfig.cs` in `App_Start` with the line that follows, you will essentially require authorization for every routed request in your application.
```csharp
filters.Add(new AuthorizeAttribute());
```

### Enabling anonymous access to the `HomeController` actions
In order to allow anonymous users (including those that simply want to be able to login to your site) to see the content provided by the home 
controller — like a welcome page with a login form on it – you would add the `AllowAnonymous` attribute to your controller class as shown below.

```csharp
[AllowAnonymous]
public class HomeController : Controller
{
    //.....
}
```

### You mean we’re not done yet?
When I first started attempting to lock down the Swagger UI, I kinda figured I would be about done at this point — the global 
authorization filter should require authentication on every request except my explicitly-allowed HomeController – right? Wrong. 
For some reason (I suspect the route for Swagger is added later and the global filter never found its way to the new route), 
you can still get to “/swagger/ui/index” even with the configuration we now have in place, so more work was needed.

### Answer: Add a swagger directory and its own `web.config` file
Gabriel-a had a fairly elegant solution for this that I discovered in the GitHub issue set for Swashbuckle:
https://github.com/domaindrivendev/Swashbuckle/issues/384
If you add a new folder to your api project called “swagger” and then put a new web.config file in the folder with the contents below, we have 
achieved the desired result:
```xml
<configuration> 
  <system.web> 
   <authorization> 
     <deny users="?" /> 
   </authorization> 
  </system.web> 
  <system.webServer> 
     <modules runAllManagedModulesForAllRequests="true" /> 
  </system.webServer> 
 </configuration>
 ```
 At this point, no anonymous users can access the swagger/ui/index URL to browse the API. In fact, NO ONE can, because we haven’t enabled any 
 kind of login functionality yet — that comes next. But before that, I wanted to point out one more thing.

### Disabling the Swagger validation
You may have been noticing a little “Error” validation piece at the bottom of your Swagger page — as shown in the picture below.

![::img-center img-shadow](/images/ValidationError.png)

This image/tag appears due to validation that is pre-configured to happen that ensures you have a valid Swagger document. By default, this is “on” 
and the validator is the swagger webserver itself. Assuming you have a valid Swagger document, the main reason you would see an error is 
because the [Swagger website](https://swagger.io) cannot see the Swagger doc provided by your URL. And this can be because you’re running locally 
on some “localhost” type address that is not publicly accessible, OR because your swagger documents have been locked down (by the changes we just made).

You can either provide your own validator (I chose not to go this route), or disable the validation. To disable the validation, you 
might be able to add/uncomment the following line of `SwaggerConfig.cs` within the `.EnableSwaggerUi` method:

```csharp
c.DisableValidator();
```

But if you have introduced your own custom HTML (as I did in the [previous post]({{< ref "2015/09/secure-web-apis-with-swagger-swashbuckle-and-oauth2-part-3" >}})), 
you will need to modify the custom HTML javascript with the `validatorUrl` line below (note the “null”):

```js
window.swashbuckleConfig = {
     rootUrl: '%(RootUrl)',
     discoveryPaths: arrayFrom('%(DiscoveryPaths)'),
     booleanValue: arrayFrom('true|false'),
     validatorUrl: stringOrNullFrom('null'),
     customScripts: arrayFrom(''),
     docExpansion: '%(DocExpansion)',
     oAuth2Enabled: '%(OAuth2Enabled)',
     oAuth2ClientId: '%(OAuth2ClientId)',
     oAuth2Realm: '%(OAuth2Realm)',
     oAuth2AppName: '%(OAuth2AppName)'
 };
 ```
 Onto enabling login!!

## Enabling authenticated access
This segment will involve adding login functionality to the “browse-able” site — which will be cookie-based, rather than the token-based authentication 
used by the actual API we are providing.

### Enabling Cookie Authentication
First we should tell our site that it can accept cookie-based authentication if it is present. We do that by adding to the `ConfigAuth` method 
we created in [part 1]({{< ref "2015/08/secure-web-apis-with-swagger-swashbuckle-and-oauth2-part-1" >}}).

```csharp
public void ConfigureAuth(IAppBuilder app)
{
    app.UseIdentityServerBearerTokenAuthentication(new IdentityServerBearerTokenAuthenticationOptions
    {
        Authority = SomeStaticHelperClass.GetIssuerUri(),
        RequiredScopes = new[] { "sampleapi" }
    });
 
    app.UseCookieAuthentication(new CookieAuthenticationOptions
    {
        AuthenticationType = "Cookies"
    });
}
```
Next we need to figure out how we want our users to be able to log in.

### Endgame to support login — fundamentals
In order to support cookie-based login, you need to have some code somewhere that does the following:

```csharp
var id = new ClaimsIdentity(claims, "Cookies");
Request.GetOwinContext().Authentication.SignIn(id);
```
If you come up with a list of claims that you want based on some form input or anything else going on in your app, just 
create a new `ClaimsIdentity` with an authenticationType of “Cookies” and sign that id in to the `OwinContext`. What follows below is more
specific to my own login process / business requirements and how I am using IdentityServer3.

### My login options leveraging IdentityServer3
In my project, I wanted to meet a few requirements. I wanted users to be able to log in right from the home page (with a username/password form and a login button), 
and ALSO be able to support external authentication through Azure ActiveDirectory. As you may recall from part 1 (I encourage you to read 
it if you haven’t already), I have set up Thinktecture’s IdentityServer3 to use with my site. I have also added Azure to the identity server as a valid 
external authentication provider for this particular client (API and its browser interface). Doing that configuration is beyond the scope of this 
article (but I may write about that in another post); for this article, I want to focus on the client code to enable the login flows I mentioned, 
assuming that I already have a login provider that has been configured for me to use.

## Putting the login options on the home page
{{% notice note Note %}}
I *could* have opted to use the login screens within IdentityServer3, but I chose to use my own so that the identity functionality 
was much more transparent to the users.
{{% /notice %}}

To add the login form to the home page of the site, I added a model class called LoginModel to the Models folder as follows:

```csharp
public class LoginModel
{
    public string UserName { get; set; }
    public string Password { get; set; }        
}
```
Then I modified the controller code to use the model as shown here:
```csharp
public ActionResult Index()
{
    ViewBag.Title = "Home Page";
    var model = new LoginModel();
     
    return View(model);
}
```
Then I modified the view to present the form and buttons necessary (it doesn’t really matter where you put your login form — just so that it’s somewhere on the page):
```html
<div class="col-md-4">
    <h2>Start writing clients that can call the APIs</h2>
    <p>Log in and try them out.</p>
    <div id=" customLoginForm" class="input-group">
        <section id="loginForm">
            @using (Html.BeginForm(new { ViewBag.ReturnUrl }))
            {
                @Html.AntiForgeryToken()
                @Html.ValidationSummary(true)
                <div class="focus">
                        @Html.LabelFor(m => m.UserName)
                        @Html.TextBoxFor(m => m.UserName, new Dictionary<string, object> { { "class", "nwpinput" } })<br />
                        @Html.ValidationMessageFor(m => m.UserName)
                </div>
                <div>
                        @Html.LabelFor(m => m.Password)
                        @Html.PasswordFor(m => m.Password, new Dictionary<string, object> { { "class", "nwpinput" } })<br />
                        @Html.ValidationMessageFor(m => m.Password)
                </div>
                <input class="btn btn-success" type="submit" value="Login"/>
                <div>
                    <input type="button" class="btn btn-success" value="Login with Organizational Account" onclick="location.href='/home/signinexternal'"/>
                </div>
            }
        </section>
        </div>            
    </div>
```
At this point you have a login form — when you submit the form using the “Login” button, it will “POST” to the `Index` controller 
action with an input of the model. When you press the button “Login with Organizational Account” it will “GET” the `SignInExternal` 
controller action. We’ll see where both of these take us below.

### Using the local login form with the POST method
Essentially what is happening in this logic is that we want to take a username and password that the user have entered, turn those 
into a `ClaimsPrincipal` with an authenticationType of “Cookies”, and then use that principal to sign in to the `OwinContext`, followed by 
redirecting them to our swagger page. You could create your own claims principal if you wanted by validating the username and password 
yourself, and then creating whatever claims you want.

In the code below, I am using a two-step process with the identity server — (1) use the resource owner “password” grant to 
validate the login attempt, and then (2) use the provided access token to call the userinfo endpoint to get a list of claims 
for the user. Once I have the claims, then I just create a ClaimsPrincipal and sign the user in to the OwinContext.

{{% notice warning "Warning!" %}}
The resource owner password grant is no longer a recommended / supported flow for OIDC/OAuth2.  It is preferable to log in on 
the IdentityServer itself, so any customization to create the user experience you want should be done on the IdentityServer login 
page rather than creating a page within a client application.  The content below is simply preserved for posterity.  :)
{{% /notice %}}

```csharp
[HttpPost]        
public async Task<ActionResult> Index(LoginModel model)
{            
    if (!ModelState.IsValid)
        return View(model);
 
    var claims = await GetIdentityTokenAsync(model);
 
    if (claims != null && claims.Any())
    {
        if (claims.All(a => a.Type != ClaimTypes.NameIdentifier))
            claims.Add(new Claim(ClaimTypes.NameIdentifier, claims.First(a => a.Type == "sub").Value));
 
        if (claims.All(a => a.Type != "http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider"))
            claims.Add(new Claim("http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider", "myidp"));
 
        var id = new ClaimsIdentity(claims, "Cookies");
        Request.GetOwinContext().Authentication.SignIn(id);
         
        return Redirect("/swagger/ui/index");                
    }
 
    ModelState.AddModelError("loginfail", "Invalid username or password.");
    return View(model);
}
 
private async Task<List<Claim>> GetIdentityTokenAsync(LoginModel model)
{
    var baseUrl = SomeStaticHelperClass.GetIssuerUri();
    var tokenEndpoint = baseUrl + "/connect/token";
     
    var client = new OAuth2Client(
        new Uri(tokenEndpoint),
        "sampleapiapp",
        "sampleapisecret");
 
    var response = client.RequestResourceOwnerPasswordAsync(model.UserName, model.Password, scopesForAuth).Result;
 
    if (string.IsNullOrEmpty(response.AccessToken))
    {
        return null;  // must have failed the login process
    }
 
    var userInfoEndpoint = baseUrl + "/connect/userinfo";
 
    var userInfoClient = new UserInfoClient(
                            new Uri(userInfoEndpoint),
                            response.AccessToken);
 
    var userInfo = await userInfoClient.GetAsync();
 
    return userInfo.Claims.Select(claim => new Claim(claim.Item1, claim.Item2)).ToList();
}
```

### Using an external login service with a form post callback
For this flow, I am using the “Login with Organizational Account” button on the home page to indicate an Azure Active Directory login as 
provided for my identity server. The button should take the user straight to their Azure sign in location, and then back to the API 
site (after any permissions are granted / agreed to).

Here is the code I used for that — it starts with the GET for the `SignInExternal` method. Note that `SignInExternal` creates a `callbackUrl` 
that is included with the `CreateAuthorizeUrl` method. The `AuthorizeUrl` that is created indicates that we should receive a POST to the `callbackUrl` 
with the `id_token` if everything has been successful. Then it’s up to us (again) to turn that id_token into a `ClaimsPrincipal` and use it to 
log in to the `OwinContext` with a “Cookies” authenticationType.

The `id_token` that we get back below is a Json Web Token (JWT), and turning one of those into a `ClaimsIdentity` involves calling 
the `JwtSecurityTokenHandler.ValidateToken` method, which returns a `ClaimsIdentity`. But in order to even call that method, we need to 
set the `TokenValidationParameters`. The `ValidAudience` and `ValidIssuer` parameters are easy enough, but the `IssuerSigningToken` bears some explanation. 
This parameter needs to be the public portion of the certificate used to create the `id_token` — i.e. the certificate used by 
my identity server in this case. The API is not the IdentityServer, so I had to write 
the `GetPublicCertificateForIssuer` method to obtain that certificate. It works just fine and uses the 
same “GetIssuerUri” method that has been used in a number of places in our code.

```csharp
public ActionResult SignInExternal()
{
    var callbackUrl = string.Format("{0}://{1}/home/callback", Request.Url.Scheme, Request.Url.Authority);
    var state = Guid.NewGuid().ToString("N");
    var nonce = Guid.NewGuid().ToString("N");
 
    var client = new OAuth2Client(new Uri(UserHelper.GetIssuerUri() + "/connect/authorize"));
    var url = client.CreateAuthorizeUrl("sampleapisso", "id_token token", scopesForAuth, callbackUrl, state, nonce, acrValues: "idp:AzureAD", responseMode: "form_post");
    return Redirect(url);
}
 
[HttpPost]
public ActionResult Callback()
{
    var token = Request["id_token"];
    var state = Request["state"];
 
    var cert = GetPublicCertificateForIssuer();
    var parameters = new TokenValidationParameters
    {
        ValidAudience = "sampleapisso",
        ValidIssuer = UserHelper.GetIssuerUri().Replace("identity", ""),
        IssuerSigningToken = new X509SecurityToken(cert)
    };
 
    var handler = new JwtSecurityTokenHandler();
    SecurityToken jwt;
    var claimsId = handler.ValidateToken(token, parameters, out jwt);
    var claims = claimsId.Claims.ToList();
 
    if (claims.All(a => a.Type != ClaimTypes.NameIdentifier))
        claims.Add(new Claim(ClaimTypes.NameIdentifier, claims.First(a => a.Type == "sub").Value));
 
    if (claims.All(a => a.Type != "http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider"))
        claims.Add(new Claim("http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider", "myidp"));
 
    var id = new ClaimsIdentity(claims, "Cookies");
     
    Request.GetOwinContext().Authentication.SignIn(id);
 
    return Redirect("/swagger/ui/index");
}
 
private static X509Certificate2 GetPublicCertificateForIssuer()
{
    var issuerUri = new Uri(SomeStaticHelperClass.GetIssuerUri());
    var sp = ServicePointManager.FindServicePoint(issuerUri);
 
    var groupName = Guid.NewGuid().ToString();
    var req = WebRequest.Create(issuerUri) as HttpWebRequest;
    req.ConnectionGroupName = groupName;
 
    using (var resp = req.GetResponse())
    {
        /* empty body of using just to make sure webresponse is closed properly after getting response */
    }
    sp.CloseConnectionGroup(groupName);
    var certBytes = sp.Certificate.Export(X509ContentType.Cert);
 
    var cert = new X509Certificate2(certBytes);
    return cert;
}
```

## Wrapping it all up
That’s it! You now have an API that supports token-based authentication for the API calls and a secure, custom Swagger interface that 
supports browsing and testing your API (using OAuth2 tokens) for authenticated users!!

Enjoy, and happy API-ing! 🙂