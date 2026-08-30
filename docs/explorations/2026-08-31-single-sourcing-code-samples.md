---
title: Single-sourcing the code samples from compiled projects
date: 2026-08-31
status: done
supersedes:
---

# Single-sourcing the code samples from compiled projects

For DRA-176. Run against `waystone-dotnet` at `main` (`1f9bd46`) and this repository at
`main` (`2905d3c`), .NET SDK 10.0.111.

## The question

A published sample exists twice: once in the markdown here, once in the project that
proves it compiles in `waystone-dotnet`. Nothing ties the two together, so either can be
edited without the other noticing. DRA-176 proposed
[MarkdownSnippets](https://github.com/SimonCropp/MarkdownSnippets) (`mdsnippets`) and
named three ways to bridge the two repositories. Which one?

## MarkdownSnippets cannot span two repositories

This is the finding that decides everything else, and it is not a matter of taste.

`mdsnippets` takes **one** target directory and walks it for both the snippet sources
and the markdown. There is no separate source root: no `--source` flag, no second path
argument, and no configuration key for one. `mdsnippets.json` carries `ExcludeDirectories`,
`ExcludeMarkdownDirectories` and `ExcludeSnippetDirectories` — three ways to narrow the
one tree, and none to add a second.

`UrlsAsSnippets` reads a snippet from a URL, which would mean pushing the sample before
the page could be filled from it. That inverts the guarantee: the check would pass
against whatever was last pushed rather than what is about to be.

## The samples cannot move here

The first option in the issue was to move the sample projects into this repository. That
would make `mdsnippets` work unchanged, and it is not available.

Every project under `waystone-dotnet/sample/Waystone.Monads.Docs/` takes the library by
`ProjectReference`:

```xml
<ProjectReference Include="..\..\..\src\Waystone.Monads\Waystone.Monads.csproj"/>
```

That is the whole point of them. They compile against the working tree, so a break is
caught in the PR that causes it. A copy here would compile against a published package
and could only ever catch a break after it shipped. `TreatWarningsAsErrors` makes the
same argument for deprecations: an obsolete API fails that build, but only when the
build sees the unreleased source.

## The options that remain

| Option | Verdict |
| -- | -- |
| Run `mdsnippets` in `waystone-dotnet` CI and open a bot PR here | Rejected. Needs a cross-repo token, and the check lands after the push rather than before it |
| Consume `waystone-dotnet` as a submodule here | Rejected. A permanent cost to every clone of a repository that holds no code |
| Stage a copy of the samples into a gitignored folder here on every run, then run `mdsnippets` | Rejected. Reintroduces the copy this work exists to remove, plus a global tool install |
| Read both roots directly from a purpose-built tool | **Taken** |

## What was built

`waystone-dotnet/tools/Waystone.DocSnippets`, run as
`dotnet run --project tools/Waystone.DocSnippets`. It reads `#region <key>` blocks out
of the sample projects and fills `<!-- snippet: key -->` slots in the pages here.
`--check` reports instead of writing, and a `pre-push` hook in each repository runs it.

It is roughly 500 lines including tests, against a global tool install plus a staging
script for the `mdsnippets` route. Neither repository gains a dependency: `dotnet` is
already required in one, and the hook here degrades to a warning when it is missing from
the other.

Three things it does that the staging route could not:

* Names both roots as arguments (`--repo`, `--docs`), so the hook in this repository can
  drive a tool that lives in the other one.
* Finds the other checkout without a path being written down anywhere — an argument, an
  environment variable, `git config`, then a sibling scan.
* Exits `3` rather than `1` when there is no counterpart checkout, so a contributor who
  has cloned only one repository is warned rather than blocked.

## Where the knowledge of the other repository lives

Deliberately in one place each way round, because that is what a reader will look for.

* This repository is recognised by holding a `waystone.monads` directory
  (`Locator.IsDocumentationRepository`).
* That repository is recognised by holding `tools/Waystone.DocSnippets` (this
  repository's `.githooks/pre-push`).

Neither is a path. Renaming a space breaks the first and the error message says so.

## What is proved

`guides/configuration.md` is filled entirely from
`sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Configuration.cs`
— nine of its ten blocks, the tenth being the Logging section, whose methods ship in a
package that project does not reference.

Changing a region without regenerating was tried: `--check` reported
`stale: waystone.monads/guides/configuration.md` and exited `1`.

## What is not done

The remaining pages still carry copies. They move onto slots as each is next edited,
which is what the issue asked for — a sweep of 343 code blocks in one change would be
unreviewable, and most of them are already correct.

## What was rejected and might be asked about again

**A visible "snippet source" link under each block.** `mdsnippets` writes one and it is
genuinely useful. At 343 blocks it is 343 lines of chrome on pages that are already
long. The path is written as an HTML comment instead, so it is in the diff a reviewer
reads and absent from the page a reader sees.

**Wiring the check into CI.** Neither repository can see the other on a runner without a
checkout step and a token, and this repository has no CI at all. The hook was accepted as
sufficient on the understanding that it is not a gate — a clone without
`core.hooksPath` set has no hook, exactly as with every other hook in `waystone-dotnet`.
