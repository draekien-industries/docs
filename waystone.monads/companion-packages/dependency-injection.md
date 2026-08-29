---
description: >-
  Lets a container write the library's configuration, so start-up code no longer
  calls MonadOptions.Configure by hand.
---

# Waystone.Monads.Extensions.DependencyInjection

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

Install this when your application already has a container and you would rather
configure the library there than in a hand-written static call.

```
dotnet add package Waystone.Monads.Extensions.DependencyInjection
```

```csharp
builder.Services.AddWaystoneMonads(
    options => options.UseFallbackErrorCode("Contoso"));

var app = builder.Build();

app.Services.UseWaystoneMonads();
```

{% hint style="success" %}
**On a host, install [Waystone.Monads.Extensions.Hosting](hosting.md) instead.** It
depends on this package, so you get everything here, and it makes the second call for
you.
{% endhint %}

This package does not change what the library does. `MonadOptions` stays ambient —
`Option` and `Result` still read it statically, no monad gains a constructor
dependency, and nothing is threaded through your call sites. What changes is only
who writes those options.

Everything it ships sits in the `Microsoft.Extensions.DependencyInjection`
namespace, which a host application already has in scope. You do not add a `using`
for any of it, including for `ReadFromConfiguration`.

## Two calls, and why

`AddWaystoneMonads` registers. `UseWaystoneMonads` installs. They are separate
because the configuration needs services the container has not built yet — an
`ILoggerFactory` does not exist while the collection is still being populated.

This is Serilog's bootstrap-logger split, with the expensive half left out.
Serilog buffers events written before the bind, because a log event emitted early
is lost forever. Nothing is lost here. Options read between the two calls are
answered from the defaults, which are valid settings rather than a broken state.

## Forgetting the install

**This is the failure mode, and it is silent.** The library keeps working on its
defaults, so nothing throws and nothing looks wrong.

So the library instruments it. A read taken after `AddWaystoneMonads` and before
`UseWaystoneMonads` writes a `Waystone.Monads.ConfigurationNotApplied` event to the
`Waystone.Monads` `DiagnosticListener`:

```csharp
// In a test suite, subscribe and throw to make the omission fatal.
listener.Subscribe(
    observer,
    name => name == MonadDiagnostics.ConfigurationNotAppliedEventName);
```

The signal is held rather than spent while nothing is subscribed, so a subscriber
attached at any point before the install still receives it. See
[Watching for configuration that was never installed](../using-the-library/observability.md#watching-for-configuration-that-was-never-installed).

[Waystone.Monads.Extensions.Hosting](hosting.md) removes the second call, and with
it the chance of forgetting it.

## Calling it twice

`AddWaystoneMonads` accumulates rather than conflicts. Each `configure` delegate
is kept, and they run in registration order at install time, so a later call
overrides an earlier one on the settings it touches. Everything else the method
does is idempotent.

That makes it safe for a library to call during its own registration without
knowing whether the application already has.

The `configure` argument is optional. `AddWaystoneMonads()` with no delegate asks
for the defaults and nothing else.

## The builder it returns

`AddWaystoneMonads` returns a `MonadServicesBuilder`, not the service collection.
Its `Services` property is the same collection you passed in, so carry on from
there:

```csharp
builder.Services.AddWaystoneMonads()
       .Services.AddSingleton<IClock, SystemClock>();
```

The builder exists so a companion package can offer a call that only makes sense
once registration has happened. `Waystone.Monads.Extensions.Hosting` hangs
[`EnableInstallOnStart()`](hosting.md#on-the-older-ihostbuilder) off it, so asking
for the install without first asking for the registration does not compile.

## What the container supplies

At install time three things are applied, in this order, each overwriting the
last:

1. The options already in effect, so an earlier `MonadOptions.Configure` call is
   carried forward rather than discarded.
2. `ErrorCodeFactory` and `ILoggerFactory`, if the container holds them.
3. Every delegate passed to `AddWaystoneMonads`, in registration order.

A delegate therefore has the last word, including over the resolved logger.

**Logging is wired for you.** `AddWaystoneMonads` registers nothing for it. The
install resolves whatever `ILoggerFactory` the container has and points
`Waystone.Monads.Extensions.Logging` at it, under that package's own
`Waystone.Monads` category. A container with no `ILoggerFactory` — a worker that
never called `AddLogging` — leaves logging unconfigured rather than failing.

**`ErrorCodeFactory` has no interface.** It is a public, non-sealed class with
`virtual` members, so you replace it by subclassing:

```csharp
builder.Services.AddSingleton<ErrorCodeFactory, ContosoErrorCodeFactory>();
```

Register it before `AddWaystoneMonads` or after — it wins either way, because the
default is registered with `TryAddSingleton`.

{% hint style="danger" %}
**Do not register a scoped service that the options will hold.** The options are
one process-wide snapshot, published once, so anything resolved into them
outlives the scope it came from. Register an `ErrorCodeFactory` as scoped and the
install either fails scope validation or quietly captures the root instance and
hands it to every request for the life of the process.

Per-request configuration is a different problem, and this package does not solve
it. Use [`MonadOptions.BeginScope`](../using-the-library/configuration.md#scoped-configuration).
{% endhint %}

## Reading from configuration

Binding is opt-in, the way Serilog's `ReadFrom.Configuration()` is.
`AddWaystoneMonads` never reaches for an `IConfiguration` on its own — you call
`ReadFromConfiguration` from the delegate you pass it:

```csharp
builder.Services.AddWaystoneMonads(
    options => options.ReadFromConfiguration(builder.Configuration));
```

```json
{
  "WaystoneMonads": {
    "FallbackErrorCode": "Contoso",
    "FallbackErrorMessage": "Something went wrong.",
    "CatchesCancellation": false
  }
}
```

| Key | Sets |
| --- | --- |
| `FallbackErrorCode` | `UseFallbackErrorCode` |
| `FallbackErrorMessage` | `UseFallbackErrorMessage` |
| `CatchesCancellation` | `UseCancellationAsFailure` |

Every key is optional, and an absent key leaves its setting alone, so a section
with one key changes one setting. Pass a second argument to read a section other
than `WaystoneMonads`.

`CatchesCancellation` is honoured in both directions. Setting it to `false` puts
the setting back even where code earlier in the chain called
`UseCancellationAsFailure()`.

**A key that is present but unusable throws.** That is the point of opting in: an
empty `FallbackErrorCode`, or a `CatchesCancellation` that is neither `true` nor
`false`, stops start-up where the mistake is written rather than degrading to a
default nobody chose.

Binding goes through the builder's `Use…` methods rather than the reflection
binder, because the settings have no public setters to bind to.

## Without a Microsoft container

Both services are resolved through `IServiceProvider.GetService` rather than any
container-specific API, so `UseWaystoneMonads` works on a provider produced by any
conforming container. `AddWaystoneMonads` needs an `IServiceCollection`; a
container that populates itself from one — which most do — is enough.
