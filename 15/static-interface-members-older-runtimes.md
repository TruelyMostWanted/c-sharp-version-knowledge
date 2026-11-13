---
id: cs15-static-interface-members-older-runtimes
csharp: 15
tfm: net11.0 (compiler); targets older runtimes
status: stable
tags: [interface, static, netstandard, library]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
---
# Non-virtual static interface members on older runtimes (C# 15)

You can declare and use **non-virtual static** interface members even when the target runtime does not
support default interface implementations (e.g. `netstandard2.0`, .NET Framework).

## Use when
- Writing a library for older targets and you want helpers next to the contract.

## Syntax
```csharp
public interface IMeasurement
{
    public static double ToMeters(double centimeters) => centimeters / 100;
}

double m = IMeasurement.ToMeters(250);
```

## Still requires runtime support
- Instance default implementations
- `static abstract` and `static virtual` members
- `protected` interface members

## Pitfalls
- Irrelevant for projects already on modern runtimes; they supported this before.
