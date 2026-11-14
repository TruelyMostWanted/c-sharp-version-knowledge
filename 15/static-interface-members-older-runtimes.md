---
id: cs15-static-interface-members-older-runtimes
csharp: 15
tfm: net11.0 (compiler); targets older runtimes
status: stable
tags: [interface, static, netstandard, library]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
---
# Non-virtual static interface members on older runtimes (C# 15)

> **Abstract.** With the C# 15 compiler you can declare and call **non-virtual static** members (members that are
> neither `abstract` nor `virtual`, for example static helper methods) in interfaces even when the target runtime does not support default
> interface members, e.g. `netstandard2.0` or .NET Framework. Helpers can live next to the contract they belong to.

| | |
|---|---|
| Introduced | C# 15 compiler (.NET 11 SDK); applies to older target frameworks |
| Status | Stable |
| Kind | Compiler behaviour (runtime capability check relaxed) |
| Runtime support needed | No (that is the point) |
| Audience | Library authors multi-targeting older frameworks |
| Modernize category | n/a |

## 1. Motivation

C# 8 introduced static and default members in interfaces together, and the compiler required runtime support
(`RuntimeFeature.DefaultImplementationsOfInterfaces`) for **any** of them. Non-virtual static members do not actually need
new runtime behaviour, but libraries targeting `netstandard2.0` still had to put helpers into separate static classes
(`MeasurementHelpers.ToMeters`).

## 2. The feature explained

### 2.1 Syntax

```csharp
public interface IMeasurement
{
    public static double ToMeters(double centimeters) => centimeters / 100;
}
```

### 2.2 Semantics and rules

1. Allowed regardless of runtime: non-virtual static interface members (static helper methods are the typical case).
   Check the docs for the exact list of member kinds if you need static fields or properties on old targets.
2. Still requires runtime support: instance default implementations, `static abstract` and `static virtual` members, `protected` interface members.
3. Call them through the interface name (`IMeasurement.ToMeters(...)`).

## 3. Code examples

### 3.1 Minimal example

```csharp
public interface IMeasurement
{
    public static double ToMeters(double centimeters) => centimeters / 100;
}

double m = IMeasurement.ToMeters(250);
```

### 3.2 Extended examples

Helpers next to the contract (multi-targeted library):

```csharp
public interface ISaveFormat
{
    string Extension { get; }
    byte[] Serialize(SaveGame game);

    public static bool IsSupported(int version) => version is >= 1 and <= 3;
    public static string FileName(ISaveFormat format, string slot) => $"{slot}.{format.Extension}";
}

string file = ISaveFormat.FileName(format, "slot1");
```

## 4. Before this feature

```csharp
public interface IMeasurement { }

public static class Measurement
{
    public static double ToMeters(double centimeters) => centimeters / 100;
}
```

## 5. Under the hood

Interfaces have always been allowed to contain static methods in CLI metadata; older runtimes can call them. The C# 15
compiler simply stops requiring the default-interface-implementation runtime feature for this subset.

## 6. Performance and allocation

None; ordinary static calls.

## 7. Pitfalls

- Callers compiled with older C# compilers targeting old frameworks may not be able to call these members; consumers need a C# 15 compiler.
- Multi-targeting: test the library on every target framework, not only on the newest one.

## 8. Concerns and limitations

- Irrelevant for projects already on modern runtimes; they supported this since C# 8 / .NET Core 3.0.
- Godot: Godot 4 C# targets modern .NET; the feature matters only for shared libraries that also target `netstandard2.0`.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Writing a library for older targets and you want helpers next to the contract.

### 9.2 Avoid when
- The helper is unrelated to the interface's concept.

### 9.3 Good versus bad

```csharp
// Good
public interface IVersioned { int Version { get; } public static bool IsNewer(IVersioned a, IVersioned b) => a.Version > b.Version; }

// Bad: interface used as a dumping ground for utilities
public interface IUtils { public static string Slugify(string s) => ...; public static int Clamp(int v) => ...; }
```

## 10. Related features

- [../11/generic-math.md](../11/generic-math.md): `static abstract` members (need runtime support).

## 11. References

- [What's new in C# 15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15)
- [Interfaces: static members](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/interface)
