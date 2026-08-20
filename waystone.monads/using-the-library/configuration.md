# Configuration

This library contains some configurable behaviours. Configuration is available via the `Configure` method of the `MonadOptions` class. It is recommended to configure each option below _once_ in the lifecycle of your application.

If you need different settings for one part of your code, use a [scope](#scoped-configuration) instead of reconfiguring globally.

## Logging

There are several places in this library where exceptions are silently handled and transformed into non-throwing types. You can configure a custom logging action to inspect these exceptions as they are handled by the library.

```csharp
MonadOptions.Configure(options => options.UseExceptionLogger((ex) => {
    Console.WriteLine(ex); // replace with your logger's log method
}));
```

## Cancellation

`Option.Try`, `Option.TryAsync`, `Result.Try` and `Result.TryAsync` catch the
exceptions your factory throws and turn them into a `None` or an `Err`. From
6.0.0 they make one exception to that: an `OperationCanceledException` propagates
to your caller instead.

Cancelling is you telling the work to stop. It is not the work failing, and
turning it into a `None` leaves the caller unable to tell "cancelled" from
"genuinely absent".

`TaskCanceledException` inherits from `OperationCanceledException`, so it
propagates too.

If you need the pre-6.0.0 behaviour, opt back in:

```csharp
MonadOptions.Configure(options => options.UseCancellationAsFailure());
```

A cancellation is then caught, passed to your exception logger, and becomes a
`None` or an `Err` as it did before.

{% hint style="info" %}
We recommend leaving this off. It exists so that upgrading to 6.0.0 does not force
you to rewrite every call site at once. Scope it with
[`MonadOptions.BeginScope`](#scoped-configuration) if only part of your code needs
it.
{% endhint %}

## Error Code Generation

There are a few factory methods included in the library for generating `ErrorCode` instances from `Enum` and from `Exception` instances. To customise how these error codes are generated, create a class inheriting from `ErrorCodeFactory` and override the methods you wish to customise. Then create an instance and pass it into the `MonadOptions` instance via the `UseErrorCodeFactory`.

```csharp
public class MyErrorCodeFactory : ErrorCodeFactory
{
    public override ErrorCode FromEnum(Enum @enum)
    {
        // your implementation
    }
    
    public override ErrorCode FromException(Exception exception)
    {
        // your implementation
    }
}

MonadOptions.Configure(options => options.UseErrorCodeFactory(new MyErrorCodeFactory()));
```

## Error Code and Message Fallbacks

There may be exception circumstances which cause the `string` used to create the `ErrorCode` or the message of the `Error` classes to be null or white-space. In these situations, a set of fallbacks are used. These fallbacks can be configured.

```csharp
MonadOptions.Configure(options => {
    options.FallbackErrorCode = "unknown"; // default is `Unspecified`
    options.FallbackErrorMessage = "Something went wrong!" // default is `An unspecified error has occurred.`
});
```

## Scoped Configuration

Use `MonadOptions.BeginScope` when you want different options for one region of
code — a single request, a test, or a block you are debugging. The scope applies
until you dispose it, and your global configuration is untouched.

```csharp
using (MonadOptions.BeginScope(options => options.UseFallbackErrorCode("Debug")))
{
    Result<int, Error> result = Result.Try<int>(() => int.Parse(input));
}

// out here, your global configuration applies again
```

A scope accepts the same configuration methods as `Configure`, so it can override
any option:

```csharp
using (MonadOptions.BeginScope(options => options
    .UseErrorCodeFactory(new MyErrorCodeFactory())
    .UseFallbackErrorMessage("Something went wrong while debugging.")))
{
    // ...
}
```

### What a scope does

* **Inherits what you do not set.** Options you leave alone keep the values they
  had when the scope opened.
* **Takes a snapshot.** Calling `Configure` while a scope is open does not change
  that scope. The new global value applies once the scope ends.
* **Nests.** Disposing an inner scope restores the scope around it.
* **Isolates concurrent work.** A scope applies to the current asynchronous flow,
  so parallel work each sees its own options. This makes scopes safe to use in
  tests that run in parallel.

{% hint style="info" %}
A scope affects work you start inside it. It does not affect work that was already
running when you opened the scope.
{% endhint %}

{% hint style="info" %}
`Waystone.Monads.FluentValidation` options are covered by the same scope, so you
only ever open one.
{% endhint %}
