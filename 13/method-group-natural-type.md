---
id: cs13-method-group-natural-type
csharp: 13
tfm: net9.0
status: stable
tags: [method-group, var, delegate, inference]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13
---
# Method group natural type improvements (C# 13)

When inferring the natural (delegate) type of a method group (e.g. `var f = SomeMethod;`), the compiler now
prunes inapplicable candidates (wrong generic arity, unsatisfied constraints) scope by scope and moves to
an outer scope only if none were found.
If all candidates at a scope fail, the method group has no natural type.

Effect: fewer surprising ambiguity errors with `var` and method groups. No new syntax.
