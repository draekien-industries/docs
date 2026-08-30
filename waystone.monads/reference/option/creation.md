---
description: The factory methods that build an Option<T>.
---

# Creation

## Option.Some

```csharp
Option<T> Option.Some<T>(T value)
```

Wraps a value. The result is always `Some`.

```csharp
Option<string> some = Option.Some("Hello Bees!");
```

**On `null`:** throws `ArgumentNullException`. `T` is constrained `notnull`, and
the constructor enforces it. A *default* value is fine — `Option.Some(0)` is a
`Some` holding zero.

## Option.None

```csharp
Option<T> Option.None<T>()
```

The empty case. You supply the type parameter because there is no value to infer
it from.

```csharp
Option<string> none = Option.None<string>();
```

## Option.FromNullable

```csharp
Option<T> Option.FromNullable<T>(T? value)
```

`Some` when the value is not `null`, `None` when it is. Use it at the edge of your
code, where the shape is not yours to choose.

## Option.Try

```csharp
Option<T> Option.Try<T>(Func<T> factory)
```

Runs a factory that might throw, and asks one question: did it hand back a value
you can work with?

```csharp
Option<Adventurer> maybeAdventurer = Option.Try(() => GetCurrentAdventurer());
```

**On a throw:** the exception is caught, sent to your
[configured exception logger](../../guides/configuration.md), and you get `None`.

**On `null`:** you get `None`, because a `Some` cannot hold one. Nothing is
logged, because nothing threw. `Option.Try(() => 0)` gives you `Some(0)` — only
`null` is rejected.

{% hint style="warning" %}
**A cancellation is not caught.** `Try` and `TryAsync` let an
`OperationCanceledException` propagate, so a cancelled operation throws rather
than becoming a `None`. Cancelling is you asking the work to stop, not the work
failing. See [Configuration](../../guides/configuration.md#cancellation) to get
the pre-6.0.0 behaviour back.
{% endhint %}

{% hint style="danger" %}
**Do not pass an async factory to `Try`.** It compiles, gives you an
`Option<Task<T>>`, and catches nothing. Use `TryAsync`.
[`WM1011`](../../analyzers/runtime-bugs.md#wm1011) reports every
occurrence.
{% endhint %}

## Option.TryAsync

```csharp
ValueTask<Option<T>> Option.TryAsync<T>(Func<Task<T>> asyncFactory)
```

The same, for a factory that returns a `Task`. See
[Async](../../guides/async.md#create-a-monad-from-async-work).

## Passing state to the factory

`Try` and `TryAsync` each take an optional first argument that they hand to your
factory. Use it to keep the factory from capturing:

```csharp
Option<int> parsed = Option.Try(text, static value => int.Parse(value));
```

See [State overloads](../state-overloads.md) for why this matters.
