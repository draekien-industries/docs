---
description: >-
  Three changes in v6 keep compiling and change what your code does. Read this
  page before you upgrade, and start with the two silent ones.
---

# v5.x to v6.x

## Upgrade with an agent

Copy this into Claude Code or a similar tool, pointed at your solution. It covers
every mechanical part of the upgrade.

````text
Upgrade this solution from Waystone.Monads v5 to v6. Work through these steps in
order and report what you changed at each one.

1. Find every call to `Option.Try(`, `Option.TryAsync(`, `Result.Try(` and
   `Result.TryAsync(`. For each one, decide whether the factory is asynchronous.
   If it is, and the call uses `Try` rather than `TryAsync`, change it to
   `TryAsync` and add an `await` at the point the caller already awaits. Do not
   rely on the compiler to find these — most of them still compile after the
   upgrade and silently stop catching exceptions.

2. Find every `Try` and `TryAsync` call whose factory can be cancelled — it takes
   a CancellationToken, or calls something that does. In v6 an
   OperationCanceledException propagates instead of becoming a None or an Err.
   Add a `catch (OperationCanceledException)` where the caller needs to handle
   cancellation, or tell me if you think we should opt back into the old
   behaviour with `MonadOptions.UseCancellationAsFailure()`.

3. Rename every `FlatMap` to `AndThen` and every `FlatMapAsync` to
   `AndThenAsync`. Parameters and behaviour are unchanged.

4. Find every call that hands an async delegate to a synchronous Waystone
   method — `option.Map(x => FetchAsync(x))` and the like — and switch it to the
   Async sibling with an `await`. You can spot these by the return type: an
   `Option<Task<T>>` or a `Result<Task<T>, E>` is always wrong. A `Task<T>`
   returned by `Match` or `MapOr` is fine, because the caller can await it.

5. Build. For every CS0029 or CS1503 involving `Task` and `ValueTask` on a
   Waystone async extension, add `.AsTask()` to the call. If the site simply
   awaits the result, remove the annotation instead of adding `.AsTask()`.

6. Find every `Task.WhenAll` over Waystone async extensions and add `.AsTask()`
   to each argument.

7. Delete every `catch (InvalidOperationException)` that wraps an `Option.Some`
   call. It is dead code in v6. If the code needs to handle a null, catch
   `ArgumentNullException` instead.

8. Find any type that derives from `Option<T>` or `Result<TOk, TErr>`. These no
   longer compile and cannot be fixed by changing the derived type. Report them
   to me with a suggestion for composing the monad instead.

9. Remove any `.editorconfig` entries for WM1004, WM1007, WM1009, WM1010 and
   WM2014. Those rules no longer exist.

10. Build again and report anything left over. Do not suppress a WM1011 warning
    — bring it to me instead.
````

