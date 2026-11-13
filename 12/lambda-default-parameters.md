---
id: cs12-lambda-default-parameters
csharp: 12
tfm: net8.0
status: stable
tags: [lambda, default-parameters]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12
---
# Default parameter values in lambdas (C# 12)

Same syntax and rules as default arguments of methods and local functions.

```csharp
var spawn = (string enemy, int count = 1) => $"{count}x {enemy}";
spawn("Goblin");      // "1x Goblin"
spawn("Orc", 3);      // "3x Orc"
```
