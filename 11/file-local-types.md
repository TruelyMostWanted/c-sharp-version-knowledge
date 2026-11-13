---
id: cs11-file-local-types
csharp: 11
tfm: net7.0
status: stable
tags: [file, access-modifier, source-generator, encapsulation]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/file
modernizes: private nested helper types or internal helpers with name clashes
---
# File-local types (C# 11)

The `file` modifier makes a type visible only inside the source file that declares it.

```csharp
file static class SaveFormatConstants
{
    public const int Version = 3;
}

file sealed class JsonContext { }
```

## Use when
- Helpers used by one file only; source generators emitting helper types without name clashes.

## Rules
- Applies to top-level types (class, struct, record, interface, enum, delegate), not to members.
- A file-local type cannot appear in the signature of a non-file-local member.
