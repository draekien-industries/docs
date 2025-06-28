# Validating Values

This library provides extension methods for validating values using [FluentValidation](https://docs.fluentvalidation.net/en/latest/index.html) and encapsulating the result in a functional [Result](https://app.gitbook.com/s/nQgeZ1m9pTmEbKceyEkw/core-concepts/results) type. These methods allow you to easily perform synchronous or asynchronous validation of values, returning either the validated value or details about the validation failures.

{% hint style="info" %}
It is recommended to familiarise yourself with [Waystone.Monads](https://app.gitbook.com/o/eVVl3k56v8DR211ydaly/s/nQgeZ1m9pTmEbKceyEkw/ "mention") before continuing.
{% endhint %}

## Overview

These extensions enhance the usability of [FluentValidation](https://docs.fluentvalidation.net/en/latest/index.html) by integrating it with the [Result\<T, E>](https://app.gitbook.com/s/nQgeZ1m9pTmEbKceyEkw/result-less-than-t-e-greater-than "mention") type from the [Waystone.Monads](https://app.gitbook.com/o/eVVl3k56v8DR211ydaly/s/nQgeZ1m9pTmEbKceyEkw/ "mention") library. Instead of throwing exceptions or dealing with error lists, you receive a clear result indicating success or failure, making error handling more straightforward and composable.

## Methods

{% hint style="info" %}
The methods in this library are designed to promote functional error handling patterns when using FluentValidation
{% endhint %}

### Validate

The `Validate` extension method can be invoked on any value to synchronously validate the value using a provided `AbstractValidator`.&#x20;

```csharp
record Hello(string value);
class HelloValidator : AbstractValidator<Hello>
{
    public HelloValidator()
    {
         RuleFor(x => x.value).NotEmpty();   
    }
}

Hello hello = new("world!");
HelloValidator validator = new();
Result<Hello, ValidationErr> result = hello.Validate(validator);
```

If the validation succeeds, then an `Ok` will be returned containing the original value. Otherwise, an `Err` containing a [validationerr.md](validationerr.md "mention") will be returned.

### ValidateAsync

The `ValidateAsync` method can be invoked on any value to asynchronously validate the value using a provided `AbstractValidator`.&#x20;

```csharp
Hello hello = new("world!");
HelloValidator validator = new();
CancellationTokenSource cts = new();
Result<Hello, ValidationErr> result = await hello.ValidateAsync(validator, cts.Token);
```

If the validation succeeds, then an `Ok` will be returned containing the original value. Otherwise, an `Err` containing a [validationerr.md](validationerr.md "mention") will be returned.
