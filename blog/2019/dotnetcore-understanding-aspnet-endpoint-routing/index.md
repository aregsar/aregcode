# Understanding ASP.NET Core Endpoint Routing

[Understanding .NET Core Endpoint Routing](https://aregcode.com/blog/2019/dotnetcore-understanding-aspnet-endpoint-routing)

## Introduction

In this post I will explain the new Endpoint Routing feature that has been added to the ASP.NET Core middleware pipeline starting with version 2.2 and how it is evolving through to the upcoming version 3.0 that is in preview 3 at present time.

## The motivation behine endpoint routing

Prior to endpoint routing the routing reslotion for an ASP.NET Core application was done in the ASP.NET Core MVC middleware at the end of the HTTP request processing pipeline. This meant that route information was not available to middleware before the MVC middleware.

It is particularly usefull to have this route information available for example in a CORS or authorization middleware to use the information as a factor in the authorization process.

Another potential usage is to dynamically determine which pipeline middleware should apply to which routes. In fact this is done in many other web frameworks such as Laravel.

Using endpoint routing also allows us to decouple the route resolution logic from the MVC middleware,  moving it into its own middleware.
It allows the mvc middleware to focus on its responsablity of dispatching the request to the particular "endpoint" aka "controller action" that was resolved by the endpoint routing middleware.

Thus endpoint routing was born to allow the route resolution to happen earlier in the pipeline in a separate endpoint routing middleware that can be placed at any point in the pipeline (usually after the exception handling and static content middleware)
after which other middleware in the pipeline, including the final MVC endpoint dispatcher middleware, can access resolved route information.

The middleware API for endpoint routing is evolving with the upcomming 3.0 version of the .NET Core framework so what I will describe in the following sections may not be the final version, however the over all concept and understanding of how route resolution and dispacch work will still apply.

In the following sections I will walk you through the current iterations of endpoint routing implementation from verion 2.2 to version 3.0 preview, then I will note some changes that are comming based on the the current ASP.NET Core source code repository.

## The three core concepts involved in Endpoint Routing

The three overall concepts that you need to understand to understand how endpoint routing works are the following:

+ Endpoint route resolution
+ Endpoint dispatch
+ Endpoint route mapping

I have already mentioned the first two of these.

The endpoint route resolution is the concept of looking at the incomming request and matching to an endpoint using route mappings.

This was done in the old MVC middleware that we add using the AddMvc() extension method. Going forward this will be done in its own endpoint routing middleware that we can add separately using its own AddRouting() extension method. 

The job of the route resolution middleware is to construct and Endpoint object using the route information from the route that it resolves based on the route mappings and place that object into the httpcontext where the following middleware in the pipeline can access this endpoint object to use the route information within.

The endpoint dipatch middleware is the last middleware in the pipeline that
grabs the endpoint object from the httpcontext and dispatches the endpoint that was resolved by the route resolution middleware to call the particular controller action that the resolved endpoint specifies.

The route mappings that the route resolution uses can be declared as configuration options that could be passed to either one of the route resolution or route dispatcher middleware. In fact this is the part of the endpoint routing where the API is in flux as I write this post and it looks like the mapping API is being moved from the route resolution middleware to the route dispatcher middleware which is the MVC middleware at the end of the pipeline.

To drive this last point how I will show you the route mapping API in version 3 preview first then show you the latest route mapping API in the ASP.NET Core source code as yet unrelease that moves the route mapping the route dispatcher middleware that is now called
`UseEndpoint()` instead of `UseMvc()`

Its important to note that the Endpoint resolution and dispatch happen during runtime request handling after the routing middleware and route mapping is configured during application startup configuration. Therefor the route resolution middleware during request handling has access to the route mappings configured during startup.

## Endpoint routing configuration

The routing pipeline endpoint resolver middleware, endpoint dispatcher middleware and route mapping configuration is configured in the `Configure` method of the `Startup.cs` file of ASP.NET projects. This configuration changed between 2.2 and 3.0 preview versions and is still changing before the v3 release. 

However we can declare the general form of the endpoint routing middleware configuration as pseudo code based on the core concepts listed above:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseBeforeEndpointResolutionMiddleware()

    app.UseEndpointRoutingResolverMiddleware(routes =>
    {
        routes.MapControllers();
    })

    app.UseAfterEndpointResolutionMiddleware()

    app.UseEndpointDispatcherMiddleware();
}
```

This version of our pseudocode shows the route mappings with the UseEndpointRoutingResolver middleware.
An alternate version could be with the mappings done in the UseEndpointDispatcherMiddleware

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseBeforeEndpointResolutionMiddleware()

    app.UseEndpointRoutingResolverMiddleware();

    app.UseAfterEndpointResolutionMiddleware()

    app.UseEndpointDispatcherMiddleware(routes =>
    {
        routes.MapControllers();
    })
}
```

