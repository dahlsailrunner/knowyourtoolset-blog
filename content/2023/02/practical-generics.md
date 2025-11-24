---
title: "Using Generics and Interfaces to Simplify C# Code" # Title of the blog post.
date: 2023-02-03T05:51:55-05:00 # Date of post creation.
summary: "Practical examples of generics and interfaces that can be used to simplify some C# code." # Description used for search engine.
codeMaxLines: 30 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: true # Override global value for showing of line numbers within code block.
#thumbnail: "/images/xxx.png" # Sets thumbnail image appearing inside card on homepage.
#shareImage: "/images/xxx.png" # Designate a separate image for social media sharing.
toc: true
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - C#
  - ASP.NET  
---

## Background

If you're working in .NET or other languages that support **interfaces** and
**generics**, you've probably used them. Finding reasons to create them can
be more challenging, but I recently came across two completely different
instances that clearly benefited from the interface / generic combo and
thought they would be worth sharing.  Enjoy!  :)

## Example 1: Simplify Common Code Assignments

I recently came across an API that was using a very consistent
pattern of including a `List<ErrorInfo>` in `...Response` objects
to include validation errors or other errors that might have occurred.

When errors would occur that would set those properties, the code
looked something like this (note the many lines of code and semi-blank
lines with just curly-braces):

```c#
try
{
    ///... ommitted for brevity
    return new SecurityQuestionsResponse
    {
        ErrorInfo = new List<ErrorInfo>()
        {
            new ErrorInfo()
            {
                Code = "USERAPI_ERROR",
                Message = Messages.USERAPI_ERROR + " - Null respInfo" 
            }
        }
    };
}
catch (Exception)
{
    return new SecurityQuestionsResponse
    {
        ErrorInfo = new List<ErrorInfo>()
        {
            new ErrorInfo()
            {
                Code = "USERAPI_ERROR",
                Message = Messages.USERAPI_ERROR
            }
        }
    };
}
```

There were ***many*** instances like this throughout the
code I was seeing and the observation I had was this:

> Too much "noise" is created by this code which makes it harder
and clumsier to read the "real" logic of the code.

There were many classes that looked similar that created this
same kind of "noise" when reporting errors:

```C#
public class SecurityQuestionsResponse
{
    //... ommitted
    public List<ErrorInfo>? ErrorInfo { get; set; }
}

public class ApplicationSettingsResponse
{
    //... ommitted
    public List<ErrorInfo>? ErrorInfo { get; set; }
}
public class CompanyConfigurationResponse
{
    //... ommitted
    public List<ErrorInfo>? ErrorInfo { get; set; }
}

```

The next observation is that the repeated code to set the
different error responses basically had three moving parts or
variables:

* the `Code` string,
* the `Message` string
* type of the Response - `SecurityQuestionsResponse`, `ApplicationSettingsResponse`, or `CompanyConfigurationResponse`
(there were more `Response` examples, but you get the point)

### Step 1: Create an Interface that marks common traits

The first step in simplifying this was to define a new interface
that identifies the commonality of the `Response` classes, as follows:

```c#
public interface IHaveErrorInfo
{
    List<ErrorInfo>? ErrorInfo { get; set; }
}
```

Then just add the interface marker to each of the classes:

```C#
public class SecurityQuestionsResponse : IHaveErrorInfo { ... }

public class ApplicationSettingsResponse : IHaveErrorInfo { ... }

public class CompanyConfigurationResponse : IHaveErrorInfo { ... }
```

### Step 2: Create a Static Generic Method

With the interface in place, a new static generic method can be defined.  Note
that the `T` argument is constrained to be types that implement the `IHaveErrorInfo` interface (and requires a parameterless constructor).

```C#
public static class QuickError
{
    public static T Create<T>(string code, string message) where T : IHaveErrorInfo, new()
    {
        return new T
        {
            ErrorInfo = new List<ErrorInfo>()
            {
                new() { Code = code, Message = message }
            }
        };
    }
}
```

