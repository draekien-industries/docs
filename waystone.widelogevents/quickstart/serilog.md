# Serilog

You can use `Waystone.WideLogEvents` with `Serilog` by installing the `Serilog.Enrichers.Waystone.WideLogEvents` package.

```bash
dotnet add package Serilog.Enrichers.Waystone.WideLogEvents
```

### Usage

Configure Serilog to use the Wide Log Events enricher and optional sampling filter:

```cs
using Serilog;
using Serilog.Enrichers.Waystone.WideLogEvents;

Log.Logger = new LoggerConfiguration()
    .Enrich.FromWideLogEventsContext()
    .Filter.WithWideLogEventsSampling(options => {
        options.InformationSampleRate = 0.5; // Log 50% of info logs

        // Optionally provide a custom random number generator
        options.RandomDoubleProvider = new MyRandomProvider();
    })
    // ... other configuration
    .CreateLogger();
```

### Customizing Randomness

The sampling filter uses `IRandomDoubleProvider` to determine whether a log event should be sampled. By default, it uses a simple implementation that instantiates a `new Random()`.

To use `Random.Shared` or a custom RNG, implement the interface:

```cs
public class MyRandomProvider : IRandomDoubleProvider
{
    public double NextDouble() => Random.Shared.NextDouble();
}
```
