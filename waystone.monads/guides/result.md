---
description: Return failure as a value instead of throwing it, and chain fallible work.
icon: binary
---

# Result\<T, E>

`Result<T, E>` says the work might fail. It has exactly two shapes:

* `Ok<T, E>` — it worked, and here is the value.
* `Err<T, E>` — it failed, and here is why.

Where [`Option<T>`](option.md) models absence, `Result<T, E>` models failure. The
difference is the `E`. An `Option` tells you nothing arrived. A `Result` tells you
what went wrong.

{% hint style="info" %}
Some languages call this `Either<E, T>`. This library puts the success case first,
because that is the one you read most.
{% endhint %}

## Why not just throw?

Exceptions are control flow with a bomb strapped to it.

* They are invisible in a signature. `Reward ClaimReward(Quest)` looks total.
* They are easy to forget. Nothing makes you handle one.
* They do not compose. You cannot chain a method that throws.
* They are catastrophic in a chain. One throw unwinds everything above it.

You cannot tell which methods throw without reading their source. A `Result` puts
the failure in the return type, where you were already looking.

There is a second point, and it matters more. **`Err` is not an emergency.** When
you write:

```csharp
Result<Character, Error> FindCharacter(string name);
```

you are saying "this can fail, and here is what failure looks like". That is a
statement about your domain, not a panic button. Keep exceptions for the things
that really are exceptional.

{% hint style="info" %}
If you are writing `try`/`catch` just to return a fallback or log something, you
wanted a `Result`.
{% endhint %}

## Create one

```csharp
Result<int, string> ok = Result.Ok<int, string>(1);
Result<int, string> err = Result.Err<int, string>("Something went wrong");
```

Leave `TErr` off and it defaults to the library's own `Error` type:

```csharp
Result<int, Error> okWithDefaultError = Result.Ok<int>(1);
Result<int, Error> errWithDefaultError =
    Result.Err<int>(new Error("quest.failed", "Something went wrong"));
```

See [Errors](errors.md) for what `Error` holds and how to build one from an
enum.

### Neither side can hold null

```csharp
Result.Ok<string, Error>(null!); // throws ArgumentNullException
```

`TOk` and `TErr` are both constrained `notnull`, and the constructors enforce it.
The guard arrived in 5.5.0 — before it, an `Ok` could hold `null`, and the `null`
surfaced later as a `NullReferenceException` somewhere in your own code.

A **default** value is fine, though. It is a real value:

```csharp
Result<int, string> zero = Result.Ok<int, string>(0);
Result<Guid, string> empty = Result.Ok<Guid, string>(Guid.Empty);
```

{% hint style="info" %}
`Result.Try` is the exception. It returns an `Err` for a `null` rather than
throwing — that is the point of it.
{% endhint %}

## Chain fallible steps

`AndThen` is the workhorse. Each step returns a `Result`, and the first `Err`
short-circuits the rest.

```csharp
Result<Reward, Error> reward = FindCharacter(name)
    .AndThen(GetQuest)
    .AndThen(ClaimReward);
```

If `FindCharacter` fails, `GetQuest` and `ClaimReward` never run, and the original
error arrives at the caller untouched. No `try`/`catch`. No special cases.

{% hint style="info" %}
This is what people mean by railway-oriented programming. Your computation runs on
one of two tracks, and it never jumps between them by accident.
{% endhint %}

`Map` is the same idea when the next step *cannot* fail — it changes the `Ok`
value and leaves an `Err` alone.

## Work on the error side

This is what separates a `Result` from an `Option`, so it is worth knowing well.

### MapErr

`MapErr` transforms the error, leaving the success value alone. Reach for it when
two pieces of code disagree about the error type and you need them to chain.

```csharp
Result<int, Error> lengthResult = RollForName()            // Result<string, string>
    .MapErr(message => new Error("name.failed", message))  // Result<string, Error>
    .Map(name => CountRunes(name))                         // Result<Result<int, Error>, Error>
    .Flatten();                                            // Result<int, Error>
```

The last two lines are a `Map` that nests, then a `Flatten` that undoes it. That
is exactly what `AndThen` does in one step:

```csharp
Result<int, Error> lengthResult = RollForName()            // Result<string, string>
    .MapErr(message => new Error("name.failed", message))  // Result<string, Error>
    .AndThen(name => CountRunes(name));                    // Result<int, Error>
```

Prefer the second. The first is shown because you will meet it in code that grew
one method at a time.

### InspectErr

`InspectErr` runs a side effect on the error and hands the result back unchanged,
so the chain continues. Logging is what it is for.

```csharp
Result<string, string> username = FindCharacter("Percy")
    .InspectErr(err => Console.WriteLine($"Find character failed: {err.Message}"))
    .Map(character => character.Username)
    .MapErr(err => err.Message);
```

`Inspect` is the same thing on the `Ok` branch. Use both together and each log
line runs only on its own track:

```csharp
Result<Quest, Error> quest = LoadQuest(id)
    .Inspect(q => logger.LogInformation("Loaded quest {Id}", q.Id))
    .InspectErr(e => logger.LogWarning("Load failed: {Code} {Message}", e.Code, e.Message));
```

### UnwrapErr and ExpectErr

These pull the error out and throw if the result was actually `Ok`.

```csharp
Result<int, string> ok = Result.Ok<int, string>(10);
ok.UnwrapErr(); // throws UnwrapException

Result<int, string> err = Result.Err<int, string>("Error");
err.UnwrapErr(); // returns "Error"
```

