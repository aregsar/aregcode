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
