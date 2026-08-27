# Result\<T, E>

## What a Result can hold

Neither side of a `Result` can hold null. `Result.Ok<string, Error>(null!)` and
`Result<string, int> x = null!;` both throw `ArgumentNullException`, because
`TOk` and `TErr` are constrained `notnull` and the constructors now enforce it.

A default value is fine. `Result.Ok<int, string>(0)` is an `Ok` holding `0`, and
`Result.Ok<Guid, string>(Guid.Empty)` is an `Ok` holding `Guid.Empty`. This is
where `Result` differs from `Option` in 5.x, where a `Some` rejects the default of
its type as well — see [v5.x to v6.x](../upgrading-and-deprecations/v5-to-v6.md), which brings the two
into line.

{% hint style="info" %}
The one exception is [`Result.Try`](core-functionality.md), which returns an `Err`
for a null instead of throwing. That is the point of it.
{% endhint %}

The null guard is new in 5.5.0. Before it, an `Ok` could hold null and the null
surfaced later, as a `NullReferenceException` somewhere in your own code.

## Printing and logging

**`ToString()` never shows the wrapped value.** You get the state and nothing else:

```csharp
Result.Ok<int, Error>(1).ToString()  // "Ok { IsOk = True, IsErr = False }"
Result.Err<int, Error>(e).ToString() // "Err { IsOk = False, IsErr = True }"
```

`Result<TOk, TErr>` is a record, and `Ok` and `Err` both keep their value in a
private property, so the compiler-generated `ToString()` has nothing to print.
Interpolating a result into a log message tells you which branch you are on, never
what it holds.

Log each branch with the inspect pair. Both run only on their own branch, and both
return the result unchanged so you can keep chaining:

```csharp
Result<Order, Error> order = LoadOrder(id)
    .Inspect(o => logger.LogInformation("Loaded order {Id}", o.Id))
    .InspectErr(e => logger.LogWarning("Load failed: {Code} {Message}", e.Code, e.Message));
```

