---
title: Document the v5.2.0 surface and the v6.0.0 deprecations
date: 2026-08-17
status: done
supersedes:
---

# Document the v5.2.0 surface and the v6.0.0 deprecations

## Goal

Cover everything `Waystone.Monads` 5.2.0 added, close the async documentation gap
that predates it, and give consumers a forward-looking record of what v6.0.0
removes so they can migrate before the major lands.

Follows [ADR 0003](../adr/0003-edit-everything-in-git-and-reserve-mcp-for-new-spaces.md):
every change here is a hand edit in git, including the new pages. No new space is
needed, so the GitBook MCP flow does not come into it.

## What shipped

`v5.2.0` (commit `7adb340`, tag `v5.2.0`) added the following to
`Waystone.Monads`:

- **Factories that default `TErr` to `Error`** — `Result.Ok<TOk>`,
  `Result.Err<TOk>(Error)`, `Result.Try<TOk>` (converting via
  `Error.FromException`), and `Result.TryAsync<TOk>`.
- **Enum to error** — `Error.FromEnum(Enum, string)` and
  `Result.Err<TOk>(Enum, string)`. The message is required.
- **Async terminals** on `Task` and `ValueTask` receivers — `UnwrapAsync`,
  `UnwrapOrAsync`, `UnwrapOrDefaultAsync`, `ExpectAsync` for both monads, plus
  `UnwrapErrAsync` and `ExpectErrAsync` for `Result`.
- **`TryAsync`** on both monads, with the async-factory `Try` overloads
  obsoleted.
- **`MonadOptions.BeginScope`** — a disposable that overrides configuration for
  the current asynchronous flow, inheriting unset options from the configuration
  in effect when the scope opens, nesting, and isolating concurrent flows.
  `Waystone.Monads.FluentValidation` participates, so one scope covers both
  packages.

Deprecated in 5.2.0 and removed in 6.0.0:

| Deprecated | Replacement |
| --- | --- |
| `Option.Try<T>(Func<Task<T>>, …)` | `Option.TryAsync<T>(Func<Task<T>>, …)` |
| `Result.Try<TOk, TErr>(Func<Task<TOk>>, Func<Exception, TErr>, …)` | `Result.TryAsync<TOk, TErr>(…)` |

The synchronous `Try` overloads keep their name. `Result.TryAsync<TOk>` never had
a `Try` spelling, so it has nothing to migrate.

## The gap this uncovered

The async surface has never been documented. `Async` appears in the
`waystone.monads` space only inside `upgrading.md`, which mentions the v3 rename
in passing. Thirty-eight async extension methods are undocumented — nineteen per
monad, listed in step 1 below — and the v5.2.0 terminals cannot be slotted
alongside them because there is nowhere to slot them into.

## Steps

Steps 1 to 3 add pages. Write each markdown file and add its `SUMMARY.md` entry in
the same commit. Miss the `SUMMARY.md` entry and the page reaches git but never
appears in GitBook.

Two spaces need a `SUMMARY.md` change:

- `waystone.monads/SUMMARY.md` — add *Async* between Core Functionality and
  Option\<T>, and *Deprecations* after Upgrading, both under the
  `## Using the Library` group.
- `waystone.monads.fluentvalidation/SUMMARY.md` — add *Configuration* under the
  `## Using The Library` group, before Validating Values.

Match the existing entry format, including the escaped angle brackets used in
`[Option\<T>]`.

### 1. New page: `waystone.monads` → Using the Library → Async

Place it between *Core Functionality* and *Option\<T>*.

Explain the model once, because it is the part that is genuinely surprising:

- The async methods are extension methods on `Task<Option<T>>`,
  `ValueTask<Option<T>>`, `Task<Result<T, E>>` and `ValueTask<Result<T, E>>` —
  not instance methods on the monad. That is what lets calls chain without an
  intervening `await`.
- Chaining: `await option.MapAsync(...).MapAsync(...)` rather than awaiting each
  step into a local.
- Some methods return `ValueTask` rather than `Task`, because the value may
  already be available synchronously. This changed in v5 and `upgrading.md`
  records it.

Then table the surface. `Option<T>`:

`MapAsync`, `MapOrAsync`, `MapOrElseAsync`, `MatchAsync`, `FilterAsync`,
`FlatMapAsync`, `FlattenAsync`, `InspectAsync`, `IsSomeAndAsync`,
`IsNoneOrAsync`, `OkOrAsync`, `OkOrElseAsync`, `OrElseAsync`, `ZipWithAsync`,
`UnwrapAsync`, `UnwrapOrAsync`, `UnwrapOrDefaultAsync`, `UnwrapOrElseAsync`,
`ExpectAsync`.

`Result<T, E>`:

`MapAsync`, `MapErrAsync`, `MapOrAsync`, `MapOrElseAsync`, `MatchAsync`,
`AndThenAsync`, `OrElseAsync`, `FlattenAsync`, `InspectAsync`, `InspectErrAsync`,
`IsOkAndAsync`, `IsErrAndAsync`, `UnwrapAsync`, `UnwrapErrAsync`,
`UnwrapOrAsync`, `UnwrapOrDefaultAsync`, `UnwrapOrElseAsync`, `ExpectAsync`,
`ExpectErrAsync`.

Cover `Option.TryAsync` and `Result.TryAsync` here too, and note that the
async-factory `Try` overloads are deprecated, linking the Deprecations page.

Use the space's existing `{% tabs %}` Option/Result convention and a `description`
in frontmatter, matching `core-functionality.md`.

Worked example for the terminals, since their value is not obvious from a name:

