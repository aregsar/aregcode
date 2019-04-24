# Understanding ASP.NET Core Endpoint Routing

April 19, 2019 by [Areg Sarkissian](https://aregcode.com/about)

last update April 24, 2019

## Introduction

In this post I will explain the new Endpoint Routing feature that has been added to the ASP.NET Core middleware pipeline starting with version 2.2 and how it is evolving through to the upcoming version 3.0 that is at preview 3 at present time.

## The motivation behind endpoint routing

Prior to endpoint routing the routing resolution for an ASP.NET Core application was done in the ASP.NET Core MVC middleware at the end of the HTTP request processing pipeline. This meant that route information, such as what controller action would be executed, was not available to middleware that processed the request before the MVC middleware in the middleware pipeline.

It is particularly useful to have this route information available for example in a CORS or authorization middleware to use the information as a factor in the authorization process.

Endpoint routing also allows us to decouple the route matching logic from the MVC middleware, moving it into its own middleware. It allows the MVC middleware to focus on its responsibility of dispatching the request to the particular controller action method that is resolved by the endpoint routing middleware.

## The new Endpoint Routing middleware

Due to the above mentioned reasons, the endpoint routing feature was born to allow the route resolution to happen earlier in the pipeline in a separate endpoint routing middleware. This new middleware can be placed at any point in the pipeline after which other middleware in the pipeline can access the resolved route data.

The endpoint routing middleware API is evolving with the upcoming 3.0 version of the .NET Core framework. For that reason, the API that I will describe in the following sections may not be the final version of the feature. However the over all concept and understanding of how route resolution and dispatch work using endpoint routing should still apply.

In the following sections I will walk you through the current iterations of endpoint routing implementation from version 2.2 to version 3.0 preview 3, then I will note some changes that are coming based on the current ASP.NET Core source code.

## The three core concepts involved in Endpoint Routing

There are three overall concepts that you need to understand to be able to understand how endpoint routing works.

These are the following:

+ Endpoint route resolution
+ Endpoint dispatch
+ Endpoint route mapping

### Endpoint route resolution

The endpoint route resolution is the concept of looking at the incoming request and mapping the request to an endpoint using route mappings. An endpoint represents the controller action that the incoming request resolves to, along with other metadata attached to the route that matches the request.

The job of the route resolution middleware is to construct and __Endpoint__ object using the route information from the route that it resolves based on the route mappings. The middleware then places that object into the http context where other middleware the come after the endpoint routing middleware in the pipeline can access the endpoint object and use the route information within.

Prior to endpoint routing, route resolution was done in the MVC middleware at the end of the middleware pipeline. The current 2.2 version of the framework adds a new endpoint route resolution middleware that can be placed at any point in the pipeline, but keeps the endpoint dispatch in the MVC middleware. This will change in the 3.0 version where the endpoint dispatch will happen in a separate endpoint dispatch middleware that will replace the MVC middleware.

### Endpoint dispatch

Endpoint dispatch is the process of invoking the controller action method that corresponds to the endpoint that was resolved by the endpoint routing middleware.

The endpoint dispatch middleware is the last middleware in the pipeline that grabs the endpoint object from the http context and dispatches to particular controller action that the resolved endpoint specifies.

Currently in version 2.2 dispatching to the action method is done in the MVC middleware at the end of the pipeline.

In the version 3.0 preview 3, the MVC middleware is removed. Instead the endpoint dispatch happens at the end of the middleware pipeline by default. Because the MVC middleware is removed, the route map configuration that is usually passed to the MVC middleware, is instead passed to the endpoint route resolution middleware.

Based on the current source code, the upcoming 3.0 final release should have a new endpoint routing middleware that is placed at the end of the pipeline to make the endpoint dispatch explicit again. The route map configuration will be passed to this new middleware instead of the endpoint route resolution middleware as it is in version 3 preview 3.

### Endpoint route mapping

When we define route middleware we can optionally pass in a lambda function that contains route mappings that override the default route mapping that ASP.NET Core MVC middleware extension method specifies.

Route mappings are used by the route resolution process to match the incoming request parameters to a route specified in the rout map.

With the new endpoint routing feature the ASP.NET Core team had to decide which middleware, the endpoint resolution or the endpoint dispatch middleware, should get the route mapping configuration lambda as a parameter.

In fact this is the part of the endpoint routing where the API is in flux. As I write this post, the route mapping is being moved from the route resolution middleware to the endpoint dispatcher middleware.

I will show you the route mapping API in version 3 preview 3 first, then show you the latest route mapping API in the ASP.NET Core source code. In the source code version, we will see that the route mapping is moved to the endpoint dispatcher middleware extension method.

Its important to note that the endpoint resolution happens during runtime request handling after the route mapping is setup during application startup configuration. Therefor the route resolution middleware has access to the route mappings during request handling regardless of which middleware the route map configuration is passed to.

## Accessing the resolved endpoint

Any Middleware after the endpoint route resolution middleware will be able to access the resolved endpoint through the HttpContext.

The following code snippet shows how this can be done in your own middleware:

```csharp
//our custom middleware
app.Use((context, next) =>
{
    var endpointFeature = context.Features[typeof(Microsoft.AspNetCore.Http.Features.IEndpointFeature)]
                                           as Microsoft.AspNetCore.Http.Features.IEndpointFeature;

    Microsoft.AspNetCore.Http.Endpoint endpoint = endpointFeature?.Endpoint;

    //Note: endpoint will be null, if there was no
    //route match found for the request by the endpoint route resolver middleware
    if (endpoint != null)
    {
        var routePattern = (endpoint as Microsoft.AspNetCore.Routing.RouteEndpoint)?.RoutePattern
                                                                                   ?.RawText;

        Console.WriteLine("Name: " + endpoint.DisplayName);
        Console.WriteLine($"Route Pattern: {routePattern}");
        Console.WriteLine("Metadata Types: " + string.Join(", ", endpoint.Metadata));
    }
    return next();
});
```

As you can see I am accessing the resolved endpoint object through the IEndpointFeature or the Http Context.
The framework provides wrapper methods to access the endpoint object without having to reach directly into the context as I have shown here.

## Endpoint routing configuration

The middleware pipeline endpoint route resolver middleware, endpoint dispatcher middleware and endpoint route mapping lambda is setup in the `Startup.Configure` method of the `Startup.cs` file of ASP.NET Core project.

This configuration changed between 2.2 and 3.0 preview 3 versions and is still changing before the 3.0 release version. So in order to demonstrate the endpoint routing configuration I am going to declare the general form of the endpoint routing middleware configuration as pseudo code based on the three core concepts listed above:

```csharp
//psuedocode that passes route map to endpoint resolver middleware
public void Configure(IApplicationBuilder app
                     , IHostingEnvironment env)
{
    //middleware configured before the UseEndpointRouteResolverMiddleware middleware
    //that does not have access to the endpoint object
    app.UseBeforeEndpointResolutionMiddleware();

    //middleware that inspects the incoming request, resolves a match to the route map
    //and stores the resolved endpoint object into the httpcontext
    app.UseEndpointRouteResolverMiddleware(routes =>
    {
        //This is the route mapping configuration passed to the endpoint resolver middleware
        routes.MapControllers();
    })

    //middleware after configured after the UseEndpointRouteResolverMiddleware middleware
    //that can access to the endpoint object
    app.UseAfterEndpointResolutionMiddleware();

    //The middleware at the end of the pipeline that dispatches the controler action method
    //will replace the current MVC middleware
    app.UseEndpointDispatcherMiddleware();
}
```

This version of our pseudo code shows the route mappings lambda passed as a parameter to the `UseEndpointRouteResolverMiddleware` endpoint route resolution middleware extension method.

An alternate version that matches what the current source code looks like, is shown below:

```csharp
//psuedocode version 2 that passes route map to endpoint dispatch middleware
public void Configure(IApplicationBuilder app
                     , IHostingEnvironment env)
{
    app.UseBeforeEndpointResolutionMiddleware()

    //This is the endpoint route resolver middleware
    app.UseEndpointRouteResolverMiddleware();

    //This middleware can access the resolved endpoint object via HttpContext
    app.UseAfterEndpointResolutionMiddleware();

    //This is the endpoint dispatch middleware
    app.UseEndpointDispatcherMiddleware(routes =>
    {
        //This is the route mapping configuration passed to the endpoint dispatch middleware
        routes.MapControllers();
    });
}
```

In this version the route mapping configuration is passed as a parameter to the endpoint dispatch middleware extension method `UseEndpointDispatcherMiddleware`.

Either way the `UseEndpointRouteResolverMiddleware()` endpoint resolver middleware will have access to the route mappings at request handling time to do the route matching.

Once the route is matched by `UseEndpointRouteResolverMiddleware` an Endpoint object will be constructed with the route parameters and set to the httpcontext so that the middleware in the pipeline that follow, can access the Endpoint object and use it if needed.

In version 3 preview 3 version of this pseudo code, the route mapping is passed to `UseEndpointRouteResolverMiddleware` and there does not exist a `UseEndpointDispatcherMiddleware` at the end of the pipeline. This is because in this version the ASP.NET framework itself implicitly dispatches the resolved endpoint at the end of the request pipeline.

So the pseudo code representing version 3 preview 3 has the form below:

```csharp
//pseudo code representing v3 preview 3 endpoint routing API 
public void Configure(IApplicationBuilder app
                     , IHostingEnvironment env)
{
    app.UseBeforeEndpointResolutionMiddleware()

    //This is the endpoint route resolver middleware
    app.UseEndpointRouteResolverMiddleware(routes =>
    {
        //The route mapping configuration is passed to the endpoint resolution middleware
        routes.MapControllers();
    })

    //This middleware can access the resolved endpoint object via HttpContext
    app.UseAfterEndpointResolutionMiddleware()

    // The resolved endpoint is implicitly dispatched here at the end of the pipeline
    // and so there is no explicit call to a UseEndpointDispatcherMiddleware
}
```

This API looks to be changing with the 3.0 release version, as the current source code shows that the `UseEndpointDispatcherMiddleware` is added back in and it is the middleware that takes the route mappings as a parameter as illustrated by the second version of the pseudo code above.

### Endpoint routing in version 2.2

If you create a Web API project using version 2.2 of the .NET Core SDK you will see the following code in the `Startup.Configure` method:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    app.UseHttpsRedirection();

    //By default endpoint routing is not added
    //the MVC middleware dispatches the controller action
    //and the MVC middleware configures the default route mapping
    app.UseMvc();
}
```

The MVC middleware is configured at the end of the middleware pipeline using the `UseMvc()` extension method. This method internally sets the default MVC route mapping configuration at startup configuration time and dispatches the controller action during request handling.

By default the out of the box template in v2.2 just configures the MVC dispatcher middleware. As such the MVC middleware also handles the route resolution based on the route map configuration and the incoming request data.

However we can add Endpoint Routing using some additional configuration as shown below:

```csharp

