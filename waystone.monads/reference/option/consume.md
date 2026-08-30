---
description: Methods that end the chain and hand you a plain value.
---

# Consume

Everything on this page takes you out of the `Option`. To keep chaining, see
[Transform](transform.md).

## IsSome and IsNone

```csharp
bool IsSome { get; }
bool IsNone { get; }
```

The state, as a `bool`. Properties, not methods.

```csharp
Option<string> maybeName = Option.Some("Laudna");

maybeName.IsSome; // true
maybeName.IsNone; // false
```

{% hint style="info" %}
Good for a short-circuit or a guard. Reach for [`Match`](#match) when both
branches matter.
{% endhint %}

## IsSomeAnd

```csharp
bool IsSomeAnd(Predicate<T> predicate)
```

There is a value **and** it passes the predicate.

```csharp
Option<string> maybePatron = Option.Some("The Raven Queen");
maybePatron.IsSomeAnd(patron => patron.Length > 0); // true
```

## IsNoneOr

```csharp
bool IsNoneOr(Predicate<T> predicate)
```

There is no value, **or** the one there passes.

```csharp
maybePatron.IsNoneOr(patron => patron.Length > 0);                 // true
maybePatron.IsNoneOr(patron => string.IsNullOrWhiteSpace(patron)); // false
```

## Match

```csharp
TOut Match<TOut>(Func<T, TOut> onSome, Func<TOut> onNone)
void Match(Action<T> onSome, Action onNone)
```

Both branches, one plain value out. This is the default way to end a chain.

```csharp
Option<string> maybeName = Option.Some("Travis");

int length = maybeName.Match(
    name => name.Length,
    () => 0);
```

**On a `None`:** `length` is `0` — the `onNone` branch runs and `onSome` does not.

`Match` also has the state overload that saves the most, because a capturing call
pays for two delegates. See
[Match saves the most](../state-overloads.md#match-saves-the-most).

## Pattern matching with Deconstruct

From 7.0.0 the case types deconstruct, so C# pattern matching binds the value
positionally.

```csharp
using Waystone.Monads.Options;

if (maybeName is Some<string>(var name))
{
    Console.WriteLine(name.Length);
}
```

Three `Deconstruct` methods exist across both types, and only three:

| Type | Signature | Binds |
| --- | --- | --- |
| `Some<T>` | `Deconstruct(out T value)` | The contained value |
| `Ok<TOk, TErr>` | `Deconstruct(out TOk value)` | The Ok value |
| `Err<TOk, TErr>` | `Deconstruct(out TErr error)` | The error |

Each is documented as never handing you `null`.

### None has none, deliberately

There is nothing to bind, and `option is None<string>` already tests the case.

So `option is None<string>()`, **with** the parentheses, is a compile error rather
than a redundant spelling. An empty positional pattern still needs a `Deconstruct`
to bind against:

```
CS8129: No suitable 'Deconstruct' instance or extension method was found for type
        'None<string>', with 0 out parameters and a void return type.
```

Write it without the parentheses.

### You cannot deconstruct the monad itself

`Deconstruct` is on the case types, not on `Option<T>`. So this does not compile:

```csharp
var (a, b) = maybeName; // CS1061, CS8129, CS8130
```

There is no state-plus-value tuple to destructure. Test the case first, then bind.

### Where Match still wins

**A `switch` expression over the closed hierarchy warns**, even with both cases
covered:

```csharp
int length = maybeName switch
{
    Some<string>(var name) => name.Length,
    None<string> => 0,
};
```

```
CS8509: The switch expression does not handle all possible values of its input type
        (it is not exhaustive). For example, the pattern '_' is not covered.
```

The hierarchy really is closed — an internal member stops anything outside the
assembly deriving from `Option<T>` — but the compiler's exhaustiveness check has
no knowledge of that idiom. Silencing the warning means an unreachable arm:

```csharp
_ => throw new UnreachableException(),
```

`Match` needs neither. It takes exactly two branches, both required, and returns a
value with no warning to suppress.

**So use `Match` when you want a value out of both cases**, which is most of the
time. **Reach for a positional pattern in statement position** — an `if` guarding
a block, a `switch` statement, a `when` clause — where `Match` would mean wrapping
statements in a lambda that returns nothing.

{% hint style="info" %}
**A positional pattern does not trip `WM2021`.** That rule reports a *property*
pattern reading `IsSome`, `IsNone`, `IsOk` or `IsErr` — a state check written so
nothing recognises it as one. A positional pattern reads none of those
properties. See [`WM2021`](../../analyzers/idioms.md#wm2021).
{% endhint %}

## Unwrap

```csharp
T Unwrap()
```

The value, or a throw.

```csharp
Option<string> maybeName = Option.Some("Lorekeeper");
string name = maybeName.Unwrap();
```

**On a `None`:** throws `UnwrapException`.

{% hint style="info" %}
An intentional point of failure, like `First` on an empty sequence. Use it only
when you have established the value is there upstream. Otherwise reach for
[`Match`](#match).
{% endhint %}

## UnwrapOr

```csharp
T UnwrapOr(T value)
```

The value, or the fallback you already have.

```csharp
Option<string> maybeNickname = Option.None<string>();
string nickname = maybeNickname.UnwrapOr("Lautna");
//     ^? "Lautna"
```

## UnwrapOrElse

```csharp
T UnwrapOrElse(Func<T> createElse)
```

The same, but the factory runs only on a `None`. Use it when the fallback costs
something to build.

```csharp
Option<Uri> maybePortrait = Option.None<Uri>();
Uri portrait = maybePortrait.UnwrapOrElse(() => GeneratePortrait());
//  ^? generated portrait
```

## UnwrapOrDefault

```csharp
T? UnwrapOrDefault()
```

The value, or `default(T)`.

```csharp
Option<string> maybeName = Option.None<string>();
string? name = maybeName.UnwrapOrDefault();
//      ^? null
```

{% hint style="warning" %}
**On a value type this is the catch.** The signature reads `T?`, but `T` is
constrained `notnull`, so the `?` is an annotation rather than a `Nullable<T>`.
`UnwrapOrDefault` on an `Option<int>` hands you `0`, and nothing tells you whether
that `0` came from a `Some` or from the absent case. Reach for
[`UnwrapOrNull`](#unwrapornull) there. `WM2015` points this out for you.
{% endhint %}

## UnwrapOrNull

```csharp
T? UnwrapOrNull<T>(this Option<T> option) where T : struct
```

The value, or `null` — a real `Nullable<T>`, so absence stays visible. An
extension method, in `Waystone.Monads.Options.Extensions`.

```csharp
using Waystone.Monads.Options.Extensions;

Option<int> maybeCount = Option.None<int>();
int? count = maybeCount.UnwrapOrNull();
//   ^? null, where UnwrapOrDefault would have given you 0
```

{% hint style="info" %}
Constrained to `T : struct`, so it does not appear on an `Option<string>`. A
reference type needs no equivalent — `UnwrapOrDefault` already gives `null`.
{% endhint %}

## Expect

```csharp
T Expect(string message)
```

Like `Unwrap`, but you supply the message the exception carries.

```csharp
Option<string> maybeName = Option.Some("Greymore");
string name = maybeName.Expect("Expected a name, but got nothing.");
```

**On a `None`:** throws `UnmetExpectationException` carrying your message.

Use it where an absent value means a logic error rather than a runtime condition
to recover from.

## MapOr

```csharp
TOut MapOr<TOut>(TOut defaultValue, Func<T, TOut> map)
```

Transforms the value, or returns your fallback. Unlike [`Map`](transform.md#map),
it ends the chain.

```csharp
Option<string> maybeName = Option.None<string>();
int length = maybeName.MapOr(0, name => name.Length);
//  ^? 0
```

## MapOrElse

```csharp
TOut MapOrElse<TOut>(Func<TOut> createDefault, Func<T, TOut> map)
```

The same, building the fallback lazily.

```csharp
Option<Adventurer> maybeAdventurer = Option.None<Adventurer>();

Uri portrait = maybeAdventurer.MapOrElse(
    () => GeneratePortrait(),
    adventurer => adventurer.Portrait);
```

Its state overload threads the same state through *both* delegates — see
[MapOrElse threads state through both delegates](../state-overloads.md#maporelse-threads-state-through-both-delegates).

## MapOrDefault

```csharp
TOut? MapOrDefault<TOut>(Func<T, TOut> map)
```

The same, falling back to `default(TOut)`, so you write no fallback at all.

```csharp
Option<string> maybeName = Option.None<string>();
int length = maybeName.MapOrDefault(name => name.Length);
//  ^? 0
```

{% hint style="warning" %}
The signature reads `TOut?`, but `TOut` is constrained `notnull`, so on a value
type that `?` is an annotation and not a `Nullable<TOut>`. Map to an `int` and the
absent case gives you `0`, not `null`. That is what `MapOrNull` is for.
{% endhint %}

## MapOrNull

```csharp
TOut? MapOrNull<TOut>(Func<T, TOut> map) where TOut : struct
```

The same, falling back to `null`. This one is on `Option<T>` itself, so it needs
no extra `using` — unlike [`UnwrapOrNull`](#unwrapornull), which is an extension.

```csharp
Option<string> maybeName = Option.None<string>();
int? length = maybeName.MapOrNull(name => name.Length);
//   ^? null, where MapOrDefault would have given you 0
```

{% hint style="info" %}
Constrains its result to `TOut : struct`. Map to a reference type and
`MapOrDefault` already gives you `null`.
{% endhint %}
