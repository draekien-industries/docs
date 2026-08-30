---
description: Every method on Result<TOk, TErr>, grouped by what it does.
icon: binary
---

# Result\<T, E> API

`Result<TOk, TErr>` holds either a success value (`Ok<TOk, TErr>`) or a failure
(`Err<TOk, TErr>`). Neither side can hold `null`.

This is the lookup. If you are learning the type rather than looking a method up,
read the [Result\<T, E> guide](../../guides/result.md) first.

The five categories match the
[Option\<T> reference](../option/README.md) exactly, so a reader who knows one
tree can navigate the other.

## Creation

[Full page →](creation.md)

| Method | What it does |
| --- | --- |
| `Result.Ok<TOk, TErr>(value)` | The success case. Throws on `null`. |
| `Result.Err<TOk, TErr>(error)` | The failure case. Throws on `null`. |
| `Result.Ok<TOk>(value)` | The same, defaulting `TErr` to `Error`. |
| `Result.Err<TOk>(error)` | The same, defaulting `TErr` to `Error`. |
| `Result.Try(factory, onError)` | Runs a factory that might throw. |
| `Result.Try<TOk>(factory)` | The same, converting the exception with `Error.FromException`. |
| `Result.TryAsync(…)` | The same, for a factory returning a `Task`. |

## Transform

[Full page →](transform.md)

| Method | What it does |
| --- | --- |
| `Map` | Changes the success value. |
| `MapErr` | Changes the error. |
| `AndThen` | Chains a step that itself returns a `Result`, without nesting. |
| `And` | The second result, if the first was `Ok`. |
| `Or` | The first `Ok` of the two. |
| `OrElse` | The same, building the second only if needed. |

## Consume

[Full page →](consume.md)

| Member | What it does |
| --- | --- |
| `IsOk` / `IsErr` | The state, as a `bool`. |
| `IsOkAnd` / `IsErrAnd` | The state combined with a predicate. |
| `Match` | Both branches, one plain value out. |
| `Deconstruct` | Lets C# pattern matching bind either side positionally. |
| `Unwrap` | The success value, or an `UnwrapException`. |
| `UnwrapErr` | The error, or an `UnwrapException`. |
| `UnwrapOr` | The success value, or the fallback you supplied. |
| `UnwrapOrElse` | The success value, or one built from the error. |
| `UnwrapOrDefault` | The success value, or `default(TOk)`. |
| `UnwrapOrNull` | The success value, or `null`. Value types only. |
| `Expect` | Like `Unwrap`, with your own exception message. |
| `ExpectErr` | Like `UnwrapErr`, with your own exception message. |
| `MapOr` | Transforms the success value, or returns your fallback. |
| `MapOrElse` | The same, building the fallback from the error. |
| `MapOrDefault` | The same, falling back to `default(TOut)`. |
| `MapOrNull` | The same, falling back to `null`. Value types only. |

## Side effects

[Full page →](side-effects.md)

| Method | What it does |
| --- | --- |
| `Inspect` | Runs an action on the success value and hands the result back. |
| `InspectErr` | The same, on the error. |

## Nesting and conversion

[Full page →](nesting.md)

| Method | What it does |
| --- | --- |
| `Flatten` | Collapses a `Result<Result<TOk, TErr>, TErr>`. |
| `Transpose` | Turns a `Result<Option<T>, E>` into an `Option<Result<T, E>>`. |
| `GetOk` | Converts to an `Option`, keeping the success value. |
| `GetErr` | Converts to an `Option`, keeping the error. |

## Collections

[Full page →](collections.md)

| Method | What it does |
| --- | --- |
| `Flatten` | Keeps the successes, drops the failures. |
| `FlattenErr` | Keeps the failures, drops the successes. |
| `Partition` | Both halves, reading the source once. |
| `Collect` | `Ok` of every value, or `Err` carrying the first failure. |
| `CollectAsync` | The same, over an `IAsyncEnumerable`. |
| `AsEnumerable` | Treats one result as a sequence of nothing or one. |

## Async

Every method above has an `Async` counterpart. They are extension methods on the
task that wraps the result, not on the result itself — see
[Async](../../guides/async.md).

## State overloads

Most methods that take a delegate also take one that receives your data instead of
capturing it. See [State overloads](../state-overloads.md).
