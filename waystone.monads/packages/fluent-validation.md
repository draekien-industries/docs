---
description: Run a FluentValidation validator and get a Result back, with the failures attached to the error.
---

# FluentValidation

`Waystone.Monads.FluentValidation` — a validator that returns a `Result`.

## What it adds

Two extension methods, `Validate` and `ValidateAsync`, on any value. Each takes an
`IValidator<T>` you already wrote and hands back a `Result<TValue, Error>`.

```csharp
using FluentValidation;
using FluentValidation.Extensions;

record UserInput(int Range, string Search);

class UserInputValidator : AbstractValidator<UserInput>
{
    public UserInputValidator()
    {
        RuleFor(x => x.Range).GreaterThan(0);
        RuleFor(x => x.Search).NotEmpty();
    }
}

UserInput input = new(1, "bob");

Result<UserInput, Error> result = input.Validate(new UserInputValidator());
```

A value that passes comes back as an `Ok` holding the value you gave it. A value that
fails comes back as an `Err` holding a [`ValidationError`](#validationerror).

The async form is the same shape, and takes a cancellation token:

```csharp
Result<UserInput, Error> result =
    await input.ValidateAsync(new UserInputValidator(), cancellationToken);
```

## When to reach for it

Reach for it where a validation failure is an ordinary outcome your caller handles,
and you are already writing FluentValidation validators. `Validate` gives you a
`Result` that chains with everything else in the library.

Skip it if you throw on invalid input by design, or if you do not use
FluentValidation. Nothing else here depends on it.

## Install it

```
dotnet add package Waystone.Monads.FluentValidation
```

You also need `FluentValidation` itself, which you almost certainly already have —
you have to write the validator.

The package supports `FluentValidation >= 11.1.0 && < 13.0.0`. Bring your own version
inside that range.

{% hint style="info" %}
FluentValidation 12 targets `net8.0` only. If you build for .NET Framework or
`netstandard2.0`, you cannot resolve it. That is FluentValidation's constraint, not
this package's — stay on 11.x there.
{% endhint %}

## Where the types live

The package shadows FluentValidation's own namespaces. Its types sit beside
`IValidator` and `ValidationFailure` rather than under a parallel `Waystone` tree, so
the validator file you already wrote usually needs no new `using` at all.

| Member | Namespace |
| --- | --- |
| `ValidationError` | `FluentValidation` |
| `Validate`, `ValidateAsync` | `FluentValidation.Extensions` |
| `UseValidationErrorCode` | `FluentValidation.Configs` |

The package and assembly are still called `Waystone.Monads.FluentValidation`. Only the
namespaces shadow.

Before `7.0.0` these lived under `Waystone.Monads.FluentValidation.*`. Every type and
member name is unchanged — only the `using` directives move. See
[Every v7 break](../upgrading/v7/breaking-changes.md#waystone-monads-fluentvalidation).

## It errs with `Error`, so it chains

This is the point of the package. `Validate` errs with `Error`, the same type the rest
of `Waystone.Monads` uses, so a validation step drops into a chain without a conversion
at the seam.

```csharp
Result<UserInput, Error> result = input.Validate(new UserInputValidator())
                                       .AndThen(Normalise);
```

The async form composes the same way:

```csharp
Result<UserInput, Error> result =
    await input.ValidateAsync(new UserInputValidator(), cancellationToken)
               .AndThenAsync(Save);
```

`ValidateAsync` returns a `ValueTask`, which is what lets a chain ending in it be passed
as a step to `AndThenAsync`. See [Async](../guides/async.md).

## `ValidationError`

`ValidationError` is a `sealed record` that derives from `Error`. It is an error, not
something you convert into one.

| Member | What it gives you |
| --- | --- |
| `Code` | The configured validation error code. Default `validation.failed` |
| `Message` | Every failure message joined with `"; "` |
| `Failures` | The `ValidationFailure` list the validator reported, never empty |
| `ToDictionary()` | Those messages grouped by property name |

You reach the detail by pattern matching, at the one place you need it:

```csharp
if (error is ValidationError validationError)
{
    return ValidationProblem(validationError.ToDictionary());
}
```

`ToDictionary()` builds a fresh dictionary each call, so hold the result if you need it
twice.

### You cannot build an empty one

The constructor is `internal`, and only the failure branch inside `Validate` and
`ValidateAsync` reaches it. So a `ValidationError` always carries at least one failure,
and there is no "no failures" case for you to handle.

### Two of them compare on code and message

`Failures` takes no part in equality. `Message` is rendered from it, so comparing both
would only add reference equality over a list — two errors describing the same failures
would come out unequal.

A `ValidationError` never equals a plain `Error`, even with the same code and message.
Records compare their type as well as their values.

## Configure the error code

Configure this through `MonadOptions`, alongside the core settings. There is no separate
options class.

```csharp
using FluentValidation.Configs;
using Waystone.Monads.Configs;

MonadOptions.Configure(options => options.UseValidationErrorCode("input.invalid"));
```

The default is `validation.failed`. The code cannot be null or whitespace — passing
either throws an `ArgumentException`.

`UseValidationErrorCode` returns `MonadValidationOptionsBuilder`, not
`MonadOptionsBuilder`, so set the core options first if you need both:

```csharp
MonadOptions.Configure(options =>
{
    options.UseFallbackErrorCode("Contoso");
    options.UseValidationErrorCode("input.invalid");
});
```

### Scopes work, and the code is read once

Validation options honour `MonadOptions.BeginScope`. One scope covers this package and
the core together.

```csharp
using (MonadOptions.BeginScope(options => options.UseValidationErrorCode("debug.validation")))
{
    // errors created in here carry "debug.validation"
}
```

The code is read **when the validation runs**, not when you later read the error. An
error created inside a scope keeps that scope's code after the scope closes. See
[Configuration](../guides/configuration.md) for the full scope semantics.

## Exceptions still throw

Only validation failures become an `Err`.

- An exception thrown by your validator propagates to the caller.
- `Validate` throws if the validator declares asynchronous rules. Call `ValidateAsync`
  for those.
- A cancelled token surfaces as an `OperationCanceledException`, not as an `Err`.

## It changed a lot in 7.0.0

If you are coming from `6.x`, this package no longer has a `ValidationErr` type and the
extension methods return something different. See
[Every v7 break](../upgrading/v7/breaking-changes.md#waystone-monads-fluentvalidation).

## What it does not do

* It does not turn exceptions into an `Err`. Only validation failures become one.
* It does not run asynchronous rules from `Validate`. That call throws; use
  `ValidateAsync`.
* It does not register your validators. That is FluentValidation's own container
  wiring, unchanged.
