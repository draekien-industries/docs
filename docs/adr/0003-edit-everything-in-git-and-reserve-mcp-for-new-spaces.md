---
id: 0003
title: Edit everything in git by hand and reserve GitBook MCP for creating a space
status: accepted
date: 2026-08-17
deciders: [william-pei]
tags: [workflow, gitbook]
supersedes: 0001-split-edit-workflow-between-git-and-gitbook-mcp
superseded-by:
---

# 0003 — Edit everything in git by hand and reserve GitBook MCP for creating a space

## Context

[ADR 0001](0001-split-edit-workflow-between-git-and-gitbook-mcp.md) split the
editing workflow in two. Content edits went straight to git. Anything structural
— a new page, a new space, a reorganised page tree — went through the GitBook MCP
change-request flow, so that GitBook kept ownership of the page tree.

Using that split showed two problems.

The common case is adding a page, and the change-request flow is heavy for it.
Three MCP calls replace one commit, and the page's content still has to be
written somewhere.

The flow also needs the GitBook MCP server to be connected. `.mcp.json` declares
it, but its tools are not loaded in every session. Under ADR 0001, a session
without them cannot add a page at all, even though it can write the markdown.

Hand-editing works. A new file plus a matching `SUMMARY.md` entry, committed
together, reaches GitBook on the next sync. The cost is that the change appears
on sync rather than immediately, and that `SUMMARY.md` has to be updated by hand
because GitBook does not regenerate it from directory contents on this side.

## Decision

We will make every edit in git by hand — wording, examples, new pages, and page
tree reorganisation alike — and update the affected `SUMMARY.md` in the same
commit. We will use the GitBook MCP change-request flow for one case only:
creating a new space.

## Consequences

There is one workflow to learn instead of two, and no judgment call about which
bucket a change falls into. Any session that can write a file can add a page, so
work no longer stalls on MCP availability.

Adding a page is now two edits that must stay together: the markdown file and the
`SUMMARY.md` entry. Forget the second and the page exists in git but never
appears in GitBook. Nothing enforces the pairing, so it is the main thing to
check when a new page does not show up.

Git now effectively owns page-level structure, and changes land on sync rather
than live. If someone reorganises a tree in the GitBook UI while a git-side
reorganisation is in flight, the two can conflict — the sync is bidirectional, so
whichever lands second wins. We accept that risk because tree reorganisations are
rare and usually done by one person at a time.

Space creation stays in GitBook because a new top-level directory does not create
a space by itself. The space has to exist and be Git Sync–wired in GitBook before
its directory means anything here, so there is no hand-edit equivalent.

## Alternatives considered

**Keep the ADR 0001 split.** Structural changes through MCP, content through git.
Rejected because the overhead falls on the most common structural change — adding
a page — and because it makes page creation impossible in sessions where the MCP
tools are not loaded, for no benefit that a `SUMMARY.md` edit does not also
deliver.

**Route everything through GitBook MCP, including content.** Considered and
rejected in ADR 0001 for the same reason it is rejected here: it adds a
change-request cycle to small wording fixes that are cheap and safe to commit
directly.
