# Result\<T, E>

{% hint style="info" %}
**Looking for how to use `Result<T, E>`?** That is the
[Result\<T, E> guide](../../guides/result.md). This page covers the collection
extensions only.
{% endhint %}

## Collections

Working with a `List<Result<TOk, TErr>>` (the results of validating a batch, or of calling something once per item) comes up often enough to have its own methods.

### Flatten

Use `Flatten` to keep the successes and drop the failures. You get an `IEnumerable<TOk>` in the original order.

```csharp
List<Result<int, string>> results = [
    Result.Ok<int, string>(1),
    Result.Err<int, string>("bad"),
    Result.Ok<int, string>(3)
];

IEnumerable<int> values = results.Flatten();
//               ^? [1, 3]
```

### FlattenErr

Use `FlattenErr` for the other half. It keeps the failures and drops the successes.

```csharp
IEnumerable<string> errors = results.FlattenErr();
//                  ^? ["bad"]
```

{% hint style="info" %}
`Flatten` and `FlattenErr` are both lazy and each walks the source once. Call both on the same sequence and you enumerate it twice, which matters when the source is a database query or anything else you would rather not run again. Reach for `Partition` there.
{% endhint %}

### Partition

Use `Partition` when you want both halves and you want the source read once. It returns a tuple of two lists.

```csharp
(IReadOnlyList<int> oks, IReadOnlyList<string> errs) = results.Partition();
//                  ^? [1, 3]              ^? ["bad"]
```

Unlike `Flatten` and `FlattenErr`, `Partition` is eager. It enumerates the source immediately and hands back two materialised lists.

```csharp
var (succeeded, failed) = items.Select(Validate).Partition();

if (failed.Count > 0)
{
    return Result.Err<Report, IReadOnlyList<string>>(failed);
}
```

### Collect

Use `Collect` when the batch has to succeed as a whole. You get an `Ok` holding every value, or an `Err` carrying the first failure.

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

`Collect` stops at the first `Err`. It never looks at the rest of the source, so `"worse"` above is never seen and anything that would have produced the later elements does not run.

Choose between `Collect` and `Partition` by what you owe the caller:

| You need | Use |
| --- | --- |
| All the values, or one failure | `Collect` |
| Every failure, to report them together | `Partition` |
| The successes, ignoring failures | `Flatten` |

{% hint style="info" %}
`Collect` is eager, and it builds a list as it goes. Do not call it on an unbounded sequence.
{% endhint %}

An empty sequence gives you `Ok` of an empty list. There is nothing in it to fail.

The values that came before the failure are discarded. If you need them, use `Partition`.

### CollectAsync

Use `CollectAsync` for the same job over an `IAsyncEnumerable<Result<TOk, TErr>>`.

```csharp
Result<IReadOnlyList<int>, string> all = await stream.CollectAsync(cancellationToken);
```

It stops pulling from the stream at the first `Err`, so the work behind the later elements never happens. That is the reason to use it instead of reading the whole stream into a list and calling `Collect`.

### AsEnumerable

Use `AsEnumerable` on a single `Result` to treat it as a sequence of nothing or one, which is what lets the methods above compose out of LINQ. It discards the error on the way — an `Err` becomes an empty sequence.

To write a query that stays a `Result` and keeps the error, see [Waystone.Monads.Linq](../companion-packages/linq.md).

```csharp
Result<int, string> result = Result.Ok<int, string>(1);

IEnumerable<int> sequence = result.AsEnumerable();
//               ^? [1], and [] for an Err
```

`Option<T>` has the same method, and `Flatten` on a sequence of either is built out of it.
