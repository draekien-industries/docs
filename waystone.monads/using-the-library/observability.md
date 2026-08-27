---
description: >-
  See the exceptions the library swallows — as metrics with no extra package,
  and as logs with one line of setup.
---

# Observability

`Option.Try` and `Result.Try` catch the exception your factory throws and hand
you back a `None` or an `Err`. That is the point of them. But it means the
exception never reaches you, and in the `Option` case it is gone for good.

This page shows you how to see those exceptions anyway.

From 6.7.0 the library reports them itself, on sources named after itself. You
get two signals:

| Signal | What you install | What you get |
| --- | --- | --- |
| Metrics | Nothing | A count of handled exceptions, tagged by exception type |
| Logs | `Waystone.Monads.Extensions.Logging` | One entry per handled exception, with the call site |

{% hint style="info" %}
**Why metrics need no package and logs do.** Your metrics pipeline discovers
meters by name, so we publish one and you name it. `Microsoft.Extensions.Logging`
has no equivalent — there is no ambient logger to find. You have to hand us one.
{% endhint %}

## Count handled exceptions

Add `Waystone.Monads` to the meters your pipeline already collects. That is the
whole setup.

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics.AddMeter("Waystone.Monads"));
```

Prometheus, Datadog and anything else that reads .NET meters work the same way —
none of them needs a Waystone package.

You now get one instrument:

| Instrument | Type | Unit |
| --- | --- | --- |
| `waystone.monads.exceptions_handled` | `Counter<long>` | `{exception}` |

It carries two tags:

| Tag | Values | What it tells you |
| --- | --- | --- |
| `error.type` | The exception's full type name, such as `System.FormatException` | Which exception. This is the OpenTelemetry attribute of the same name. |
| `waystone.monads.monad` | `option` or `result` | Whether the exception is gone or survived |

That second tag matters more than it looks. An exception counted as `option` was
discarded — this counter is the only record that it happened. One counted as
`result` also went into the `Err`, so your error handling still has it.

{% hint style="info" %}
**Exception types are safe to tag with.** Any one application throws a handful of
types, so `error.type` will not blow up your metrics backend's cardinality
budget.
{% endhint %}

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
[`BeginScope`](configuration.md#scoped-configuration) redirects them for one
asynchronous flow and leaves the rest of your process alone:

```csharp
using (MonadOptions.BeginScope(
    options => options.UseLogger(captured, LogLevel.Warning)))
{
    // Everything in here logs at Warning, to `captured`.
}
```

This is what makes the logging usable in tests that run in parallel.

## Subscribe to the raw event

Skip this unless you are building your own integration. The logging package
already does it for you.

The library writes a `Waystone.Monads.ExceptionHandled` event to a
`DiagnosticListener` named `Waystone.Monads`. Subscribe and you receive an
`ExceptionHandled` record:

```csharp
public sealed record ExceptionHandled(
    Exception Exception,
    CallerInfo Caller,
    MonadKind Monad);
```

`MonadDiagnostics` holds every name as a constant, so you never type one out.

Watch for the listener, then subscribe to the one event you want:

```csharp
using System.Diagnostics;
using Waystone.Monads.Diagnostics;

public sealed class MonadWatcher : IObserver<DiagnosticListener>
{
    public void OnNext(DiagnosticListener listener)
    {
        if (listener.Name != MonadDiagnostics.ListenerName)
        {
            return;
        }

        listener.Subscribe(
            new HandledExceptions(),
            name => name == MonadDiagnostics.ExceptionHandledEventName);
    }

    public void OnCompleted() { }
    public void OnError(Exception error) { }
}

public sealed class HandledExceptions : IObserver<KeyValuePair<string, object?>>
{
    public void OnNext(KeyValuePair<string, object?> written)
    {
        if (written.Value is ExceptionHandled handled)
        {
            // handled.Exception, handled.Caller, handled.Monad
        }
    }

    public void OnCompleted() { }
    public void OnError(Exception error) { }
}
```

Hook it up once, at start-up:

```csharp
DiagnosticListener.AllListeners.Subscribe(new MonadWatcher());
```

`AllListeners` replays listeners that already exist, so it does not matter whether
you subscribe before or after the first `Try` runs.

{% hint style="danger" %}
**Your subscriber runs on the thread that threw, synchronously, inside the
`catch`.** Two consequences you have to plan for:

- Slow work in the subscriber delays the caller waiting for its `None` or `Err`.
- An exception thrown from your subscriber escapes the `Try` that was supposed to
  swallow the original one.

Queue the work and return.
{% endhint %}

## What the library does not report

**Exceptions it lets through.** Both signals fire only when `Try` or `TryAsync`
catches something. An exception that propagates to you is yours to log.

**Cancellations, by default.** From 6.0.0 an `OperationCanceledException` is not
caught, so nothing counts or logs it. Call
[`UseCancellationAsFailure`](configuration.md#cancellation) and it becomes an
ordinary caught exception, counted and logged like any other.

**Traces.** The library publishes no `ActivitySource`. It creates no spans of its
own, and OpenTelemetry's conventions no longer recommend recording an exception
that gets handled and never escapes a span — which is exactly what these
exceptions are. If you want an `Err` marked on a span you own, do it at the call
site you chose:

```csharp
result.InspectErr(
    error => Activity.Current?.SetStatus(
        ActivityStatusCode.Error,
        error.Message));
```

## You pay nothing when nobody is listening

Both signals check whether anything is subscribed before they do any work. With
no listener attached, a `Try` that throws allocates exactly what it allocated
before 6.7.0 — measured, not assumed. Attach both and you pay 40 bytes per
handled exception, which is the event payload; the counter allocates nothing at
all.

## These names are a contract

Dashboards and alert rules bind to strings, and no compiler warns you when a
string changes. So treat every name on this page the way you treat a public type:

| Thing | Name |
| --- | --- |
| Meter | `Waystone.Monads` |
| `DiagnosticListener` | `Waystone.Monads` |
| Event | `Waystone.Monads.ExceptionHandled` |
| Counter | `waystone.monads.exceptions_handled` |
| Tags | `error.type`, `waystone.monads.monad` |
| Log category | `Waystone.Monads` |

We will not rename them outside a major release, and we will tell you in
[Deprecations](../upgrading-and-deprecations/deprecations.md) when we do.

## Replacing UseExceptionLogger

`MonadOptions.UseExceptionLogger` used to be the only way to see these
exceptions. It is obsolete from 6.7.0 and goes in 7.0.0.

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
