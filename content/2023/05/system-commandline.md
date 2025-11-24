---
title: "Using the System.CommandLine Package to Create Great CLI Programs" # Title of the blog post.
date: 2023-05-06T05:51:55-05:00 # Date of post creation.
summary: "System.CommandLine adds a great usability boost to almost any .NET console app you may create." # Description used for search engine.
codeMaxLines: 30 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: true # Override global value for showing of line numbers within code block.
thumbnail: "/images/CLI-iteration-1.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/CLI-iteration-1.png" # Designate a separate image for social media sharing.
toc: true
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - C#
  - System.CommandLine
  - NBomber  
---

## Problem Statement

A [recent blog post]({{< ref async-vs-sync >}}) of mine
showed a significant performance difference between an
async API method and a synchronous one. The performance
test was generating 100 new requests each second for 30
seconds, making a total of 3,000 requests.

The async code behaved much better under this load, but
at what level of load did this really become true?

I hard-coded both the test to be run as well as the
injection rate for the requests.

In this post I want to walk through creating a more
flexible CLI experience to run the tests such that we
can more easily find the "tipping point" between async
and synchronous for my example, and we'll use the new
[System.CommandLine](https://learn.microsoft.com/en-us/dotnet/standard/commandline/) package to do it.

I'll do this over a few "iterations" so that (hopefully)
it's clear how this all fits together.

{{% notice info "Still in Preview" %}}
As I write this post, System.CommandLine is still in a preview
status (2.0.0-beta4). It seems pretty apparent that this will
move out of preview before too long, as the functionality seems
mature and very useful.
{{% /notice %}}

## Iteration 1: Get Started with System.CommandLine

*Here's the code for this iteration: [https://github.com/dahlsailrunner/async-vs-sync/tree/cli-iteration-1](https://github.com/dahlsailrunner/async-vs-sync/tree/cli-iteration-1)*

Here's what we're after in this iteration:

![iteration-1-helptext](/images/CLI-iteration-1.png)

Couple of things to notice in the above:

- The help content is auto-generated. This useful
content eliminates the need to go scouring a code base
to determine the usage arguments for a console app.
- Note the alternative options for each argument (both
`--url` and `-u` work for the first option as an example)
- Note the first option is showing as `REQUIRED`
- Note the second option has a `default` value of 30

### Iteration 1 Code Blocks and Explanations

The code for System.CommandLine usage is basically a set of predictable steps:

- Define options
- Define Command(s) (and Handlers)
- Invoke the `RootCommand`

#### Define Options

The CLI we're after right now has two options that we need to
define - a base URI option for the URL that our tests will
run against, and the injection rate for the requests.

Here's the code that does that:

```C#
var (baseUriOption, injectionRateOption) = DefineGlobalOptions();
//...
(Option<Uri> BaseUrlOption, Option<int> InjectionRateOption) DefineGlobalOptions()
{
    var baseUrlOption = new Option<Uri>(
        "--url", "The base URL to test, e.g. https://localhost:7213")
    {
        IsRequired = true
    };
    baseUrlOption.AddAlias("-u");

    var injectionRate = new Option<int>("--rate", 
        "Injection rate. Number of new requests to generate each second.");
    injectionRate.SetDefaultValue(30);
    injectionRate.AddAlias("-r");

    return (baseUrlOption, injectionRate);
}
```

The first line is a top-level statement that calls a
a method to define the options (this could have been
done in-line but I like to separate things into individual
methods for certain things).

The method `DefineGlobalOptions` returns a tuple containing
two options. The code for each option is pretty self-explanatory
and some content is likely recognizable based on the screenshot
above.

#### Define Command(s) and Handlers

The CLI app will have a single command (for now) that will be invoked when
the program runs and the options are validated.  That is defined as a `RootCommand`
which has a description that can be specified, as well as the options that are part
of it.

```C#
var rootCommand = new RootCommand("CarvedRock Performance Test CLI")
{
    baseUriOption,
    injectionRateOption
};
```

Setting up the handler for the command is done as follows:

```C#
rootCommand.SetHandler(DoSomething, baseUriOption, injectionRateOption);
```

The above line bears a little explanation. The `DoSomething` parameter is a
method that takes arguments of type `Uri` and `int` based on the `Option<Uri>`
and `Option<int>` parameters that follow it (the `Option` parameters are of
type `IValueDescriptor<T>`) -- note the generic types of the `Option` values.

Then the `DoSomething` method is pretty straight-forward:

```C#
void DoSomething(Uri uriArgument, int injectionRate)
{
    Console.WriteLine($"The base URL is {uriArgument}");
    Console.WriteLine($"The injection rate is {injectionRate} requests per second");
}
```

#### Invoke the RootCommand

The last thing to do for iteration 1 is to actually invoke the `RootCommand` that
we've defined:

```C#
await rootCommand.InvokeAsync(args);
```

At this point we should have a working app that can be run.

### Running the App

To run the app directly from your IDE (Visual Studio, Rider, VS Code) you can
specify command line arguments in the profile:

```json
"profiles": {
    "CarvedRock.PerformanceTest.Cli": {
        "commandName": "Project",
        "commandLineArgs": "-h"
    }
}
```

Replace the `commandLineArgs` with whatever you want to pass.

Alternatively, you can run it more directly from the command line with either
`dotnet run` or after doing a `dotnet publish/build`.

If you do a `dotnet run` you need to be aware that you may have defined some
options that may conflict with options in `dotnet run`.

For example:

```bash
dotnet run -u https://localhost:7213 -r 40   # won't work: -r conflicts with runtime arg
dotnet run -u https://localhost:7213 --rate 40 # works fine
```

If you've done a `dotnet publish` or a `dotnet build` you can run the command
without those conflict worries:

```bash
./CarvedRock.PerformanceTest.Cli.exe -u https://localhost:7213 -r 40   # works
./CarvedRock.PerformanceTest.Cli.exe -u https://localhost:7213 --rate 40 # also works
```

## Iteration 2: Incorporate the NBomber Async Test

*Here's the code for this iteration: [https://github.com/dahlsailrunner/async-vs-sync/tree/cli-iteration-2](https://github.com/dahlsailrunner/async-vs-sync/tree/cli-iteration-2)*

One of the "real" questions that I wanted to answer with this CLI program was:

> At what point does the async code start to fail and/or go above a 1
> second response time?

So I wanted to try that as the next scenario.

In this iteration, I changed the name of the `DoSomething` method to
a more meaningful name, and created a helper class for the NBomber code
for the load test.

```C#
rootCommand.SetHandler(RunPerformanceTest, baseUriOption, injectionRateOption);
//...
void RunPerformanceTest(Uri baseUri, int injectionRate)
{
    var httpClient = new HttpClient();
    var urlFormat = $"{baseUri}Product?category={{0}}"; // category to be provided dynamically/randomly

    var scenario = NBomberHelper.GetScenario("ASYNC requests", 
        urlFormat, injectionRate, httpClient);

    NBomberRunner.RegisterScenarios(scenario)
        .Run();
}
```

My usage of NBomber has been explained in that
[recent blog post]({{< ref "async-vs-sync.md#using-nbomber-for-performance-tests" >}}),
but the source code for this slightly-changed logic is in the `NBomberHelper.cs` file.

I'm running a single scenario with this, and the scenario name and the path
to be hit will always be on the async version of my API (we'll come back to that
in the next iteration).

But with the iteration two changes in place, we can run a variety of
performance tests against the async API to see where it starts to break:

```bash
# works fine
./CarvedRock.PerformanceTest.Cli.exe -u https://localhost:7213 --rate 400 

# still works fine
./CarvedRock.PerformanceTest.Cli.exe -u https://localhost:7213 --rate 800 

# starts slowing down - some responses almost 3 seconds
./CarvedRock.PerformanceTest.Cli.exe -u https://localhost:7213 --rate 1600 

# slower still, and some "connection refused" errors
./CarvedRock.PerformanceTest.Cli.exe -u https://localhost:7213 --rate 2000 
```

It wasn't until I hit a full 2,000 requests per second that I started getting
"connection refused" errors.  Super impressive, and the CLI helped me quickly
determine that without me having to update code!

## Iteration 3: Add a Sub-Command for the Sync Tests

*Here's the code for this iteration: [https://github.com/dahlsailrunner/async-vs-sync/tree/cli-iteration-3](https://github.com/dahlsailrunner/async-vs-sync/tree/cli-iteration-3)*

To test the syncrhonous version of the API without losing support for the
async version, I opted to use the "sub-command" functionality of System.CommandLine.

There are certainly more than one way to do this, but it gave me the opportunity
to explore sub-commands and also show how they work.

For this iteration, I've moved the async performance test into a sub-command
and created a second sub-command for the synchronous test.

I want the URL and iteration rate options to be options available to both
of the commands.  So here's the new code for the `RootCommand` and its two
sub-commands:

```C#
var rootCommand = new RootCommand("CarvedRock Performance Test CLI");
rootCommand.AddGlobalOption(baseUriOption);
rootCommand.AddGlobalOption(injectionRateOption);

var asyncCommand = new Command("async", "Run load test against the async API endpoint");
asyncCommand.SetHandler((baseUri, injRate) =>
    RunPerformanceTest(baseUri, injRate, "ASYNC scenario", "Product?category={0}"),
    baseUriOption, injectionRateOption);
rootCommand.AddCommand(asyncCommand);

var syncCommand = new Command("sync", "Run load test against the synchronous API endpoint");
syncCommand.SetHandler((baseUri, injRate) =>
        RunPerformanceTest(baseUri, injRate, "SYNCHRONOUS scenario", "SyncProduct?category={0}"),
    baseUriOption, injectionRateOption);
rootCommand.AddCommand(syncCommand);
```

I've added the two options as "Global Options" on the `rootCommand`.

Then I create both the `asyncCommand` and the `syncCommand`, set the handler to a
method invocation with some parameters for each of them, and add the new
sub-commands to the `rootCommand` with calls to `rootCommand.AddCommand()`.

The method invocation is a little different now - since I want to pass some
additional parameters to the method, I can use a lambda expression when calling
`SetHandler`.  The initial arguments `(baseUri, injRate)` will correspond to
the two `Option` values (`IValueDescriptor<T>` as described above).

The updated `RunPerformanceTest` method is as follows:

```C#
void RunPerformanceTest(Uri baseUri, int injectionRate, string scenarioName, string apiRoute)
{
    var httpClient = new HttpClient();
    var urlFormat = $"{baseUri}{apiRoute}"; // category provided dynamically/randomly

    var scenario = NBomberHelper.GetScenario(scenarioName,
        urlFormat, injectionRate, httpClient);

    NBomberRunner.RegisterScenarios(scenario)
        .Run();
}
```

Instead of hard-coded relative paths and scenario names now, the method
accepts them as parameters which get specified with different values from
the `SetHandler` calls.

And with those sub-commands in place, the help text now looks like this:

![cli-iteration-3](/images/cli-iteration-3.png)

Now to run the tests you can run commands like these examples (the first one
runs the synchronous test and the second runs the async one):

```bash
./CarvedRock.PerformanceTest.Cli.exe sync -u https://localhost:7213 --rate 60
./CarvedRock.PerformanceTest.Cli.exe async -u https://localhost:7213 --rate 200
```

{{% notice note "Sync Breaks Down Under Load Fairly Quickly!" %}}
The results of my tests (at least on my machine - a 32GB Surface Book 3)
showed the results of a 60-request per second injection rate on the synchronous
test still working fine but responses climbing into the 2-5 second range, and 75 requests
per second were getting lots of responses in the 5-10 second range, and kept climbing
until around 250 requests per second, when "connection refused" type
errors start happening.

Compare that to the around 2K requests per second that I needed to hit
before the async requests started breaking down!
{{% /notice %}}

## Expansion Thoughts

CLI programs can come in very handy.  Here are some ideas for further
experimentation if you're interested.

- Package the CLI app as a `dotnet tool`, as [described here](https://codeburst.io/creating-a-custom-net-core-global-tool-40cf3c1410c9)
- Model some of the other [NBomber load simulation](https://nbomber.com/docs/using-nbomber/basic-api/load-simulation) techniques as options (not just injection rate)

Happy coding!
