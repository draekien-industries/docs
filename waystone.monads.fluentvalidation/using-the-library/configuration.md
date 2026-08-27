---
description: >-
  Configure the error code and fallback message used when a validation failure
  becomes an Error.
---

# Configuration

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads.FluentValidation --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads.FluentValidation" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

Configure this library through `MonadOptions`, the same entry point you use for
[Waystone.Monads](https://app.gitbook.com/o/eVVl3k56v8DR211ydaly/s/nQgeZ1m9pTmEbKceyEkw/ "mention").
There is no separate options class to reach for.

```csharp
using Waystone.Monads.Configs;
using Waystone.Monads.FluentValidation.Configs;

MonadOptions.Configure(options => options
    .UseValidationErrorCode("validation.failed")
    .UseFallbackValidationErrorMessage("One or more validation errors occurred."));
```

Configure this once in your application's lifecycle.

## Validation error code

`UseValidationErrorCode` sets the `ErrorCode` used when
[#methods](validationerr.md#methods "mention") converts a `ValidationErr` into an
`Error`.

The default is `validation.failed`.

```csharp
MonadOptions.Configure(options => options.UseValidationErrorCode("input.invalid"));
```

The error code cannot be null or whitespace. Passing either throws an
`ArgumentException`.

## Fallback validation error message

`UseFallbackValidationErrorMessage` sets the message used when a validation
failure carries no message of its own.

The default is `One or more validation errors occurred.`

```csharp
MonadOptions.Configure(options => options
    .UseFallbackValidationErrorMessage("Please check the values you supplied."));
```

The message cannot be null or whitespace. Passing either throws an
`ArgumentException`.

## Chaining

Both methods return `MonadValidationOptionsBuilder`, so you can chain them
together:

```csharp
MonadOptions.Configure(options => options
    .UseValidationErrorCode("input.invalid")
    .UseFallbackValidationErrorMessage("Please check the values you supplied."));
```

They do not return `MonadOptionsBuilder`, so configure the core options first if
you need both:

```csharp
MonadOptions.Configure(options =>
{
    options.UseLogger(logger);
    options.UseValidationErrorCode("input.invalid");
});
```

{% hint style="info" %}
`UseLogger` comes from `Waystone.Monads.Extensions.Logging`. It replaces
`MonadOptions.UseExceptionLogger`, which 7.0.0 removed.
{% endhint %}

## Scoped configuration

Validation options honour `MonadOptions.BeginScope`. One scope covers both this
library and the core, so there is no second scope to open.

```csharp
using (MonadOptions.BeginScope(options => options.UseValidationErrorCode("debug.validation")))
{
    // ValidationErr.ToError() uses "debug.validation" in here
}

// outside the scope, your global configuration applies again
```

Use this when you want different validation settings for one request, one test, or
a block you are debugging, without changing the configuration for your whole
application.

Options you do not set inside the scope are inherited from the configuration in
effect when the scope opened.

{% hint style="info" %}
Scopes apply to the current asynchronous flow, so concurrent work each sees its
own options. See the Scoped Configuration section of the Waystone.Monads
[Configuration](https://app.gitbook.com/o/eVVl3k56v8DR211ydaly/s/nQgeZ1m9pTmEbKceyEkw/ "mention")
page for the full semantics.
{% endhint %}
