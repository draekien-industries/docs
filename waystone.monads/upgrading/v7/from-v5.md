---
description: >-
  Skipping v6 means every v6 change and every v7 change land together. This is
  the order to do them in.
---

# v5.x to v7.0.0

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

<details>

<summary>Upgrade with an agent — copy this prompt</summary>

Pointed at your solution, in Claude Code or a similar tool. It does the v6 work
first and reports it separately.

```text
You are upgrading a C# codebase from Waystone.Monads 5.x to 7.0.0. Every v6 change
lands at the same time as every v7 change, so do the v6 work first and report it
separately.

Work in this order. Do not skip step 1 — it is the only step the compiler cannot do for
you.

## Step 1 — find the silent behaviour changes first

These compile without error and behave differently. Search for each, decide per call
site, and report every one you changed and every one you deliberately left. Do this
before you touch anything that fails to build, because the build errors will otherwise
bury them.

1. A projection returning null now throws. On an Option, Map, MapAsync, Reduce and
   ReduceAsync used to carry a null return from your delegate into the Some. They now
   throw ArgumentNullException, naming the delegate parameter. Find delegates passed to
   those members that can return null — a nullable reference, a FirstOrDefault, a
   dictionary lookup, an explicit return null. Each is a latent exception. Fix by
   projecting into an option instead: AndThen with Option.FromNullable.

2. A factory returning a null monad now throws. AndThen and AndThenAsync used to accept
   a null Option or Result back from your factory. They now throw
   ArgumentNullException naming optionFactory or resultFactory. Return
   Option.None<T>() or an Err rather than null.

3. Disposing an options scope out of order no longer restores. If the codebase disposes
   a MonadOptionsScope by hand rather than with using, or stores one in a field, the
   out-of-order case now declines to restore and writes a
   Waystone.Monads.ScopeDisposedOutOfOrder diagnostic event instead of silently putting
   the wrong options back. Convert every one to using.

4. A cancellation is no longer caught. Try and TryAsync let an
   OperationCanceledException propagate instead of converting it to a None or an Err. If
   the codebase relied on the old behaviour, restore it with
   MonadOptions.Configure(options => options.UseCancellationAsFailure()); — but prefer
   letting cancellation propagate, and say so if you change it.

5. Option.Some accepts value-type defaults. Option.Some(0)
   returns a Some where it used to throw, and Option<int> x = 0; gave you a None
   before. Report every IsNone branch on a value-type option — you cannot decide these
   without me.

There is deliberately no item here about holding a MonadOptions instance. Options did
become an immutable snapshot in 7.0.0, but the type no longer exposes any public
instance member or accessor at all, so code that held one fails to build rather than
quietly going stale. That is compile-time work, and step 4 covers it. Do not
reintroduce it as a silent change.

## Step 2 — upgrade the package and build

Set the package version to 7.0.0. Then build and capture every
diagnostic. Do not fix anything yet. Count the diagnostics by code and report the
counts, so both of us know the size of the job.

Build twice before you trust the count. Three diagnostics in this upgrade fire at the
declaration phase and mask every body-phase error in the same project: CS0234 on a
using static of a removed extension class, CS0115 on an ErrorCodeFactory.FromEnum
override, and — only if the FluentValidation package is referenced — CS0246 on a
signature naming the removed ValidationErr type. Fix those first, then re-count.

## Step 3 — take the code fixes the package offers

Waystone.Monads ships code fixes keyed to the compiler diagnostics this upgrade
produces, not to analyzer rules — so dotnet format analyzers will not apply them. They
appear on the error itself in your IDE, and "Fix all occurrences in Project" applies
them in a batch:

- CS1739 — a renamed parameter. The fix substitutes the new name.
- CS0029 and CS1503 — a removed implicit conversion. The fix wraps the value in
  Option.Some, Result.Ok or Result.Err. Where a Result carries the same type on both
  sides it offers both and does not pick for you.
- CS0029 and CS1503 on an async chain — the fix adds .AsTask(). Prefer changing the
  declared type or awaiting the value; .AsTask() allocates.
- WM2022 — a Task-returning method group passed to AndThenAsync or OrElseAsync. The fix
  wraps it in an async lambda.

If you cannot drive an IDE, do these by hand from the diagnostic list in step 4.
Rebuild and re-count either way.

## Step 4 — work the remaining diagnostics by code

- Anything touching configuration — CS1061, CS1503, CS0029, CS0103. In 7.0.0
  MonadOptions publishes an immutable snapshot and the authoring surface moved to a new
  MonadOptionsBuilder. The Use* methods live on the builder, not on MonadOptions.
  - MonadOptions.Configure(options => options.UseFallbackErrorCode("x")); still
    compiles unchanged, because the lambda parameter's type is inferred. This is the
    dominant call shape. Leave call sites that already build alone.
  - An explicitly typed lambda parameter breaks. (MonadOptions options) => becomes
    (MonadOptionsBuilder options) =>, or drop the type annotation and let it infer.
  - A field, parameter, local or property typed MonadOptions has no replacement.
    Configuration is reachable only inside a Configure or BeginScope callback now. Move
    the Use* calls into one; do not try to pass options around.
  - In the satellite packages, MonadOptionsExtensions is renamed to
    MonadOptionsBuilderExtensions, and its extension receiver changes from MonadOptions
    to MonadOptionsBuilder. A using static or a qualified static call naming the old
    class must be updated; a reduced extension call on the callback's parameter needs no
    change.
  - Never stash the MonadOptionsBuilder the callback hands you. It is authoring state,
    not the published options — calls made on it after the callback returns are
    discarded without error.
- CS1739 — "does not have a parameter named X". A parameter was renamed. Read the new
  name from the signature and update the named argument. Do not convert the call to
  positional arguments to dodge it; the name was chosen to say what the argument is for.
- CS0029 / CS1503 — cannot convert. Either the implicit conversions to Option and Result
  were removed, so wrap the value explicitly in Option.Some, Result.Ok or Result.Err; or
  an async member now returns ValueTask<T> where it returned Task<T> — in the core
  package Option.TryAsync, Result.TryAsync and CollectAsync are the three that changed in
  7.0.0, and the FluentValidation package's ValidateAsync changed with them. For the
  ValueTask case, prefer changing the local's type or awaiting it to converting with
  .AsTask(), which allocates. A call to Task.WhenAll is the one place .AsTask() is the
  right answer.
- CS0103 / CS0234 — the name does not exist. The per-family extension classes were
  collapsed into one class per monad. A using static or a qualified static call naming
  the old class must move to OptionExtensions or ResultExtensions, or better, become a
  reduced extension call on the receiver.
- CS0117 / CS1061 / CS1501 — no such member or overload. A member obsoleted in 6.x was
  removed. The five are ErrorCode.FromEnum, Error.FromEnum, ErrorCodeFactory.FromEnum,
  MonadOptions.UseExceptionLogger, and the Result.Err<TOk>(Enum, string) overload. Read
  the obsoletion message in the 6.x package for the replacement it names.
  - UseExceptionLogger is the one worth spelling out. If its delegate only logged, replace
    it with UseLoggerFactory, UseLoggerFactoryFrom or UseLogger on the builder, from the
    Waystone.Monads.Extensions.Logging package. If it did anything else — a metric, a
    span, a bug report — that is an observer, not a logger: subscribe with
    MonadDiagnostics.ExceptionHandledEvent.Subscribe instead. Do not reach for a
    DiagnosticListener by name; the typed token exists so a wrong name cannot fail
    silently.
- CS0115 — no suitable method found to override. A virtual a consumer overrode is gone.
  ErrorCodeFactory.FromEnum is the one this hits: enum codes are settled at compile time
  now and a factory cannot change them. Delete the override and use [ErrorCodeCatalog]
  instead.
- CS0411 — type arguments cannot be inferred. An async chaining step's delegate returns
  a Task where a ValueTask is wanted. WM2022 reports the same call and says which
  parameter. Change the step to return ValueTask, or wrap it in an async lambda.
- Anything naming Waystone.Monads.FluentValidation. Check whether the solution references
  that package before reading this; if it does not, skip the whole bullet. If it does,
  the package was rewritten in 7.0.0 rather than deprecated, so nothing warned in 6.x and
  no code fix exists. Every call site needs a hand edit.
  - The namespaces shadow FluentValidation's own (CS0246, CS0234). Delete every using of
    Waystone.Monads.FluentValidation.Results.Extensions, .Results and .Configs, and use
    FluentValidation.Extensions, FluentValidation and FluentValidation.Configs. A file
    that already has using FluentValidation; needs nothing back for ValidationError. The
    package and assembly names are unchanged, so leave the PackageReference alone.
  - ValidationErr is gone (CS0246). ValidationError replaces it and derives from Error,
    so a failed validation now joins a chain without a MapErr at the seam.
  - Validate and ValidateAsync err with Error, not ValidationErr (CS0029). Change the
    declared type to Result<T, Error>, and delete the MapErr that used to convert.
  - ToError() is gone (CS1061). Delete the call — you already hold an Error.
  - ValidationErr.Create() is gone (CS0117). Only Validate and ValidateAsync build a
    ValidationError now; its constructor is internal.
  - AsValidationResult() and RuleSetsExecuted are gone (CS1061). Failures carries the
    ValidationFailure list, and ToDictionary() groups messages by property as before.
  - UseFallbackValidationErrorMessage is gone (CS1061) with no replacement. A
    ValidationError always carries at least one failure, so the empty case it covered
    cannot happen. Delete the call.
  - To read failure detail from a plain Error, pattern match: if (error is
    ValidationError validationError). Report every place you had to add that match.
  - Tell me if the resolved FluentValidation version moved. The package now allows
    >= 11.1.0 && < 13.0.0 where 6.x pinned one version, so the upgrade can pull a
    different FluentValidation than the solution was tested against. Do not pin it back
    without asking me.

## Step 5 — verify

Build clean, then run the test suite. Report: the diagnostic counts before and after,
every silent change from step 1 with the decision you made, and anything you could not
resolve.

## Step 6 — offer these, do not apply them

Once the build is clean, tell me about these optional things and let me decide. Do not
install a package or change a severity in this step.

- Waystone.Monads.Shouldly, which replaces assertions on IsSome and Unwrap in the test
  suite. Its WMS2001 and WMS2002 rules are batch-fixable.
- Waystone.Monads.Linq, which adds C# query syntax over the monads.
- The recommended severity preset, which makes the seven misuse rules build errors. A
  major upgrade is a reasonable moment to turn it on, but it is my call, not yours.
- Waystone.Monads.Extensions.Hosting, if the application is built on
  Microsoft.Extensions.Hosting. builder.AddWaystoneMonads(...) registers the
  configuration and installs it from the host's own start-up sequence, replacing the
  hand-written MonadOptions.Configure call. Say where the current Configure call lives so
  I can see what it would replace.
- Waystone.Monads.Extensions.DependencyInjection, only if the application has a container
  but is not built on the hosting abstractions. It gives services.AddWaystoneMonads(...),
  and the configuration is applied by a separate provider.UseWaystoneMonads() call. That
  second call is easy to forget; if it is missed the library writes a
  Waystone.Monads.ConfigurationNotApplied event rather than failing. Prefer the hosting
  package where both would work, because it makes that call for you.

## Rules

- Never suppress a diagnostic, add a pragma to disable one, or add a null-forgiving !
  to make an error go away. Every diagnostic here has a real fix.
- Do not change behaviour to make a test pass. If a test fails after the upgrade, that
  is either a step 1 change you missed or a real finding — report it, do not paper over
  it.
- Do not reformat, rename, or refactor anything the upgrade does not require.
```

