---
id: cs13-implicit-index-in-object-initializers
csharp: 13
tfm: net9.0
status: stable
tags: [index, hat-operator, object-initializer]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
---
# `^` index in object initializers (C# 13)

The from-end index `^` is allowed in object initializers for single-dimension collections.

```csharp
public class TimerRemaining
{
    public int[] buffer { get; set; } = new int[10];
}

var countdown = new TimerRemaining
{
    buffer =
    {
        [^1] = 0,
        [^2] = 1,
        [^3] = 2,
    }
};
```
