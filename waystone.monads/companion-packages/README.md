---
description: >-
  Packages that sit beside the library. None is required, and none changes how
  Waystone.Monads behaves.
---

# Overview


{% hint style="warning" %}
**This page describes `7.0.0-beta.x`, a pre-release.** NuGet gives you `6.x` unless you ask for a pre-release:

```
dotnet add package Waystone.Monads --prerelease
```

Or set the version yourself: `<PackageReference Include="Waystone.Monads" Version="7.0.0-beta.*" />`.

The API can still change before `7.0.0` is stable.
{% endhint %}

These all ship from the same repository as `Waystone.Monads`, on the same version
number, and all are purely additive. You install one because you want what it adds,
not because an upgrade made you.

| Package | Install it in | What it adds |
| --- | --- | --- |
| [Waystone.Monads.Shouldly](shouldly.md) | Test projects | Shouldly assertions that take an `Option` or a `Result` directly, so a failure names the state it found |
| [Waystone.Monads.Linq](linq.md) | Anywhere | `Select`, `SelectMany` and `Where`, so C# query syntax works over `Option` and `Result` |
| [Waystone.Monads.Extensions.DependencyInjection](dependency-injection.md) | Any application with an `IServiceCollection` | `AddWaystoneMonads` and `UseWaystoneMonads`, so a container writes the configuration |
| [Waystone.Monads.Extensions.Hosting](hosting.md) | Applications built on `Microsoft.Extensions.Hosting` | Runs that install at host start, so there is no second call to forget |
| [Waystone.Monads.FluentValidation](fluentvalidation.md) | Anywhere you already use FluentValidation | `Validate` and `ValidateAsync`, so a validator returns a `Result` that errs with an `Error` |
| [Waystone.Monads.SystemTextJson](systemtextjson.md) | Anywhere an `Option` or `Result` crosses a wire | Converters that round-trip both types through `System.Text.Json` |
| [Waystone.Monads.NewtonsoftJson](newtonsoftjson.md) | The same, on Json.NET | The same converters, writing byte-identical JSON |

No package here changes the behaviour of anything in `Waystone.Monads`. Three of them
add vocabulary: remove one and the code that used it stops compiling, and nothing else
moves. Two change only *who writes* your configuration, not what the settings mean —
you can write the same settings by hand with `MonadOptions.Configure`. The last two
teach a serializer a format it did not know; your own code does not change at all.

The two JSON packages write the same format on purpose. Pick the serializer you already
use, and a payload one of them writes is a payload the other reads.

There is one more package, [Waystone.Monads.Extensions.Logging](logging.md). It sits
apart from the list because it configures the library rather than extending it — it
hands the library your `ILogger`, so the exceptions it swallows reach your logs.
[Observability](../guides/observability.md) covers the signals that need no package
at all.
