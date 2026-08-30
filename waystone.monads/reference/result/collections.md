---
description: Methods for working with a sequence of Result<TOk, TErr>.
---

# Collections

A `List<Result<TOk, TErr>>` — the results of validating a batch, or of calling
something once per item — comes up often enough to have its own methods.

Every method here is an extension method, in
`Waystone.Monads.Results.Extensions`. Add the `using`.

## Flatten

```csharp
IEnumerable<TOk> Flatten<TOk, TErr>(this IEnumerable<Result<TOk, TErr>> source)
```

Keeps the successes and drops the failures, in the original order.

```csharp
List<Result<int, string>> results = [
    Result.Ok<int, string>(1),
    Result.Err<int, string>("bad"),
    Result.Ok<int, string>(3)
];

IEnumerable<int> values = results.Flatten();
//               ^? [1, 3]
```

## FlattenErr

```csharp
IEnumerable<TErr> FlattenErr<TOk, TErr>(this IEnumerable<Result<TOk, TErr>> source)
```

The other half. Keeps the failures and drops the successes.

```csharp
IEnumerable<string> errors = results.FlattenErr();
//                  ^? ["bad"]
```

{% hint style="info" %}
Both are lazy and each walks the source once. Call both on the same sequence and
you enumerate it twice, which matters when the source is a database query or
anything else you would rather not run again. Reach for
[`Partition`](#partition) there.
{% endhint %}

## Partition

```csharp
(IReadOnlyList<TOk>, IReadOnlyList<TErr>) Partition<TOk, TErr>(
    this IEnumerable<Result<TOk, TErr>> source)
```

Both halves, reading the source once.

```csharp
(IReadOnlyList<int> oks, IReadOnlyList<string> errs) = results.Partition();
//                  ^? [1, 3]              ^? ["bad"]
```

**Eager.** It enumerates the source immediately and hands back two materialised
lists.

```csharp
var (succeeded, failed) = items.Select(Validate).Partition();

if (failed.Count > 0)
{
    return Result.Err<Report, IReadOnlyList<string>>(failed);
}
```

This is the method to use when you owe the caller *every* failure, not just the
first.

## Collect

```csharp
Result<IReadOnlyList<TOk>, TErr> Collect<TOk, TErr>(
    this IEnumerable<Result<TOk, TErr>> source)
```

For when the batch has to succeed as a whole. You get an `Ok` holding every value,
or an `Err` carrying the **first** failure.

```csharp
List<Result<int, string>> results = [
    Result.Ok<int, string>(1),
    Result.Ok<int, string>(3)
];

Result<IReadOnlyList<int>, string> all = results.Collect();
//                                 ^? Ok([1, 3])
```

One failure fails the whole call:

```csharp
List<Result<int, string>> withAFailure = [
    Result.Ok<int, string>(1),
    Result.Err<int, string>("bad"),
    Result.Err<int, string>("worse")
];

Result<IReadOnlyList<int>, string> all = withAFailure.Collect();
//                                 ^? Err("bad")
```

**It stops at the first `Err`.** `"worse"` above is never seen, and anything that
would have produced the later elements does not run.

**On an empty sequence:** you get `Ok` of an empty list. There is nothing in it to
fail.

**The values before the failure are discarded.** If you need them, use
[`Partition`](#partition).

Choose between the three by what you owe the caller:

| You need | Use |
| --- | --- |
| All the values, or one failure | `Collect` |
| Every failure, to report them together | `Partition` |
| The successes, ignoring failures | `Flatten` |

{% hint style="info" %}
`Collect` is eager, and builds a list as it goes. Do not call it on an unbounded
sequence.
{% endhint %}

## CollectAsync

```csharp
ValueTask<Result<IReadOnlyList<TOk>, TErr>> CollectAsync<TOk, TErr>(
    this IAsyncEnumerable<Result<TOk, TErr>> source,
    CancellationToken cancellationToken = default)
```

The same job over an `IAsyncEnumerable`.

```csharp
Result<IReadOnlyList<int>, string> all = await stream.CollectAsync(cancellationToken);
```

It stops pulling from the stream at the first `Err`, so the work behind the later
elements never happens. That is the reason to use it rather than reading the whole
stream into a list and calling `Collect`.

Returned `Task` up to 6.7.0. Returns `ValueTask` from 7.0.0.

## AsEnumerable

```csharp
IEnumerable<TOk> AsEnumerable<TOk, TErr>(this Result<TOk, TErr> result)
```

Treats a single result as a sequence of nothing or one, which is what lets the
methods above compose out of LINQ. **It discards the error** — an `Err` becomes an
empty sequence.

```csharp
Result<int, string> result = Result.Ok<int, string>(1);

IEnumerable<int> sequence = result.AsEnumerable();
//               ^? [1], and [] for an Err
```

To write a query that stays a `Result` and keeps the error, see
[Waystone.Monads.Linq](../../companion-packages/linq.md).

`Option<T>` has the same method, and `Flatten` on a sequence of either is built
out of it.
