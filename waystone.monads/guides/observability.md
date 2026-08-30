---
description: >-
  See the exceptions the library swallows, with nothing extra installed.
---

# Observability

`Option.Try` and `Result.Try` catch the exception your factory throws and hand
you back a `None` or an `Err`. That is the point of them. But it means the
exception never reaches you, and in the `Option` case it is gone for good.

This page shows you how to see those exceptions anyway.

**Everything on this page works with no extra package.** From 6.7.0 the library
reports on sources named after itself, and your pipeline finds them by name.

| Signal | What you get |
| --- | --- |
| Metrics | A count of handled exceptions, tagged by exception type |
| Diagnostic events | Three events you can subscribe to, including two configuration mistakes |

Logs are the exception, and they are a separate page. See
[Logging](../companion-packages/logging.md).

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

## Subscribe to an event

Skip this unless you are building your own integration. The logging package
already does it for you.

The library writes three events to a `DiagnosticListener` named `Waystone.Monads`.
`MonadDiagnostics` gives you a token for each one. A token pairs the event's name
with the type of payload it carries:

| Token | Payload |
| --- | --- |
| `MonadDiagnostics.ExceptionHandledEvent` | `ExceptionHandled` |
| `MonadDiagnostics.ScopeDisposedOutOfOrderEvent` | `ScopeDisposedOutOfOrder` |
| `MonadDiagnostics.ConfigurationNotAppliedEvent` | `ConfigurationNotApplied` |

Call `Subscribe` on the token. Your callback gets the payload directly:

```csharp
using System;
using Waystone.Monads.Diagnostics;

IDisposable watching = MonadDiagnostics.ExceptionHandledEvent.Subscribe(
    handled =>
    {
        // handled.Exception, handled.Caller, handled.Monad
    });
```

The `ExceptionHandled` payload is:

```csharp
public sealed record ExceptionHandled(
    Exception Exception,
    CallerInfo Caller,
    MonadKind Monad);
```

Subscribe once, at start-up. It does not matter whether you subscribe before or
after the first `Try` runs.

Use the token rather than the name constants. Typing a name by hand gives you three
ways to get it wrong: the listener name, the event name, and the payload type. Every
one of them fails silently. You get no exception, no warning, and an empty
dashboard. The token cannot point at the wrong event.

### Disposing the subscription

Dispose the return value to detach. Disposing it twice is safe.

A subscriber that runs for the life of your application can be left alone. Anything
shorter-lived must be disposed, or it leaks an observer for the rest of the process.

{% hint style="danger" %}
**Your subscriber runs on the thread that wrote the event, synchronously.** For
`ExceptionHandledEvent` that is the throwing thread, inside the `catch`. Two
consequences you have to plan for:

- Slow work in the subscriber delays the caller waiting for its `None` or `Err`.
- An exception thrown from your subscriber escapes the `Try` that was supposed to
  swallow the original one. We do not catch it for you. That is deliberate — it is
  how you make one of these events fatal in a test suite.

Queue the work and return.
{% endhint %}

### Without the token

You never need a Waystone package to observe the library, and that has not changed.
The token is a shortcut over the standard `DiagnosticListener` API, not a
replacement for it. Here is the same subscription written by hand:

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
        if (written.Key == MonadDiagnostics.ExceptionHandledEventName
         && written.Value is ExceptionHandled handled)
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

Two traps to handle yourself if you go this way:

- **`DiagnosticListener.Write` ignores your predicate.** The predicate only decides
  what `IsEnabled` reports. Your observer receives every event written to that
  listener, so check `written.Key` yourself — as the sample above does.
- **The payload arrives as `object?`.** Test its type rather than casting it.

## Watching for a scope disposed out of order

