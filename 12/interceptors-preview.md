---
id: cs12-interceptors-preview
csharp: 12
tfm: net8.0
status: preview / experimental
tags: [interceptors, source-generator]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12
---
# Interceptors (C# 12, experimental)

An interceptor substitutes a call to an interceptable method with a call to itself at compile time,
by declaring the source locations it intercepts. Intended to be produced by **source generators**
(ASP.NET Core uses it for Minimal APIs).

```xml
<InterceptorsPreviewNamespaces>$(InterceptorsPreviewNamespaces);MyLibrary.Generated</InterceptorsPreviewNamespaces>
```

## Rules for agents
- Do **not** hand-write interceptors in application code.
- Only touch this when maintaining a source generator, and check the current docs first:
  the opt-in property and attribute shape have changed across SDK versions.
