# C# Language Reference for Agents

Compact notes on C# language features, one file per feature, each checked against its source.
Purpose: a coding agent can see which language features the project's C# version allows
and how to use them correctly, without guessing from training data or decompiling assemblies.

- Covers C# 10 (.NET 6) to C# 15 (.NET 11). C# 9 and earlier: baseline table in [MODERNIZE.md](MODERNIZE.md).
- Verified against learn.microsoft.com on 2026-11-13.
- Every feature file links its source. If a file and the official docs disagree, the docs win.

## Files
- `README.md`: this file. Version ceiling and feature index.
- [`MODERNIZE.md`](MODERNIZE.md): procedure for "as modern as possible", walking the versions old to new, plus defaults for new code.
- `<version>/<feature>.md`: one file per feature (frontmatter: version, TFM, status, tags, source, `modernizes`).

---

## Step 1: Determine the allowed C# version (always do this first)

1. Read the project's `.csproj` and, if present, `Directory.Build.props`.
2. If `<LangVersion>` is set, it is the ceiling.
   - `preview`: newest version plus preview features.
   - `latest`: newest released version supported by the installed SDK.
3. Otherwise the default comes from `<TargetFramework>`:

| TargetFramework | Default C# | Folder | Support |
|---|---|---|---|
| `net11.0` | 15 | [15/](15/) | STS, released 2026-11-10 |
| `net10.0` | 14 | [14/](14/) | LTS |
| `net9.0`  | 13 | [13/](13/) | end of support 2026-11-10 |
| `net8.0`  | 12 | [12/](12/) | end of support 2026-11-10 |
| `net7.0`  | 11 | [11/](11/) | out of support |
| `net6.0`  | 10 | [10/](10/) | out of support |
| `net5.0` and older, `netstandard2.x`, .NET Framework | 9 or lower | baseline in [MODERNIZE.md](MODERNIZE.md) | out of support / legacy |

4. **Never** use a feature from a folder above the ceiling.
5. Features with `status: preview` need `<LangVersion>preview</LangVersion>` (sometimes also a feature flag).
   Use them only if the project already opts in.
6. Raising `LangVersion` above the TargetFramework default is not supported by Microsoft. Some features
   also need types that only exist in the matching runtime (marked `needs-runtime: true`).

## Step 2: Open only what you need

Use the index below. Open single feature files on demand; do not load whole folders.
When a newer feature fits the code you are writing, prefer it over the older idiom, but
never rewrite working code just to use a newer feature unless asked.

## Step 3: "As modern as possible"

If the task asks for modern or up-to-date C# (or for a modernization pass), follow [MODERNIZE.md](MODERNIZE.md):
it walks from the baseline up to the ceiling version and says per idiom whether to apply it automatically,
only in touched code, case by case or never.

---

## Feature index

### C# 15 (`net11.0`)
| Feature | Use it for | File |
|---|---|---|
| Collection expression arguments | `[with(capacity: n), ..]`, comparers in `[...]` | [15/collection-expression-arguments.md](15/collection-expression-arguments.md) |
| Union types | Value is exactly one of several unrelated types | [15/union-types.md](15/union-types.md) |
| Closed hierarchies | Exhaustive `switch` over your own class hierarchy | [15/closed-hierarchies.md](15/closed-hierarchies.md) |
| Extension indexers | `obj[i]` on types you don't own | [15/extension-indexers.md](15/extension-indexers.md) |
| Labeled `break` / `continue` | Leave or skip outer loops without flags or `goto` | [15/labeled-break-continue.md](15/labeled-break-continue.md) |
| Static interface members on older runtimes | Static helpers in interfaces for old TFMs | [15/static-interface-members-older-runtimes.md](15/static-interface-members-older-runtimes.md) |
| Memory safety (preview) | `unsafe` only where memory is dereferenced | [15/memory-safety-preview.md](15/memory-safety-preview.md) |