The library writes a second event, `Waystone.Monads.ScopeDisposedOutOfOrder`, to the
same listener. It fires when a `MonadOptionsScope` is disposed at a point where it is
not the innermost open scope, which means nothing was restored — see
[What happens when you dispose out of order](configuration.md#what-happens-when-you-dispose-out-of-order).

The payload is:

```csharp
public sealed record ScopeDisposedOutOfOrder(
    MonadOptions? Scope,
    MonadOptions? Live);
```

* `Scope` is what the disposed scope had installed. It is `null` exactly when a
  `default(MonadOptionsScope)` was disposed, because that scope was never begun.
* `Live` is what is in effect instead. It is `null` when no scope remains open.

Subscribe the same way you subscribe to the exception event:

```csharp
IDisposable watching = MonadDiagnostics.ScopeDisposedOutOfOrderEvent.Subscribe(
    disposed =>
    {
        // disposed.Scope is gone; disposed.Live is what is in effect instead.
    });
```

**There is no caller information in the payload.** `IDisposable.Dispose()` takes none,
so the library has nothing to pass. Your subscriber does run synchronously inside
`Dispose`, on the disposing thread, so capturing a stack trace there names the offending
call site — which is the reason to write one of these at all. This is a bug in your own
code that the library cannot fix for you, and the event is how you find it.

**Expect duplicates.** A scope that has already declined to restore reports again on
every further `Dispose`, because a readonly struct cannot record that it reported. If
you alert on this, deduplicate.

## Watching for configuration that was never installed

The library writes a third event, `Waystone.Monads.ConfigurationNotApplied`, to the
same listener. It fires when something reads `MonadOptions` after configuration has
been registered but before it has been installed — in practice, when an application
called `AddWaystoneMonads` and never called `UseWaystoneMonads`. See
[Dependency injection and hosting](../companion-packages/dependency-injection.md#forgetting-the-install).

The payload carries no data:

```csharp
public sealed record ConfigurationNotApplied;
```

There is nothing useful to put in it. The event's whole meaning is that it fired at
all, and the read that triggered it was answered from the defaults.

**The signal is held, not spent, while nobody is listening.** If no subscriber is
attached when the first early read happens, the library keeps the flag set, so a
subscriber attached later still receives it.

**Configuration arriving by any route disarms it** — `UseWaystoneMonads`, the host
install, or a plain `MonadOptions.Configure` call — whether or not the event was ever
written.

**Expect it once per process, but do not rely on it.** Writing the event disarms the
flag, so later reads write nothing. Two threads reading at the same moment can each
write before either disarms it. The payloads are identical and carry no data, so
deduplicate in your subscriber if that matters.

**It reports reads, not registrations.** `AddWaystoneMonads` on its own writes
nothing. The event needs something to actually read the options in the window between
registration and install. An application that registers, never reads early, and never
installs gets no event — and no wrong behaviour either, because nothing consulted the
options.

In a test suite, subscribe and throw to make the omission fatal. We do not catch
what your callback throws:

```csharp
using IDisposable watching =
    MonadDiagnostics.ConfigurationNotAppliedEvent.Subscribe(
        _ => throw new InvalidOperationException(
            "AddWaystoneMonads was called but UseWaystoneMonads was not."));
```

{% hint style="info" %}
**You will not see this event unless you use the dependency injection package.**
Nothing else in the library marks configuration as pending, so an application that
calls `MonadOptions.Configure` directly never triggers it.
{% endhint %}

## What the library does not report

**Exceptions it lets through.** The metric and the `ExceptionHandled` event fire only
when `Try` or `TryAsync` catches something. An exception that propagates to you is yours
to log. `ScopeDisposedOutOfOrder` is unrelated to `Try` and has no counter — it reports
a misuse of configuration, not a failure in your work.

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

Every signal the library publishes checks whether anything is subscribed before it
does any work. With no listener attached, a `Try` that throws allocates exactly what it
allocated before 6.7.0 — measured, not assumed. Attach both and you pay 40 bytes per
handled exception, which is the event payload; the counter allocates nothing at all.

`ScopeDisposedOutOfOrder` is gated the same way, so an out-of-order disposal in a
process with no listener allocates no payload either. It also costs nothing on the
normal path — the check runs only once `Dispose` has already decided it cannot
restore.

## These names are a contract

Dashboards and alert rules bind to strings, and no compiler warns you when a
string changes. So treat every name the library publishes the way you treat a
public type:

| Thing | Name |
| --- | --- |
| Meter | `Waystone.Monads` |
| `DiagnosticListener` | `Waystone.Monads` |
| Event | `Waystone.Monads.ExceptionHandled` |
| Event | `Waystone.Monads.ScopeDisposedOutOfOrder` |
| Event | `Waystone.Monads.ConfigurationNotApplied` |
| Counter | `waystone.monads.exceptions_handled` |
| Tags | `error.type`, `waystone.monads.monad` |
| Log category | `Waystone.Monads` |

`MonadDiagnostics` holds every one of them as a constant, so you never have to type
one out. Use the constants for anything that names a signal — a dashboard query, a
log line, a hand-written subscription. To subscribe, use the event tokens instead:
they name the event for you and fix the payload type at the same time.

We will not rename these names outside a major release, and we will tell you in
[Deprecations](../upgrading-and-deprecations/deprecations.md) when we do.
