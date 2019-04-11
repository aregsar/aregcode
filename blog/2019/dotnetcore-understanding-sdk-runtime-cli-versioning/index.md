# Understanding .NET Core SDK, Runtime and CLI versioning

April 11, 2019 by [Areg Sarkissian](https://aregcode.com/about)

[Understanding .NET Core SDK, Runtime and CLI versioning](https://aregcode.com/blog/2019/dotnetcore-understanding-sdk-runtime-cli-versioning)

## Introduction

In this post I will detail how to install the .net core framework and how the SDK and Runtime versions are specified for our projects using the CLI and the global.json and .csproj files.

## Installing the .net core SDK and Runtime

You can find the .Net Core SDK and Runtime downloads for your platform at:

https://dotnet.microsoft.com/download/dotnet-core/3.0

Select and download the right version for your OS.

For my system I Selected macOS .net core x64 SDK installer for version 3.0.100-preview3-010431

Run the installer to install the latest version of .Net Core SDK, Runtime and CLI tool

## .net core SDK version and the CLI

We can use the CLI to list all installed SDK versions

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

We can use the CLI to show the default SDK version that will be used by the CLI

Aregs-MacBook-Pro:~ aregsarkissian$ dotnet --version
3.0.100-preview3-010431

we can see that version 3.0.100-preview3-010431 is the default SDK version being used by the dotnet cli commands

## .net SDK version for projects

We can force the dotnet cli to use a specific version of the SDK for individual 
projects using a global.json file

But for now lets just use the dotnet cli to create a new web api project that will
use the default latest installed SDK version: 

Aregs-MacBook-Pro:asp3 aregsarkissian$ dotnet new webapi -o demo

The -o flag specifies a project directory that will be created when we run the command

We can enter project directory and check the SDK version to see that we are using the 
latest sdk version 3.0.100-preview3-010431 version installed on my system:

Aregs-MacBook-Pro:asp3 aregsarkissian$ cd demo
Aregs-MacBook-Pro:demo aregsarkissian$ dotnet --version
3.0.100-preview3-010431

We can see that the default version is being used

## dotnet core Runtime version and the .csproj file

The runtime version for our project is specified in the generated .csproj file 

We can  dump content of the demo.csproj file to see that we are targeting netcoreapp3.0 runtime

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

TargetFramework element in demo.csproj file specifies the dotnet core runtime version
that we are targeting.

See https://docs.microsoft.com/en-us/dotnet/standard/frameworks

I can list all the installed runtimes on my system using the CLI


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



## Selecting the .net core SDK version to use with the global.json file

From the docs "The global.json file allows you to define which .NET Core SDK version is 
used when you run .NET Core CLI commands"

https://docs.microsoft.com/en-us/dotnet/core/tools/global-json

When you run for example dotnet new or dotnet build, the dotnet command will search for a global.json
file first in the current directory then all parent directories up to the root directory. 
If it can't find global.json file, it will default to the newest installed version of the SDK, otherwise
it will use the SDK version specified in the first global.json file it finds.

So we can specify the SDK version the dotnet cli commands should use 
by adding a global.json file at the root of the project or the root of
the solution for multi project solutions.  

We can create a new global.json file that specifies the sdk version 2.2.100 using the cli:

```bash
Aregs-MacBook-Pro:asp3 aregsarkissian$ dotnet new globaljson --sdk-version 2.2.100 -o demo2
```

> Note: to create global.json file inside an existing project directory, just omit the -o output directory flag.

enter the project directory and the generated global.json file to see the sdk version 2.2.100 specifiedL

```bash
Aregs-MacBook-Pro:asp3 aregsarkissian$ cd demo2
Aregs-MacBook-Pro:demo2 aregsarkissian$ cat global.json
{
  "sdk": {
    "version": "2.2.100"
  }
}
```

We can check the SDK version to see that we are now using the specific 2.2.100 version

Aregs-MacBook-Pro:demo2 aregsarkissian$ dotnet --version
2.2.100

now lets create the project the same webapi project inside the project directory

Aregs-MacBook-Pro:demo2 aregsarkissian$ dotnet new webapi

> Note: we didnt specify an output directory option this time since we are creating the project
inside the already created project directory when we created the global.json file. The
dotnet new command will use the current directory name demo2 as the project name


We can dump the content of the generated demo2.csproj project file to see we are targeting netcoreapp2.2

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


> recommendation: always set a global.json file for your projects to 
control the SDK version used by the CLI when installing multiple versions of the
framework. So for examp;e if you upgrade to the latest preview version of the 
framework, your project can still use the version specified in the global.json

> recomendation: always set create the global.json file before using the CLI to 
generate the project files so they are created with the version of the SDK specified
in global.json

## Extending the .Net Core CLI by installing .Net Core Global tools

Global tools are NuGet packages that add features to the dotnet cli and are unrelated to the SDK and CLI versioning

You can add features to the dotnet cli by intalling global tools as documented here:

https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools

you can find a list of available tools here:

https://github.com/natemcmaster/dotnet-tools

## Conclusion

In this post I detailed What the .Net Core SDK and Runtime versions are and how you can select the SDK and Runtime versions by using the CLI and the global.json and .csproj files.

Thanks for reading





