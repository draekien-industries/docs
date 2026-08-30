---
description: >-
  Three changes that keep compiling and change what your code does, and eight
  that break the build.
---

# v6.x to v7.0.0

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

## Upgrade with an agent

There is one prompt for this upgrade, and it serves the 5.x path as well. It is on its
own page so both upgrade pages point at the same copy:

* **[Upgrade with an agent](v7-agent-prompt.md)** — copy it into Claude Code or a
  similar tool, pointed at your solution.

{% hint style="warning" %}
**Two steps it cannot do for you**, both in step 1 of the prompt. Whether a null
projection meant "absent" or was never supposed to happen is a domain question, and only
you can answer it. Read
[What it cannot do for you](v7-agent-prompt.md#what-it-cannot-do-for-you) before you
run it.
{% endhint %}

## Read this first

Three changes keep compiling and change what your code does:

1. [A projection that returns null now throws](#silent-change-1-a-null-projection-throws).
2. [`AndThen` rejects a factory that returns a null monad](#silent-change-2-andthen-rejects-a-null-monad).
3. [A scope disposed out of order restores nothing](#silent-change-3-a-scope-disposed-out-of-order-restores-nothing).

Everything else in 7.0.0 breaks loudly. The compiler will find it for you, and
[Every v7 break](v7-breaks.md) lists which diagnostic each one produces.

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
[Configuration](../guides/configuration.md#what-happens-when-you-dispose-out-of-order);
the event and a subscriber on
[Observability](../guides/observability.md#watching-for-a-scope-disposed-out-of-order).

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
[Configuration](../guides/configuration.md#how-configuration-works).

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

See [Generated error codes](../source-generation/README.md) for the
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

If you were following [v5.x to v6.x](v5-to-v6.md#loud-change-async-extensions-all-return-valuetask),
it told you to leave `TryAsync` alone. That advice held for v6 and stops holding here.

## Analyzer rules that changed

### Added

| Rule | Severity | What it reports |
| --- | --- | --- |
| [`WM2022`](../analyzers/idioms.md#wm2022) | Suggestion | A `Task`-returning method group passed to `AndThenAsync` or `OrElseAsync`, whose step returns a `ValueTask` |

Two more ship in the new test package rather than in the library:
[`WMS2001` and `WMS2002`](../analyzers/assertion-rules.md).

### Removed

| Rule | Why |
| --- | --- |
| `WM2010` | It reported a `Result<T, T>` whose two implicit conversions were ambiguous. There are no implicit conversions left for it to report on. |

Remove any `.editorconfig` entry or `#pragma` naming `WM2010`. A retired id is never
reused, so a stale entry does nothing at all — it neither errors nor warns.

## New, and worth a look once the build is clean

* **[Waystone.Monads.Shouldly](../packages/shouldly.md)** — assertions that
  take the monad, so a failing test names the `None` or `Err` it found. `WMS2001` and
  `WMS2002` convert an existing suite in a batch.
* **[Waystone.Monads.Linq](../packages/linq.md)** — C# query syntax over
  `Option` and `Result`.
* **[Severity presets](../analyzers/severity-presets.md)** — one MSBuild
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
