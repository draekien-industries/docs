---
description: >-
  Three changes that keep compiling and change what your code does, and eight
  that break the build.
icon: six
---

# v6.x to v7.0.0

<details>

<summary>Upgrade with an agent — copy this prompt</summary>

Pointed at your solution, in Claude Code or a similar tool. It covers every
mechanical part of this upgrade.

```text
You are upgrading a C# codebase from Waystone.Monads 6.x to 7.0.0.

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
**One step it cannot do for you**, in step 1 of the prompt. Whether a null projection
meant "absent" or was never supposed to happen is a domain question, and only you can
answer it. Expect the agent to bring it to you; if it decided on its own, that is a
finding.
{% endhint %}

## Read this first

Three changes keep compiling and change what your code does:

1. [A projection that returns null now throws](#silent-change-1-a-null-projection-throws).
2. [`AndThen` rejects a factory that returns a null monad](#silent-change-2-andthen-rejects-a-null-monad).
3. [A scope disposed out of order restores nothing](#silent-change-3-a-scope-disposed-out-of-order-restores-nothing).

Everything else in 7.0.0 breaks loudly. The compiler will find it for you, and
[Every v7 break](breaking-changes.md) lists which diagnostic each one produces.

## Silent change 1: a null projection throws

`Map`, `MapAsync`, `Reduce` and `ReduceAsync` on an `Option` used to carry a null return
from your delegate straight into the `Some`. They now throw `ArgumentNullException`,
naming the delegate parameter.

```csharp
Option<string> name = user.Map(u => u.MiddleName); // MiddleName can be null

// 6.x: a Some holding null.
// 7.0.0: ArgumentNullException, naming 'map'.
```

This is a fix, not a tightening for its own sake. A `Some` holding null broke the one
promise the type makes, and the failure surfaced later as a `NullReferenceException` in
code that had every right to assume a `Some` held a value — a long way from the
projection that caused it.

### The repair

Project into an option rather than into a value:

```diff
-Option<string> name = user.Map(u => u.MiddleName);
+Option<string> name = user.AndThen(u => Option.FromNullable(u.MiddleName));
```

### What to check

Any delegate passed to those four members that can return null: a nullable reference, a
`FirstOrDefault`, a dictionary lookup, an explicit `return null`. Nullable reference
warnings will point at most of them if you have them switched on, because the delegate's
return type is constrained to a non-nullable type.

### There is no analyzer for this one

A rule would have to prove a delegate can return null across a call boundary, which is
what the nullable reference annotations already do better. Turn those on if they are off.

## Silent change 2: AndThen rejects a null monad

`AndThen` and `AndThenAsync` used to accept a null `Option` or `Result` back from your
factory and hand it onward. They now throw `ArgumentNullException`, naming
`optionFactory` or `resultFactory`.

```csharp
// 7.0.0: ArgumentNullException if the cache returns null.
Option<Order> order = id.AndThen(i => _cache[i]);
```

### The repair

Return the empty case rather than null:

```diff
-Option<Order> order = id.AndThen(i => _cache[i]);
+Option<Order> order = id.AndThen(i => Option.FromNullable(_cache[i]));
```

`OrElse` is not affected. A factory that produces the fallback runs only when there is
nothing to carry forward, so a null there has never been ambiguous.

## Silent change 3: a scope disposed out of order restores nothing

A `MonadOptionsScope` now restores only when it is the innermost scope still open.
Dispose it at any other time and nothing is restored — the library writes a
`Waystone.Monads.ScopeDisposedOutOfOrder` diagnostic event instead.

In 6.x the same mistake restored the *outer* scope's predecessor and silently discarded
the inner scope, so the options in effect afterwards were wrong and nothing said so.

```csharp
var outer = MonadOptions.BeginScope(o => o.UseFallbackErrorCode("Outer"));
var inner = MonadOptions.BeginScope(o => o.UseFallbackErrorCode("Inner"));

