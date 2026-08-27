# Configuration

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

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
* **Nests.** Disposing the innermost scope restores the scope around it.
* **Restores only from the inside out.** A scope that is no longer the innermost one
  declines to restore anything when you dispose it, and reports itself instead. See
  below.
* **Isolates concurrent work.** A scope applies to the current asynchronous flow,
  so parallel work each sees its own options. This makes scopes safe to use in
  tests that run in parallel.

{% hint style="info" %}
A scope affects work you start inside it. It does not affect work that was already
running when you opened the scope.
{% endhint %}

{% hint style="warning" %}
**Dispose scopes in the reverse of the order you opened them.** A `using` block does
this for you. Nothing else guarantees it.
{% endhint %}

#### What happens when you dispose out of order

Since 7.0.0, a scope restores only when it is the innermost one still open. Disposing
it at any other time changes nothing and reports the mistake.

`Dispose` looks at the options in effect on the current flow and picks one of three
paths:

| What it finds | What it does |
| --- | --- |
| The options this scope installed — so this scope is the innermost one | Restores what came before it |
| The options this scope restored to — so it has already been disposed | Nothing, silently |
| Anything else | Nothing, and writes a `ScopeDisposedOutOfOrder` diagnostic event |

It never throws, on any path.

**The third path leaves the early-disposed scope's options in effect.** They stay live
until the inner scope that is still open is disposed, which then restores them as *its
own* predecessor — so those options outlive the scope that installed them. That is the
bug the event exists to tell you about.

Two more cases take the third path, both worth knowing:

* Disposing a scope from a different asynchronous flow than the one that opened it. A
  scope lives in the flow, so another flow's `Dispose` never sees it.
* Disposing a `default(MonadOptionsScope)`. Before 7.0.0 this dropped the flow back to
  your global configuration; now it reports like any other out-of-order disposal.

**Repeated disposal on the third path reports every time.** A scope that has already
declined cannot remember that it did — it is a readonly struct — so each further
`Dispose` writes the event again. Deduplicate in your subscriber if that matters. The
"harmless twice" promise covers the restoring path only.

To see these events, see
[Watching for a scope disposed out of order](observability.md#watching-for-a-scope-disposed-out-of-order).

{% hint style="info" %}
`Waystone.Monads.FluentValidation` options are covered by the same scope, so you
only ever open one.
{% endhint %}
