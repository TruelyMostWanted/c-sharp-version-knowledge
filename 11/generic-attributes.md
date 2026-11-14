---
id: cs11-generic-attributes
csharp: 11
tfm: net7.0
status: stable (preview in C# 10)
tags: [attribute, generics]
source: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#generic-attributes
modernizes: attributes taking a Type parameter ([X(typeof(Foo))])
---
# Generic attributes (C# 11)

> **Abstract.** Attribute classes may be generic: `class ComponentAttribute<T> : Attribute`, used as
> `[Component<HealthComponent>]`. Type arguments get compile-time checking and constraints, replacing the
> `Type` constructor parameter (`[Component(typeof(HealthComponent))]`) that accepted any type.

| | |
|---|---|
| Introduced | C# 11 · .NET 7 (`net7.0`); preview in C# 10 |
| Status | Stable |
| Kind | Syntax + metadata |
| Runtime support needed | Reading generic attributes via reflection is reliable on .NET 7+ |
| Audience | Library and framework authors; consumers of such APIs |
| Modernize category | CASE (public API change) |

## 1. Motivation

Attributes that refer to a type had to take a `System.Type` argument:

```csharp
[TypeConverter(typeof(MyConverter))]
```

The compiler could not check that `MyConverter` actually is a converter; mistakes surfaced at runtime.
Generic constraints are the natural tool for this, but attribute classes could not be generic.

## 2. The feature explained

### 2.1 Syntax

```csharp
public sealed class ValidatorAttribute<T> : Attribute where T : IValidator, new() { }

[Validator<EmailValidator>]
public string Email { get; set; } = "";
```

### 2.2 Semantics and rules

1. Any class deriving from `Attribute` may declare type parameters and constraints.
2. Type arguments must be fully known at compile time. Not allowed: `dynamic`, nullable reference annotations
   (`string?`), tuple element names (`(int X, int Y)`), and type parameters of an enclosing generic type or method.
3. The usual attribute rules apply (constructor arguments must be constants, `typeof`, arrays of these).
4. `[Attr<T>]` and `[AttrAttribute<T>]` both work (suffix rule).
5. `AttributeUsage(AllowMultiple = true)` allows `[Attr<A>, Attr<B>]` on the same target.

## 3. Code examples

### 3.1 Minimal example

```csharp
public class ComponentAttribute<T> : Attribute where T : class { }

[Component<HealthComponent>]
public partial class Enemy { }
```

### 3.2 Extended examples

Constraints catch mistakes at compile time:

```csharp
public interface IValidator { bool IsValid(object? value); }
public sealed class EmailValidator : IValidator { public bool IsValid(object? v) => v is string s && s.Contains('@'); }

[AttributeUsage(AttributeTargets.Property, AllowMultiple = true)]
public sealed class ValidateAttribute<T> : Attribute where T : IValidator, new()
{
    public IValidator Create() => new T();
}

public sealed class Account
{
    [Validate<EmailValidator>] public string Email { get; set; } = "";
    // [Validate<string>]  -> compile error: string does not implement IValidator
}
```

Reading a generic attribute with reflection:

```csharp
foreach (PropertyInfo p in typeof(Account).GetProperties())
foreach (Attribute a in p.GetCustomAttributes())
{
    Type t = a.GetType();
    if (t.IsGenericType && t.GetGenericTypeDefinition() == typeof(ValidateAttribute<>))
    {
        var validator = (IValidator)t.GetMethod("Create")!.Invoke(a, null)!;
        ...
    }
}

// or, if the closed type is known:
var attr = typeof(Enemy).GetCustomAttribute<ComponentAttribute<HealthComponent>>();
```

A source generator registry:

```csharp
[RegisterSystem<MovementSystem>]
[RegisterSystem<CombatSystem>]
public partial class GameWorld { }
```

## 4. Before this feature

```csharp
[AttributeUsage(AttributeTargets.Property)]
public sealed class ValidateAttribute : Attribute
{
    public ValidateAttribute(Type validatorType)
    {
        if (!typeof(IValidator).IsAssignableFrom(validatorType))
            throw new ArgumentException("Not a validator", nameof(validatorType));   // runtime only
        ValidatorType = validatorType;
    }
    public Type ValidatorType { get; }
}

[Validate(typeof(EmailValidator))] public string Email { get; set; } = "";
```

## 5. Under the hood

The custom attribute's constructor token refers to a method on the **closed generic type**
(`ValidateAttribute`1<EmailValidator>::.ctor`) via a `MemberRef` on a `TypeSpec`. Constraints are checked by the compiler.
Older runtimes and some tools had bugs reading such attributes, which is why C# 11 requires .NET 7 for full support.

## 6. Performance and allocation

Same as other attributes: instantiated on demand by reflection. No runtime cost unless read.

## 7. Pitfalls

- `GetCustomAttributes(typeof(ValidateAttribute<>))` with the open generic type does **not** match closed instances;
  compare `GetGenericTypeDefinition()` as shown above.
- `[Attr<string?>]` is an error; nullable annotations are not allowed.
- Changing `[X(typeof(T))]` to `[X<T>]` in a public library breaks consumers that read `XAttribute` by reflection.

## 8. Concerns and limitations

- Tooling and frameworks must understand generic attributes; older serializers, DI containers or analyzers may ignore them.
- Godot: Godot's own attributes (`[Export]`, `[GlobalClass]`) are not generic; use generic attributes in your own C# tooling and source generators only.

## 9. When to use: clean-code guidance

### 9.1 Use when
- Designing new attributes that refer to a type, especially with constraints.
- Source-generator inputs (`[Generate<TModel>]`).

### 9.2 Avoid when
- An existing public attribute would need a breaking rewrite without clear benefit.
- Target frameworks older than .NET 7 that must read the attribute at runtime.

### 9.3 Good versus bad

```csharp
// Good: constraint encodes the contract
public sealed class HandlerAttribute<THandler> : Attribute where THandler : IMessageHandler { }

// Bad: generic but unconstrained; no benefit over typeof
public sealed class HandlerAttribute<T> : Attribute { }
```

## 10. Related features

- [../10/preview-features.md](../10/preview-features.md): the C# 10 preview.
- [file-local-types.md](file-local-types.md): generator-friendly types.

## 11. References

- [Generic attributes](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#generic-attributes)
- [Proposal: Generic attributes](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/generic-attributes)
