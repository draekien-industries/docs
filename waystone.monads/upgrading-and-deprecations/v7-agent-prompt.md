---
description: >-
  One prompt for an agent to run the 6.x or 5.x upgrade to 7.0.0, and the parts
  it deliberately leaves to you.
---

# Upgrade with an agent

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

Copy the prompt below into Claude Code or a similar tool, pointed at your solution. It
covers both upgrade paths — it asks whether you are coming from 5.x and skips the
5.x-only steps if you are not.

Read [What it cannot do for you](#what-it-cannot-do-for-you) before you run it.

## The prompt

`````text
You are upgrading a C# codebase from Waystone.Monads 5.x or 6.x to 7.0.0-beta.x.

First, read the version currently referenced and tell me whether this is a 6.x or a 5.x
upgrade. If it is 5.x, every v6 change lands at the same time as every v7 change, so do
the v6 work first and report it separately.

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

4. Only if coming from 5.x: a cancellation is no longer caught. Try and TryAsync let an
   OperationCanceledException propagate instead of converting it to a None or an Err. If
   the codebase relied on the old behaviour, restore it with
   MonadOptions.Configure(options => options.UseCancellationAsFailure()); — but prefer
   letting cancellation propagate, and say so if you change it.

5. Only if coming from 5.x: Option.Some accepts value-type defaults. Option.Some(0)
   returns a Some where it used to throw, and Option<int> x = 0; gave you a None
   before. Report every IsNone branch on a value-type option — you cannot decide these
   without me.

There is deliberately no item here about holding a MonadOptions instance. Options did
become an immutable snapshot in 7.0.0, but the type no longer exposes any public
instance member or accessor at all, so code that held one fails to build rather than
quietly going stale. That is compile-time work, and step 4 covers it. Do not
reintroduce it as a silent change.

## Step 2 — upgrade the package and build

Add --prerelease, or set Version="7.0.0-beta.*". Then build and capture every
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
`````

## What it cannot do for you

{% hint style="warning" %}
**Step 1 needs your judgement.** The prompt asks the agent to find and report every
silent change, and to decide only the ones with a mechanical answer.

Two have no mechanical answer:

* **A projection that can return null.** Whether the null meant "absent" or was never
  supposed to happen decides whether you convert to `Option.FromNullable` or fix the
  caller. Only someone who knows the domain can say.
* **Coming from 5.x, an `IsNone` branch on a value-type option.** `Option.Some(0)` now
  gives you a `Some`. Whether that branch was standing in for "zero" needs someone who
  knows what the code means. See
  [Silent change 3](v5-to-v6.md#silent-change-3-some-accepts-value-type-defaults).

Expect the agent to bring these to you. If it decided them on its own, that is a
finding.
{% endhint %}

## If you would rather not use an agent

Every step above maps to a section on the upgrade pages, and
[Every v7 break](v7-breaks.md) is the same information as a reference table naming the
diagnostic each break produces.
