# Welcome

`Waystone.WideLogEvents` is a set of libraries inspired by [Logging Sucks - by Boris Tane](https://loggingsucks.com/). This package provides a way for capturing and managing a set of properties throughout a logical scope, which can then be enriched into your log events.

### Key Concepts

#### Wide Log Events

A wide log event is a pattern where instead of logging many small, disconnected events, you accumulate relevant information throughout a process (like a HTTP request) and log it all at once at the end. This makes it much easier to correlate data and understand the full context of an operation.

#### Wide Log Event Context

The central point for managing properties within a scope. It uses `AsyncLocal` to ensure properties are correctly tracked across async calls.

#### Wide Log Event Scope

A disposable scope that manages the lifecycle of properties. When a scope is created, it starts a new set of properties. When disposed, it restores the previous scope's properties.

### Usage

#### Start a Scope

Wrap your operation in a `WideLogEventScope`

```cs
using (var scope = WideLogEventContext.BeginScope())
{
    // Your code here
}
```

#### Pushing Properties

You can push properties to the current scope at any time

```cs
WideLogEventContext.PushProperty("CustomerId", 12345);
WideLogEventContext.PushProperty("OrderDetails", new { Total = 100.00, ItemCount = 3 });
```

## Links

* [Source on GitHub](https://github.com/draekien-industries/waystone-dotnet) — these packages live under `src/Serilog.Enrichers.Waystone.WideLogEvents`
* [Serilog.Enrichers.Waystone.WideLogEvents on NuGet](https://www.nuget.org/packages/Serilog.Enrichers.Waystone.WideLogEvents)
* [Serilog.Enrichers.Waystone.WideLogEvents.AspNetCore on NuGet](https://www.nuget.org/packages/Serilog.Enrichers.Waystone.WideLogEvents.AspNetCore)
* [Report an issue](https://github.com/draekien-industries/waystone-dotnet/issues) — for the libraries or for these docs
