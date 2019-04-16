# ASP.NET Core MVC Global Error Handling for HTML and AJAX Endpoints

[ASP.NET Core MVC Global Error Handling with HTML or JSON Response](https://aregsar.com/blog/2019/dotnetcore-aspnet-mvc-global-error-handling)

## Introduction

It is critical to have global exception handling for ASP.NET Core applications to respond appropriatly to exceptions that are not handled by application logic.

Global exception handling allows us to log all unhandled exceptions in a central location in our application and then provide a user friendly response to the user.

In ASP.NET MVC projects, there are generally two types of content returned to the web browser. There are normal web page requests that require returning HTML content and there are AJAX requests that normally require returning JSON formatted content.

When the browser requests a HTML rendered page and an exception occures that the application can not handle, we generally redirect the browser to an HTML error page.

However, when the browser makes an AJAX request that expects a JSON response then we need to return a JSON error response instead of redirecting to an HTML error page.

Given this, in a global exception handler, we need to distinguish between normal web page requests and ajax requests that expect JSON, so that we can return the appropriate error response.

## Using HTTP request headers in the Global Exception Handler

The way we can detect if a request expects a JSON response is by inspecting the HTTP __Accept__ header sent by AJAX requests.

The global exception handler in our MVC application can determine whether to send a JSON error response or redirect to a HTML error page based on the value of the Accept header.

If the Accept header value contains the text `application/json` then the handler needs to respond with a JSON error response.

> Note: Detecting if a request is a AJAX request is not the same as detecting whether the request accepts a JSON response. To detect an AJAX request you can check for the __X-Requested-With__ request header for the `'xmlhttprequest'` value.

## Adding a Global exception handler middleware

We can add a Global exception handler middleware that can access the unhandled exception and the request headers to decide how to format the exception data for the response.

This middleware will redirect to a HTML error page for full page requests or return JSON data for requests, that contain the `application\json` Accept header.

Below you can see my sample implementation of the global exception handler middleware implemented as an `IApplicationBuilder` extensio method:

```csharp
public static class GlobalExceptionHandlerExtension
{
    //This method will globally handle logging unhandled execptions.
    //It will respond json response for ajax calls that send the json accept header
    //otherwise it will redirect to an error page
    public static void UseGlobalExceptionHandler(this IApplicationBuilder app
                                                , ILogger logger
                                                , string errorPagePath
                                                , bool respondWithJsonErrorDetails=false)
    {
        app.UseExceptionHandler(appBuilder =>
        {
            appBuilder.Run(async context =>
            {
                //============================================================
                //Log Exception
                //============================================================
                var exception = context.Features.Get<IExceptionHandlerFeature>().Error;

                string errorDetails = $@"{exception.Message}
                                            {Environment.NewLine}
                                            {exception.StackTrace}";

                int statusCode = (int)HttpStatusCode.InternalServerError;

                context.Response.StatusCode = statusCode;

                var problemDetails = new ProblemDetails
                {
                    Title = "Unexpected Error",
                    Status = statusCode,
                    Detail = errorDetails,
                    Instance = Guid.NewGuid().ToString()
                };

                var json = JsonConvert.SerializeObject(problemDetails);

                logger.LogError(json);

                //============================================================
                //Return response
                //============================================================
                var matchText="JSON";

                bool requiresJsonResponse = context.Request
                                                    .GetTypedHeaders()
                                                    .Accept
                                                    .Any(t => t.Suffix.Value?.ToUpper() == matchText
                                                            || t.SubTypeWithoutSuffix.Value?.ToUpper() == matchText);

                if (requiresJsonResponse)
                {
                    context.Response.ContentType = "application/json; charset=utf-8";

                    if(!respondWithJsonErrorDetails)
                        json = JsonConvert.SerializeObject(new {Title = "Unexpected Error", Status = statusCode});

                    await context.Response
                                    .WriteAsync(json, Encoding.UTF8);
                }
                else
                {
                    context.Response.Redirect(errorPagePath);

                    await Task.CompletedTask;
                }
            });
        });
    }
}
```

ASP.NET Core 2.2 has infrastucture code that makes it easy to parse out the Accept header components and an error data container that can be serialized to json and returned as a response.

The handler first logs the error using the supplied logger and then returns a response based on the content of the Accept header. An additional flag is used to limit the json data returned in the response.

We can now replace the original `app.UseExceptionHandler("/Home/Error")` call in the `Startup.Configure(...)` method with our own exception handler middleware:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseGlobalExceptionHandler( _logger
                                    , errorPagePath: "/Home/Error"
                                    , respondWithJsonErrorDetails: true);

        //Replaced UseExceptionHandler with UseGlobalExceptionHandler
        //app.UseExceptionHandler("/Home/Error");

        app.UseHsts();
    }

    app.UseHttpsRedirection();
    app.UseStaticFiles();
    app.UseCookiePolicy();

    app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");
    });
}
```

The complete source code of the demo app can be found in my Github [https://github.com/aregsar/mvcapp](https://github.com/aregsar/mvcapp).

## Testing the Global Exception Handler middleware

I added an `Ajax` action to the `HomeController` and modified the `Privacy` action in the same file, to do a quick test with the global exception handler  middleware installed.

```csharp
public IActionResult Privacy(int? id)
{
    if(id.HasValue)
        throw new Exception("privacy page exception");

    return View();
}

