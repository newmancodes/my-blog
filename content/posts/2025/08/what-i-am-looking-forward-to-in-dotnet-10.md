+++
date = '2025-08-14T09:15:00+01:00'
draft = false
title = 'What I am Looking Forward to in .NET 10'
tags = ['programming', '.net']
+++

Another November is almost upon us, and that means we get a new release of .NET! This year is an odd year, that is to say `DateTime.UtcNow.Year % 2 == 1`, so we know it's a Long Term Support (LTS) release. That means three years of support giving organisations more confidence in shifting to it. On the flip side, we have one year to get off .NET 8 (the previous LTS release) and only six months to get off .NET 9 (the previous Standard Term Support or STS release).

Along with some good improvements in ASP.NET Core (do we really need to keep saying Core these days?) like some updated security samples and a declarative persistence option for Blazor. There is one feature in particular which I think is going to be useful and further evolves .NET into an "easy to get up and running" platform. That feature is the ability to run a C# file directly without needing the project ceremony - `dotnet run app.cs`!

Until now, if I want to create a new utility or example app in C# I need to have at least two files, a .csproj file, and a (typically) Program.cs file. You can see this setup today by executing `dotnet new console -n SimpleExample`. Once this has executed, you'll have a new directory named SimpleExample which contains the following...

```bash
.
├── obj
│   ├── project.assets.json
│   ├── project.nuget.cache
│   ├── SimpleExample.csproj.nuget.dgspec.json
│   ├── SimpleExample.csproj.nuget.g.props
│   └── SimpleExample.csproj.nuget.g.targets
├── Program.cs
└── SimpleExample.csproj
```

The project file contains the following content:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

</Project>
```

and the Program.cs has the following boilerplate:

```csharp
// See https://aka.ms/new-console-template for more information
Console.WriteLine("Hello, World!");
```

Executing `dotnet run` displays the expected "Hello, World!" message but, for people just starting out with .NET, there is a lot here. Just as the .NET team have made strides to simplify the `Program.cs` file, this new capability removes more of this ceremony.

Let's take a look at the new simpler approach. With .NET 10 installed, currently available in [preview 7](https://dotnet.microsoft.com/en-us/download/dotnet/10.0), create a new empty folder. In that folder add a file named `app.cs`, containing the following content.

```csharp
Console.WriteLine("Hello, World!");
```

You should have this content in your folder.

```bash
.
└── app.cs
```

Now, execute `dotnet run app.cs`.

```bash
> dotnet run app.cs
Hello, World!
```

That's it, that's all you need to get a basic console application up and running. But, this feels pretty limiting. What if I wanted to make use of NuGet packages as I would in a normal application?

You can import NuGet packages by adding the `#:package` directive to your `app.cs` file.

```csharp
#:package Spectre.Console.Cli@0.50.0

using Spectre.Console;

var calendar = new Calendar(2025, 8);
AnsiConsole.Write(calendar);
```

Now when we execute `dotnet run app.cs` we see a rendering of the month of August 2025.

```bash
> dotnet run app.cs
                2025 August
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ Sun │ Mon │ Tue │ Wed │ Thu │ Fri │ Sat │
├─────┼─────┼─────┼─────┼─────┼─────┼─────┤
│     │     │     │     │     │ 1   │ 2   │
│ 3   │ 4   │ 5   │ 6   │ 7   │ 8   │ 9   │
│ 10  │ 11  │ 12  │ 13  │ 14  │ 15  │ 16  │
│ 17  │ 18  │ 19  │ 20  │ 21  │ 22  │ 23  │
│ 24  │ 25  │ 26  │ 27  │ 28  │ 29  │ 30  │
│ 31  │     │     │     │     │     │     │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┘
```

By default, this method of executing C# code sets the SDK property to the Microsoft.NET.Sdk. If you want to create a web application, you need to change this to Microsoft.NET.Sdk.Web. This is also achieved by applying a directive. In the same folder create a new file `api.cs` and add the following content.

```csharp
#:sdk Microsoft.NET.Sdk.Web

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

Now, execute `dotnet run api.cs`.

```bash
> dotnet run api.cs
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /Users/steve/Projects
```

If you navigate to http://localhost:5000, you'll see our "Hello, World!" message again. All we had to do was specify the appropriate Sdk via the `#:sdk Microsoft.NET.Sdk.Web` instruction.

I think this is going to be really useful for creating small utilities, demos, and learning material. Now that we've removed the extra ceremony around the code, it is easier to focus on what we're trying to get across.

What happens when the single file is no longer enough and I need to fall back to a more full-fat .NET project? No problem, you can do this too via the `dotnet convert project api.cs -o Api` command. This will create an `Api` folder containing the `api.cs` file and the correct `api.csproj`. Note: the csproj file doesn't have the Api casing. The project file appears to inherit the name from the source file being converted.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <PublishAot>true</PublishAot>
  </PropertyGroup>

</Project>
```

So this feels like a really nice way to start off with a simple single file and know that you aren't prevented from "upgrading" to a full .NET project. This will be my approach to [Advent of Code](https://adventofcode.com/) this year! I just wish I could add tests too (like I can with Rust)...

```csharp
#:package XUnit.v3@3.0.0

using Xunit;

Console.Write("Dividend: ");
var dividend = decimal.Parse(Console.ReadLine());
Console.Write("Divisor: ");
var divisor = decimal.Parse(Console.ReadLine());

var division = new Division(dividend, divisor);
var quotient = division.GetQuotient();

Console.WriteLine($"The result of {dividend} divided by {divisor} is: {quotient}");

public record Division(decimal Dividend, decimal Divisor)
    {
        public decimal GetQuotient()
        {
            return Dividend / Divisor;
        }
    }

public class DivisionTests
{
    [Fact]
    public void Ten_divided_by_two_is_equal_to_five()
    {
        // Arrange
        var division = new Division(10, 2);

        // Act
        var result = division.GetQuotient();

        // Assert
        Assert.Equal(5, result);
    }
}
```

Sadly this doesn't appear to be fully supported. At least, not yet. To keep an eye on what's coming with .NET 10, make sure to refer to [learn.microsoft.com](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-10/overview) and for more information on this feature take a look at [the announcement](https://devblogs.microsoft.com/dotnet/announcing-dotnet-run-app/#what-is-dotnet-run-app.cs).
