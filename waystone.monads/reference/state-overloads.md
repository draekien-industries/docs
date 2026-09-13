---
description: >-
  Pass your data to a delegate instead of capturing it, and stop allocating a
  closure on every call.
icon: layer-group
---

# State overloads

There are two ways to hand your data to a delegate instead of letting it capture.
Bind the data to the receiver with `With`, or pass it as the call's first
argument. Both remove the closure.

```diff
-option.Map(value => value + offset);
+option.With(offset).Map(static (value, state) => value + state);
```

Both lines do the same thing. The second allocates 88 fewer bytes every time it
runs — 24 for the display class the closure needs, 64 for the delegate.

**Use `With` first.** It reads in call order, it covers the whole async surface
where the overloads cover only part of it, and it is what
[`WM2017`](../analyzers/idioms.md#wm2017) tells you to use.

This page applies to both `Option<T>` and `Result<T, E>`.

## `With` comes from the extensions namespace

`With` is an extension method. Add `using Waystone.Monads.Options.Extensions;`
for `Option<T>`, or `using Waystone.Monads.Results.Extensions;` for
`Result<T, E>`.

If `With` does not turn up on your option, that using is what is missing. It is
the first thing about this API that looks broken.

## Bind the data, then call the method

`With(state)` gives you a binder carrying the same methods you already know. Each
one hands your data to the delegate as its **last** argument, so the value comes
first and reads the way it always did.

<!-- snippet: state-overloads-with-map -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
Option<int> share = reward
    .With(partySize)
    .Map(static (gold, party) => gold / party);
```
<!-- endSnippet -->

### The data is spent, not sticky

Every method on the binder returns a plain `Option<T>` or `Result<T, E>`. Your
data is gone the moment one call uses it, so the rest of the chain is ordinary.
Bind again when you need it again.

<!-- snippet: state-overloads-with-does-not-stick -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
Option<int> total = reward
    .With(partySize)
    .Map(static (gold, party) => gold / party) // the state is spent here
    .Filter(static share => share > 0)         // so this is an ordinary call
    .With(bonus)                               // bind again to spend more
    .Map(static (share, extra) => share + extra);
```
<!-- endSnippet -->

There is no method for getting off the binder, because you are never on it for
more than one call.

### Pass a tuple for more than one value

There is one slot. C# names tuple members after the variables you put in them, so
the delegate reads `state.partySize` because you wrote `partySize` — you never
have to invent a name.

<!-- snippet: state-overloads-with-a-tuple -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
// C# names the members after the variables, so nothing is invented
string summary = quest
    .With((partySize, fallback))
    .Match(
        static (found, state) =>
            $"{found.Name} splits {found.GoldReward / state.partySize}",
        static state => state.fallback);
```
<!-- endSnippet -->

### A branch with no value takes the data alone

On `Option<T>` the delegates that run for `None` have no value to receive, so they
take your data by itself. That is the `onNone` branch of `Match`, and all of
`UnwrapOrElse`, `OrElse` and `OkOrElse`.

<!-- snippet: state-overloads-with-no-value-branch -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
// None has no value to hand over, so the delegate takes the state alone
int gold = reward.With(fallback).UnwrapOrElse(static state => state);
```
<!-- endSnippet -->

On `Result<T, E>` every delegate receives a value first — the success value or the
error, depending on the branch it runs in.

<!-- snippet: state-overloads-with-on-a-result -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
Result<Quest, string> tagged = attempt
    .With(realm)
    .MapErr(static (error, where) => $"{where}: {error}");
```
<!-- endSnippet -->

### It works on async delegates

This is the part the overloads cannot do. Every method on the binder has an
`…Async` form, so an async delegate receives your data exactly as a synchronous
one does.

<!-- snippet: state-overloads-with-async -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
Option<int> share = await quest
    .With(partySize)
    .MapAsync(static async (found, party) => await ShareOf(found, party));
```
<!-- endSnippet -->

### The factories bind as well

`Try` and `TryAsync` are static, so you bind on the factory rather than on a
value.

<!-- snippet: state-overloads-with-a-factory -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
Option<int> gold = Option.With(entry).Try(static text => int.Parse(text));
```
<!-- endSnippet -->

## Write the lambda `static`

This applies to both forms, and it is the part that is easy to get wrong. A lambda
that happens not to capture measures the same as a `static` one — the compiler
caches both. But nothing stops a later edit from reaching for an outer variable
again, and the allocation comes straight back with no warning.

Marking the lambda `static` makes the compiler enforce it. Every example on this
page does it — here it is on the overload form, but it makes no difference which
form you pick:

<!-- snippet: state-overloads-the-state-overload -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
// the compiler rejects any capture in here
reward.Map(partySize, static (gold, party) => gold / party);
```
<!-- endSnippet -->

Write `static` every time. It costs nothing, and it is the only part of this the
compiler checks for you.

## Passing the data as the first argument

The older form, and it is not going away. It still works, we still support it,
and for a single call it does marginally less work because there is no binder to
build. Use it where you prefer it.

### Where you can use it

| Type | Methods |
| --- | --- |
| `Option<T>` | `IsSomeAnd`, `IsNoneOr`, `Match`, `Map`, `MapOr`, `MapOrDefault`, `MapOrNull`, `MapOrElse`, `AndThen`, `Filter`, `ZipWith`, `Reduce`, `Inspect`, `UnwrapOrElse`, `OrElse`, `OkOrElse` |
| `Result<T, E>` | `IsOkAnd`, `IsErrAnd`, `Match`, `Map`, `MapOr`, `MapOrDefault`, `MapOrNull`, `MapOrElse`, `MapErr`, `AndThen`, `OrElse`, `UnwrapOrElse`, `Inspect`, `InspectErr` |
| Factories | `Option.Try`, `Option.TryAsync`, `Result.Try`, `Result.TryAsync` |

Every method that takes a delegate has one. `ZipWith` and `Reduce` were the
exceptions until 7.3.0, because both hand their delegate every value the call
involves. That covers the values and nothing else — a combiner still captures a
comparer or a format — so they have a state overload now, and the binder carries
them.

### What the delegate receives

Here your data is always the **first** argument of the call. What the delegate
receives depends on whether the branch it runs in has a value to give it.

On `Result<T, E>` every delegate takes the value and then your data — the success
value or the error, depending on which branch it is.

<!-- snippet: state-overloads-the-state-overload-on-a-result -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
result.UnwrapOrElse(fallback, static (error, state) => state);
```
<!-- endSnippet -->

On `Option<T>` the delegates that run when the option is `None` take your data
alone, because there is no value to pass. That is the `onNone` branch of `Match`,
and all of `UnwrapOrElse`, `OrElse` and `OkOrElse`.

<!-- snippet: state-overloads-the-state-overload-on-an-option -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
option.UnwrapOrElse(fallback, static state => state);
```
<!-- endSnippet -->

### Match saves the most

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

This is one argument too, so pass a tuple when you need more than one value. It
works for both forms of `Match` — the one that returns a value, and the one that
takes `Action` delegates and returns nothing.

### MapOrElse threads the same data through both delegates

`MapOrElse` takes two delegates and hands the same value to each one. Passing it
to the map alone would leave the other delegate capturing, and the allocation
would still be there.

<!-- snippet: state-overloads-map-or-else -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Reference/StateOverloads.cs -->
```csharp
option.MapOrElse(
    fallback,
    static state => state,
    static (value, state) => value + state);
```
<!-- endSnippet -->

### How the overloads stay apart

Each state overload adds its parameter at the front and takes exactly one more
argument than its closure sibling. That is what keeps the compiler from having to
choose between them. Do not "tidy" a future overload by reusing an existing slot.

### On the async surface

On a `Task` or `ValueTask` receiver, every `…Async` method that takes a delegate
has a state overload. The rest take a value or nothing — `UnwrapAsync`,
`FlattenAsync`, `ZipAsync` and the like — so there is no delegate to keep from
capturing.

On a monad you already hold, no `…Async` method takes state, and none is coming.
Use `With` instead, which is what it is for. The binder carries an `…Async` form
of every shape the monad carries, so there is no gap to work around. You never
have to `await` the task early just to get at a synchronous overload.

{% hint style="info" %}
[`WM2017`](../analyzers/idioms.md#wm2017) reports a delegate that captures where
`With` would avoid it, and its quick fix does the rewrite for you, so you do not
have to find these by hand.
{% endhint %}
