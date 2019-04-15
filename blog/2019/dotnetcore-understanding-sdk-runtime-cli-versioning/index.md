# Understanding .NET Core SDK, Runtime and CLI versioning

April 15, 2019 by [Areg Sarkissian](https://aregcode.com/about)

## Introduction

In this post I will detail how to install the .NET Core framework and how the SDK and Runtime versions are specified for projects using the __CLI__ and the `global.json` and `.csproj` files.

## Installing the .NET Core SDK and Runtime

You can find the .NET Core SDK and Runtime downloads for your platform at:

[https://dotnet.microsoft.com/download/dotnet-core](https://dotnet.microsoft.com/download/dotnet-core)

Navgiate to the .NET Core version that you want to install, then download the appropriate version of the .NET Core SDK installer for your platform.

For my system I selected the macOS x64 .NET Core SDK installer for version 3.0.100-preview3-010431.

Run the installer to install the .Net Core SDK, Runtime and CLI tools.

## .NET Core SDK versions and the CLI

We can use a CLI command to list all installed SDK versions on our system.

Here is a list of the SDK versions installed on my system:

```bash
dotnet --list-sdks
1.0.0-preview1-002702 [/usr/local/share/dotnet/sdk]
1.0.1 [/usr/local/share/dotnet/sdk]
1.0.4 [/usr/local/share/dotnet/sdk]
2.0.0 [/usr/local/share/dotnet/sdk]
2.1.302 [/usr/local/share/dotnet/sdk]
2.2.100 [/usr/local/share/dotnet/sdk]
3.0.100-preview3-010431 [/usr/local/share/dotnet/sdk]
```

We can see that version 3.0.100-preview3-010431 is the last installed version.

By default the latest installed SDK version will be used by the CLI commands.

I can execute the CLI with the version flag to show the default SDK version that will be used by the CLI tooling on my system:

```bash
dotnet --version
3.0.100-preview3-010431
```

We can see that version 3.0.100-preview3-010431 is the default SDK version that will be used.

## Using the .NET Core CLI to create a new project

Let's use the CLI to create a new Web API project that will use the default SDK version:

```bash
dotnet new webapi -o demo
```

The -o flag specifies a project directory that will be created when we run the command.

We can enter the project directory and check the SDK version:

```bash
cd demo
dotnet --version
3.0.100-preview3-010431
```

We can see that the default version 3.0.100-preview3-01043 is being used.

## The .NET Core Runtime version and the .csproj file

The runtime version for our project is specified by the generated demo.csproj file for the project that we created above.

We can dump the content of the demo.csproj file to inspect the Runtime version being targeted.

```bash
cat demo.csproj
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp3.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Mvc.NewtonsoftJson" Version="3.0.0-preview3-19153-02" />
  </ItemGroup>

</Project>
```

The `TargetFramework` element in demo.csproj file specifies the .NET Core runtime version that we are targeting. In this case, the version that we are targeting is netcoreapp3.0.

You can find more information about target frameworks in the docs at [https://docs.microsoft.com/en-us/dotnet/standard/frameworks](https://docs.microsoft.com/en-us/dotnet/standard/frameworks)

Just as we did for the SDK versions, We can list all the installed .NET Core Runtimes on our system using the CLI.

On my system the list looks like this:

```bash
dotnet --list-runtimes
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
```

## Specifying the .NET Core SDK version with global.json

By adding a global.json file to the root directory of our project, we can force the dotnet CLI to use a specific version of the SDK for our project.

The .NET Core documentation at [https://docs.microsoft.com/en-us/dotnet/core/tools/global-json](https://docs.microsoft.com/en-us/dotnet/core/tools/global-json) gives us the following definition of the global.json file:

> The global.json file allows you to define which .NET Core SDK version is used when you run .NET Core CLI commands

When you run a CLI command such as `dotnet new` or `dotnet build`, the command will search for a global.json
file first in the current directory then all parent directories up to the root directory of the filesystem.

If it can't find a global.json file, it will default to the newest installed version of the SDK, otherwise
it will use the SDK version specified in the first global.json file it encounters.

So by adding a global.json file at the root directory of a project or at the root directory of a solution for multi project solutions, we can specify the exact SDK version that the dotnet CLI commands should use.  

As an example let's use the CLI to create a new `global.json` file that specifies the .NET Core SDK version 2.2.100 is to be used:

```bash
dotnet new globaljson --sdk-version 2.2.100 -o demo2
```

The -o flag specifies that the output directory for the generated global.json file.

We can enter the created directory and dump the content of the global.json file to see that the SDK version 2.2.100 is specified:

```bash
cd demo2
cat global.json
{
  "sdk": {
    "version": "2.2.100"
  }
}
```

We can also check the SDK version using the CLI, from within the demo2 directory, to see that we are now using the specific 2.2.100 version:

```bash
dotnet --version
2.2.100
```

Now let's create a new Web API project inside the directory containing the global.json file:

```bash
dotnet new webapi
```

> Note: we didnt specify an output directory option this time since we are running the `dotnet new` command
inside the project directory that was created when we created the global.json file. The `dotnet new` command will use the current directory name, `demo2`, as the `.csproj` project file name.

We can dump the content of the generated demo2.csproj project file to see the runtime version that we are targeting:

```bash
cat demo2.csproj
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

This time we can see that the TargetFramework is set to the netcoreapp2.2 runtime.

## Adding a global.json file to an existing project

When we want to add a global.json file to an already existing project we can go into the project directory and run the CLI command to generate the global.json file there, without specifying the output directory option.

As an example I can add a global.json file to the first demo project that we created, like so:

```bash
cd demo
dotnet new globaljson --sdk-version 3.0.100-preview3-010431
```

Now we have explicitly locked that project to use version 3.0.100-preview3-010431. The project will continue using version 3.0.100-preview3-010431 even if we later install a newer version of the framework on our system.

If an existing project already contains a global.json file, you can change its .NET Core version, going forward, by changing the framework version specified inside the global.json file.

## Recommendations for using global.json files

I have two general recomendations regarding the usage of global.json with any .NET Core project:

+ Always add a global.json file to your projects to lock the SDK version used by the CLI. This will allow every project to use its own .NET Core version regardless of the default installed SDK version or the versions used in other projects.

+ Always create a global.json file before using the CLI to generate a new project. This way, the project scaffolding will use the .NET Core version specified in global.json when creating the inital project files.

## Extending the .Net Core CLI by installing .Net Core Global tools

Global tools are NuGet packages that add features to the .NET Core CLI and are unrelated to the SDK and CLI versioning.

You can add features to the dotnet cli by intalling global tools as documented here:

[https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools](https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools)

You can find a list of available tools here:

[https://github.com/natemcmaster/dotnet-tools](https://github.com/natemcmaster/dotnet-tools)

## References used for this article

The following articles contain source material that I used as a reference for this article:

[https://elanderson.net/2018/09/controlling-net-cores-sdk-version/](https://elanderson.net/2018/09/controlling-net-cores-sdk-version/)

[https://andrewlock.net/the-net-core-2-0-preview-1-version-numbers-and-global-json/](https://andrewlock.net/the-net-core-2-0-preview-1-version-numbers-and-global-json/)

## Conclusion

In this post I explained how the the .NET Core SDK and Runtime versions are specified and how you can set the SDK and Runtime versions by using the CLI and the global.json and .csproj files.

Thanks for reading.