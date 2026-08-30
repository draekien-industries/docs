---
description: Methods that take a Result and give you back a Result.
---

# Transform

Every method here returns a `Result`, so the chain continues. To end it, see
[Consume](consume.md).

## Map

```csharp
Result<TOut, TErr> Map<TOut>(Func<TOk, TOut> map)
```

Applies a transformation to the success value.

```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Consent");
Result<int, string> lengthResult = nameResult.Map(name => name.Length);
```

**On an `Err`:** the delegate never runs, and the error passes through untouched.

## MapErr

```csharp
Result<TOk, TOut> MapErr<TOut>(Func<TErr, TOut> map)
```

The counterpart. Transforms the error, leaving the success value alone. Reach for
it when two pieces of code disagree about the error type and you need them to
chain.

```csharp
Result<int, Error> lengthResult = RollForName()             // Result<string, string>
    .MapErr(message => new Error("name.failed", message))   // Result<string, Error>
    .AndThen(name => CountRunes(name));                     // Result<int, Error>
```

**On an `Ok`:** the delegate never runs.

## AndThen

```csharp
Result<TOut, TErr> AndThen<TOut>(Func<TOk, Result<TOut, TErr>> map)
```

Chains a step that itself returns a `Result`. It performs the same operation as
[`And`](#and), for each lazily evaluated function.

```csharp
Result<string, string> SquareThenToString(int value)
    => Result.Try<int, string>(() => checked(value * value), _ => "overflow")
        .Map(x => x.ToString());

Result<int, string> two = Result.Ok<int, string>(2);
two.AndThen(SquareThenToString);          // Ok("4")

Result<int, string> big = Result.Ok<int, string>(int.MaxValue);
big.AndThen(SquareThenToString);          // Err("overflow")

Result<int, string> nan = Result.Err<int, string>("NaN");
nan.AndThen(SquareThenToString);          // Err("NaN")
```

**On an `Err`:** short-circuits. Later steps never run.

`Map` followed by [`Flatten`](nesting.md#flatten) does the same thing in two calls.
Prefer `AndThen`.

## And

```csharp
Result<TOut, TErr> And<TOut>(Result<TOut, TErr> other)
```

Gives you the first `Err`, or the last `Ok`.

| Left | Right | Output |
| --- | --- | --- |
| `Ok1` | `Ok2` | `Ok2` |
| `Ok` | `Err` | `Err` |
| `Err` | `Ok` | `Err` |
| `Err1` | `Err2` | `Err1` |

```csharp
Result.Ok<int, string>(1).And(Result.Err<int, string>("late error"));
//     ^? Err("late error")

Result.Err<int, string>("early error").And(Result.Ok<int, string>(1));
//     ^? Err("early error")
```

{% hint style="warning" %}
**Evaluated eagerly.** If the argument is the result of a function call, use
[`AndThen`](#andthen).
{% endhint %}

## Or

```csharp
Result<TOk, TOut> Or<TOut>(Result<TOk, TOut> other)
```

Gives you the first `Ok`, or the last `Err`.

| Left | Right | Output |
| --- | --- | --- |
| `Ok1` | `Ok2` | `Ok1` |
| `Ok` | `Err` | `Ok` |
| `Err` | `Ok` | `Ok` |
| `Err1` | `Err2` | `Err2` |

```csharp
Result.Ok<int, string>(1).Or(Result.Err<int, string>("error"));
//     ^? Ok(1)

Result.Err<int, string>("error 1").Or(Result.Err<int, string>("error 2"));
//     ^? Err("error 2")
```

{% hint style="warning" %}
**Evaluated eagerly.** Use [`OrElse`](#orelse) if the argument costs something.
{% endhint %}

## OrElse

```csharp
Result<TOk, TOut> OrElse<TOut>(Func<TErr, Result<TOk, TOut>> createElse)
```

The same as `Or`, lazily. This is how you recover — and note the factory takes the
**error**, not the success value.

```csharp
Result<int, string> Recover(string error)
    => error == "NaN"
        ? Result.Ok<int, string>(0)
        : Result.Err<int, string>(error);

Result.Ok<int, string>(2).OrElse(Recover);            // Ok(2), untouched
Result.Err<int, string>("NaN").OrElse(Recover);       // Ok(0), recovered
Result.Err<int, string>("overflow").OrElse(Recover);  // Err("overflow"), still failed
```

**On an `Ok`:** the factory never runs.