using Microsoft.AspNetCore.Internal;

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    //added endpoint routing that will resolve the endpoint object
    app.UseEndpointRouting();

    //middleware below will have access to the Endpoint

    app.UseHttpsRedirection();

    //the MVC middleware dispatches the controller action
    //and the MVC middleware configures the default route mapping
    app.UseMvc();
}
```

Here we have added the namespace `Microsoft.AspNetCore.Internal`. Including it, enables an additional `IApplicationBuilder` extension method `UseEndpointRouting` that is the endpoint resolution middleware that resolves the route and adds the Endpoint object to the httpcontext.

You can check out the source code for `UseEndpointRouting` extension method in version 2.2 at:

[https://github.com/aspnet/AspNetCore/blob/v2.2.4/src/Http/Routing/src/Internal/EndpointRoutingApplicationBuilderExtensions.cs](https://github.com/aspnet/AspNetCore/blob/v2.2.4/src/Http/Routing/src/Internal/EndpointRoutingApplicationBuilderExtensions.cs)

In version 2.2 the MVC middleware at the end of the pipeline acts as the endpoint dispatcher middleware. It will dispatch the resolved endpoint to the proper controller action.

The endpoint resolution middleware uses the route mappings configured by the MVC middleware.

Once we have enabled endpoint routing we can actually inspect the resolved endpoint object if we add our own custom middleware show below:

```csharp
using Microsoft.AspNetCore.Http.Features;
using Microsoft.AspNetCore.Routing;

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    app.UseEndpointRouting();

    app.UseHttpsRedirection();

    //our custom middlware
    app.Use((context, next) =>
    {
        var endpointFeature = context.Features[typeof(IEndpointFeature)] as IEndpointFeature;
        var endpoint = endpointFeature?.Endpoint;

        //note: endpoint will be null, if there was no resolved route
        if (endpoint != null)
        {
            var routePattern = (endpoint as RouteEndpoint)?.RoutePattern
                                                          ?.RawText;

            Console.WriteLine("Name: " + endpoint.DisplayName);
            Console.WriteLine($"Route Pattern: {routePattern}");
            Console.WriteLine("Metadata Types: " + string.Join(", ", endpoint.Metadata));
        }
        return next();
    });

    app.UseMvc();
}
```

As you can see we can inspect and print out the endpoint object that the endpoint routing resolution middleware
`UseEndpointRouting` has resolved. The endpoint object will be null, if the resolver was not able to match the request to a mapped route. We needed to pull in two additional namespaces to access the endpoint routing features.

### Endpoint routing in Version 3 preview 3

In version 3 preview 3, endpoint routing will become a full fledged citizen of ASP.NET Core and we will finally have separation between the MVC controller action dispatcher and the route resolution middleware.

Here is the endpoint startup configuration in version 3 preview 3.

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    app.UseHttpsRedirection();

    app.UseRouting(routes =>
    {
        routes.MapControllers();
    });

    app.UseAuthorization();

    //No need to have a dispatcher middleware here.
    //The resolved endpoint is automatically dispatched to a controller action at the end
    //of the middleware pipeline
    //If an endpoint was not able to be resolved, a 404 not found is returned at the end
    //of the middleware pipeline
}
```

