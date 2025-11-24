---
title: "Hello, AI World and Streaming Content Between ASP.NET APIs and Angular" 
date: 2025-01-11T15:29:04-05:00 
summary: "Getting a basic chat response from Azure OpenAI, providing it
as a streamed response from an ASP.NET API, and then receiving and displaying
it while streaming in an Angular app." 
featured: true 
toc: true
thumbnail: "/images/hello-ai-streaming.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/hello-ai-streaming.png" # Designate a separate image for social media sharing.
codeMaxLines: 15 
codeLineNumbers: false 
figurePositionShow: true 
tags:
  - .NET
  - ASP.NET
  - C#
  - Aspire
  - BFF
  - Duende Software
  - Azure OpenAI
  - OpenAI
  - Angular
  - Fetch API
---

***tl;dr - just show me the code (and the readme):*** <https://github.com/dahlsailrunner/hello-ai-world>

AI technologies and tooling are evolving more rapidly than most technology in recent past and are not slowing down.  Getting started in this space can be intimidating.  In this post I intend to make this as easy as possible, as well as show how to provide streamed content from an ASP.NET API and then consume it (as streamed content) in an Angular application.

{{% notice note "It's Not About OpenAI" %}}
The streaming handling (both in ASP.NET and Angular) in this post
don't require any use of OpenAI -- just some content that you would
get incrementally.  You could modify the ASP.NET `stream-chat` endpoint
with whatever logic you want.
{{% /notice %}}

## The End Result

The final result of this code is an Angular app has a page
that makes an API call to an ASP.NET API, which asks a hard-coded
question to Azure OpenAI, provides a streaming response, and that
response is rendered as it comes in nice HTML based on the markdown
that it came as.

Shown here:
![:: img-shadow](/images/hello-ai-streaming.png)
Might not be SUPER exciting or useful as a final application,
but provides a simple example using some powerful techniques that
***open the door to lots of great possibilities***.

## Setting UP an Azure OpenAI Instance

The code uses an Azure OpenAI instance which needs to exist.  For small little
experimentation purposes like I'm describing here, there is little to no
cost, but you do need to have your own instance.

