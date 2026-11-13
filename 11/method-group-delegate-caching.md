---
id: cs11-method-group-delegate-caching
csharp: 11
tfm: net7.0
status: stable
tags: [delegate, method-group, allocation, performance]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11
---
# Improved method group conversion to delegate (C# 11)

The compiler may now reuse (cache) the delegate object when a **static** method group is converted to a delegate,
as it already did for static lambdas. Passing `list.Sort(CompareByDistance)` repeatedly no longer allocates
a new delegate each time.

Consequence for agents: rewriting `items.Select(Transform)` into `items.Select(x => Transform(x))`
for performance reasons is no longer necessary when `Transform` is static.
Instance method groups still allocate a delegate per conversion.
