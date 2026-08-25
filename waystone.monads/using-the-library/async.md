---
description: >-
  Learn how to chain asynchronous work through Option and Result without
  awaiting every step.
---

# Async

## Introduction

Most core operations have an async counterpart with an `Async` suffix. Use them
when your transform, predicate, or side effect returns a `Task`. The full list is
at the [bottom of this page](#the-full-surface).

The async methods are extension methods on the **task that wraps the monad**, not
methods on the monad itself. They extend `Task<Option<T>>`,
`ValueTask<Option<T>>`, `Task<Result<T, E>>` and `ValueTask<Result<T, E>>`.

That is the part worth understanding, because it is what lets you chain without
awaiting each step:

```csharp
// no intermediate awaits, one await at the end
User user = await FetchUserAsync(id)
    .MapAsync(u => EnrichAsync(u))
    .UnwrapOrAsync(Guest);
```

Without the extensions you would await into a local variable at every step:

```csharp
Option<User> fetched = await FetchUserAsync(id);
Option<User> enriched = await fetched.MapAsync(u => EnrichAsync(u));
User user = enriched.UnwrapOr(Guest);
```

{% hint style="info" %}
The async extensions live in the `Waystone.Monads.Options.Extensions` and
`Waystone.Monads.Results.Extensions` namespaces. Add the `using` for the monad you
are working with.
{% endhint %}

## Every async extension returns ValueTask

From 6.0.0 the rule is uniform: **every** async extension on `Option` and
`Result` returns `ValueTask` or `ValueTask<T>`. In 5.x some returned `Task` and
some returned `ValueTask`, and you had to check.

The rule covers extension methods. `Option.TryAsync` and `Result.TryAsync` are
static factories rather than extensions, so they still return `Task<Option<T>>`
and `Task<Result<TOk, TErr>>`. Never call `.AsTask()` on a `TryAsync` — it already
hands you a `Task`.

```csharp
ValueTask<string> output = result.MatchAsync(async x => await DoWorkAsync(x), e => e.ToString());
```

You await a `ValueTask` the same way you await a `Task`, so this rarely changes
your code. Two things to know:

- **Await it once, and only once.** This matters if you store the result before
  awaiting it.
- **Call `.AsTask()` when you need a `Task`** — most often for `Task.WhenAll`.

```csharp
await Task.WhenAll(
    a.MapAsync(FetchAsync).AsTask(),
    b.MapAsync(FetchAsync).AsTask());
```

Both `Task` and `ValueTask` work as receivers, so a chain that mixes them still
composes.

{% hint style="info" %}
`ValueTask` is cheaper when the work finishes synchronously and slightly more
expensive when it does not. A three-link chain saves 144 bytes on a synchronous
`Option` receiver and costs 84 bytes when the head is genuinely pending. See
[v5.x to v6.x](upgrading/v5-to-v6.md#the-measured-trade-off) for the numbers.
{% endhint %}

## Creating a monad from async work

Use `TryAsync` to capture a factory that returns a `Task` and may throw.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<User> maybeUser = await Option.TryAsync(() => FetchUserAsync(id));
```

If `FetchUserAsync` throws, the exception is caught and logged via your
[configured exception logger](configuration.md), and you get back a `None<User>`.

You also get a `None<User>` if the task completes with null, because a `Some`
cannot hold one. Nothing is logged in that case, because nothing threw.
{% endtab %}

{% tab title="Result" %}
```csharp
// supply your own error type
Result<User, string> result = await Result.TryAsync(
    onOk: () => FetchUserAsync(id),
    onErr: ex => ex.Message
);

// or let the error type default to Error
Result<User, Error> builtIn = await Result.TryAsync<User>(() => FetchUserAsync(id));
```

The single type parameter overload converts the exception with
`Error.FromException`, so you do not pass an `onErr` delegate.

`TryAsync` also calls `onErr` when the task completes with null, passing you an
`ArgumentNullException` that names the `asyncFactory` argument.
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
**Never pass an async factory to `Try`.** The overloads that accepted one were
removed in 6.0.0, and the call still compiles — it binds to the synchronous
overload, gives you an `Option<Task<T>>`, and catches nothing.
[`WM1011`](analyzer-rules.md#wm1011) reports every occurrence. See
[Silent change 1](upgrading/v5-to-v6.md#silent-change-1-try-with-an-async-factory).
{% endhint %}

{% hint style="warning" %}
`TryAsync` lets an `OperationCanceledException` through rather than turning it
into a `None` or an `Err`. See [Configuration](configuration.md#cancellation).
{% endhint %}

## Ending a chain

The consuming operations have async counterparts too, so you can finish a chain
without awaiting the monad first.

{% tabs %}
{% tab title="Option" %}
```csharp
User user = await FetchUserAsync(id).UnwrapAsync();
User orGuest = await FetchUserAsync(id).UnwrapOrAsync(Guest);
User orDefault = await FetchUserAsync(id).UnwrapOrDefaultAsync();
User expected = await FetchUserAsync(id).ExpectAsync("the user must exist");
```
{% endtab %}

{% tab title="Result" %}
```csharp
User user = await FetchUserAsync(id).UnwrapAsync();
User orGuest = await FetchUserAsync(id).UnwrapOrAsync(Guest);
User orDefault = await FetchUserAsync(id).UnwrapOrDefaultAsync();
User expected = await FetchUserAsync(id).ExpectAsync("the user must exist");

Error error = await FetchUserAsync(id).UnwrapErrAsync();
Error expectedErr = await FetchUserAsync(id).ExpectErrAsync("the fetch must fail");
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
`UnwrapAsync`, `UnwrapErrAsync`, `ExpectAsync` and `ExpectErrAsync` throw for the
same reasons their synchronous versions do. See
[UnwrapException and UnmetExpectationException](errors-and-exceptions.md#custom-exceptions).
{% endhint %}

## The full surface

Every method below behaves exactly like the synchronous version documented in
[Core Functionality](core-functionality.md), [Option\<T>](option-of-t/README.md)
and [Result\<T, E>](result-of-t-and-e.md). The only difference is that it accepts
an async delegate, a task receiver, or both.

{% hint style="info" %}
Some of these take state, so the delegate does not have to capture. Not all of
them do yet — see
[On the async surface](core-functionality.md#on-the-async-surface).
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
