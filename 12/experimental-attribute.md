---
id: cs12-experimental-attribute
csharp: 12
tfm: net8.0
status: stable
needs-runtime: true (ExperimentalAttribute, .NET 8 or polyfill)
tags: [attribute, experimental, api-design, library]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#experimental-attribute
---
# `[Experimental]` attribute (C# 12)

> **Abstract.** `System.Diagnostics.CodeAnalysis.ExperimentalAttribute` marks types, members or whole assemblies
> as experimental. Every use produces a compiler **error** with the diagnostic ID you choose, which callers must
> suppress deliberately. It is the official way to ship unstable APIs without promising compatibility.

| | |
|---|---|
| Introduced | C# 12 · .NET 8 (`net8.0`) |
| Status | Stable |
| Kind | Compiler-recognized attribute |
| Runtime support needed | Attribute type from .NET 8 (can be declared internally for older targets) |
| Audience | Library and framework authors |
| Modernize category | n/a |

## 1. Motivation

Libraries had no standard way to say "this API may change or disappear". `[Obsolete]` means the opposite
(going away), `[EditorBrowsable(Never)]` only hides IntelliSense, and naming conventions (`Preview`, `Internal`)
are not enforced. Consumers silently depended on unstable APIs.

## 2. The feature explained

### 2.1 Syntax

```csharp
[Experimental("TMW0001")]
[Experimental("TMW0002", UrlFormat = "https://example.com/diag/{0}")]
```

### 2.2 Semantics and rules

1. Using an experimental symbol reports the given diagnostic ID as an **error** (suppressible like a warning).
2. Suppress with `#pragma warning disable ID`, `<NoWarn>ID</NoWarn>`, or `[SuppressMessage]`.
3. Applies to assemblies, modules, types, members, parameters... If the assembly is marked, all its types are experimental.
4. Uses **inside** the experimental API itself do not report.
5. `UrlFormat` lets IDEs link to documentation (`{0}` is replaced by the ID).
6. Later .NET versions may add properties to the attribute; check the API reference of your target framework.

## 3. Code examples

### 3.1 Minimal example

```csharp
[Experimental("TMW0001")]
public sealed class NewPathfinder { }

#pragma warning disable TMW0001
var pf = new NewPathfinder();
#pragma warning restore TMW0001
```

### 3.2 Extended examples

Project-wide opt-in in a consuming project:

```xml
<PropertyGroup>
  <NoWarn>$(NoWarn);TMW0001</NoWarn>
</PropertyGroup>
```

Experimental member on a stable type:

```csharp
public sealed class SaveService
{
    public void Save(SaveGame g) { ... }

    [Experimental("TMW0003", UrlFormat = "https://docs.example.com/diagnostics/{0}")]
    public ValueTask SaveIncrementalAsync(SaveGame g) { ... }
}
```

Polyfill for `netstandard2.0` libraries:

```csharp
#if !NET8_0_OR_GREATER
namespace System.Diagnostics.CodeAnalysis
{
    [AttributeUsage(AttributeTargets.All, Inherited = false)]
    internal sealed class ExperimentalAttribute : Attribute
    {
        public ExperimentalAttribute(string diagnosticId) => DiagnosticId = diagnosticId;
        public string DiagnosticId { get; }
        public string? UrlFormat { get; set; }
    }
}
#endif
```

## 4. Before this feature

```csharp
/// <summary>EXPERIMENTAL: may change without notice.</summary>
public sealed class NewPathfinder { }    // nothing enforces the warning
```

## 5. Under the hood

The attribute is stored in metadata. The compiler checks every symbol reference against it and reports the
diagnostic. No IL change at the call site.

## 6. Performance and allocation

None.

## 7. Pitfalls

- Reusing the same ID for unrelated APIs forces consumers to opt into all of them at once. Use one ID per feature area.
- Reflection-based use bypasses the diagnostic.

## 8. Concerns and limitations

- Promote APIs out of experimental status deliberately (remove the attribute, document in release notes).
- Diagnostic IDs must not collide with compiler or analyzer IDs; use a project-specific prefix.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Shipping preview APIs in a library or shared core package consumed by other repositories.

### 9.2 Avoid when
- Application-internal code with no external consumers (use feature flags or branches).

### 9.3 Good versus bad

```csharp
// Good
[Experimental("CORE0102", UrlFormat = "https://docs.example.com/diag/{0}")]

// Bad: generic ID that collides with tooling, no documentation
[Experimental("CS0001")]
```

## 10. Related features

- [interceptors-preview.md](interceptors-preview.md)
- [../13/overload-resolution-priority.md](../13/overload-resolution-priority.md): another API-evolution tool.

## 11. References

- [Experimental attribute](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#experimental-attribute)
- [Proposal: Experimental attribute](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/experimental-attribute)
