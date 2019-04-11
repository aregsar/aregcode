# dotnetcore aspnet mvc global error handling

[dotnetcore aspnet mvc global error handling](https://aregsar.com/blog/2019/dotnetcore-aspnet-mvc-global-error-handling)

# Introduction

It is critical to have global exception handling for ASP.NET Core applications when
Exceptions are not handled by application logic.

For production releases, this will allow us to log all unhandled exceptions and also provide a user friendly response to the user.

ASP.NET Core Web API projects by default return json data for API endpoint request.
When an unhandled exception occures the exception error is also returned as json data. Clients of Web APIs expect json responses and can handle json error responses in a normal fashion.

In ASP.NET MVC projects, where we are serving web pages, global exception handling becomes a little more complicated.

Generally we serve HTML pages to the client web browser, so if an unhandled exception occures we can just redirect the browser to an HTML error page.

But many MVC web applications also make ajax request to the same application. Usually the format of the response requested by these ajax calls is json. So when an unhandled exception occures on the server, we need to return a json response instead of redirecting to an html error page.

Given this, we need to figure out a way in our MVC projects to distinguish between normal web page requests and ajax requests.

The way we can do that is with the HTTP Accept header sent by ajax requests.
Our MVC application can determine whether to send a json error response or redirect to an error page based on the value of the Accept header.

If the `Accept` header value contains the text `application/json` then the application needs to respond with a json error response.

## Adding a Global exception handler middleware

We can add a Global exception handler middleware that can switch the exception handling logic based on the value of the `Accept` header.

This middleware will redirect to an error page for full page requests that trigger and unhandled exception and return json data for ajax requests, that send the `application\json` accept header, that trigger an unhandled exception.

Below you can see a sample implementation of the global exception handler middleware:

```csharp
private void UseGlobalExceptionHandler(IApplicationBuilder app)
{
    app.UseExceptionHandler(appBuilder =>
    {
        appBuilder.Run(async context =>
        {
            var exception = context.Features.Get<IExceptionHandlerFeature>().Error;

            string errorDetails = $@"{exception.Message}
                                    {Environment.NewLine}
                                    {exception.StackTrace}";

            int statusCode = (int)HttpStatusCode.InternalServerError;

            context.Response.StatusCode = statusCode;

            //here is where we check the accept header for application json
            bool requiresJsonResponse = context.Request
                                                .GetTypedHeaders()
                                                .Accept
                                                .Any(t => t.Suffix.Value?.ToUpper() == "JSON"
                                                       || t.SubTypeWithoutSuffix.Value?.ToUpper() == "JSON");

            context.Response.ContentType = "application/json";

            var problemDetails = new ProblemDetails
            {
                Title = "Unexpected error",
                Status = statusCode,
                Detail = errorDetails,
                Instance = Guid.NewGuid().ToString()
            };

            var json = JsonConvert.SerializeObject(problemDetails);

            _logger.LogError(json);

            if (requiresJsonResponse)
            {
                await context.Response
                                .WriteAsync(json, Encoding.UTF8);
            }
            else
            {
                context.Response.Redirect("/Home/Error");

                await Task.CompletedTask;
            }
        });
    });
}
```

ASP.NET Core 2.2 has infrastucture code that makes it easy to parse out the Accept header components and a convinient error data container that can be serialized to json and returned as a response.

> Note: the sample implementation returns the error details in the response. For production builds you will want to omit the details and return a generic error message.

## Analysing the Global handler code

var exception = context.Features.Get<IExceptionHandlerFeature>().Error;
 
var problemDetails = new ProblemDetails
                        {
                            Title = "Unexpected error",
                            Status = statusCode,
                            Detail = errorDetails,
                            Instance = Guid.NewGuid().ToString()
                        };

## Testing the global exceptionhandler middleware

-To test the middleware we can create a new MVC application 

`dotnet new mvc`

-add the required using statements for the middleware code to the startup.cs file

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.HttpsPolicy;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using System.Net;
using Newtonsoft.Json;
using Microsoft.AspNetCore.Diagnostics;
using System.Text;
```

-add the middleware convinience method to startup.cs

-call the middleware convinience method inside the startup.configure method
by replacing the app.UseExceptionHandler("/Home/Error") call

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        //replaced app.UseExceptionHandler("/Home/Error") call with:
        UseGlobalExceptionHandler(app);

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

-Add a new action in the home controller

```csharp
public IActionResult ajax(int? id)
{
    if(id.HasValue)
        throw new Exception("test exception");

    var data = new{data="ajax"};

    return Json(data);
}
```

-Change the environment to production and run the application.

We can quickly test the global exception handling middleware code by running the application and issuing curl commands.

First we navigate to the https://localhost:5001/home/ajax URL to activate the ajax action normally, which returns the hard coded json data.

```bash
Aregs-MacBook-Pro:mvcapp aregsarkissian$ curl -i -H "Accept: application/json" https://localhost:5001/home/ajax
HTTP/1.1 200 OK
Date: Thu, 11 Apr 2019 22:47:56 GMT
Content-Type: application/json; charset=utf-8
Server: Kestrel
Transfer-Encoding: chunked

{"name":"areg"}
```

Next we can add an id parameter to the URL to activate the exception and we get the exception information as json data.

```bash
Aregs-MacBook-Pro:mvcapp aregsarkissian$ curl -i -H "Accept: application/json" https://localhost:5001/home/ajax/1
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