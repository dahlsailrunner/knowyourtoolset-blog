---
title: "Secure Web APIs With Swagger, Swashbuckle, and OAuth2 (Part 3)" 
date: 2015-09-01T05:45:40-05:00 # Date of post creation.
description: "Writing an ASP.NET Framework Web API that uses Swashbuckle for a Swagger UI and OAuth2 for security - customizing the user interface." # Description used for search engine.
thumbnail: "/images/TotallyCustomSwagger.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/TotallyCustomSwagger.png" # Designate a separate image for social media sharing.
codeMaxLines: 25 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: true # Override global value for showing of line numbers within code block.
tags:
  - ASP.NET 
  - API
  - OAuth2
---

This article continues the process started in [part 1({{< ref "/2015/08/secure-web-apis-with-swagger-swashbuckle-and-oauth2-part-1" >}})] which 
concluded with us having an API that has both anonymous and secure methods that can be called, and a Swagger interface provided by Swashbuckle. For 
this post, I will be discussing how to customize the Swagger interface and make it your own. Following this, we will lock down the 
Swagger interface ([next and final post in this series]({{< ref "/2015/09/secure-web-apis-with-swagger-swashbuckle-and-oauth2-part-4" >}})).

You really have three options to customize the Swagger interface using Swashbuckle:

* Injecting your own stylesheet (CSS) into the mix
* Injecting some javascript into the mix
* Replacing the entire html contents of the page

I’m using the first and third options above in order to achieve maximum control over the results. Since I’m not really using “inject javascript” at all, 
I won’t discuss that, beyond saying that you can review the DOM and manipulate as you see fit by injecting your javascript using the same approach as used for CSS and for the custom HTML.

## Injecting your own stylesheet
To start with, I created a new stylesheet called `Swagger.css` in the `Content` folder of the API project. You can use an existing stylesheet like `Site.css` 
or any name that you like (and place it anywhere in the project). Once you have created the file, you need to access its properties (right 
click on the filename from within Visual Studio, choose Properties at the bottom of the menu) and set the Build Action to 
be “Embedded Resource”. This is a very important step — your customization will not work without it.

Secondly, you need to make sure you have a line like this within the `.EnableSwaggerUi` extension method call:

```csharp
c.InjectStylesheet(thisAssembly, "KeenNewApi.Content.Swagger.css");
```

“KeenNewApi” is the name of your project / assembly, the “Content” is the name of the subfolder you have your CSS file in. If you have not 
put your CSS file in a subfolder, then the string would be “KeenNewApi.Swagger.css”, and if you have multiple subfolders, each 
directory should be separated by a “.”.

Then it’s pretty typical CSS modification. I used the F12 developer tools to locate the top Swagger navbar which has an element like 
this (the … is just there to represent some stuff in between the elements):

```html
<div id="header">
  ....
</div>
```

I wanted to change the header bar from green (saw this in the F12 tools) to orange, so I just made my `Swagger.css` look like this:

```css
#header {
    background-color: orange !important;
}
```
The reason for the important is because something in the core Swagger CSS still overrode this, but you get the point. You can use the DOM explorer to 
find the elements you want to re-style and then modify your CSS to your specs. 

{{% notice warning "Important!" %}}
Since this is an “embedded resource”, just updating the CSS and refreshing the Swagger page will not have any effect. You will need to re-run the application 
if you want the change to take effect.
{{% /notice %}}

Here is the unmodified, standard Swagger UI provided by Swashbuckle:

![::img-med img-center img-shadow](/images/UncustomizedSwagger.png)

Here is the result of injecting the custom CSS with the orange header:

![::img-med img-center img-shadow](/images/SwaggerWithCustomCss.png)

## Replacing the entire HTML contents of the page
The beginning to the approach here is the same as with the CSS;

1. reate a new HTML page somewhere in your project (I put mine in the Content folder and called it CustomSwagger.html)
1. Modify the Build Action property of the file and set it as an “Embedded Resource”
1. Modify the SwaggerConfig.cs EnableSwaggerUi method to have set a CustomAsset for “index” that references your embedded resource
1. Modify the HTML contents to suit your needs

