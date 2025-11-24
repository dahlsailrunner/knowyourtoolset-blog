---
title: "Testing for the new SameSite Cookie-handling Changes" # Title of the blog post.
date: 2020-01-09T09:59:25-05:00 # Date of post creation.
summary: "Testing applications for the new behavior change in Chrome and Chromium regarding the handling of SameSite cookies" 
thumbnail: "/images/same-site-feature.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/same-site-feature.png" # Designate a separate image for social media sharing.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - Authentication
# comment: false # Disable comment if false.
---

A topic that has been getting some heightened attention lately is the [upcoming change that Chrome is introducing that will change the way SameSite cookies are handled](https://www.chromium.org/updates/same-site), specifically in POST requests from different domains.

The Auth0 blog has a pretty [good summary of the changes, why they are happening and how they specifically affect OpenID Connect / OAuth2 flows](https://auth0.com/blog/browser-behavior-changes-what-developers-need-to-know/).

I was trying to test out how these changes are going to affect our existing applications, and ran into problems actually reproducing the issue described. But a little digging (and a nudge to a forum from @BrockLAllen) led me to clean ways to test for the issues.

## Seeing the Behavior (Kind of)

You can turn on the flag in the current version of Chrome by going to the Flags section of Chrome (chrome://flags) and toggling the “SameSite by default cookies” to Enabled (see screenshot below).

![::img-med img-center img-shadow](/images/same-site-feature.png)

{{% notice warning "Warning!" %}}
At least for OpenID Connect / OAuth2 flows, this may not be enough to show a “broken” application flow. The reason for this is that Chrome has a mitigation feature called “Lax + POST” that will actually include cookies that have been created within a two-minute window that would have been affected by the behavior change. You can read about this in the Nov 1, 2019 update of the [Chromium Project Updates page](https://www.chromium.org/updates/same-site) mentioned above.
{{% /notice %}}

What you **can** see with this behavior enabled is Developer Tools Console messages (assuming you have the “Preserve Log” setting checked), and they tell you whether any cookies WOULD be affected but were allowed because they were less than 2 minutes old, as shown here:

![::img-shadow](/images/samesite-devtools.png)

## Seeing the Behavior (for real)

You need to run a [Canary build of Chrome](https://www.google.com/chrome/canary/) with the ``--enable-features=SameSiteDefaultChecksMethodRigorously`` command line option specified.

This means that you can’t just "launch" the Chrome Canary build but that you need to run it from a terminal or command line. I used the link above to download and install the canary build on my Windows machine and it was placed here: `C:\Users\edahl1\AppData\Local\Google\Chrome SxS\Application`.

Then from a terminal in that directory (wherever your Chrome canary version was installed), you can run the following:

```
./chrome.exe --enable-features=SameSiteDefaultChecksMethodRigorously
```

Instead of telling you via the console that a cookie would not be sent, the cookie is simply not sent. And this is the change / behavior that will break some application flows.

Happy breaking / testing!
