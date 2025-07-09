---
title: "Actually using Application Insights"
date: 2024-05-19T21:07:19+02:00
tags: [telemetry]
draft: true
---

So, we just deployed to production for the first time our new shiny product, since my company is a microsoft shop we used blazor + dotnet core. We started testing our 3rd party integrations and something was not working as intended. Meetings were called to find out what system sent what etc. Since our team has an aggressive telemetry strategy, we could tell instantly what we sent, when, and what the response was.

The conversation went something like this:
Me : We know that we got a 500 response from application insights
A PM from another team : You guys use that?? Its very expensive. I envy your budget.

Our application has medium traffic last month we got a 26$ bill for production

Don't get me wrong, someone could very well read the logs, but in a production scenario, this is not only extremely inconvenient but also a very very bad idea.
 In the modern world, where application live primarily in the cloud, when horizontal scaling, multiple services, are a reality, how could someone find a log about an incoming http call? Should they search the logs of instance 1, instance 2? An error prone procedure. What about searching for keywords, datetime ranges? I won't even go there.



1 Create a preprocessor to ignore hangfire sql calls
  Out dev application insights used 20$/month, so naturally since dev environment did not have the traffic to warrant such high volume of logs
  we investigated. We found out that most data stored in applciation insights were sql dependency events relating to Hangfire.  
2 Create a preprocessor to ignore calls that have no value, e.g. robots.txt etc.
3 Create a preprocessor to add trace identifier to all telemetry items, yes yes all items have already operationId, but if you use traces for your logs

-- insert code example?
-- maybe redo this with opentelemetry?

```
  internal static LoggerConfiguration AddApplicationInsightsLogging(this LoggerConfiguration logger, IServiceProvider services, LogEventLevel eventLevel = LogEventLevel.Information)
    {
        logger.WriteTo.Async(a => a.ApplicationInsights(
            services.GetRequiredService<TelemetryConfiguration>(),
            TelemetryConverter.Traces,
            eventLevel));

        return logger;
    }
```

```
public class IgnoreCommonDependenciesProcessor : ITelemetryProcessor
{
    private readonly ITelemetryProcessor _next;
    private readonly TelemetryOptions _telemetryOptions;

    private static readonly HashSet<string> HangfireCommandsToIgnore = new(comparer: StringComparer.InvariantCulture)
    {
        "sp_releaseapplock",
        "sp_getapplock",
        "set xact_abort on;set nocount on;",
        "SELECT SYSUTCDATETIME()",
        "Commit",
        "SELECT 1",
    };

    public IgnoreCommonDependenciesProcessor(ITelemetryProcessor next, IOptions<TelemetryOptions> telemetryOptions)
    {
        _next = next;
        _telemetryOptions = telemetryOptions.Value;
    }

    public void Process(ITelemetry item)
    {
        if (item is not DependencyTelemetry dependencyTelemetry)
        {
            _next.Process(item);
            return;
        }

        if (dependencyTelemetry.Type == "Azure Service Bus"
            && dependencyTelemetry.Name == "ServiceBusReceiver.Receive")
        {
            return;
        }

        if (dependencyTelemetry.Type == "SQL")
        {
            if (dependencyTelemetry.Data.Contains("HangFire]", StringComparison.InvariantCultureIgnoreCase) ||
                HangfireCommandsToIgnore.Contains(dependencyTelemetry.Data))
            {
                return;
            }

            if (!_telemetryOptions.EnableSqlCommandTextInstrumentation)
            {
                dependencyTelemetry.Data = string.Empty;
            }
        }

        _next.Process(item);
    }
}

```

```
public static IServiceCollection AddTelemetry(
        this IServiceCollection services,
        IConfiguration configuration
        )
    {
        var isApplicationInsightsEnabled = configuration.GetValue<bool?>("Logging:ApplicationInsights:Enabled") ?? false;
        if (!isApplicationInsightsEnabled)
            return services;

        services.AddApplicationInsightsTelemetry(options =>
        {
            options.ConnectionString = configuration.GetConnectionString("ApplicationInsights")
                ?? throw new InvalidOperationException("There is no connection string for Application Insights");

            // DeveloperMode disables batching of telemetry, that has a serious performance implication
            // since each telemetry trace will be sent immediately, instead of batches of 100-500 etc.
            //
            // https://learn.microsoft.com/en-us/previous-versions/azure/dn817703(v=azure.100)
            // https://learn.microsoft.com/en-us/azure/azure-monitor/app/configuration-with-applicationinsights-config
            options.DeveloperMode = configuration.GetValue<bool>("Logging:ApplicationInsights:DeveloperMode");

            options.EnableDebugLogger = false;

            // Sampling reduces the collected events by a percentage,
            // for evidence of sampling of logs run:
            //            
            // union requests,dependencies,pageViews,browserTimings,exceptions,traces
            // | where timestamp > ago(1d) and itemType == 'trace'
            // | summarize RetainedPercentage = 100/avg(itemCount) by bin(timestamp, 1d), itemType
            //
            // https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling
            options.EnableAdaptiveSampling = false;
        });

        services.ConfigureTelemetryModule<DependencyTrackingTelemetryModule>((module, o) =>
        {
            // Command text is used to filter out some traces, it will be scrubbed if false on app settings
            module.EnableSqlCommandTextInstrumentation = true;
        });

        services.Configure<TelemetryOptions>(configuration.GetSection(TelemetryOptions.OptionsPath));

        services.AddApplicationInsightsTelemetryProcessor<IgnoreBlazorStaticFilesProcessor>();
        services.AddApplicationInsightsTelemetryProcessor<IgnoreKnownPathsProcessor>();
        services.AddApplicationInsightsTelemetryProcessor<EnrichWithPropertiesProcessor>();
        services.AddApplicationInsightsTelemetryProcessor<IgnoreCommonDependenciesProcessor>();

        var enableRedisCommandMetrics = configuration.GetValue<bool?>("Logging:ApplicationInsights:EnableRedisCommandMetrics") ?? false;
        if (enableRedisCommandMetrics)
        {
            services.Decorate<IDistributedCache, DistributedCacheDependencyTracker>();
        }

        return services;
    }
```