public IActionResult Ajax(int? id)
{
    if(id.HasValue)
        throw new Exception("ajax exception");

    return Json(new {name="ajax"});
}
```

Looking in the `Startup.Configure(...)` method we can see that the global exception handler middleware is installed in the middleware pipeline only for production builds.

Therefore, to run in production mode, I added a Production run configuration profile in the profiles section of the properties\launchSettings.json file. Snippet below:

```csharp
 "prod": {
      "commandName": "Project",
      "launchBrowser": true,
      "applicationUrl": "https://localhost:5001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Production"
      }
```

To run the app using the prod configuration we can use the dotnet run command with the --launch-profile flag:

```bash
dotnet run --launch-profile prod
```

We can now quickly test the global exception handling middleware code by issuing `curl` commands against the `Ajax` action endpoints.

To do so we can open a new terminal tab and curl to the `https://localhost:5001/home/ajax` URL:

```bash
curl -i -H "Accept: application/json" https://localhost:5001/home/ajax
HTTP/1.1 200 OK
Date: Thu, 11 Apr 2019 22:47:56 GMT
Content-Type: application/json; charset=utf-8
Server: Kestrel
Transfer-Encoding: chunked

{"name":"ajax"}
```

We can see a the normal JSON response of the Ajax action method since there was no unhandled exeption.

Next we can add an id parameter to the URL to activate the exception in the Ajax action method.

```bash
curl -i -H "Accept: application/json" https://localhost:5001/home/ajax/1
HTTP/1.1 500 Internal Server Error
Date: Thu, 11 Apr 2019 22:46:27 GMT
Content-Type: application/json
Server: Kestrel
Cache-Control: no-cache
Pragma: no-cache
Transfer-Encoding: chunked
Expires: -1

{
  "title": "Unexpected error",
  "status": 500,
  "detail": "test exception\r\n                                             \n\r\n                                                at mvcapp.Controllers.HomeController.ajax(Nullable`1 id) in /Users/aregsarkissian/projects/asp3/mvcapp/Controllers/HomeController.cs:line 29\n   at lambda_method(Closure , Object , Object[] )\n   at Microsoft.Extensions.Internal.ObjectMethodExecutor.Execute(Object target, Object[] parameters)\n   at Microsoft.AspNetCore.Mvc.Internal.ActionMethodExecutor.SyncActionResultExecutor.Execute(IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)\n   at Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker.InvokeActionMethodAsync()\n   at Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker.InvokeNextActionFilterAsync()\n   at Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker.Rethrow(ActionExecutedContext context)\n   at Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)\n   at Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker.InvokeInnerFilterAsync()\n   at Microsoft.AspNetCore.Mvc.Internal.ResourceInvoker.InvokeNextResourceFilter()\n   at Microsoft.AspNetCore.Mvc.Internal.ResourceInvoker.Rethrow(ResourceExecutedContext context)\n   at Microsoft.AspNetCore.Mvc.Internal.ResourceInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)\n   at Microsoft.AspNetCore.Mvc.Internal.ResourceInvoker.InvokeFilterPipelineAsync()\n   at Microsoft.AspNetCore.Mvc.Internal.ResourceInvoker.InvokeAsync()\n   at Microsoft.AspNetCore.Routing.EndpointMiddleware.Invoke(HttpContext httpContext)\n   at Microsoft.AspNetCore.Routing.EndpointRoutingMiddleware.Invoke(HttpContext httpContext)\n   at Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware.Invoke(HttpContext context)\n   at Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware.Invoke(HttpContext context)",
  "instance": "9f238f4e-97b4-478d-9ee3-96e91cb1a93c"
}
```

This time can see the response now includes the JSON formatted exception data returned by the global exception handler.

Also if we look at the terminal tab where we ran the application, we can see that the unhandled exception is logged as JSON data to the console by the global exception handler.

Next we can use the web browser to test the `Privacy` action methods with the global exception handler.

We can use the web browser to navigate to the Privacy action method endpoint URL:

[https://localhost:5001/home/privacy](https://localhost:5001/home/privacy)

We can see the normal privacy HTML page response displayed in the browser.

Next we add an the id parameter to the URL to activate the exception in the Privacy action method and use the web browser to navigate to the URL:

[https://localhost:5001/home/privacy/1](https://localhost:5001/home/privacy/1)

This time we can see that we are redirected to a server error page as usual for HTML endpoints that throw unhandled exeptions.

## Conclusion

It is easy to add a globalexception handling middleware to ASP.NET Core MVC applications to perform custom exception handling.

ASP.NET Core gives us all the fascilities that we need to access information in the request headers and the exception data to make our own descision on how we want to respond to unhandled application errors.

Thanks for reading