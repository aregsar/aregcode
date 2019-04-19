# Understanding ASP.NET Core Endpoint Routing

April 11, 2019 by [Areg Sarkissian](https://aregcode.com/about)

[Understanding .NET Core Endpoint Routing](https://aregcode.com/blog/2019/dotnetcore-understanding-aspnet-endpoint-routing)

## Introduction

In this post I will explain the new Endpoint Routing feature that has been added to the ASP.NET Core middleware pipeline starting with version 2.2 and how it is evolving through to the upcoming version 3.0 that is at preview 3 at present time.

## The motivation behine endpoint routing

Prior to endpoint routing the routing reslotion for an ASP.NET Core application was done in the ASP.NET Core MVC middleware at the end of the HTTP request processing pipeline. This meant that route information, such as what controller action would be executed, was not available to middleware that come before the MVC middleware in the pipeline.

It is particularly usefull to have this route information available for example in a CORS or authorization middleware to use the information as a factor in the authorization process.

Another potential usage of endpoint routing is that we can simulate the ability of frameworks like Laravel and Phoenix to can assign different middleware pipelines to different routes. 

With endpoint routing we can simulate this behavior by dynamically determining which middleware in the ASP.NET Core static middleware pipeline should apply to a route resolved by the endpoint routing middleware.

Endpoint routing allows us to decouple the route matching logic from the MVC middleware, moving it into its own middleware. It allows the MVC middleware to focus on its responsablity of dispatching the request to the particular controller action method that is resolved by the endpoint routing middleware.

Thus endpoint routing was born to allow the route resolution to happen earlier in the pipeline in a separate endpoint routing middleware that can be placed at any point in the pipeline (usually after the exception handling and static content serving middleware) after which other middleware in the pipeline, including the final MVC action method dispatcher middleware, can access the resolved route information.

The endpoint routing middleware API is evolving with the upcoming 3.0 version of the .NET Core framework. For that reason, the API that I will describe in the following sections may not be the final version of the feature. However the over all concept and understanding of how route resolution and dispacch work using endpoint routing should still apply.

In the following sections I will walk you through the current iterations of endpoint routing implementation from verion 2.2 to version 3.0 preview, then I will note some changes that are comming based on the the current ASP.NET Core source code repository.

## The three core concepts involved in Endpoint Routing

There are three overall concepts that you need to understand to be able to understand how endpoint routing works.

These are the following:

+ Endpoint route resolution
+ Endpoint dispatch
+ Endpoint route mapping

### Endpoint route resolution

The endpoint route resolution is the concept of looking at the incomming request and matching the request to an endpoint using route mappings.

Prior to endpoint routing, route resolution was done in the old MVC middleware that can be added to the middleware pipeline using the AddMvc() extension method. The current 2.2 version of the framework adds endpoint routing capability but keeps the route resolution in the MVC middleware. This will change in the 3.0 version where the route resolution will happen in the endpoint routing middleware.

So going forward route resolution will be done in its own endpoint routing middleware that we can add separately using its own AddRouting() extension method.

The job of the route resolution middleware is to construct and __Endpoint__ object using the route information from the route that it resolves based on the route mappings. The middleware then places that object into the httpcontext where the middleware in the pipeline the come after the endpoint routimg middleware can access the endpoint object to use the route information within.

### Endpoint dispatch

Endpoint dispatch is the process of invoking the controller action method that corresponds to the endpoint that was resolved by the endpoint routing middleware.

The endpoint dipatch middleware is the last middleware in the pipeline that grabs the endpoint object from the httpcontext and dispatches the the particular controller action that the resolved endpoint specifies.

Currently in version 2.2 dispatching to the action method is done in the MVC middleware at the end of the pipline, in the 3.0 preview 3 the MVC middleware is still being used to dispatch, but the upcoming 3.0 final release should have a new endpoint routing middleware that will assume the dispacth responsablity.

## Endpoint route mapping

When we define route middlware we can optionally pass in a lambda that contains custom route mappings that override the default route mapping that ASP.NET Core middleware extension method specifies.
Route mappings are used by the route resolution to match the incomming request parameters to a route specified in the rout mappings.

With the new endpoint routing we have to decide which middlware, the endpoint resolution or the endpoint dispatch middleware, should get the route mapping lambda as a parameter.

In fact this is the part of the endpoint routing where the API is in flux. As I write this post, the route mapping is being moved from the route resolution middleware to the endpoint dispatcher middleware, which is the middleware at the end of the pipeline.

To make this clear, I will show you the route mapping API in version 3 preview 3 first, then show you the latest route mapping API in the ASP.NET Core source code. In the code we can see that the route mapping is moved to the endpoint dispatcher middleware extension method which is currently called `UseEndpoint()`.

Its important to note that the endpoint resolution happens during runtime request handling after the route mapping is setup during application startup configuration. Therefor the route resolution middleware has access to the route mappings during request handling regardless of which middleware is passed the mappings as a parameter.

## Endpoint routing configuration

The middleware pipeline endpoint route resolver middleware, endpoint dispatcher middleware and endpoint route mapping lambda is setup in the `Startup.Configure` method of the `Startup.cs` file of ASP.NET Core project. This configuration changed between 2.2 and 3.0 preview 3 versions and is still changing before the 3.0 release version.

So in order to demonstrate the endpoint routing configuration I am going to declare the general form of the endpoint routing middleware configurartion as pseudo code based on the three core concepts listed above:

```csharp
public void Configure(IApplicationBuilder app
                     , IHostingEnvironment env)
{
    //middleware configured before the UseEndpointRouteResolverMiddleware middleware
    //that does not have access to the endpoint object
    app.UseBeforeEndpointResolutionMiddleware()

    //middleware that inspects the incoming request, resolves a match to the route map
    //and stores the resolved endpoint object into the httpcontext
    app.UseEndpointRouteResolverMiddleware(routes =>
    {
        routes.MapControllers();
    })

    //middleware after configured after the UseEndpointRouteResolverMiddleware middleware
    //that can access to the endpoint object
    app.UseAfterEndpointResolutionMiddleware()

    app.UseEndpointDispatcherMiddleware();
}
```

This version of our pseudocode shows the route mappings lambda passed as a parameter to the `UseEndpointRouteResolverMiddleware` endpoint route resolution middleware extension method.

An alternate version that matches what the current source code looks like, is shown below:

```csharp
public void Configure(IApplicationBuilder app
                     , IHostingEnvironment env)
{
    app.UseBeforeEndpointResolutionMiddleware()

    app.UseEndpointRouteResolverMiddleware();

    app.UseAfterEndpointResolutionMiddleware();

    app.UseEndpointDispatcherMiddleware(routes =>
    {
        routes.MapControllers();
    });
}
```

In this version the route mapping configuration is passed as a parameter to the endpoint dispatch middleware extension method `UseEndpointDispatcherMiddleware`.

Either way the `UseEndpointRouteResolverMiddleware()` endpoint resolver middleware will have access to the route mappings at request handling time to do the route matching.

Once the route is matched by `UseEndpointRouteResolverMiddleware` an Endpoint object will be constructed with the route parameters and set to the httpcontext so that the middleware in the pipeline that follow, can access the Endpoint object and use it if needed.

In version 3 preview 3 version of this pseudo code, the route mapping is passed to `UseEndpointRouteResolverMiddleware` and there does not exist a `UseEndpointDispatcherMiddleware` at the end of the pipeline. This is because the ASP.NET framework itself will implicitly dispatch the resolved endpoint at the end of the request pipeline.

So in version 3 preview 3 the code has the form below:

```csharp
public void Configure(IApplicationBuilder app
                     , IHostingEnvironment env)
{
    app.UseBeforeEndpointResolutionMiddleware()

    app.UseEndpointRouteResolverMiddleware(routes =>
    {
        routes.MapControllers();
    })


    app.UseAfterEndpointResolutionMiddleware()

    // The resolved endpoint is implicitly dispatched here at the end of the pipeline
    // and so there is no explicit call to UseEndpointDispatcherMiddleware
}
```

This looks like to be changing with the 3.0 release version, as the current source code shows that the `UseEndpointDispatcherMiddleware` is added back in and it is the middleware that takes the route mappings as a parameter as illustrated by the second version of the pseudocode above.

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
    //the MVC middleware configures the default route mapping
    //Note: we could pass in our own custom route map configuration lambda
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
    //DefaultEndpointDataSource
    //app.UseEndpoint();

    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseHsts();

    //added endpoint routing
    app.UseEndpointRouting();

    //middleware belwo will have access to the Endpoint

    app.UseHttpsRedirection();

    //the MVC middleware dispatches the controller action
    //the MVC middleware configures the default route mapping
    app.UseMvc();
}
```

Here we have added the namespace `Microsoft.AspNetCore.Internal`. Including it, enables an additional `IApplicationBuilder` extension method `UseEndpointRouting` that is the endpoint resolution middleware that resolves the route and adds the Endpoint object to the httpcontext.

In version 2.2 the MVC middleware at the end of the pipeline acts as the endpoint dispatcher middleware. It will dispatch the resolved endpoint to the proper controller action.

The endpoint resolution middleware uses the route mappings configured by the MVC middleware.

### Endpoint routing in V3 preview

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

    //No need to have a dipatcher middleware here.
    //The resolved endpoint is automatically dispacthed to a controller action at the end
    //of the middleware pipeline
    //If an endpoint was not able to be resolved, a 404 not found is returned at the end
    //of the middleware pipeline
}
```

As you can see we have a `app.UseRouting()` method that configures the endpoint route resolution middleware. The method also takes a anonymous lambda function that configures the route mappings that the route resolver middleware will use to resolve the incoming request endpoint.

The `routes.MapControllers()` inside the mapping function configures the default MVC routes.

You will also notice that there is a `app.UseAuthorization()` after `app.UseRouting()` which configures the authorization middleware. This middleware will have access the httpcontext endpoint object set by the endpoint routing middleware.

Notice that we dont have any MVC or endpoint dispatch middleware configured at the end of the method, after all other middleware configuration.

This is becuase the behavior of the version 3 preview 3 build is that the resolved endpoint will be implicitly dispatched to the controller action by the framework itself.

### Endpoint routing in upcoming ASP.NET Core version 3.0 source code repository

As we approach the version 3.0 release of the framework, it looks like the team is trying to make endpoint routing more explicit by adding back in the call to endpoint dispatcher middleware configuration. They are also moving back the route mapping configuration option to the dispatcher middleware configuration method.

We can see how this is changed yet again by peeking at the current source code.

Here is a sinppet from the source code for the version 3.0 sample application music store at:

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

As you can see that we have something similar to my pseudocode implementation that I detailed above. 

Particularly, we still have the  `app.UseRouting()` middleware setup from the version 3 preview 3, but now we also have an explicit  `app.UseEndpoints()` endpoint dispatch method that more aptly named for what it does.

The `UseEndpoints` is a new `IApplicationBuilder` extension method that provides the endpoint dispatch implementation.

You can check out the source code for the `UseEndpoints` and `UseRouting` methods here:

[https://github.com/aspnet/AspNetCore/blob/master/src/Http/Routing/src/Builder/EndpointRoutingApplicationBuilderExtensions.cs](https://github.com/aspnet/AspNetCore/blob/master/src/Http/Routing/src/Builder/EndpointRoutingApplicationBuilderExtensions.cs)

Also note that the route mapping configuration lambda have been moved from the `UseRouting` middleware to the new `UseEndpoints` middleware. 

The `UseRouting` endpoint route resolver will still have access to the mappings to resolve the endpoint at request handling time. Even though they are passed to the `UseEndpoints` middleware  during startup configuration.

## Adding Endpoint routing middleware to the DI container

To use endpoint routing, we also need to add the middleware to the DI container in the `Startup.ConfigureServices` method.

### ConfigureServices in Version 2.2

For version 2.2 of the framework we need to add the call to `services.AddRouting()` method shown below:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddRouting()

    services.AddMvc()
            .SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
}
```

### ConfigureServices in Version 3 preview 3

For version 3 preview 3 build of the framework, endpoint routing already comes configured under the covers in the `AddMvc()` extension method:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc()
            .AddNewtonsoftJson();
}
```

## Setting up Endpoint Authorization using endpoint routing and route mappings

When working with the version 3 preview 3 release we can attach autorization metadata to an endpoint. We do this using the route mappings configuration fluent API `RequireAuthorization` method.

The endpoint routing resolver will access this metadata when processing a request and add it to the Endpoint object that it sets on the httpcontext.

Any middleware in the pipeline after the route resolution middleware can access this authorization data by accessing the resolved Endpoint object.

In particular the authorization middleware can use this data to make authorization deciesions.

Currently the route mapping configuration parameters are passed into the endpoint route resolver middleware, but as previously mentioned, in the future releases, the route mapping configuration will be passed into the endpoint dispatcher middleware.

Either way the attached authorization metadata will be availailble for the endpoint resolver middleware to use.

Below is an example of the version 3 preview 3 `Startup.Configure` method where I have added a new `/hello` route
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

        //Mapped route that gets attaced authorization metadata using the RequireAuthorization extension method.
        //This metadata will be added to the resolved endpoint for this route by the endpoint resolver
        //The app.UseAuthorization() middleware later in the pipeline will get the resolved endpoint
        //for the /hello route and use the authorization metadata attached to the endpoint
        routes.MapGet("/hello", context =>
        {
            return context.Response.WriteAsync("hello");
        }).RequireAuthorization(new AuthorizeAttribute(){ Roles = "admin" });
    });

    app.UseAuthentication();

    //the Authorization middleware check the resolved endpoint object
    //to see if it requires authorization. If it does as in the case of
    //the "/hello" route, then it will authorize the route, if it the user is in the admin role
    app.UseAuthorization();

    //the framework implicitly dispaches the endpoint here.
}
```

You can see that I am using the `RequireAuthorization` method to add an `AuthorizeAttribute` attribute to the `/hello` route. This route will then only be authorized to be dispactched for a user in the admin role, by the authorization middleware, that comes before the endpoint dispatch occurs.

## Conclusion

Endpoint routing allows ASP.NET Core applications to determine the endpoint that will be dispatched, early on in the middleware pipeline, so that later middleware can use that information to provide features not possible with the current pipeline configuration.

This makes the ASP.NET Core framework more flexible since it decouples the route matching and resolution functionality from the endpoint dispatching functionality, which until now was all bundled in with the MVC middleware.