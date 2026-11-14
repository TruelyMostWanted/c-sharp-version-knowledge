---
id: cs10-record-sealed-tostring
csharp: 10
tfm: net6.0
status: stable
tags: [record, ToString, sealed]
source: https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#record-types-can-seal-tostring
---
# `sealed` `ToString` override in records (C# 10)

> **Abstract.** A record can declare `public sealed override string ToString()`. Derived records then **stop**
> synthesizing their own `ToString` and inherit this one. Use it to enforce one textual format across a whole
> record hierarchy (events, log entries, messages).

| | |
|---|---|
| Introduced | C# 10 · .NET 6 (`net6.0`) |
| Status | Stable |
| Kind | Semantics (record synthesis) |
| Runtime support needed | No |
| Audience | Everyday code with record hierarchies |
| Modernize category | n/a (design choice) |

## 1. Motivation

For every record type the compiler synthesizes `ToString()` (printing `TypeName { Prop = value, ... }`),
including derived records. If a base record defines a custom `ToString`, a derived record **overrides it again**
with a synthesized one. In C# 9 there was no way to stop that, because `sealed` on the override was not allowed in records.

## 2. The feature explained

### 2.1 Syntax

```csharp
public abstract record Base
{
    public sealed override string ToString() => ...;
}
```

### 2.2 Semantics and rules

1. Allowed in `record class` types only (records structs cannot be derived from).
2. If any base record seals `ToString`, derived records do not synthesize `ToString`.
3. `PrintMembers` is still synthesized and can be called from the sealed `ToString` to include derived members.
4. A derived record cannot declare its own `ToString` override (it is sealed).

## 3. Code examples

### 3.1 Minimal example

```csharp
public abstract record GameEvent(int Round)
{
    public sealed override string ToString() => $"[{Round}] {GetType().Name}";
}

public sealed record UnitDied(int Round, string UnitId) : GameEvent(Round);

Console.WriteLine(new UnitDied(3, "orc-7"));   // "[3] UnitDied"
```

### 3.2 Extended examples

Keep the base format but include derived members via `PrintMembers`:

```csharp
public abstract record LogEntry(DateTime At)
{
    public sealed override string ToString()
    {
        var sb = new StringBuilder();
        sb.Append(At.ToString("HH:mm:ss")).Append(' ').Append(GetType().Name).Append(' ');
        PrintMembers(sb);   // virtual: includes members of the derived record
        return sb.ToString();
    }
}

public sealed record Login(DateTime At, string User) : LogEntry(At);
// "12:00:01 Login At = 01/01/2026 12:00:01, User = ann"
```

Hiding secrets uniformly:

```csharp
public abstract record Command
{
    public sealed override string ToString() => GetType().Name;   // never print payloads (tokens, passwords)
}

public sealed record LoginCommand(string User, string Password) : Command;
```

## 4. Before this feature

```csharp
public abstract record GameEvent(int Round)
{
    public override string ToString() => $"[{Round}] {GetType().Name}";
}
public record UnitDied(int Round, string UnitId) : GameEvent(Round);
// prints "UnitDied { Round = 3, UnitId = orc-7 }": derived synthesized ToString wins
```

Workaround in C# 9: override `PrintMembers` in each derived type, or avoid records.

## 5. Under the hood

The base record's method is emitted as `virtual final` (`sealed override`). For derived records the compiler simply
does not emit a `ToString` method; calls dispatch to the base implementation. `PrintMembers` remains a
`protected virtual` method that each derived record overrides (in a `sealed` record it is `private`).

## 6. Performance and allocation

Same as any `ToString`. `GetType().Name` is cached by the runtime and cheap.

## 7. Pitfalls

- Sealing removes useful debug output of derived records unless you call `PrintMembers` yourself.
- Do not rely on `ToString()` output for persistence or equality; it is for humans.

## 8. Concerns and limitations

- API: sealing `ToString` in a public base record prevents consumers from customizing it in their derived records.
- Security: sealing is a good place to guarantee that secrets are never printed by synthesized `ToString`.

## 9. When to use: clean-code guidance

### 9.1 Use when
- A record hierarchy needs one consistent textual format (events, log entries, commands).
- Derived records must not print sensitive properties.

### 9.2 Avoid when
- Derived types legitimately need different output.

### 9.3 Good versus bad

```csharp
// Good: one format, derived data still included
public sealed override string ToString() { var sb = new StringBuilder(GetType().Name + " "); PrintMembers(sb); return sb.ToString(); }

// Bad: sealing ToString only to save typing, losing all derived information
public sealed override string ToString() => "Event";
```

## 10. Related features

- [record-structs.md](record-structs.md)
- [../15/closed-hierarchies.md](../15/closed-hierarchies.md): closed record hierarchies.

## 11. References

- [What's new in C# 10: sealed ToString](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#record-types-can-seal-tostring)
- [Records: built-in formatting](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record#built-in-formatting-for-display)
