---
title: "Adding a TypeScript Task to a TFS 2017 Build for ASP.NET Core" # Title of the blog post.
date: 2018-01-22T11:01:26-05:00 # Date of post creation.
description: "Article description." # Description used for search engine.
thumbnail: "/images/tsconfig-setup.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/tsconfig-setup.png" # Designate a separate image for social media sharing.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - Azure DevOps
  - TypeScript
  - ASP.NET
  - Kendo UI
  - ASP.NET
# comment: false # Disable comment if false.
---

## First Forays into ASP.NET Core (2.0)
I recently have been taking my first forays into ASP.NET Core and am liking what I’m seeing so far. The journey hasn’t been without curveballs, though, and I’ll be posting a few findings here.

## New Project using Kendo UI Core and TypeScript
A project I recently set up using VS 2017 used the ASP.NET Core Web Site template (not the MVC or API templates, and not the React or Angular flavors). I wanted to use the Kendo View Models with TypeScript in the project (see [previous post]({{< ref "/2016/12/typescript-bootstrap-and-kendo-ui-for-jquery-a-powerful-triple-threat" >}})).

After I got the packages added to the project (I used Bower because the Kendo UI Core npm package isn’t immediately usable — you need to set up webpack or some other stuff to “build” the package first) and set up TypeScript based on the configuration noted in my previous post, everything seemed be working great. The VS 2017 build would run the TypeScript compiler, put the built JavaScript output files where I wanted them, and when I ran the project all was well.

## Problem: TFS 2017 didn’t run the TypeScript compiler!
Enter the on-prem TFS 2017 build, which when I set up a build using the ASP.NET Core Preview build template, did ***NOT*** run the TypeScript build. I’m not sure if this is the same as VSTS, but I suspect that it may be.

![::img-center img-med img-shadow](/images/tfs-aspnetcore-preview.png)

I tried a handful of different things to try to get the TypeScript to build as part of the .NET Core Build Task, but couldn’t seem to get anything to work.

## Creating a TypeScript Task in the Build definition
This led me to create a standalone Task in the build definition to do that. Here are the steps I followed to make it happen:

#### 1. Set up a `tsconfig.json` file that will put output JS files into the `wwwroot` folder somewhere
The tsconfig.json file for your project tells the TypeScript compiler how to behave — what to ignore and where to put your output files.

In the image below, I’m showing the `tsconfig.json` contents and the Solution Explorer view of my project. The TypeScript compiler is configured to put compiled output into the `wwwroot/js` folder, and to completely ignore the packages that I’ve downloaded into `node_modules` and `bower_components`.

![::img-shadow img-med img-center](/images/tsconfig-setup.png)

#### 2. Set up the project to always use the latest version of TypeScript
This step is something I did just to make sure that whatever version of the compiler we have installed as the latest version can be used without being specifically tied to a specific version. The diagram below shows the TypeScript Build tab of the Project Properties where this setting exists.

![::img-shadow img-center](/images/project-properties.png)

#### 3. Install TypeScript on your build server
This is just a matter of getting the latest version of TypeScript from https://www.typescriptlang.org/index.html#download-links and installing it on the build server. For my purposes I used the VS 2017 version of TypeScript and installed that.

#### 4. Set up the TypeScript build Task in the build definition
You can start by adding a Command Line task that will be executed right after the .NET Core Build Task in your definition.

![::img-med img-center img-shadow](/images/TypeScriptTask.png)

Make the `Display Name` something meaningful, like “TypeScript Compilation” so that you can easily spot it later on. Then set the tool to `tsc`. If you have `tsc` in the `PATH` environment variable you might be able to get by just specifying `tsc` here. I did not have it in the `PATH`, so I needed to provide the full executable path of `tsc` for it to run.

I didn’t need to specify ANY arguments, and this will try to pick up a `tsconfig.json` file for the build settings from whatever directory you’re running the command in, and is specified in the Advanced section for the “Working folder” option. In my case I had a subdirectory called “Rover” which is where my `tsconfig.json` file was, so I needed to specify that as the Working folder.

Now when you run the build your TypeScript files should be compiled and put into the `wwwroot` folder, which will be included in the published artifact from your build — which means that it will be included when the build is deployed by a Release definition!