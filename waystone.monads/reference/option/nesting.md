---
description: Remove a level of nesting, or convert to a Result.
icon: code-branch
---

# Nesting and conversion

`Flatten` and `Transpose` are extension methods, in
`Waystone.Monads.Options.Extensions` — add the `using`. `OkOr` and `OkOrElse` are
on `Option<T>` itself and need nothing.

## Flatten

```csharp
Option<T> Flatten<T>(this Option<Option<T>> option)
```

Removes one level of nesting.

```csharp
Option<Option<string>> some = Option.Some(Option.Some("Chetney"));
Option<string> result = some.Flatten();
```

**On a `None` at either level:** you get `None`.

{% hint style="info" %}
This is the single-option `Flatten`. The one that drops the `None`s out of a
*sequence* is a different method — see [Collections](collections.md#flatten). No
receiver is both, so the two never compete.
{% endhint %}

## Transpose

```csharp
Result<Option<T>, E> Transpose<T, E>(this Option<Result<T, E>> option)
```

Turns an option holding a result into a result holding an option.

```csharp
Option<int> maybeNumber = Option.Try(() => RollD20());

Option<Result<int, string>> maybeResult = maybeNumber
    .Map(number => Divide(number, 2));

Result<Option<int>, string> result = maybeResult.Transpose();
```

Calling `Transpose` here declares that an `Ok` holding `None` is a valid outcome
in your business rules.

**On a `None`:** you get `Ok(None)` — nothing failed, there was just nothing
there.

`Result<Option<T>, E>` transposes the other way. See
[the Result page](../result/nesting.md#transpose).

## OkOr

```csharp
Result<T, TErr> OkOr<TErr>(TErr error)
```

Converts to a `Result`. A `Some` becomes an `Ok`, a `None` becomes an `Err`
carrying the error you supplied.

```csharp
Option<int> some = Option.Some(1);
Option<int> none = Option.None<int>();
Error error = new("ER1", "Missing number.");

Result<int, Error> ok = some.OkOr(error);
//                 ^? Ok(1)

Result<int, Error> err = none.OkOr(error);
//                 ^? Err(error)
```

**Evaluated eagerly.** If the error comes from a function call, use
[`OkOrElse`](#okorelse).

## OkOrElse

```csharp
Result<T, TErr> OkOrElse<TErr>(Func<TErr> errorFactory)
```

The same, but the error is built only when the option is `None`.

```csharp
Result<int, string> ok = some.OkOrElse(() => "Missing number");
//                  ^? Ok(1)

Result<int, string> err = none.OkOrElse(() => "Missing number");
//                  ^? Err("Missing number")
```

## Going the other way

To convert a `Result` into an `Option`, see
[`GetOk` and `GetErr`](../result/nesting.md#getok).
