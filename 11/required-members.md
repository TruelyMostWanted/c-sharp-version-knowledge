---
id: cs11-required-members
csharp: 11
tfm: net7.0
status: stable
needs-runtime: true (RequiredMemberAttribute, SetsRequiredMembersAttribute, CompilerFeatureRequiredAttribute; .NET 7 or polyfill)
tags: [required, init, object-initializer, dto, config]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/required
modernizes: large constructors only to enforce mandatory values; runtime null checks on DTOs
---
# Required members (C# 11)

> **Abstract.** The `required` modifier on a field or property forces every caller that creates the object to set
> that member, typically in an object initializer. Missing assignments are **compile errors**. This combines the
> readability of object initializers with the guarantees of constructor parameters, and System.Text.Json honours it
> during deserialization.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`) |
| Status | Stable |
| Kind | Language (initialization contract) |
| Runtime support needed | Attribute types from .NET 7 (can be polyfilled) |
| Audience | Everyday code: DTOs, configuration, options |
| Modernize category | CASE (public API) |

## 1. Motivation

Two ways existed to create objects with mandatory data, both with drawbacks:

- **Constructor parameters**: guaranteed, but long parameter lists are unreadable at call sites and painful to evolve
  (every new member changes all constructors and derived classes).
- **Object initializers with `init`**: readable, but nothing prevents forgetting a member; with nullable reference types
  this produced warnings (CS8618) or `= null!` hacks.

## 2. The feature explained

### 2.1 Syntax

```csharp
public sealed class UnitConfig
{
    public required string Id { get; init; }
    public required int MaxHealth { get; init; }
    public float Speed { get; init; } = 3f;
}
```

### 2.2 Semantics and rules

1. `required` applies to fields and properties (with a `set` or `init` accessor) of classes, structs and records.
2. Every object creation must set all required members in its object initializer, unless the constructor used is marked
   `[SetsRequiredMembers]` (`System.Diagnostics.CodeAnalysis`).
3. Required members must be at least as accessible as the containing type, and their setters must be accessible to callers.
4. Required members are inherited: derived types must also have them set; overrides cannot remove `required`.
5. `new()` generic constraint: a type with required members cannot be used for a `new()` type parameter (CS9040),
   unless its parameterless constructor is marked `[SetsRequiredMembers]`.
6. System.Text.Json (.NET 7+) treats `required` properties as required (throws `JsonException` when missing).
7. `required` checks presence, not value: `Id = null!` or `Id = ""` still compiles.

## 3. Code examples

### 3.1 Minimal example

```csharp
var archer = new UnitConfig { Id = "archer", MaxHealth = 40 };   // OK
var broken = new UnitConfig { Id = "orc" };                        // error CS9035: MaxHealth must be set
```

### 3.2 Extended examples

Records with required members (nominal style):

```csharp
public sealed record ServerOptions
{
    public required string Host { get; init; }
    public required int Port { get; init; }
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
}
```

`[SetsRequiredMembers]` on a convenience constructor:

```csharp
public sealed class Player
{
    public required string Name { get; init; }
    public required int Level { get; init; }

    public Player() { }

    [SetsRequiredMembers]
    public Player(string name, int level) { Name = name; Level = level; }
}

var a = new Player { Name = "Ann", Level = 1 };
var b = new Player("Bob", 3);                     // no initializer needed
```

JSON deserialization:

```csharp
var cfg = JsonSerializer.Deserialize<UnitConfig>("""{ "Id": "archer" }""");
// JsonException: missing required property 'MaxHealth'
```

Options pattern with validation on startup:

```csharp
public sealed class DatabaseOptions
{
    public required string ConnectionString { get; init; }
}
builder.Services.AddOptions<DatabaseOptions>().BindConfiguration("Database").ValidateOnStart();
```

(Configuration binding uses reflection and does not enforce `required` itself; validate explicitly.)

## 4. Before this feature

```csharp
public sealed class UnitConfig
{
    public UnitConfig(string id, int maxHealth) { Id = id; MaxHealth = maxHealth; }
    public string Id { get; }
    public int MaxHealth { get; }
    public float Speed { get; init; } = 3f;
}

// or
public string Id { get; init; } = null!;   // silences the warning, guarantees nothing
```

## 5. Under the hood

- Each required member is marked `[RequiredMember]`; the containing type too.
- Every constructor not marked `[SetsRequiredMembers]` gets
  `[Obsolete("Constructors of types with required members are not supported in this version of your compiler.", true)]`
  and `[CompilerFeatureRequired("RequiredMembers")]`, so older compilers refuse to call it rather than silently skipping members.
- Checking happens only at compile time; at runtime it is an ordinary constructor call followed by setter calls.

## 6. Performance and allocation

None.

## 7. Pitfalls

- `[SetsRequiredMembers]` is a promise the compiler does **not** verify. A constructor that forgets a member makes the guarantee false.
- Reflection-based creation (`Activator.CreateInstance`, ORMs, older serializers, configuration binding) bypasses the check.
- `required` does not mean "non-null" or "non-empty". Combine with nullable annotations and validation.

## 8. Concerns and limitations

- API evolution: adding a `required` member to a public type is a **breaking change** for all callers.
- Godot: `GodotObject`-derived types (`Node`, `Resource`) are created by the engine without an initializer;
  do not use `required` on them. Use it on plain C# config and DTO types.
- `netstandard2.0` libraries need polyfills for the attributes; consumers need a C# 11+ compiler.

## 9. When to use: clean-code guidance

### 9.1 Use when
- DTOs, options, configuration and message types with several mandatory values.
- Replacing `= null!` initializers and long constructors used only to force values.

### 9.2 Avoid when
- The object has invariants across members (validate in a constructor or factory).
- Engine-instantiated types (Godot nodes and resources).

### 9.3 Good versus bad

```csharp
// Good
public sealed class SpawnRequest
{
    public required string UnitId { get; init; }
    public required Vector2 Position { get; init; }
    public int Count { get; init; } = 1;
}

// Bad: required on a Godot resource; the engine cannot satisfy it
[GlobalClass] public partial class UnitData : Resource { public required string Id { get; set; } }
```

## 10. Related features

- [../12/primary-constructors.md](../12/primary-constructors.md): the constructor-based alternative.
- [../10/record-structs.md](../10/record-structs.md)

## 11. References

- [required modifier](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/required)
- [Proposal: Required members](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/required-members)
- [Required properties in System.Text.Json](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/required-properties)
