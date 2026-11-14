---
id: cs10-constant-interpolated-strings
csharp: 10
tfm: net6.0
status: stable
tags: [const, string-interpolation]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10
modernizes: const strings concatenated with +
---
# Constant interpolated strings (C# 10)

> **Abstract.** A `const string` may be initialized with an interpolated string, as long as **every**
> placeholder is itself a constant string. The compiler folds the result into a single literal.
> This makes composed constants (paths, routes, keys) readable without `+` chains, and they remain usable
> wherever constants are required: attributes, `switch` cases, default parameter values.

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Semantics (constant folding) |
| Runtime support needed | No (compile-time only) |
| Audience | Everyday code |
| Modernize category | AUTO |

## 1. Motivation

Constants built from other constants could only use `+`:

```csharp
const string Root = "res://data";
const string Units = Root + "/units";
const string Archer = Units + "/archer.tres";
```

That works, but interpolation is the idiomatic way to compose strings and is easier to read once more pieces
are involved. Before C# 10, `$"..."` was never a constant expression, so it could not be used for `const`,
attribute arguments or `case` labels.

## 2. The feature explained

### 2.1 Syntax

```csharp
const string A = "x";
const string B = $"{A}/y";
```

### 2.2 Semantics and rules

1. Every hole must be a constant expression of type `string`.
2. Numeric, `char`, `bool` or enum constants are **not** allowed, because their formatting depends on
   culture or `ToString()` at runtime. Use `nameof(...)` (a constant string) or a string constant instead.
3. No alignment or format specifiers (`{A,10}`, `{A:X}`) in constant interpolations.
4. The result is a constant and can be used anywhere constants are required.
5. Verbatim (`$@"..."`) forms are allowed under the same rules; raw string literals (C# 11) also work.

## 3. Code examples

### 3.1 Minimal example

```csharp
const string SaveDir  = "user://saves";
const string SaveFile = $"{SaveDir}/slot.json";   // "user://saves/slot.json"
```

### 3.2 Extended examples

Attribute arguments and routes:

```csharp
public static class Routes
{
    public const string Api = "api/v1";
    public const string Units = $"{Api}/units";
}

[Route(Routes.Units)]
public sealed class UnitsController : ControllerBase { }
```

`switch` case labels:

```csharp
const string Prefix = "cmd";
const string Spawn = $"{Prefix}.spawn";
const string Kill  = $"{Prefix}.kill";

switch (command)
{
    case Spawn: DoSpawn(); break;
    case Kill:  DoKill();  break;
}
```

Composing with `nameof` (constant string) for diagnostic IDs or keys:

```csharp
public const string HealthKey = $"{nameof(Player)}.{nameof(Player.Health)}";   // "Player.Health"
```

Default parameter value:

```csharp
public void Load(string path = $"{SaveDir}/autosave.json") { ... }
```

## 4. Before this feature

```csharp
const string SaveFile = SaveDir + "/slot.json";      // still valid, still constant
static readonly string SaveFile2 = $"{SaveDir}/slot.json"; // not a constant: cannot be used in attributes
```

## 5. Under the hood

The expression is folded at compile time. IL contains only the final literal:

```
ldstr "user://saves/slot.json"
```

No `string.Concat`, no `DefaultInterpolatedStringHandler`, no runtime work. Like every `const`, the value is
also copied into the **metadata of each consuming assembly**.

## 6. Performance and allocation

None. The string is an interned literal.

## 7. Pitfalls

- `const string Name = $"Unit{Count}";` with `const int Count` does **not** compile (numeric hole).
  Either make it `"Unit" + "3"` style literals, or use `static readonly`.
- Changing a public `const` requires recompiling all consumers; old binaries keep the old value baked in.

## 8. Concerns and limitations

- Versioning of public constants (see pitfalls). For values that may change between releases, prefer `static readonly`.
- Culture: the restriction to string holes is precisely what guarantees culture-independent results.
- Godot: useful for `res://` and `user://` paths and input-action names used in many places.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Building constants from other constants (paths, routes, cache keys, diagnostic IDs).

### 9.2 Avoid when
- The value must be computed or contains numbers: use `static readonly`.
- A public API constant may change in later versions.

### 9.3 Good versus bad

```csharp
// Good
const string ScenesDir = "res://scenes";
const string MainMenu  = $"{ScenesDir}/main_menu.tscn";

// Bad: magic duplication of the root path
const string MainMenu2 = "res://scenes/main_menu.tscn";
const string Options   = "res://scenes/options.tscn";
```

## 10. Related features

- [interpolated-string-handlers.md](interpolated-string-handlers.md): non-constant interpolation performance.
- [../11/raw-string-literals.md](../11/raw-string-literals.md): raw interpolated strings.

## 11. References

- [What's new in C# 10](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#constant-interpolated-strings)
- [String interpolation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated)
