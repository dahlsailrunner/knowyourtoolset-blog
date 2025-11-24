---
title: "Put a Silver Background on Your Comments" # Title of the blog post.
date: 2015-03-11T15:39:00-05:00 # Date of post creation.
summary: "Update your IDE to bring less attention to your comments by applying a silver background to them." # Description used for search engine.
thumbnail: "/images/SilverComments.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/SilverComments.png" # Designate a separate image for social media sharing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - Development
# comment: false # Disable comment if false.
---

Putting a silver color background on your comments is a quick and easy productivity booster for me.  It may seem subtle but I’ve found it way easier to visually recognize comments with the silver background than if I just leave it white.  I do this both within Visual Studio *and* SQL Server Management Studio for SQL statements and procs.

Here are the benefits I get from doing this:

* It’s ridiculously easy for me to ignore comments if I’m trying to see and understand the code that is actually “in play”
* I can easily single out comments and review them if I choose to do so

![::img-med img-center img-shadow](/images/SilverComments.png)

To do this for yourself, it’s a simple Options change to VS / SSMS (Tools->Options in both places) — just change the Item background setting for Comment:

![::img-center](/images/SilverCommentsOption.png)
