---
title: "Code Coverage Reports for .NET Projects" 
date: 2024-01-28T01:51:55-05:00 
summary: "Generating local code coverage reports, and getting pipelines set up to evaluate coverage." 
codeMaxLines: 30 .
codeLineNumbers: true 
thumbnail: "/images/coverage-report.png" 
shareImage: "/images/coverage-report.png" 
toc: true
tags:
  - C#
  - API
  - ASP.NET
  - Testing
  - XUnit
  - Coverlet
  - Code Coverage
  - GitHub Actions
  - Azure DevOps
---

My last 3 posts were all about writing good integration
tests for your ASP.NET Core API projects.

But how much of your code do the tests you've written
actually cover?  And what's missing?

{{% notice tip "Show me the Code!" %}}
The code I used as a reference for this article is in the same repo
as the previous posts I did about integration testing, and the coverage
reporting is done in the
[4-api-with-postgres-and-auth](https://github.com/dahlsailrunner/testing-examples/tree/main/04-api-with-postgres-and-auth) repo.
{{% /notice %}}

This post will show you how to create a code coverage report like
the one shown below using some easy-to-use tools.

![coverage-report](/images/coverage-report.png)

## Summary

Generate a coverage report locally with the following commands:

```bash
dotnet test --settings tests.runsettings
reportgenerator -reports:".\**\TestResults\**\coverage.cobertura.xml" -targetdir:"coverage" -reporttypes:Html
.\coverage\index.htm
```

The final line above just launches the generated report in a browser.

Further explanation follows, along with how to get some coverage information into
automated pipelines - in both GitHub and Azure DevOps!

## Setup

The tools that help make the magic happen are the following:

- The [coverlet.collector NuGet package](https://github.com/coverlet-coverage/coverlet)
  - A `tests.runsettings` file
- The [reportgenerator dotnet tool](https://github.com/danielpalme/ReportGenerator)
- The tests we've already written

Assuming you've got an API and some integration tests, you should be able to
run `dotnet test` in your solution directory and it should run the tests
properly.

## Create a `.runsettings` file

There are many command line options you can pass to `dotnet test`, but I've found
that using a `.runsettings` file is easier.

Here's a starting point (we'll be adding more later) for a `tests.runsettings` file
in the root directory of the solution:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat code coverage">
        <Configuration>
          <Format>cobertura,opencover</Format>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

The above runsettings file just indicates that we want to capture "cross-platform"
code coverage (meaning this will work on Windows or Mac or Linux).

## Run `dotnet test` to generage coverage details

Then we can modify the `dotnet test` command to be:

``` bash
dotnet test --settings tests.runsettings
```

The above command will create a `cobertura.xml` file (as well as an
`opencover.xml` file) that can be used as an input to the report
generation process.

{{% notice info "Opencover not explicitly needed" %}}
You don't actually need the `opencover` version of the coverage
file to do what I'm describing here.  There are lots of options
and this post is meant to get you started with a real solution
that works and show the moving parts so that you can configure them
to your needs.
{{% /notice %}}

The default location for the `xml` files that get created
are a subfolder called `TestResults` for each of the test projects
that get executed.

## Use `reportgenerator` to Generate the Coverage Report

The `reportgenerator` is a global tool that can read the coverage XML file (more
than one to merge them) and create HTML reports that are super easy to read.

To install the tool:

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
```

Once that's installed, you can create the report pretty easily:

```bash
reportgenerator \
    -reports:".\**\TestResults\**\coverage.cobertura.xml" \
    -targetdir:"coverage" \
    -reporttypes:Html
```

The `reports` arg (all of the above is a single command) specifies the path
to your coverage files - use wildcards like I've done if you have more
than one test coverage file to merge.  If you don't have more than one
test project you can be more explicit in the filename.

The `targetdir` parameter is required and indicates where the report
will be placed (note that there are a number of files that are part
of the report, so you probably want some directory like `coverage`).

There are many types of reports available but `Html` is a good place
to start.

{{% notice note "Report Formats" %}}
Here's a list of the report formats from the [docs](https://github.com/danielpalme/ReportGenerator), which you can separated by semicolon:
Badges, Clover, Cobertura, CsvSummary, MarkdownSummary,
MarkdownSummaryGithub, MarkdownDeltaSummary, OpenCover,
Html, Html_Light, Html_Dark, Html_BlueRed, HtmlChart, HtmlInline,
HtmlSummary, Html_BlueRed_Summary, HtmlInline_AzurePipelines, HtmlInline_AzurePipelines_Light, HtmlInline_AzurePipelines_Dark,
JsonSummary, Latex, LatexSummary, lcov, MHtml, SvgChart, SonarQube,
TeamCitySummary, TextSummary, TextDeltaSummary, Xml, XmlSummary
{{% /notice %}}

The command above will generate `coverage/index.htm` (or in a different
subdirectory if you provided a different one) that is shown in the screenshot
above.

Explore away!

Feel free to experiment with other report formats.

## Excluding Items from Coverage

With slight modifications to the `.runsettings` file, you can exclude things
like auto-properties and EF Core migrations from your coverage reporting, which
can give you more accurate coverage results.

Here's a modified `.runsettings` file:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat code coverage">
        <Configuration>
          <Format>cobertura,opencover</Format>
          <Exclude>[*.Tests?]*</Exclude> <!-- [Assembly-Filter]Type-Filter -->
          <ExcludeByFile>**/Migrations/*.cs,</ExcludeByFile> <!-- Globbing filter -->
          <SkipAutoProps>true</SkipAutoProps>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

Note the lines that exclude the `Tests` assemblies, the `Migrations` code files,
and any auto-properties.

Slick!

## Bonus 1: GitHub Actions

As an added bonus, you can set up a GitHub Action to report on your code coverage
on triggers of your choosing.

Here's a sample yaml file (which uses the [Code Coverage Summary](https://github.com/marketplace/actions/code-coverage-summary) action - make sure to expand beyond the initial 30 visible lines to see everything):

```yaml
name: .NET Coverage

on:
  workflow_dispatch:
    branches: [ main ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --no-restore 
    
    - name: Test
      run: dotnet test --no-build --settings tests.runsettings

    - name: Publish coverage
      uses: irongut/CodeCoverageSummary@v1.3.0
      with:
        filename: '**/TestResults/**/coverage.cobertura.xml'
        badge: true
        fail_below_min: true
        format: markdown
        indicators: true
        output: both
        thresholds: '30 60'

    - name: Write to Job Summary
      run: cat code-coverage-results.md >> $GITHUB_STEP_SUMMARY
```

The final step in the above will add to the action pipline summary
as shown in this screenshot:

![job-summary](/images/github-action.png)

If you want to run this when a pull request has been created you
can create a PR comment with the following step at the bottom
of the action YAML:

```yaml
    - name: Add Coverage PR Comment
      uses: marocchino/sticky-pull-request-comment@v2
      if: github.event_name == 'pull_request'
      with:
        recreate: true
        path: code-coverage-results.md
```

Great stuff, this!

## Bonus 2: Azure DevOps Pipelines

Azure DevOps will show you a complete HTML file like the
`reportgenerator` one that was run locally above.

The pipeline for Azure DevOps looks something like this:

```yaml
steps:
- task: UseDotNet@2
  inputs:
    packageType: 'sdk'
    version: '8.0.x'

- task: DotNetCoreCLI@2
  displayName: 'Run tests'
  inputs:
    command: 'test'    
    arguments: '--settings Tests/tests.runsettings' 
    publishTestResults: true

- task: PublishCodeCoverageResults@2
  displayName: 'Publish code coverage: Azure DevOps'
  inputs:
    codeCoverageTool: 'Cobertura'
    summaryFileLocation: '$(Agent.TempDirectory)/**/*cobertura.xml'    
```

Then when your pipeline runs you should have a `Code Coverage` tab
on the pipline results.

Happy coding and testing!