</details>

{% hint style="warning" %}
**Two steps it cannot do for you**, both in step 1 of the prompt. Whether a null
projection meant "absent" or was never supposed to happen is a domain question. So is
whether an `IsNone` branch on a value-type option was standing in for "zero" — see
[Silent change 3](../older/v5-to-v6.md#silent-change-3-some-accepts-value-type-defaults).
Expect the agent to bring both to you.
{% endhint %}

## Read this first

Six changes keep compiling and change what your code does. Three came in v6 and three in
v7.

| From | Change |
| --- | --- |
| v6 | [`Try` with an async factory stops catching](#silent-change-1-try-with-an-async-factory) |
| v6 | [Cancellation propagates instead of becoming a failure](#silent-change-2-cancellation-propagates) |
| v6 | [`Option.Some` accepts value-type defaults](#silent-change-3-optionsome-accepts-value-type-defaults) |
| v7 | [A projection returning null throws](from-v6.md#silent-change-1-a-null-projection-throws) |
| v7 | [`AndThen` rejects a null monad](from-v6.md#silent-change-2-andthen-rejects-a-null-monad) |
| v7 | [A scope disposed out of order restores nothing](from-v6.md#silent-change-3-a-scope-disposed-out-of-order-restores-nothing) |

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

1. **Read [v5.x to v6.x](../older/v5-to-v6.md) in full and do its silent changes.** These are the
   ones with no compiler help, and they are the reason this page exists.
2. **Upgrade to 6.x and get a clean build.** Do not skip this. A single intermediate
   build separates two sets of diagnostics that are hard to tell apart in one pile.
3. **Read [v6.x to v7.0.0](from-v6.md) and do its three silent changes.**
4. **Upgrade to `7.0.0` and work the diagnostics.** Build twice — see
   [Diagnostics that mask other diagnostics](breaking-changes.md#diagnostics-that-mask-other-diagnostics).

If you cannot ship an intermediate 6.x build, the agent prompt handles both sets in one
pass and reports them separately. It is a worse position to be in, not an equal one.

## The three v6 silent changes, in brief

Full detail on [v5.x to v6.x](../older/v5-to-v6.md). This is enough to know whether they apply to
you.

### Silent change 1: Try with an async factory

`Option.Try` and `Result.Try` given an `async` factory return before the factory has
finished, so nothing is caught. Use `TryAsync`.

`WM1011` reports every one of these, as a **warning**, so this is the one v6 silent
change your build will actually mention. Do not suppress it.

See [Silent change 1](../older/v5-to-v6.md#silent-change-1-try-with-an-async-factory).

### Silent change 2: cancellation propagates

From 6.0.0 an `OperationCanceledException` is no longer converted into a `None` or an
`Err`. It propagates to your caller.

Prefer that. If you genuinely relied on the old behaviour, opt back in:

```csharp
MonadOptions.Configure(options => options.UseCancellationAsFailure());
```

Note the modern spelling — in 7.0.0 the `Use…` methods are on a builder, and the
inferred lambda parameter above is what makes this line identical in both versions.

See [Silent change 2](../older/v5-to-v6.md#silent-change-2-cancellation-propagates).

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

See [Silent change 3](../older/v5-to-v6.md#silent-change-3-some-accepts-value-type-defaults) on
the v6 page.

## The loud changes from both versions

Rather than repeat two tables, here is where each list lives:

* **v6's loud changes** — `ValueTask` everywhere, `FlatMap` removed, deriving from
  `Option` or `Result` no longer allowed. On [v5.x to v6.x](../older/v5-to-v6.md).
* **v7's loud changes** — implicit conversions removed, configuration moved to a builder,
  extension classes collapsed, five obsolete members gone, parameter renames, and
  `TryAsync` and `CollectAsync` returning `ValueTask`. On
  [v6.x to v7.0.0](from-v6.md), and as a reference table with diagnostics on
  [Every v7 break](breaking-changes.md).

Two of v6's loud changes are worth flagging here because they multiply on this path:

**`.AsTask()` sites come from both versions.** v6 moved the async extensions to
`ValueTask`; v7 moved `TryAsync` and `CollectAsync` too. Coming from 5.x you cannot tell
the two apart from the diagnostic, and you do not need to — treat every `CS0029` between
`Task` and `ValueTask` the same way. Prefer changing the declared type or awaiting the
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
see [v5.x to v6.x](../older/v5-to-v6.md#analyzer-rules-that-changed) — and `WM2022` in v7.

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
| `TryAsync` and `CollectAsync` return `ValueTask` | v7 | **Yes**, if you name the type | Change the declared type, or add `.AsTask()` |
