---
description: >-
  Chain asynchronous work through Option and Result without awaiting every
  step.
icon: hourglass-half
---

# Async

Most operations have an async counterpart with an `Async` suffix. Use one when
your transform, predicate, or side effect returns a `Task`. The full list is at
the [bottom of this page](#the-full-surface).

## The receiver is the task, not the monad

This is the part worth understanding, and it is not obvious.

Most async methods are extension methods on the **task that wraps the monad**,
not on the monad itself. They extend `Task<Option<T>>`, `ValueTask<Option<T>>`,
`Task<Result<T, E>>` and `ValueTask<Result<T, E>>`.

That is what lets you chain without awaiting each step:

<!-- snippet: async-one-await-at-the-end -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
// no intermediate awaits, one await at the end
Character character = await SummonCharacterAsync(id)
    .MapAsync(c => EnrichAsync(c))
    .UnwrapOrAsync(Commoner);
```
<!-- endSnippet -->

Without them, you would await into a local at every step:

<!-- snippet: async-the-same-chain-step-by-step -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
Option<Character> fetched = await SummonCharacterAsync(id);
Option<Character> enriched = await fetched.MapAsync(c => EnrichAsync(c));

Character character = enriched.UnwrapOr(Commoner);
```
<!-- endSnippet -->

Two things about that first sample, because both catch people out:

* **`EnrichAsync` must return `Task<Character>`.** `MapAsync` takes
  `Func<T, Task<TOut>>` or a plain `Func<T, TOut>`. It does not take a
  `ValueTask` factory.
* **`Commoner` is a value, not a function.** `UnwrapOrAsync` takes `T`, the same
  as `UnwrapOr`. Pass a method group and you get `CS0411`.

{% hint style="info" %}
These overloads are generated, by `Waystone.SourceGenerators`, from the
synchronous methods. That is why the surface is so uniform, and why you will not
find them written out in the library source.
{% endhint %}

{% hint style="info" %}
They live in `Waystone.Monads.Options.Extensions` and
`Waystone.Monads.Results.Extensions`. Add the `using` for the monad you are
working with.
{% endhint %}

`MatchAsync` is the exception — it also extends the monad directly, so you can
pass an async branch to an `Option<T>` or `Result<T, E>` you already hold. See
[Matching with an async branch](#matching-with-an-async-branch).

## Every async member returns ValueTask

From 7.0.0 the rule has no exceptions: **every** async member on `Option` and
`Result` returns `ValueTask` or `ValueTask<T>`. That includes the static
factories. `Option.TryAsync`, `Result.TryAsync` and `CollectAsync` returned
`Task` up to 6.7.0 — see
[Loud change: TryAsync and CollectAsync return ValueTask](../upgrading/v7/from-v6.md#loud-change-tryasync-and-collectasync-return-valuetask).

<!-- snippet: async-every-async-member-returns-valuetask -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
ValueTask<string> output = result.MatchAsync(
    async x => await RenderAsync(x),
    async e => await DescribeAsync(e));
```
<!-- endSnippet -->

You await a `ValueTask` the same way you await a `Task`, so this rarely changes
your code. Two things to know:

* **Await it once, and only once.** This matters if you store it before awaiting.
* **Call `.AsTask()` when you need a `Task`** — most often for `Task.WhenAll`.

<!-- snippet: async-as-task-for-when-all -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
await Task.WhenAll(
    a.MapAsync(FetchAsync).AsTask(),
    b.MapAsync(FetchAsync).AsTask());
```
<!-- endSnippet -->

Both `Task` and `ValueTask` work as receivers, so a chain that mixes them still
composes.

{% hint style="info" %}
`ValueTask` is cheaper when the work finishes synchronously and slightly more
expensive when it does not. A three-link chain saves 144 bytes on a synchronous
`Option` receiver and costs 84 bytes when the head is genuinely pending. See
[v5.x to v6.x](../upgrading/older/v5-to-v6.md#the-measured-trade-off)
for the numbers.
{% endhint %}

## Create a monad from async work

`TryAsync` captures a factory that returns a `Task` and may throw.

{% tabs %}
{% tab title="Option" %}
<!-- snippet: async-option-try-async -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
Option<Character> maybeCharacter = await Option.TryAsync(
    () => SummonCharacterOrThrowAsync(id));
```
<!-- endSnippet -->

If the factory throws, the exception is caught and sent to your
[configured exception logger](configuration.md), and you get
back a `None<Character>`.

You also get a `None<Character>` if the task completes with `null`, because a
`Some` cannot hold one. Nothing is logged in that case, because nothing threw.
{% endtab %}

{% tab title="Result" %}
<!-- snippet: async-result-try-async -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
// supply your own error type
Result<Character, string> result = await Result.TryAsync(
    asyncFactory: () => SummonCharacterOrThrowAsync(id),
    onError: ex => ex.Message);

// or let the error type default to Error
Result<Character, Error> builtIn = await Result.TryAsync<Character>(
    () => SummonCharacterOrThrowAsync(id));
```
<!-- endSnippet -->

The single type parameter overload converts the exception with
`Error.FromException`, so you do not pass an `onError` delegate.

`TryAsync` also calls `onError` when the task completes with `null`, passing you
an `ArgumentNullException` that names the `asyncFactory` argument.
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
**Never pass an async factory to `Try`.** The overloads that accepted one were
removed in 6.0.0, and the call still compiles — it binds to the synchronous
overload, gives you an `Option<Task<T>>`, and catches nothing.
[`WM1011`](../analyzers/runtime-bugs.md#wm1011) reports every
occurrence. See
[Silent change 1](../upgrading/older/v5-to-v6.md#silent-change-1-try-with-an-async-factory).
{% endhint %}

{% hint style="warning" %}
`TryAsync` lets an `OperationCanceledException` through rather than turning it
into a `None` or an `Err`. See
[Configuration](configuration.md#cancellation).
{% endhint %}

## Matching with an async branch

`MatchAsync` works on a monad you already hold, not only on a task that wraps
one. Use it when one or both branches do async work.

<!-- snippet: async-mixing-sync-and-async-branches -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
// both branches async
string text = await option.MatchAsync(
    async x => await RenderNumberAsync(x),
    async () => await LoadDefaultAsync());

// only the Some branch is async
string fromSome = await option.MatchAsync(
    async x => await RenderNumberAsync(x),
    () => "none");

// only the None branch is async
string fromNone = await option.MatchAsync(
    x => x.ToString(),
    async () => await LoadDefaultAsync());
```
<!-- endSnippet -->

Pick the overload that matches your branches. A branch you write as a plain value
stays a plain value instead of being wrapped in a completed task, and the branch
that does not match never runs.

The same shapes work on `Task<Option<T>>` and `ValueTask<Option<T>>`
receivers, so a chain reaches them too.

{% hint style="info" %}
These three arrived on the plain `Option<T>` receiver in 7.0.0. Before that they
existed only on the `Task` and `ValueTask` receivers, so matching an `Option<T>`
you already held meant wrapping it in `Task.FromResult` first.
{% endhint %}

### Result covers the same combinations

Both monads take the same six combinations: three that return a value and three
that return nothing, each with the first branch async, the second async, or both.

<!-- snippet: async-mixing-branches-on-a-result -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
// both branches async
string text = await result.MatchAsync(
    async x => await RenderAsync(x),
    async e => await DescribeAsync(e));

// only the Ok branch is async
string fromOk = await result.MatchAsync(
    async x => await RenderAsync(x),
    e => e.ToString());

// only the Err branch is async
string fromErr = await result.MatchAsync(
    x => x.Name,
    async e => await DescribeAsync(e));
```
<!-- endSnippet -->

Before 7.3.0 the two differed. `Option` had only the shapes that return a value,
and `Result` only the ones that return nothing. Mixing a plain branch with an
async one on a `Result` then bound to the overload returning nothing, and the
assignment failed with `CS0029`.

## End a chain

The consuming operations have async counterparts too, so you can finish a chain
without awaiting the monad first.

{% tabs %}
{% tab title="Option" %}
<!-- snippet: async-consuming-an-option -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
Character character = await SummonCharacterAsync(id).UnwrapAsync();
Character orCommoner = await SummonCharacterAsync(id).UnwrapOrAsync(Commoner);
Character? orDefault = await SummonCharacterAsync(id).UnwrapOrDefaultAsync();
Character expected = await SummonCharacterAsync(id).ExpectAsync("the character must exist");
```
<!-- endSnippet -->
{% endtab %}

{% tab title="Result" %}
<!-- snippet: async-consuming-a-result -->
<!-- source: sample/Waystone.Monads.Docs/Waystone.Monads.Docs.Core.Sample/Guides/Async.cs -->
```csharp
Character character = await LoadCharacterAsync(id).UnwrapAsync();
Character orCommoner = await LoadCharacterAsync(id).UnwrapOrAsync(Commoner);
Character? orDefault = await LoadCharacterAsync(id).UnwrapOrDefaultAsync();
Character expected = await LoadCharacterAsync(id).ExpectAsync("the character must exist");

Error error = await LoadCharacterAsync(id).UnwrapErrAsync();
Error expectedErr = await LoadCharacterAsync(id).ExpectErrAsync("the load must fail");
```
<!-- endSnippet -->
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
**`UnwrapOrDefaultAsync` returns `T?`, not `T`.** Assign it to a nullable local.
With nullable reference types on, `Character orDefault = …` is `CS8600`.
{% endhint %}

{% hint style="info" %}
`UnwrapAsync`, `UnwrapErrAsync`, `ExpectAsync` and `ExpectErrAsync` throw for the
same reasons their synchronous versions do. See
[Exceptions](exceptions.md).
{% endhint %}

## The full surface

Every method below behaves exactly like the synchronous version documented in
[Option\<T>](option.md) and [Result\<T, E>](result.md). The only difference is
that it accepts an async delegate, a task receiver, or both.

{% hint style="info" %}
The ones that take a delegate also take state, so the delegate does not have to
capture — see
[On the async surface](../reference/state-overloads.md#on-the-async-surface).
{% endhint %}

### Option\<T>

| Category | Methods |
| --- | --- |
| Transform | `MapAsync`, `MapOrAsync`, `MapOrDefaultAsync`, `MapOrElseAsync`, `AndThenAsync` |
| State checks | `IsSomeAndAsync`, `IsNoneOrAsync` |
| Consume | `MatchAsync`, `UnwrapAsync`, `UnwrapOrAsync`, `UnwrapOrElseAsync`, `UnwrapOrDefaultAsync`, `ExpectAsync` |
| Side effect | `InspectAsync` |
| Filter and combine | `FilterAsync`, `ZipWithAsync`, `OrElseAsync` |
| Nesting | `FlattenAsync` |
| Conversion | `OkOrAsync`, `OkOrElseAsync` |

### Result\<T, E>

| Category | Methods |
| --- | --- |
| Transform | `MapAsync`, `MapErrAsync`, `MapOrAsync`, `MapOrDefaultAsync`, `MapOrElseAsync` |
| State checks | `IsOkAndAsync`, `IsErrAndAsync` |
| Consume | `MatchAsync`, `UnwrapAsync`, `UnwrapErrAsync`, `UnwrapOrAsync`, `UnwrapOrElseAsync`, `UnwrapOrDefaultAsync`, `ExpectAsync`, `ExpectErrAsync` |
| Side effect | `InspectAsync`, `InspectErrAsync` |
| Logical operators | `AndThenAsync`, `OrElseAsync` |
| Nesting | `FlattenAsync` |
