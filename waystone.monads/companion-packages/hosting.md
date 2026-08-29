---
description: >-
  Installs the container-registered configuration from the host's own start-up
  sequence, so there is no second call to forget.
---

# Waystone.Monads.Extensions.Hosting

{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

Install this if your application is built on `Microsoft.Extensions.Hosting`. It is
the shape most applications want.

```
dotnet add package Waystone.Monads.Extensions.Hosting
```

```csharp
builder.AddWaystoneMonads(options => options.UseFallbackErrorCode("Contoso"));

var app = builder.Build();
app.Run();

// No second call.
```

That `AddWaystoneMonads` is an extension on `IHostApplicationBuilder`, which both
`WebApplicationBuilder` and the builder from `Host.CreateApplicationBuilder`
implement.

This package depends on
[Waystone.Monads.Extensions.DependencyInjection](dependency-injection.md), so
installing it gives you both. Read that page for what the configuration delegate
can do, how the container supplies an `ErrorCodeFactory` and an `ILoggerFactory`,
and how to bind settings from `IConfiguration`. Everything here is about *when*
those settings are applied.

## The call it removes

The dependency injection package splits registration from installation, because
configuration registered on an `IServiceCollection` needs services the container
has not built yet. That leaves an application holding a second call it has to
remember:

```csharp
var app = builder.Build();

app.Services.UseWaystoneMonads();   // easy to forget
```

Forgetting it is silent — the library keeps working on its defaults. This package
removes the call rather than relying on anybody to remember it.

## On the older IHostBuilder

`IHostBuilder` does not implement `IHostApplicationBuilder`, so reach the same
pair through `ConfigureServices`:

```csharp
new HostBuilder().ConfigureServices(
    services => services
               .AddWaystoneMonads(options => options.UseFallbackErrorCode("Contoso"))
               .EnableInstallOnStart());
```

`EnableInstallOnStart` hangs off the `MonadServicesBuilder` that
`AddWaystoneMonads` returns, so asking for the install without first asking for
the registration does not compile. It registers the installer and nothing else, so
`AddWaystoneMonads` is still where configuration goes.

Calling it twice installs once — the registration is deduplicated on the
implementation type.

## Registration order does not matter

The install runs in `IHostedLifecycleService.StartingAsync`, which the host calls
on every hosted service before it calls `StartAsync` on any of them. So a
background service that reads `MonadOptions` in its own `StartAsync` sees the
installed configuration, whether it was registered before `EnableInstallOnStart`
or after.

That is the whole reason this is a lifecycle service rather than a plain
`IHostedService`. A plain one would install in `StartAsync`, in registration
order, and a service registered ahead of it would read the defaults.

{% hint style="warning" %}
**Work done before the host starts is still too early.** A read taken while the
service collection is being populated, or between `Build()` and `Run()`, runs
ahead of every hosted service. It is answered from the defaults and reported
through the `Waystone.Monads.ConfigurationNotApplied` diagnostic event, exactly as
it is without this package. See
[Watching for configuration that was never installed](../using-the-library/observability.md#watching-for-configuration-that-was-never-installed).

Configuration is applied at host start, not at container build.
{% endhint %}

## Without a host

Nothing here applies. Call `UseWaystoneMonads()` on the provider yourself —
[Waystone.Monads.Extensions.DependencyInjection](dependency-injection.md) is all a
console application, a test, or a container built by hand needs.
