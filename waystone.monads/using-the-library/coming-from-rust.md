# Coming from Rust

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

Waystone.Monads ports Rust's `std::option::Option` and `std::result::Result`. If you already know those types, most of what you know carries over. This page covers the parts that do not.

Read it for three things:

* the name a Rust member has here
* the four behaviours that differ, not just in spelling
* what this library adds that Rust has no counterpart for

## Option

Most members keep their meaning and change only their casing.

| Rust | Waystone | Note |
| --- | --- | --- |
| `is_some` | `IsSome` | A property, not a method |
| `is_none` | `IsNone` | A property, not a method |
| `is_some_and` | `IsSomeAnd` |  |
| `is_none_or` | `IsNoneOr` |  |
| `expect` | `Expect` |  |
| `unwrap` | `Unwrap` |  |
| `unwrap_or` | `UnwrapOr` |  |
| `unwrap_or_else` | `UnwrapOrElse` |  |
| `unwrap_or_default` | `UnwrapOrDefault` | Behaves differently — see [UnwrapOrDefault is riskier here](#unwrapordefault-is-riskier-here) |
| `map` | `Map` |  |
| `map_or` | `MapOr` |  |
| `map_or_else` | `MapOrElse` |  |
| `inspect` | `Inspect` |  |
| `ok_or` | `OkOr` |  |
| `ok_or_else` | `OkOrElse` |  |
| `and` | `And` |  |
| `and_then` | `AndThen` |  |
| `filter` | `Filter` |  |
| `or` | `Or` |  |
| `or_else` | `OrElse` |  |
| `xor` | `Xor` |  |
| `zip` | `Zip` |  |
| `zip_with` | `ZipWith` |  |
| `unzip` | `Unzip` |  |
| `Option::flatten` | `Flatten` | Collapses a nested `Option<Option<T>>` |
| `Iterator::flatten` | `Flatten` | On a sequence of `Option<T>` — drops the `None`s |
| `transpose` | `Transpose` |  |
| `iter` | `AsEnumerable` | Renamed to the .NET convention |
| `collect::<Option<Vec<T>>>()` | `Collect` | On a sequence of `Option<T>` |
| `match` | `Match` | A method, not a language expression |

## Result

| Rust | Waystone | Note |
| --- | --- | --- |
| `is_ok` | `IsOk` | A property, not a method |
| `is_err` | `IsErr` | A property, not a method |
| `is_ok_and` | `IsOkAnd` |  |
| `is_err_and` | `IsErrAnd` |  |
| `ok` | `GetOk` | Renamed, since `Ok` is the case type |
| `err` | `GetErr` | Renamed, since `Err` is the case type |
| `expect` | `Expect` |  |
| `expect_err` | `ExpectErr` |  |
| `unwrap` | `Unwrap` |  |
| `unwrap_err` | `UnwrapErr` |  |
| `unwrap_or` | `UnwrapOr` |  |
| `unwrap_or_else` | `UnwrapOrElse` |  |
| `unwrap_or_default` | `UnwrapOrDefault` | Behaves differently — see [UnwrapOrDefault is riskier here](#unwrapordefault-is-riskier-here) |
| `map` | `Map` |  |
| `map_err` | `MapErr` |  |
| `map_or` | `MapOr` |  |
| `map_or_else` | `MapOrElse` |  |
| `inspect` | `Inspect` |  |
| `inspect_err` | `InspectErr` |  |
| `and` | `And` |  |
| `and_then` | `AndThen` |  |
| `or` | `Or` |  |
| `or_else` | `OrElse` |  |
| `transpose` | `Transpose` |  |
| `iter` | `AsEnumerable` | Renamed to the .NET convention |
| `collect::<Result<Vec<T>, E>>()` | `Collect` | On a sequence of `Result<TOk, TErr>` |
| `match` | `Match` | A method, not a language expression |

`ok()` and `err()` are the renames that catch people out. Searching for `Ok` and `Err` finds the case types instead, so reach for `GetOk` and `GetErr`.

## What behaves differently

Four things differ beyond the name. The first two exist because C# has references and Rust does not.

### There is a fourth state: null

Rust's `Option<T>` is a value type. It cannot be uninitialised.

Here, `Option<T>` and `Result<TOk, TErr>` are reference types. `default(Option<T>)` is null, and calling anything on it throws a `NullReferenceException`. Your Rust instincts will not warn you about this, because the state does not exist there.

The analyzer covers it. `WM1003` reports `default` on either type, and `WM1002` reports null assigned to one. See [analyzer-rules.md](analyzer-rules.md "mention").

### Null is rejected at run time, not compile time

`Option.Some(null)` and `Result.Ok<T, E>(null)` throw `ArgumentNullException` when you construct them. In Rust you cannot write the equivalent at all.

This moves a whole class of mistake from compile time to run time. Nullable reference types narrow the gap but do not close it, because they are annotations rather than guarantees.

### UnwrapOrDefault is riskier here

Rust gates `unwrap_or_default` behind a `T: Default` bound. You opt in by implementing the trait.

In C#, `default(T)` always exists. On a value type an absent value silently becomes `0`, `false`, or `Guid.Empty`, and nothing in the signature warns you. A missing count and a real count of zero look identical afterwards.

Two things help:

* `WM2015` reports `UnwrapOrDefault` and `MapOrDefault` on a value type.
* `UnwrapOrNull` and `MapOrNull` return `null` instead, so the absent case stays distinguishable.

Prefer `UnwrapOrNull` on value types unless you genuinely want the default.

### `match` becomes a method, and it is the only exhaustive option

Use `Match` wherever you would reach for Rust's `match`. It is the only way to consume either type exhaustively.

C#'s exhaustiveness check cannot see that the hierarchy is closed. The `internal` member that stops anything outside the assembly deriving from `Option<T>` is invisible to it, so a `switch` expression covering both cases still reports `CS8509` and asks for a `_` arm you can never reach. `Match` takes exactly two branches, both required, and warns about nothing.

### `if let Some(x)` becomes a positional pattern

From 7.0.0 the case types deconstruct, so the closest thing to Rust's `if let` is:

| Rust | Waystone |
| --- | --- |
| `if let Some(x) = option` | `if (option is Some<T>(var x))` |
| `if let Ok(x) = result` | `if (result is Ok<TOk, TErr>(var x))` |
| `if let Err(e) = result` | `if (result is Err<TOk, TErr>(var e))` |
| `if let None = option` | `if (option is None<T>)` — no parentheses; there is nothing to bind |

Reach for these in statement position, where Rust would use `if let`. Reach for `Match` where Rust would use `match`. See [Pattern matching with Deconstruct](core-functionality.md#pattern-matching-with-deconstruct).

Do not name a case type in a declaration either. A variable, parameter or return typed as `Some<T>` can hold only one of the two states, which defeats the point. `WM2011` reports it and points you at the base type.

## What is not ported

| Rust | Why |
| --- | --- |
| `?` operator | A language feature. Nothing a library can supply. Chain `AndThen` instead. |
| `unwrap_unchecked`, `unwrap_err_unchecked` | C# has no unsafe-contract idiom to hang it on. |
| `contains` | Unstable in Rust too. `IsSomeAnd(x => x == value)` does the same job. |
| `take`, `replace`, `insert`, `get_or_insert`, `as_mut`, `iter_mut` | These mutate in place. Both types here are immutable records. |
| `as_ref`, `as_deref`, `as_slice`, `copied`, `cloned` | Borrow and ownership projections. C# reference semantics make them unnecessary. |

## What Waystone adds

These have no Rust counterpart. Do not go looking for the original.

| Member | What it does |
| --- | --- |
| `Reduce` | Merges two `Option<T>` of the same type |
| `FromNullable` | Builds an `Option<T>` from a `T?` |
| `UnwrapOrNull`, `MapOrNull` | Return `null` rather than `default` on value types |
| `MapOrDefault` | `MapOr` with `default(T)` as the fallback |
| `FirstOrNone`, `LastOrNone`, `FirstOr`, `FirstOrElse` | Predicate searches over a sequence of `Option<T>` |
| `Map`, `Filter` over sequences | Apply a transform or predicate to every element in place |
| `Flatten`, `FlattenErr` on a sequence of `Result` | One side of the sequence, dropping the other. `Result::flatten` is still unstable in Rust. |
| `Partition` | Splits a sequence into successes and failures — close to itertools' `partition_result` |
| `Try`, `TryAsync` | Run a delegate and turn a thrown exception into a `None` or an `Err` |
| The `*Async` surface | Every operation over a `Task` or `ValueTask` receiver |
| The state overloads | Pass a captured value as an argument so the delegate allocates no closure |
| `Deconstruct` on the case types | Positional patterns, so `if let Some(x)` has a C# spelling |
| `MonadOptions` | Global configuration — see [configuration.md](configuration.md "mention") |
| `Select`, `SelectMany`, `Where` | C# query syntax over a monad, in a companion package — see [Waystone.Monads.Linq](../companion-packages/linq.md) |
