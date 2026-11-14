---
id: cs14-file-based-app-directives
csharp: 14
tfm: .NET 10 SDK (and later)
status: stable
tags: [file-based-apps, dotnet-run, scripts, tooling]
source:
  - https://learn.microsoft.com/en-us/dotnet/core/sdk/file-based-apps
  - https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/preprocessor-directives
---
# File-based apps and `#:` directives (.NET 10 SDK / C# 14)

> **Abstract.** A single `.cs` file can be built and run with `dotnet run app.cs`, without a `.csproj`.
> Directives starting with `#:` at the top of the file add NuGet packages, project references, MSBuild properties or a
> different SDK. The C# compiler ignores them; the .NET SDK reads them and builds a virtual project. Ideal for repo tools
> and scripts; convert to a real project when the tool grows.

| | |
|---|---|
| Introduced | .NET 10 SDK, C# 14 (`#:` is part of the language's preprocessor grammar) |
| Status | Stable |
| Kind | Tooling + preprocessor directive |
| Runtime support needed | .NET 10 SDK on the machine running the script |
| Audience | Tooling, scripts, samples |
| Modernize category | CASE |

## 1. Motivation

Small helper programs (asset converters, release scripts, data generators) needed a full project: folder, `.csproj`,
`Program.cs`, solution entry. Many teams used PowerShell, Python or `dotnet-script` instead, losing type safety and
shared code with the main project.

## 2. The feature explained

### 2.1 Syntax

```csharp
#!/usr/bin/env dotnet                        // optional shebang (Unix), first line only
#:package Serilog@3.1.1                      // NuGet package (version, or @* for latest)
#:project ../SharedLibrary/SharedLibrary.csproj
#:property TargetFramework=net10.0
#:sdk Microsoft.NET.Sdk.Web
#:include helpers.cs                         // .NET 11 SDK / 10.0.300+: additional source files, globs allowed
```

### 2.2 Semantics and rules

1. Directives must appear at the top of the file, before any code (after an optional shebang).
2. `#:package Name@Version`: version required unless central package management supplies it; `@*` takes the latest.
3. `#:project path`: references another project (relative path to a `.csproj` or its folder).
4. `#:property Name=Value`: sets an MSBuild property of the virtual project.
5. `#:sdk Name[@Version]`: chooses the project SDK (default `Microsoft.NET.Sdk`).
6. `#:include`: newer SDKs only; included files may not contain top-level statements; globs disable build caching.
7. The file uses top-level statements; implicit usings and nullable are enabled by default in the virtual project.
8. Build outputs are cached in a temporary artifacts folder; rebuilt when the file or its inputs change.
9. `dotnet project convert app.cs` creates a regular project from the file.

## 3. Code examples

### 3.1 Minimal example

```csharp
// hello.cs
Console.WriteLine($"Hello from {Environment.OSVersion}");
```

```
dotnet run hello.cs
```

### 3.2 Extended examples

CSV to JSON asset tool:

```csharp
#!/usr/bin/env dotnet
#:package CsvHelper@33.0.1
#:property Nullable=enable

using System.Globalization;
using System.Text.Json;
using CsvHelper;

if (args is not [var input, var output])
{
    Console.Error.WriteLine("usage: dotnet run csv2json.cs -- <input.csv> <output.json>");
    return 1;
}

using var reader = new StreamReader(input);
using var csv = new CsvReader(reader, CultureInfo.InvariantCulture);
var rows = csv.GetRecords<dynamic>().ToList();
File.WriteAllText(output, JsonSerializer.Serialize(rows, new JsonSerializerOptions { WriteIndented = true }));
Console.WriteLine($"Wrote {rows.Count} rows to {output}");
return 0;
```

Reusing game logic from a project:

```csharp
#:project ../src/Game.Core/Game.Core.csproj

using Game.Core.Balance;

foreach (var unit in BalanceTable.LoadDefaults())
    Console.WriteLine($"{unit.Id,-12} DPS {unit.Dps:F1}");
```

A tiny web endpoint:

```csharp
#:sdk Microsoft.NET.Sdk.Web

var app = WebApplication.Create(args);
app.MapGet("/health", () => "ok");
app.Run();
```

Running:

```
dotnet run tool.cs -- arg1 arg2
dotnet tool.cs arg1 arg2          # shorthand
chmod +x tool.cs && ./tool.cs     # Unix with shebang
dotnet project convert tool.cs    # turn it into a normal project
```

## 4. Before this feature

```
mkdir Csv2Json && cd Csv2Json
dotnet new console
dotnet add package CsvHelper
# edit Program.cs, then
dotnet run -- input.csv output.json
```

Or third-party `dotnet-script` (`.csx` files with `#r "nuget: ..."`).

## 5. Under the hood

The SDK parses the `#:` lines, generates an in-memory project file with the corresponding `PackageReference`,
`ProjectReference`, properties and SDK, then runs a normal MSBuild restore/build into a cache directory and executes the
result. The compiler treats `#:` lines as ignored preprocessor directives (and `#!` on the first line as a comment).

## 6. Performance and allocation

- First run includes restore and build (seconds). Later runs reuse the cached build if nothing changed.
- Runtime performance equals a normal console app.

## 7. Pitfalls

- If a project file exists in the current directory, `dotnet run file.cs` may run that project instead; use `dotnet run --file file.cs`.
- In normal project builds, `#:` directives produce warnings; keep them in standalone tool files (exclude tool folders from the main project's `Compile` items).
- `#:include` and some newer directives need newer SDKs; pin the SDK with `global.json` for team tools.
- Unpinned versions (`@*`) make scripts non-reproducible.

## 8. Concerns and limitations

- Security: `#:package` downloads code on first run; review packages like any dependency.
- Not meant for large programs; convert once the tool needs multiple files, tests or CI packaging.
- Godot: useful for pipeline tools (import preprocessing, localization CSV checks); the scripts run outside the engine and cannot use Godot APIs.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Repository tools, one-off migrations, data generators, samples and reproductions.

### 9.2 Avoid when
- Production services or anything that needs tests, multiple files and versioned releases.

### 9.3 Good versus bad

```csharp
// Good: pinned versions, clear usage message, exit codes
#:package Spectre.Console@0.49.1

// Bad: floating versions in a team tool
#:package Spectre.Console@*
```

## 10. Related features

- [../10/global-using.md](../10/global-using.md): implicit usings apply.
- [../11/list-patterns.md](../11/list-patterns.md): argument parsing with `args is [...]`.

## 11. References

- [File-based apps](https://learn.microsoft.com/en-us/dotnet/core/sdk/file-based-apps)
- [Preprocessor directives](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/preprocessor-directives)
