---
title: ".NET Templates with Optional Content" 
date: 2024-11-28T15:29:04-05:00 
summary: "Updating some .NET templates that use the templating enging and GitHub actions for NuGet packages and API projects to include Aspire, and choice of database to use, and an optional UI using the BFF security pattern with
the Duende BFF libraries." 
featured: true 
toc: true
# menu: main
#featureImage: "/images/kyt-aspire-visualstudio.png" # Sets featured image on blog post.
thumbnail: "/images/kyt-aspire-visualstudio.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/kyt-aspire-visualstudio.png" # Designate a separate image for social media sharing.
codeMaxLines: 15 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - .NET
  - ASP.NET
  - C#
  - Aspire
  - BFF
  - Duende Software
  - PostgreSQL
  - SQL Server
  - Open Telemetry
  - templates
---

***tl;dr - just show me the code (and the readme):*** <https://github.com/dahlsailrunner/knowyourtoolset-templates>

{{% notice info "Updates to Templates from a Previous Post" %}}
This post builds upon templates that were described in [an earlier post of mine]({{< ref "/2021/08/creating-useful-net-templates" >}}).  That post has some background information on templates
that you may find useful. This post takes that a bit further into additional usefulness.
{{% /notice %}}

## The End Result

The updated template that this post describes is an Aspire solution
for an API with a database and JWT bearer token authentication.

It includes a **choice for the database** (PostgreSQL, SQL Server, and SQLite).

