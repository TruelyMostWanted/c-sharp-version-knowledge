# AGENTS.md

Guide for AI agents (and humans) using or changing this repository.

## Using it

- This repository is a reference, not code: one Markdown file per C# language feature, from C# 10 (.NET 6) to C# 15 (.NET 11).
- Always start with [`README.md`](README.md), step 1: determine the C# version the project allows (`.csproj`, `LangVersion`, `TargetFramework`). Never use a feature above that ceiling.
- To modernize code, follow [`MODERNIZE.md`](MODERNIZE.md).
- Read only the feature files you need (`<version>/<feature>.md`); the frontmatter (`version`, `status`, `tags`, `modernizes`) tells you whether a file is relevant before reading it.
- Each file is a chapter with fixed sections ([`TEMPLATE.md`](TEMPLATE.md)). Start with the abstract and fact table; open further sections only when needed.
- If a file and the official documentation on learn.microsoft.com disagree, the documentation wins.

## As upstream

Games of Truely Interactive embed this repository through the chain
`c-sharp-version-knowledge` → `godot-file-system` → `godot-core-systems` → games,
at `.agents/knowledge/csharp/` (`git subtree`). Change the files here, not in the downstream copies;
downstream repositories pull with `git subtree pull --prefix=.agents/knowledge/csharp <url> main --squash`.

## Changing it

- One feature per file, with frontmatter and a `source` link to the official documentation.
- Follow the chapter structure in [`TEMPLATE.md`](TEMPLATE.md): same sections in the same order; a section that does not apply says so in one sentence.
- Code examples must compile on the chapter's C# version (no newer features in older chapters).
- Do not invent compiler output: show lowered C# sketches, and IL only where it is certain.
- New C# version: new folder `<version>/`, a row in the README table, entries in `MODERNIZE.md`.
- No project-specific content; Godot notes only where the engine boundary matters.
- Commits in English.