### Step 3: Simplify Original Code

With the interface and the new generic method in place, the original code
from the snippet above can be simplified to what follows:

```C#
try
{
    ///... ommitted for brevity
    return QuickError.Create<SecurityQuestionsResponse>("USERAPI_ERROR",
        Messages.USERAPI_ERROR + " - Null respInfo");
}
catch (Exception)
{
    return QuickError.Create<SecurityQuestionsResponse>("USERAPI_ERROR",
        Messages.USERAPI_ERROR);
}
```

## Example 2: Encapsulate Paginated API Response-Handling logic

I was calling various methods of an API that had hard limits of
response content of 50 items per call.  I needed the full results
of the various calls, and the `offset` / `limit` functionality of
each endpoint was the same.

I had code in different places that looked like this:

```C#
var fieldsToReturn = "id,name,description,updated"

bool needToGetMoreItems;
var offset = 0;
const int limit = 50;
do
{
    var queryString = $"fields={fieldsToReturn}&limit={limit}&offset={offset}{filter}";
    var response = await client.GetFromJsonAsync<ReleaseApiResponse>($"api/v3/releases?{queryString}");
    if (response?.Items != null)
    {
        releases.AddRange(response.Items);
    }
    needToGetMoreItems = (response?.Items.Count ?? 0) == limit;
    offset += response?.Items.Count ?? 0;
} while (needToGetMoreItems);
```

For the true "moving parts" of the above code, the things that were different were
as follows:

* API route (`api/v3/releases` in the example)
* `fieldsToReturn` : this represented a list of the fields I wanted back from the API
* The response type of the API (`ReleaseApiResponse` in the example)
* The type of the `Items` property of the `...Response`

### Step 1: Define an Interface for the Shared Traits of the Responses

The different API responses all had an `Items` collection of a different type,
so creating an interface to define that was first.

```C#
public interface IHaveItems<T>
{
    List<T> Items { get; set; }
}
```

Then I could add the interface to the different API response types:

```C#
public record ApplicationResponse : IHaveItems<Application>
{
    public List<Application> Items { get; set; } = new();
}
public record CompanyResponse : IHaveItems<Company>
{
    public List<Company> Items { get; set; } = new();
}
public record RelaseResponse : IHaveItems<Release>
{
    public List<Release> Items { get; set; } = new();
}
```

### Step 2: Define Generic Method that Encapsulates the Pagination Logic

The generic method I defined had two generics, the top response object and
the instance object, and the response object was constrained to need an Items
property that was a `List` of the instance object.

```C#
public static class PaginatedApiHelper
{
    public static async Task<List<T>> GetAllDataFromPaginatedApi<T, T1>(this HttpClient client, 
        string route, string fieldsToReturn, string? filter = null) where T1 : IHaveItems<T>
    {
        var fullResults = new List<T>();
        
        bool needToGetMoreItems;
        var offset = 0;
        const int limit = 50;
        do
        {
            var queryString = $"fields={fieldsToReturn}&limit={limit}&offset={offset}{filter}";
            var response = await client.GetFromJsonAsync<T1>($"{route}?{queryString}");
            if (response?.Items.Any() ?? false)
            {
                fullResults.AddRange(response.Items);
            }
            needToGetMoreItems = (response?.Items?.Count ?? 0) == limit;
            offset += response?.Items?.Count ?? 0;
        } while (needToGetMoreItems);

        return fullResults;
    }
}
```

### Step 3: Simplify Original Code

With the interface and method in place, the original code above is simplified to this:

```C#
var fieldsToReturn = "id,name,description,updated"

var releases = client.GetAllDataFromPaginatedApi<Release, ReleaseApiResponse>("api/v3/releases", fieldsToReturn);
```

With the many times that my paginated logic was repeated for different calls, this
made things a lot simpler to read!

Comments? Leave them below. Thanks for reading!
