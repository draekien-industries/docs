---
id: 0002
title: Map each top-level directory to exactly one GitBook space
status: accepted
date: 2026-08-17
deciders: [william-pei]
tags: [gitbook, structure]
supersedes:
superseded-by:
---

# 0002 — Map each top-level directory to exactly one GitBook space

## Context

This repository holds documentation for multiple, independently versioned libraries (`waystone.monads`, `waystone.monads.fluentvalidation`, `waystone.widelogevents`), each published as its own GitBook space. Git Sync wires a directory in this repository to a specific GitBook space; that wiring is configured in GitBook, not inferred from the repository's layout. Nothing in the file structure itself prevents someone from nesting two spaces' content under one directory, or splitting one space's content across two directories, and a new top-level directory does not create a new GitBook space on its own — the space must already exist and be Git Sync–wired in GitBook first.

## Decision

Each top-level directory in this repository will always correspond to exactly one GitBook space. A new space is created and wired to a new directory in GitBook first; the directory then appears here as a result of that sync, not the other way around.

## Consequences

The repository's top-level layout stays a reliable map of "which GitBook space is this" for anyone browsing the tree, and each space's `README.md`/`SUMMARY.md`/content pages stay self-contained within one directory. The cost is that creating a new space is a two-step process — set it up in GitBook, then work in the directory that appears — rather than something an agent can bootstrap by just creating a folder and files in git.

## Alternatives considered

- **Let directory structure vary freely (multiple spaces per directory, or one space split across directories)**: more flexible on paper, but nothing enforces the mapping, so it would become ambiguous which files belong to which GitBook space, and Git Sync's directory-to-space wiring would have to be re-derived by inspection each time.
