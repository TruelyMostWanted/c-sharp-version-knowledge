---
id: cs10-deconstruction-mixed
csharp: 10
tfm: net6.0
status: stable
tags: [deconstruction, tuple]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10
---
# Assignment and declaration in the same deconstruction (C# 10)

```csharp
int y = 0;
(var x, y) = GetPosition();   // declares x, assigns existing y
```
Before C# 10 a deconstruction had to either declare all variables or assign all existing ones.
