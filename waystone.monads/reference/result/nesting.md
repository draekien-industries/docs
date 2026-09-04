---
description: Remove a level of nesting, or convert to an Option.
icon: code-branch
---

# Nesting and conversion

`Flatten` and `Transpose` are extension methods, in
`Waystone.Monads.Results.Extensions` — add the `using`. `GetOk` and `GetErr` are
on `Result<TOk, TErr>` itself and need nothing.

## Flatten

```csharp
Result<TOk, TErr> Flatten<TOk, TErr>(this Result<Result<TOk, TErr>, TErr> result)
```

Removes one level of nesting. You get here by calling `Map` with a function that
itself returns a `Result`.

```csharp
Result<string, string> start = Result.Ok<string, string>("Storm Weaver");
Result<Result<int, string>, string> output = start.Map(x => CountRunes(x));
Result<int, string> flattened = output.Flatten();
```

**Prefer [`AndThen`](transform.md#andthen)**, which does both steps at once. This
exists for code that grew one method at a time.

{% hint style="info" %}
This is the single-result `Flatten`. The one that drops the failures out of a
*sequence* is a different method — see [Collections](collections.md#flatten).
{% endhint %}

## Transpose

```csharp
Option<Result<TOk, TErr>> Transpose<TOk, TErr>(this Result<Option<TOk>, TErr> result)
```

Turns a result holding an option into an option holding a result.

```csharp
Result<Option<decimal>, string> calculationResult =
    CreateCalculator(Realm.TalDorei)
        .Map(calculator => calculator.GetToll(100.00m));

Option<Result<decimal, string>> maybeToll = calculationResult.Transpose();
```

Calling `Transpose` here declares that the absence of a toll is a valid outcome in
your business rules.

**On an `Err`:** you get `Some(Err(…))` — the failure survives.

`Option<Result<T, E>>` transposes the other way. See
[the Option page](../option/nesting.md#transpose).

## GetOk

```csharp
Option<TOk> GetOk()
```

Converts to an `Option`, keeping the success value and discarding the error.

```csharp
Result<int, string> ok = Result.Ok<int, string>(1);
Result<int, string> err = Result.Err<int, string>("Error");

Option<int> some = ok.GetOk();
//          ^? Some(1)

Option<int> none = err.GetOk();
//          ^? None()
```

**The error is gone.** Use this only when you have already dealt with it, or do
not care.

## GetErr

```csharp
Option<TErr> GetErr()
```

The other direction. Keeps the error and discards the success value.

```csharp
Option<string> none = ok.GetErr();
//             ^? None()

Option<string> some = err.GetErr();
//             ^? Some("Error")
```

## Going the other way

To convert an `Option` into a `Result`, see
[`OkOr` and `OkOrElse`](../option/nesting.md#okor).
