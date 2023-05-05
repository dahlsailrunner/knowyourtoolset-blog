---
title: "Cancellation Tokens in ASP.NET APIs" # Title of the blog post.
date: 2023-05-04T02:51:55-05:00 # Date of post creation.
description: "The use case and benefit of cancellation tokens in ASP.NET APIs" # Description used for search engine.
codeMaxLines: 30 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: true # Override global value for showing of line numbers within code block.
thumbnail: "" # Sets thumbnail image appearing inside card on homepage.
shareImage: "" # Designate a separate image for social media sharing.
toc: true
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - C#
  - ASP.NET
---

## Background

*If you're just looking for the code, it's here:
[https://github.com/dahlsailrunner/async-vs-sync](https://github.com/dahlsailrunner/async-vs-sync)*

The most recent post I did was all about the improvements that asynchronous code can provide
a website when it experiences higher load levels. That led to the notion of cancellation
tokens, which I was initially pretty confused about.

An oversimplified summary, but one that might provide some good initial benefits, is
that they can be useful when you have longer-running API methods that do read-only
operations - things like dashboard API calls, or pages with lots of content that
is resource-intensive to retrieve.

{{% notice tip "When to Use Cancellation Tokens" %}}
As a starting point, use cancellation tokens when you have resource-intensive read operations that
take a long time to run ("long time" is relative, but generally more than at least a second
or two).
{{% /notice %}}

## Use Case for Experimentation

I created an API method `GET /LongRead` in a sample project.  The "logic" method is shown
here, and it just iterates 10 times calling a repository method asynchronously.

```C#
public async Task<string> GetSequentialLongQueryAsync(CancellationToken token = default)
{
    var resultBuilder = new StringBuilder();
    for (var i = 1; i <= 10; i++)
    {
        resultBuilder.Append(await _carvedRockRepository.GetSequentialLongQuery(i, token));
    }
    return resultBuilder.ToString();
}
```

And here's the repository method - which doesn't make an actual DB query but
does `await Task.Delay` for a second and passes a cancellation token:

```C#
public async Task<string> GetSequentialLongQuery(int sequenceNumber, CancellationToken token = default)
{
    await Task.Delay(1000, token); // simulates long single query
    Log.Information($"Query {sequenceNumber} completed.");
    return $"Query {sequenceNumber} completed.\n";
}
```

Both of the above methods include a `CancellationToken` parameter with a default value.
This makes the parameter *optional* for the caller but allows for cancellation support.

## Cancellation Token as an API Method Parameter

It's worth mentioning the controller action on the API - here's the code:

```C#
[HttpGet]
public async Task<string> Get(bool includeCancellation, CancellationToken token)
{
    if (!includeCancellation)
    {
        token = CancellationToken.None;
    }
    return await _longReadLogic.GetSequentialLongQueryAsync(token);
}
```

In the above code, `includeCancellation` parameter is the only actual querystring
parameter for the API, and it's only present to enable experimentation. You can
choose to use or not use the cancellation token support by setting the
parameter to `true` or `false` respectively.

The `CancellationToken` parameter is something that is not passed by a
caller directly but rather set by the browser or caller to enable the API
to know if a request has been canceled.

So invoking this method is a simple GET call: `GET https://localhost:7213/LongRead?includeCancellation=true`
is an example that will actually use a cancellaton token.

## Testing Cancellation Requests

Actually cancelling a request can be done in a few different ways:

* Use a tool like [Postman](https://www.postman.com/) or [Insomnia](https://insomnia.rest/) to make the request and cancel the request while it's running
* Use the Swagger UI from a ***browser that isn't hooked to your IDE*** and either close the tab/browser while the
request is running or navigate to a new URL in the same tab -- in either case the browser sends a cancellation
request. (If your Swagger UI browser is connected to the IDE while debugging closing the browser
will stop the debugger)
* Use the REST Client extension in VS Code (click the spinning "Waiting" icon) and
cancel the request while it's running (this feature appears to be missing within Visual Studio and Rider HTTP file support as I write this)

{{% notice info "Canceled Requests in Real Life" %}}
In most cases, a request cancellation will be triggered from the browser or HTTP client
that's making the API call when someone navigates away from the page that initiated the
request or simply closes their browser.
{{% /notice %}}

The short video (animated GIF) below shows a cancellation request from Insomnia that
uses cancellation in the first request and notice that the "queries" stop executing and a
`499 Client Closed Request` response is returned from the API. The second request does NOT
use the cancellation token, and when the request is canceled the queries continue to
run and a 200 response is returned.

![cancellation](/images/cancellation.gif)

## Cancellation Response Handling

I have a global exception and response handler in most of my APIs that will return
a `ProblemDetails` response when an error occurs in an API, and without any special
handling, a canceled request like the ones we saw would return a `500 Server Error`
response and the logging would log an unhandled exception of type `TaskCanceledException`.

Cancellations should be considered more normal processing in this case, so I didn't want
to return a `200 OK` or a `500 Server Error`, but rather a `499 Client Closed Request`.

Kristian Hellang's useful [`ProblemDetails` middleware](https://github.com/khellang/Middleware)
makes this pretty easy (note the mapping for the `TaskCanceledException`):

```C#
builder.Services.AddProblemDetails(opts => 
{
    opts.IncludeExceptionDetails = (_, _) => false;
    
    opts.OnBeforeWriteDetails = (_, details) => {
        if (details.Status == 500)
        {
            details.Detail = "An error occurred in our API. Use the trace id when contacting us.";
        }
    };
    opts.MapToStatusCode<TaskCanceledException>(499);  // 499 Client Closed Request
    opts.MapToStatusCode<Exception>(StatusCodes.Status500InternalServerError);
});
```

## Concluding Thoughts

Cancellation tokens - in this initial use case, anyway - can help you avoid wasting
resources executing code/queries/remote calls in APIs where the caller is no longer
waiting for a response.  This may be especially useful in larger, resource-intensive
pages where dashboard-like content is loaded or other such situations.

Using cancellation tokens "everywhere", though, is not something to be done automatically.
It may not make sense in very short read operations, and for any kind of updates or
transactions with multiple update steps, more caution should be applied and those are
more complex topics for potentially another post here.

But applying cancellation tokens to longer read-only operations may give you some
significant savings in compute / IO resources in circumstances like the ones I've
described in this article.

Happy coding!
