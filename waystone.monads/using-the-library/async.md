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

## Task and ValueTask

Some async methods return `ValueTask` instead of `Task`. This is deliberate. When
a method can finish without doing asynchronous work, `ValueTask` avoids allocating
a task for the synchronous path.

```csharp
ValueTask<string> output = result.MatchAsync(async x => await DoWorkAsync(x), e => e.ToString());
```

You await a `ValueTask` the same way you await a `Task`, so this rarely changes
your code. It matters if you store the result before awaiting it — await a
`ValueTask` once, and only once.

Both `Task` and `ValueTask` work as receivers, so a chain that mixes them still
composes.

## Creating a monad from async work

Use `TryAsync` to capture a factory that returns a `Task` and may throw.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<User> maybeUser = await Option.TryAsync(() => FetchUserAsync(id));
```

If `FetchUserAsync` throws, the exception is caught and logged via your
[configured exception logger](configuration.md), and you get back a `None<User>`.
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
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
The `Try` overloads that accept an async factory are deprecated and will be
removed in v6.0.0. Use `TryAsync` instead. See [Deprecations](deprecations.md).
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

### Option\<T>

| Category | Methods |
| --- | --- |
| Transform | `MapAsync`, `MapOrAsync`, `MapOrElseAsync`, `FlatMapAsync` |
| State checks | `IsSomeAndAsync`, `IsNoneOrAsync` |
| Consume | `MatchAsync`, `UnwrapAsync`, `UnwrapOrAsync`, `UnwrapOrElseAsync`, `UnwrapOrDefaultAsync`, `ExpectAsync` |
| Side effect | `InspectAsync` |
| Filter and combine | `FilterAsync`, `ZipWithAsync`, `OrElseAsync` |
| Nesting | `FlattenAsync` |
| Conversion | `OkOrAsync`, `OkOrElseAsync` |

### Result\<T, E>

| Category | Methods |
| --- | --- |
| Transform | `MapAsync`, `MapErrAsync`, `MapOrAsync`, `MapOrElseAsync` |
| State checks | `IsOkAndAsync`, `IsErrAndAsync` |
| Consume | `MatchAsync`, `UnwrapAsync`, `UnwrapErrAsync`, `UnwrapOrAsync`, `UnwrapOrElseAsync`, `UnwrapOrDefaultAsync`, `ExpectAsync`, `ExpectErrAsync` |
| Side effect | `InspectAsync`, `InspectErrAsync` |
| Logical operators | `AndThenAsync`, `OrElseAsync` |
| Nesting | `FlattenAsync` |
