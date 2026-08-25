# draekien-industries/docs

## Purpose

Public documentation for the Waystone family of .NET libraries (`waystone.monads`, `waystone.monads.fluentvalidation`, `waystone.widelogevents`), published via GitBook for developers consuming those NuGet packages. This repo holds no code — only the markdown content GitBook renders.

## Conventions

- Each top-level directory is exactly one GitBook space, always. Never nest multiple spaces under one directory, and never create a new top-level directory as a way to create a new space — the space must exist in GitBook first (see Gotchas).
- Every space has a `README.md` (its landing/welcome page) and a `SUMMARY.md` (its page tree). Content files live in subdirectories matching the structure declared in `SUMMARY.md`.
- Edit in git by hand. This covers wording, examples, fixing a page, adding a page, and reorganising a page tree. Commit and push as normal, and GitBook's bidirectional Git Sync mirrors it in. When you add or move a page, update that space's `SUMMARY.md` in the same commit — see Gotchas.
- Use the GitBook MCP change-request flow (`create_change_request` → `invoke_operation` → `submit_or_merge_change_request`) for one thing only: creating a new space. Everything else is a hand edit.
- Use the `output-styles:plain-language` skill when writing or editing documentation content in this repo.
- Compile code samples rather than reading them. Both samples on the Waystone.Monads configuration page failed against 6.x — one used a delegate arity that never existed, the other assigned internal properties — and neither was visible to inspection. A scratch file under `waystone-dotnet/sample/` proves it in seconds.
- Commit messages follow `docs(<scope>): <subject>`, e.g. `docs(waystone-monads-28): fix links`. The scope is the identifier of the content unit being changed, not a free-form area name. Commits with `No subject` were authored from the GitBook UI editor, not typed by hand — that's expected, not a defect to fix.

## Gotchas

- If you add a new markdown file directly via git, GitBook will not pick it up until it's also added to that space's `SUMMARY.md` — GitBook does not regenerate `SUMMARY.md` from directory contents on this side of the sync. Manually edited `SUMMARY.md` entries only take effect on the next sync.
- A new top-level directory will not create a new GitBook space by itself. The space has to be created and Git Sync–wired in GitBook first; only then does its directory start appearing here.

## Documentation

Agent-facing documentation lives in `docs/`. Read [docs/AGENTS.md](docs/AGENTS.md) before reading or writing anything there.
