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

Ready to eliminate `null` checks and stop catching exceptions? Here's how to get up and running with Waystone.Monads.

## Installation

Install via the dotnet CLI:

```sh
dotnet add package Waystone.Monads
```

## Using Option\<T>

The `Option<T>` type represents a value that may or may not be present. It eliminates the ambiguity and risks of `null` , and provides a way to mark a value as intentionally absent.

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

The `Result<T, E>` type represents a success or failure as a value instead of throwing exceptions. It allows you to reserve exceptions for exceptional scenarios, and provides a framework for handling success and failure outcomes explicitly. Use it instead of throwing exceptions for recoverable failures.

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
