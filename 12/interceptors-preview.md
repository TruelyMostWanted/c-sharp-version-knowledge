---
id: cs12-interceptors-preview
csharp: 12
tfm: net8.0
status: preview / experimental
tags: [interceptors, source-generator]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#interceptors
---
# Interceptors (C# 12, experimental)

> **Abstract.** An interceptor is a method that, at compile time, **replaces a specific call site** of another
> method. The interceptor declares which locations it intercepts via an attribute. The feature exists for
> **source generators** (ASP.NET Core Minimal APIs, configuration binding, AOT-friendly serialization) to swap
> reflection-based calls for generated code. It is never meant for hand-written application code.

| | |
|---|---|
| Introduced | C# 12 · .NET 8 (`net8.0`), experimental |
| Status | Experimental / preview; opt-in shape changed across SDK versions |
| Kind | Compiler behaviour |
| Runtime support needed | No |
| Audience | Source-generator authors only |
| Modernize category | NEVER-AUTO |

## 1. Motivation

Reflection-heavy APIs (`app.MapGet("/x", handler)`, `config.Get<T>()`) are slow to start and break under Native AOT
trimming. Source generators can produce specialized code, but they cannot change user code to call it. Interceptors let
a generator redirect existing calls to generated methods while the user's source stays unchanged.

## 2. The feature explained

### 2.1 Syntax

Opt-in in the project (property name depends on SDK version):

```xml
<InterceptorsNamespaces>$(InterceptorsNamespaces);MyLibrary.Generated</InterceptorsNamespaces>
<!-- .NET 8 SDK used <InterceptorsPreviewNamespaces> -->
```

Generated code (shape as of .NET 9/10 SDKs; earlier versions used file path + line + column):

```csharp
namespace MyLibrary.Generated
{
    file static class Interceptors
    {
        [System.Runtime.CompilerServices.InterceptsLocation(version: 1, data: "...opaque location data...")]
        public static void InterceptedLog(this Logger logger, string message) { /* generated */ }
    }
}
```

### 2.2 Semantics and rules

1. The interceptor's signature must be compatible with the intercepted method (same parameters, `this` for instance methods).
2. Only call sites listed by the attribute are intercepted; the original method still exists.
3. The interceptor's namespace must be opted in via the MSBuild property.
4. Location data is produced by the generator API (`SemanticModel.GetInterceptableLocation`) and is opaque.
5. Only generators should emit `[InterceptsLocation]`; the format is not stable for humans.

## 3. Code examples

### 3.1 Minimal example

User code:

```csharp
logger.Log("Started");     // generator decides to intercept this call
```

Generator output:

```csharp
file static class GeneratedInterceptors
{
    [InterceptsLocation(1, "...")]
    public static void Log_Intercepted(this Logger logger, string message)
        => logger.WriteFast(message);
}
```

### 3.2 Extended examples

Real-world consumers (you benefit without writing interceptors):

- ASP.NET Core Request Delegate Generator (`<EnableRequestDelegateGenerator>true</EnableRequestDelegateGenerator>`, on by default for AOT).
- Configuration binding source generator (`<EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>`).

## 4. Before this feature

Generators could only **add** code. Users had to call generated APIs explicitly (`app.MapGetGenerated(...)`) or keep reflection.

## 5. Under the hood

During binding, the compiler checks each invocation against the interceptor table and emits a `call` to the interceptor
method instead of the original. There is no runtime indirection.

## 6. Performance and allocation

Depends entirely on the generated code; usually faster startup and AOT-safe.

## 7. Pitfalls

- Hand-written interceptors break on any edit that moves the call site.
- Debugging: stepping into a call lands in generated code, which can surprise developers.

## 8. Concerns and limitations

- The opt-in property and attribute shape **changed** across SDK versions; check the current docs before touching generator code.
- Readability: what a call does is no longer visible at the call site.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Maintaining a source generator that must replace reflection with generated code at existing call sites.

### 9.2 Avoid when
- Any application code. Do **not** hand-write interceptors.

### 9.3 Good versus bad

```csharp
// Good: generator emits interceptors via GetInterceptableLocation
// Bad: developer pastes [InterceptsLocation("Program.cs", 12, 5)] by hand
```

## 10. Related features

- [../11/file-local-types.md](../11/file-local-types.md): generators emit interceptors as `file` classes.
- [../10/line-pragma-format.md](../10/line-pragma-format.md)

## 11. References

- [What's new in C# 12: interceptors](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#interceptors)
- [Interceptors feature specification (Roslyn)](https://github.com/dotnet/roslyn/blob/main/docs/features/interceptors.md)
