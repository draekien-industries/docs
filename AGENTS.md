# draekien-industries/docs

## Purpose

Public documentation for the Waystone family of .NET libraries (`waystone.monads`, `waystone.widelogevents`), published via GitBook for developers consuming those NuGet packages. This repo holds no code — only the markdown content GitBook renders.

A space is not one per NuGet package. `waystone.monads` documents the core library and every companion package that ships alongside it, under *Companion Packages*. `waystone.monads.fluentvalidation` had a space of its own until 7.0.0 and no longer does — see [ADR 0004](docs/adr/0004-companion-packages-live-in-the-core-space.md).

## Setup

Run this once per clone, or the hook below never fires:

```
git config core.hooksPath .githooks
```

`pre-push` checks that no generated code block has drifted from the source it quotes.
It needs a checkout of
[draekien-industries/waystone-dotnet](https://github.com/draekien-industries/waystone-dotnet)
and the .NET SDK; missing either is a warning rather than a failed push, because this
repository holds no code and a writer may have neither.

## Conventions

- Each top-level directory is exactly one GitBook space, always. Never nest multiple spaces under one directory, and never create a new top-level directory as a way to create a new space — the space must exist in GitBook first (see Gotchas).
- Every space has a `README.md` (its landing/welcome page) and a `SUMMARY.md` (its page tree). Content files live in subdirectories matching the structure declared in `SUMMARY.md`.
- Edit in git by hand. This covers wording, examples, fixing a page, adding a page, and reorganising a page tree. Commit and push as normal, and GitBook's bidirectional Git Sync mirrors it in. When you add or move a page, update that space's `SUMMARY.md` in the same commit — see Gotchas.
- Use the GitBook MCP change-request flow (`create_change_request` → `invoke_operation` → `submit_or_merge_change_request`) for one thing only: creating a new space. Everything else is a hand edit.
- Use the `output-styles:plain-language` skill when writing or editing documentation content in this repo.
- Compile code samples rather than reading them. Both samples on the Waystone.Monads configuration page failed against 6.x — one used a delegate arity that never existed, the other assigned internal properties — and neither was visible to inspection. A scratch file under `waystone-dotnet/sample/` proves it in seconds.
- Commit messages follow `docs(<scope>): <subject>`, e.g. `docs(waystone-monads-28): fix links`. The scope is the identifier of the content unit being changed, not a free-form area name. Commits with `No subject` were authored from the GitBook UI editor, not typed by hand — that's expected, not a defect to fix.

## Generated code blocks

Some code blocks are not written here. They are read out of the sample projects in
`waystone-dotnet` that compile them, so a sample exists once rather than twice.

A generated block looks like this, and **everything between the two comments is
overwritten**:

```
<!-- snippet: configuration-the-usual-call -->
<!-- source: sample/Waystone.Monads.Docs/... -->
<code block>
<!-- endSnippet -->
```

To change one, edit the `#region configuration-the-usual-call` block in the file the
`source` comment names, then run
`dotnet run --project tools/Waystone.DocSnippets` in that repository. Editing the page
instead looks like it worked and is undone by the next run.

A slot inside a fenced code block is left alone, which is why the example above is not
itself filled in.

### Converting a page is part of editing it

**Editing a page that still holds a hand-written C# block converts that block.** Do
not leave one behind and do not convert the whole space in a sweep — the pages move
across one edit at a time, so the work lands next to a reviewer who is already
reading that page.

The steps, in `waystone-dotnet`:

1. Find the sample project that page belongs to. Most are under
   `sample/Waystone.Monads.Docs/`; the analyzer, hosting, configuration and
   observability pages quote the runnable projects directly under `sample/`.
2. Add the block's code there so it compiles, wrapped in
   `#region <page-slug>-<what-it-shows>`. Lower case, hyphens only. The name is
   published, so it has to read as a description.
3. Replace the page's fenced block with `<!-- snippet: key -->` and
   `<!-- endSnippet -->` on their own lines, nothing between them.
4. Build the sample project, then run `dotnet run --project tools/Waystone.DocSnippets`.
5. Read the diff here. Only the two comment lines and the fence should be new. Any
   other change means the sample and the page disagreed — the compiling sample wins,
   but look at it before accepting it.

This applies to C# only. `diff`, `ini`, `jsonc`, `xml` and shell blocks stay
hand-written, and so do the `upgrading/` pages, whose samples are older majors that
no longer compile against the current source.

Converted so far: `guides/configuration.md`. See
[docs/explorations/2026-08-31-single-sourcing-code-samples.md](docs/explorations/2026-08-31-single-sourcing-code-samples.md).

## Gotchas

- If you add a new markdown file directly via git, GitBook will not pick it up until it's also added to that space's `SUMMARY.md` — GitBook does not regenerate `SUMMARY.md` from directory contents on this side of the sync. Manually edited `SUMMARY.md` entries only take effect on the next sync.
- A new top-level directory will not create a new GitBook space by itself. The space has to be created and Git Sync–wired in GitBook first; only then does its directory start appearing here.

## Documentation

Agent-facing documentation lives in `docs/`. Read [docs/AGENTS.md](docs/AGENTS.md) before reading or writing anything there.
