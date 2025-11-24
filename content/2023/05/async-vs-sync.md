---
title: "Async Versus Sync Code in ASP.NET APIs" # Title of the blog post.
date: 2023-05-01T05:51:55-05:00 # Date of post creation.
summary: "The significant difference between async and sync code under load in ASP.NET APIs" # Description used for search engine.
codeMaxLines: 30 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: true # Override global value for showing of line numbers within code block.
thumbnail: "" # Sets thumbnail image appearing inside card on homepage.
shareImage: "" # Designate a separate image for social media sharing.
toc: true
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - C#
  - ASP.NET
  - NBomber  
---

## Background

*If you're just looking for the code, it's here:
[https://github.com/dahlsailrunner/async-vs-sync](https://github.com/dahlsailrunner/async-vs-sync)*

I have recently been doing more work relative to performance, and
one of the things I wanted to more specifically test and quantify
with concrete results was differences between async and synchronous
code (I'll use "async" to refer to asynchronous code and "synchronous"
as the other word because the long (or short) words together are almost
the same).  [NBomber](https://nbomber.com/) is a great .NET-based
load testing package that makes this testing pretty straighforward.

## A Simple API

I created a simple API project that uses EF Core and a SQLite database.

There are two controller actions on the API:

* `GET /Products`: async for everything
* `GET /SyncProducts`: syncrhonous for everything

I also added a 400-millisecond "sleep" to each operation - the async version
uses `await Task.Delay` and the synchronous version uses `Thread.Sleep`.

Here is the repository code that gets the data:

```C#
public async Task<List<Product>> GetProductListAsync(string category)
{
    await Task.Delay(400); // simulates heavy query
    return await _ctx.Products.Where(p => p.Category == category || category == "all")
        .ToListAsync();
}

public List<Product> GetProductList(string category)
{
    Thread.Sleep(400); // simulates heavy query
    return _ctx.Products.Where(p => p.Category == category || category == "all")
        .ToList();
}
```

## Basic Results

It might not surprise you that if you just run these different
endpoints from the Swagger UI, they behave about the same - taking
about a half-second each.

It's only when we can generate some load / concurrency against the API
that we should see some differences in performance.

## Using NBomber for Performance Tests

NBomber is a great package for creating simple performance tests and
can [simulate load using different models](https://nbomber.com/docs/using-nbomber/basic-api/load-simulation).

A basic `Scenario` in NBomber might look something like this:

```C#
var asyncHttpClient = new HttpClient();
var asyncScenario = Scenario.Create("ASYNC requests", async context =>
{
    var requestParameter = ApiParams.GetNextItem(context.ScenarioInfo);

    var request = Http.CreateRequest("GET", $"{BaseUrl}/Product?Category={requestParameter}");

    var clientArgs = new HttpClientArgs(
        httpCompletion: HttpCompletionOption.ResponseContentRead,
        cancellationToken: CancellationToken.None
    );

    return await Http.Send(asyncHttpClient, clientArgs, request);
});
```

There are different ways to provide an `HttpClient` and my method
above is a pretty simple one.  But then you can basically just
set up the HTTP request you want to make and use the `HttpClientArgs`
to indicate what "complete" means to the test of each scenario.  In
the above example I want to have read all of the content.

And note this line:

```C#
var requestParameter = ApiParams.GetNextItem(context.ScenarioInfo);
```

The above code uses the very handy `DataFeed` functionality
within NBomber to get a random query string parameter
from a list I set up for the `DataFeed`:

```C#
private static readonly IDataFeed<string> ApiParams = DataFeed.Random(new List<string> { "all", "boots", "equip", "kayak" });
```

Then you can set up the "load simulation" using the fluent API syntax;
an example is shown here:

```C#
asyncScenario = asyncScenario.WithLoadSimulations(Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)));
```

The code above will create 100 new requests each second for a duration of
30 seconds, meaning 3,000 requests will be made over the course of
the 30 seconds.  Other patterns exist - you can ramp up or down,
wait for a warm-up period, and more.  Check the docs mentioned above.

## Results

The results are pretty remarkable.  Note specifically the `latency`
and `latency percentile` values in the tables below - first for the
synchronous test and second for the async one.

For the p50 latency percentile, ***almost 14 seconds for
the synchronous code, and only 410 milliseconds for the
async version of the same!***

### Synchronous Results

load simulations:

* `inject`, rate: `100`, interval: `00:00:01`, during: `00:00:30`

|step|ok stats|
|---|---|
|name|`global information`|
|request count|all = `3000`, ok = `3000`, RPS = `100`|
|latency|min = `409.5`, mean = `13982.73`, max = `28544.16`, StdDev = `6077.32`|
|latency percentile|p50 = `13746.18`, p75 = `19054.59`, p95 = `22691.84`, p99 = `24854.53`|
|data transfer|min = `0.244` KB, mean = `0.734` KB, max = `1.408` KB, all = `2.2` MB|

### Async Results

load simulations:

* `inject`, rate: `100`, interval: `00:00:01`, during: `00:00:30`

|step|ok stats|
|---|---|
|name|`global information`|
|request count|all = `3000`, ok = `3000`, RPS = `100`|
|latency|min = `402.34`, mean = `411.99`, max = `437.64`, StdDev = `5.2`|
|latency percentile|p50 = `410.88`, p75 = `412.93`, p95 = `424.7`, p99 = `430.34`|
|data transfer|min = `0.244` KB, mean = `0.737` KB, max = `1.408` KB, all = `2.2` MB|

Feel free to experiment with this simple repo and its tests on your own.
For example, see if you can find the point at which the async code
in this code becomes noticeably better than the synchronous version!

As noted above, the full working code is here: [https://github.com/dahlsailrunner/async-vs-sync](https://github.com/dahlsailrunner/async-vs-sync)

Happy coding!
