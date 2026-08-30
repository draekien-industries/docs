---
description: Build the failure value you put inside an Err.
icon: triangle-exclamation
---

# Errors

You have decided a method returns `Result<T, E>`. Now you need something to put
in the `E`.

You can use any type you like — a `string`, your own record, an enum. This page is
about the two types the library ships for when you would rather not write your
own.

{% hint style="info" %}
Looking for what the library **throws** at you? That is
[Exceptions](exceptions.md).
{% endhint %}

## The two types

```csharp
public record ErrorCode
{
    public ErrorCode(string value);

    public string Value { get; }
}

public record Error
{
    public Error(ErrorCode code, string message);

    public ErrorCode Code { get; }
    public string Message { get; }
}
```

An `ErrorCode` is a short, stable identifier for a *kind* of failure. An `Error`
is one *occurrence* of it, with a message a human can read.

Two details that trip people up:

* `Error` names its code property `Code`, not `ErrorCode`. Read it as
  `error.Code.Value`.
* Both types declare their constructor explicitly rather than positionally. So you
  cannot deconstruct them, and you cannot use `with` to change a property. Build a
  new instance instead.

## Define your error codes

An error code should not change between one occurrence of a failure and the next.
That means defining them up front, not building them at the call site.

### As static fields (recommended)

The simplest thing that works. A static class holding every code your application
can produce — in your domain layer, if you are following domain-driven design.

```csharp
public static class SpellErrorCodes
{
    public static readonly ErrorCode ComponentMissing = new("component.missing");
    public static readonly ErrorCode SigilMalformed = new("sigil.malformed");
    public static readonly ErrorCode LevelOutOfRange = new("level.out_of_range");
}
```

Start here. If you outgrow it, the next section is where to go.

### From an enum

If you would rather group codes as an enum, mark it `[ErrorCodeCatalog]` and the
source generator writes the codes for you:

```csharp
[ErrorCodeCatalog]
public enum SpellErrors
{
    ComponentMissing = 1,
    SigilMalformed = 2,
    LevelOutOfRange = 3,
}
```

```csharp
ErrorCode code = SpellErrorsCatalog.Codes.SigilMalformed;
//        ^? "SpellErrors.SigilMalformed"
```

This is a generated API with more to it than one line — the naming format, the
`ToErrorCode()` extension for when you only know the member at run time, the
`using` it needs. See
[Generated error codes](../using-the-library/generated-error-codes.md).

{% hint style="danger" %}
**`ErrorCode.FromEnum` was obsolete from 6.2.0 and 7.0.0 removed it.** So is
overriding `ErrorCodeFactory.FromEnum` to shape what it returns. Mark the enum
with `[ErrorCodeCatalog]` and use the generated members instead. See
[deprecations.md](../upgrading-and-deprecations/deprecations.md "mention") for the
migration. `FromException` is unaffected.
{% endhint %}

## Build an error

```csharp
Error malformedSigil = new(
    SpellErrorCodes.SigilMalformed,
    "Expected an absolute sigil but received a relative one");

Error unparseable = new(
    SpellErrorCodes.SigilMalformed,
    "Failed to parse the sigil as a rune sequence");
```

Two occurrences, one code. That is the split working as intended: the code is what
you branch on and log against, the message is what you show a person.

{% hint style="info" %}
You need an `ErrorCode` before you can build an `Error`. There is no message-only
constructor.
{% endhint %}

### From a catalog enum

If your codes come from a `[ErrorCodeCatalog]` enum, the generated `Errors`
factory builds the whole `Error` in one call:

```csharp
Error error = SpellErrorsCatalog.Errors.SigilMalformed(
    "Failed to parse the sigil as a rune sequence");
//    ^? Code: "SpellErrors.SigilMalformed", Message: "Failed to parse the sigil…"
```

The message is required here, unlike the code-only form. An `Error` always carries
one.

To turn that straight into a `Result`, pass it to `Result.Err`:

```csharp
Result<int, Error> result = Result.Err<int>(
    SpellErrorsCatalog.Errors.SigilMalformed(
        "Failed to parse the sigil as a rune sequence"));
```

{% hint style="danger" %}
**`Error.FromEnum` was obsolete from 6.3.0 and 7.0.0 removed it**, along with the
`Result.Err<TOk>(enum, message)` overload that used to collapse the two calls into
one. Use the generated factory as above, or `value.ToError(message)` where you only
know the member at run time. See
[Generated error codes](../using-the-library/generated-error-codes.md).
{% endhint %}

## From an exception you caught

Sometimes you are at the boundary — you caught something, and you want it as a
`Result` rather than a rethrow. Both types convert.

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

## Where to go next

* [Exceptions](exceptions.md) — what the library throws, and when.
* [Generated error codes](../using-the-library/generated-error-codes.md) — the `[ErrorCodeCatalog]` surface in full.
* [Result\<T, E>](result.md) — putting the error to work.
