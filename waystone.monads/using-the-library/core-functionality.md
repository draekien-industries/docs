---
description: >-
  Learn about the different functionality that is common to both the Option and
  Result types.
---

# Core Functionality

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

## Introduction

While `Option<T>` and `Result<T, E>` serve different purposes (absence vs. success/failure), they share a common operational model. If you are fluent with either the `Option<T>` or `Result<T, E>` already, you're 90% of the way to mastering the other.

The core APIs defined below work the same across both types, differing only in what they mean.

## Creation

Both `Option` and `Result` types have factory methods that allow you to create an instance of the monad in one of it's two states.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<string> some = Option.Some("Hello Bees!");
Option<string> none = Option.None<string>();
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<int, string> ok = Result.Ok<int, string>(1);
Result<int, string> err = Result.Err<int, string>("Something went wrong...");
```

Supply both type parameters when you use your own error type. If you are happy
with the built in [`Error`](errors-and-exceptions.md#error) type, use the single
type parameter overloads instead — they default the error type to `Error`.

```csharp
Result<int, Error> ok = Result.Ok<int>(1);
Result<int, Error> err = Result.Err<int>(new Error("MyCode", "Something went wrong..."));
```

You can also create the error from an enum. Mark the enum with `[ErrorCodeCatalog]` and
the source generator gives you a factory per member, with the message required.

```csharp
[ErrorCodeCatalog]
enum UserErrors
{
    NotFound
}

Result<User, Error> err = Result.Err<User>(
    UserErrorsCatalog.Errors.NotFound("The user was not found"));
```

{% hint style="info" %}
`[ErrorCodeCatalog]` generates `UserErrorsCatalog` at compile time. Passing an enum
straight to `Result.Err` was removed in 7.0.0 — see
[Generated error codes](generated-error-codes.md).
{% endhint %}

{% hint style="warning" %}
**Neither an `Ok` nor an `Err` can hold null.** Pass one and you get an
`ArgumentNullException`. Every factory method above funnels through the same guard.

```csharp
Result.Ok<string, Error>(null!); // throws
```

New in 5.5.0. Before that, `Result.Ok<string, Error>(null!)` gave you an `Ok`
holding null, and the null surfaced later as a `NullReferenceException` in your
own code. `TOk` and `TErr` are constrained `notnull`, so the compiler already
warned you; now the runtime agrees.

A default value is fine and always has been. `Result.Ok<int, string>(0)` is an
`Ok` holding `0`. Only null is rejected.
{% endhint %}
{% endtab %}
{% endtabs %}

Use `Try` to safely capture potentially exception-throwing logic inside a monadic wrapper. This is useful if you want to begin a monadic chain from a method you do not have control over.

`Try` asks one question: did the factory hand back a value you can work with? If it did not, because it threw or because it returned something the monad cannot hold, you get the empty or failed case.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<User> maybeUser = Option.Try(() => GetCurrentUser());
```

If the `GetCurrentUser` call throws, the exception is caught and logged via your [configured exception logger](configuration.md), and you get back a `None<User>` instance.

You also get a `None<User>` if the factory returns null, because a `Some` cannot hold one. Nothing is logged in that case, because nothing threw. A default value is fine: `Option.Try(() => 0)` gives you `Some(0)`.

