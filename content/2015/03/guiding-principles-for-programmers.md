---
title: "Guiding Principles for Programmers" # Title of the blog post.
date: 2015-03-20T14:52:38-05:00 # Date of post creation.
summary: "General but opinionated principles to follow when doing software development for a living." # Description used for search engine.
featured: true # Sets if post is a featured post, making appear on the home page side bar.
#thumbnail: "/images/path/thumbnail.png" # Sets thumbnail image appearing inside card on homepage.
#shareImage: "/images/path/share.png" # Designate a separate image for social media sharing.
toc: true
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - Development
# comment: false # Disable comment if false.
---

Having some guiding principles (rules, if you will) that you follow when performing tasks can help you be more effective when
doing those tasks.  Here are some of the prgramming rules I follow.  The [SOLID principles](https://en.wikipedia.org/wiki/SOLID) are worth knowing as well.

## 1. Know and leverage your toolset

Be aware of the framework(s) that you are working in and the tools at your disposal.  Leverage them to the maximum benefit for your business.

If you are using a third party tool like the [Telerik web controls](https://www.telerik.com/) or the [Bootstrap responsive layout framework](https://getbootstrap.com/), make sure you are familiar with all of the different controls or elements within those frameworks.  If you only know two items within the frameworks, you may be cheating the business or your application out of some very useful functionality that could benefit you greatly.

## 2. Keep your toolset current

Make sure you update any components in your toolset as they get updated (but test things as you do it, of course).  A corollary of this point is that you should review the release notes to the extent possible to understand what has been changed in the new version.  You may have some coding work to do in some cases — whether to deal with deprecated functionality or to take advantage of new features that have been made available.  Yet another aside is that by staying current on any kind of framework (even .NET itself or whatever base tech stack you are using), you effectively expand your own development shop by leveraging the resources that have been improving the frameworks you use.

We use the Telerik components in our applications.  By keeping them on the current version (or close to it), we do two things:

* Make it easier to get support when we have issues or questions
* Take advantage of their latest improvements — whether those are bug fixes or additional features

## 3. Look for new tools to add to your toolset

The converse of this is potentially a better way to illustrate the point — “our tools are A, B, and C and we are good with those.”. By adopting such an attitude you will
almost certainly get left behind by advances that don’t necessarily fit into your existing tools.  That isn’t to say just go out and jump on any new
technologies or techniques that you see in the blogosphere or at tech conferences, but rather to objectively evaluate them and how they might apply to your business.

Two examples:

* We started out saying “we don’t need third party controls — we’ll just use the .NET Framework's built in controls” for our applications.  Then we started building and realized that our users wanted (and needed) the interactive and advanced features that were NOT available without custom coding on the core .net controls, but built-in and configurable on third party controls (we chose Telerik).  Done.
* We wanted to expose our functionality on mobile devices but had not yet done so — we had websites, but they were old and did not use any modern frameworks.  By updating our sites and using the Bootstrap framework we were able to make our websites great on mobile devices, but also have the SAME sites look great on tablets and desktops, and did not need to specifically invest in mobile application development.

## 4. Build incrementally – always functional – minimum viable product / feature

Ship your code to production frequently if possible (after testing, unless your are [this guy](http://packetpushers.net/wp-content/uploads/2011/09/test-in-production.png)).  This makes release events not as impactful, risky, or traumatic as they would if you did, say, only one or two releases per year.  And what you’re shipping should always be functional code using the minimum viable product/feature approach.

{{% notice tip Tip %}}
A friend of mine refers to this as the "nibble" approach.  This can be applied when building new applications or finding ways to refactor older ones.
{{% /notice %}}

Two examples:

* We do monthly releases to production with on-demand interim releases when needed.
* We had a nebulous project with some broad goals and only a few specifics.  Start with what you KNOW and build and ship it in its entirety, then let your understanding of the rest of what is needed evolve as you live with your shipped code being used in production.  The business is better off as soon as you put your FIRST release into production.

## 5. Get cross cutting concerns right and consistent

The most prominent cross-cutting concerns in my mind are:

* database access
* logging
* security
* error handling

By centralizing these items in some kind of framework appropriate to your organization and adopting them across the various applications, you establish a common “language” that developers can speak even when moving between product / application assignments.  And things are simpler for your support staff — the main differences between applications becomes business functionality rather than plumbing, logging, and/or security configuration.

We use a minimally-wrapped Enterprise Library logging framework in an “Architecture” library that we’ve written and the vast majority of our code references.  By using this, all of our logging is consistent (even across batch jobs, web applications – both MVC and WebForms, and WPF desktop applications with WCF services)- making it easy for developers and support staff to understand.  If you want to see the details of our framework, check out my CodeProject article [Happiness is Good Logging](http://www.codeproject.com/Articles/536532/Happiness-is-good-logging).  🙂

{{% notice tip Update %}}
This post was written quite a while ago (6+ years now) and the concepts still apply, but for logging we now use the excellent [Serilog](https://github.com/serilog/serilog/#readme) library.
{{% /notice %}}

## 6. Don’t over-complicate / over-engineer / over-architect

This goes along with the minimum viable product / feature concept, but I think it’s important enough to mention on its own.  As fast as things change in this industry, the less code you have written the less overhead it is to take a different direction.  And I’ve had my share of missteps along the way where I’m glad I didn’t have *too* much code written that had to be discarded in favor of a new approach.  By building what you KNOW you need and not trying too hard to anticipate things that really might not become a reality you save yourself time to work on other more important known needs.

If you’re deciding if or how to enable some complex configuration options for an applications, ask yourself how often you anticipate them changing.  If you can’t honestly and confidently say “quite often” then you might consider NOT developing the configurability — just change the code when someone wants it changed.

## 7. Continue to maintain code as you touch it

No one wants to maintain spaghetti code, but it ALL gets that way unless you make a conscious and ongoing effort to avoid it.  If you have an application or some code that gets touched regularly, as it gets touched you should probably also review it to make sure that it adheres to your current coding and thinking standards (e.g. are functions small and tidy, are variable and method names meaningful, is the error handling / logging that it’s doing accurate).  By investing this small effort on an ongoing basis, you should not really have any time where you need to block off an extended period just to rewrite or refactor code — which is always hard to convince the business to give you.

The most frequently-touched code should be some of your most solid and the reason you’re touching it is because it’s a key and changing are of the business.  If it is not, then evaluate for even deeper refactoring.  Maybe the reason you’re having to touch it so often is because it is in need of deeper care and affection.

## 8. Write efficient code – and address performance when you need to

Modern databases and improved hardware and network infrastructures have done a lot to help developers and the performance of their applications — even when the developers do not write their code efficiently.  Take the time to make sure your code is efficient, and then address performance problems as they arise.  A complex caching infrastructure can be important, but it does raise a lot of other questions and headaches that you might be able to completely avoid.  NOTE:  I’m not advocating “not caching” in general — but rather writing good code and then evaluating performance and caching when you find that you need to or that you’re trying to get more performance gains in a certain area.

We *had* a very complex caching mechanism to store “configuration data” for an application once for each machine running the application.  This made the cache very fast (in memory locally on a machine) but also had many complicating aspects of it — how to coordinate changes to multiple machines when a config value changed and what to do if the cache was somehow stale or corrupted.  In our new approach, we simply don’t cache much and the database buffering and hardware / network speed make this a non-impactful change.

## 9. Review your environment on a monthly basis

This has proven to be a best practice for us and it helps our whole technology team (database, software, hardware, network, etc.) speak a very common language and improved our communication about changes and directions.  The things that are good candidates for review:

* **Database health**
  * What version are ALL of your databases running (includes patch levels)?
  * How is the disk space on every drive on every server hosting your databases?
  * How is the index health of your databases?
  * Any long running stored procs that need attention?
  * Excessive deadlocking or timeouts?
* **Application Health**
  * Where do most errors occur in your applications?
  * What are the slowest transactions in your apps?
  * How fast are the key transactions / activities in your apps?
* **Application Usage**
  * Which features are used the most?
  * What times of day / days of week are your heaviest?
  * How much concurrency do you currently have / support?
  * What browsers are your customers using?
  * Which features are most important to different types of customers?
* **Delivery Health**
  * How often do you introduce an error to production?
  * Have you identified root cause of any production errors?
  * Are you delivering enough value to the business?

Most often evaluating these questions will involve rolling your own reports — assuming you have more than one environment.  For web activity tracking, Google Analytics and AWStats will take you only so far — often times to get functionally interesting metrics and/or notes you may have to do some custom coding and/or reporting of your own.  

## 10. Have reasons for what you do and be able to explain them clearly – “it depends”

When asked almost any interesting question about our line of work, the answer is often “it depends” and for good reason.  Many factors go into providing an answer that the inquirer often believes is simple.  So-called “cookie-cutter” approaches or “one size fits all” solutions are don’t always address the needs of *your* organization in the right way. The core principle here in my mind is that if you can’t justify why your are doing something in terms that your stakeholders can understand — not just “it’s the latest thing” — than you have no business doing that something.

A developer on our team started down the path of using a factory pattern for some of the code he was working on — he had read about it and believed that it was important to use.  When asked why he wanted to use it, he just said the same thing – he read about it and it seemed important.  He couldn’t elaborate beyond this.  It turned out NOT to make sense in this case and he didn’t have a very good grasp of when they should be used — the problem they are trying to address.  If we are writing code “to write code” and not “solve a business problem in an efficient way” we are probably missing the mark.

## 11. Prefer good names to comments

Use good names for variables, methods, functions, classes, etc rather than try to explain what they do in comments.  The comments will get stale over time but the
names you choose will have more of a tendency to stay up-to-date.

## 12. Always include a "readme"

Having a `readme` file in your code repository that describes how a new developer can run the solution/project and contribute to it, and even some notes about
how it's organized is critical in giving your code a longer lifespan.