### C# 14 (`net10.0`)
| Feature | Use it for | File |
|---|---|---|
| Extension members | Extension properties, static extensions, operators | [14/extension-members.md](14/extension-members.md) |
| `field` keyword | Logic in accessors without a manual backing field | [14/field-keyword.md](14/field-keyword.md) |
| Null-conditional assignment | `obj?.Prop = x;`, `obj?.Count += 1;` | [14/null-conditional-assignment.md](14/null-conditional-assignment.md) |
| Implicit span conversions | Arrays and spans convert implicitly; span extensions | [14/implicit-span-conversions.md](14/implicit-span-conversions.md) |
| `nameof` with unbound generics | `nameof(List<>)` | [14/nameof-unbound-generic-types.md](14/nameof-unbound-generic-types.md) |
| Lambda parameter modifiers | `(text, out result) => ...` without types | [14/lambda-parameter-modifiers.md](14/lambda-parameter-modifiers.md) |
| Partial events and constructors | Source-generator friendly declarations | [14/partial-events-and-constructors.md](14/partial-events-and-constructors.md) |
| User-defined compound assignment | In-place `+=`, `++` on mutable types | [14/user-defined-compound-assignment.md](14/user-defined-compound-assignment.md) |
| File-based app directives | `dotnet run tool.cs` with `#:package` etc. | [14/file-based-app-directives.md](14/file-based-app-directives.md) |

### C# 13 (`net9.0`)
| Feature | Use it for | File |
|---|---|---|
| `params` collections | `params ReadOnlySpan<T>` without array allocation | [13/params-collections.md](13/params-collections.md) |
| `System.Threading.Lock` | Faster, type-safe `lock` | [13/lock-type.md](13/lock-type.md) |
| `\e` escape | ANSI escape character | [13/escape-sequence-e.md](13/escape-sequence-e.md) |
| Method group natural type | Better `var f = Method;` inference | [13/method-group-natural-type.md](13/method-group-natural-type.md) |
| `^` in object initializers | `[^1] = x` inside initializers | [13/implicit-index-in-object-initializers.md](13/implicit-index-in-object-initializers.md) |
| `ref` / `unsafe` in async and iterators | `Span<T>` locals in async methods | [13/ref-and-unsafe-in-async-iterators.md](13/ref-and-unsafe-in-async-iterators.md) |
| `allows ref struct` | Generics over `Span<T>` and other ref structs | [13/allows-ref-struct.md](13/allows-ref-struct.md) |
| `ref struct` interfaces | ref structs implementing interfaces | [13/ref-struct-interfaces.md](13/ref-struct-interfaces.md) |
| Partial properties and indexers | Source generators | [13/partial-properties-and-indexers.md](13/partial-properties-and-indexers.md) |
| Overload resolution priority | Library authors steering overloads | [13/overload-resolution-priority.md](13/overload-resolution-priority.md) |
| `field` keyword (preview in 13) | See C# 14 | [13/field-keyword-preview.md](13/field-keyword-preview.md) |

### C# 12 (`net8.0`)
| Feature | Use it for | File |
|---|---|---|
| Primary constructors | Constructor parameters in scope for the whole class | [12/primary-constructors.md](12/primary-constructors.md) |
| Collection expressions | `[1, 2, 3]`, spread `..` | [12/collection-expressions.md](12/collection-expressions.md) |
| Inline arrays | Fixed-size buffers inside structs | [12/inline-arrays.md](12/inline-arrays.md) |
| Lambda default parameters | `(int x = 5) => ...` | [12/lambda-default-parameters.md](12/lambda-default-parameters.md) |
| `ref readonly` parameters | Pass large structs by readonly reference | [12/ref-readonly-parameters.md](12/ref-readonly-parameters.md) |
| Alias any type | `using Point = (int X, int Y);` | [12/alias-any-type.md](12/alias-any-type.md) |
| `[Experimental]` | Mark unstable APIs | [12/experimental-attribute.md](12/experimental-attribute.md) |
| Interceptors (preview) | Source generators only | [12/interceptors-preview.md](12/interceptors-preview.md) |

