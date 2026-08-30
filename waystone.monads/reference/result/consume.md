---
description: Methods that end the chain and hand you a plain value.
---

# Consume

Everything on this page takes you out of the `Result`. To keep chaining, see
[Transform](transform.md).

## IsOk and IsErr

```csharp
bool IsOk { get; }
bool IsErr { get; }
```

The state, as a `bool`. Properties, not methods.

```csharp
Result<DateTime, Error> safeParseResult = SafeParse("2025-01-01");

safeParseResult.IsOk;  // true
safeParseResult.IsErr; // false
```

{% hint style="info" %}
Good for a short-circuit or a guard. Reach for [`Match`](#match) when both
branches matter.
{% endhint %}

## IsOkAnd

```csharp
bool IsOkAnd(Predicate<TOk> predicate)
```

It succeeded **and** the value passes the predicate.

```csharp
safeParseResult.IsOkAnd(dateTime => dateTime > new DateTime(2024, 1, 1)); // true
```

## IsErrAnd

```csharp
bool IsErrAnd(Predicate<TErr> predicate)
```

It failed **and** the error passes the predicate.

```csharp
Result<DateTime, Error> failed = SafeParse("2025");
//                      ^? Err(new Error(ErrorCodes.MalformedDateTime, "not a date"))

failed.IsErrAnd(error => error.Code == ErrorCodes.MalformedDateTime); // true
```

## Match

```csharp
TOut Match<TOut>(Func<TOk, TOut> onOk, Func<TErr, TOut> onErr)
void Match(Action<TOk> onOk, Action<TErr> onErr)
```

Both branches, one plain value out. This is the default way to end a chain.

```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Sam");

int length = nameResult.Match(
    name => name.Length,
    _ => 0);
```

**On an `Err`:** `length` is `0` — the `onErr` branch runs and `onOk` does not.

`Match` also has the state overload that saves the most. See
[Match saves the most](../state-overloads.md#match-saves-the-most).

## Pattern matching with Deconstruct

From 7.0.0 the case types deconstruct, so C# pattern matching binds either side
positionally.

```csharp
using Waystone.Monads.Results;

if (nameResult is Ok<string, string>(var name)) { /* ... */ }
if (nameResult is Err<string, string>(var error)) { /* ... */ }
```

The full list of `Deconstruct` methods, and why a `switch` expression still warns,
is on the [Option page](../option/consume.md#pattern-matching-with-deconstruct).
Both types behave the same way.

Unlike `Option`, **both** of a `Result`'s cases deconstruct, because both carry a
value.

## Unwrap

```csharp
TOk Unwrap()
```

The success value, or a throw.

```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Danny");
string name = nameResult.Unwrap();
```

**On an `Err`:** throws `UnwrapException`.

{% hint style="info" %}
An intentional point of failure, like `First` on an empty sequence. Otherwise
reach for [`Match`](#match).
{% endhint %}

## UnwrapErr

```csharp
TErr UnwrapErr()
```

The other direction. The error, or a throw.

```csharp
Result<int, string> ok = Result.Ok<int, string>(10);
ok.UnwrapErr(); // throws UnwrapException

Result<int, string> err = Result.Err<int, string>("Error");
err.UnwrapErr(); // returns "Error"
```

**On an `Ok`:** throws `UnwrapException`.

## UnwrapOr

```csharp
TOk UnwrapOr(TOk value)
```

The success value, or the fallback you already have.

```csharp
Result<string, Error> nameResult =
    Result.Err<string, Error>(new Error(ErrorCodes.MissingName, "no name was supplied"));

string name = nameResult.UnwrapOr("Unknown");
//     ^? "Unknown"
```

## UnwrapOrElse

```csharp
TOk UnwrapOrElse(Func<TErr, TOk> createElse)
```

The same, but the factory runs only on an `Err` — and it receives the error, so
the fallback can depend on what went wrong.

```csharp
Result<Loadout, Error> getLoadoutResult = GetLoadout("Ashton");
//                     ^? Err<Loadout, Error>

Loadout loadout = getLoadoutResult.UnwrapOrElse(error => GenerateDefaultLoadout());
//      ^? generated loadout
```

## UnwrapOrDefault

```csharp
TOk? UnwrapOrDefault()
```

The success value, or `default(TOk)`.

```csharp
Result<int, string> numberResult = Result.Err<int, string>("Error");
int number = numberResult.UnwrapOrDefault();
//  ^? 0
```

{% hint style="warning" %}
**On a value type this is the catch.** The signature reads `TOk?`, but `TOk` is
constrained `notnull`, so the `?` is an annotation rather than a `Nullable<TOk>`.
`UnwrapOrDefault` on a `Result<int, string>` hands you `0`, and nothing tells you
whether that `0` came from an `Ok` or from the failure. Reach for
[`UnwrapOrNull`](#unwrapornull) there. `WM2015` points this out for you.
{% endhint %}

## UnwrapOrNull

```csharp
TOk? UnwrapOrNull<TOk, TErr>(this Result<TOk, TErr> result) where TOk : struct
```

The success value, or `null` — a real `Nullable<TOk>`, so failure stays visible.
An extension method, in `Waystone.Monads.Results.Extensions`.

```csharp
using Waystone.Monads.Results.Extensions;

Result<int, string> countResult = Result.Err<int, string>("Error");
int? count = countResult.UnwrapOrNull();
//   ^? null
```

{% hint style="info" %}
Constrained to `TOk : struct`. A reference type needs no equivalent —
`UnwrapOrDefault` already gives `null`.
{% endhint %}

## Expect

```csharp
TOk Expect(string message)
```

Like `Unwrap`, but you supply the message the exception carries.

```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Pelor");
string name = nameResult.Expect("Expected a name, but got an error");
```

**On an `Err`:** throws `UnmetExpectationException` carrying your message.

## ExpectErr

```csharp
TErr ExpectErr(string message)
```

The other direction, and the same idea.

```csharp
Result.Ok<int, string>(10).ExpectErr("Must be error");
// throws UnmetExpectationException with message "Must be error"
```

**On an `Ok`:** throws `UnmetExpectationException` carrying your message.

## MapOr

```csharp
TOut MapOr<TOut>(TOut defaultValue, Func<TOk, TOut> map)
```

Transforms the success value, or returns your fallback. Unlike
[`Map`](transform.md#map), it ends the chain.

```csharp
Result<string, string> nameResult = Result.Err<string, string>("Error");

int length = nameResult.MapOr(0, name => name.Length);
//  ^? 0
```

## MapOrElse

```csharp
TOut MapOrElse<TOut>(Func<TErr, TOut> createDefault, Func<TOk, TOut> map)
```

The same, building the fallback from the error.

```csharp
Result<Adventurer, Error> getAdventurerResult = GetAdventurer("Changebringer");

Uri portrait = getAdventurerResult.MapOrElse(
    error => GeneratePortrait(),
    adventurer => adventurer.Portrait);
```

Its state overload threads the same state through *both* delegates — see
[MapOrElse threads state through both delegates](../state-overloads.md#maporelse-threads-state-through-both-delegates).

## MapOrDefault

```csharp
TOut? MapOrDefault<TOut>(Func<TOk, TOut> map)
```

The same, falling back to `default(TOut)`.

```csharp
Result<string, string> nameResult = Result.Err<string, string>("Error");
int length = nameResult.MapOrDefault(name => name.Length);
//  ^? 0
```

{% hint style="warning" %}
The signature reads `TOut?`, but `TOut` is constrained `notnull`, so on a value
type that `?` is an annotation and not a `Nullable<TOut>`. Map to an `int` and the
failure case gives you `0`, not `null`.
{% endhint %}

## MapOrNull

```csharp
TOut? MapOrNull<TOut>(Func<TOk, TOut> map) where TOut : struct
```

The same, falling back to `null`. This one is on `Result<TOk, TErr>` itself, so it
needs no extra `using` — unlike [`UnwrapOrNull`](#unwrapornull), which is an
extension.

```csharp
Result<string, string> nameResult = Result.Err<string, string>("Error");
int? length = nameResult.MapOrNull(name => name.Length);
//   ^? null, where MapOrDefault would have given you 0
```