`ExpectErr` does the same, but you supply the message:

```csharp
Result.Ok<int, string>(10).ExpectErr("Must be error");
// throws UnmetExpectationException with message "Must be error"
```

{% hint style="danger" %}
Both are intentional points of failure, like `First` on an empty sequence. Use
them when you have already established the result is an `Err` — in a test, say.
Everywhere else, use `Match`.
{% endhint %}

## Get the value back out

### Match

```csharp
string message = result.Match(
    reward => reward.Item,
    error => error.Message);
```

Both branches, one plain value out. This is the default answer.

### Unwrap with a fallback

```csharp
Reward orFallback = result.UnwrapOr(new Reward("A handful of copper"));
Reward orComputed = result.UnwrapOrElse(error => new Reward(error.Code));
```

`UnwrapOrElse` gets the error, so you can decide the fallback based on what went
wrong. It only runs on an `Err`.

### Pattern matching

Since 7.0.0, `Result<T, E>` deconstructs:

```csharp
if (result is Err<Reward, Error>(var error))
{
    logger.LogWarning("No reward: {Message}", error.Message);
}
```

And exhaustively:

```csharp
string message = result switch
{
    Ok<Reward, Error>(var reward) => reward.Item,
    Err<Reward, Error>(var error) => error.Message,
    _ => throw new UnreachableException(),
};
```

{% hint style="info" %}
The discard arm is there because the compiler cannot see that `Ok` and `Err` are
the only two cases. It never runs.
{% endhint %}

## Check without unwrapping

```csharp
Result<DateTime, Error> parsed = SafeParse("2025-01-01");

parsed.IsOkAnd(dateTime => dateTime > new DateTime(2024, 1, 1)); // true

Result<DateTime, Error> failed = SafeParse("2025");
failed.IsErrAnd(error => error.Code == ErrorCodes.MalformedDateTime); // true
```

* `IsOkAnd` — it succeeded **and** the value passes the predicate.
* `IsErrAnd` — it failed **and** the error passes the predicate.

## Combine two results

`And` and `Or` take a result you already have. `AndThen` and `OrElse` take a
function, so the second result is only built when it is needed.

{% hint style="warning" %}
`And` and `Or` evaluate their argument eagerly. If producing it costs anything,
use `AndThen` or `OrElse` instead.
{% endhint %}

`And` gives you the first `Err`, or the last `Ok`:

| Left | Right | Output |
| --- | --- | --- |
| `Ok1` | `Ok2` | `Ok2` |
| `Ok` | `Err` | `Err` |
| `Err` | `Ok` | `Err` |
| `Err1` | `Err2` | `Err1` |

`Or` gives you the first `Ok`, or the last `Err`:

| Left | Right | Output |
| --- | --- | --- |
| `Ok1` | `Ok2` | `Ok1` |
| `Ok` | `Err` | `Ok` |
| `Err` | `Ok` | `Ok` |
| `Err1` | `Err2` | `Err2` |

`OrElse` is how you recover. Its function runs on the **error**, not the value:

```csharp
Result<int, string> Recover(string error)
    => error == "NaN"
        ? Result.Ok<int, string>(0)
        : Result.Err<int, string>(error);

Result.Ok<int, string>(2).OrElse(Recover);            // Ok(2), untouched
Result.Err<int, string>("NaN").OrElse(Recover);       // Ok(0), recovered
Result.Err<int, string>("overflow").OrElse(Recover);  // Err("overflow"), still failed
```

## Work with a collection of them

A sequence of `Result` has its own helpers — `Collect`, `Partition`, `Flatten`,
`FlattenErr` and `AsEnumerable`. They live in
`Waystone.Monads.Results.Extensions`, and are covered on the
[Result\<T, E> collections reference](../reference/result/collections.md).

The short version:

* `Collect` — all or nothing. One `Err` fails the whole batch.
* `Partition` — keeps both sides, so you learn which ones failed.

For LINQ query syntax over a `Result`, see
[Waystone.Monads.Linq](../packages/linq.md).

## Printing and logging

**`ToString()` never shows the wrapped value.** You get the state and nothing
else:

```csharp
Result.Ok<int, Error>(1).ToString()  // "Ok { IsOk = True, IsErr = False }"
Result.Err<int, Error>(e).ToString() // "Err { IsOk = False, IsErr = True }"
```

`Result<TOk, TErr>` is a record and both sides keep their value in a private
property, so the compiler-generated `ToString()` has nothing to print.
Interpolating a result into a log message tells you which branch you are on, never
what it holds. Use the `Inspect` / `InspectErr` pair above instead.

## When to reach for it

Use `Result<T, E>` when:

* A function can fail and you want that visible in the signature.
* You care *why* it failed.
* You want the caller to handle the failure explicitly.
* You are parsing, validating, or transforming input you do not control.
* You want exceptions to mean something has genuinely gone wrong.

Reach for [`Option<T>`](option.md) instead when you do not care about the reason.

## Where to go next

* [Option\<T>](option.md) — the same idea, for absence.
* [Errors](errors.md) — the `Error` type, error codes, and building one from an exception.
* [Exceptions](exceptions.md) — what the library throws, and when.
* [Async](async.md) — keeping a chain intact across an `await`.
* [Result\<T, E> API](../reference/result/README.md) — every overload, when you need one this page did not show.
