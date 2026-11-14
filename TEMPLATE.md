# Chapter template for feature files

Every `<version>/<feature>.md` is written as a short book chapter with the same section order,
so readers (humans and agents) always know where to look. Agents in a hurry read the
**abstract and fact table** first and jump to the section they need.

Sections that do not apply stay in place with one sentence saying why
(for example "No runtime cost: purely compile-time."). Never drop or reorder sections.
Pointer chapters (a preview feature that became stable later) use the short form at the end of this file.

---

## Full structure

````markdown
---
id: cs<version>-<feature>
csharp: <version>
tfm: net<x>.0
status: stable | preview | preview (stable in C# n)
needs-runtime: true (<type or runtime>)      # only if the feature needs runtime or BCL support
analyzer: IDE0xxx                            # only if an analyzer exists
tags: [...]
source: https://learn.microsoft.com/...      # or a list
modernizes: <old idiom>                      # only if the feature replaces an older idiom
related: <relative paths>                    # optional
---
# <Feature name> (C# <version>)

> **Abstract.** Two to four sentences: what the feature is, what problem it solves, and the one rule
> that matters most.

| | |
|---|---|
| Introduced | C# n · .NET n (`netn.0`) |
| Status | Stable / Preview |
| Kind | Syntax · Semantics · Compiler behaviour · Runtime + library |
| Runtime support needed | No (compile-time only) / Yes (...) |
| Audience | Everyday code · Library authors · Source-generator authors · Low-level code |
| Modernize category | AUTO · PREFER · CASE · NEVER-AUTO · n/a (see MODERNIZE.md) |

## 1. Motivation
The problem as it existed before the feature; why the language team added it.

## 2. The feature explained
### 2.1 Syntax
Grammar-level description with a compact snippet.
### 2.2 Semantics and rules
Numbered or bulleted rules, each one checkable.

## 3. Code examples
### 3.1 Minimal example
The smallest program that shows the feature.
### 3.2 Extended examples
Two or more realistic examples (general .NET, and game / Godot where it fits).

## 4. Before this feature
The idiom used in earlier C# versions, side by side with the new form.

## 5. Under the hood
What the compiler generates: lowered C# and, where useful, IL.
Generated names (`<>c`, `<X>k__BackingField`, ...) are compiler-internal and may change;
the snippets are equivalent sketches, not exact decompiler output.

## 6. Performance and allocation
Runtime cost, allocations, boxing, JIT behaviour. "None" is a valid answer.

## 7. Pitfalls
Mistakes that compile but do the wrong thing, or surprise readers.

## 8. Concerns and limitations
Design-level concerns: API and binary compatibility, versioning, readability, tooling,
serialization, reflection, engine (Godot) boundaries.

## 9. When to use: clean-code guidance
### 9.1 Use when
### 9.2 Avoid when
### 9.3 Good versus bad
A short pair of snippets: idiomatic use and misuse.

## 10. Related features
Links to other chapters in this repository.

## 11. References
Official documentation (the `source` links) and the language proposal where useful.
````

---

## Short form (pointer chapters)

Used only for files that point to another chapter (for example a C# 10 preview feature that became stable in C# 11).

````markdown
# <Feature> (preview in C# n)

> **Abstract.** ...

| fact table |

## 1. What this means for a C# n project
## 2. Opting in (and why you usually should not)
## 3. Code without the feature (what to write instead)
## 4. Migration once the project reaches C# m
## 5. References
````

---

## Writing rules

- English, neutral, no project-specific content. Godot notes are allowed where the engine boundary matters.
- Every snippet must compile on the stated version (assume the usual `using` directives).
- Mark guesses as such; never invent compiler output. If unsure about exact IL, show lowered C# instead.
- Link the official documentation in `source`; if this file and the docs disagree, the docs win.
