---
title: "Updating an ASP.NET Core 2.2 Solution That Integrates With Identityserver4" # Title of the blog post.
date: 2020-07-17T14:25:02-05:00 # Date of post creation.
description: "Article description." # Description used for search engine.
featured: true # Sets if post is a featured post, making appear on the home page side bar.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
#featureImage: "/images/path/file.jpg" # Sets featured image on blog post.
#thumbnail: "/images/path/thumbnail.png" # Sets thumbnail image appearing inside card on homepage.
#shareImage: "/images/path/share.png" # Designate a separate image for social media sharing.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - IdentityServer4
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