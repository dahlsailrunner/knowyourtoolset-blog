---
title: "On Exception Handling" # Title of the blog post.
date: 2015-05-29T14:22:50-05:00 # Date of post creation.
description: "Article description." # Description used for search engine.
#thumbnail: "/images/path/thumbnail.png" # Sets thumbnail image appearing inside card on homepage.
#shareImage: "/images/path/share.png" # Designate a separate image for social media sharing.
codeMaxLines: 15 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - C#
  - .NET
  - Development
# comment: false # Disable comment if false.
---

If your application is living in a production environment then good exception handling is one of the most important things you can do if you want to enable good support for your application (logging is another, but I’ll cover that in another post). A knee-jerk reaction to the question of “how do I do good exception handling?” is to put try/catch blocks everywhere, but I don’t recommend that practice as it clutters up your code with a higher percentage of non-business logic lines and does not add anything of real value to supportability.

So where does that leave us? To start, I would like to define a few terms / concepts relative to exception handling that will come into play during the discussion.

* **Swallow:** To “swallow” an exception is to catch it and not propagate it anywhere else.
* **Rethrow:** Pretty much the opposite of swallow — to rethrown an exception is to catch it and then throw the exception you just caught (maybe after adding some info to it).
* **Wrap:** To wrap an exception is to catch it, and throw a NEW exception with the caught exception set as the InnerException property of the new exception.
* **Shield:** Marvel comics group that fights for justice. But as it relates to exceptions (har har), this is to replace a caught exception with an entirely new one, often with less information so as to “shield” some information from the calling program. This often applies at service boundaries, like Web API methods or WCF services.

Major underlying principle for all that follows: Do not lose any of the details from the originally thrown exception before logging them. Those details from the original exception (and others that you might choose to add) are critical to help in troubleshooting the problem without requiring a full debugger session to do so.

I have two primary recommendations for good exception handling:

* **Use global exception handlers when you can**
* **Only catch exceptions outside of the global handlers when either you need to, or you are *adding value***

## Recommendation 1: Use global exception handlers when you can
Global exception handlers take different shapes based on the type of application you’re running:

