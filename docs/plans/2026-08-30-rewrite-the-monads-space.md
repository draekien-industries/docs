---
title: Rewrite the waystone.monads space around reader intent
date: 2026-08-30
status: active
supersedes:
---

# Rewrite the waystone.monads space around reader intent

Tracked by [DRA-161](https://linear.app/draekien-industries/issue/DRA-161). This
file is the target the whole stack moves toward. Read it before writing a page.

## Goal

The `waystone.monads` space grew a page at a time and now has three problems.

- **"Using the Library" is a catch-all.** Eleven pages sit in it with nothing in
  common beyond "not getting started".
- **Pages overlap.** `core-concepts/options.md` and
  `using-the-library/option-of-t/README.md` both explain `Option<T>`, and a
  reader cannot tell which one to open. Same for `Result<T, E>`.
- **Two pages are unreadable at their size.**
  `using-the-library/core-functionality.md` is 940 lines and mixes concept,
  tutorial and API lookup. `using-the-library/analyzer-rules.md` is 964 lines.

Regroup the space by what the reader came to do, split the oversized pages, and
delete the duplication. One space, v7 only.

## Decisions this plan assumes

| Decision | Choice |
| --- | --- |
| Versioning | One space, v7 only. No GitBook variants, no version dropdown. Older majors stay covered by the upgrade guides. |
| Grouping | By reader intent: Start here, Guides, Reference, Analyzers, Source generation, Companion packages, Upgrading. |
| Depth | Full rewrite. Every page reshaped for its group, in plain language, with samples compiled against v7. |
| Old URLs | Redirects. A `.gitbook.yaml` maps every old path to its new home. |

`Companion packages` keeps its current name rather than shortening to
`Packages`. It is the term [ADR 0004](../adr/0004-companion-packages-live-in-the-core-space.md)
and the repository `AGENTS.md` already use, and it says the one thing `Packages`
does not — that the core package is documented elsewhere in the space. The
directory is `packages/`, because a group name and a URL segment do not have to
match and the shorter segment reads better in a link.

## Page naming rules

Apply these to any page added later, not only the ones in the table below.

1. **Sentence case.** Only the first word is capitalised. `Side effects`, not
   `Side Effects`.
2. **A proper noun keeps its own casing.** `Shouldly`, `FluentValidation`,
   `System.Text.Json`, `Newtonsoft.Json`, `LINQ`, `Rust`.
3. **A C# type is written as code, exactly as declared.** `Option<T>`,
   `Result<T, E>`.
4. **Name the topic, not the task.** No `Working with…`, no `How to…`. A reader
   scanning a sidebar is looking for a noun.
5. **A page about a diagnostic tier names the prefix in brackets.**
   `Runtime bugs (WM1xxx)`. The prefix is what the reader searched for.
6. **`API` is a suffix used in exactly two places** — `Option<T> API` and
   `Result<T, E> API` in Reference. Those are the only names that would
   otherwise repeat a guide.
7. **A package page is named for what the package integrates with**, not for the
   package id's tail. `Dependency injection`, not `Extensions.DependencyInjection`.
8. **Every group's landing page is called `Overview`.**
9. **File and directory names are kebab-case.** `fluent-validation.md`, not
   `fluentvalidation.md`. `README.md` and `SUMMARY.md` are the only exceptions —
   GitBook requires those exact names.

## Page ordering rules

Naming decides what a page is called. These decide where it sits. Each layer
writes its own `SUMMARY.md` entries, so the order has to be followed by whoever
writes that layer.

1. **`Overview` first**, in any group that has one.
2. **Where the material has its own order, follow it** — a value lifecycle, a
   numeric sequence, a version sequence.
3. **Otherwise, most-opened page first.**
4. **A page for a minority of readers goes last**, however short it is.
5. **Groups run: what every reader gets, then what they opt into, then what they
   visit once.**

Rule 5 is why `Companion packages` sits after `Analyzers` and
`Source generation`. Analyzers ship with the core package — nobody chooses them.
A companion package is a choice. `Upgrading` is last because it is a one-time
visit.

## Drop the pre-release blocks

`7.0.0` is stable. Nothing in the space should still tell a reader to ask NuGet
for a pre-release. **Every layer removes these from the pages it touches.** It is
not a separate layer — most of these pages are being rewritten anyway, and a
sweep afterwards would only find what the rewrites missed.

Three things to remove:

1. **The `{% hint style="warning" %}` block** that opens 20 pages with *"This
   page describes `7.0.0-beta.x`, a pre-release."* Delete the whole block, not
   just the version string.
2. **`--prerelease` on an install command.**
   `dotnet add package Waystone.Monads --prerelease` becomes
   `dotnet add package Waystone.Monads`.
3. **A pinned pre-release version.**
   `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />` and
   the sentence introducing it both go.

One judgement call rather than a mechanical one:
`companion-packages/dependency-injection.md` carries a note that *"Earlier
`7.0.0` pre-releases resolved `ILoggerFactory` at install themselves."* That
describes a change between two betas, not between two releases. Nobody upgrading
from `6.x` needs it, so the companion packages layer deletes it.

By the time the redirects layer closes the stack,
`git grep -n 'prerelease\|pre-release\|beta' waystone.monads` returns nothing.

## Target page tree

Sibling order below is the order `SUMMARY.md` lists them in. It is part of the
target, not an accident of how the table was typed.

```
Start here
  Welcome                         README.md
  Quickstart                      start-here/quickstart.md
  Why monads                      start-here/why-monads.md
  Agent skills                    start-here/agent-skills.md

Guides                            (flat)
  Option<T>                       guides/option.md
  Result<T, E>                    guides/result.md
  Errors                          guides/errors.md
  Exceptions                      guides/exceptions.md
  Async                           guides/async.md
  Configuration                   guides/configuration.md
  Observability                   guides/observability.md
  Coming from Rust                guides/coming-from-rust.md

Reference
  Option<T> API                   reference/option/README.md
    Creation                      reference/option/creation.md
    Transform                     reference/option/transform.md
    Consume                       reference/option/consume.md
    Side effects                  reference/option/side-effects.md
    Nesting and conversion        reference/option/nesting.md
  Result<T, E> API                reference/result/README.md
    Creation                      reference/result/creation.md
    Transform                     reference/result/transform.md
    Consume                       reference/result/consume.md
    Side effects                  reference/result/side-effects.md
    Nesting and conversion        reference/result/nesting.md
  State overloads                 reference/state-overloads.md

Analyzers                         (flat)
  Overview                        analyzers/README.md
  Runtime bugs (WM1xxx)           analyzers/runtime-bugs.md
  Idioms (WM2xxx)                 analyzers/idioms.md
  Migration aids (WM3xxx)         analyzers/migration-aids.md
  Assertion rules (WMSxxxx)       analyzers/assertion-rules.md
  Severity presets                analyzers/severity-presets.md

Source generation                 (flat)
  Overview                        source-generation/README.md
  Error code catalogs             source-generation/error-code-catalogs.md
  Code format language            source-generation/code-format.md
  Reviewing generated codes       source-generation/reviewing-codes.md
  Generator diagnostics (WMGxxxx) source-generation/diagnostics.md

Companion packages                (flat)
  Overview                        packages/README.md
  Shouldly                        packages/shouldly.md
  LINQ                            packages/linq.md
  Dependency injection            packages/dependency-injection.md
  Hosting                         packages/hosting.md
  Logging                         packages/logging.md
  FluentValidation                packages/fluent-validation.md
  System.Text.Json                packages/system-text-json.md
  Newtonsoft.Json                 packages/newtonsoft-json.md

Upgrading
  Overview                        upgrading/README.md
  Upgrade to v7                   upgrading/v7/README.md
    Breaking changes              upgrading/v7/breaking-changes.md
    From v6.x                     upgrading/v7/from-v6.md
    From v5.x                     upgrading/v7/from-v5.md
  Deprecations                    upgrading/deprecations.md
  Older upgrades                  upgrading/older/README.md
    v5.x to v6.x                  upgrading/older/v5-to-v6.md
    v4.x to v5.x                  upgrading/older/v4-to-v5.md
    v3.x to v4.x                  upgrading/older/v3-to-v4.md
    v2.x to v3.x                  upgrading/older/v2-to-v3.md
    v1.x to v2.x                  upgrading/older/v1-to-v2.md
```

### Nesting

Child pages are used where a group has more entries than a reader can scan, or
where one page would run past roughly 400 lines. Two places qualify.

- **Reference API pages.** The Option and Result material together runs about
  1,750 lines. Split by category, each child lands near 150-200 lines and the
  parent `README.md` is a one-screen index.
- **Upgrading.** `Upgrade to v7` holds what nearly every reader needs;
  `Older upgrades` collapses the five historical hops out of the way.

Five groups deliberately stay flat.

- **Guides.** Each guide is meant to be read start to finish. Nesting one splits
  a single narrative across two clicks.
- **Start here.** Four pages. Nesting hides nothing worth hiding.
- **Companion packages.** Nine pages, none over 240 lines. Nesting the eight
  under `Overview` would make the overview look like a container, but a reader
  picks a package from its table and never returns to it. It is a signpost.
- **Analyzers.** Six pages. The group is the container; a second level is one
  click too many for someone chasing a single rule id.
- **Source generation.** Five pages, all short.

### Ordering notes

- **Guides.** `Errors` and `Exceptions` sit beside `Result<T, E>`. An `Error` is
  what a `Result` carries, so splitting them with `Async` would put a reader's
  next question two pages away. `Coming from Rust` is a minority page and goes
  last by rule 4.
- **Upgrading.** `Deprecations` sits below `Upgrade to v7`. Far more readers
  want the v7 path than want the deprecation list. It is still one click from
  the group root.
- **Older upgrades.** Newest-first. Rule 2 says follow the version sequence and
  rule 3 says which end to start from — there are far more readers on v5 than on
  v1.
- **Start here.** `Quickstart` stays ahead of `Why monads`. Most arrivals want to
  run something before they want the theory.
- **Upgrade to v7.** `Breaking changes` leads. It answers "how bad is this",
  which is the question a reader has before they pick a path.

## Mapping table

Every page in today's `SUMMARY.md` has a row. This table is the input to the
redirects layer, so keep it here rather than in a commit message.

The anchor column matters because splitting a page moves its anchors to
different pages. `analyzer-rules.md#wm1001` and `analyzer-rules.md#wm2017` do
not land in the same place. A redirect can only point at one target, so a page
that fans out records a **primary** destination — marked **P** — and the rest
are reached by the link sweep, not by a redirect.

### Start here

| Old path | Anchor | New path | New name |
| --- | --- | --- | --- |
| `README.md` | whole page | `README.md` | Welcome |
| `getting-started/quickstart.md` | whole page | `start-here/quickstart.md` | Quickstart |
| `core-concepts/monads.md` | whole page | `start-here/why-monads.md` | Why monads |
| `getting-started/agent-skills.md` | whole page | `start-here/agent-skills.md` | Agent skills |

### Guides

| Old path | Anchor | New path | New name |
| --- | --- | --- | --- |
| `core-concepts/options.md` | whole page | `guides/option.md` **P** | Option\<T> |
| `using-the-library/option-of-t/README.md` | `#printing-and-logging`, `#control-flow`, `#transform`, `#logical-operators` | `guides/option.md` **P** | Option\<T> |
| `using-the-library/option-of-t/README.md` | `#collections` and its nine children | `reference/option/*` | — |
| `core-concepts/results.md` | whole page | `guides/result.md` **P** | Result\<T, E> |
| `using-the-library/result-of-t-and-e.md` | `#what-a-result-can-hold`, `#printing-and-logging`, `#control-flow`, `#transform`, `#consume`, `#side-effect`, `#logical-operators` | `guides/result.md` **P** | Result\<T, E> |
| `using-the-library/result-of-t-and-e.md` | `#collections` and its six children | `reference/result/*` | — |
| `using-the-library/errors-and-exceptions.md` | `#built-in-error-types`, `#errorcode`, `#static-error-codes-recommended`, `#error-code-from-enum`, `#error`, `#error-from-enum` | `guides/errors.md` **P** | Errors |
| `using-the-library/errors-and-exceptions.md` | `#custom-exceptions`, `#unwrapexception`, `#unmetexpectationexception`, `#error-code-from-exception`, `#error-from-exception` | `guides/exceptions.md` | Exceptions |
| `using-the-library/async.md` | whole page | `guides/async.md` | Async |
| `using-the-library/configuration.md` | whole page | `guides/configuration.md` | Configuration |
| `using-the-library/observability.md` | `#count-handled-exceptions`, `#subscribe-to-an-event`, `#watching-for-a-scope-disposed-out-of-order`, `#watching-for-configuration-that-was-never-installed`, `#what-the-library-does-not-report`, `#you-pay-nothing-when-nobody-is-listening`, `#these-names-are-a-contract` | `guides/observability.md` **P** | Observability |
| `using-the-library/observability.md` | `#log-handled-exceptions` and its four children, `#replacing-useexceptionlogger` | `packages/logging.md` | Logging |
| `using-the-library/coming-from-rust.md` | whole page | `guides/coming-from-rust.md` | Coming from Rust |

### Reference

`using-the-library/core-functionality.md` fans out across eleven pages. Each
category exists twice — once for `Option<T>`, once for `Result<T, E>` — because
the old page covered both types under one heading. The primary destination is
`reference/option/README.md`, since a reader landing on the old URL is more
often looking for `Option<T>`.

| Old anchor | New path |
| --- | --- |
| `#introduction` | `reference/option/README.md` **P** and `reference/result/README.md` |
| `#creation`, `#passing-state-to-the-factory` | `reference/option/creation.md`, `reference/result/creation.md` |
| `#transform`, `#map`, `#more-transforms` | `reference/option/transform.md`, `reference/result/transform.md` |
| `#state-overloads` and its six children | `reference/state-overloads.md` |
| `#state-checks` | `reference/option/consume.md`, `reference/result/consume.md` |
| `#consume`, `#match`, `#pattern-matching-with-deconstruct`, `#where-match-still-wins`, `#unwrap`, `#unwrapor`, `#unwraporelse`, `#unwrapordefault`, `#unwrapornull`, `#expect`, `#more-consumes` | `reference/option/consume.md`, `reference/result/consume.md` |
| `#transform-and-consume`, `#mapor`, `#maporelse`, `#mapordefault`, `#mapornull` | `reference/option/consume.md`, `reference/result/consume.md` |
| `#side-effect`, `#inspect`, `#more-side-effects` | `reference/option/side-effects.md`, `reference/result/side-effects.md` |
| `#nesting`, `#flatten`, `#transpose`, `#conversion`, `#okor`, `#okorelse`, `#getok`, `#geterr` | `reference/option/nesting.md`, `reference/result/nesting.md` |

`Transform and Consume` lands on `consume.md`, not `transform.md`. `MapOr` and
its siblings return a raw value, so they end the chain — which is what the
Consume page is for.

### Analyzers

`using-the-library/analyzer-rules.md` fans out across six pages. Primary is
`analyzers/README.md`.

| Old anchor | New path |
| --- | --- |
| `#what-this-page-is-for` | `analyzers/README.md` **P** |
| `#runtime-bugs`, `#wm1001`, `#wm1002`, `#wm1003`, `#wm1005`, `#wm1006`, `#wm1008`, `#wm1011` | `analyzers/runtime-bugs.md` |
| `#idioms`, `#wm2001` through `#wm2022` | `analyzers/idioms.md` |
| `#migration-aids`, `#wm3001`, `#wm3002` | `analyzers/migration-aids.md` |
| `#assertion-rules`, `#wms2001`, `#wms2002` | `analyzers/assertion-rules.md` |
| `#changing-a-rule` | `analyzers/severity-presets.md` |

| Old path | Anchor | New path | New name |
| --- | --- | --- | --- |
| `using-the-library/severity-presets.md` | whole page | `analyzers/severity-presets.md` | Severity presets |

Anchors survive the move. `#wm2017` on the old page is still `#wm2017` on
`analyzers/idioms.md`, so a redirect that carries the fragment lands correctly.

### Source generation

`using-the-library/generated-error-codes.md` fans out across five pages. Primary
is `source-generation/README.md`.

| Old anchor | New path |
| --- | --- |
| `#what-this-page-is-for` | `source-generation/README.md` **P** |
| `#marking-an-enum`, `#what-you-get`, `#a-value-that-is-not-a-declared-member` | `source-generation/error-code-catalogs.md` |
| `#choosing-the-code-format`, `#the-format-language`, `#one-format-for-the-whole-assembly`, `#the-format-is-your-contract-not-your-factory` | `source-generation/code-format.md` |
| `#reviewing-your-codes-as-a-list`, `#making-a-divergence-fail-the-build`, `#renaming-is-a-breaking-change`, `#reusing-a-code-across-two-enums` | `source-generation/reviewing-codes.md` |
| `#diagnostics`, `#wmg0001` through `#wmg0006` | `source-generation/diagnostics.md` |

### Companion packages

| Old path | Anchor | New path | New name |
| --- | --- | --- | --- |
| `companion-packages/README.md` | whole page | `packages/README.md` | Overview |
| `companion-packages/shouldly.md` | whole page | `packages/shouldly.md` | Shouldly |
| `companion-packages/linq.md` | whole page | `packages/linq.md` | LINQ |
| `companion-packages/dependency-injection.md` | whole page | `packages/dependency-injection.md` | Dependency injection |
| `companion-packages/hosting.md` | whole page | `packages/hosting.md` | Hosting |
| `companion-packages/fluentvalidation.md` | whole page | `packages/fluent-validation.md` | FluentValidation |
| `companion-packages/systemtextjson.md` | whole page | `packages/system-text-json.md` | System.Text.Json |
| `companion-packages/newtonsoftjson.md` | whole page | `packages/newtonsoft-json.md` | Newtonsoft.Json |
| new | — | `packages/logging.md` | Logging |

`packages/logging.md` is the only page in this group with no old path. It does
inherit content — the logging half of
`using-the-library/observability.md`, recorded in the Guides table above — so
the redirects layer skips it rather than hunting for a source of its own.

The two JSON pages were written and merged
([docs#43](https://github.com/draekien-industries/docs/pull/43)) after the
issues behind this rewrite were filed, so they are ordinary moves rather than
new pages. They need a rewrite for group shape and a kebab-case rename, not a
first draft.

Rule 9 forces three renames: `fluentvalidation.md`, `systemtextjson.md` and
`newtonsoftjson.md`. Every other file already reads as kebab-case. Use
`git mv` for all three so the history follows.

### Upgrading

| Old path | Anchor | New path | New name |
| --- | --- | --- | --- |
| `upgrading-and-deprecations/README.md` | whole page | `upgrading/README.md` | Overview |
| `upgrading-and-deprecations/v7-breaks.md` | whole page | `upgrading/v7/breaking-changes.md` | Breaking changes |
| `upgrading-and-deprecations/v6-to-v7.md` | whole page | `upgrading/v7/from-v6.md` | From v6.x |
| `upgrading-and-deprecations/v5-to-v7.md` | whole page | `upgrading/v7/from-v5.md` | From v5.x |
| `upgrading-and-deprecations/deprecations.md` | whole page | `upgrading/deprecations.md` | Deprecations |
| `upgrading-and-deprecations/v5-to-v6.md` | whole page | `upgrading/older/v5-to-v6.md` | v5.x to v6.x |
| `upgrading-and-deprecations/v4-to-v5.md` | whole page | `upgrading/older/v4-to-v5.md` | v4.x to v5.x |
| `upgrading-and-deprecations/v3-to-v4.md` | whole page | `upgrading/older/v3-to-v4.md` | v3.x to v4.x |
| `upgrading-and-deprecations/v2-to-v3.md` | whole page | `upgrading/older/v2-to-v3.md` | v2.x to v3.x |
| `upgrading-and-deprecations/v1-to-v2.md` | whole page | `upgrading/older/v1-to-v2.md` | v1.x to v2.x |
| `upgrading-and-deprecations/v7-agent-prompt.md` | `#rules`, `#what-it-cannot-do-for-you`, `#if-you-would-rather-not-use-an-agent` | `upgrading/README.md` **P** | Overview |
| `upgrading-and-deprecations/v7-agent-prompt.md` | `#the-prompt` and its six steps | a collapsible block at the top of each of the seven upgrade pages | — |

`Upgrade with an agent` stops being a page. A reader who has landed on
`From v6.x` and wants an agent to do the work should not have to find a second
page and work out which parts apply to their hop. Every upgrade page instead
opens with a collapsible block holding the prompt for **that** hop, collapsed by
default so a reader doing it by hand scrolls past one line. The material that is
not hop-specific moves to `upgrading/README.md` and is said once. The older hops
get a short prompt derived from their own break list — `v4-to-v5.md` is 15 lines
and its prompt should be proportionate.

`upgrading/older/README.md` is a new page with no old path: a one-screen index
for the five historical hops.

### Every package accounted for

Every project under `waystone-dotnet/src/` is either assigned a page or excluded
here with a reason. No standing exclusion is left.

| Project | Page |
| --- | --- |
| `Waystone.Monads` | The whole space |
| `Waystone.Monads.Analyzers` | `analyzers/` |
| `Waystone.Monads.Shouldly` | `packages/shouldly.md` |
| `Waystone.Monads.Shouldly.Analyzers` | `analyzers/assertion-rules.md` |
| `Waystone.Monads.Linq` | `packages/linq.md` |
| `Waystone.Monads.Extensions.DependencyInjection` | `packages/dependency-injection.md` |
| `Waystone.Monads.Extensions.Hosting` | `packages/hosting.md` |
| `Waystone.Monads.Extensions.Logging` | `packages/logging.md` |
| `Waystone.Monads.FluentValidation` | `packages/fluent-validation.md` |
| `Waystone.Monads.SystemTextJson` | `packages/system-text-json.md` |
| `Waystone.Monads.NewtonsoftJson` | `packages/newtonsoft-json.md` |
| `Waystone.Monads.SourceGenerators` | `source-generation/` |
| `Waystone.SourceGenerators` | Excluded — see below |
| `Waystone.Monads.EntityFrameworkCore` | Excluded — does not exist |

`Waystone.Monads.EntityFrameworkCore` has a page written for it in
[docs#44](https://github.com/draekien-industries/docs/pull/44), closed unmerged
on 2026-08-30 alongside waystone-dotnet#198. The package is not shipping — the
recommendation is to map into persistence models instead. Do not resurrect that
page. Storing an `Option` in a column worked, but querying one mostly did not,
and one of the failures was silent.

`Waystone.SourceGenerators` is `IsPackable=false` and absent from the
`PackMonadAnalyzers` target, so it never runs on a consumer's compilation and
nobody outside the repository can apply `[GenerateAwaitedReceivers]`.
Documenting it would describe a surface no reader has. The awaited receivers it
produces *are* consumer-facing and stay on the Async guide, described as
ordinary API.

## The layers

One `gh stack` of pull requests against `main`, bottom to top. Each layer
updates `SUMMARY.md` in the same commit as its page moves — GitBook does not
regenerate `SUMMARY.md` from directory contents, so a page added without an
entry never appears.

| # | Issue | Layer |
| --- | --- | --- |
| 1 | DRA-162 | This plan. Nothing moves. |
| 2 | DRA-164 | Start here. |
| 3 | DRA-165 | The `Option<T>` guide, merging three sources. |
| 4 | DRA-166 | The `Result<T, E>` guide, merging three sources. |
| 5 | DRA-167 | Async into Guides; Errors and exceptions split in two. |
| 6 | DRA-168 | Configuration, Observability and Coming from Rust into Guides. |
| 7 | DRA-169 | `core-functionality.md` split into the nested Reference. |
| 8 | DRA-170 | The Analyzers group. |
| 9 | DRA-171 | The Source generation group. |
| 10 | DRA-172 | Companion packages. |
| 11 | DRA-173 | Upgrading. |
| 12 | DRA-174 | Redirects and the final link audit. |

### The sample projects are not a layer

DRA-163 is not in the table, because its deliverable cannot live in this
repository. This repo holds no code — only the markdown GitBook renders — and a
new top-level directory here is an attempt to create a GitBook space, not a
place to put a `.csproj`.

The samples belong in `waystone-dotnet/sample/`, where the pattern already
exists: `Waystone.Monads.Configuration.Sample` and
`Waystone.Monads.Observability.Sample` are each one project for one doc page,
referencing `src/` by `ProjectReference` rather than by `PackageReference`. A
project reference is the better choice anyway — it compiles the samples against
exactly what shipped, with no wait for a NuGet index to catch up.

So DRA-163 is a standalone pull request in `waystone-dotnet`
([#202](https://github.com/draekien-industries/waystone-dotnet/pull/202)), and it
lands before this stack does. It is `sample/Waystone.Monads.Docs/`, nine library
projects with no entry point — the build succeeding is the test.

### Building the samples

```
dotnet build sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Samples.slnx
```

Run it from the `waystone-dotnet` root before opening any layer's PR. The
projects are also in `Waystone.Net.sln`, which is what CI builds; the `.slnx` is
a convenience for working on one page without building the test matrix.

**One project per install list, not one project referencing everything.** A page
tells a reader which package to add. If every sample compiled in one project
that referenced all nine, a page could quietly depend on a package its install
section never mentions and the build would stay green.

**One file per page**, named for the page, with the page path in a `<summary>`
on the type. When a build breaks, the file name is the page to go and fix. The
core project mirrors the page tree in folders.

`TreatWarningsAsErrors` is on, so an obsolete API in a sample fails the build
rather than scrolling past in a log.

### The samples are D&D-themed

Every type and value is tabletop fantasy — characters, quests, spells, rituals,
dice. The published pages already lean that way (`Grog`, `Keyleth`, `Thordak`,
`Pike`, `Vex'ahlia`, `The Raven Queen`), and the sample projects now match.

It is not decoration. `Order`, `UserInput` and `Person` are the shapes every
sample in every library uses, so a reader skims them. A `Quest` with a
`GoldReward`, or a `SpellInput` with a `Range`, forces them to look — and
`Range` is a real constraint on a real spell in a way `UserInput.Range` never
was. Reuse the cast when writing a page rather than inventing one.

### Eight page bugs it found before the rewrite started

Every one reads fine and none was visible to inspection. Each is recorded on the
Linear issue for the layer that rewrites its page.

| Page | Bug |
| --- | --- |
| `fluentvalidation.md` | Chains `UseValidationErrorCode` after `UseLogger`, which ships in a package the install section never names |
| `systemtextjson.md`, `newtonsoftjson.md` | Both packages shadow the serializer namespaces; this plan first asserted the opposite |
| `newtonsoftjson.md` | Its converters are not generic; the System.Text.Json ones are |
| `async.md` | `UnwrapOrAsync` takes a value, not a factory |
| `async.md` | `UnwrapOrDefaultAsync` returns `T?`, so the page's assignment does not compile under nullable |
| `async.md` | `MapAsync` never takes a `ValueTask` factory |
| `errors-and-exceptions.md` | Declares `Error error1` twice in one block |
| `core-functionality.md` | Annotates `UnwrapOrDefault` on a `Result` as `int?`, then says the value is `0` |

`quickstart.md` has a ninth: its namespace table calls
`Waystone.Monads.Options.Extensions` the home of "the `Async` extensions", but
`UnwrapOrNull`, `MapOrNull`, `Transpose` and the collection surface are all
synchronous and all live there too.

Two blocks in the space cannot be pinned at all, because the page shows them
*because* they fail: the `var (a, b) = maybeName;` deconstruction, and the
`MatchAsync` whose `Err` branch is synchronous. Each has a comment in the sample
file saying so rather than being silently dropped.

Two layers share a file and have to land in the same stack.
`packages/logging.md` is **written** by DRA-172 and **removed from the
Observability guide** by DRA-168. Whichever lands second will conflict if the
other is dropped.

## Done when

- Every page in `SUMMARY.md` sits in exactly one of the seven groups, and its
  content matches that group's job.
- Every page name obeys the naming rules, and every file and directory name is
  kebab-case.
- Every group's siblings are listed in the order this plan states.
- Every upgrade page opens with a collapsible agent prompt for its own hop.
- No page explains the same thing as another page.
- Every code sample compiles against Waystone.Monads 7.0.0.
- `git grep -n 'prerelease\|pre-release\|beta' waystone.monads` returns nothing.
- `.gitbook.yaml` redirects every old path in the mapping table, and no old URL
  404s.
- `git grep` finds no link pointing at an old path.

## Does this need an ADR

Yes, for the grouping — and only for the grouping.
[ADR 0005](../adr/0005-group-the-monads-space-by-reader-intent.md) records it.

The grouping clears all three bars in `docs/adr/AGENTS.md`. It is hard to
reverse, because regrouping again means a second round of redirects on URLs
already in the wild and in shipped analyzer help links. It is surprising without
context — a reader will ask why analyzer rules are a top-level group rather than
part of Reference, and why there is no version dropdown. And it is a real
trade-off: GitBook content variants were on the table and were rejected.

Nothing else here clears the bar. The naming and ordering rules are conventions,
cheap to change and recorded in this file. The layer split is a work plan, not a
decision. Neither gets an ADR.

## Open

Nothing blocking. One call was made here rather than deferred: the group keeps
the name `Companion packages` rather than shortening to `Packages`, for the
reason given at the top of this file.

## Analyzer anchor mapping

DRA-170 split `using-the-library/analyzer-rules.md` into five pages. Every anchor
below was a live URL, and analyzer build output in 6.x links to some of them.
DRA-174 turns this into `.gitbook.yaml` redirects, and DRA-177 uses the same
right-hand column to set `helpLinkUri` on each descriptor for the first time.

| Old URL | New URL |
| --- | --- |
| `using-the-library/analyzer-rules` | `analyzers` |
| `using-the-library/analyzer-rules#what-this-page-is-for` | `analyzers#what-this-page-is-for` |
| `using-the-library/analyzer-rules#changing-a-rule` | `analyzers#changing-a-rule` |
| `using-the-library/analyzer-rules#runtime-bugs` | `analyzers/runtime-bugs` |
| `using-the-library/analyzer-rules#wm1001` … `#wm1011` | `analyzers/runtime-bugs#wm1001` … `#wm1011` |
| `using-the-library/analyzer-rules#idioms` | `analyzers/idioms` |
| `using-the-library/analyzer-rules#wm2001` … `#wm2022` | `analyzers/idioms#wm2001` … `#wm2022` |
| `using-the-library/analyzer-rules#migration-aids` | `analyzers/migration-aids` |
| `using-the-library/analyzer-rules#wm3001`, `#wm3002` | `analyzers/migration-aids#wm3001`, `#wm3002` |
| `using-the-library/analyzer-rules#assertion-rules` | `analyzers/assertion-rules` |
| `using-the-library/analyzer-rules#wms2001`, `#wms2002` | `analyzers/assertion-rules#wms2001`, `#wms2002` |
| `using-the-library/severity-presets` | `analyzers/severity-presets` |

A redirect carries no fragment, so GitBook drops the `#wmXXXX` part. One redirect
per rule id is therefore not possible from `.gitbook.yaml` alone — the four
tier-level redirects above are what DRA-174 can write, and a reader following an
old rule link lands on the right page rather than the right heading. DRA-177 fixes
that properly by pointing `helpLinkUri` at the full new URL, so 7.x build output
never uses an old anchor.

### Rule ids that do not exist

Checked against `src/Waystone.Monads.Analyzers/Rules.cs` and
`AnalyzerReleases.Shipped.md` on the v7 branch, not assumed:

* `WM1004`, `WM1007`, `WM1009`, `WM1010`, `WM2014` — shipped in 5.x, removed in 6.0.0.
* `WM2010` — shipped in 5.3.0, removed in 7.0.0. Its anchor is kept on
  `analyzers/idioms` because 6.x build output links to it.

`WMG0001`–`WMG0006` stay with DRA-171. They are generator diagnostics, not
analyzer rules, and they belong on the Source generation group's pages.

## Source generation anchor mapping

DRA-171 split `using-the-library/generated-error-codes.md` across five pages.
Unlike the analyzer split, the old page's anchors do not all land on one new page,
so a single redirect cannot serve them. DRA-174 redirects the page URL to the
group overview and accepts that a reader following an old fragment lands on the
overview's chooser table.

| Old anchor | New URL |
| --- | --- |
| *(none)* | `source-generation` |
| `#what-this-page-is-for` | `source-generation#why-it-exists` |
| `#marking-an-enum` | `source-generation#marking-an-enum` |
| `#what-you-get` | `source-generation/error-code-catalogs#what-you-get` |
| `#a-value-that-is-not-a-declared-member` | `source-generation/error-code-catalogs#a-value-that-is-not-a-declared-member` |
| `#reusing-a-code-across-two-enums` | `source-generation/error-code-catalogs#reusing-a-code-across-two-enums` |
| `#choosing-the-code-format` | `source-generation/code-format#choosing-the-code-format` |
| `#the-format-language` | `source-generation/code-format#the-format-language` |
| `#one-format-for-the-whole-assembly` | `source-generation/code-format#one-format-for-the-whole-assembly` |
| `#the-format-is-your-contract-not-your-factory` | `source-generation/code-format#the-format-is-your-contract-not-your-factory` |
| `#reviewing-your-codes-as-a-list` | `source-generation/reviewing-codes#reviewing-your-codes-as-a-list` |
| `#making-a-divergence-fail-the-build` | `source-generation/reviewing-codes#making-a-divergence-fail-the-build` |
| `#renaming-is-a-breaking-change` | `source-generation/reviewing-codes#renaming-is-a-breaking-change` |
| `#diagnostics` | `source-generation/diagnostics` |
| `#wmg0001` … `#wmg0006` | `source-generation/diagnostics#wmg0001` … `#wmg0006` |

`using-the-library/` no longer exists after this layer. Its three pages went to
`analyzers/` (two) and `source-generation/` (one).

## Link checking

Two scripts were run at the end of DRA-170 and DRA-171, and DRA-174 should run
both again over the finished tree:

1. Every relative `.md` link target exists on disk.
2. Every `#anchor` matches a heading on its target page.

The second reports one standing false positive:
`companion-packages/fluentvalidation.md` links to
`v7-breaks.md#waystone-monads-fluentvalidation`. GitBook turns the dots in
`## Waystone.Monads.FluentValidation` into hyphens; a naive slugifier drops them.
The link is correct.

## Companion packages URL mapping

DRA-172 moved the directory and renamed three files. Every one of these is a live
URL. DRA-174 redirects all nine; none of the anchors moved, and GitBook carries a
fragment across a page-level redirect, so no anchor column is needed here.

| Old URL | New URL |
| --- | --- |
| `companion-packages` | `packages` |
| `companion-packages/shouldly` | `packages/shouldly` |
| `companion-packages/linq` | `packages/linq` |
| `companion-packages/dependency-injection` | `packages/dependency-injection` |
| `companion-packages/hosting` | `packages/hosting` |
| `companion-packages/logging` | `packages/logging` |
| `companion-packages/fluentvalidation` | `packages/fluent-validation` |
| `companion-packages/systemtextjson` | `packages/system-text-json` |
| `companion-packages/newtonsoftjson` | `packages/newtonsoft-json` |

The group is still named `Companion packages` in `SUMMARY.md`, per the decision at
the top of this file. Only the URL segment shortened.

## Upgrading URL mapping

DRA-173 dropped `-and-deprecations` from the directory, nested the v7 and older
material, and deleted one page. Every row is a live URL for DRA-174.

| Old URL | New URL |
| --- | --- |
| `upgrading-and-deprecations` | `upgrading` |
| `upgrading-and-deprecations/deprecations` | `upgrading/deprecations` |
| `upgrading-and-deprecations/v6-to-v7` | `upgrading/v7/from-v6` |
| `upgrading-and-deprecations/v5-to-v7` | `upgrading/v7/from-v5` |
| `upgrading-and-deprecations/v7-breaks` | `upgrading/v7/breaking-changes` |
| `upgrading-and-deprecations/v7-agent-prompt` | `upgrading` |
| `upgrading-and-deprecations/v5-to-v6` | `upgrading/older/v5-to-v6` |
| `upgrading-and-deprecations/v4-to-v5` | `upgrading/older/v4-to-v5` |
| `upgrading-and-deprecations/v3-to-v4` | `upgrading/older/v3-to-v4` |
| `upgrading-and-deprecations/v2-to-v3` | `upgrading/older/v2-to-v3` |
| `upgrading-and-deprecations/v1-to-v2` | `upgrading/older/v1-to-v2` |

`v7-agent-prompt` is the one page with no equivalent. Its prompt was split into the
two v7 pages' collapsible blocks and its general material moved to
`upgrading/README.md`, so the group overview is the closest landing.

Two anchors moved off pages that still exist:
`upgrading-and-deprecations/v5-to-v6#upgrade-with-an-agent` and
`upgrading-and-deprecations/v6-to-v7#upgrade-with-an-agent` are now `<summary>`
elements rather than headings, so neither is an anchor any more. A reader following
one lands at the top of the right page, which is where the block is.

### The collapsible block is unverified

`<details>` / `<summary>` is GitBook's markdown for an expandable block, and the
seven prompts use it with a plain ```` ``` ```` fenced block inside — no nested
fences, which is the case most likely to break. Nothing in this repository used one
before, and there is no way to run a GitBook preview from a local clone, so **this
has not been seen rendered.** Check it on the first preview after DRA-173 merges.

The fallback, if it does not render, is one edit per page: replace the `<details>`
wrapper with a `{% hint %}` linking down to an `## Upgrade with an agent` section
holding the same fenced block. That costs a click and carries no rendering risk.
