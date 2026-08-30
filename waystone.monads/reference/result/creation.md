---
description: The factory methods that build a Result<TOk, TErr>.
---

# Creation

## Result.Ok and Result.Err

```csharp
Result<TOk, TErr> Result.Ok<TOk, TErr>(TOk value)
Result<TOk, TErr> Result.Err<TOk, TErr>(TErr error)
```

Supply both type parameters when you use your own error type.

```csharp
Result<int, string> ok = Result.Ok<int, string>(1);
Result<int, string> err = Result.Err<int, string>("Something went wrong...");
```

{% hint style="warning" %}
**Neither an `Ok` nor an `Err` can hold `null`.** Pass one and you get an
`ArgumentNullException`. Every factory funnels through the same guard.

```csharp
Result.Ok<string, Error>(null!); // throws
```

New in 5.5.0. Before that, an `Ok` could hold `null` and the `null` surfaced later
as a `NullReferenceException` in your own code. `TOk` and `TErr` are constrained
`notnull`, so the compiler already warned you; now the runtime agrees.

A **default** value is fine and always has been. `Result.Ok<int, string>(0)` is an
`Ok` holding `0`. Only `null` is rejected.
{% endhint %}

## The single-type-parameter overloads

```csharp
Result<TOk, Error> Result.Ok<TOk>(TOk value)
Result<TOk, Error> Result.Err<TOk>(Error error)
```

If you are happy with the built-in [`Error`](../../guides/errors.md) type, leave
`TErr` off and it defaults to `Error`.

```csharp
Result<int, Error> ok = Result.Ok<int>(1);
Result<int, Error> err = Result.Err<int>(
    new Error("MyCode", "Something went wrong..."));
```

### From a generated catalog

Mark an enum with `[ErrorCodeCatalog]` and the source generator gives you a
factory per member, with the message required.

```csharp
[ErrorCodeCatalog]
enum PartyErrors
{
    NotFound
}

Result<Adventurer, Error> err = Result.Err<Adventurer>(
    PartyErrorsCatalog.Errors.NotFound("The adventurer was not found"));
```

{% hint style="info" %}
Passing an enum straight to `Result.Err` was removed in 7.0.0. See
[Generated error codes](../../using-the-library/generated-error-codes.md).
{% endhint %}

## Result.Try

```csharp
Result<TOk, TErr> Result.Try<TOk, TErr>(Func<TOk> factory, Func<Exception, TErr> onError)
```

Runs a factory that might throw, and asks one question: did it hand back a value
you can work with?

```csharp
Result<Adventurer, string> result = Result.Try(
    factory: () => GetCurrentAdventurer(),
    onError: ex => ex.Message);
```

**On a throw:** the exception is caught, sent to your
[configured exception logger](../../guides/configuration.md), and `onError` runs.

**On `null`:** `onError` runs too, passed an `ArgumentNullException` naming the
`factory` argument. Nothing is logged, because nothing threw.

That last case is the one place a `null` does not throw. `Try` exists so you can
hand over a delegate and learn whether a workable value came back without wrapping
the call yourself — so it turns the `null` into an `Err` for you.

{% hint style="warning" %}
**A cancellation is not caught.** `Try` and `TryAsync` let an
`OperationCanceledException` propagate. See
[Configuration](../../guides/configuration.md#cancellation).
{% endhint %}

{% hint style="danger" %}
**Do not pass an async factory to `Try`.** It compiles, gives you a
`Result<Task<T>, E>`, and catches nothing. Use `TryAsync`.
[`WM1011`](../../using-the-library/analyzer-rules.md#wm1011) reports every
occurrence.
{% endhint %}

### Defaulting to Error

```csharp
Result<TOk, Error> Result.Try<TOk>(Func<TOk> factory)
```

Converts the exception with
[`Error.FromException`](../../guides/exceptions.md#turning-an-exception-into-an-error),
so you pass no `onError` delegate.

```csharp
Result<Adventurer, Error> result = Result.Try<Adventurer>(
    () => GetCurrentAdventurer());
```

## Result.TryAsync

```csharp
ValueTask<Result<TOk, TErr>> Result.TryAsync<TOk, TErr>(
    Func<Task<TOk>> asyncFactory,
    Func<Exception, TErr> onError)
```

The same, for a factory that returns a `Task`. See
[Async](../../guides/async.md#create-a-monad-from-async-work).

## Passing state to the factory

`Try` and `TryAsync` each take an optional first argument that they hand to your
factory. Use it to keep the factory from capturing:

```csharp
Result<int, Error> result = Result.Try(text, static value => int.Parse(value));
```

See [State overloads](../state-overloads.md) for why this matters.
