---
description: >-
  Pass your data to a delegate instead of capturing it, and stop allocating a
  closure on every call.
icon: layer-group
---

# State overloads

Nearly every method that takes a delegate has a sibling that takes **your data
first** and hands it to the delegate. Use it when the delegate would otherwise
capture a local or a parameter.

```diff
-option.Map(value => value + offset);
+option.Map(offset, static (value, state) => value + state);
```

Both lines do the same thing. The second one allocates 88 fewer bytes every time
it runs — 24 for the display class the closure needs, 64 for the delegate.

This page applies to both `Option<T>` and `Result<T, E>`.

## Where you can use it

| Type | Methods |
| --- | --- |
| `Option<T>` | `IsSomeAnd`, `IsNoneOr`, `Match`, `Map`, `MapOr`, `MapOrDefault`, `MapOrElse`, `AndThen`, `Filter`, `Inspect`, `UnwrapOrElse`, `OrElse`, `OkOrElse` |
| `Result<T, E>` | `IsOkAnd`, `IsErrAnd`, `Match`, `Map`, `MapOr`, `MapOrDefault`, `MapOrElse`, `MapErr`, `AndThen`, `OrElse`, `UnwrapOrElse`, `Inspect`, `InspectErr` |
| Factories | `Option.Try`, `Option.TryAsync`, `Result.Try`, `Result.TryAsync` |

`ZipWith` and `Reduce` are the exceptions, and they are not getting one. Both
already hand their delegate every value the call involves, so there is normally
nothing left for it to capture.

## What the delegate receives

The state is always the first argument of the call. What your delegate receives
depends on whether the branch it runs in has a value to give it.

On `Result<T, E>` every delegate takes the value and then the state — the success
value or the error, depending on which branch it is.

```csharp
result.UnwrapOrElse(fallback, static (error, state) => state);
```

On `Option<T>` the delegates that run when the option is `None` take the state
alone, because there is no value to pass. That is the `onNone` branch of `Match`,
and all of `UnwrapOrElse`, `OrElse` and `OkOrElse`.

```csharp
option.UnwrapOrElse(fallback, static state => state);
```

## Match saves the most

`Match` takes two delegates, so a capturing call pays twice. The two branches
share one display class, but each one needs its own delegate — 152 bytes a call
rather than 88.

```diff
-option.Match(
-    name => name.Length + offset,
-    () => fallback);
+option.Match(
+    (offset, fallback),
+    static (name, state) => state.offset + name.Length,
+    static state => state.fallback);
```

State is a single argument, so pass a tuple when you need more than one value.
This works for both forms of `Match` — the one that returns a value, and the one
that takes `Action` delegates and returns nothing.

## Write the lambda `static`

This is the part that is easy to get wrong. A lambda that happens not to capture
measures the same as a `static` one — the compiler caches both. But nothing stops
a later edit from reaching for an outer variable again, and the allocation comes
straight back with no warning.

Marking the lambda `static` makes the compiler enforce it:

```csharp
// the compiler rejects any capture in here
option.Map(offset, static (value, state) => value + state);
```

Use `static` in every state overload you write.

## MapOrElse threads state through both delegates

`MapOrElse` takes two delegates and hands the same state to each one. Passing it
to the map alone would leave `createDefault` capturing, and the allocation would
still be there.

```csharp
option.MapOrElse(
    fallback,
    static state => state,
    static (value, state) => value + state);
```

## How the overloads stay apart

Each state overload adds its parameter at the front and takes exactly one more
argument than its closure sibling. That is what keeps the compiler from having to
choose between them. Do not "tidy" a future overload by reusing an existing slot.

## On the async surface

Some of the `…Async` methods take state as well, but not all of them yet.

| Type | Async methods that take state |
| --- | --- |
| `Option<T>` | `IsNoneOrAsync`, `InspectAsync`, `MapOrDefaultAsync` |
| `Result<T, E>` | `IsOkAndAsync`, `IsErrAndAsync`, `MatchAsync`, `InspectAsync`, `InspectErrAsync`, `MapOrDefaultAsync` |

Everywhere else on the
[async surface](../guides/async.md#the-full-surface) you still need a closure.
Where the allocation matters, `await` the task first and call the synchronous
overload on the result.

{% hint style="info" %}
[`WM2017`](../analyzers/idioms.md#wm2017) reports a delegate that
captures where one of these overloads exists, so you do not have to find them by
hand.
{% endhint %}
