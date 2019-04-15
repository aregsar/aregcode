# Understanding .NET Core SDK, Runtime and CLI versioning

April 11, 2019 by [Areg Sarkissian](https://aregcode.com/about)

[Understanding .NET Core SDK, Runtime and CLI versioning](https://aregcode.com/blog/2019/dotnetcore-understanding-sdk-runtime-cli-versioning)

## Introduction

In this post I will detail how to install the .NET Core framework and how the SDK and Runtime versions are specified for projects using the CLI and the global.json and .csproj files.

## Installing the .NET Core SDK and Runtime

You can find the .Net Core SDK and Runtime downloads for your platform at:

[https://dotnet.microsoft.com/download/dotnet-core](https://dotnet.microsoft.com/download/dotnet-core)

Naviate to the .NET Core version that you want to install, then download the right version of the .NET Core SDK installer for your platform.

For my system I Selected the macOS x64 .NET Core SDK installer for version 3.0.100-preview3-010431

Run the installer to install the .Net Core SDK, Runtime and CLI tools

## .NET Core SDK versions and the CLI

We can use the CLI to list all installed SDK versions on our system.

Here is what is installed on my system:

```bash
Aregs-MacBook-Pro:~ aregsarkissian$ dotnet --list-sdks
1.0.0-preview1-002702 [/usr/local/share/dotnet/sdk]
1.0.1 [/usr/local/share/dotnet/sdk]
1.0.4 [/usr/local/share/dotnet/sdk]
2.0.0 [/usr/local/share/dotnet/sdk]
2.1.302 [/usr/local/share/dotnet/sdk]
2.2.100 [/usr/local/share/dotnet/sdk]
3.0.100-preview3-010431 [/usr/local/share/dotnet/sdk]
```

We can see 3.0.100-preview3-010431 as the last installed version

We can also use the CLI to show the default SDK version that will be used by the CLI

```bash
Aregs-MacBook-Pro:~ aregsarkissian$ dotnet --version
3.0.100-preview3-010431
```

We can see that version 3.0.100-preview3-010431 is the default SDK version that will be used by the dotnet CLI commands

## Using the .NET Core CLI to create a new project

Let's use the dotnet cli to create a new Web API project that will
use the default latest installed SDK version:

```bash
Aregs-MacBook-Pro:asp3 aregsarkissian$ dotnet new webapi -o demo
```

The -o flag specifies a project directory that will be created when we run the command.

We can enter project directory and check the SDK version:

```bash
Aregs-MacBook-Pro:asp3 aregsarkissian$ cd demo
Aregs-MacBook-Pro:demo aregsarkissian$ dotnet --version
3.0.100-preview3-010431
```

We can see that the default version 3.0.100-preview3-01043 is being used.

## The .NET Core Runtime version and the .csproj file

The runtime version for our project is specified in the generated demo.csproj file for the project that we created.

We can dump content of the demo.csproj file to see that we are targeting netcoreapp3.0 runtime:

```bash
Aregs-MacBook-Pro:demo aregsarkissian$ cat demo.csproj
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp3.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Mvc.NewtonsoftJson" Version="3.0.0-preview3-19153-02" />
  </ItemGroup>

</Project>
```

The TargetFramework element in demo.csproj file specifies the dotnet core runtime version
that we are targeting, which is netcoreapp3.0.

You can find more information about this in the docs at [https://docs.microsoft.com/en-us/dotnet/standard/frameworks](https://docs.microsoft.com/en-us/dotnet/standard/frameworks)

Just as we did for the SDK versions, We can list all the installed .NET Core Runtimes on our system using the CLI.

On my system it looks like this:

Aregs-MacBook-Pro:~ aregsarkissian$ dotnet --list-runtimes
Microsoft.AspNetCore.All 2.1.2 [/usr/local/share/dotnet/shared/Microsoft.AspNetCore.All]
Microsoft.AspNetCore.All 2.2.0 [/usr/local/share/dotnet/shared/Microsoft.AspNetCore.All]
Microsoft.AspNetCore.App 2.1.2 [/usr/local/share/dotnet/shared/Microsoft.AspNetCore.App]
Microsoft.AspNetCore.App 2.2.0 [/usr/local/share/dotnet/shared/Microsoft.AspNetCore.App]
Microsoft.AspNetCore.App 3.0.0-preview3-19153-02 [/usr/local/share/dotnet/shared/Microsoft.AspNetCore.App]
Microsoft.NETCore.App 1.0.0-rc2-3002702 [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 1.0.4 [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 1.0.5 [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 1.1.1 [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 1.1.2 [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 2.0.0 [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 2.1.2 [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 2.2.0 [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 3.0.0-preview3-27503-5 [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]

## Specifying the .NET Core SDK version with global.json

By using a global.json file at the root of the project, we can force the dotnet CLI to use a specific version of the SDK for individual projects.

The .NET Core docs at [https://docs.microsoft.com/en-us/dotnet/core/tools/global-json](https://docs.microsoft.com/en-us/dotnet/core/tools/global-json) give us information about the global.json file:

> The global.json file allows you to define which .NET Core SDK version is used when you run .NET Core CLI commands

When you run a CLI command such as dotnet new or dotnet build, the command will search for a global.json
file first in the current directory then all parent directories up to the root directory.

If it can't find a global.json file, it will default to the newest installed version of the SDK, otherwise
it will use the SDK version specified in the first global.json file it finds.

So by adding a global.json file at the root of the project or the root of the solution for multi project solutions, we can specify the exact SDK version that the dotnet CLI commands should use.  

As an example lets use the CLI to create a new global.json file that specifies the .NET Core SDK version 2.2.100:

```bash
Aregs-MacBook-Pro:asp2 aregsarkissian$ dotnet new globaljson --sdk-version 2.2.100 -o demo2
```

The -o flag specifies that the output directory to place the global.json file in.

We can enter the project directory and the dump the generated global.json file to see that the SDK version 2.2.100 is specified:

```bash
Aregs-MacBook-Pro:asp3 aregsarkissian$ cd demo2
Aregs-MacBook-Pro:demo2 aregsarkissian$ cat global.json
{
  "sdk": {
    "version": "2.2.100"
  }
}
```

We can also check the SDK version using the CLI, from within the demo2 directory, to see that we are now using the specific 2.2.100 version:

```bash
Aregs-MacBook-Pro:demo2 aregsarkissian$ dotnet --version
2.2.100
```

Now let's create a new Web API project inside the directory containing the global.json file:

```bash
Aregs-MacBook-Pro:demo2 aregsarkissian$ dotnet new webapi
```

> Note: we didnt specify an output directory option this time since we are running the `dotnet new` command
inside the project directory that was created when we created the global.json file. The `dotnet new` command will use the current directory name, `demo2`, as the .csproj project file name.

We can dump the content of the generated demo2.csproj project file to see that we are now targeting netcoreapp2.2:

```bash
Aregs-MacBook-Pro:demo2 aregsarkissian$ cat demo2.csproj
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp2.2</TargetFramework>
    <AspNetCoreHostingModel>InProcess</AspNetCoreHostingModel>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.App" />
    <PackageReference Include="Microsoft.AspNetCore.Razor.Design" Version="2.2.0" PrivateAssets="All" />
  </ItemGroup>

</Project>
```

This time we can see that the TargetFramework is set to netcoreapp2.2.

## Adding a global.json file to an existing project

When we want to add a global.json file to an already existing project we can go into the project directory and run the CLI command to create the global.json file without specifying the output directory option.

As an example I can add a global.json file to the first demo project that we created, like so:

```bash
Aregs-MacBook-Pro:asp2 aregsarkissian$ cd demo
Aregs-MacBook-Pro:demo aregsarkissian$ dotnet new globaljson --sdk-version 3.0.100-preview3-010431
```

If an existing project already contains a global.json file, you can change its .NET Core version, going forward, by changing the version specified inside the global.json file.

## Recommendations for using global.json files

I have two general recomendations regarding the usage of global.json with any .NET Core project:

+ Always add a global.json file to your projects to 
control the SDK version used by the CLI, specially when you install multiple versions of the
framework on your system. This will allow every project to use its own .NET Core version regardless of the default instaled version or versions used in other projects. So for example if you install the latest preview version of the framework on your system, your existing projects can still use the older version specified by their local global.json files.

+ Always create a global.json file before using the CLI to 
generate a new project. This way, the project scaffolding uses the .NET Core version specified in global.json when creating the inital project  files.

## Extending the .Net Core CLI by installing .Net Core Global tools

Global tools are NuGet packages that add features to the dotnet cli and are unrelated to the SDK and CLI versioning

You can add features to the dotnet cli by intalling global tools as documented here:

https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools

you can find a list of available tools here:

https://github.com/natemcmaster/dotnet-tools

## References used for this article



## Conclusion

In this post I detailed What the .Net Core SDK and Runtime versions are and how you can select the SDK and Runtime versions by using the CLI and the global.json and .csproj files.

Thanks for reading