outer.Dispose(); // 6.x: silently wrong. 7.0.0: nothing restored, event written.
inner.Dispose();
```

### The repair

Use `using`. It disposes in reverse order for you, and nothing else guarantees it.

```diff
-var scope = MonadOptions.BeginScope(o => o.UseFallbackErrorCode("Debug"));
-// ...
-scope.Dispose();
+using (MonadOptions.BeginScope(o => o.UseFallbackErrorCode("Debug")))
+{
+    // ...
+}
```

### What to check

Every scope held in a field or a local rather than a `using`. Also every scope disposed
from a different asynchronous flow than the one that opened it — a scope lives in the
flow, so another flow's `Dispose` never sees it and now reports.

Full contract on
[Configuration](../../guides/configuration.md#what-happens-when-you-dispose-out-of-order);
the event and a subscriber on
[Observability](../../guides/observability.md#watching-for-a-scope-disposed-out-of-order).

## Loud change: the implicit conversions are gone

`T` no longer converts to `Option<T>`, and `TOk` and `TErr` no longer convert to
`Result<TOk, TErr>`.

```diff
-Option<int> count = 5;
+Option<int> count = Option.Some(5);
```

```diff
-Result<int, string> parsed = 5;
+Result<int, string> parsed = Result.Ok<int, string>(5);
```

**A code fix handles this.** It appears on the `CS0029` or `CS1503` error, and
**Fix all occurrences in Project** applies it in a batch. Where a `Result` carries the
same type on both sides it offers both `Ok` and `Err` and does not choose for you — which
is the whole point.

### Why there was no warning release

This is the one removal in 7.0.0 that was not obsoleted first, and the reason is
mechanical. An `[Obsolete]` implicit conversion still takes part in overload resolution.
Marking it would have produced a warning and left the conversion working, so the silent
wrong-branch behaviour it was removed to prevent would have carried on for another major
version. There was no ordering that gave both a warning and a fix.

## Loud change: configuration moved to a builder

`MonadOptions` now publishes an immutable snapshot, and the `Use…` methods moved to a
new `MonadOptionsBuilder`.

**Most call sites do not change**, because the lambda parameter's type is inferred:

```csharp
// Compiles against 6.x and against 7.0.0.
MonadOptions.Configure(options => options.UseFallbackErrorCode("Unknown"));
```

Three things do break:

* An explicitly typed lambda parameter. `(MonadOptions options) =>` becomes
  `(MonadOptionsBuilder options) =>`, or drop the annotation.
* A field, parameter, local or property typed `MonadOptions`. There is no replacement —
  configuration is reachable only inside a `Configure` or `BeginScope` callback now.
* `MonadOptionsExtensions` in the satellite packages, now
  `MonadOptionsBuilderExtensions`. Only a `using static` or a qualified static call is
  affected.

Do not keep the builder past the callback. Calls on it afterwards are discarded without
an error. Full model on
[Configuration](../../guides/configuration.md#how-configuration-works).

## Loud change: the extension classes collapsed

The per-family extension classes (`AndThenExtensions`, `MapExtensions`,
`IsSomeAndExtensions` and the rest) are now one class per monad: `OptionExtensions` and
`ResultExtensions`.

```diff
-using static Waystone.Monads.Options.Extensions.AndThenExtensions;
-
-Option<Order> order = AndThenAsync(option, Load);
+Option<Order> order = await option.AndThenAsync(Load);
```

Called as extensions, the normal way, nothing changes. Only a `using static` or a
qualified static call breaks, as `CS0234` or `CS0103`.

These could not overlap for a version either: two static classes declaring the same
extension member for the same receiver is `CS0121`, and `[Obsolete]` does not remove a
member from overload resolution.

{% hint style="danger" %}
**Fix the `using static` lines first.** `CS0234` is a declaration-phase error, so it
stops the compiler reporting anything in the body of every file that has the `using`.
Parameter renames and conversion errors in those files stay invisible until the qualifier
is fixed, and your first green build is not the whole job.
{% endhint %}

## Loud change: five obsolete members are gone

All five were `[Obsolete]` in 6.x with a message naming the replacement.

| Removed | Replacement | Diagnostic |
| --- | --- | --- |
| `MonadOptions.UseExceptionLogger` | `UseLogger`, `UseLoggerFactory` or `UseLoggerFactoryFrom` from `Waystone.Monads.Extensions.Logging` | `CS1061` |
| `ErrorCode.FromEnum` | `[ErrorCodeCatalog]` and the generated `ToErrorCode()` | `CS0117` |
| `Error.FromEnum` | `[ErrorCodeCatalog]` and the generated `{Enum}Catalog.Errors.{Member}(message)` | `CS0117` |
| `ErrorCodeFactory.FromEnum` | `[ErrorCodeCatalog]`. Enum codes are settled at compile time now, so a factory cannot change them. | `CS0115` |
| `Result.Err<TOk>(Enum, string)` | `Result.Err(code.ToError(message))` | `CS1501` |

{% hint style="danger" %}
**`ErrorCodeFactory.FromEnum` masks other errors too.** An override of it is `CS0115` at
the declaration, so a codebase with both an override and call sites sees only the
override error, fixes it, and discovers the call sites on the next build. Build twice.
{% endhint %}

See [Generated error codes](../../source-generation/README.md) for the
`[ErrorCodeCatalog]` route.

## Loud change: parameter names

Parameters were renamed across `Option`, `Result` and their case types, for consistency:
`default` became `defaultValue`, `else` became `valueFactory`, `createDefault` became
`defaultFactory`, and `createOther` became `optionFactory` or `resultFactory`.

**Positional calls are unaffected.** Argument order did not change, so this only breaks a
call that names its arguments, as `CS1739`.

```diff
-option.MapOr(default: 0, map: x => x + 1);
+option.MapOr(defaultValue: 0, map: x => x + 1);
```

A code fix substitutes the new name. Do not switch to positional arguments to dodge it —
the name was chosen to say what the argument is for.

## Loud change: TryAsync and CollectAsync return ValueTask

v6 made every async *extension* return `ValueTask` and left the static members alone.
v7 finishes the job. `Option.TryAsync`, `Result.TryAsync` and `CollectAsync` returned
`Task` up to 6.7.0 and return `ValueTask` now, so the rule has no exceptions left.

**If you await the call, nothing changes.** You await a `ValueTask` the same way. It
breaks only where you name the type or hand the task to something that wants a `Task`:

```diff
-Task<Option<int>> pending = Option.TryAsync(() => FetchAsync());
+ValueTask<Option<int>> pending = Option.TryAsync(() => FetchAsync());
```

```diff
-await Task.WhenAll(Result.TryAsync(A, Fail), Result.TryAsync(B, Fail));
+await Task.WhenAll(
+    Result.TryAsync(A, Fail).AsTask(),
+    Result.TryAsync(B, Fail).AsTask());
```

You get `CS0029` on the assignment and `CS1503` on the `Task.WhenAll` call. There is no
code fix. Two things to carry over from the v6 change:

* **Await it once, and only once.** This matters when you store it first, as above.
* `.AsTask()` allocates. Only reach for it where a `Task` is genuinely required.

If you were following [v5.x to v6.x](../older/v5-to-v6.md#loud-change-async-extensions-all-return-valuetask),
it told you to leave `TryAsync` alone. That advice held for v6 and stops holding here.

## Analyzer rules that changed

### Added

| Rule | Severity | What it reports |
| --- | --- | --- |
| [`WM2022`](../../analyzers/idioms.md#wm2022) | Suggestion | A `Task`-returning method group passed to `AndThenAsync` or `OrElseAsync`, whose step returns a `ValueTask` |

Two more ship in the new test package rather than in the library:
[`WMS2001` and `WMS2002`](../../analyzers/assertion-rules.md).

### Removed

| Rule | Why |
| --- | --- |
| `WM2010` | It reported a `Result<T, T>` whose two implicit conversions were ambiguous. There are no implicit conversions left for it to report on. |

Remove any `.editorconfig` entry or `#pragma` naming `WM2010`. A retired id is never
reused, so a stale entry does nothing at all — it neither errors nor warns.

