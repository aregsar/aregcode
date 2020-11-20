# Create a .NET 5 Console App

[Create a .NET 5 Console App](https://aregcode.com/blog/2020/create-a-dotnet-5-console-app)

## .NET 5 installation on Mac

1-Download and install .NET 5 SDK.

2-Check the version of the .NET CLI in installation directory

```bash
cd /usr/local/share/dotnet
./dotnet --version
```

Should display the SDK version
5.0.100

3-Create a symlink to the .NET CLI installation directory to the global executables directory on the system path

```bash
ln -s /usr/local/share/dotnet/dotnet /usr/local/bin/
```

4-Check that the `dotnet` command is available globally

```bash
dotnet --version
```

Should display the SDK version
5.0.100

## Creating a .NET project

1- Create the project directory including a global.json file for the SDK version

```bash
mkdir blogapp && cd blogapp
dotnet new globaljson --sdk-version 5.0.100
cat global.json
```

You should see a new global.json file generated withe the following content

```json
{
  "sdk": {
    "version": "5.0.100"
  }
}
```

Open the file and add the `rollForward` strategy for future versions of the .NET framework SDK

```json
{
  "sdk": {
    "version": "5.0.100",
    "rollForward": "minor"
  }
}
```

> See details for the different rollForward strategies at https://docs.microsoft.com/en-us/dotnet/core/tools/global-json

Check the project SDK version

```bash
dotnet --version
```

Should display the SDK version
5.0.100

2- Create a git ignore file

```bash
dotnet new .gitignore
cat .gitignore
```

> To see all available `dotnet new` templates run `dotnet new -l`

3- Create the console app

```bash
dotnet new console
cat blogapp.csproj
```

we can see the .NET 5 framework is the target

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net5.0</TargetFramework>
  </PropertyGroup>

</Project>
```

Open the project in VSCode

```bash
code .
```

The first time we open a .NET project in VSCode, it will automatically start downloading and installing the editor packages it needs for intellisense, building and debugging .NET applications

```bash
Installing C# dependencies...
Platform: darwin, x86_64

Downloading package 'OmniSharp for OSX' (49870 KB).................... Done!
Validating download...
Integrity Check succeeded.
Installing package 'OmniSharp for OSX'

Downloading package '.NET Core Debugger (macOS / x64)' (43578 KB).................... Done!
Validating download...
Integrity Check succeeded.
Installing package '.NET Core Debugger (macOS / x64)'

Downloading package 'Razor Language Server (macOS / x64)' (52935 KB).................... Done!
Installing package 'Razor Language Server (macOS / x64)'

Finished
```

The installed `Omnisharp` package will automatically run `dotnet build` which will generate the `bin` directory with project binaries.

The package will then pop up `required assets to build and run the application are missing, install?` notification.

Clicking the `yes` button makes will add the `.vscode` directory contains the `launch.json` launch settings file for running and debugging the application from the VSCode GUI and the `tasks.json` tasks settings file for running build tasks from VSCode GUI

/Users/aregsarkissian/.nuget/packages/
/Users/aregsarkissian/.nuget/NuGet/NuGet.Config

---

dotnet tool install --global dotnet-ef --version 5.0.0

export PATH="\$PATH:~/.dotnet/tools"

dotnet ef --version
