---
description: Install the package and get both types working, without leaving this page.
icon: bullseye-arrow
layout:
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Quickstart

Ready to stop writing `null` checks and stop catching exceptions you expected?
Here is the whole setup.

## Install

```sh
dotnet add package Waystone.Monads
```

That is the only package you need. The analyzer and the source generator come
with it, already switched on.

## Add the usings

**`using Waystone.Monads;` on its own gets you nothing.** The root namespace
holds no types, so that line compiles and then every type name fails with
`CS0234`. Pick the namespaces you need instead:

| Namespace | What it holds |
| --- | --- |
| `Waystone.Monads.Options` | `Option<T>` and the static `Option` factories |
| `Waystone.Monads.Options.Extensions` | Extension methods on `Option<T>`: `Flatten`, `Transpose`, `Unzip`, `UnwrapOrNull`, the collection helpers, and every `Async` overload |
| `Waystone.Monads.Results` | `Result<TOk, TErr>` and the static `Result` factories |
| `Waystone.Monads.Results.Extensions` | Extension methods on `Result<TOk, TErr>`: `Flatten`, `Transpose`, `UnwrapOrNull`, the collection helpers, and every `Async` overload |
| `Waystone.Monads.Results.Errors` | `Error`, `ErrorCode` and `[ErrorCodeCatalog]` |
| `Waystone.Monads.Exceptions` | `UnwrapException` and `UnmetExpectationException` |
| `Waystone.Monads.Configs` | `MonadOptions` and `ErrorCodeFactory` |

{% hint style="warning" %}
**The two `Extensions` namespaces are not just async.** They hold synchronous
methods too. Import only `Waystone.Monads.Options` and a call to `UnwrapOrNull`
fails with `CS1061`, even though the method exists.
{% endhint %}

On C# 10 or later, put the ones you use everywhere in a single `GlobalUsings.cs`
and stop repeating them per file:

```csharp
global using Waystone.Monads.Options;
global using Waystone.Monads.Options.Extensions;
global using Waystone.Monads.Results;
global using Waystone.Monads.Results.Extensions;
global using Waystone.Monads.Results.Errors;
```

{% hint style="warning" %}
**Generated catalog members need a using for your own namespace.** When you mark an
enum with `[ErrorCodeCatalog]`, the generated `{EnumName}Catalog` class and the
`ToError`, `ToErrorCode` and `ToErrorCodeName` extensions are emitted into the enum's
own namespace, not into a Waystone one. Fully qualifying the enum at the call site
is not enough — the extensions still need `using` for the namespace the enum lives
in. See [generated-error-codes.md](../using-the-library/generated-error-codes.md "mention").
{% endhint %}

## Your first Option\<T>

`Option<T>` says a value might not be there. Instead of returning `null` and
hoping the caller checks, you return a type that cannot be read without handling
both cases.

```csharp
Option<string> patron = Option.Some("The Raven Queen");
Option<string> noPatron = Option.None<string>();

string vow = patron.Match(
    some => $"You are sworn to {some}.",
    () => "You are sworn to no one.");
// "You are sworn to The Raven Queen."

string silence = noPatron.Match(
    some => $"You are sworn to {some}.",
    () => "You are sworn to no one.");
// "You are sworn to no one."
```

`Match` is the way out. It takes one function for each case and returns a plain
value, so there is no state left to forget about.

## Your first Result\<T, E>

`Result<T, E>` says the work might fail. The failure is a value you return, not
an exception you throw, so the caller sees it in the signature.

```csharp
Result<int, string> RollDie(string sides)
{
    return int.TryParse(sides, out int faces) && faces > 0
        ? Result.Ok<int, string>(Random.Shared.Next(1, faces + 1))
        : Result.Err<int, string>($"'{sides}' is not a number of sides.");
}

Result<int, string> roll = RollDie("20");

int value = roll.Match(
    ok => ok,
    err => 0);
// somewhere between 1 and 20
```

Same shape as before. One function for success, one for failure, a plain value
out the other end.

{% hint style="success" %}
No exceptions. No `try`/`catch`. Both outcomes are visible in the return type.
{% endhint %}

## Where to go next

* [Why monads](why-monads.md) — the case for doing it this way at all.
* [Option\<T>](../guides/option.md) and [Results](../core-concepts/results.md) — working with each type properly.
* [Agent skills](agent-skills.md) — teach your coding agent the same habits.
