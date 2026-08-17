---
id: 0001
title: Split the edit workflow between direct git and GitBook MCP change requests
status: superseded
date: 2026-08-17
deciders: [william-pei]
tags: [workflow, gitbook]
supersedes:
superseded-by: 0003-edit-everything-in-git-and-reserve-mcp-for-new-spaces
---

# 0001 — Split the edit workflow between direct git and GitBook MCP change requests

## Context

This repository's content is mirrored to GitBook via bidirectional Git Sync: a change made in git can flow to GitBook, and a change made in the GitBook UI or API can flow back to git. Because the sync is bidirectional, either side is a technically valid place to make any change, including reorganizing a space's page tree. But GitBook, not the git repository, owns the authoritative page tree (`SUMMARY.md`) and space configuration, and manually edited `SUMMARY.md` entries only take effect on the next sync rather than updating GitBook's page tree live. A change that touches structure (a new page, a new space, a reorganized tree) made purely in git risks drifting out of sync with what GitBook considers the tree to be, or being invisible in GitBook until a sync is triggered.

## Decision

Agents and contributors will edit existing page content (wording, examples, fixes) directly in git and commit/push as normal. Changes that alter structure — adding a page, adding a space, or reorganizing a page tree — will go through the GitBook MCP change-request flow (`create_change_request` → `invoke_operation` → `submit_or_merge_change_request`) instead of being made as raw file edits.

## Consequences

Content fixes stay fast and cheap, using the same git workflow as any other repo. Structural changes go through an extra step (a change request in GitBook) rather than a plain commit, which is slower but keeps GitBook's page tree and space configuration authoritative and avoids `SUMMARY.md` drift. Contributors need to know which bucket a given change falls into, which is not enforced by tooling — it depends on judgment.

## Alternatives considered

- **Always edit via git, including structure**: simplest single workflow, but manually edited `SUMMARY.md` entries do not update GitBook's page tree until the next sync, so new pages or reorganizations added this way can silently fail to appear in GitBook, or diverge from what GitBook's editor would produce.
- **Always edit via GitBook MCP, including content**: keeps everything under GitBook's review flow, but adds unnecessary overhead to small wording fixes that don't touch structure and are cheap to change directly.
