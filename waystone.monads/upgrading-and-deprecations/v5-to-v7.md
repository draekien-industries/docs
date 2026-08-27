---
description: >-
  Skipping v6 means every v6 change and every v7 change land together. This is
  the order to do them in.
---

# v5.x to v7.0.0

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

## This page exists because skipping a major is normal

Going from 5.x straight to 7.0.0 means the v6 changes and the v7 changes arrive in the
same build. That is six silent changes rather than three, and they interact — so the
order you work in matters more than it does on either single-version page.

{% hint style="danger" %}
**One v6 change had a warning you will not get.** `WM1010` was the analyzer rule that
warned about `Option.Some` accepting a value-type default. It shipped in 5.5.0 and v6
retired it, so it only ever protected people who happened to be on a 5.5.x release when
they upgraded. If you are on 5.4 or earlier, that warning never existed for you.

[Silent change 5](#silent-change-3-optionsome-accepts-value-type-defaults) is that
change. It is the one on this page with no tooling behind it at all.
{% endhint %}

## Upgrade with an agent

The same prompt covers both paths. It asks which version you are on and does the v6 work
first if the answer is 5.x:

* **[Upgrade with an agent](v7-agent-prompt.md)**

Two of its steps are 5.x-only, and one of those it explicitly refuses to decide for you.
Read [What it cannot do for you](v7-agent-prompt.md#what-it-cannot-do-for-you) first.

## Read this first

Six changes keep compiling and change what your code does. Three came in v6 and three in
v7.

| From | Change |
| --- | --- |
| v6 | [`Try` with an async factory stops catching](#silent-change-1-try-with-an-async-factory) |
| v6 | [Cancellation propagates instead of becoming a failure](#silent-change-2-cancellation-propagates) |
| v6 | [`Option.Some` accepts value-type defaults](#silent-change-3-optionsome-accepts-value-type-defaults) |
| v7 | [A projection returning null throws](v6-to-v7.md#silent-change-1-a-null-projection-throws) |
| v7 | [`AndThen` rejects a null monad](v6-to-v7.md#silent-change-2-andthen-rejects-a-null-monad) |
| v7 | [A scope disposed out of order restores nothing](v6-to-v7.md#silent-change-3-a-scope-disposed-out-of-order-restores-nothing) |

Everything else breaks the build.

## Do it in this order

Work through the v6 page and then the v7 page, rather than reading both at once. Two
reasons, both practical.

**The v6 async changes come first because the v7 changes sit on top of them.** v6 made
every async extension return `ValueTask` and moved async factories to `TryAsync`. The v7
`WM2022` rule and the null-projection guard both describe chains you will have rewritten
by then, so doing v6 first means you touch each chain once.

**The loud v7 changes will hide the loud v6 ones.** `CS0234` on a removed extension class
is a declaration-phase error, so it stops the compiler reporting anything in the body of
every file that has the `using`. Land v6's loud changes, get a green build, then upgrade
to 7.0.0.

1. **Read [v5.x to v6.x](v5-to-v6.md) in full and do its silent changes.** These are the
   ones with no compiler help, and they are the reason this page exists.
2. **Upgrade to 6.x and get a clean build.** Do not skip this. A single intermediate
   build separates two sets of diagnostics that are hard to tell apart in one pile.
3. **Read [v6.x to v7.0.0](v6-to-v7.md) and do its three silent changes.**
4. **Upgrade to `7.0.0-beta.x` and work the diagnostics.** Build twice — see
   [Diagnostics that mask other diagnostics](v7-breaks.md#diagnostics-that-mask-other-diagnostics).

If you cannot ship an intermediate 6.x build, the agent prompt handles both sets in one
pass and reports them separately. It is a worse position to be in, not an equal one.

## The three v6 silent changes, in brief

Full detail on [v5.x to v6.x](v5-to-v6.md). This is enough to know whether they apply to
you.

### Silent change 1: Try with an async factory

`Option.Try` and `Result.Try` given an `async` factory return before the factory has
finished, so nothing is caught. Use `TryAsync`.

`WM1011` reports every one of these, as a **warning**, so this is the one v6 silent
change your build will actually mention. Do not suppress it.

See [Silent change 1](v5-to-v6.md#silent-change-1-try-with-an-async-factory).

### Silent change 2: cancellation propagates

From 6.0.0 an `OperationCanceledException` is no longer converted into a `None` or an
`Err`. It propagates to your caller.

Prefer that. If you genuinely relied on the old behaviour, opt back in:

```csharp
MonadOptions.Configure(options => options.UseCancellationAsFailure());
```

Note the modern spelling — in 7.0.0 the `Use…` methods are on a builder, and the
inferred lambda parameter above is what makes this line identical in both versions.

See [Silent change 2](v5-to-v6.md#silent-change-2-cancellation-propagates).

### Silent change 3: Option.Some accepts value-type defaults

`Option.Some(0)` returns a `Some` where 5.x threw, and `Option<int> x = 0;` gave you a
`None` in 5.x and a `Some(0)` in 6.x.

This is the change with no tooling behind it for anyone below 5.5.0, and no tooling at all
from 6.0.0 onward. Every `IsNone` branch on a value-type option has to be read by
someone who knows whether it was standing in for zero.

{% hint style="info" %}
**In 7.0.0 the implicit conversion is gone entirely**, so `Option<int> x = 0;` no longer
compiles at all. That turns this particular assignment from a silent change into a build
error on the v7 step — which is the one piece of luck on this path. It does not help with
`Option.Some(0)` written out in full.
{% endhint %}

See [Silent change 3](v5-to-v6.md#silent-change-3-some-accepts-value-type-defaults) on
the v6 page.

## The loud changes from both versions

Rather than repeat two tables, here is where each list lives:

* **v6's loud changes** — `ValueTask` everywhere, `FlatMap` removed, deriving from
  `Option` or `Result` no longer allowed. On [v5.x to v6.x](v5-to-v6.md).
* **v7's loud changes** — implicit conversions removed, configuration moved to a builder,
  extension classes collapsed, five obsolete members gone, parameter renames. On
  [v6.x to v7.0.0](v6-to-v7.md), and as a reference table with diagnostics on
  [Every v7 break](v7-breaks.md).

Two of v6's loud changes are worth flagging here because they multiply on this path:

**`.AsTask()` sites come from v6, not v7.** If you see `CS0029` between `Task` and
`ValueTask`, that is the v6 change. Prefer changing the declared type or awaiting the
value; `.AsTask()` allocates.

**`FlatMap` and the removed extension classes overlap.** A call written as
`FlatMapExtensions.FlatMap(...)` breaks twice — once because the method was renamed to
`AndThen` in v6, and once because the class is gone in v7. Rename first, then drop the
qualifier.

## Analyzer rules across both versions

Delete `.editorconfig` entries and `#pragma` directives for every retired id. None of
them is reused, so a stale entry neither errors nor warns — it simply does nothing, which
is worse, because it reads as though something is configured.

| Retired in | Ids |
| --- | --- |
| v6.0.0 | `WM1004`, `WM1007`, `WM1009`, `WM1010`, `WM2014` |
| v7.0.0 | `WM2010` |

Added across the two versions: `WM1011`, `WM2015`, `WM2016`, `WM2017` and more in v6 —
see [v5.x to v6.x](v5-to-v6.md#analyzer-rules-that-changed) — and `WM2022` in v7.

## Everything on one page

| Change | From | Breaks the build? | What to do |
| --- | --- | --- | --- |
| `Try` with an async factory | v6 | **No**, but `WM1011` warns | Use `TryAsync` |
| Cancellation propagates | v6 | **No** | Handle it, or `UseCancellationAsFailure` |
| `Option.Some` accepts value-type defaults | v6 | **No** | Read every `IsNone` branch on a value type |
| Async extensions return `ValueTask` | v6 | **Yes** | Change the declared type or await it |
| `FlatMap` removed | v6 | **Yes** | Rename to `AndThen` |
| Deriving from `Option` or `Result` | v6 | **Yes** | Compose rather than derive |
| A projection returning null throws | v7 | **No** | `AndThen` with `Option.FromNullable` |
| `AndThen` rejects a null monad | v7 | **No** | Return `None` or an `Err` |
| A scope disposed out of order | v7 | **No** | Use `using` |
| Implicit conversions removed | v7 | **Yes** | Take the code fix on `CS0029` / `CS1503` |
| Configuration moved to a builder | v7 | **Yes**, for explicit types only | Let the lambda parameter infer |
| Extension classes collapsed | v7 | **Yes** | Drop the `using static` |
| Five obsolete members removed | v7 | **Yes** | See the v7 page |
| Parameter renames | v7 | **Yes**, for named arguments only | Take the code fix on `CS1739` |
