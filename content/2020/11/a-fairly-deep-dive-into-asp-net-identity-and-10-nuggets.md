---
title: "A (Fairly) Deep Dive Into Asp Net Identity and 10+ 'Nuggets'" # Title of the blog post.
date: 2020-11-04T13:25:37-05:00 # Date of post creation.
description: "A handful of useful features within ASP.NET Core Identity and how to put them to use" # Description used for search engine.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
#featureImage: "/images/path/file.jpg" # Sets featured image on blog post.
#thumbnail: "/images/path/thumbnail.png" # Sets thumbnail image appearing inside card on homepage.
#shareImage: "/images/path/share.png" # Designate a separate image for social media sharing.
codeMaxLines: 20 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - ASP.NET 
  - ASP.NET Identity
  - Authentication
---

I just recently had the privilege of creating a new course for Pluralsight called [Secure User Account and Authentication Practices in ASP.NET and ASP.NET Core](https://app.pluralsight.com/library/courses/secure-account-authentication-practices-asp-dot-net-core/), and it brought me down a path that involved a lot of exploration around how identity works in ASP.NET (and Core — I’ll just use "ASP.NET" going forward in this post unless it’s specifically worth noting here).

**Just show me the code:** The completed source code for all of the items below for both ASP.NET Core and ASP.NET WebForms is here: https://github.com/dahlsailrunner/secure-authentication
Look at the commit history for some of the changes to the original code that built all of the below items up.

{{% notice tip Tip %}}
In my [next post]({{< ref "2020/11/moving-authentication-from-an-asp-net-site-into-identityserver4" >}}) I will move all of this ASP.NET Core Identity goodness into a standalone IdentityServer4 (OpenID-Connect / OAuth2) instance that separates the authentication concerns into an application that can support any of your applications – like SPAs, machine-to-machine auth needs, and more.
{{% /notice %}}

## More Than 10 "Nuggets" 

I’ve watched many Pluralsight courses and am almost always surprised by what I call "nuggets" in courses. I may consider myself somewhat knowledgeable in an area where I’m watching a course, and then the author does something or uses something that I hadn’t considered before and that will certainly assist me in future endeavors — and it may not have even been the focal point of the course!

These nuggets are awesome discoveries and it continues to emphasize the benefit of continuous learning and collaboration for me.

While creating this account/authentication/authorization course I noted more than 10 nuggets that might be of interest to folks — some of which are squarely in the domain of the course and a couple that aren’t.

Here’s a listing of them – the list is almost a “cookbook” where you can pick and choose what you’re interested in:

* **Adopt ASP.NET Identity with an existing user database and existing passwords**
* **Auto-update password hashing to improve its security** via a custom password hasher
* **Use [the haveibeenpwned API](https://haveibeenpwned.com/API/v3)** (via [Andrew Lock’s NuGet package](https://github.com/andrewlock/PwnedPasswords)) to prevent passwords from previous breaches from being used
* More **password validation** – add additional checks to avoid use of your site name or other predictable text
* **Email confirmation made simple** – [smtp4dev](https://github.com/rnwood/smtp4dev) and its Docker image
* **Authenticator app two-factor auth in both ASP.NET Core and WebForms** – still using our custom schema / user database
* **Customizing the base UserManager** for failed login attempts
* Adding **[request logging with Serilog](https://nblumhardt.com/2019/10/serilog-mvc-logging/)**
* **Creating a custom IUserClaimStore** for authorization and other claims-based needs on top of our existing schema
* Adding **role and claims-based authorization policies**
* **Rights based authorization** (this involves a custom policy provider and handler)
* **MFA enrollment and challenge authorization requirements** – either require a user to be enrolled in MFA, or go further and
require an MFA challenge during their login session to access pages / features
* ([Next post]({{< ref "2020/11/moving-authentication-from-an-asp-net-site-into-identityserver4" >}})) Moving all of the above (except the Authorization policies) into an IdentityServer4 instance

I discuss each of the above items in a little more detail on the readme of the github repo – and they’re all shown and explained in the course that I mentioned.

One of my goals for the course was to be able to demonstrate how to embrace ASP.NET identity and many of its very good (and easy to adopt) security practices if you had an existing user database and didn’t know exactly how to bring that into current ASP.NET identity functionality and certainly didn’t want to rewrite your application or completely change its user store (database). This really shines some light on how things work and how we can leverage the built-in features without necessarily needing to go
“all in” and use the default schema as well.

Have a look at the [code and readme](https://github.com/dahlsailrunner/secure-authentication) or the [course](https://app.pluralsight.com/library/courses/secure-account-authentication-practices-asp-dot-net-core/), and I hope this helps you in some way. 🙂