# Configuration

This library contains some configurable behaviours. Configuration is available via the `Configure` method of the `MonadOptions` class. It is recommended to configure each option below _once_ in the lifecycle of your application.

If you need different settings for one part of your code, use a [scope](#scoped-configuration) instead of reconfiguring globally.

## Logging

This library catches exceptions in several places and turns them into
non-throwing types. To see those exceptions, install
`Waystone.Monads.Extensions.Logging` and hand the library your logger once:

```csharp
MonadOptions.Configure(options => options.UseLoggerFactoryFrom(app.Services));
```

Use `UseLoggerFactory(factory)` if you have no service provider, or
`UseLogger(logger)` if you already hold a logger.

Each entry carries the exception plus the call site that caught it — the member
name, the source text of the delegate you passed, and the line number.

To count these exceptions instead, install nothing at all. Read
[observability.md](observability.md "mention") for both signals, the levels, and the
names your dashboards will bind to.

{% hint style="warning" %}
**`UseExceptionLogger` is obsolete from 6.7.0 and goes in 7.0.0.** It takes a
delegate you write yourself, and it holds only one — so a second integration
silently replaces the first. Both it and the new package fire while it exists, so
delete the old call when you add the package or you will log everything twice. See
[Deprecations](../upgrading-and-deprecations/deprecations.md#seeing-handled-exceptions-through-a-hand-written-delegate).
{% endhint %}

{% hint style="info" %}
**The library also writes to the console whenever a debugger is attached**, whether or
not you configure a logger. It prints the exception, the call site and the argument
expression, then reports it through the signals above as well. This is a debugging aid,
so it costs nothing in a normal run — but do not read a console message as proof that
your logger ran.
{% endhint %}

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

A cancellation is then caught, counted and logged like any other handled
exception, and becomes a `None` or an `Err` as it did before.

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
    public override ErrorCode FromException(Exception exception)
    {
        // your implementation
    }
}

MonadOptions.Configure(options => options.UseErrorCodeFactory(new MyErrorCodeFactory()));
```

{% hint style="warning" %}
**`FromEnum` is no longer one of the methods to override.** It is obsolete from 6.2.0
and removed in 7.0.0, because a factory runs too late for the compiler, the analyzers
or the error code registry to see what it returns. Shape enum codes with
`[ErrorCodeCatalog(Format = "…")]` instead — see
[generated-error-codes.md](generated-error-codes.md "mention") — and keep the factory
for `FromException`, which is unaffected.
{% endhint %}

## Error Code and Message Fallbacks

There may be exception circumstances which cause the `string` used to create the `ErrorCode` or the message of the `Error` classes to be null or white-space. In these situations, a set of fallbacks are used. These fallbacks can be configured.

```csharp
MonadOptions.Configure(options => options
    .UseFallbackErrorCode("unknown")            // default: Unspecified
    .UseFallbackErrorMessage("Something went wrong!")); // default: An unexpected error occurred.
```

**The substitution is silent.** `new ErrorCode(code)` and `new Error(code, message)`
trim what you pass, and swap in the fallback when the result is empty. Neither throws,
and nothing is logged. So an `Error` whose message reads `An unexpected error
occurred.` is telling you a call site passed a blank message, not that the library hit
an unexpected error. Pass a real message at every call site — a fallback says nothing
about what actually failed.

Both configuration methods reject a blank argument themselves. `UseFallbackErrorCode`
and `UseFallbackErrorMessage` throw an `ArgumentException` when you pass null, empty or
whitespace, because a fallback that is itself unusable would leave nothing to fall back
to.

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

{% hint style="warning" %}
**Dispose scopes in the reverse of the order you opened them.** A `using` block does
this for you. If you hold scopes in variables and dispose an outer one while an inner
one is still open, the outer scope restores what came before *it* and the inner scope
is discarded — silently, with no exception. Disposing the same scope twice is harmless:
each call restores the same saved options.
{% endhint %}

{% hint style="info" %}
`Waystone.Monads.FluentValidation` options are covered by the same scope, so you
only ever open one.
{% endhint %}
