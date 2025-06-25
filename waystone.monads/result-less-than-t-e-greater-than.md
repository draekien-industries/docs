# Result\<T, E>

## Control Flow

### IsOk / IsErr

Use `IsOk` and `IsErr` when you need to check the state of the `Result<T, E>` and don't need to access it's value just quite yet.

```csharp
Result<DateTime, Error> safeParseResult = SafeParse("2025-01-01");

safeParseResult.IsOk; // true
safeParseResult.IsErr; // false
```

{% hint style="info" %}
These are ideal for short-circuiting logic or quick guards, but avoid using them for full branching. Reach for [`Match`](using-the-library/core-functionality.md#match) when both branches matter.
{% endhint %}

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
//                      ^? Err<DateTime, Error>(new Error(ErrorCodes.MalformedDateTime))

safeParseResult.IsErrAnd(error => error.Code == ErrorCodes.MalformedDateTime); // true
```

## Transform

{% hint style="info" %}
Refer to the [#transform](using-the-library/core-functionality.md#transform "mention")section under [core-functionality.md](using-the-library/core-functionality.md "mention") to learn about the other transforms available for a `Result<T, E>`
{% endhint %}

### MapErr

The counterpart to [#map](using-the-library/core-functionality.md#map "mention"), use `MapErr` when you need to transform the contained value if it is an `Err`. It is useful when you need to transform the `Err` type in order to continue chaining monadic operations.

```csharp
Result<string, string> GenerateName();
Result<int, Error> GetLength(string value);

Result<int, Error> lengthResult = GenerateName() // Result<string, string>
    .MapErr(message => new Error(message))       // Result<string, Error>
    .Map(name => GetLength(name))                // Result<Result<int, Error>, Error>
    .Flatten();                                  // Result<int, Error>
    
```

## Consume

### ExpectErr

The counterpart to [#inspect](using-the-library/core-functionality.md#inspect "mention"), use `ExpectErr` when you want to consume the monadic wrapper and fail loudly when the `Result` is an `Ok`. It allows you to provide a meaningful exception message to explain why the error is expected.

```csharp
Result<int, string> result = Result.Ok<int, string>(10);
result.ExpectErr("Must be error"); // throws UnmetExpectationException with message "Must be error"
```

{% hint style="warning" %}
An `UnmetExpectationException` with your provided message will be thrown when the value is a success.
{% endhint %}

### UnwrapErr

The counterpart to [#unwrap](using-the-library/core-functionality.md#unwrap "mention"), use `UnwrapErr` when you are certain the `Result` is an `Err` and you want to fail loudly if it is an `Ok`.

{% hint style="info" %}
Avoid `UnwrapErr` unless you've validated the presence of an `Err` upstream. It's an intentional point of failure, like `First` on an empty sequence. In most cases, you should reach for [#match](result-less-than-t-e-greater-than.md#match "mention") or [#map](result-less-than-t-e-greater-than.md#map "mention").
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

## Side-Effect

{% hint style="info" %}
Refer to the [#side-effect](using-the-library/core-functionality.md#side-effect "mention")section under [core-functionality.md](using-the-library/core-functionality.md "mention") to learn about the other side-effects available for a `Result<T, E>`
{% endhint %}

### InspectErr

The counterpart to [#inspect](using-the-library/core-functionality.md#inspect "mention"), use `InspectErr` when you want to run a side effect against an `Err` without modifying the underlying value. The most common use case for `InspectErr` is to log an error in the event of a failure.

{% hint style="info" %}
Reach for [#maperr](result-less-than-t-e-greater-than.md#maperr "mention")if you want to transform the value of an `Err`
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
Arguments passed to `And` are eagerly evaluated. If your arguments are the result of a function call, use [#andthen](result-less-than-t-e-greater-than.md#andthen "mention") instead, which is lazily evaluated.
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

Use `AndThen` when you need to chain a series of functions together that all return `Result` instances and you only care about the first `Err` or the last `Ok` . It performs the same operation as [#and](result-less-than-t-e-greater-than.md#and "mention") for each lazily evaluated function.

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
Arguments passed to `Or` are eagerly evaluated. If your arguments are the result of a function call, use [#orelse](result-less-than-t-e-greater-than.md#orelse "mention") instead, which is lazily evaluated.
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

Use `OrElse` when you need to chain a series of functions together that all return `Result` instances and you only care about the first `Ok` or the last `Err`. It performs the same operation as [#or](result-less-than-t-e-greater-than.md#or "mention") for each lazily evaluated function.

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

