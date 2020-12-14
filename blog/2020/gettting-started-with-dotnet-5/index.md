# Create a .NET 5 Console App

December 10

[Create a .NET 5 Console App](https://aregcode.com/blog/2020/create-a-dotnet-5-console-app)

December 10, 2020

In this post I will show how to create a basic .NET 5 console application project using the `dotnet` command line interface and demonstrate a few new C# 9 features in the process.

## .NET 5 installation on Mac

Download and install .NET 5 SDK from [https://dotnet.microsoft.com/download](https://dotnet.microsoft.com/download)

Check the version of the `dotnet` executable in installation directory:

```bash
cd /usr/local/share/dotnet
./dotnet --version
```

Should see the displayed .NET SDK version:
5.0.100

Create a symlink to the `dotnet` executable in the binaries directory that is on the system path:

```bash
ln -s /usr/local/share/dotnet/dotnet /usr/local/bin/
```

Verify the `dotnet` command is available globally:

```bash
dotnet --version
```

Should see the displayed the SDK version:

5.0.100

## Creating a .NET project

Create the project directory and add a `global.json` file to specify the SDK version:

```bash
mkdir ConsoleApp && cd ConsoleApp
dotnet new globaljson --sdk-version 5.0.100
cat global.json
```

You should see a new global.json file generated withe the following content:

```json
{
  "sdk": {
    "version": "5.0.100"
  }
}
```

Open the file and add the `rollForward` strategy for rolling forward with future versions of the .NET framework SDK:

```json
{
  "sdk": {
    "version": "5.0.100",
    "rollForward": "minor"
  }
}
```

I chose the strategy to roll forward only with minor versions.

> See details for the different rollForward strategies at https://docs.microsoft.com/en-us/dotnet/core/tools/global-json

Verify the project SDK version again:

```bash
dotnet --version
```

Should still display the SDK version unless you have upgraded to a newer minor version of the SDK.

5.0.100

Create a .NET `.gitignore` file:

```bash
dotnet new gitignore
cat .gitignore
```

Create the console app and print out the created project file:

```bash
dotnet new console
cat ConsoleApp.csproj
```

We can see the .NET 5 framework is the target:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net5.0</TargetFramework>
  </PropertyGroup>

</Project>
```

> To see all available `dotnet new` command project type templates, run `dotnet new -l`.

Open the project in VSCode:

```bash
code .
```

> I have configured the my VSCode executable to launch from the command line

The first time we open a .NET project in VSCode, it will automatically start downloading and installing the editor extension packages it needs for IntelliSense, building and debugging .NET applications.

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

The installed `Omnisharp` package will automatically run `dotnet build` command which will generate the `bin` directory that contains the project binaries.

The `dotnet build` command also runs the `dotnet restore` command which will try to install any packages in the `csproj` file if it finds any.

The package will then pop up `Required assets to build and debug are missing from 'ConsoleApp'. Add them?` notification.

This happens every time you open a new project in VSCode where the project does not contain the `.vscode` directory and files.

Clicking the `yes` button will add the `.vscode` directory which contains the `launch.json` launch settings file for running and debugging the application from the VSCode GUI.

The `.vscode` directory will also contain the `tasks.json` tasks settings file for running build tasks from VSCode GUI.

> You can find the code for the OmniSharp at [https://github.com/OmniSharp/omnisharp-vscode](https://github.com/OmniSharp/omnisharp-vscode)

## Running the program from the command line

We can run the code as is by using the following command:

```bash
dotnet run
```

> the `dotnet run` command will run the `dotnet build` command which will in turn run the `dotnet restore` command, so there is no need to run each command individually.

The response should be:

```bash
Hello World!
```

## Modifying the code

The first thing I will show you is how to use the C# 9 top level statements to eliminate the boilerplate in the `program.cs` file.

Open `program.cs` in your editor:

```cs
using System;

namespace ConsoleApp
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

We can remove much of this boilerplate using the C# 9 top level statements feature:

```cs
using System;

Console.WriteLine("Hello World!");
```

Here I have removed all the boilerplate code.

Run the program again

```bash
dotnet run
```

The response should still be:

```bash
Hello World!
```

Now lets refactor to add top level functions:

```cs
using System;

Print("Hello World!");

void Print(string outputText) =>
    Console.WriteLine(outputText);
```

Here I have added a Print function which is called from the Print statement above it.

Run the program and the see that the output is unchanged.

You can also add top level classes to the file and then instantiate the class and call class methods in the top level statements.

How about adding command line arguments to our program to change the output text or print multiple text lines?

```cs
using System;

Print("Hello World!");

foreach (var arg in args)
    Print(arg);

void Print(string outputText) =>
    Console.WriteLine(outputText);
```

Now we can pass zero or more arguments to this program.

Try these out:

```bash
#no arguments
dotnet run
#one argument
dotnet run -- one
#two arguments
dotnet run -- one two
```

It should print each argument of a separate line then exit.

Finally lets also make this interactive to print more lines:

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

Run the code again:

```bash
dotnet run
```

After it prints its output, it waits for user input.

Typing any text and hitting the enter key, echos the text to the output.

Hitting enter key alone will exit the program.

## Debugging using VSCode

To run this program using the VSCode debugger, use the `cmd+shift+d` keyboard shortcut to open the debugger panel and start the debugger.

Before you start the debugger, replace the `"console": "internalConsole"` setting in the `"name": ".NET Core Launch (console)"` configuration section in the `.vscode/launch.json` file to `"console": "integratedTerminal"`.

This will allow the `Console.ReadLine()` line in the program to except input from the VSCode integrated terminal.

> Make sure you open the VSCode terminal tab to see the output of the program and type input. By default when you run the debugger the debug tab is in focus showing the debug output of the program.

Another option is to use `"console": "externalTerminal"` setting instead of the `"console": "internalConsole"` setting. This will launch an external MacOS terminal when you start debugging, instead of using the integrated terminal, just as when you ran the program in the terminal before.

## Installing Entity Framework 5

One Other thing we may want to do is install the `Entity Framework` command line tooling.

This will give us an opportunity to see how the `dotnet tools` command line extensions can be installed to add more command line capabilities.

Install the EF Core tool:

```bash
dotnet tool install --global dotnet-ef --version 5.0.0
```

In order to be able to access the tools we need to add the tools directory to our system path.

Add the following to your shell profile file:

```bash
export PATH="$PATH:~/.dotnet/tools"
```

Verify the installed EF Core version:

```bash
dotnet ef --version
```

You should see the response

```bash
Entity Framework Core .NET Command-line Tools
5.0.0
```

If you check the `~/.dotnet/tools/` directory you should see the `dotnet-ef` binary file.

To see the available commands run:

```bash
dotnet ef --help
```

To uninstall a global tool we can run the `dotnet tool uninstall` command with the same tool name.

```bash
dotnet tool uninstall ef
```

## Installing other useful tools

Other tools can be added using the same `dotnet tool install` command.

Here is a tool that is a REPL for making HTTP calls to web APIs:

```bash
dotnet tool install -g Microsoft.dotnet-httprepl
```

You run this command directly without prefixing it with the `dotnet` command:

```bash
httprepl
```

> You can use Ctrl-c to exit the tool

We can also list all installed tools:

```bash
dotnet tool list --global
```

The output lists the two tools we have installed so far.

```bash
Package Id                     Version      Commands
-----------------------------------------------------
dotnet-ef                      5.0.0        dotnet-ef
microsoft.dotnet-httprepl      5.0.0        httprepl
```

## Building a self contained executable

Create a single file self contained executable, from the project directory run the `dotnet publish` command:

```bash
dotnet publish -c Release -r osx-x64 -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true
```

The -c flag specifies a Release version of the app and the -r flag specifies the target platform.

The command will produce the following files relative to your project directory.

```bash
#binary executable
bin/Release/net5.0/osx-x64/publish/ConsoleApp
#symbol file
bin/Release/net5.0/osx-x64/publish/ConsoleApp.pdb
```

We can run the program directly from the project directory by executing:

```bash
bin/Release/net5.0/osx-x64/publish/ConsoleApp
```

You should see the app running.

> Note we could have used `IncludeAllContentForSelfExtract` instead of `IncludeNativeLibrariesForSelfExtract` to make a the resulting executable extract files to disk as was done in .NET 3.1. However in .NET 5 we have the `IncludeNativeLibrariesForSelfExtract` option which will only extract some native libraries to the disk and load the rest of the program directly into memory. Consult the documentation as there are some issues with some of the single file executable options when your program uses reflection.

## One More Thing

I like to create a new Github repository anytime I create a new project so I can keep track of my changes from day one.

I prefer the Github command line client to create my remote repository and then using the standard Git command line client to create my local tracking repository.

You can install Github command line tool using the Homebrew MacOS package manager:

```bash
brew install gh
gh --version
```

The manual for `gh` is located at [https://cli.github.com/manual](https://cli.github.com/manual)

Macs come with a preinstalled version of the Git client. If you don't want to use your native MacOS git client you can install git using brew as well:

```bash
brew install git
git --version
```

By default the `gh` client and the remote Github `git` repository to communicate using the http protocol.

I like to configure the `gh` client and the Github repository to use the SSH protocol:

```bash
#set local git client protocol
gh config set git_protocol ssh
#set remote git repo protocol
gh config set -h github.com git_protocol ssh
#verify local setting
gh config get git_protocol
#verify remote setting
gh config get -h github.com git_protocol
```

As configured above the `gh config get` commands should return `ssh`. You can run them anytime to change or check the git protocol setting.

Now we are ready to use the `gh` client. But before using it we need to first authenticate with Github:

> Note: In order to be able to use the SSH protocol with Github, we have to generate and upload SSH keys to Github. Instructions for generating a local SSH key pair and uploading the public key to Github.com can be found in the resources section at the end of this post.

```bash
#login to github using the web browser
gh auth login -h github.com --web
```

This command will print a verification code that you need to copy then paste in the browser window that the command launches to authenticate with Github.

> Tip: After you copy the code put your cursor in the leftmost edit box on the web page and paste to fill in the entire code in one shot across all the edit boxes in the web page.

### Create the remote repository on Github

Now that the client is authenticated, Create the local and remote repository from within the project directory:

```bash
#need to first initialize a local git repository before using the gh client 'repo create' command so the command can add the remote to the local repository
git init
# create a public repo (use --private instead to make the repo private)
# the command will add the remote repo to our remotes so we wont need to add it manually like we normally need to do when using the git client
gh repo create ConsoleApp -d "First .NET 5 App" --public
```

The command will ask to create the repo named ConsoleApp in the current directory.

Type `y` and hit enter to continue to create the remote repo and add it as a remote to your local git repo, showing the output below:

```bash
? This will create 'ConsoleApp' in your current directory. Continue?  Yes
✓ Created repository aregsar/ConsoleApp on GitHub
✓ Added remote git@github.com:aregsar/ConsoleApp.git
```

Now that the local and remote repos are created, we can make our first commit and push, creating a tracking branch in the process.

```bash
# track all files in the directory
git add -A
# the first commit
git commit -m "first commit"
# push changes to remote
# the -u flag creates a local tracking branch to our upstream remote branch
git push -u origin master
git status
```

Finally, we can navigate to our repo from the command line:

```bash
# inspect the new repository in the web browser
gh repo view --web
```

When you are done creating the repository, you can logout:

```bash
gh auth logout
```

You can still use the standard `git` commands to commit and push changes after logging out with `gh`.

And with that we are done!

## Resources

https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/connecting-to-github-with-ssh

https://www.freecodecamp.org/news/the-ultimate-guide-to-ssh-setting-up-ssh-keys/

https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys

[https://cli.github.com/manual](https://cli.github.com/manual)
