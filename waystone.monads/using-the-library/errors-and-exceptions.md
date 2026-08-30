# Errors and Exceptions

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

## Built in Error Types

Waystone.Monads gives you a set of built in error types, for when you do not want to write your own.

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

Two things to note. `Error` names its code property `Code`, not `ErrorCode` — read
it as `error.Code.Value`. And both types declare their constructor explicitly
rather than positionally, so you cannot deconstruct them and you cannot use `with`
to change a property. Build a new instance instead.

### ErrorCode

The `ErrorCode` type is a short code representing an error type in the application. These codes should not change between occurrence to occurrence of the same error type. It is recommended to predefine your error codes.

#### Static Error Codes (recommended)

The simplest way of defining your error codes is to create a static `ErrorCodes` class that contains the list of error codes that may be encountered during the lifecycle of your application. This class would normally live inside your domain layer if you are following domain driven design.

```csharp
public static class ErrorCodes
{
    public static readonly ErrorCode InputMissing = new("input.missing");
    public static readonly ErrorCode InputMalformed = new("input.malformed");
    public static readonly ErrorCode InputOutOfRange = new("input.out_of_range");
}
```

#### Error Code from Enum

You may want to use enums to define and organise your error codes, and then create `ErrorCode` instances during runtime from these enums. A factory method does this for you.

```csharp
[ErrorCodeCatalog]
public enum InputErrors
{
    Missing = 1,
    Malformed = 2,
    OutOfRange = 3
}

[ErrorCodeCatalog]
public enum RegexErrors // etc.
```

```diff
-var errorCode = ErrorCode.FromEnum(InputErrors.Missing); // "InputErrors.Missing"
+var errorCode = InputErrorsCatalog.Codes.Missing;        // "InputErrors.Missing"
```

{% hint style="danger" %}
**`ErrorCode.FromEnum` was obsolete from 6.2.0 and 7.0.0 removes it.** So is
overriding `ErrorCodeFactory.FromEnum` to shape what it returns. Mark the enum with
`[ErrorCodeCatalog]` instead and use the members that generates —
`OrderErrorCatalog.Codes.NotFound` where you can name the member, or the
generated `ToErrorCode()` extension where you cannot. See
[deprecations.md](../upgrading-and-deprecations/deprecations.md "mention") for the migration and
[generated-error-codes.md](generated-error-codes.md "mention") for the `Format` that
replaces a factory override. `FromException` is unaffected.
{% endhint %}

#### Error Code from Exception

You may want to use the exception type itself as the source of your error codes when they are caught during runtime. A factory method does this for you.

The code is the exception's type name with a trailing `Exception` removed. There is
no prefix. The suffix match ignores case, and the exception's message is never
read, so nothing from its text reaches the code.

| Exception type | Resulting code |
| --- | --- |
| `SqlException` | `Sql` |
| `InvalidOperationException` | `InvalidOperation` |
| `TimeoutException` | `Timeout` |
| `Exception` | `Exception` |

`Exception` itself is the one special case: it keeps its whole name rather than
reducing to an empty code.

{% hint style="warning" %}
This approach may introduce inconsistencies in your error codes. It also does not work well if you are raising errors that are not caused by exceptions elsewhere in your application.
{% endhint %}

```csharp
try
{
    // do work
}
catch (SqlException e)
{
    var errorCode = ErrorCode.FromException(e); // "Sql"
}
```

{% hint style="info" %}
If you want to customise the error code that is generated from the `Exception`, you can supply your own instance of `ErrorCodeFactory` to the global `MonadOptions` and override the `FromException` method.
{% endhint %}

### Error

The `Error` type captures an instance of an error associated with a specific `ErrorCode`. Use it to give human-readable information about the error instance that occurred during runtime.

{% hint style="info" %}
You must define an `ErrorCode` before creating an `Error`
{% endhint %}

```csharp
Error error1 = new(ErrorCodes.InputMalformed, "Expected an absolute URI but received a relative URI");
Error error1 = new(ErrorCodes.InputMalformed, "Failed to parse input as a number");
```

#### Error from Enum

If your error codes come from enums, you can create the `Error` in one call. This
uses [#error-code-from-enum](errors-and-exceptions.md#error-code-from-enum "mention")
under the hood to generate the `ErrorCode`.

```csharp
[ErrorCodeCatalog]
public enum InputErrors
{
    Missing = 1,
    Malformed = 2,
    OutOfRange = 3
}

```

```diff
-Error error = Error.FromEnum(InputErrors.Malformed, "Failed to parse input as a number");
+Error error = InputErrorsCatalog.Errors.Malformed("Failed to parse input as a number");
//    ^? Code: "InputErrors.Malformed", Message: "Failed to parse input as a number"
```

{% hint style="danger" %}
**`Error.FromEnum` was obsolete from 6.3.0 and 7.0.0 removes it.** Mark the enum with
`[ErrorCodeCatalog]` and use the generated `Errors` factory, as above — or
`value.ToError(message)` where you only know the member at run time. See
[generated-error-codes.md](generated-error-codes.md "mention").
{% endhint %}

{% hint style="info" %}
The message is required here, unlike the code-only form, which takes just the enum
member. An `Error` always carries a human-readable message.
{% endhint %}

To create a `Result` directly from one of these, pass the generated error to
`Result.Err`:

```csharp
Result<int, Error> result = Result.Err<int>(
    InputErrorsCatalog.Errors.Malformed("Failed to parse input as a number"));
```

The `Result.Err<TOk>(enum, message)` overload that used to do this in one step was
removed in 7.0.0. See [#creation](core-functionality.md#creation "mention").

#### Error from Exception

You may want to create errors on the fly when catching exceptions without having to first define your error code. A factory method does this for you. It uses [#error-code-from-exception](errors-and-exceptions.md#error-code-from-exception "mention") under the hood to generate the `ErrorCode`.

{% hint style="warning" %}
This approach may introduce inconsistencies in your error codes. It also does not work well if you are raising errors that are not caused by exceptions elsewhere in your application.
{% endhint %}

```csharp
try
{
    // do work
}
catch (SqlException e)
{
    var error = Error.FromException(e);
    //  ^? Code: "Sql", Message: e.Message
}
```

If you want to customise the error code that is generated from the `Exception`, you can supply your own instance of `ErrorCodeFactory` to the global `MonadOptions` and override the `FromException` method. See [#error-code-from-exception](errors-and-exceptions.md#error-code-from-exception "mention") for an example.

## Custom Exceptions

This library contains some custom exceptions that describe certain scenarios.

### UnwrapException

An exception that is thrown when you try to [#unwrap](core-functionality.md#unwrap "mention") an `Option<T>` or a `Result<T, E>` when they are in their `None` or `Err` states, or when you try to [UnwrapErr](../guides/result.md#unwraperr-and-expecterr "mention") on a `Result<T, E>` when it is in it's `Ok` state.

{% hint style="info" %}
Always check the monad's state before performing an `Unwrap` or `UnwrapErr`  to avoid encountering this exception.
{% endhint %}

### UnmetExpectationException

An exception that is thrown when when invoking [#expect](core-functionality.md#expect "mention") on an `Option<T>` or a `Result<T, E>` when they are in their `None` or `Err` states, or when invoking [ExpectErr](../guides/result.md#unwraperr-and-expecterr "mention") on a `Result<T, E>` when it is in it's `Ok` state.
