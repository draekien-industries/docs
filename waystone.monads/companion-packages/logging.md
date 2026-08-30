---
description: >-
  See every exception the library swallows, as a log entry with the call site
  attached.
icon: file-lines
---

# Logging

`Waystone.Monads.Extensions.Logging` sends the exceptions the library catches to
your `ILogger`. It is the only signal that needs a package — counters and
diagnostic events work with no install at all. See
[Observability](../guides/observability.md) for those.

## Log handled exceptions

Install the package:

```
dotnet add package Waystone.Monads.Extensions.Logging
```

Then point it at your logger once, during start-up. Pick the line that matches
how your application is built.

**You have a host and a service provider:**

```csharp
var app = builder.Build();

MonadOptions.Configure(options => options.UseLoggerFactoryFrom(app.Services));
```

**You have no container** — a console app, a test, a worker you built by hand:

```csharp
using ILoggerFactory factory = LoggerFactory.Create(
    builder => builder.AddConsole());

MonadOptions.Configure(options => options.UseLoggerFactory(factory));
```

{% hint style="info" %}
`LoggerFactory.Create` is not in `Microsoft.Extensions.Logging.Abstractions`, which
is all this package brings with it. Add `Microsoft.Extensions.Logging` to build a
factory yourself, plus a provider package such as
`Microsoft.Extensions.Logging.Console`. An application with a host already has
both.
{% endhint %}

**You already hold a logger:**

```csharp
MonadOptions.Configure(options => options.UseLogger(logger));
```

`UseLoggerFactoryFrom` asks the provider for an `ILoggerFactory` through
`IServiceProvider.GetService`. It takes no dependency-injection package, so any
container that can hand you a provider works — not just Microsoft's. If no
factory is registered it throws and tells you to call `UseLoggerFactory` instead.

### What you get in each entry

The exception itself, plus three properties describing the call site the compiler
recorded for you:

| Property | What it holds |
| --- | --- |
| `MemberName` | The member that called `Try` |
| `ArgumentExpression` | The source text of the delegate you passed |
| `LineNumber` | The line the `Try` call sits on |

The exception goes in the logger's exception parameter, not into properties of
its own. Your OpenTelemetry logging bridge reads `exception.type`,
`exception.message` and `exception.stacktrace` off it, so you get those for free
and you get them once.

{% hint style="info" %}
**Property names are PascalCase on purpose.** Serilog only accepts property names
matching `[A-Za-z0-9_]+`. Write `{code.function.name}` in a message template and
Serilog prints it as text instead of binding it. So the log properties use
PascalCase, and the dotted OpenTelemetry spellings stay on the metric tags, where
nothing parses a template.
{% endhint %}

### Choose the level

The default is `Debug`. A `Try` that hands back a `None` or an `Err` did what you
asked it to, so warning about it fills your logs with noise.

Pass a different level if you disagree:

```csharp
MonadOptions.Configure(
    options => options.UseLoggerFactoryFrom(app.Services, LogLevel.Warning));
```

{% hint style="info" %}
OpenTelemetry's semantic conventions suggest `WARN` for an exception the
application expects to handle. We default to `Debug` instead. Pass
`LogLevel.Warning` to follow the convention.
{% endhint %}

### Filter the library's own output

`UseLoggerFactory` and `UseLoggerFactoryFrom` create a logger in the
`Waystone.Monads` category, so you can turn the library up or down without
touching anything else:

```json
{ "Logging": { "LogLevel": { "Waystone.Monads": "Warning" } } }
```

`UseLogger` does not do this — the logger you pass keeps whatever category it
already had.

### Change the logger for one block

Both the logger and the level live on the `MonadOptions` scope, so
[`BeginScope`](../guides/configuration.md#scoped-configuration) redirects them for one
asynchronous flow and leaves the rest of your process alone:

```csharp
using (MonadOptions.BeginScope(
    options => options.UseLogger(captured, LogLevel.Warning)))
{
    // Everything in here logs at Warning, to `captured`.
}
```

This is what makes the logging usable in tests that run in parallel.

## Replacing UseExceptionLogger

`MonadOptions.UseExceptionLogger` used to be the only way to see these
exceptions. It was obsolete from 6.7.0 and **7.0.0 removes it**.

```diff
-MonadOptions.Configure(options => options.UseExceptionLogger((ex, caller) =>
-{
-    Console.WriteLine($"{ex} at {caller.MemberName}:{caller.LineNumber}");
-}));
+MonadOptions.Configure(options => options.UseLoggerFactoryFrom(app.Services));
```

{% hint style="warning" %}
**Do not configure both.** They both still fire in 6.x, so every handled
exception is reported twice until you delete the old call.
{% endhint %}

The old hook held one delegate. A second observer replaced the first without
saying so, which meant it could never support more than one integration at a
time. The diagnostic event has no such limit.