```csharp
User user = await FetchUserAsync(id).UnwrapAsync();
User orGuest = await FetchUserAsync(id).UnwrapOrAsync(Guest);
Error error = await FetchUserAsync(id).UnwrapErrAsync();
```

### 2. New page: `waystone.monads` → Using the Library → Deprecations

Place it after *Upgrading*. Mirror `BREAKING_CHANGES.md` from the code repo: what
is deprecated, the version that deprecated it, the version that removes it, and
the rename to make. Include the ambiguity rationale — a `throw`-only lambda
converts to both `Func<T>` and `Func<Task<T>>`, so the sync and async `Try`
overloads collide and the caller has to declare the delegate type — because it
explains why a rename was worth a breaking change.

State plainly that nothing on the page is broken yet: every entry still compiles
and behaves as before, and emits `CS0618`.

Leave `upgrading.md` alone. It documents completed upgrades, and the v5.x → v6.x
section belongs there when v6 actually ships — at which point this page's
contents move into it.

### 3. New page: `waystone.monads.fluentvalidation` → Using The Library → Configuration

The space has no configuration page, and 5.2.0 changed how its options resolve.
Cover `UseValidationErrorCode` and `UseFallbackValidationErrorMessage`, that they
are reached through `MonadOptions.Configure` alongside the core options, and that
they honour `MonadOptions.BeginScope` — one scope covers both packages, so there
is no second scope to open.

### 4. Content edit: `waystone.monads/using-the-library/configuration.md`

Add a *Scoped Configuration* section after *Error Code and Message Fallbacks*.

The page currently says configuration should happen "once in the lifecycle of
your application", which is still the right default but is now the whole story
only for global configuration. Note the scope as the exception and keep the
recommendation.

Cover: the `using` shape, inheritance of unset options, snapshot semantics (a
later `Configure` does not change an open scope), nesting, and per-flow isolation
so concurrent work sees its own options. Give the two motivating cases — a single
request or a block under debugging, and tests that would otherwise fight over the
global singleton.

```csharp
using (MonadOptions.BeginScope(options => options.UseFallbackErrorCode("Debug")))
{
    Result<int, Error> result = Result.Try<int>(() => int.Parse(input));
}
```

Add a `{% hint style="info" %}` noting that a scope applies to work started
inside it, not to work already running when it opened.

### 5. Content edit: `waystone.monads/using-the-library/core-functionality.md`

In *Creation* only — the rest of the page stays untouched.

Add the single-type-parameter factories to the Result tab, showing them next to
the two-parameter forms so the choice is visible:

```csharp
Result<int, string> custom = Result.Err<int, string>("something went wrong");
Result<int, Error> builtIn = Result.Err<int>(new Error("MyCode", "something went wrong"));
Result<User, Error> fromEnum = Result.Err<User>(UserErrors.NotFound, "the user was not found");
```

Extend the existing `Try` coverage with `Result.Try<TOk>`, noting that it defaults
the error conversion to `Error.FromException` so no `onErr` delegate is needed.
Link the Async page for `TryAsync` rather than covering async here.

### 6. Content edit: `waystone.monads/using-the-library/errors-and-exceptions.md`

Two changes.

Add an *Error from Enum* subsection under *Error*, covering
`Error.FromEnum(Enum, string)` and pointing at the existing
[#error-code-from-enum](../../waystone.monads/using-the-library/errors-and-exceptions.md)
anchor for how the code is derived. The message is required — say so, since
`ErrorCode.FromEnum` takes no message and the asymmetry invites a wrong guess.

**Fix an existing error.** The *Error from Exception* sample reads:

```csharp
var error = ErrorCode.FromException(e);
//  ^? ErrorCode: "Err.Sql", Message: e.Message
```

`ErrorCode.FromException` returns an `ErrorCode`, which has no message. The
sample should call `Error.FromException(e)`. The surrounding prose is correct;
only the call is wrong.

### 7. Content edit: `waystone.monads/using-the-library/result-of-t-and-e.md`

Add a pointer under *ExpectErr* and *UnwrapErr* to their async counterparts on
the Async page. No other change — the sync coverage is fine.

## Done when

- The Async page lists all thirty-eight async methods, and a reader can tell from
  it why the methods are extensions on `Task`/`ValueTask` rather than instance
  methods.
- The Deprecations page names both deprecated overloads, their replacements, the
  version that removes them, and states that nothing is broken yet.
- `configuration.md` covers `BeginScope` including snapshot and per-flow
  semantics, and its "configure once" advice no longer reads as the only option.
- `core-functionality.md` shows the `Error`-defaulted factories and the enum
  overload in the Creation section.
- `errors-and-exceptions.md` documents `Error.FromEnum` and the
  `ErrorCode.FromException` sample is corrected.
- The FluentValidation space has a Configuration page stating that its options
  honour `MonadOptions.BeginScope`.
- Each new page has a `SUMMARY.md` entry in the same commit, and GitBook renders
  it in the position described above after the next sync.

## Notes and follow-ups

- **A new page needs both edits.** The markdown file alone is invisible to
  GitBook. If a page does not turn up after a sync, check `SUMMARY.md` first.
- **`BREAKING_CHANGES.md` in the code repo is stale.** It still reads
  "Latest release: **v5.1.1**" although v5.2.0 has shipped and is tagged. Worth
  fixing there before the Deprecations page is written from it.
- **When v6.0.0 cuts**, the Deprecations page contents move into a v5.x → v6.x
  section in `upgrading.md`, the removed overloads come out of every sample, and
  the Async page's deprecation note goes away. That is a separate plan.
- The `output-styles:plain-language` skill applies to all prose written under
  this plan, per the repo's `AGENTS.md`.
