---
id: 0005
title: Group the waystone.monads space by reader intent
status: accepted
date: 2026-08-30
deciders: [william-pei]
tags: [documentation, gitbook, information-architecture]
supersedes:
superseded-by:
---

# 0005 — Group the waystone.monads space by reader intent

## Context

The `waystone.monads` space grew a page at a time over six major versions. By
`7.0.0` it had five groups, and one of them — *Using the Library* — held eleven
pages whose only shared property was that they were not getting-started
material. Configuration sat beside a 964-line analyzer rule catalogue.

Three symptoms made the shape untenable.

**Nobody could tell which page to open.** `core-concepts/options.md` and
`using-the-library/option-of-t/README.md` both explained `Option<T>`. The first
was conceptual, the second was API, and nothing in either title said so. The
same pair existed for `Result<T, E>`.

**Two pages were past reading size.** `using-the-library/core-functionality.md`
was 940 lines across 38 headings, covering creation, transformation, consumption
and conversion for both types at once. It was neither readable start to finish
nor scannable for one method. `using-the-library/analyzer-rules.md` was 964
lines holding about 40 rules.

**Growth had no rule to follow.** A new page went wherever it fit, because there
was no statement of what each group was for. Four packages shipped into
`7.0.0` — Logging, FluentValidation, `System.Text.Json`, `Newtonsoft.Json` — and
the group that should have held them was already ending with a paragraph
apologising that one package was not on the list.

A second question arrived with `7.0.0`: whether to keep documentation for older
majors alongside the new. GitBook supports this through content variants, where
each version is a separate space behind a dropdown.

## Decision

We will group the space by what the reader came to do, not by what the library
contains. Seven groups: **Start here**, **Guides**, **Reference**,
**Analyzers**, **Source generation**, **Companion packages**, **Upgrading**. A
page belongs to the group whose reader-question it answers, and the boundary
between "explain this to me" (Guides) and "look this up" (Reference) is the one
that decides where duplicated material lands. We will keep exactly one space,
documenting `7.0.0` only, and cover older majors through the upgrade guides
rather than through a version dropdown.

## Consequences

Every page has one home and the test for that home is a question, not a
category, so a page added in a year has a rule to follow rather than a
precedent to copy.

Duplication becomes visible. Once *Guides* and *Reference* are separate groups,
two pages explaining `Option<T>` are obviously one too many. That is the point,
and it forces a merge that had been avoidable for three majors.

Two groups exist that a smaller space would not need. Analyzers and Source
generation are both build-time concerns and both would sit awkwardly inside
Reference — a reader reaches them because the compiler printed something, not
because they had a question about the API. Splitting them out costs two extra
top-level entries and leaves Reference as one coherent thing.

Every existing URL moves. `.gitbook.yaml` redirects carry the old paths, and
the analyzer descriptors already bake an ID-keyed `wm/<id>` help link rather
than a page path, so shipped binaries survive the move. Anchors are the harder
half: splitting a 964-line rules page means `#wm2017` and `#wm1001` land on
different pages, and only a per-anchor mapping catches that.

Regrouping again would be expensive. That is the cost accepted here — a second
round of redirects on URLs that are by then in shipped binaries, blog posts and
Stack Overflow answers.

Readers on `6.x` lose their documentation. The space documents `7.0.0` and
nothing else, so someone who has not upgraded reads about API they do not have.
The upgrade guides are the mitigation, and they are less good than a version
dropdown would be.

## Alternatives considered

**GitBook content variants, one per major version.** Rejected because
[ADR 0002](0002-one-directory-per-gitbook-space.md) maps one directory to one
space, so every version becomes a full duplicate of the content in this
repository, kept in sync by hand. At roughly 35 pages, two versions means 70
pages and every fix applied twice. The library is small and its upgrade hops are
short; the guides cover the gap at a fraction of the cost.

**Group by package.** One group per NuGet package —
`Waystone.Monads`, `Waystone.Monads.Shouldly`, and so on. Rejected because it
answers the question "what did I install", which no reader asks. It also splits
`Option<T>` across the core group and the LINQ group for no reason a reader can
see, and it puts a two-page package at the same level as the entire core
library.

**Split concept from API only, and leave the rest.** The cheap fix: merge the
duplicated pages, split `core-functionality.md`, and keep the existing five
groups. Rejected because it treats the symptom. *Using the Library* would still
be a catch-all, and the next page added would still have nowhere obvious to go.
The oversized pages were a consequence of having no rule, not the cause.

**Keep analyzers inside Reference.** Rejected because it splits build-time
diagnostics across two places with nothing to distinguish them — `WMG` codes in
one group and `WM` codes in another. An analyzer rule is not API, and a reader
chasing `WM2017` is not reading a reference.