For point 3 above, the line in `SwaggerConfig.cs` ends up looking something like this – using the same referencing conventions as noted above:
```csharp
c.CustomAsset("index", thisAssembly, "KeenNewApi.Content.CustomSwagger.html");
```
The tricky part in all of this is figuring out what to make the HTML look like. To figure this out, I first tried using the sample custom HTML file 
from the GitHub project, but that didn’t really work for me. I then just ran the project WITHOUT a custom asset for “index” and did a “view source” on 
the web browser from the swagger/ui/index page. I then copy/pasted that content into the CustomSwagger.html file and started my mods from there, 
which worked pretty well.

I ended up with a file that looked like this (I’ll have some comments on the code below the code listing):

```html
<!DOCTYPE html>
<html>
<head>
    <title>Swagger UI</title>
    <link rel="icon" type="image/png" href="images/favicon-32x32-png" sizes="32x32" />
    <link rel="icon" type="image/png" href="images/favicon-16x16-png" sizes="16x16" />
    <link href='css/typography-css' media='screen' rel='stylesheet' type='text/css' />
    <link href='css/reset-css' media='screen' rel='stylesheet' type='text/css' />
    <link href='css/screen-css' media='screen' rel='stylesheet' type='text/css' />
    <link href='css/reset-css' media='print' rel='stylesheet' type='text/css' />
    <link href='css/print-css' media='print' rel='stylesheet' type='text/css' />
    <link href='ext/KeenNewApi-Content-Swagger-css' media='screen' rel='stylesheet' type='text/css' />
 
    <link href="/Content/bootstrap.css" rel="stylesheet" />
    <script src="/Scripts/modernizr-2.6.2.js"></script>
 
    <script src="/Scripts/jquery-1.10.2.js"></script>
 
    <script src='lib/jquery-slideto-min-js' type='text/javascript'></script>
    <script src='lib/jquery-wiggle-min-js' type='text/javascript'></script>
    <script src='lib/jquery-ba-bbq-min-js' type='text/javascript'></script>
    <script src='lib/handlebars-2-0-0-js' type='text/javascript'></script>
    <script src='lib/underscore-min-js' type='text/javascript'></script>
    <script src='lib/backbone-min-js' type='text/javascript'></script>
    <script src='swagger-ui-js' type='text/javascript'></script>
    <script src='lib/highlight-7-3-pack-js' type='text/javascript'></script>
    <script src='lib/marked-js' type='text/javascript'></script>
 
    <script src='lib/swagger-oauth-js' type='text/javascript'></script>
 
    <script src="/Scripts/bootstrap.js"></script>
    <script src="/Scripts/respond.js"></script>
 
    <script type="text/javascript">
    $(function () {
      var url = window.location.search.match(/url=([^&]+)/);
      if (url && url.length > 1) {
        url = decodeURIComponent(url[1]);
      } else {
        url = "http://petstore.swagger.io/v2/swagger.json";
      }
 
        // Get Swashbuckle config into JavaScript
      function arrayFrom(configString) {
          return (configString !== "") ? configString.split('|') : [];
      }
 
      function stringOrNullFrom(configString) {
          return (configString !== "null") ? configString : null;
      }
 
      window.swashbuckleConfig = {
          rootUrl: '%(RootUrl)',
          discoveryPaths: arrayFrom('%(DiscoveryPaths)'),
          booleanValue: arrayFrom('true|false'),
          validatorUrl: stringOrNullFrom(''),
          customScripts: arrayFrom(''),
          docExpansion: '%(DocExpansion)',
          oAuth2Enabled: '%(OAuth2Enabled)',
          oAuth2ClientId: '%(OAuth2ClientId)',
          oAuth2Realm: '%(OAuth2Realm)',
          oAuth2AppName: '%(OAuth2AppName)'
      };
 
      window.swaggerUi = new SwaggerUi({
        url: swashbuckleConfig.rootUrl + "/" + swashbuckleConfig.discoveryPaths[0],
        dom_id: "swagger-ui-container",
        booleanValues: swashbuckleConfig.booleanValues,
        onComplete: function(swaggerApi, swaggerUi){
          if (typeof initOAuth == "function" && swashbuckleConfig.oAuth2Enabled) {
            initOAuth({
              clientId: swashbuckleConfig.oAuth2ClientId,
              realm: swashbuckleConfig.oAuth2Realm,
              appName: swashbuckleConfig.oAuth2AppName
            });
          }
          $('pre code').each(function(i, e) {
            hljs.highlightBlock(e)
          });
 
          addApiKeyAuthorization();
 
          window.swaggerApi = swaggerApi;
          _.each(swashbuckleConfig.customScripts, function (script) {
              $.getScript(script);
          });
        },
        onFailure: function(data) {
          log("Unable to Load SwaggerUI");
        },
        docExpansion: swashbuckleConfig.docExpansion,
        sorter : "alpha"
      });
 
      if (window.swashbuckleConfig.validatorUrl !== '')
          window.swaggerUi.options.validatorUrl = window.swashbuckleConfig.validatorUrl;
 
      function addApiKeyAuthorization(){
        var key = encodeURIComponent($('#input_apiKey')[0].value);
        if(key && key.trim() != "") {
            var apiKeyAuth = new SwaggerClient.ApiKeyAuthorization("api_key", key, "query");
            window.swaggerUi.api.clientAuthorizations.add("api_key", apiKeyAuth);
            log("added key " + key);
        }
      }
 
      $('#input_apiKey').change(addApiKeyAuthorization);
 
      window.swaggerUi.load();
 
      function log() {
        if ('console' in window) {
          console.log.apply(console, arguments);
        }
      }
  });
    </script>
</head>
 
<body class="swagger-section">
    <div id="header">
        <div class="navbar navbar-inverse navbar-fixed-top">
            <div class="container">
                <div class="navbar-header">
                    <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
                        <span class="icon-bar"></span>
                        <span class="icon-bar"></span>
                        <span class="icon-bar"></span>
                    </button>
                    <a class="navbar-brand" href="/">Keen New API</a>
                </div>
                <div class="navbar-collapse collapse">
                    <ul class="nav navbar-nav">
                        <li><a href="/swagger">Swagger</a></li>
                        <li><a href="/Help">API</a></li>
                        <li class="text-right">
                            <a href="javascript:document.getElementById('logoutForm').submit()">Log out</a>
                            <form action="/Home/SignOut" id="logoutForm" method="post"></form>
                        </li>
                    </ul>
                </div>
            </div>
        </div>
    </div>
 
    <div class="container" style="padding-top: 30px;">
        <div class="alert alert-success alert-dismissible" role="alert">
            <button type="button" class="close" data-dismiss="alert" aria-label="Close"><span aria-hidden="true">&times;</span></button>
            <strong>NEW! </strong>Check out our keen new authentication toggles!
        </div>
    </div>
 
    <div id="swagger-ui-container" class="swagger-ui-wrap"></div>
 
</body>
</html>
```

