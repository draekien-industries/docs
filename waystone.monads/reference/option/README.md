---
description: Every method on Option<T>, grouped by what it does.
icon: option
---

# Option\<T> API

`Option<T>` holds either a value (`Some<T>`) or nothing (`None<T>`). It has no
third state, and it cannot hold `null`.

This is the lookup. If you are learning the type rather than looking a method up,
read the [Option\<T> guide](../../guides/option.md) first.

## Creation

[Full page →](creation.md)

| Method | What it does |
| --- | --- |
| `Option.Some(value)` | Wraps a value. Throws on `null`. |
| `Option.None<T>()` | The empty case. |
| `Option.FromNullable(value)` | `Some` if the value is not `null`, `None` if it is. |
| `Option.Try(factory)` | Runs a factory that might throw. `None` if it throws or returns `null`. |
| `Option.TryAsync(factory)` | The same, for a factory returning a `Task`. |

## Transform

[Full page →](transform.md)

| Method | What it does |
| --- | --- |
| `Map` | Changes the value if there is one. |
| `AndThen` | Chains a step that itself returns an `Option`, without nesting. |
| `Filter` | Keeps the value only if it passes a predicate. |
| `Zip` | Pairs two options into one holding a tuple. |
| `ZipWith` | Pairs two options, combining them yourself. |
| `Unzip` | Splits an option holding a tuple back into two. |
| `And` | The second option, if the first was `Some`. |
| `Or` | The first `Some` of the two. |
| `OrElse` | The same, building the second only if needed. |
| `Xor` | The value only if exactly one of the two is `Some`. |
| `Reduce` | Merges two options of the same type. |

## Consume

[Full page →](consume.md)

| Member | What it does |
| --- | --- |
| `IsSome` / `IsNone` | The state, as a `bool`. |
| `IsSomeAnd` / `IsNoneOr` | The state combined with a predicate. |
| `Match` | Both branches, one plain value out. |
| `Deconstruct` | Lets C# pattern matching bind the value positionally. |
| `Unwrap` | The value, or an `UnwrapException`. |
| `UnwrapOr` | The value, or the fallback you supplied. |
| `UnwrapOrElse` | The value, or one built by a factory. |
| `UnwrapOrDefault` | The value, or `default(T)`. |
| `UnwrapOrNull` | The value, or `null`. Value types only. |
| `Expect` | Like `Unwrap`, with your own exception message. |
| `MapOr` | Transforms the value, or returns your fallback. |
| `MapOrElse` | The same, building the fallback lazily. |
| `MapOrDefault` | The same, falling back to `default(TOut)`. |
| `MapOrNull` | The same, falling back to `null`. Value types only. |

## Side effects

[Full page →](side-effects.md)

| Method | What it does |
| --- | --- |
| `Inspect` | Runs an action on the value and hands the option back unchanged. |

## Nesting and conversion

[Full page →](nesting.md)

| Method | What it does |
| --- | --- |
| `Flatten` | Collapses an `Option<Option<T>>` into an `Option<T>`. |
| `Transpose` | Turns an `Option<Result<T, E>>` into a `Result<Option<T>, E>`. |
| `OkOr` | Converts to a `Result`, with an error you already have. |
| `OkOrElse` | The same, building the error only on a `None`. |

## Collections

[Full page →](collections.md)

| Method | What it does |
| --- | --- |
| `Filter` | Flips every option that fails the predicate to `None`. |
| `Map` | Transforms every `Some` in the sequence. |
| `Flatten` | Drops the `None`s and keeps the values. |
| `Collect` | `Some` of every value, or `None` if any is missing. |
| `CollectAsync` | The same, over an `IAsyncEnumerable`. |
| `AsEnumerable` | Treats one option as a sequence of nothing or one. |
| `FirstOrNone` | The first match, or `None`. |
| `FirstOr` | The first match, or a fallback you supplied. |
| `FirstOrElse` | The same, building the fallback lazily. |

## Async

Every method above has an `Async` counterpart. They are extension methods on the
task that wraps the option, not on the option itself — see
[Async](../../guides/async.md).

## State overloads

Most methods that take a delegate also take one that receives your data instead of
capturing it. See [State overloads](../state-overloads.md).