### C# 11 (`net7.0`)
| Feature | Use it for | File |
|---|---|---|
| Raw string literals | JSON, SQL, shader or code text without escaping | [11/raw-string-literals.md](11/raw-string-literals.md) |
| Generic math | Numeric algorithms once for all number types; static abstract members | [11/generic-math.md](11/generic-math.md) |
| Generic attributes | `[Attr<T>]` instead of `typeof` | [11/generic-attributes.md](11/generic-attributes.md) |
| UTF-8 string literals | `"text"u8` as `ReadOnlySpan<byte>` | [11/utf8-string-literals.md](11/utf8-string-literals.md) |
| Newlines in interpolation holes | Multi-line expressions inside `{ }` | [11/newlines-in-interpolations.md](11/newlines-in-interpolations.md) |
| List patterns | Match arrays and lists by shape, slices `..` | [11/list-patterns.md](11/list-patterns.md) |
| File-local types | `file class` visible in one file only | [11/file-local-types.md](11/file-local-types.md) |
| Required members | Force initialization in object initializers | [11/required-members.md](11/required-members.md) |
| Auto-default structs | Struct constructors need not assign every field | [11/auto-default-structs.md](11/auto-default-structs.md) |
| Span pattern on constant string | `span is "text"` without allocation | [11/span-pattern-constant-string.md](11/span-pattern-constant-string.md) |
| Extended `nameof` scope | `nameof(param)` inside attributes | [11/extended-nameof-scope.md](11/extended-nameof-scope.md) |
| `nint` / `nuint` aliases | Native-sized integers equal `IntPtr` / `UIntPtr` | [11/nint-nuint.md](11/nint-nuint.md) |
| `ref` fields and `scoped` | Low-level ref structs | [11/ref-fields-scoped.md](11/ref-fields-scoped.md) |
| Method group delegate caching | Static method groups no longer allocate per conversion | [11/method-group-delegate-caching.md](11/method-group-delegate-caching.md) |
| Warning wave 7 | CS8981 lowercase type names | [11/warning-wave-7.md](11/warning-wave-7.md) |

### C# 10 (`net6.0`)
| Feature | Use it for | File |
|---|---|---|
| Record structs | Value-type records, `readonly record struct` | [10/record-structs.md](10/record-structs.md) |
| Struct improvements | Parameterless constructors, initializers, `with` on structs | [10/struct-improvements.md](10/struct-improvements.md) |
| Interpolated string handlers | Fast, allocation-aware interpolation | [10/interpolated-string-handlers.md](10/interpolated-string-handlers.md) |
| `global using` | Project-wide usings | [10/global-using.md](10/global-using.md) |
| File-scoped namespace | `namespace X;` | [10/file-scoped-namespace.md](10/file-scoped-namespace.md) |
| Extended property patterns | `{ A.B.C: value }` | [10/extended-property-patterns.md](10/extended-property-patterns.md) |
| Lambda improvements | Natural type (`var`), explicit return type, attributes | [10/lambda-improvements.md](10/lambda-improvements.md) |
| Constant interpolated strings | `const string B = $"{A}/x";` | [10/constant-interpolated-strings.md](10/constant-interpolated-strings.md) |
| `sealed ToString` in records | Keep one `ToString` for derived records | [10/record-sealed-tostring.md](10/record-sealed-tostring.md) |
| Improved definite assignment | Fewer false warnings with `?.` and `is` | [10/improved-definite-assignment.md](10/improved-definite-assignment.md) |
| Mixed deconstruction | `(var x, y) = ...` | [10/deconstruction-mixed.md](10/deconstruction-mixed.md) |
| `[AsyncMethodBuilder]` on methods | Pooled async state machines | [10/asyncmethodbuilder-on-methods.md](10/asyncmethodbuilder-on-methods.md) |
| `[CallerArgumentExpression]` | Guards and asserts with the source expression | [10/caller-argument-expression.md](10/caller-argument-expression.md) |
| New `#line` format | Code generators only | [10/line-pragma-format.md](10/line-pragma-format.md) |
| Preview features of C# 10 | Generic attributes, static abstract: see C# 11 | [10/preview-features.md](10/preview-features.md) |

---

## Godot (C#) notes that apply to all versions

- Godot C# projects use `Godot.NET.Sdk`. Raise `<TargetFramework>` to use a newer C# version; test editor, build and every export platform afterwards.
- Types that cross into the engine (`[Export]`, signal parameters, `Variant`) must be Variant-compatible.
  Newer C# constructs (unions, records, closed hierarchies) belong in the pure C# logic layer.
- Script classes deriving from `GodotObject` need a parameterless constructor.
- Hot paths (per entity, per frame): avoid allocations and boxing. Feature files flag where this matters.

---

## Wiring this into AGENTS.md

Copy this section into the repository's `AGENTS.md`:

```markdown
## C# language features
- Before writing C#, determine the allowed C# version as described in
  `.agents/knowledge/csharp/README.md` (Step 1). Never use features above it.
- Look up features in the index of that README and open only the files you need.
- Preview features only if the project already sets `<LangVersion>preview</LangVersion>`.
- "As modern as possible" / modernization tasks: follow `.agents/knowledge/csharp/MODERNIZE.md`.
- If a note conflicts with the official docs linked in the file, follow the docs.
```