As you can see we have a `app.UseRouting()` method that configures the endpoint route resolution middleware. The method also takes a anonymous lambda function that configures the route mappings that the route resolver middleware will use to resolve the incoming request endpoint.

The `routes.MapControllers()` inside the mapping function configures the default MVC routes.

You will also notice that there is a `app.UseAuthorization()` after `app.UseRouting()` which configures the authorization middleware. This middleware will have access the httpcontext endpoint object set by the endpoint routing middleware.

Notice that we don't have any MVC or endpoint dispatch middleware configured at the end of the method, after all other middleware configuration.

This is because the behavior of the version 3 preview 3 build is that the resolved endpoint will be implicitly dispatched to the controller action by the framework itself.

Similar to what we did for version 2.2 we can add the same custom middleware to inspect the resolved endpoint object.

```csharp
using Microsoft.AspNetCore.Http.Features;
using Microsoft.AspNetCore.Routing;

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    app.UseHttpsRedirection();

    app.UseRouting(routes =>
    {
        routes.MapControllers();
    });

    app.UseAuthorization();

    //our custom middleware
    app.Use((context, next) =>
    {
        var endpointFeature = context.Features[typeof(IEndpointFeature)] as IEndpointFeature;
        var endpoint = endpointFeature?.Endpoint;

        //note: endpoint will be null, if there was no
        //route match found for the request by the endpoint route resolver middleware
        if (endpoint != null)
        {
            var routePattern = (endpoint as RouteEndpoint)?.RoutePattern
                                                          ?.RawText;

            Console.WriteLine("Name: " + endpoint.DisplayName);
            Console.WriteLine($"Route Pattern: {routePattern}");
            Console.WriteLine("Metadata Types: " + string.Join(", ", endpoint.Metadata));
        }
        return next();
    });

    //the endpoint is dispatched by default at the end of the middleware pipeline
}
```