My comments fall into three areas: (1) the head section with its references, (2) the javascript, and (3), the main html body.

### The head section

For this, I wanted to make sure I could reference bootstrap classes and the latest jquery, as well as my own CSS file, so I added 
lines 12-17, 31-32 to accomplish that.

### The javascript
It took a little tinkering, but the operative section in the javascript that you’ll specifically want to pay attention to is the 
one where it calls `window.swashbuckleConfig` as shown below. I think you should be able to use the code below in the place of your own version 
if you copied/pasted from the standard Swagger index page. Your version likely has hard-coded values, and I wanted to make 
sure that the javascript honored what I had in `SwaggerConfig.cs` for these values.

```js
window.swashbuckleConfig = {
       rootUrl: '%(RootUrl)',
       discoveryPaths: arrayFrom('%(DiscoveryPaths)'),
       booleanValue: arrayFrom('true|false'),
       validatorUrl: stringOrNullFrom(''),
       customScripts: arrayFrom(''),
       docExpansion: '%(DocExpansion)',
       oAuth2Enabled: '%(OAuth2Enabled)',
       oAuth2ClientId: '%(OAuth2ClientId)',
       oAuth2Realm: '%(OAuth2Realm)',
       oAuth2AppName: '%(OAuth2AppName)'
   };
```

### The HTML content
In the HTML body section above, there are basically three main divs: `id=”header”`, the “container” (no id), and `id=”swagger-ui-container”`. You 
want to make sure that you KEEP the “swagger-ui-container”, as that is where all of the real swagger stuff is going to show up.

The “header” I modified based on the `_layout.cshtml` file from my MVC folders so that the nav bar looked like the rest of the site.

The id-less container is simply a bootstrap “dismissable alert” that I wanted to use to show that I had access to both 
the bootstrap css functionality as well as its javascript.

Here is the result of the customized HTML:

![::img-med img-center img-shadow](/images/TotallyCustomSwagger.png)

As for your own modifications, I’m guessing you have enough to go on at this point. Happy customizing!!

PS. The last post (coming shortly) will describe how to secure the Swagger interface so that only authenticated users can access it.