## New, and worth a look once the build is clean

* **[Waystone.Monads.Shouldly](../../packages/shouldly.md)** — assertions that
  take the monad, so a failing test names the `None` or `Err` it found. `WMS2001` and
  `WMS2002` convert an existing suite in a batch.
* **[Waystone.Monads.Linq](../../packages/linq.md)** — C# query syntax over
  `Option` and `Result`.
* **[Severity presets](../../analyzers/severity-presets.md)** — one MSBuild
  property makes the seven misuse rules build errors. A major upgrade is a reasonable
  moment to turn it on.

Do these after the upgrade, not during it.

## Everything on one page

| Change | Breaks the build? | What to do |
| --- | --- | --- |
| A projection returning null throws | **No** | Project with `AndThen` and `Option.FromNullable` |
| `AndThen` rejects a null monad | **No** | Return `None` or an `Err` rather than null |
| A scope disposed out of order restores nothing | **No** | Use `using` |
| Implicit conversions removed | **Yes** | Take the code fix on `CS0029` / `CS1503` |
| Configuration moved to a builder | **Yes**, only for explicit types | Let the lambda parameter infer |
| Extension classes collapsed | **Yes** | Drop the `using static`; call as an extension |
| Five obsolete members removed | **Yes** | See the table above |
| Parameter renames | **Yes**, only for named arguments | Take the code fix on `CS1739` |
| `TryAsync` and `CollectAsync` return `ValueTask` | **Yes**, only if you name the type | Change the declared type, or add `.AsTask()` |
| `WM2010` retired | **No** | Delete any `.editorconfig` entry for it |