{% hint style="warning" %}
**One step it cannot do for you.** `Option.Some(0)` returns a `Some` in v6 where
it used to throw, and `Option<int> x = 0;` gives you `Some(0)` where it used to
give you `None`. Deciding whether an `IsNone` branch was standing in for "zero"
needs someone who knows what the code means. See
[Silent change 3](#silent-change-3-some-accepts-value-type-defaults).
{% endhint %}

## Read this first

v6 has three changes that do **not** break your build:

1. **`Option.Try(() => SomethingAsync())` stops catching exceptions.** It still
   compiles. It now returns `Option<Task<T>>`.
2. **A cancelled operation inside `Try` now propagates** instead of becoming
   `None` or an `Err`.
3. **`Option.Some(0)` returns a `Some`** where it used to throw, and
   `Option<int> x = 0;` gives you `Some(0)` where it used to give you `None`.

Everything else in v6 breaks loudly. The compiler will find it for you.

Work through the three sections below in order. The
[agent prompt](#upgrade-with-an-agent) above covers everything mechanical.

## Silent change 1: Try with an async factory

**This is the most dangerous change in the release.** It compiles, it runs, and
it silently stops handling exceptions.

v5 deprecated `Option.Try(Func<Task<T>>)` and
`Result.Try(Func<Task<TOk>>, Func<Exception, TErr>)`. v6 deletes them.

Deleting them does not leave you with a compiler error, because the synchronous
overload can take their place. `Option.Try<T>(Func<T>)` constrains `T` to
`notnull`, and a `Task<int>` is not null:

```csharp
// v5: Option<int>, exceptions caught, awaited inside Try
// v6: Option<Task<int>>, exceptions NOT caught, task never awaited
var result = Option.Try(() => FetchCountAsync());
```

`T` binds to `Task<int>`. `Try` calls the factory, gets a task back, wraps it in
a `Some`, and returns. It never awaits, so:

- **Your exception handling is gone.** A throw inside `FetchCountAsync` escapes
  to your caller. It does not become a `None` or an `Err`, and your
  [configured exception logger](../configuration.md) never sees it.
- **The task may never be awaited**, depending on what you do with the result.

You only get a compiler error if the call site assigns to an explicitly typed
local:

```csharp
// CS0029 — this one is safe, the compiler catches it
Task<Option<int>> safe = Option.Try(() => FetchCountAsync());
```

### The repair

Call `TryAsync` and `await` where you were already awaiting:

```diff
-var result = Option.Try(() => FetchCountAsync());
+var result = await Option.TryAsync(() => FetchCountAsync());

-var result = Result.Try(() => FetchCountAsync(), ex => ex.Message);
+var result = await Result.TryAsync(() => FetchCountAsync(), ex => ex.Message);
```

### The analyzer finds these for you

[`WM1011`](../analyzer-rules.md#wm1011) is a **warning**, not a suggestion,
because it fires on code that runs. It reports any call that traps a task inside
an `Option` or a `Result` — `Try` with an async factory, but also
`option.Map(x => FetchAsync(x))` and anything else with an `Async` sibling it
should have used.

It ships with no quick fix, deliberately. Renaming to the `Async` sibling leaves
you with an unawaited task, and no fix can decide where your `await` belongs.

{% hint style="info" %}
`WM1011` also catches this mistake in code that predates v6, where someone passed
an async factory to the synchronous overload by accident. That is a real bug
being found, not an upgrade artefact.
{% endhint %}

## Silent change 2: cancellation propagates

In v5, `Try` and `TryAsync` caught **every** exception, including
`OperationCanceledException`. A cancelled operation came back as a `None` or an
`Err`, indistinguishable from a genuine failure.

In v6 they let cancellation through:

```csharp
using var cts = new CancellationTokenSource();
cts.Cancel();

// v5: None<int>, and your exception logger records the cancellation
// v6: throws OperationCanceledException
Option<int> option = await Option.TryAsync(() => FetchAsync(cts.Token));
```

`TaskCanceledException` derives from `OperationCanceledException`, so it
propagates too.

### Why we changed it

Cancellation is not a failure. It is you telling the operation to stop. Swallowing
it turns a deliberate shutdown into what looks like a bad result, and the caller
that requested the cancellation then has to guess whether the `None` it received
means "cancelled" or "genuinely absent".

### What to check

Look for code that relied on a cancellation becoming a `None` or an `Err`. It now
needs a `catch`:

```diff
-Option<int> option = await Option.TryAsync(() => FetchAsync(token));
-if (option.IsNone) { /* cancelled or failed, cannot tell which */ }
+try
+{
+    Option<int> option = await Option.TryAsync(() => FetchAsync(token));
+}
+catch (OperationCanceledException)
+{
+    // handle the cancellation
+}
```

### If you want the old behaviour

Opt back in once, at startup:

```csharp
MonadOptions.Configure(options => options.UseCancellationAsFailure());
```

That restores the v5 behaviour everywhere: a cancellation is caught, logged, and
becomes a `None` or an `Err` again. You can also scope it to one region with
`MonadOptions.BeginScope`. See [Configuration](../configuration.md).

We recommend leaving it off. The opt-in exists so that upgrading is not blocked
on rewriting every call site at once.

## Silent change 3: Some accepts value-type defaults

In v5, a `Some` cannot hold the default of its type. `Option.Some(0)` throws, and
`Option<int> x = 0;` gives you `None`.

In v6, only `null` is rejected. `Option.Some(0)` gives you a `Some` holding `0`,
and so does `Option<int> x = 0;`.

We made this change because `Option<int>` could not represent part of its own
domain. Zero is an ordinary integer. `Option<bool>` was close to useless, because
`false` is an ordinary bool.

### The expressions that flip

Nothing fails to compile. No signature changes. Your build stays green and these
expressions start returning a different value:

| Expression | v5 | v6 |
| --- | --- | --- |
| `Option.Some(0)` | throws | `Some(0)` |
| `Option<int> x = someInt;` when `someInt` is `0` | `None` | `Some(0)` |
| `Option.FromNullable(nullableInt)` when the value is `0` | `None` | `Some(0)` |
| `Option.Try(() => ComputeCount())` when the count is `0` | `None` | `Some(0)` |
| `option.Map(x => x - x)` | `None` | `Some(0)` |
| `option.Reduce(...)` producing a default | `None` | `Some(default)` |

The same applies to `false`, `'\0'`, `Guid.Empty`, `DateTime.MinValue`,
`DateTimeOffset.MinValue`, `TimeSpan.Zero`, `IntPtr.Zero` and any enum's zero
member.

### What to do

1. **Search for `Option<` over a value type.** Every one is a candidate.
2. **Check each `IsNone` branch on those options.** Ask whether it treats a zero
   as "no result". If it does, that branch stops running in v6.
3. **Delete any `catch (InvalidOperationException)` around `Option.Some`.** It is
   dead code now.

There is no automatic migration, and the analyzer cannot do this for you. A rule
that fired on every `Option<T>` where `T` is a value type would fire on most of
the library's users.

{% hint style="warning" %}
`WM1010` shipped in 5.5.0 to warn you about exactly this, and v6 removes it — the
change it forecast has happened. Upgrade to 5.5.0 **first**, fix what `WM1010`
reports, then move to v6. It only reaches the call sites where it can prove the
value is a default, so it covers none of the six expressions in the table above.
{% endhint %}

### Null still throws, with a different exception

`Option.Some(null!)` throws in v6, as it did in v5. The exception type changes
from `InvalidOperationException` to `ArgumentNullException`.

Use [`Option.FromNullable`](../core-functionality.md) when the value may be null.

`FromNullable<T>(T?) where T : struct` no longer rejects the default either, so it
now behaves the same way as its reference-type sibling.

### UnwrapOrDefault gets harder to read

This follows directly from the relaxation. Both of these return `0`:

```csharp
Option.None<int>().UnwrapOrDefault();  // 0, because there is no value
Option.Some(0).UnwrapOrDefault();      // 0, because the value is 0
```

You cannot tell them apart. This is not new in kind — `Result` has always been in
this position — but it now applies to every `Option` over a value type.

Use `UnwrapOrNull` and `MapOrNull`, shipped in 5.4.0, when you need to
distinguish:

```csharp
int? absent = Option.None<int>().UnwrapOrNull();  // null
int? present = Option.Some(0).UnwrapOrNull();     // 0
```

[`WM2015`](../analyzer-rules.md#wm2015) points you at them.

## Loud change: async extensions all return ValueTask

Every async extension on `Option` and `Result` now returns `ValueTask` or
`ValueTask<T>`. 185 signatures changed. In v5 some returned `Task` and some
returned `ValueTask`; now the rule is uniform.

You get `CS0029` or `CS1503` at every affected call site. Nothing changes
silently.

### The repair

Add `.AsTask()` where you need a `Task`:

```diff
-Task<Option<int>> task = option.MapAsync(FetchAsync);
+Task<Option<int>> task = option.MapAsync(FetchAsync).AsTask();
```

Your IDE offers this fix automatically. It registers against the compiler's own
`CS0029` and `CS1503`, so it appears wherever the error appears.

If you simply `await` the result, nothing changes at all.

### Where this costs you

`Task.WhenAll` needs `Task` arguments, so fan-out code now has to call `.AsTask()`
on each one — which allocates the very `Task` the change avoids:

```csharp
await Task.WhenAll(
    a.MapAsync(FetchAsync).AsTask(),
    b.MapAsync(FetchAsync).AsTask());
```

This is the one place the change makes your code worse.

### The measured trade-off

`ValueTask` is not unconditionally cheaper. For a three-link chain:

| Receiver | Change |
| --- | --- |
| Synchronous `Option`, `Some` | −144 B (−33%) |
| Synchronous `Option`, `None` | −144 B (−67%) |
| Already-completed `Task` | −216 B |
| **Genuinely pending task** | **+84 B (+10%)** |

A fluent chain is built eagerly, so a pending head makes every link pending —
there is no partial case. `AsyncValueTaskMethodBuilder` holds the state machine
inline in its box, which makes it larger than the `Task` it replaces.

Full numbers are in `bench/Waystone.Monads.Benchmarks/README.md` in the source
repository.

## Loud change: FlatMap is gone

`Option<T>.FlatMap` and the five `FlatMapAsync` extensions are deleted. Rename to
`AndThen` and `AndThenAsync`:

```diff
-Option<int> option = Find(id).FlatMap(Parse);
+Option<int> option = Find(id).AndThen(Parse);
```

The parameters, the behaviour and the return type are unchanged.

`WM2014`, the rule that reported every `FlatMap` call, is removed in v6 — with no
`FlatMap` left it would report nothing forever. Upgrade to 5.5.0 first and let it
build your to-do list.

## Loud change: you can no longer derive from Option or Result

`Option<T>` and `Result<TOk, TErr>` are closed. `Some`, `None`, `Ok` and `Err` are
the only cases, and an outside type that tries to add a third gets `CS0534` on an
internal member it cannot see or override.

**There is no way around this and no migration path.** If you added a case, you
have to compose the monad instead of inheriting it — hold an `Option<T>` in your
type rather than being one.

`WM1007`, the v5 warning that told you not to derive, is gone in v6 because the
compiler now says it instead.

Six members moved from the base type to the cases and are now `abstract`:

- `Option.And`, `Option.MapOrDefault`, `Option.Reduce`, `Option.AsEnumerable`
- `Result.MapOrDefault`, `Result.AsEnumerable`

Behaviour is identical. This only affects anyone who **overrode** them on a
derived case — the same people the closed hierarchy already stops.

## Fixed: Unzip no longer throws on a defaulted component

`Option.Some((0, "x")).Unzip()` threw in v5. It now returns
`(Some(0), Some("x"))`.

This falls out of the `Some` relaxation. We considered returning `None` for the
defaulted component instead and rejected it: `UnwrapOr(-1)` would then hand back
`-1` for a `0` that was genuinely there.

If you wrote defensive code around the old throw, delete it rather than
re-pointing it.

## Changed: two Nones of the same type are now the same object

`None<T>` is a cached singleton in v6, so `Option.None<int>()` returns the same
instance every time.

```csharp
// v5: false
// v6: true
ReferenceEquals(Option.None<int>(), Option.None<int>());
```

Equality and hashing are unchanged — two `None` values were already equal, and
still are. Only `ReferenceEquals` answers differently.

This removes an allocation from every operation that produces a `None`:
`Option.None<int>()` went from 1.65 ns and 24 B to 0.18 ns and 0 B.

{% hint style="info" %}
A `with` expression goes through the compiler-generated clone, not the factory, so
it still hands back a second instance. The singleton is a guarantee of
`Option.None<T>()`, not of the type.
{% endhint %}

## New: state overloads that avoid a closure

Every hot-path transform gained a sibling that takes your data as an argument and
hands it to the delegate, so the delegate captures nothing:

```diff
-option.Map(value => value + offset);
+option.Map(offset, static (value, state) => value + state);
```

This is purely additive. No existing signature changed and there is nothing to
migrate.

Covered methods:

- `Option`: `Map`, `MapOr`, `MapOrElse`, `Filter`, `AndThen`
- `Result`: `Map`, `MapOr`, `MapOrElse`, `MapErr`
- `Option.Try`, `Option.TryAsync`, `Result.Try`, `Result.TryAsync`

The closure costs exactly 88 bytes at every call site — 24 for the display class,
64 for the delegate — and the state overload removes all of it.

See [Core Functionality](../core-functionality.md#state-overloads) for the detail,
including why the `static` keyword matters.

[`WM2017`](../analyzer-rules.md#wm2017) points you at these when it sees a
delegate that captures.

## Analyzer rules that changed

### Removed

| Rule | Why |
| --- | --- |
| `WM1004` | Described the default-value invariant, which no longer exists |
| `WM1007` | Deriving from `Option` or `Result` is a compile error now |
| `WM1009` | `Option<bool>` is genuinely useful in v6, so the advice is withdrawn |
| `WM1010` | Forecast the `Some` relaxation, which has now happened |
| `WM2014` | There is no `FlatMap` left to report |

None of these IDs will be reused. If you suppressed one in `.editorconfig`, drop
the entry.

### Added

| Rule | Severity | What it reports |
| --- | --- | --- |
| [`WM1011`](../analyzer-rules.md#wm1011) | Warning | An async delegate passed to a synchronous method |
| [`WM2016`](../analyzer-rules.md#wm2016) | Suggestion | An eager argument that is not free to evaluate |
| [`WM2017`](../analyzer-rules.md#wm2017) | Suggestion | A delegate that captures where a state overload exists |

### Reworded

`WM1001` and `WM1005` now describe `Some` as rejecting null rather than rejecting
the default of the type, and `WM1001` names `ArgumentNullException`. `WM2015` now
names the value it hands back. No behaviour changed in any of the three.

## Everything on one page

| Change | Breaks the build? | What to do |
| --- | --- | --- |
| `Try` with an async factory | **No** | Switch to `TryAsync` and `await` |
| Cancellation propagates | **No** | Catch it, or `UseCancellationAsFailure()` |
| `Some` accepts value-type defaults | **No** | Review every `IsNone` on a value type |
| Async extensions return `ValueTask` | Yes | Add `.AsTask()` |
| `FlatMap` removed | Yes | Rename to `AndThen` |
| Hierarchies closed | Yes | Compose instead of inherit |
| `Unzip` fixed | No | Delete defensive code |
| `None<T>` is a singleton | No | Nothing, unless you used `ReferenceEquals` |
| State overloads added | No | Nothing — adopt them where it helps |
