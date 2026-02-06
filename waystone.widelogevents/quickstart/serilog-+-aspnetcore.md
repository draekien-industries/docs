# Serilog + AspNetCore

You can use `Waystone.WideLogEvents` with `Serilog`  in `ASP.NET Core` applications by installing the `Serilog.Enrichers.Waystone.WideLogEvents.AspNetCore` package.

```bash
dotnet add package Serilog.Enrichers.Waystone.WideLogEvents.AspNetCore
```

### Usage

#### Configure Serilog

In your `Program.cs`, configure Serilog to use the wide log events enricher and the sampling filter:

```cs
using Serilog;
using Serilog.Enrichers.Waystone.WideLogEvents;

builder.Host.UseSerilog((context, config) => config
   .Enrich.FromWideLogEventsContext()
   .Filter.WithWideLogEventsSampling()
   .ReadFrom.Configuration(context.Configuration));
```

#### Register Middleware

Add the `UseWideLogEventsContext` middleware to your application pipeline. This middleware should typically be placed early in the pipeline to capture as much information as possible.

```cs
using Serilog.Enrichers.Waystone.WideLogEvents.AspNetCore;

var app = builder.Build();

app.UseWideLogEventsContext();

// ... other middleware
```

#### Push Properties

You can now push properties to the `WideLogEventContext` anywhere in your request lifecycle (e.g. in controllers, minimal API handlers, or services):

```cs
using Waystone.WideLogEvents;

app.MapGet("/weatherforecast", () =>
{
    var forecast = // ... get forecast

    // This property will be included in the final "wide" log for this request
    WideLogEventContext.PushProperty("ForecastCount", forecast.Length);

    return forecast;
});
```

### Configuration Options

You can customize the sampling behavior of the log events by configuring the sampling filter

```cs
builder.Host.UseSerilog((context, config) => config
   .Enrich.FromWideLogEventsContext()
   .Filter.WithWideLogEventsSampling(options =>
   {
       options.InformationSampleRate = 0.5; // Log 50% of information logs
       options.ErrorSampleRate = 1.0;       // Log 100% of error logs

       // Custom random provider for sampling decisions
       options.RandomDoubleProvider = new MyRandomProvider();
   })
   .ReadFrom.Configuration(context.Configuration));
```

#### Random Provider

See [#customizing-randomness](serilog.md#customizing-randomness "mention")
