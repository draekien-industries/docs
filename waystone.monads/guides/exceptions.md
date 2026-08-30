---
description: What the library throws, when it throws it, and what to write instead.
icon: bomb
---

# Exceptions

The point of this library is that failure is a value. So it throws rarely, and
when it does, it is because you asked for a value that was not there.

There are two exception types, and both mean the same thing: **you skipped the
check.**

{% hint style="info" %}
Looking for how to build the failure you put *inside* an `Err`? That is
[Errors](errors.md).
{% endhint %}

## UnwrapException

Thrown when you take the value out of a monad that does not have one.

```csharp
Option<string> none = Option.None<string>();
none.Unwrap(); // throws UnwrapException

Result<int, string> err = Result.Err<int, string>("the ritual fizzled");
err.Unwrap(); // throws UnwrapException
```

And in the other direction, when you take the error out of something that
succeeded:

```csharp
Result<int, string> ok = Result.Ok<int, string>(20);
ok.UnwrapErr(); // throws UnwrapException
```

So: `Unwrap` on a `None` or an `Err`, and `UnwrapErr` on an `Ok`.

The async versions — `UnwrapAsync` and `UnwrapErrAsync` — throw for exactly the
same reasons.

### What to write instead

Nine times out of ten you wanted one of these:

| Instead of | Write | Because |
| --- | --- | --- |
| `option.Unwrap()` | `option.Match(…, …)` | Handles both cases, returns a plain value. |
| `option.Unwrap()` | `option.UnwrapOr(fallback)` | You already have something to fall back to. |
| `option.Unwrap()` | `option.UnwrapOrElse(() => …)` | The fallback costs something to build. |
| `option.Unwrap()` | `option.Map(…)` | You were going to keep working with it anyway. |

`Unwrap` is not forbidden. It is the right call when absence really would be a
bug and you want to fail loudly and immediately — the same reasoning as `First`
on a sequence you know is not empty. The analyzer will tell you when you have
reached for it out of habit.

## UnmetExpectationException

Thrown for the same reasons, by `Expect` and `ExpectErr` rather than `Unwrap` and
`UnwrapErr`. The difference is that you supply the message.

```csharp
Option<string> none = Option.None<string>();
none.Expect("the familiar must be summoned");
// throws UnmetExpectationException with message "the familiar must be summoned"

Result<int, string> err = Result.Err<int, string>("the ritual fizzled");
err.Expect("the ritual must succeed"); // throws UnmetExpectationException

Result<int, string> ok = Result.Ok<int, string>(20);
ok.ExpectErr("the ritual must fail"); // throws UnmetExpectationException
```

`ExpectAsync` and `ExpectErrAsync` behave the same way.

### When to prefer Expect over Unwrap

Use `Expect` when the throw is deliberate and you can say something useful about
why. The message goes straight into the exception, so whoever reads the log gets
your reasoning rather than a stack trace and a shrug.

That makes `Expect` the better choice in a test, in application startup, and
anywhere else the invariant is worth stating out loud. `Unwrap` is the terser
option when there is nothing to add.

## Exceptions the library catches

Separately from the two above, some operations *swallow* exceptions on purpose.

`Option.Try`, `Result.Try` and their async counterparts run a factory that might
throw, and turn a throw into a `None` or an `Err`. That is the whole point of
them.

```csharp
Option<int> parsed = Option.Try(() => int.Parse(text));
// None if the parse threw
```

Two things to know about that:

* **The exception is not lost.** It goes to your configured exception logger. See
  [Configuration](../using-the-library/configuration.md) and
  [Observability](../using-the-library/observability.md).
* **`OperationCanceledException` is let through.** Cancellation is not a failure
  of your work, so it is not turned into an `Err`. See
  [Configuration](../using-the-library/configuration.md#cancellation).

## Turning an exception into an Error

The other direction. You caught something at a boundary, and you want it as a
`Result` rather than a rethrow. Both [error types](errors.md) convert.

```csharp
try
{
    // do work
}
catch (ScryingFailedException e)
{
    Error error = Error.FromException(e);
    //    ^? Code: "ScryingFailed", Message: e.Message
}
```

The code is the exception's type name with a trailing `Exception` removed. There
is no prefix, the suffix match ignores case, and the exception's message is never
read — so nothing from its text reaches the code.

| Exception type | Resulting code |
| --- | --- |
| `SqlException` | `Sql` |
| `InvalidOperationException` | `InvalidOperation` |
| `ScryingFailedException` | `ScryingFailed` |
| `TimeoutException` | `Timeout` |
| `Exception` | `Exception` |

`Exception` itself is the one special case. It keeps its whole name rather than
reducing to an empty code.

`ErrorCode.FromException` does the same job when you want only the code:

```csharp
ErrorCode code = ErrorCode.FromException(e); // "ScryingFailed"
```

{% hint style="warning" %}
**This is a fallback, not a strategy.** Codes derived from exception types drift
out of step with the codes you define by hand, and they do not help at all for
failures that were never exceptions. Reach for it at a boundary you do not
control, not throughout your domain.
{% endhint %}

{% hint style="info" %}
To change what these produce, supply your own `ErrorCodeFactory` to the global
`MonadOptions` and override `FromException`. See
[Configuration](../using-the-library/configuration.md).
{% endhint %}

## Exceptions from the constructors

One more, and it is not from this library's own hierarchy.

Neither side of a `Result` can hold `null`, and nor can a `Some`. The constructors
enforce it:

```csharp
Result.Ok<string, Error>(null!); // throws ArgumentNullException
```

A *default* value is fine — `Result.Ok<int, string>(0)` is an `Ok` holding zero.
It is `null` specifically that is rejected. See
[Result\<T, E>](result.md#neither-side-can-hold-null).

## Where to go next

* [Errors](errors.md) — building the failure value you return.
* [Option\<T>](option.md) and [Result\<T, E>](result.md) — the safe ways out of a monad.
* [Analyzer rules](../using-the-library/analyzer-rules.md) — what flags an `Unwrap` you should not have written.
