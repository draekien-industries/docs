# Errors and Exceptions

## Built in Error Types

Waystone.Monads provides a set of built in error types for convenience in the case you do not wish to write your own.

```csharp
record ErrorCode(string Value);
record Error(ErrorCode ErrorCode, string Message);
```

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

You may want to use enums to define and organise your error codes, and then create `ErrorCode` instances during runtime from these enums. A factory method has been provided to facilitate this approach.

```csharp
public enum InputErrors
{
    Missing = 1,
    Malformed = 2,
    OutOfRange = 3
}

public enum RegexErrors // etc.

var errorCode = ErrorCode.FromEnum(InputErrors.Missing); // "InputErrors.Missing"
```

If you want to customise the error code that is generated from your enum, you can provide your own instance of `IErrorCodeFormatter<TEnum>` to the `FromEnum` method.

```csharp
public class CustomEnumFormatter<T> : IErrorCodeFormatter<T> where T : Enum
{
    public string Format(T value) => $"err.{value}".ToLower();
}

var errorCode = ErrorCode.FromEnum(InputErrors.Missing, new CustomEnumFormatter<InputErrors>());
//  ^? "err.missing"
```

#### Error Code from Exception

You may want to use the exception type itself as the source of your error codes when they are caught during runtime. A factory method has been provided to facilitate this approach.

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
    var errorCode = ErrorCode.FromException(e); // "Err.Sql"
}
```

If you want to customise the error code that is generated from the `Exception`, you can provide your own instance of `IErrorCodeFormatter<TException>` to the `FromException` method.

```csharp
public class CustomExceptionFormatter<T> : IErrorCodeFormatter<T> where T : Exception
{
    public string Format(T value) => $"{nameof(T)}".ToLower();
}

var errorCode = ErrorCode.FromException(new SqlException(), new CustomExceptionFormatter<SqlException>());
//  ^? "sqlexception"
```

### Error

The `Error` type captures an instance of an error associated with a specific `ErrorCode`. Use it to provide human-readable information about the error instance that occurred during runtime.

{% hint style="info" %}
You must define an `ErrorCode` before creating an `Error`
{% endhint %}

```csharp
Error error1 = new(ErrorCodes.InputMalformed, "Expected an absolute URI but received a relative URI");
Error error1 = new(ErrorCodes.InputMalformed, "Failed to parse input as a number");
```

#### Error from Exception

You may want to create errors on the fly when catching exceptions without having to first define your error code. A factory method has been provided to facilitate this approach. It uses [#error-code-from-exception](errors-and-exceptions.md#error-code-from-exception "mention") under the hood to generate the `ErrorCode`.

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
    var error = ErrorCode.FromException(e);
    //  ^? ErrorCode: "Err.Sql", Message: e.Message
}
```

If you want to customise the error code that is generated from the `Exception`, you can provide your own instance of `IErrorCodeFormatter<TException>` to the `FromException` method. See [#error-code-from-exception](errors-and-exceptions.md#error-code-from-exception "mention") for an example.

## Custom Exceptions

This library contains some custom exceptions that describe certain scenarios.

### UnwrapException

An exception that is thrown when attempting to [#unwrap](core-functionality.md#unwrap "mention") an `Option<T>` or a `Result<T, E>` when they are in their `None` or `Err` states, or when attempting to [#unwraperr](result-of-t-and-e.md#unwraperr "mention") on a `Result<T, E>` when it is in it's `Ok` state.

{% hint style="info" %}
Always check the monad's state before performing an `Unwrap` or `UnwrapErr`  to avoid encountering this exception.
{% endhint %}

### UnmetExpectationException

An exception that is thrown when when invoking [#expect](core-functionality.md#expect "mention") on an `Option<T>` or a `Result<T, E>` when they are in their `None` or `Err` states, or when invoking [#expecterr](result-of-t-and-e.md#expecterr "mention") on a `Result<T, E>` when it is in it's `Ok` state.
