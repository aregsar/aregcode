# Create a .NET 5 Console App

[Create a .NET 5 Console App](https://aregcode.com/blog/2020/create-a-dotnet-5-console-app)

## .NET 5 installation on Mac

1-Download and install .NET 5 SDK from [https://dotnet.microsoft.com/download](https://dotnet.microsoft.com/download)

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
dotnet new gitignore
cat gitignore
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

The first time we open a .NET project in VSCode, it will automatically start downloading and installing the editor extension packages it needs for IntelliSense, building and debugging .NET applications. You can find the code for OmniSharp at https://github.com/OmniSharp/omnisharp-vscode

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

The `Omnisharp` package will automatically run `dotnet build` which will generate the `bin` directory that contains project binaries. `dotnet build` also runs `dotnet restore` which will try to install any packages in the `csproj` file if any.

The package will then pop up `Required assets to build and debug are missing from 'blogapp'. Add them?` notification. This happens every time you open a new project in VSCode where the project does not contain the required `.vscode` directory files.

Clicking the `yes` button will add the `.vscode` directory which contains the `launch.json` launch settings file for running and debugging the application from the VSCode GUI and the `tasks.json` tasks settings file for running build tasks from VSCode GUI.

## Running the program from the command line

```bash
dotnet run
```

> dotnet run will run dotnet build which will in turn run dotnet restore, so there is no need to run each command individually

The response should be:

```bash
Hello World!
```

## Modifying the code

Just as an example I will show you how to use the C# 9 top level statements to eliminate the boilerplate in the program.cs file

OPen program.cs in your editor and you should see

```cs
using System;

namespace blogapp
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
        }
    }
}
```

We can remove much of this boilerplate using the C# 9 top level statements feature

```cs
using System;

Console.WriteLine("Hello World!");
```

Run the program again

```bash
dotnet run
```

The response should still be:

```bash
Hello World!
```

Now lets refactor to add top level functions

```cs
using System;

Print("Hello World!");

void Print(string outputText) =>
    Console.WriteLine(outputText);
```

Here I have added a Print function which is called with the Print statement above it.

Run the program and the see the output is unchanged.

You can also add top level classes to the file and then instantiate and call class methods in the top level statements.

How about adding command line arguments to our program to change the output text or print multiple text lines?

```cs
using System;

Print("Hello World!");

foreach (var arg in args)
    Print(arg);

void Print(string outputText) =>
    Console.WriteLine(outputText);
```

Now we can pass zero or more arguments to this program. Try these out

```bash
#no arguments
dotnet run
#one argument
dotnet run -- one
#two arguments
dotnet run -- one two
```

Finally lets also make this interactive to print more lines

```cs
using System;

Print("Hello World!");

foreach (var arg in args)
    Print(arg);

Console.WriteLine("Type text to print or hit enter to exit");

while (true)
{
    var lineText = Console.ReadLine();

    if (lineText.Length == 0)
        break;

    Print(lineText);
}

void Print(string outputText) =>
    Console.WriteLine(outputText);
```

Run the code again.

After it prints its output it waits for user input.

Typing any text and hitting the enter key, echos the text to the output.

Hitting enter key alone will exit the program.

## Installing Entity Framework 5

One Other thing we may want to do is install is the Entity Framework command line tooling.

This will give us an opportunity to see how dotnet tools can be installed to add more command line capabilities.

Install the EF Core tool

```bash
dotnet tool install --global dotnet-ef --version 5.0.0
```

In order to be able to access the tools we need to add the tools directory to our system path

Add the following to your shell profile file

```bash
export PATH="$PATH:~/.dotnet/tools"
```

Check the EF Core version

```bash
dotnet ef --version
```

You should see the response

```bash
Entity Framework Core .NET Command-line Tools
5.0.0
```

If you check `~/.dotnet/tools/` directory you should see the `dotnet-ef` binary file.

To uninstall a global tool we can run the `dotnet tool uninstall` command with the same tool name.

## Installing other useful tools

Other tool can be added using the same dotnet tool install command

Here is a tool that is a REPL for making HTTP calls to web apis

```bash
dotnet tool install -g Microsoft.dotnet-httprepl
```

Run this command directly without the dotnet command

```bash
httprepl
```

## One more thing

I like to create a new Github repository anytime I create a new project so I can keep track of my changes from day one.

I prefer the Github command line tool to create my remote repository and then use standard Git command line to create my local tracking repository.

You can install Github command line tool using Homebrew

```bash
brew install gh
```

Macs come with a preinstalled version of the Git client but I like to install and manage mine through brew

```bash
brew install git
```

Create the remote repository on Github

> You need to setup ssh keys for connecting to Github prior to using the tool.

```bash
gh
```

Create the repository

```bash
cd blogapp
git init
git add remote "myremoterepo"
git add -A
git commit -am "created new project"
git push -m origin master
```

And with that we are done!