Use [#inspect](core-functionality.md#inspect "mention") for the `Ok` branch and
[#inspecterr](result-of-t-and-e.md#inspecterr "mention") for the `Err` branch.

## Control Flow

### IsOkAnd

Use `IsOkAnd` when you need to check if the `Result` is an `Ok` and the value inside the `Ok` matches a predicate.

```csharp
Result<DateTime, Error> safeParseResult = SafeParse("2025-01-01");

safeParseResult.IsOkAnd(dateTime => dateTime > new DateTime(2024, 1, 1)); // true
```

### IsErrAnd

Use `IsErrAnd` when you need to check if the `Result` is an `Err` and the value inside the `Err` matches a predicate.

```csharp
Result<DateTime, Error> safeParseResult = SafeParse("2025");
//                      ^? Err<DateTime, Error>(
//                             new Error(ErrorCodes.MalformedDateTime, "not a date"))

safeParseResult.IsErrAnd(error => error.Code == ErrorCodes.MalformedDateTime); // true
```

## Transform

{% hint style="info" %}
Refer to the [#transform](core-functionality.md#transform "mention")section on the [core-functionality.md](core-functionality.md "mention") page to learn about the other transforms available for a `Result<T, E>`
{% endhint %}

### MapErr

The counterpart to [#map](core-functionality.md#map "mention"), use `MapErr` when you need to transform the contained value if it is an `Err`. It is useful when you need to transform the `Err` type in order to continue chaining monadic operations.

```csharp
Result<string, string> GenerateName();
Result<int, Error> GetLength(string value);

Result<int, Error> lengthResult = GenerateName()          // Result<string, string>
    .MapErr(message => new Error("name.failed", message)) // Result<string, Error>
    .Map(name => GetLength(name))                         // Result<Result<int, Error>, Error>
    .Flatten();                                           // Result<int, Error>
    
```

{% hint style="info" %}
Some [#logical-operators](result-of-t-and-e.md#logical-operators "mention") can actually be used as transforms. The last 2 lines of the above example can be rewritten with [#andthen](result-of-t-and-e.md#andthen "mention").

```csharp
Result<int, Error> lengthResult = GenerateName()          // Result<string, string>
    .MapErr(message => new Error("name.failed", message)) // Result<string, Error>
    .AndThen(name => GetLength(name))                     // Result<int, Error>
```
{% endhint %}

## Consume

{% hint style="info" %}
Refer to the [#consume](core-functionality.md#consume "mention") section on the [core-functionality.md](core-functionality.md "mention") page to learn about the other consume methods available for a `Result<T, E>`
{% endhint %}

### ExpectErr

The counterpart to [#inspect](core-functionality.md#inspect "mention"), use `ExpectErr` when you want to consume the monadic wrapper and fail loudly when the `Result` is an `Ok`. It allows you to provide a meaningful exception message to explain why the error is expected.

```csharp
Result<int, string> result = Result.Ok<int, string>(10);
result.ExpectErr("Must be error"); // throws UnmetExpectationException with message "Must be error"
```

{% hint style="warning" %}
An `UnmetExpectationException` with your provided message will be thrown when the value is a success.
{% endhint %}

Use `ExpectErrAsync` to do the same on a `Task<Result<T, E>>` without awaiting it first. See [async.md](async.md "mention").

### UnwrapErr

The counterpart to [#unwrap](core-functionality.md#unwrap "mention"), use `UnwrapErr` when you are certain the `Result` is an `Err` and you want to fail loudly if it is an `Ok`.

{% hint style="info" %}
Avoid `UnwrapErr` unless you've validated the presence of an `Err` upstream. It's an intentional point of failure, like `First` on an empty sequence. In most cases, you should reach for [#match](result-of-t-and-e.md#match "mention") or [#map](result-of-t-and-e.md#map "mention").
{% endhint %}

```csharp
Result<int, string> ok = Result.Ok<int, string>(10);
ok.UnwrapErr(); // throws UnwrapException

Result<int, string> err = Result.Err<int, string>("Error");
err.UnwrapErr(); // returns "Error"
```

{% hint style="warning" %}
An `UnwrapException` will be thrown if the `Result` is an `Ok`.
{% endhint %}

Use `UnwrapErrAsync` to do the same on a `Task<Result<T, E>>` without awaiting it first. See [async.md](async.md "mention").

## Side-Effect

{% hint style="info" %}
Refer to the [#side-effect](core-functionality.md#side-effect "mention")section on the [core-functionality.md](core-functionality.md "mention") page to learn about the other side-effects available for a `Result<T, E>`
{% endhint %}

### InspectErr

The counterpart to [#inspect](core-functionality.md#inspect "mention"), use `InspectErr` when you want to run a side effect against an `Err` without modifying the underlying value. The most common use case for `InspectErr` is to log an error in the event of a failure.

{% hint style="info" %}
Reach for [#maperr](result-of-t-and-e.md#maperr "mention")if you want to transform the value of an `Err`
{% endhint %}

```csharp
Result<string, string> result = GetUser("John")
    .InspectErr(err => Console.WriteLine($"Get user failed: {err.Message}"))
    .Map(user => user.Username)
    .MapErr(err => err.Message);
```

## Logical Operators

### And

Use `And` when you need to chain together a series of `Result` instances and you want to know about the first `Err` or the last `Ok` in the chain.

{% hint style="warning" %}
Arguments passed to `And` are eagerly evaluated. If your arguments are the result of a function call, use [#andthen](result-of-t-and-e.md#andthen "mention") instead, which is lazily evaluated.
{% endhint %}

{% hint style="info" %}
A logical `AND` operator is performed between the current result and the next result in the chain.
{% endhint %}

| Left   | Right  | Output |
| ------ | ------ | ------ |
| `Ok1`  | `Ok2`  | `Ok2`  |
| `Ok`   | `Err`  | `Err`  |
| `Err`  | `Ok`   | `Err`  |
| `Err1` | `Err2` | `Err1` |

```csharp
var x = Result.Ok<int, string>(1);
var y = Result.Err<int, string>("late error");
Debug.Assert(x.And(y) == Result.Err<int, string>("late error"));

var x = Result.Err<int, string>("early error");
var y = Result.Ok<int, string>(1);
Debug.Assert(x.And(y) == Result.Err<int, string>("early error"));

var x = Result.Err<int, string>("early error");
var y = Result.Err<int, string>("late error");
Debug.Assert(x.And(y) == Result.Err<int, string>("early error"));

var x = Result.Ok<int, string>(1);
var y = Result.Ok<int, string>(2);
Debug.Assert(x.And(y) == Result.Ok<int, string>(2));
```

### AndThen

Use `AndThen` when you need to chain a series of functions together that all return `Result` instances and you only care about the first `Err` or the last `Ok` . It performs the same operation as [#and](result-of-t-and-e.md#and "mention") for each lazily evaluated function.

```csharp
Result<string, string> SquareThenToString(int value)
    => Result.Try<int, string>(() => value ^ 2, _ => "overflow")
        .Map(x => x.ToString());
        
var x = Result.Ok<int, string>(2);
var y = Result.Ok<int, string>(4);
Debug.Assert(x.AndThen(SquareThenToString) == y);

var x = Result.Ok<int, string>(Int.MaxValue);
Debug.Assert(x.AndThen(SquareThenToString) == Result.Err<string, string>("overflow"));

var x = Result.Err<int, string>("NaN");
Debug.Assert(x.AndThen(SquareThenToString) == Result.Err<string, string>("NaN"));

```

### Or

Use `Or` when you need to chain together a series of `Result` instances and you want to know about the first `Ok` or the last `Err` in the chain.

{% hint style="warning" %}
Arguments passed to `Or` are eagerly evaluated. If your arguments are the result of a function call, use [#orelse](result-of-t-and-e.md#orelse "mention") instead, which is lazily evaluated.
{% endhint %}

{% hint style="info" %}
A logical `OR` operator is performed between the current result and the next result in the chain.
{% endhint %}

| Left   | Right  | Output |
| ------ | ------ | ------ |
| `Ok1`  | `Ok2`  | `Ok1`  |
| `Ok`   | `Err`  | `Ok`   |
| `Err`  | `Ok`   | `Ok`   |
| `Err1` | `Err2` | `Err2` |

```csharp
var x = Result.Ok<int, string>(1);
var y = Result.Ok<int, string>(2);
Debug.Assert(x.Or(y) == Result.Ok<int, string>(1));

var x = Result.Ok<int, string>(1);
var y = Result.Err<int, string>("error");
Debug.Assert(x.Or(y) == Result.Ok<int, string>(1));

var x = Result.Err<int, string>("error");
var y = Result.Ok<int, string>(1);
Debug.Assert(x.Or(y) == Result.Ok<int, string>(1));

var x = Result.Err<int, string>("error 1");
var y = Result.Err<int, string>("error 2");
Debug.Assert(x.Or(y) == Result.Err<int, string>("error 2"));

```

### OrElse

Use `OrElse` when you need to chain a series of functions together that all return `Result` instances and you only care about the first `Ok` or the last `Err`. It performs the same operation as [#or](result-of-t-and-e.md#or "mention") for each lazily evaluated function.

```csharp
Result<string, string> SquareThenToString(int value)
    => Result.Try<int, string>(() => value ^ 2, _ => "overflow")
        .Map(x => x.ToString());
        
var x = Result.Ok<int, string>(2);
var y = Result.Ok<int, string>(4);
Debug.Assert(x.OrElse(SquareThenToString) == x;

var x = Result.Ok<int, string>(Int.MaxValue);
Debug.Assert(x.OrElse(SquareThenToString) == Result.Err<string, string>("overflow"));

var x = Result.Err<int, string>("NaN");
Debug.Assert(x.OrElse(SquareThenToString) == Result.Err<string, string>("NaN"));

```

## Collections

Working with a `List<Result<TOk, TErr>>` — the results of validating a batch, or of calling something once per item — comes up often enough to have its own methods.

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

Use `AsEnumerable` on a single `Result` to treat it as a sequence of nothing or one, which is what lets the methods above compose out of LINQ.

```csharp
Result<int, string> result = Result.Ok<int, string>(1);

IEnumerable<int> sequence = result.AsEnumerable();
//               ^? [1], and [] for an Err
```

`Option<T>` has the same method, and `Flatten` on a sequence of either is built out of it.