You'll need to have an **Azure OpenAI instance** and a **model deployed as
a "Chat Deployment"**.  The [simple instructions](https://github.com/dahlsailrunner/hello-ai-world?tab=readme-ov-file#1-get-an-azure-openai-instance) to do that are in the
readme tied to this post.

When you're done, you should be able to identify three things:

* **Azure OpenAI Endpoint**: The URL for your Azure OpenAI instance, something
like <https://YOUR-NAME.openai.azure.com>
* **API Key**: This is a key for authenticating to your Azure OpenAI instance
* **Deployment Model**: This will be a short string like `YOUR-VALUE-gpt-4o-mini` (as an example)

## Using OpenAI in an ASP.NET API

An easy way to use OpenAI capabilities (including both Azure OpenAI and
OpenAI) in an ASP.NET application is to reference the
[Aspire.Azure.AI.OpenAI](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/ai.openai-readme?view=azure-dotnet).

{{% notice info "I'm Using Aspire" %}}
In my example I'm using the Aspire version of the Nuget package mentioned
above that is currently in pre-release. It adds some simple Open Telementry
items that might not otherwise be there.  But the base package (Azure.AI.OpenAI) referenced
in the link above should work just fine, too.
{{% /notice %}}

Once you've got the package referenced, you need a single line in `Program.cs`:

```csharp
builder.AddAzureOpenAIClient("azureOpenAi");
```

That registers an `OpenAIClient` with the dependency injection provider
that can be injected where you need it.

The string `"azureOpenAi"` is a connection string from your configuration.
I recommend using
[user secrets](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-9.0&tabs=windows),
which in Visual Studio is as simple as right-clicking on the API project
and choosing "Manage User Secrets".

The JSON for the connection string looks like this and should use
values from the Azure OpenAI instance mentioned above:

```json
{
  "ConnectionStrings": {
    "azureOpenAi": "Endpoint=https://YOUR-VALUE.openai.azure.com/;Key=YOUR-KEY"
  }
}
```

Then in a constructor for an API controller (or wherever), you can simply
inject an `OpenAIClient` and you should be able to use it!

## Providing a Streamed Response in ASP.NET

Consider the following method as the simplest possible example of using
the `OpenAIClient`:

``` csharp
public async Task<ActionResult<string>> GetHello(CancellationToken cxl)
{
    var chatClient = aiClient.GetChatClient(_chatDeploymentModel); // set to your deployment model from above

    ChatCompletion completion = await chatClient.CompleteChatAsync(
    [
        new UserChatMessage("What are the main steps to set up an Azure OpenAI instance?")
    ], cancellationToken: cxl);
    return completion.Content[0].Text;
}
```

The method will work just fine - but it waits until the complete response
is obtained before returning anything at all, and returns the entire
response at once.

A better experience is to return the response in chunks so that the
user can start reading as soon as the earliest information is available.

A new method that does that is here:

```csharp
public async IAsyncEnumerable<string> StreamChat(
        [EnumeratorCancellation] CancellationToken cxl)
{
    var chatClient = aiClient.GetChatClient(_chatDeploymentModel);

    var response = chatClient.CompleteChatStreamingAsync(
    [new UserChatMessage("What are the main steps to set up an Azure OpenAI instance?")],
            cancellationToken: cxl);

    await foreach (var message in response)
    {
        foreach (var contentPart in message.ContentUpdate)
        {
            yield return contentPart.Text;
        }
    }
}
```

Note first the return type of the method: `IAsyncEnumerable<string>`. Also
note that cancellation is handled with different types here, too.

We're calling the `CompleteChatStreamingAsync` method instead of
`CompleteChatAsync` in this example (and not awaiting the method call),
and then there's an awaited `foreach` loop and the `yield return` of the
text from each response part.

If you keep in mind the different return type, cancellation handling, and
the yield return, then you should be able to provide streamed API
responses to callers.

{{% notice note "Swagger UI Works Fine" %}}
The streamed response works fine in Swagger, but it also is only
displayed once all content has been received.  The display will look
something like this:

```json
[
  "",
  "Setting",
  " up",
  " an",
  " Azure",
  " Open",
  "AI",
  ... MORE FOLLOWS IN SINGLE ARRAY
```

{{% /notice %}}

## Receiving Streamed Responses in Angular

Gettting a streamed response in Angular to display as it comes in
can be done pretty easily, but requires the use of the [JavaScript Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
which isn't enabled by default on the Angular HTTP client.

### Enabling the Fetch API for HttpClient

In the `app.config.ts` of your Angular app, you can
[enable the Fetch API](https://angular.dev/guide/http/setup#withfetch)
with the `withFetch()` call.  All of your existing calls should still
work, but now you have the option of getting "progress reports" from
streaming APIs.

So my `app.config.ts` contains this code (note `provideHttpClient`):

```ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }), 
    provideRouter(routes), 
    provideAnimationsAsync(),
    provideHttpClient(
      withInterceptors([csrfInterceptor]),
      withFetch() // <-- this is the important line
    ),
  ]
};
```

### The Angular Service

The service code in Angular is as follows for my example:

```ts
export class ChatService {

  constructor(private http: HttpClient) { }
  private apiUrl = '/api/v1/openai/stream-chat';

  streamChat() {
    return this.http.get(this.apiUrl, {
      observe: 'events',
      responseType: 'text',
      reportProgress: true,
    });
  }
}
```

I'm using the [Duende Backend For Frontend (BFF) security pattern](https://docs.duendesoftware.com/identityserver/v7/bff/)
here to avoid making a remote API call with an exposed access token,
but the concept above should be pretty clear.

An HTTP call will be made to the endpoint that provides a streaming
response, and with the Fetch API I want to make sure we can observe
the progress events from this call.

### The Angular Component - with a Signal

The Angular component code is as follows (notes below the code):

```ts
export class ChatComponent implements OnInit {
  streamedMessages = signal('');

  constructor(
    private chatService: ChatService,
    private sanitizer: DomSanitizer,
  ) {}

  ngOnInit(): void {

    this.chatService.streamChat().subscribe((event) => {      
      if (event.type === HttpEventType.DownloadProgress) {
        // getting streamed content
        const partial = (event as HttpDownloadProgressEvent).partialText!;
        const partialAsString = this.convertToString(partial!);
        // The response in this example is a markdown string, 
        // so we need to convert it to HTML
        const updatedMessage = this.sanitizer.bypassSecurityTrustHtml(
          marked(partialAsString) as string
        );
        this.streamedMessages.set(updatedMessage as string);
      }
      // else if (event.type === HttpEventType.Response) { 
      //   Do stuff when the streaming is done
      //   console.log('Streaming done');
      //}
    });
  }

  convertToString(responseContent: string): string {
    // the response might or might not be a valid JSON array, 
    // but it always starts with [, so we need to conditionallyadd the ]
    if (responseContent.slice(-1) !== ']') {
      responseContent += ']';
    }
    return JSON.parse(responseContent).join('');
  }
}
```

Two things are injected: the `ChatService` that we just described above,
and a `DomSanitizer` - so that we can make the markdown-formatted content
returned by the streaming API look nice when displayed.

The `ngOnInit()` method has the logic that subscribes to the
events from the `streamChat()` method -- when it gets a `DownloadProgress`
event it adds the partial content received to what is displayed.

The partial content is received as a JSON array, but it might not
be a closed-array (with the final `]`), so that is what the `convertToString()`
method handles.

{{% notice info "Example partial content" %}}

```json
[ "hi", ", there", ". This",
```

{{% /notice %}}

The final string is parsed into HTML by the [marked](https://marked.js.org/) library, and
the `signal` (the `streamedMessages` variable) is updated.

Also note that if you wanted to perform some logic when the final
content is received, there is a commented-out conditional block
that shows where you could do that.

### The Component HTML

The HTML is really simple - and just references the `streamedMessages`
signal:

```html
<h2 class="heading-lg">Chat responses below</h2>
<h2 class="heading-md">What are the main steps to set up an Azure OpenAI instance?</h2>

<div [innerHtml]="streamedMessages()"></div>
```

## Where to Go Next

This repo and post are meant as "stepping stones" into the
world of AI and its capabilities.

A **great** place to go next is to start exploring the use
of "tools" in chat.

Or do the same thing as above with [SemanticKernel](https://learn.microsoft.com/en-us/semantic-kernel/overview/).

Happy coding!
