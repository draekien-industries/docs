---
id: 0004
title: Document companion packages inside the core space rather than giving each its own
status: accepted
date: 2026-08-29
deciders: [william-pei]
tags: [gitbook, structure]
supersedes:
superseded-by:
---

# 0004 — Document companion packages inside the core space rather than giving each its own

## Context

`Waystone.Monads` ships with five packages beside it: `Shouldly`, `Linq`,
`Extensions.DependencyInjection`, `Extensions.Hosting` and `FluentValidation`. They
share a repository and a version number with the core, and none of them is useful on
its own — every one of them extends or configures `Waystone.Monads`.

`FluentValidation` was the only one with a GitBook space of its own. It got one because
it was the first companion package to exist, before there was a pattern to follow. The
four that came later were documented as pages under *Companion Packages* in the
`waystone.monads` space, because that is where a reader looking for them was already
standing.

That left the reader with two places to look for the same kind of thing, and the
inconsistency was the visible half of the cost. The larger half was that a separate
space cannot cross-link cheaply. A page in the `FluentValidation` space referring to
`Result` had to use an absolute `app.gitbook.com` URL carrying the other space's id,
which is opaque in the source, and which nothing checks. Six pages accumulated a dozen
of them.

7.0.0 rewrote `Waystone.Monads.FluentValidation`, so every page in that space described
an API that no longer existed. The choice was forced at that moment: rewrite the space,
or fold it in.

Note that `Waystone.Monads.Extensions.Logging` was already an exception to the exception
— it has no page under *Companion Packages* at all, because it configures the library
rather than extending it, and lives on *Observability* instead.

## Decision

We will document every companion package as a page inside the `waystone.monads` space,
under *Companion Packages*, and we will not create a GitBook space for a package that
extends or configures another package. A space is reserved for an independently useful
library — today `waystone.monads` and `waystone.widelogevents`.

The `waystone.monads.fluentvalidation` space is removed, and its content is replaced by
`waystone.monads/companion-packages/fluentvalidation.md`.

## Consequences

A reader finds every companion package in one table, on the page they were already
reading. Cross-links between a companion page and the core become ordinary relative
paths, which are visible in the source and break loudly when a file moves.

The published URLs for the old space are gone. Anything linking to them — a blog post, a
StackOverflow answer, a bookmark — breaks, and GitBook is not holding a redirect for us.
We accept that; the space was small and lightly linked.

A companion package can no longer have its own page tree. If one grows past what a
single page can hold, it has to become a directory with a `README.md` under
*Companion Packages*, not a new space. That is a real constraint on the largest
companion package we might ship, and we accept it.

The core space's page tree grows by one entry per companion package. At five that is
comfortable. At fifteen the *Companion Packages* group would need its own overview
page to stay navigable, which is a cost deferred rather than avoided.

## Alternatives considered

**Keep the space and rewrite its six pages for 7.0.0.** Rejected: it preserves the
inconsistency and the opaque cross-space URLs, and pays the rewrite cost anyway. The
work was the same either way; only the destination differed.

**Give every companion package its own space, for consistency the other way.** Rejected:
five near-empty spaces, each needing its own `README.md`, `SUMMARY.md` and Git Sync
wiring, and every link between them absolute. It also splits the reader's search across
six spaces, since GitBook search is per-space.

**Keep the space and redirect it to the new page.** Rejected: GitBook redirects are
configured per space, so keeping the redirect means keeping the space, its directory and
its Git Sync wiring alive indefinitely. That is the maintenance this decision exists to
end.