### ASP.Net Web applications
A global exception handler within an ASP.Net web app is usually just placed in the `global.asax.cs` file and looks something like the listing below (MVC is slightly different but very similar in nature — the key point is to get the exception with the Server.GetLastError().GetBaseException() method, log the exception somehow, and then redirect the user to a graceful page that is not the yellow screen of death:

```csharp
protected void Application_Error(object sender, EventArgs e)
{
    var application = sender as HttpApplication;
    if (application == null) return;            
 
    var ex = Server.GetLastError().GetBaseException();
    if (ex is SecurityException)
    {                
        Response.Redirect("~/Pages/TechnicalAccessError.aspx", false);
        Context.ApplicationInstance.CompleteRequest();
    }
 
    var context = HttpContext.Current;
    if (context.Session != null)
        Session.Add("ErrorMessage", ex.Message); // used in TechnicalError page
     
    if (context.Request.UserAgent != "Microsoft Office Protocol Discovery")
        CustomLogger.WriteLog(ex, CustomLoggingCategory.Web, SessionHelper.GetWebLoggingData());
     
    Server.ClearError();
 
    Response.Redirect("~/Pages/TechnicalError.aspx", false);
    Context.ApplicationInstance.CompleteRequest();
}
```

### Console Application
Console apps are sometimes little workhorses for little jobs that just need to run every so often without a user interface, so I think they bear mentioning here. A global exception handler for a console app is basically a `try / catch` block in the `Main` method. When caught here, the exception should be logged and then the app could exit with some kind of non-zero return code that indicates the nature of the error.

### WPF Desktop Application
A Windows Presentation Foundation application takes a little more code (as shown below), but the concept is similar. Grab the exception, log it, and tell the user in a friendly way that something bad happened.

```csharp
public partial class App
{
    const string TechErrorMsg = "A technical error has occurred. Please try your process again, and contact technical support if the problem persists.";
 
    protected override void OnStartup(StartupEventArgs e)
    {
        try
        {
            Current.DispatcherUnhandledException += GlobalExceptionHandler; // set up a global exception handler for all unhandled exceptions
            AppDomain.CurrentDomain.UnhandledException += AppDomainExceptionHandler;
 
            base.OnStartup(e);
        }            
        catch (Exception ex)
        {
            CustomLogger.WriteLog(ex, CustomLoggingCategory.UserInterface);
            MessageBox.Show("Something failed starting up the application!", "Application Name",
                           MessageBoxButton.OK, MessageBoxImage.Exclamation);
            Environment.Exit(9);
        }
    }
 
    private void AppDomainExceptionHandler(object sender, UnhandledExceptionEventArgs e)
    {
        HandleUnhandledException(e.ExceptionObject as Exception);
    }
 
    static void GlobalExceptionHandler(object sender, System.Windows.Threading.DispatcherUnhandledExceptionEventArgs e)
    {
        e.Handled = true;
        HandleUnhandledException(e.Exception);            
    }
 
    private static void HandleUnhandledException(Exception exception)
    {
        try
        {             
            CustomLogger.WriteLog(exception, CustomLoggingCategory.UserInterface);
            MessageBox.Show(TechErrorMsg, "Error", MessageBoxButton.OK, MessageBoxImage.Error);
        }
        catch (Exception ex)
        {
            CustomLogger.WriteLog(ex, CustomLoggingCategory.UserInterface);
            MessageBox.Show(TechErrorMsg, "Error", MessageBoxButton.OK, MessageBoxImage.Error);
        }
    }
}
```

### Web API apps
Web API apps are a place where you most likely want to do exception shielding — just sending an generic “error” message with an HTTP error code to the caller of your API so that no detailed data like server names, procedure names, etc get included in the error message. But if you want to support the caller who received the error, you probably also will want to have logged the entire exception and be able to associate a specific failed request reported by a caller to the actual error recorded in your logs.

A global exception handler and logger (this was new functionality that could be leveraged in Web API v2) comes to our rescue to accomplish these goals.

Three steps: 
* Define your custom exception logger
* Define your custom exception handler
* Modify the `WebApiConfig.cs` file in your `App_Start` folder to use your custom exception logic

A sample custom exception logger is shown here. Note that it inherits from the framework’s ExceptionLogger class.

```csharp
public class CustomApiExceptionLogger : ExceptionLogger
{        
    public override void Log(ExceptionLoggerContext context)
    {
        var dict = new Dictionary<string, object>
        {
            {"RequestURI", context.Request.RequestUri},
            {"CatchBlockName", context.CatchBlock.Name},
            {"Principal Name", context.RequestContext.Principal.Identity.Name}
        };
 
        var errorId = Guid.NewGuid().ToString();  // This is here because the Logger is called BEFORE the Handler in the Web API exception pipeline   
        context.Exception.Data.Add("ErrorId", errorId); 
 
        CustomLogger.WriteLog(context.Exception, CustomLoggingCategory.WebService, dict);
    }
}
```

Notes:

* The `dict` object is a dictionary that is used to include some additional information to log using the logging framework. The additional information is coming straight from the Web API exception framework — note that the `RequestURI` item contains the entire URL (including query string) for the request.
* The `errorId` is the ID that will be sent in the response (using the handler below) to the caller, and is also logged into this entry for association with the response if the caller contacts our support team for help.

An example of a logged exception using this approach is shown below. Key points to note are the `DATA-` elements on the exception, along with the `ExtendedInfo` properties containing the framework information (including the URL).

![::img-med img-center img-shadow](/images/exception.png)

A sample custom exception handler implementation is shown here, inheriting from the base ExceptionHandler framework class.

```csharp
    {        
        public override void Handle(ExceptionHandlerContext context)
        {
            var errorInfo = string.Empty;
             
            if (context.Exception.Data.Contains("ErrorId")) // this is set within the custom *logger* which is called BEFORE this in the exception pipeline
                errorInfo = "Error ID: " + context.Exception.Data["ErrorId"];
 
            context.Result = new TextPlainErrorResult
            {
                Request = context.ExceptionContext.Request,
                Content = "Oops! Sorry! Something went wrong." +
                          "Please contact NWP support so we can try to fix it. Error ID: " + errorInfo
            };           
        }
 
        private class TextPlainErrorResult : IHttpActionResult
        {
            public HttpRequestMessage Request { get; set; }
 
            public string Content { get; set; }
 
            public Task<HttpResponseMessage> ExecuteAsync(CancellationToken cancellationToken)
            {
                var response = new HttpResponseMessage(HttpStatusCode.InternalServerError)
                {
                    Content = new StringContent(Content),
                    RequestMessage = Request
                };
                return Task.FromResult(response);
            }
        }
    }
```

An exception encountered by the calling user then is returned like this:
*  **Response Body:** Oops! Sorry! Something went wrong.Please contact our support team so we can try to fix it. Error ID: 842a8a6c-a2a4-4f48-8f37-93a375857093
* **Response Code:** 500

The last step to get all of this actually wired in to your Web API is to update the `WebApiConfig.cs` file in `App_Start`:

```csharp
config.Services.Replace(typeof(IExceptionHandler), new CustomApiExceptionHandler());
config.Services.Add(typeof(IExceptionLogger), new CustomApiExceptionLogger());
```
That’s it! Now *all* of the methods within your Web API project will automatically handle and log exceptions with lots of helpful details.

### WCF Service Libraries
I’m not aware of anything specific for global exception handlers within a WCF service library application, so each method should have its own try / catch block utilizing a similar exception shielding technique as described in the Web API section above. Main idea: catch the exception, add an “error id” to it and then log it completely, then turn around and send a generic error/exception (WCF works well with FaultExceptions) to the caller that includes the error id for reference.

## Recommendation 2: Only catch exceptions outside of the global handlers when you can add value, or you when you need to based on the application
***Adding value*** usually means providing additional details that the global exception handler won’t have. These are often things like input parameters to a called method or some kind of derived value that causes the failure. In these cases the exception can be wrapped with a new exception or updated with additional data and then thrown/rethrown.

Note that an exception looks like has the following properties (screenshot taken from the [MSDN documentation for the Exception class](https://msdn.microsoft.com/en-us/library/system.exception(v=vs.110).aspx)):

![::img-med img-center img-shadow](/images/exception-class.png)

I want to specifically focus on the highlighted properties and constructors. To add value by simply adding some parameter information to an immediately caught exception, you can just modify the “Data” collection of the caught exception and rethrown, as shown in the code below. This will simply augment the original exception by adding some additional information to it, which would then later be logged by your global handler.

```csharp
public void MethodWithParameters(string questionableParam)
{
    try
    {
        // ..... do something that might throw an exception based on param
    }
    catch (Exception ex)
    {
        ex.Data.Add("MethodParam", questionableParam);
        throw;
    }
}
```
You might favor wrapping a caught exception with a more fully descriptive exception of your own. Again — I would not recommend replacing the exception unless you are shielding — and then only AFTER you’ve logged the unmodified original exception and any additional data you add.

To do this, you create and throw a new exception, using the constructor with (string, Exception) as inputs — you use the originally caught exception as the Exception parameter and any new descriptive text as the string parameter for the new exception. See the example below.
```csharp
public void MethodWithParameters(string questionableParam)
{
    try
    {
        // ..... do something that might throw an exception based on param
    }
    catch (Exception ex)
    {
        var newEx = new Exception(string.Format("Something really bad happened with param: {0}", questionableParam), ex);
        throw newEx;
    }
}
```

As for other situations when you really have no choice but to catch thrown exceptions — that enters the realm of “it depends on your application” more heavily. One example where you probably most commonly want to catch thrown exceptions outside of a global handler is within code that sits behind an AJAX call on a website (not Web API calls which are covered above — this probably most typically applies to ASP.NET Web Forms apps).

In a case like this — where you have a postback call in WebForms that is being invoked via Ajax — it would be a completely silent (but logged) failure to the user, you need a way to actually tell the user that something went wrong. To do that, you have to catch exceptions in your AJAX callback like such:

```csharp
catch (Exception ex)
{
      CustomLogger.WriteLog(ex, CustomLoggingCategory.PortalWeb, Helper.GetWebLoggingData());
      pnlOneTimeAlert.Visible = true;
      MessageCssClass = "alert-danger";
      lblDisplayAddUserMsg.Text = "Something went wrong with this request.";
} 
```

The assumption in the code above is that the last three lines actually are items on the ASPX page that will show themselves to the user.

## Wrapping it up
Well, that’s it. Maybe more than you ever wanted to know about exceptions. The main ideas are basically that it you put some framework in place up front in your applications to globally handle exceptions, you can minimize your code touchpoints after that without sacrificing any good information being captured and hooked on to the exceptions as they are thrown. There ***are*** still times when you need to catch exceptions manually, but they are very specific and easily identifiable once you know what you’re looking for.

Enjoy!