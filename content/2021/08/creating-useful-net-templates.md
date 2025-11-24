---
title: "Creating Useful .NET Templates" 
date: 2021-08-09T15:29:04-05:00 
summary: "Creating .NET templates that use the templating enging and GitHub actions for NuGet packages and API projects." 
featured: true 
toc: true
# menu: main
#featureImage: "/images/path/file.jpg" # Sets featured image on blog post.
#thumbnail: "/images/path/thumbnail.png" # Sets thumbnail image appearing inside card on homepage.
#shareImage: "/images/path/share.png" # Designate a separate image for social media sharing.
codeMaxLines: 15 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - .NET
  - ASP.NET
  - C#
  - templates
---

***tl;dr - just show me the code:*** <https://github.com/dahlsailrunner/knowyourtoolset-templates>

Templates in .NET are a very helpful feature that enable you to create new files, projects, or even whole solutions with some form of the dotnet new command. I have created and evolved some templates over the past few years and have found them to come in VERY handy when wanting to try something out, explain something, or start new projects.

This article will be jam-packed with helpful information surrounding the notion of templates:

* Why use templates in the first place?
* The “experience” you’re after when creating templates
* Setting up a template pack
* Creating .NET templates and some recommendations
* What types of templates to create

All of the above will be discussed in the context of a new template pack that I just created – and part of this article will be showing you how to use this pack.

## Why Use Templates?

Here’s a list of situations where having some templates at your fingertips comes in really handy:

* **Assistance in breaking down monolith(s)** — if you’re working in an effort like this, being able to create new projects / solutions fast definitely helps – especially if you have your own cross-cutting concerns (e.g. logging, security, data access, test approach) already in the templates
* **Show recommended practices** — templates can be an encapsulation of what you (and your team/org) thinks are best practices or recommended approach for doing certain things (especially technically complex ones). Being able to create a new project that uses one of these practices and then being able to step through it is a great way to enable developers to understand via debugging without a lot of setup overhead.
* **“Runway” to features/logic** — as mentioned already, including your own approach for logging, or security on an API, or how to do data access can enable developers to focus more quickly on the ***logic and functionality*** of the application rather than how to get some basic (and important!!) setup accomplished
* **Quickly try new ideas** — if you want to do an experiment with a new approach or see how something will fit in to your applications, it’s very handy to instantiate your own templates versus the base .NET ones – you have more things set up already like logging and security.

## The “Experience” You Should Seek to Achieve

When creating a template pack, you are trying to make things easier down the road for yourself AND for other developers who may have occasion to use your template pack. So keep this in mind when creating them. So you can see more clearly what I’ve got in mind when I say this, I’d strongly suggest you try out the template pack I just created for yourself.

The template pack I created is here: <https://github.com/dahlsailrunner/knowyourtoolset-templates>

The readme includes a badge / link to the package on NuGet, and two templates:

* `kyt-package`: a template that creates a solution for a NuGet package. It has a project that contains a class library and references [MinVer](https://github.com/adamralph/minver) and [Serilog](https://github.com/serilog/serilog). MinVer provides handy automatic pre-release versioning on push and formal release versioning when GitHub releases (tags) are created. The template also includes a GitHub workflow that will publish your package to NuGet assuming you create a secret called `NUGET_API_KEY` in your own repo. It also includes instructions you should follow (like updating the icon, author name, etc).
* `kyt-backend`: A more complex template for an API backend. It is a three-project solution – the API, a “logic” library, and unit tests. The API requires JWT bearer tokens for authentication (from the [demo DuendeSoftware IdentityServer](https://demo.duendesoftware.com/) but you could update this to use any OAuth2/OIDC server), supports versioned APIs, does logging to the console and [Seq](https://datalust.co/seq) and demonstrates exception handling. It also includes a package reference to [KnowYourToolset.ApiComponents](https://www.nuget.org/packages/KnowYourToolset.ApiComponents/) – which I created and published using the `kyt-package` template above — that provides some middleware for returning [ProblemDetails](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.problemdetails?view=aspnetcore-5.0) responses when exceptions occur.

I encourage you to do at least the simple steps that follow – it will take 5 minutes or so unless you find yourself exploring a bit. Note that the content of my templates is not REALLY the point of this article. Just see how they work and get a sense of what you might create on your own. If you find these templates helpful – that’s a nice bonus!

{{% notice note Note %}}
You need to have the [.NET SDK](https://dotnet.microsoft.com/download) installed to do this – the templates use .NET Standard 2.1 and .NET 6
{{% /notice %}}

### Install the templates

```bash
  dotnet new -i KnowYourToolset.Templates
```

This will add two new items to your list of available templates. The short names are as shown above and they will show up somewhere in the middle of the list of available templates you see when the command finishes.

### Use the `kyt-package` template

In the terminal, change into some directory where you can put demo projects. I use `C:\users\<username>\source\demos`.

Come up with a name for a package, maybe something like `Sample.Package`. If you’ve got something you were meaning to create as a NuGet package, use that. 🙂

Then use the following command (replace `Sample.Package` with whatever you just decided on as the name for your package):

```bash
dotnet new kyt-package -o Sample.Package
```

This will create a new subdirectory called `Sample.Package` under where you just were. Open the new solution in whatever IDE you choose. There should be an `Instructions.md` document that walks you through some stuff.

Feel free to add some code. The steps below are what you would do if you want to publish an actual package to NuGet (go ahead and try it!!):

* Create a GitHub repo and get this code committed to a branch called main
* Get logged in to NuGet.org (you may need to create a free account) and create an API Key
* Add a secret to your GitHub repo (**Settings->Secrets->New repository secret**) called `NUGET_API_KEY` with the value of the key you created
* Commit some kind of change to the repo to trigger the workflow I included in the template – you will have a pre-release version of your package published to NuGet.org!! The version will be something like `0.0.0-alpha.0.0`. You should be able to see this in your IDE if you “include pre-release” packages or just view it on NuGet.org. 🙂

There are more details in the Instructions.md and readme, but by now you get the point of this package.

### Use the `kyt-backend` template

In your “demos” directory (or whatever you decided), here’s the command to use this template (come up with your own name for an API if you like:

```bash
dotnet new kyt-backend -o BookClub.BackEnd
```

This will create a new folder called `BookClub.BackEnd`, and you can open the solution created there. It’s immediately runnable so just go ahead and run it and you should see a Swagger / OpenAPI UI. Try these things:

* Try the Weather controller `GET` method (the only one) – use anything for a postal code – doesn’t matter what. You’ll get a `401` response.
* Click the Authorize button at the top of the page and the “select all” for the scopes – you’ll be redirected to sign in via the demo DuendeIdentityServer. Sign in with the creds available on that page and you’ll end up back at the Swagger UI. Try the method now and it should return a `200` response with some data.
* When authorized (see last step), try `11111` as the postal code. This should return a `500` internal server error with a `ProblemDetails` object containing only a generic message.
* When authorized (see above), try `22222` as the postal code. This should return a `400` bad request with an additional note in the `ProblemDetails` object.
* Run the tests in the project
* Note the health check endpoint (`/health`)

The logs in the project are written to the console (go ahead and check) but they are ALSO written to a local instance of Seq – to get one of these running if you don’t already have it (you need Docker Desktop installed):

```bash
docker pull datalust/seq
docker run -d --name seq -e ACCEPT_EULA=Y -p 5341:80 datalust/seq:latest
```

Then you can explore logs at [http://localhost:5341](http://localhost:5341).

BUT WAIT! There’s one more thing! 🙂

In the “demos” directory again, use the following command to create a version of the solution that includes Docker support baked in (assuming you have [Docker Desktop](https://www.docker.com/products/docker-desktop) installed) — note the `-D` flag:

```bash
dotnet new kyt-backend -D -o BookClub.Wow
```

If you open this new solution that was created, and change the run profile to Docker, you’re ready with a containerized version of the same solution. This optional -D flag is something we’ll look at more below. NOTE: If you’re using Seq, modify the Program.cs file to write to the new location as indicated by the comments at the bottom of the file.

### (Optional) Uninstall the Templates

If you want to get rid of these templates, it’s easy:

```bash
dotnet new -u KnowYourToolset.Templates
```

## Setting Up a Template Pack

The basic documentation for setting up a template pack is here if you want to see it.

The basic steps:

* Create a directory for your templates
* Put a csproj or nuspec file in it and fill it out (use mine as examples — see below for the distinction)
* (optional) Add an icon to the directory (128×128 png or jpg)
* Add a readme to the directory
* Create a `templates` subdirectory
* Add a subdirectory and content for each template you’re creating in the templates folder
* “Pack” your template pack
* Get it published

### .csproj or .nuspec??

I used a .nuspec file instead of the .csproj that recommended in the MS docs. Why? I wanted to include .gitignore files in the templates to make it simpler on repo creation and push. I wasn’t able to figure out how to include those files using dotnet pack, but the nuget pack command includes a -NoDefaultExcludes option that achieved this. (PRs welcome if you know of a way I can do this.)

### The Readme

Maybe it goes without saying, but I believe it’s super important to get a good readme in your repo. Tell people how to use your template pack — what are the available templates, why they’re useful, and how to instantiate them.

### .nuspec File Contents and Some Notes

Here’s the .nuspec file I’m using (the .csproj is also in the repo and is similar but not used due to the .gitignore issue above):

```xml
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2010/07/nuspec.xsd">
  <metadata>
    <id>KnowYourToolset.Templates</id>
    <version>0.3.1</version>
    <authors>Erik Dahl</authors>
    <description>
Sample .NET templates that demonstrate useful templating features.
 
See the **readme** on the project site for more information.
 
https://github.com/dahlsailrunner/knowyourtoolset-templates      
    </description>
    <license type="expression">MS-PL</license>
    <readme>README.md</readme>
    <projectUrl>https://github.com/dahlsailrunner/knowyourtoolset-templates</projectUrl>    
    <title>KnowYourToolset Templates</title>
    <icon>icon.png</icon>
    <packageTypes>
      <packageType name="Template" />
    </packageTypes>
  </metadata>
</package>
```

Most of the content above is pretty straight-forward, but a couple of items are worth pointing out:

* The `<readme>` item will include the readme on the nuget.org page for your package. Pretty handy.
* If you’re publishing packages publicly, make sure you have a `<license>` specified. Valid options are the Expression column [here]().
* The `<version>` is something you need to specifically set each time you publish. You may get 403 errors if you try to publish the same version more than once.

### Building and Publishing the Template Pack

You can create the template pack locally if you want to try it out yourself before publishing publicly.

If you’re using a .nuspec file like I am with this pack, you need the NuGet CLI executable on your machine and in your PATH.

Then you can use the following command to build the template pack:

```bash
nuget pack *.nuspec -NoDefaultExclusions
```

It will create a `.nupkg` file (with a version number) in your current directory.

To install the template pack:

```bash
dotnet new -i ./YOUR_NUPKG_FILENAME
```

To publish the template pack, a nuget push is involved. The GitHub workflow that I have in my pack would work fine for you if you want to use NuGet.org, but if you are publishing to a different location you would need to make changes. But ***get something hooked up where this will auto-publish when you update the main branch of your template repository!***

## Creating Templates

You’ll put the actual templates you want to create in the templates directory you created for the template pack.

These are almost like any other project you would create. The main differences are:

* You should name them something like `MyPackName.TemplateType`
* The `.template.config/template.json` file in the template root directory
* I recommend including an `Instructions.md` and a `Readme.md`
* Another recommendation: include workflow / pipeline file(s)

### Creating the Project / File / Solution for the template

The content you create in the templates directory will become the foundation for your templates. To create the templates in the pack above, I used Visual Studio with **File->New project/solution** and created both the `KnowYourToolset.Package` template and the `KnowYourToolset.BackEnd` template.

**PRO TIP 1:** Include a “.” in your project/solution name! If you don’t include a “.” in the project / solution name, and someone provides a `-o` parameter value of “MyOrganization.MyDomain” (with a “.” in it), the templating engine has trouble renaming things properly. For more information, see this page: <https://github.com/dotnet/templating/wiki/Naming-and-default-value-forms>

**PRO TIP 2:** Keep these projects / solutions debug-able and buildable! As I was developing the templates above, I would run and debug them in Visual Studio just like any other project. The names are a little “meta” to be sure, but the general experience is the same and this enables you to modify the templates with more confidence and simplicity.

### The template.json file

I’ll walk through both `template.json` files in the template pack I created — they need to be in a directory called `.template.config`. Here’s the simpler one for the NuGet package template:

```json
{
  "$schema": "http://json.schemastore.org/template",
  "author": "Erik Dahl",
  "classifications": [ "KnowYourToolset Package" ],
  "tags": { "language": "C#"},
  "identity": "KnowYourToolset.NuGetPackage",
  "name": "KnowYourToolset NuGet Package",
  "shortName": "kyt-package",
  "sourceName": "KnowYourToolset.Package",
  "preferNameDirectory": true
} 
```

* `classifications`: This is a list of describers for your template, and if you start all of them with the same text they will sort together in the dotnet new output.
* `identity`: This is a unique describer for your package — cannot overlap with other templates (yours or otherwise)
* `shortName`: This is the value that developers will use with dotnet new to instantiate your template
* `sourceName`: This is the value in your project/solution that will get replaced with the user-provided value in the -o parameter. Make sure this matches the name you chose when creating your template project / solution. The value above matches my project / solution name if you look at the template repo.
* `preferNameDirectory`: This will create a subdirectory for the instantiated template if true

The above file is pretty much a “base case” for a `template.json` file. The one I used for the API solution has all of the same elements but some additional ones as well:

```json
{
  "$schema": "http://json.schemastore.org/template",
  "author": "Erik Dahl",
  "classifications": [ "KnowYourToolset API" ],
  "tags": { "language": "C#"},
  "identity": "KnowYourToolset.BackEnd",
  "name": "Multi-project API: API, Logic, and Tests",
  "shortName": "kyt-backend",
  "sourceName": "KnowYourToolset.BackEnd",
  "preferNameDirectory": true,
  "guids": [ "f4850358-2d63-41fe-8bb9-e03d0fc6f2ed" ],
  "symbols": {
    "DockerSupport": {
      "type": "parameter",
      "datatype": "bool",
      "defaultValue": "false"
    }
  },
  "sources": [
    {
      "modifiers": [
        {
          "condition": "(!DockerSupport)",
          "exclude": [
            "**/Dockerfile",
            "**/.dockerignore"           
          ]
        },
        {
          "condition": "(DockerSupport)",
          "exclude": [            
             
          ]
        }
      ]
    }
  ],
  "SpecialCustomOperations": {
    "**.csproj": {
      "operations": [
        {
          "type": "conditional",
          "configuration": {
            "if": [ "<!--#if" ],
            "endif": [ "<!--#endif" ],
            "wholeLine": true,
            "evaluator": "C++"
          }
        }
      ]
    }
  }
} 
```

The top of the file is almost the same as the simple one, but also includes two new items:

* guids: This is a handy feature that will take any guids you list here that appear in your template files and replace them with a new guid. How cool is that?! If you look at the .csproj for the API project, it has a guid in it that matches the number above, and if you instantiate this template, the new project will have a different guid.
* symbols: This is the way I provided that -D (or –DockerSupport) option for the kyt-backend template. It’s just a true/false parameter that is set to true when present. If you type dotnet new kyt-backend --help you will see the available options for this template.

Having the symbol of DockerSupport defined, then we can indicate by the sources element which files we want to include or exclude if the DockerSupport condition is true (or not). This is the way that the Dockerfile and .dockerignore files don’t show up if you don’t provide a -D option. Sweet!

Additionally, the SpecialCustomOperations element lets us modify the contents of files to include or exclude lines based on those same conditions. The condition I’ve defined above applies to .csproj files — have a look at the KnowYourToolset.BackEnd.Api.csproj file to see how it’s used.

{{% notice warning "Caution!" %}}
Make sure you keep your templaate projects runnable / debug-able if you are using conditional items!
{{% /notice %}}

For more information about the available options here (I definitely haven’t covered them all, see the wiki here: <https://github.com/dotnet/templating/wiki> (especially the page called **Reference for template.json**).

### Instructions and Readme

For templates, I like having a distinction between “Instructions” and the “Readme”. The Readme should become the Readme for the new instance of the template. Like if I created a template for a BookClub application by doing:

```bash
dotnet new kyt-backend -o BookClub.BackEnd
```

The Readme for that project should have some info about the BookClub domain, how to run my project and any rules for contributing to it. That content might be totally different for a “Heroes” back end created with the same template.

In contrast, the “Instructions” are what do with the template once you’ve created it. In theory, this file could be completely removed once the new project has gotten established.

### Pipeline / Workflow Files

Another recommendation I have is to include pipeline or workflow files when possible. An example is the included GitHub workflow file to publish a NuGet package in the `kyt-package` template.

I haven’t done this yet for the API template, but here are the possibilities there:

* A workflow that will run the unit tests when a PR is created
* A workflow for the containerized version that will publish a container image to hub.docker.com when main is updated

## Template Ideas

This has been a longer article than I intended, so thanks if you’ve stuck with it for this long!! 🙂

Here are some ideas for templates you might want to create on your own:

* Publish nuget packages to your own nuget repository (like MyGet or Azure DevOps)
* Your own API projects with your own cross-cutting concerns
* Worker service apps
* Console apps
* UI applications

Maybe even include Kubernetes manifests optionally for some of the above.

I hope this helps in your own efforts to create useful templates.