Either way the UseEndpointRoutingResolverMiddleware() endpoint resolver middleware will have access to the route mappings at request processing time to do the route matching.
Onece the route is matched by UseEndpointRoutingResolverMiddleware and Endpoint object will be constructed with the route parameters and set to the httpcontext so that the following middlewares in the pipeline can access the Endpoint object and use it if needed.

In version 3 preview the route mapping passed to UseEndpointRoutingResolverMiddleware and the final UseEndpointDispatcherMiddleware is not present because the framework itself will dispacth the resolved endpoint at the end of the request pipeline. This looks like to be changing with the v3 release version as the current source code shows that the UseEndpointDispatcherMiddleware is added back in and it is the middleware that takes the route mappings as a parameter.

### Endpoint routing in V2.2 preview

If you create a webapi project using version 2.2 of the .NET Core SDK you will see the 
following code in the `Startup.Configure` method.

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    app.UseHttpsRedirection();

    app.UseMvc();
}
```

The MVC middleware is configured at the end of the middleware pipeline using the `UseMvc()` method. This method internally configures the default MVC route mapping configuration and the MVC controller action dispatcher middleware that dispatches the controller action.
By default the out of the box template in v2.2 just configures the MVC dispatcher middleware. AS such the MVC middleware also handles the route resolution based on the route map configuration and the incoming request data. 

However we can endable Endpoint Routing using some additional configuration. The modified code is shown below:

```csharp

using Microsoft.AspNetCore.Internal;

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    app.UseEndpointRouting();

    app.UseHttpsRedirection();

    app.UseMvc();

    //DefaultEndpointDataSource
    //app.UseEndpoint();
}
```

Here we have added the namespace Microsoft.AspNetCore.Internal which enables an additional IApplicationBuilder extension method `UseEndpointRouting` that is the endpoint resolution middleware that resolves the route and adds the Endpoint object to the httpcontext. In version 2.2 the MVC middleware at the end of the pipeline acts as the endpoint dispatcher middleware will dispatch the matched endpoint to the proper controller action. The endpoint resolution middleware uses the routes configured by the MVC middleware.

### Endpoint routing in V3 preview

In v3 endpoint routing will become a full fledged citizen of ASP.NET Core and we will finally have separation between the MVC controller action dispatcher and the routing middleware.

Here is the endpoint routing configuration in version 3 preview.

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

    //No need to have a dipatcher middleware here.
    //The resolved endpoint is automatically dispacthed to a controller action
    //or a 404 not found is returned if an endpoint was not able to be resolved
}
```

As you can see we have a `app.UseRouting(...)` method that configures the endpoint resolution middleware. The method also takes a anonymous lambda function that configures the route mappings that the endpoint resolver will use to resolve the incoming request route endpoint. The `routes.MapControllers()` inside the mapping function configures the default MVC routes.

You will also notice that there is a `app.UseAuthorization()` after `app.UseRouting(...)` which configures the authorization middleware. This middleware can not access the httpcontext endpoint object set by the endpoint routing middleware.

Notice that we dont have any MVC middleware configuration at the end after configuring all other middleware. This is becuase the behavior of the v3.0 preview build is that the endpoint resolved by the  `app.UseRouting(...)` method will automatically be dispatched.

### Endpoint routing in ASP.NET Core source code repository for upcomming V3 release

We will see next how this is changed yet again in the current source code to make it more explicit that we have a dispacther middleware at the end. Also the route mappings are moved to the dispatcher middleware configuration.

Here is a sinppet form the v3.0 sample application music store at:

https://github.com/aspnet/AspNetCore/blob/master/src/MusicStore/samples/MusicStore/Startup.cs

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

Now you can see that we have something similar to our pseudocode implementation. Particularly we still have the  app.UseRouting() from the v3 preview version, but now we also have an explicit  app.UseEndpoints(...) endpoint resolver method that is a new IApplicationBuilder extension method that provides the endpoint dispatch implementation. Also the route mappings have been moved from the UseRouting method to the new UseEndpoints method. The UseRouting endpoint resolver will still have access to the mappings defined at application startup to resolve the endpoint at request handling time.

We can also see the explicit mappings inside the lambda function.

You can checkout  the source for the  UseEndpoints and UseRouting methods here:

https://github.com/aspnet/AspNetCore/blob/master/src/Http/Routing/src/Builder/EndpointRoutingApplicationBuilderExtensions.cs

## Adding Endpoint routing middleware to the DI container

### V3 preview

```csharp
public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc()
                .AddNewtonsoftJson();
        }
```

## Endpoint Mappings and Endpoint Authoriation in V3 preview 3

```csharp

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();

    app.UseStaticFiles();

    app.UseRouting(routes =>
    {
        routes.MapApplication();

        routes.MapGet("/hello", context =>
        {
            return context.Response.WriteAsync("Hi there! Here's your secret message");
        })
        .RequireAuthorization(new AuthorizeAttribute(){ Roles = "secret-messages", });

        routes.MapHealthChecks("/healthz").RequireAuthorization("admin");
    });

    app.UseAuthentication();
    app.UseAuthorization();

```