### Endpoint routing in upcoming ASP.NET Core version 3.0 source code repository

As we approach the version 3.0 release of the framework, it looks like the team is trying to make endpoint routing more explicit by adding back in the call to endpoint dispatcher middleware configuration. They are also moving back the route mapping configuration option to the dispatcher middleware configuration method.

We can see how this is changed yet again by peeking at the current source code.

Here is a snippet from the source code for the version 3.0 sample application music store at:

[https://github.com/aspnet/AspNetCore/blob/master/src/MusicStore/samples/MusicStore/Startup.cs](https://github.com/aspnet/AspNetCore/blob/master/src/MusicStore/samples/MusicStore/Startup.cs)

```csharp
public void Configure(IApplicationBuilder app)
{
    // Configure Session.
    app.UseSession();

    // Add static files to the request pipeline
    app.UseStaticFiles();

    // Add the endpoint routing matcher middleware to the request pipeline
    app.UseRouting();

    // Add cookie-based authentication to the request pipeline
    app.UseAuthentication();

    // Add the authorization middleware to the request pipeline
    app.UseAuthorization();

    // Add endpoints to the request pipeline
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllerRoute(
            name: "areaRoute",
            pattern: "{area:exists}/{controller}/{action}",
            defaults: new { action = "Index" });

        endpoints.MapControllerRoute(
            name: "default",
            pattern: "{controller}/{action}/{id?}",
            defaults: new { controller = "Home", action = "Index" });

        endpoints.MapControllerRoute(
            name: "api",
            pattern: "{controller}/{id?}");
    });
}
```

As you can see that we have something similar to my pseudo code implementation that I detailed above. 

Particularly, we still have the  `app.UseRouting()` middleware setup from the version 3 preview 3, but now we also have an explicit  `app.UseEndpoints()` endpoint dispatch method that more aptly named for what it does.

The `UseEndpoints` is a new `IApplicationBuilder` extension method that provides the endpoint dispatch implementation.

You can check out the source code for the `UseEndpoints` and `UseRouting` methods here:

[https://github.com/aspnet/AspNetCore/blob/master/src/Http/Routing/src/Builder/EndpointRoutingApplicationBuilderExtensions.cs](https://github.com/aspnet/AspNetCore/blob/master/src/Http/Routing/src/Builder/EndpointRoutingApplicationBuilderExtensions.cs)

Also note that the route mapping configuration lambda have been moved from the `UseRouting` middleware to the new `UseEndpoints` middleware. 

The `UseRouting` endpoint route resolver will still have access to the mappings to resolve the endpoint at request handling time. Even though they are passed to the `UseEndpoints` middleware  during startup configuration.

## Adding Endpoint routing middleware to the DI container

To use endpoint routing, we also need to add the middleware to the DI container in the `Startup.ConfigureServices` method.

### ConfigureServices in Version 2.2

For version 2.2 of the framework we need to explicitly add the call to `services.AddRouting()` method shown below to add the endpoint routing feature to the DI container:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddRouting()

    services.AddMvc()
            .SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
}
```

### ConfigureServices in Version 3 preview 3

For version 3 preview 3 build of the framework, endpoint routing already comes configured in the DI container under the covers in the `AddMvc()` extension method:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc()
            .AddNewtonsoftJson();
}
```

## Setting up Endpoint Authorization using endpoint routing and route mappings

When working with the version 3 preview 3 release we can attach authorization metadata to an endpoint. We do this using the route mappings configuration fluent API `RequireAuthorization` method.

The endpoint routing resolver will access this metadata when processing a request and add it to the Endpoint object that it sets on the httpcontext.

Any middleware in the pipeline after the route resolution middleware can access this authorization data by accessing the resolved Endpoint object.

In particular the authorization middleware can use this data to make authorization decisions.

Currently the route mapping configuration parameters are passed into the endpoint route resolver middleware, but as previously mentioned, in the future releases, the route mapping configuration will be passed into the endpoint dispatcher middleware.

Either way the attached authorization metadata will be available for the endpoint resolver middleware to use.

Below is an example of the version 3 preview 3 `Startup.Configure` method where I have added a new `/secret` route
to the endpoint resolver middleware route map configuration lambda parameter:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    app.UseHttpsRedirection();

    app.UseRouting(routes =>
    {
        routes.MapControllers();

        //Mapped route that gets attached authorization metadata using the RequireAuthorization extension method.
        //This metadata will be added to the resolved endpoint for this route by the endpoint resolver
        //The app.UseAuthorization() middleware later in the pipeline will get the resolved endpoint
        //for the /secret route and use the authorization metadata attached to the endpoint
        routes.MapGet("/secret", context =>
        {
            return context.Response.WriteAsync("secret");
        }).RequireAuthorization(new AuthorizeAttribute(){ Roles = "admin" });
    });

    app.UseAuthentication();

    //the Authorization middleware check the resolved endpoint object
    //to see if it requires authorization. If it does as in the case of
    //the "/secret" route, then it will authorize the route, if it the user is in the admin role
    app.UseAuthorization();

    //the framework implicitly dispatches the endpoint at the end of the pipeline.
}
```

You can see that I am using the `RequireAuthorization` method to add an `AuthorizeAttribute` attribute to the `/secret` route. This route will then only be authorized to be dispatched for a user in the admin role, by the authorization middleware, that comes before the endpoint dispatch occurs.

As I showed for version 3.3 preview 3 that we can add middleware to inspect the resolved endpoint object in httpcontext so can we here to inspect the AuthorizeAttribute added to the endpoint metadata:

```csharp
using Microsoft.AspNetCore.Http.Features;
using Microsoft.AspNetCore.Routing;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Authorization;

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    app.UseHttpsRedirection();

    app.UseRouting(routes =>
    {
        routes.MapControllers();

        routes.MapGet("/secret", context =>
        {
            return context.Response.WriteAsync("secret");
        }).RequireAuthorization(new AuthorizeAttribute(){ Roles = "admin" });
    });

    app.UseAuthentication();

    //our custom middleware
    app.Use((context, next) =>
    {
        var endpointFeature = context.Features[typeof(IEndpointFeature)] as IEndpointFeature;
        var endpoint = endpointFeature?.Endpoint;

        //note: endpoint will be null, if there was no
        //route match found for the request by the endpoint route resolver middleware
        if (endpoint != null)
        {
            var routePattern = (endpoint as RouteEndpoint)?.RoutePattern
                                                          ?.RawText;

            Console.WriteLine("Name: " + endpoint.DisplayName);
            Console.WriteLine($"Route Pattern: {routePattern}");
            Console.WriteLine("Metadata Types: " + string.Join(", ", endpoint.Metadata));
        }
        return next();
    });

    app.UseAuthorization();

    //the framework implicitly dispatches the endpoint here.
}
```

This time I have added the custom middleware before the Authorization middleware and pulled in two additional namespaces.

Navigating to the `/secret` route and inspecting the metadata, you can see that it contains the `Microsoft.AspNetCore.Authorization.AuthorizeAttribute` type in addition to the `Microsoft.AspNetCore.Routing.HttpMethodMetadata` type.

## References used for this article

The following articles contain source material that I used as a reference for this article:

[https://devblogs.microsoft.com/aspnet/aspnet-core-3-preview-2/](https://devblogs.microsoft.com/aspnet/aspnet-core-3-preview-2/)

[https://www.stevejgordon.co.uk/asp-net-core-first-look-at-global-routing-dispatcher](https://www.stevejgordon.co.uk/asp-net-core-first-look-at-global-routing-dispatcher)

## Update for vesrion 3 preview 4

Around the time I published this article, ASP.NET Core 3.0 preview 4 was released. It adds the changes that I described in the latest source code as can be seen in the code snippet from `Startup.cs` file below when creating a new webapi project:

```csharp
//code from Startup.cs file in a webapi project template

using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Http;

public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers()
            .AddNewtonsoftJson();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    app.UseHttpsRedirection();

    //add endpoint resolution middlware
    app.UseRouting();

    app.UseAuthorization();

    //add endpoint dispatch middleware
    app.UseEndpoints(endpoints =>
    {
        //route map configuration
        endpoints.MapControllers();

        //route map I added to show Authorization setup
        endpoints.MapGet("/secret", context =>
        {
            return context.Response.WriteAsync("secret");
        }).RequireAuthorization(new AuthorizeAttribute(){ Roles = "admin" });  
    });
}
```

As you can see this version adds the explicit `UseEndpoints` middleware extension method at the end of the pipeline that adds the endpoint dispatch middleware. Also the route configuration parameter has been moved from the `UseRouting` method in preview 3 to the `UseEndpoints` in preview 4.  

## Conclusion

Endpoint routing allows ASP.NET Core applications to determine the endpoint that will be dispatched, early on in the middleware pipeline, so that later middleware can use that information to provide features not possible with the current pipeline configuration.

This makes the ASP.NET Core framework more flexible since it decouples the route matching and resolution functionality from the endpoint dispatching functionality, which until now was all bundled in with the MVC middleware.