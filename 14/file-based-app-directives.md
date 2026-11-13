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

A single `.cs` file can be run without a `.csproj`. `#:` directives at the top of the file configure
the build. The C# compiler ignores them; the SDK build parses them.

## Use when
- Small tools and scripts in a repo: asset pipeline, CSV to resources, export or release helpers.

## Directives
```csharp
#!/usr/bin/env dotnet                        // Unix: run directly after chmod +x
#:package Serilog@3.1.1                      // NuGet package (version or @*; no version only with central package management)
#:project ../SharedLibrary/SharedLibrary.csproj
#:property TargetFramework=net10.0
#:sdk Microsoft.NET.Sdk.Web
#:include helpers.cs                         // .NET 11 / SDK 10.0.300+: more source files, globs allowed
```

## Run
```
dotnet run tool.cs -- arg1 arg2
dotnet tool.cs
dotnet project convert tool.cs     # turn it into a normal project
```

## Rules and pitfalls
- `#:include`d `.cs` files cannot contain top-level statements.
- Glob patterns in `#:include` disable build caching.
- If a project file exists in the current directory, `dotnet run file.cs` (without `--file`) runs that project instead.
- In normal project builds these directives produce warnings; keep them in standalone tool files.