{% hint style="warning" %}
**A cancellation is not caught.** `Try` and `TryAsync` let an
`OperationCanceledException` propagate, so a cancelled operation throws rather
than becoming a `None`. Cancelling is you asking the work to stop, not the work
failing. Call `MonadOptions.Configure(options => options.UseCancellationAsFailure())` if
you want the pre-6.0.0 behaviour back — see
[Configuration](configuration.md#cancellation).
{% endhint %}

{% hint style="danger" %}
**Do not pass an async factory to `Try`.** It compiles and gives you an
`Option<Task<T>>` with no exception handling at all. Use `TryAsync`.
[`WM1011`](analyzer-rules.md#wm1011) reports every occurrence.
{% endhint %}
{% endtab %}

{% tab title="Result" %}
```csharp
Result<User, string> result = Result.Try(
    factory: () => GetCurrentUser(),
    onError: ex => ex.Message
);
```

If the `GetCurrentUser` call throws, the exception is caught and logged via your configured exception logger, and the `onError` delegate you passed runs.&#x20;

`Try` also calls `onError` when the factory returns null, because an `Ok` cannot hold null. It passes you an `ArgumentNullException` naming the `factory` argument. Nothing is logged, because nothing threw.

This is the one place a null does not throw. `Try` exists so you can hand over a delegate and learn whether a workable value came back, without wrapping the call in a `try` yourself — so it turns the null into an `Err` for you.

{% hint style="info" %}
The `onError` delegate gives you a way to transform the caught exception into an error type of your choosing.
{% endhint %}

If you are happy with the built in `Error` type, use the single type parameter
overload. It converts the exception with
[`Error.FromException`](errors-and-exceptions.md#error-from-exception), so you do
not pass an `onError` delegate.

```csharp
Result<User, Error> result = Result.Try<User>(() => GetCurrentUser());
```
{% endtab %}
{% endtabs %}

If your factory returns a `Task`, use `TryAsync` instead. See [Async](async.md).

### Passing state to the factory

`Try` and `TryAsync` each take an optional first argument that they hand to your
factory. Use it to keep the factory from capturing:

```csharp
Option<int> parsed = Option.Try(text, static value => int.Parse(value));

Result<int, Error> result = Result.Try(text, static value => int.Parse(value));
```

See [State overloads](#state-overloads) below for why this matters.

## Transform

Transformations are the bread and butter of monadic chains. They are used to transform the value inside the monadic wrapper and perform operations on them.

### Map

`Map` is the most common transform. Use `Map` to apply a transformation to the contained value if it is present or successful.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<string> maybeName = Option.Some("Henry Crabgrass");
Option<int> maybeLength = maybeName.Map(name => name.Length);
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Consent");
Result<int, string> lengthResult =  nameResult.Map(name => name.Length);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
`Map` returns the same monadic wrapper type - `Option<T>` stays an `Option`, and `Result<T, E>` stays a `Result`.
{% endhint %}

### More Transforms

There are more transform methods specific to the `Option<T>` and `Result<T, E>` types.&#x20;

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th></tr></thead><tbody><tr><td><strong>Option Transforms</strong></td><td>Learn about the transform methods specific to the <code>Option&#x3C;T></code> monad.</td><td><a href="../guides/option.md#transform-it">#transform-it</a></td></tr><tr><td><strong>Result Transforms</strong></td><td>Learn about the transform methods specific to the <code>Result&#x3C;T, E></code> monad.</td><td><a href="result-of-t-and-e.md#transform">#transform</a></td></tr></tbody></table>

## State overloads

Nearly every method that takes a delegate has a sibling that takes **your data
first** and hands it to the delegate. Use it when the delegate would otherwise
capture a local or a parameter.

```diff
-option.Map(value => value + offset);
+option.Map(offset, static (value, state) => value + state);
```

Both lines do the same thing. The second one allocates 88 fewer bytes every time
it runs — 24 for the display class the closure needs, 64 for the delegate.

### Where you can use it

| Type | Methods |
| --- | --- |
| `Option<T>` | `IsSomeAnd`, `IsNoneOr`, `Match`, `Map`, `MapOr`, `MapOrDefault`, `MapOrElse`, `AndThen`, `Filter`, `Inspect`, `UnwrapOrElse`, `OrElse`, `OkOrElse` |
| `Result<T, E>` | `IsOkAnd`, `IsErrAnd`, `Match`, `Map`, `MapOr`, `MapOrDefault`, `MapOrElse`, `MapErr`, `AndThen`, `OrElse`, `UnwrapOrElse`, `Inspect`, `InspectErr` |
| Factories | `Option.Try`, `Option.TryAsync`, `Result.Try`, `Result.TryAsync` |

`ZipWith` and `Reduce` are the exceptions, and they are not getting one. Both
already hand their delegate every value the call involves, so there is normally
nothing left for it to capture.

### What the delegate receives

The state is always the first argument of the call. What your delegate receives
depends on whether the branch it runs in has a value to give it.

On `Result<T, E>` every delegate takes the value and then the state — the success
value or the error, depending on which branch it is.

```csharp
result.UnwrapOrElse(fallback, static (error, state) => state);
```

On `Option<T>` the delegates that run when the option is `None` take the state
alone, because there is no value to pass. That is the `onNone` branch of `Match`,
and all of `UnwrapOrElse`, `OrElse` and `OkOrElse`.

```csharp
option.UnwrapOrElse(fallback, static state => state);
```

### Match saves the most

`Match` takes two delegates, so a capturing call pays twice. The two branches
share one display class, but each one needs its own delegate — 152 bytes a call
rather than 88.

```diff
-option.Match(
-    name => name.Length + offset,
-    () => fallback);
+option.Match(
+    (offset, fallback),
+    static (name, state) => state.offset + name.Length,
+    static state => state.fallback);
```

State is a single argument, so pass a tuple when you need more than one value.
This works for both forms of `Match` — the one that returns a value, and the one
that takes `Action` delegates and returns nothing.

### Write the lambda `static`

This is the part that is easy to get wrong. A lambda that happens not to capture
measures the same as a `static` one — the compiler caches both. But nothing stops
a later edit from reaching for an outer variable again, and the allocation comes
straight back with no warning.

Marking the lambda `static` makes the compiler enforce it:

```csharp
// the compiler rejects any capture in here
option.Map(offset, static (value, state) => value + state);
```

Use `static` in every state overload you write.

### MapOrElse threads state through both delegates

`MapOrElse` takes two delegates and hands the same state to each one. Passing it
to the map alone would leave `createDefault` capturing, and the allocation would
still be there.

```csharp
option.MapOrElse(
    fallback,
    static state => state,
    static (value, state) => value + state);
```

### How the overloads stay apart

Each state overload adds its parameter at the front and takes exactly one more
argument than its closure sibling. That is what keeps the compiler from having to
choose between them. Do not "tidy" a future overload by reusing an existing slot.

### On the async surface

Some of the `…Async` methods take state as well, but not all of them yet.

| Type | Async methods that take state |
| --- | --- |
| `Option<T>` | `IsNoneOrAsync`, `InspectAsync`, `MapOrDefaultAsync` |
| `Result<T, E>` | `IsOkAndAsync`, `IsErrAndAsync`, `MatchAsync`, `InspectAsync`, `InspectErrAsync`, `MapOrDefaultAsync` |

Everywhere else on the [async surface](async.md#the-full-surface) you still need a
closure. Where the allocation matters, `await` the task first and call the
synchronous overload on the result.

{% hint style="info" %}
[`WM2017`](analyzer-rules.md#wm2017) reports a delegate that captures where one of
these overloads exists, so you do not have to find them by hand.
{% endhint %}

## State Checks

{% hint style="info" %}
These are ideal for short-circuiting logic or quick guards, but avoid using them for full branching. Reach for [`Match`](core-functionality.md#match) when both branches matter.
{% endhint %}

{% tabs %}
{% tab title="Option" %}
Use `IsOk` and `IsErr` when you need to check the state of the `Result<T, E>` and don't need to reach its value yet.

```csharp
Result<DateTime, Error> safeParseResult = SafeParse("2025-01-01");

safeParseResult.IsOk; // true
safeParseResult.IsErr; // false
```
{% endtab %}

{% tab title="Result" %}
Use `IsSome` and `IsNone` when you want to check the state of the monad and don't need to reach its value yet.

```csharp
Option<string> maybeName = Option.Some("John");

maybeName.IsSome; // true
maybeName.IsNone; // false
```
{% endtab %}
{% endtabs %}

## Consume

You will eventually need to escape from a monadic wrapper to reach the concrete value. This is where consume methods come in.

### Match

Use `Match` to consume the monadic wrapper when you are uncertain of the wrapper's current state. It enables you to pattern match on the outcome and apply branching logic.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<string> maybeName = Option.Some("Travis");

int length = maybeName.Match(
    name => name.Length,
    () => 0
);
```

{% hint style="info" %}
`length` will be `0` if `maybeName` is a `None<string>`
{% endhint %}
{% endtab %}

{% tab title="Result" %}
```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Sam");

int length = nameResult.Match(
    name => name.Length,
    _ => 0
);
```

{% hint style="info" %}
`length` will be `0` if `nameResult` is an `Err<string, string>`
{% endhint %}
{% endtab %}
{% endtabs %}

`Match` also has an overload that takes state, and it is the one that saves the
most — see [Match saves the most](#match-saves-the-most).

### Pattern matching with Deconstruct

From 7.0.0 the case types deconstruct, so C# pattern matching can bind the value
positionally.

```csharp
using Waystone.Monads.Options;

if (maybeName is Some<string>(var name))
{
    Console.WriteLine(name.Length);
}
```

`Result` works the same way, on both halves:

```csharp
using Waystone.Monads.Results;

if (nameResult is Ok<string, string>(var name)) { /* ... */ }
if (nameResult is Err<string, string>(var error)) { /* ... */ }
```

Three `Deconstruct` methods exist, and only three:

| Type | Signature | Binds |
| --- | --- | --- |
| `Some<T>` | `Deconstruct(out T value)` | The contained value |
| `Ok<TOk, TErr>` | `Deconstruct(out TOk value)` | The Ok value |
| `Err<TOk, TErr>` | `Deconstruct(out TErr error)` | The error |

Each one is documented as never handing you null.

#### None has none, deliberately

There is nothing to bind, and `option is None<string>` already tests the case.

So `option is None<string>()`, with the parentheses, is a **compile error**, not a
redundant spelling. An empty positional pattern still needs a `Deconstruct` to bind
against, and there is none:

```
CS8129: No suitable 'Deconstruct' instance or extension method was found for type
        'None<string>', with 0 out parameters and a void return type.
```

Write it without the parentheses.

#### You cannot deconstruct the monad itself

`Deconstruct` is on the case types, not on `Option<T>` or `Result<TOk, TErr>`. So this
does not compile:

```csharp
var (a, b) = maybeName; // CS1061, CS8129, CS8130
```

There is no state-plus-value tuple to destructure. Test the case first, then bind.

### Where Match still wins

`Match` is not the old way of doing this. It is still the right tool for the common job,
for one concrete reason.

**A `switch` expression over the closed hierarchy warns.** Even with both cases covered:

```csharp
int length = maybeName switch
{
    Some<string>(var name) => name.Length,
    None<string> => 0,
};
```

```
CS8509: The switch expression does not handle all possible values of its input type
        (it is not exhaustive). For example, the pattern '_' is not covered.
```

The hierarchy really is closed, because an internal member stops anything outside the assembly
deriving from `Option<T>`, but the compiler's exhaustiveness check has no knowledge of
that idiom, so it cannot see it. Silencing the warning means adding an unreachable arm:

```csharp
_ => throw new UnreachableException(),
```

`Match` needs neither. It takes exactly two branches, both required, and returns a value
with no warning to suppress.

**So use `Match` when you want a value out of both cases**, which is most of the time.
**Reach for a positional pattern when you are in statement position** (an `if` that
guards a block, a `switch` statement, or a `when` clause), where `Match` would mean
wrapping statements in a lambda that returns nothing.

{% hint style="info" %}
**A positional pattern does not trip `WM2021`.** That rule reports a *property* pattern
reading `IsSome`, `IsNone`, `IsOk` or `IsErr` — a state check written so that nothing
recognises it as one. A positional pattern reads none of those properties, so it is
outside the rule entirely. See
[`WM2021`](analyzer-rules.md#wm2021).
{% endhint %}

### Unwrap

Use `Unwrap` to consume the monadic wrapper when you are certain the monadic wrapper holds a value, or if you want to fail loudly if it doesn't.

{% hint style="info" %}
Avoid `Unwrap` unless you've validated the presence of a value upstream. It's an intentional point of failure, like `First` on an empty sequence. In most cases, you should reach for [#match](core-functionality.md#match "mention").
{% endhint %}

{% tabs %}
{% tab title="Option" %}
```csharp
Option<string> maybeName = Option.Some("Lorekeeper");
string name = maybeName.Unwrap();
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Danny");
string name = nameResult.Unwrap();
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
An `UnwrapException` will be thrown when the value is absent or failure.
{% endhint %}

### UnwrapOr

An alternative to [#unwrap](core-functionality.md#unwrap "mention"), use `UnwrapOr` to consume the monadic wrapper when you have a fallback value prepared for the `None` or `Err` states.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<string> maybeNickname = Option.None<string>();
string nickname = maybeNickname.UnwrapOr("Lautna");
//     ^? "Mate"
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<string, Error> nameResult = 
    Result.Err<string, Error>(
        new Error(ErrorCodes.MissingName, "no name was supplied"));

string name = nameResult.UnwrapOr("Unknown");
//     ^? "Unknown"
```
{% endtab %}
{% endtabs %}

### UnwrapOrElse

An alternative to [#unwrapor](core-functionality.md#unwrapor "mention"), use `UnwrapOrElse` to consume the monadic wrapper when you have a fallback value that requires some expensive computation.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<Uri> maybeAvatar = Option.None<Uri>();
Uri avatar = maybeAvatar.UnwrapOrElse(() => GenerateAvatar());
//  ^? Generated Avatar
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<Config, Error> getConfigResult = GetConfig("Ashton");
//                    ^? Err<Config, Error>

Config config = getConfigResult.UnwrapOrElse(error => GenerateDefaultConfig());
//     ^? generated config
```
{% endtab %}
{% endtabs %}

### UnwrapOrDefault

An alternative to [#unwrap](core-functionality.md#unwrap "mention"), use `UnwrapOrDefault` to consume the monadic wrapper when you are ok with `default(T)` as the fallback value.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<string> maybeName = Option.None<string>();
string? name = maybeName.UnwrapOrDefault();
//      ^? null
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<int, string> numberResult = Result.Err<int, string>("Error");
int? number = numberResult.UnwrapOrDefault();
//   ^? 0
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
You can use `UnwrapOrDefault` to return to the world of nullable reference types. Note that reference types will have a default value of `null`, and value types like `int` will use their default value - e.g. `0`
{% endhint %}

{% hint style="warning" %}
On a value type that last part is the catch. The signature reads `T?`, but `T` is constrained to `notnull`, so the `?` is an annotation rather than a `Nullable<T>`. `UnwrapOrDefault` on an `Option<int>` hands you `0`, and nothing tells you whether that `0` came from a `Some` or from the absent case. Reach for `UnwrapOrNull` there instead. `WM2015` points this out for you.
{% endhint %}

### UnwrapOrNull

Use `UnwrapOrNull` when the value is a value type and you need the absent case to stay visible. It returns `T?`, a real `Nullable<T>`, so `null` means absent and every other value came from a `Some` or an `Ok`.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<int> maybeCount = Option.None<int>();
int? count = maybeCount.UnwrapOrNull();
//   ^? null, where UnwrapOrDefault would have given you 0
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<int, string> countResult = Result.Err<int, string>("Error");
int? count = countResult.UnwrapOrNull();
//   ^? null
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
`UnwrapOrNull` is constrained to `T : struct`, so it does not appear on an `Option<string>`. A reference type needs no equivalent: `UnwrapOrDefault` already hands you `null` for the absent case.
{% endhint %}

### Expect

A sibling to [#unwrap](core-functionality.md#unwrap "mention"), but lets you give a meaningful error message when an exception is thrown. Use `Expect` to consume the monadic wrapper when you expect it to be in a `Some` or `Ok` state, and you want to fail loudly if it isn't.

{% hint style="info" %}
This method is useful in scenarios where an absent value means a logic error or misuse of the API - not a runtime condition to recover from.
{% endhint %}

{% tabs %}
{% tab title="Option" %}
```csharp
Option<string> maybeName = Option.Some("Greymore");
string name = maybeName.Expect("Expected a name, but got nothing.");
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Pelor");
string name = nameResult.Expect("Expected a name, but got an error");
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
An `UnmetExpectationException` carrying the message you gave is thrown when the value is absent or a failure.
{% endhint %}

### More Consumes

There are consume methods specific to a `Result<T, E>`.

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th></tr></thead><tbody><tr><td><strong>Result Consumes</strong></td><td>Learn about the consume methods specific to the <code>Result&#x3C;T, E></code> monad.</td><td><a href="result-of-t-and-e.md#consume">#consume</a></td></tr></tbody></table>

## Transform and Consume

### MapOr

Use `MapOr` when you want to apply a transformation to the contained value, and you have a fallback value ready to go in case the monadic wrapper is a `None` or `Err`. If your fallback value requires expensive computation, reach for [#maporelse](core-functionality.md#maporelse "mention").

{% tabs %}
{% tab title="Option" %}
<pre class="language-csharp"><code class="lang-csharp">Option&#x3C;string> maybeName = Option.None&#x3C;string>();
int length = maybeName.MapOr(0, name => name.Length);
<strong>//  ^? 0
</strong></code></pre>
{% endtab %}

{% tab title="Result" %}
```csharp
Result<string, string> nameResult = Result.Err<string, string>("Error");

int length = nameResult.MapOr(0, name => name.Length);
//  ^? 0result
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Unlike [#map](core-functionality.md#map "mention"), which allows you to continue the monadic chain, `MapOr` will consume the monadic wrapper and return you the underlying value.
{% endhint %}

### MapOrElse

Use `MapOrElse` when you want to apply a transformation to the contained value, consume the monadic wrapper, and you need to perform an expensive calculation to get the fallback value.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<User> maybeUser = Option.None<User>();

Uri avatar = maybeUser.MapOrElse(
    () => GenerateAvatar(),
    user => user.Avatar
);
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<User, Error> getUserResult = GetUser("Changebringer");
Uri avatar = getUserResult.MapOrElse(
    error => GenerateAvatar(),
    user => user.Avatar
);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Unlike [#map](core-functionality.md#map "mention"), which allows you to continue the monadic chain, `MapOrElse` will consume the monadic wrapper and return you the underlying value.
{% endhint %}

### MapOrDefault

Use `MapOrDefault` when you want to transform the contained value and `default(TOut)` is a fine answer for the absent case. It saves you writing the fallback that [#mapor](core-functionality.md#mapor "mention") asks for.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<string> maybeName = Option.None<string>();
int length = maybeName.MapOrDefault(name => name.Length);
//  ^? 0
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<string, string> nameResult = Result.Err<string, string>("Error");
int length = nameResult.MapOrDefault(name => name.Length);
//  ^? 0
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
The signature reads `TOut?`, but `TOut` is constrained to `notnull`, so on a value type that `?` is an annotation and not a `Nullable<TOut>`. Map to an `int` and the absent case gives you `0`, not `null`. That is what `MapOrNull` is for.
{% endhint %}

### MapOrNull

Use `MapOrNull` when the transformation produces a value type and you need the absent case to stay visible, for the same reason [#unwrapornull](core-functionality.md#unwrapornull "mention") exists.

```csharp
Option<string> maybeName = Option.None<string>();
int? length = maybeName.MapOrNull(name => name.Length);
//   ^? null, where MapOrDefault would have given you 0
```

{% hint style="info" %}
`MapOrNull` constrains its result to `TOut : struct`. Map to a reference type and `MapOrDefault` already gives you `null`.
{% endhint %}

## Side-Effect

Side effects allow you to conditionally access the value when it is `Some` or `Ok` and run some logic against the value without having to handle the other branch.

### Inspect

Use `Inspect` when you want to run some logic against the value inside the monadic wrapper when it is in it's `Some` or `Ok` state without transforming the value inside. The most common use case for `Inspect` is to inspect the value inside the wrapper and log it's value.

{% hint style="info" %}
Reach for [#map](core-functionality.md#map "mention") instead if you need to transform the contained value
{% endhint %}

{% tabs %}
{% tab title="Option" %}
```csharp
Option<string> maybeName = Option.Some("Geladon");
maybeName.Inspect(name => Console.WriteLine(name.Length));
```
{% endtab %}

{% tab title="Result" %}
```csharp
Result<string, string> nameResult = Result.Ok<string, string>("Percival");
nameResult.Inspect(name => Console.WriteLine(name.Length));
```
{% endtab %}
{% endtabs %}

### More Side-Effects

There are side-effects specific to a `Result<T, E>`.

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th></tr></thead><tbody><tr><td><strong>Result Side-Effects</strong></td><td>Learn about the side-effect methods specific to the <code>Result&#x3C;T, E></code> monad.</td><td><a href="result-of-t-and-e.md#consume">#consume</a></td></tr></tbody></table>

## Nesting

You will inevitably find yourself in a situation where you have a monadic wrapper nested inside of a monadic wrapper. Use the below functions to remove a level of nesting at a time.

### Flatten

Sometimes you will find yourself with an `Option` inside an `Option` or a `Result` inside of a `Result`. Use `Flatten` to remove a level of nesting in the monadic wrapper.

{% tabs %}
{% tab title="Option" %}
Removes one level of nesting from an `Option<Option<T>>`

```csharp
Option<Option<string>> some = Option.Some(Option.Some("Chetney"));
Option<string> result = some.Flatten();
```
{% endtab %}

{% tab title="Result" %}
Removes one level of nesting from an `Result<Result<T, E>, E>`&#x20;

```csharp
Result<int, string> DoWork(string source);
Result<string, string> start = Result.Ok<string, string>("Storm Weaver");
Result<Result<int, string>, string> output = start.Map(x => DoWork(x));
Result<int, string> flattened = output.Flatten();
```
{% endtab %}
{% endtabs %}

### Transpose

Sometimes you will find yourself with a `Result` inside an `Option` or an `Option` inside of a `Result` . Use `Transpose` to convert between them when your business logic needs it.

{% tabs %}
{% tab title="Option<Result<T, E>>" %}
```csharp
Result<int, string> Divide(int a, int b);
Option<int> maybeNumber = Option.Try(() => GetNumber());

Option<Result<int, string>> maybeResult = maybeNumber
    .Map(number => Divide(number, 2));
    
Result<Option<int>, string> result = maybeResult.Transpose();
```

Calling `Transpose` in this situation declares that a `Result` of `None<int>` is a valid outcome in our business rules.
{% endtab %}

{% tab title="Result<Option<T>, E>" %}
```csharp
class TaxCalculator
{
    Option<decimal> GetTax(decimal amount);
}

Result<TaxCalculator, string> CreateCalculator(string country);



Result<Option<decimal>, string> calculationResult = 
    CreateCalculator(Currency.Aus)
        .Map(calculator => calculator.GetTax(100.00m));
        
Option<Result<decimal, string>> maybeTax = calculationResult.Transpose();
```

Calling `Transpose` in this scenario declares that the absence of a tax amount is valid in our business rules.
{% endtab %}
{% endtabs %}

## Conversion

There may come a time where you have an `Option<T>` but you need a `Result<T, E>`. An `Option<T>` can be converted into a `Result<T, E>` and vice versa.

{% tabs %}
{% tab title="Option to Result" %}
### OkOr

Converts the `Option<T>` into a `Result<T, E>`, transforming a `Some` into an `Ok` and a `None` into an `Err`.&#x20;

{% hint style="info" %}
The error value passed into this method must be eagerly evaluated. If it is the result of a function call, it is recommended to use `OkOrElse` instead.
{% endhint %}

```csharp
Option<int> some = Option.Some(1);
Option<int> none = Option.None<int>();
Error error = new("ER1", "Missing number.");

Result<int, Error> ok = some.OkOr(error);
//                 ^? Ok(1)

Result<int, Error> err = none.OkOr(error);
//                 ^? Err(error)
```

### OkOrElse

Converts the `Option<T>` into a `Result<T, E>`, transforming a `Some` into an `Ok` and a `None` into an `Err`.  The error value is evaluated only when the `Option<T>` is a `None`.

```csharp
Option<int> some = Option.Some(1);
Option<int> none = Option.None<int>();

Result<int, string> ok = some.OkOrElse(() => "Missing number");
//                 ^? Ok(1)

Result<int, string> err = none.OkOrElse(() => "Missing number");
//                 ^? Err("Missing number")
```
{% endtab %}

{% tab title="Result to Option" %}
### GetOk

Converts the `Result<T, E>` into an `Option<T>`. Returns a `Some` if the `Result` is an `Ok`, otherwise returns a `None`.

```csharp
Result<int, string> ok = Result.Ok<int, string>(1);
Result<int, string> err = Result.Err<int, string>("Error");

Option<int> some = ok.GetOk();
//          ^? Some(1)

Option<int> none = err.GetOk();
//          ^? None()
```

### GetErr

Converts the `Result<T, E>` into an `Option<T>`. Returns a `Some` if the `Result` is an `Err`, otherwise returns a `None`.

```csharp
Result<int, string> ok = Result.Ok<int, string>(1);
Result<int, string> err = Result.Err<int, string>("Error");

Option<string> none = ok.GetErr();
//             ^? None()

Option<string> some = err.GetErr();
//             ^? Some("Error")
```
{% endtab %}
{% endtabs %}
