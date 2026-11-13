# Modernize C#: procedure for agents

Use this file when an instruction says "as modern as possible", "modernize", "use the latest C#",
"auf den neuesten Stand bringen" or similar, and also as the default style guide for **new** code.

The goal is the newest idioms the project **allows**, without changing behaviour, public APIs or
performance characteristics unless the task says so.

---

## 1. Procedure

1. **Find the ceiling.** Determine the allowed C# version (README.md, Step 1). Everything below is limited to it.
2. **Scope.** Only touch files that are part of the task, unless the task is explicitly "modernize the codebase".
   Never modify generated code (`*.g.cs`, `obj/`), vendored or upstream code, or files a CODEOWNERS or convention marks as external.
3. **Walk the versions upwards.** Start with the baseline (C# 9 and earlier, section 3), then go 10, 11, 12, ...
   up to the ceiling, and apply the rows of section 4 by category:
   - **AUTO**: mechanical and behaviour-preserving. Apply everywhere in scope.
   - **PREFER**: use in new code and in code you touch anyway; do not rewrite untouched code just for this.
   - **CASE**: needs judgement (API shape, performance, semantics). Apply only with a concrete benefit; mention it in the summary.
   - **NEVER-AUTO**: never as a drive-by change. Only when explicitly asked.
4. **Verify after each category**: build without new warnings, run the tests, then the next category.
   Keep changes per category in separate commits ("modernize: file-scoped namespaces").
5. **Report**: list what was applied per version and what was skipped, and why.

## 2. Rules that always win

- **Behaviour first.** If a rewrite could change semantics (evaluation order, null handling, equality,
  allocation in hot paths, reflection or serialization contracts), it is not AUTO.
- **Public API.** Changing constructor shapes, `required`, generic attributes, parameter modifiers or base types of
  public types is a breaking change. Only on request.
- **Hot paths** (per entity, per frame, tight loops): no new allocations, closures, boxing (unions with value-type cases), LINQ.
- **Godot:** `GodotObject`-derived scripts need a parameterless constructor (no primary constructors, no `required`);
  `[Export]`, signals and `Variant` accept only Variant-compatible types (no unions, closed records, tuples);
  `?.` does not detect freed nodes (`IsInstanceValid`).
- **Style config wins.** If `.editorconfig` or project conventions prescribe something, follow them.
- **Prefer tools over hand edits** for AUTO rows: `dotnet format style --diagnostics <IDs>` with the analyzer IDs below,
  then review the diff.

---

## 3. Baseline: C# 9 and earlier (assume known; always use for new code)

| Old idiom | Modern form | Since | Category |
|---|---|---|---|
| No nullable annotations | `<Nullable>enable</Nullable>`, `string?` | C# 8 | CASE for existing code (many warnings), PREFER for new projects |
| `switch` statements returning values | `switch` expressions | C# 8 | PREFER |
| `x != null && x is Foo` chains | `is not null`, `and` / `or` / `not` patterns, relational patterns `>= 0 and < 10` | C# 9 | AUTO |
| `using (var x = ...) { }` | `using var x = ...;` | C# 8 | PREFER |
| `x = x ?? y` | `x ??= y` | C# 8 | AUTO |
| `arr[arr.Length - 1]`, `Substring` math | `arr[^1]`, `s[1..^1]` | C# 8 | PREFER |
| `new Foo()` with explicit type on the left | target-typed `new()` (IDE0090) | C# 9 | AUTO |
| DTOs with hand-written equality | `record` / `record class` | C# 9 | CASE (equality semantics change) |
| Settable properties for immutable data | `init` accessors | C# 9 | CASE (API) |
| `Main` boilerplate in tools | top-level statements | C# 9 | PREFER for tools and samples only |
| Local functions capturing by accident | `static` local functions and lambdas | C# 8/9 | PREFER |

---

## 4. Version by version

### C# 10 (`net6.0`)
| Old idiom | Modern form | Category | File |
|---|---|---|---|
| `namespace X { ... }` | `namespace X;` (IDE0161) | AUTO | [10/file-scoped-namespace.md](10/file-scoped-namespace.md) |
| Same usings in every file | one `GlobalUsings.cs` / `ImplicitUsings` | PREFER | [10/global-using.md](10/global-using.md) |
| `{ A: { B: x } }` | `{ A.B: x }` | AUTO | [10/extended-property-patterns.md](10/extended-property-patterns.md) |
| `if (x == null) throw new ArgumentNullException(nameof(x));` | `ArgumentNullException.ThrowIfNull(x);` | AUTO | [10/caller-argument-expression.md](10/caller-argument-expression.md) |
| `const` strings concatenated with `+` | constant interpolated strings | AUTO | [10/constant-interpolated-strings.md](10/constant-interpolated-strings.md) |
| `Func<int,int> f = x => ...` only to get a type | `var f = (int x) => ...` | PREFER | [10/lambda-improvements.md](10/lambda-improvements.md) |
| Small immutable structs with hand-written equality | `readonly record struct` | CASE | [10/record-structs.md](10/record-structs.md) |
| `string.Format` / `+` concatenation | interpolation `$"..."` | PREFER | [10/interpolated-string-handlers.md](10/interpolated-string-handlers.md) |
| Dummy initialization or `!` to silence definite-assignment warnings | remove when no longer needed | CASE | [10/improved-definite-assignment.md](10/improved-definite-assignment.md) |

### C# 11 (`net7.0`)
| Old idiom | Modern form | Category | File |
|---|---|---|---|
| `@"..."` with `""` escapes, escaped JSON/SQL/shader text | raw string literals `"""` | PREFER | [11/raw-string-literals.md](11/raw-string-literals.md) |
| `[Attr("paramName")]` in attributes | `nameof(param)` | AUTO | [11/extended-nameof-scope.md](11/extended-nameof-scope.md) |
| `span.SequenceEqual("x")`, `span.ToString() == "x"` | `span is "x"` / `switch` | AUTO | [11/span-pattern-constant-string.md](11/span-pattern-constant-string.md) |
| `Encoding.UTF8.GetBytes("const")` | `"const"u8` | PREFER | [11/utf8-string-literals.md](11/utf8-string-literals.md) |
| `Length` checks plus indexing | list patterns | CASE | [11/list-patterns.md](11/list-patterns.md) |
| Private helper types with clash-prone names | `file` types | PREFER | [11/file-local-types.md](11/file-local-types.md) |
| Constructors only to force mandatory values | `required` members | CASE (API) | [11/required-members.md](11/required-members.md) |
| `[X(typeof(T))]` | `[X<T>]` | CASE (API) | [11/generic-attributes.md](11/generic-attributes.md) |
| One overload per numeric type | generic math `where T : INumber<T>` | CASE | [11/generic-math.md](11/generic-math.md) |
| `IntPtr` / `UIntPtr` in new code | `nint` / `nuint` | PREFER | [11/nint-nuint.md](11/nint-nuint.md) |
| Constructors assigning default values to every struct field | drop the redundant assignments | PREFER | [11/auto-default-structs.md](11/auto-default-structs.md) |
| All-lowercase type names | rename (CS8981) | CASE | [11/warning-wave-7.md](11/warning-wave-7.md) |

### C# 12 (`net8.0`)
| Old idiom | Modern form | Category | File |
|---|---|---|---|
| `new int[] { 1, 2 }`, `new List<T> { ... }`, `Array.Empty<T>()` for locals and fields | collection expressions `[1, 2]`, `[]` (IDE0300–IDE0305) | AUTO (not when the target is an interface whose concrete type matters) | [12/collection-expressions.md](12/collection-expressions.md) |
| Constructor that only copies parameters into private fields (services, DI) | primary constructor (IDE0290) | CASE; never on `GodotObject` scripts | [12/primary-constructors.md](12/primary-constructors.md) |
| `if (x < 0) throw new ArgumentOutOfRangeException(...)` | `ArgumentOutOfRangeException.ThrowIfNegative(x)` etc. | AUTO | [10/caller-argument-expression.md](10/caller-argument-expression.md) |
| Long tuple or array types repeated | `using Alias = (int X, int Y);` | CASE | [12/alias-any-type.md](12/alias-any-type.md) |
| `ref` parameters that are never written | `ref readonly` | CASE (API) | [12/ref-readonly-parameters.md](12/ref-readonly-parameters.md) |
| `unsafe fixed` buffers in structs | `[InlineArray(N)]` | CASE | [12/inline-arrays.md](12/inline-arrays.md) |
| Overloads only to emulate a lambda default | lambda default parameters | PREFER | [12/lambda-default-parameters.md](12/lambda-default-parameters.md) |

### C# 13 (`net9.0`)
| Old idiom | Modern form | Category | File |
|---|---|---|---|
| `private readonly object _lock = new();` | `private readonly Lock _lock = new();` | AUTO if the object is only used in `lock` statements | [13/lock-type.md](13/lock-type.md) |
| `params T[]` on hot APIs | `params ReadOnlySpan<T>` | PREFER for new APIs, CASE for existing | [13/params-collections.md](13/params-collections.md) |
| `\u001b` / `\x1b` | `\e` | AUTO | [13/escape-sequence-e.md](13/escape-sequence-e.md) |
| Copying spans to arrays only because of an `await` later | span locals between awaits | CASE | [13/ref-and-unsafe-in-async-iterators.md](13/ref-and-unsafe-in-async-iterators.md) |
| Generic helpers that exclude spans | `allows ref struct` | CASE | [13/allows-ref-struct.md](13/allows-ref-struct.md) |

### C# 14 (`net10.0`)
| Old idiom | Modern form | Category | File |
|---|---|---|---|
| Private backing field used only by one property with logic | `field` keyword | AUTO (check no member named `field` exists) | [14/field-keyword.md](14/field-keyword.md) |
| `if (x is not null) x.P = v;` | `x?.P = v;` | PREFER; keep explicit checks where null is a bug or for Godot nodes that may be freed | [14/null-conditional-assignment.md](14/null-conditional-assignment.md) |
| `GetX()` extension methods that are conceptually properties; static helper classes for a type | `extension` blocks (properties, static members) | PREFER for new code; do not rewrite existing `this` extension methods | [14/extension-members.md](14/extension-members.md) |
| `(string s, out int r) => ...` only to add `out` | `(s, out r) => ...` | AUTO | [14/lambda-parameter-modifiers.md](14/lambda-parameter-modifiers.md) |
| `x = x + y` allocating on mutable reference types | instance `operator +=` | CASE | [14/user-defined-compound-assignment.md](14/user-defined-compound-assignment.md) |
| `nameof(List<int>)` used only for the name | `nameof(List<>)` | AUTO | [14/nameof-unbound-generic-types.md](14/nameof-unbound-generic-types.md) |
| Tool projects with one file | file-based apps | CASE | [14/file-based-app-directives.md](14/file-based-app-directives.md) |

### C# 15 (`net11.0`)
| Old idiom | Modern form | Category | File |
|---|---|---|---|
| `new List<T>(n)` or `new HashSet<T>(cmp)` followed by adds | `[with(capacity: n), ...]`, `[with(cmp), ...]` | PREFER | [15/collection-expression-arguments.md](15/collection-expression-arguments.md) |
| Bool flags or `goto` to leave nested loops | labeled `break` / `continue` (IDE0410) | AUTO | [15/labeled-break-continue.md](15/labeled-break-continue.md) |
| Abstract base plus `switch` with `_ => throw` default | `closed` hierarchy, remove the default arm | CASE | [15/closed-hierarchies.md](15/closed-hierarchies.md) |
| `object`, marker interfaces, `OneOf<...>` packages, result-or-error tuples | `union` | CASE; never in hot paths with value-type cases, never across the Godot boundary | [15/union-types.md](15/union-types.md) |
| `GetPixel(x, y)`-style accessors on foreign types | extension indexers | CASE | [15/extension-indexers.md](15/extension-indexers.md) |
| `unsafe` blocks around pointer declarations | memory-safety rules | NEVER-AUTO (preview) | [15/memory-safety-preview.md](15/memory-safety-preview.md) |

---

## 5. Defaults for new code (short version)

When writing new code, use without asking (within the ceiling): file-scoped namespaces, nullable annotations,
`var` where the type is obvious, target-typed `new()`, collection expressions, switch expressions and patterns,
`is not null`, `ThrowIfNull` and friends, raw strings for multi-line text, `readonly record struct` for small value types,
`field` in properties with logic, `Lock` for locks, labeled `break`/`continue` instead of flags.
Use with judgement: primary constructors (plain C# classes only), `required`, unions, `closed`, extension members, generic math.
