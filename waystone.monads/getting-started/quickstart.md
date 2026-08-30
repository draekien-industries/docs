---
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

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release, and this page uses API that only exists in `7.0.0`.

The API can still change before `7.0.0` is stable.
{% endhint %}

Ready to remove `null` checks and stop catching exceptions? Here's how to get up and running with Waystone.Monads.

## Installation

Install via the dotnet CLI:

```sh
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

## Adding the usings

**`using Waystone.Monads;` on its own gets you nothing.** The root namespace holds
no types, so that line compiles and then every type name fails with `CS0234`. Pick
the namespaces you need instead:

| Namespace | What it holds |
| --- | --- |
| `Waystone.Monads.Options` | `Option<T>` and the static `Option` factories |
| `Waystone.Monads.Options.Extensions` | The `Async` extensions for `Option<T>` |
| `Waystone.Monads.Results` | `Result<TOk, TErr>` and the static `Result` factories |
| `Waystone.Monads.Results.Extensions` | The `Async` extensions for `Result<TOk, TErr>` |
| `Waystone.Monads.Results.Errors` | `Error`, `ErrorCode` and `[ErrorCodeCatalog]` |
| `Waystone.Monads.Exceptions` | `UnwrapException` and `UnmetExpectationException` |
| `Waystone.Monads.Configs` | `MonadOptions` and `ErrorCodeFactory` |

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

## Using Option\<T>

The `Option<T>` type represents a value that may or may not be present. It removes the ambiguity and risk of `null`, and gives you a way to mark a value as intentionally absent.

```csharp
Option<string> name = Option.Some("Liam O'Brian");
Option<string> missing = Option.None<string>();

string greeting = name.Match(
    some => $"Hello, {some}!",
    () => "Hello, stranger!"
);

// Output: "Hello, Liam O'Brian!"
```

{% hint style="info" %}
Use [#map](../using-the-library/core-functionality.md#map "mention"), [#inspect](../using-the-library/core-functionality.md#inspect "mention"), [#match](../using-the-library/core-functionality.md#match "mention"), and more to work with values safely and fluently
{% endhint %}

## Using Result\<T, E>

The `Result<T, E>` type represents a success or failure as a value instead of throwing exceptions. It lets you reserve exceptions for exceptional cases, and gives you a way to handle success and failure explicitly. Use it instead of throwing exceptions for recoverable failures.

```csharp
Result<int, string> ParseInt(string input)
{
    return int.TryParse(input, out var value)
        ? Result.Ok<int, string>(value)
        : Result.Err<int, string>($"Input '{input}' is not a valid number");
}

var result = ParseInt("42");

int value = result.Match(
    ok => ok,
    err => -1
);

// Output: 42
```

{% hint style="success" %}
No exceptions, no `try/catch`, just predictable control flow
{% endhint %}