It also includes an **optional UI** that uses the Backend For Frontend (BFF)
security pattern with the [Duende BFF](https://docs.duendesoftware.com/identityserver/v7/bff/)
packages.  The only choice for the UI at the time I'm writing this is
Angular or None, but you could easily include React, Vue, Blazor, or whatever.

The solution is immediately runnable (just hit F5) and includes some
sample data and working authentication with the [demo instance of the
Duende IdentityServer](https://demo.duendesoftware.com).

Here's an image of the choices in the Visual Studio "new project" flow:

![visual studio::img-med](/images/kyt-aspire-visualstudio.png)

And here's the same thing in Rider:

![rider::img-med](/images/kyt-aspire-rider.png)

You can also create projects using this template from the command line.

The following will create an Aspire project with an API that uses PostgreSQL
as its database and an Angular UI that will call that API via the BFF.

```bash
dotnet new kyt-aspire -D postgres -U angular -o My.Project
```

The following will create an Aspire project with just an API and a
SQLite database (no UI or BFF will be included):

```bash
dotnet new kyt-aspire -D sqlite -o My.Project
```

## Conditional Content for Templates

The foundation of conditional content in .NET templates is giving the
user one or more choices when using the template to create a project.

Enabling the choice of database in the template was done with the
following in `symbols` section of the `.template.config/template.json` file:

```json
"Database": {
    "type": "parameter",
    "description": "Database provider to be used (choose one).",
    "datatype": "choice",
    "allowMultipleValues": false,
    "enableQuotelessLiterals": true,
    "choices": [
    {
        "choice": "postgres",
        "description": "Uses PostgreSQL as the database provider.",
        "displayName": "PostgreSQL"
    },
    {
        "choice": "sqlserver",
        "description": "Uses SQL Server as the database provider.",
        "displayName": "SQL Server"
    },
    {
        "choice": "sqlite",
        "description": "Uses SQLite as the database provider (no traces available).",
        "displayName": "SQLite"
    }
    ],
    "defaultValue": "postgres"
},
```

That content will give the user a choice of database, but it doesn't mean
the code will actually reflect their choice.  To do that, the easiest way
I've found was to define "computed symbols" for each choice that is a
boolean value reflecting the state of that option.

In the same `symbols` section, I added this content:

```json
"POSTGRESQL": {
    "type": "computed",
    "value": "Database == postgres"
},
"MSSQL": {
    "type": "computed",
    "value": "Database == sqlserver"
},
"SQLITE": {
    "type": "computed",
    "value": "Database == sqlite"
},
```

The above defines, for example, a `POSTGRESQL` symbol that will be `true`
if PostgreSQL has been selected as the database choice.

### Updating Source Code to Support Choices

The computed boolean symbols (like `POSTGRESQL` above) can be used in the
template code just like compile-time symbols.  

Here's an example that will add the appropriate `AppDbContext` to the
dependency injection container during startup:

```csharp
#if POSTGRESQL
        builder.AddNpgsqlDbContext<AppDbContext>("DbConn", configureDbContextOptions: opts =>
        {
            opts.ConfigureWarnings(warnings => warnings.Ignore(RelationalEventId.PendingModelChangesWarning));
            opts.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
        });
#endif
#if MSSQL
        builder.AddSqlServerDbContext<AppDbContext>("DbConn", configureDbContextOptions: opts =>
        {
            opts.ConfigureWarnings(warnings => warnings.Ignore(RelationalEventId.PendingModelChangesWarning));
            opts.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
        });
#endif
#if SQLITE
        var dbPath = Path.Join(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), 
                                "KnowYourToolset.BackEnd.sqlite");
        builder.Services.AddDbContext<AppDbContext>(opts => opts
            .UseSqlite($"Data Source={dbPath}")
            .ConfigureWarnings(warnings => warnings.Ignore(RelationalEventId.PendingModelChangesWarning))
            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
#endif
```

With code like that above in your various `.cs` files, the instantiated
template will only include the code appropriate for the database choice.

### Updating Other Files to Support Choices

The above approach works well for `.cs` files, but doesn't work for
Markdown files, `.csproj` files, or `.sln` files.

For those types of files you can use syntax like the following example
from a `.csproj` file to include the correct project references:

```xml
<!--#if (SQLITE)-->
<PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="9.0.0" />
<!--#endif-->
<!--#if (POSTGRESQL)-->
<PackageReference Include="Aspire.Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.0.0" />
<!--#endif-->
<!--#if (MSSQL)-->
<PackageReference Include="Aspire.Microsoft.EntityFrameworkCore.SqlServer" Version="9.0.0" />
<!--#endif-->
```

The same <!--#if (condition) --> syntax will work for Markdown files and solution files.

### Excluding Entire Folders Conditionally

If someone chose "None" as the UI, then we don't want to include the `ui-with-bff` folder
or its contents at all.  

That can be accomplished with the following content in the `template.json` file:

```json
"sources": [
  {
    "modifiers": [
      {
        "condition": "!(ANGULAR)",
        "exclude": ["**/ui-with-bff/**"]
      }
    ]
  }
],
```

You can use the approach this for files, directories, or both.

### Running the Template Project to Verify

I like to have the template itself in a runnable state, and that wouldn't be the case
with all of the conditionals above.

Happily, you can add Debug build constants in the `.csproj` files where it makes
sense:

```xml
<PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|AnyCPU'">
    <DefineConstants>$(DefineConstants);POSTGRESQL</DefineConstants>
</PropertyGroup>
```

To run the template solution locally, you would need to make similar changes
in the AppHost and API `.csproj` files.  For example, if you used `MSSQL,ANGULAR`
as the constants in both projects that would run the solution with SQL Server
as the database and include the Angular UI with BFF.

Omitting those constants from projects when created can use what we've
already reviewed:

Set up a computed symbol in the `template.json` file that is simply
`true`, then use a conditional in the `.csproj` files:

```xml
<!--#if (!IsFromTemplate) -->
<PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|AnyCPU'">
    <DefineConstants>$(DefineConstants);POSTGRESQL</DefineConstants>
</PropertyGroup>
<!--#endif-->
```

### EF Migrations

This is the clumsiest part of this approach. When developing the
templates, I used the compile constants to pick a database - say I started with
`MSSQL` to use SQL Server.

Then I would use the following command to generate the EF Core migration for
the intial schema:

```bash
dotnet ef migrations add Initial -o Data/Migrations
```

When I had that in place, I would take the SQL Server-specific content and
wrap it in `#if MSSQL / #endif` lines.

Then I would **save the three migration files** somewhere else, and repeat
the process for another database.  That let me create a single set of
migration files that would support any database choice.

## Setting AppHost as Startup Project

I wanted developers to be able to simply run the solution after instantiation,
without having to choose a startup project or any other friction.

The first "trick" here was to make the AppHost project ***the first one
listed in the solution (`.sln`) file.***

The next trick only applied to the condition when the Angular UI was included.

I added this content to the `.csproj` file for the BFF:

```xml
<PropertyGroup>
  <!-- File with mtime of last successful npm install -->
  <NpmInstallStampFile>../node_modules/.install-stamp</NpmInstallStampFile>
</PropertyGroup>
<Target Name="NpmInstall" BeforeTargets="BeforeBuild" Inputs="../package.json"
        Outputs="$(NpmInstallStampFile)" Condition="$(Configuration) == Debug">
  <Exec Command="npm install" />
  <Touch Files="$(NpmInstallStampFile)" AlwaysCreate="true" />
</Target>
```

This content will execute `npm install` during a Debug build (I didn't want it
executing during a Release build or a publish activity), and it will only execute
if the package.json file is newer than the "stamp file" created by this process.

So it will execute the first time you run the solution, and then after that only
if you change any package dependencies in `package.json`.

## Opening Readme and Instructions (Visual Studio)

I wanted to have the Readme and Instructions docs open in Visual Studio
when the KnowYourToolset Aspire template was used to create a solution.

To do that, I added the following content to the `template.json`:

```json
"primaryOutputs": [
    {
        "path": "KnowYourToolset.BackEnd.sln"
    },
    {
        "condition": "(HostIdentifier != \"dotnetcli\")",
        "path": "readme.md"
    },
    {
        "condition": "(HostIdentifier != \"dotnetcli\")",
        "path": "instructions.md"
    }
],
"postActions": [
    {
        "condition": "(HostIdentifier != \"dotnetcli\")",
        "description": "Opens the readme in the editor",
        "manualInstructions": [],
        "actionId": "84C0DA21-51C8-4541-9940-6CA19AF04EE6",
        "args": {
        "files": "1;2"
        },
        "continueOnError": true
    }
]
```

## More Information

* [Custom Templates Overview](https://learn.microsoft.com/en-us/dotnet/core/tools/custom-templates)
* [Templating wiki reference](https://github.com/dotnet/templating/wiki)
