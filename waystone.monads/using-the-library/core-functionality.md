---
description: >-
  Learn about the different functionality that is common to both the Option and
  Result types.
---

# Core Functionality

## Introduction

While `Option<T>` and `Result<T, E>` serve different purposes (absence vs. success/failure), they share a common operational model. If you are fluent with either the `Option<T>` or `Result<T, E>` already, you're 90% of the way to mastering the other.

The core APIs defined below work the same across both types, differing only in terms of semantics.

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
{% endtab %}
{% endtabs %}

Use `Try` to safely capture potentially exception-throwing logic inside a monadic wrapper. This is useful if you want to begin a monadic chain from a method you do not have control over.

{% tabs %}
{% tab title="Option" %}
```csharp
Option<User> maybeUser = Option.Try(() => GetCurrentUser());
```

If the `GetCurrentUser` call throws, the exception is caught and logged via your [configured exception logger](configuration.md), and you get back a `None<User>` instance.
{% endtab %}

{% tab title="Result" %}
```csharp
Result<User, string> result = Result.Try(
    onOk: () => GetCurrentUser(),
    onErr: ex => ex.Message
);
```

If the `GetCurrentUser` call throws, the exception is caught and logged via your configured exception logger, and the `onErr` delegate you provide is invoked.&#x20;

{% hint style="info" %}
The `onErr` delegate gives you a way to transform the caught exception into an error type of your choosing.
{% endhint %}
{% endtab %}
{% endtabs %}

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

There are additional transform methods specific to the `Option<T>` and `Result<T, E>` types.&#x20;

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th></tr></thead><tbody><tr><td><strong>Option Transforms</strong></td><td>Learn about the transform methods specific to the <code>Option&#x3C;T></code> monad.</td><td><a href="option-of-t/#transform">#transform</a></td></tr><tr><td><strong>Result Transforms</strong></td><td>Learn about the transform methods specific to the <code>Result&#x3C;T, E></code> monad.</td><td><a href="result-of-t-and-e.md#transform">#transform</a></td></tr></tbody></table>

## State Checks

{% hint style="info" %}
These are ideal for short-circuiting logic or quick guards, but avoid using them for full branching. Reach for [`Match`](core-functionality.md#match) when both branches matter.
{% endhint %}

{% tabs %}
{% tab title="Option" %}
Use `IsOk` and `IsErr` when you need to check the state of the `Result<T, E>` and don't need to access it's value just quite yet.

```csharp
Result<DateTime, Error> safeParseResult = SafeParse("2025-01-01");

safeParseResult.IsOk; // true
safeParseResult.IsErr; // false
```
{% endtab %}

{% tab title="Result" %}
Use `IsSome` and `IsNone` when you want to check the state of the monad and don't need to access it's value just quite yet.

```csharp
Option<string> maybeName = Option.Some("John");

maybeName.IsSome; // true
maybeName.IsNone; // false
```
{% endtab %}
{% endtabs %}

## Consume

You will eventually need to escape from a monadic wrapper in order to access to a concrete value. This is where consume methods come in.

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
    Result.Err<string, Error>(new Error(ErrorCodes.MissingName));

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

Config config = getConfigResult.UnwrapOrElse(() => GenerateDefaultConfig());
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

### Expect

A sibling to [#unwrap](core-functionality.md#unwrap "mention"), but allows you to provide a meaningful error message when an exception is thrown. Use `Expect` to consume the monadic wrapper when you expect it to be in a `Some` or `Ok` state, and you want to fail loudly if it isn't.

{% hint style="info" %}
This method is useful in scenarios where an absent value indicates a logic error or misuse of the API - not a runtime condition to recover from.
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
An `UnmetExpectationException` with your provided message will be thrown when the value is absent or failure.
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
    () => GenerateAvatar(),
    user => user.Avatar
);
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Unlike [#map](core-functionality.md#map "mention"), which allows you to continue the monadic chain, `MapOrElse` will consume the monadic wrapper and return you the underlying value.
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
Result<Result<int, string, string>> output= start.Map(x => DoWork(x));
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
