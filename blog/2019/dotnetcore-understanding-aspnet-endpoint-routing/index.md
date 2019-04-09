# Understanding ASP.NET Core Endpoint Routing

[Understanding .NET Core Endpoint Routing](https://aregcode.com/blog/2019/dotnetcore-understanding-aspnet-endpoint-routing)

## Introduction

In this post I will explain the new Endpoint Routing feature that has been added to the ASP.NET Core middleware pipeline starting with version 2.2 and how it is evolving through to the upcoming version 3.0 that is in preview 3 at present time.

## The motivation behine endpoint routing

Prior to endpoint routing the routing reslotion for an ASP.NET Core application was done in the ASP.NET Core MVC middleware at the end of the HTTP request processing pipeline. This meant that route information was not available to middleware before the MVC middleware.

It is particularly usefull to have this route information available for example to authorization middleware to use the information as a factor in the authorization process.

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

> Note: The routing pipeline endpoint resolver middleware, endpoint dispatcher middleware and route mapping configuration is configured in the `Startup.Configure` method.

### Endpoint routing in V2.2 preview

If you create a webapi project using version 2.2 of the .NET Core SDK you will see the 
following code in the `Startup.Configure` method.
The MVC middleware is configured at the end of the middleware pipeline using the `UseMvc()` method. This method internally configures the MVC controller action dispatcher middleare and the default MVC route mapping configuratin. It also handles the route resolution based on the route map configuration and the incoming request data.

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

### Endpoint routing in V3 preview



Here is the endpoint routing configuration in version 3 preview.
As you can see we have a `app.UseRouting(...)` method that configures the endpoint resolution middleware. The method also takes a anonymous lambda function that configures the route mappings that the endpoint resolver will use to resolve the incoming request route endpoint. The `routes.MapControllers()` inside the mapping function configures the default MVC routes.

You will also notice that there is a `app.UseAuthorization()` after `app.UseRouting(...)` which configures the authorization middleware. This middleware can not access the httpcontext endpoint object set by the endpoint routing middleware.

Notice that we dont have any MVC middleware configuration at the end after configuring all other middleware. This is becuase the behavior of the v3.0 preview build is that the endpoint resolved by the  `app.UseRouting(...)` method will automatically be dispatched.

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

### Endpoint routing in ASP.NET Core source code repository for upcomming V3 release

We will see next how this is changed yet again in the current source code to make it more explicit that we have a dispacther middleware at the end. Also the route mappings are moved to the dispatcher middleware configuration.


## Adding Endpoint routing middleware to the DI container

### V3 preview

```csharp
public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc()
                .AddNewtonsoftJson();
        }
```

## Endpoint Mappings and Endpoint Authoriation
