# Introduction
## Why OpenTelemetry?
As ecosystems grow, applications invariably get tied into services that handle logging, tracing and observability data. An example of this is using datadog specific agents for collecting data from containers and using datadog specific tracers etc. 

OpenTelemetry offers an easy platform agnostic way to collect all this data. Instead of being tied to a specific platform, you can migrate from one to another as you please. You can even use multiple platforms.

# Instrumenting an application
## What we collect?
For this example, I'm gonna utilise .NET to illustrate how instrumentation works. There are three key types of data that we collect. 

1. Logs - Logs are pretty much a standard we are used to. They're a captured piece of data at a point in time.
2. Traces - These are spans of time that can have multiple child spans that illustrate an event that occurred and the length of time taken. These traces can be distributed and tracked across multiple systems provided they're instrumented for it.
   
    Traces also have ids that can be directly linked to associated logs.


        Causal relationships between Spans in a single Trace

                [Span A]  ←←←(the root span)
                    |
            +------+------+
            |             |
        [Span B]      [Span C] ←←←(Span C is a `child` of Span A)
            |             |
        [Span D]      +---+-------+
                    |           |
                [Span E]    [Span F]
[Source of Illustation](https://opentelemetry.io/docs/reference/specification/overview/)

3. Metrics - Metrics are timings of specific events that can be used to capture statistical data and observability. Eg how long a request takes or CPU info at a point in time.

## Integrating instrumentation
Most of the setup in an application is pretty streamlined and gets integrated in the startup of the application

### Traces

The below is sample configuration for one of our applications. We setup a tracer as one of the services in .NET. 


        services.AddOpenTelemetryTracing(options =>
        {
            options.AddSource("Email.RiskScore.Tracing");
            options.SetResourceBuilder(rb).SetSampler(new AlwaysOnSampler()).AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddMySqlDataInstrumentation(settings => settings.SetDbStatement = true);
            options.AddOtlpExporter(otlpOptions => { otlpOptions.Endpoint = new Uri(endpoint + "/api/v1/traces"); });
        });

In this case, I've included AspNet instrumentation which capture dotnet related metrics including things like garbage collection. On top of that, we're capturing each outgoing http request and all sql requests. This allows for a far more accurate picture to be painted of what the requests are doing or how much time they're taking out of the application flow.

On top of that, it is also possible to create custom traces. Firstly a static activity source needs to be created like below. After this is done, you can access it and use it to write traces.

    public static ActivitySource Writer = null;
    public static void Init(string tracerName)
    {
        Writer = new ActivitySource(tracerName,"1.0.0");
        var listener = new ActivityListener()
        {
            ShouldListenTo = _ => true,
            Sample = (ref ActivityCreationOptions<ActivityContext> _) => ActivitySamplingResult.AllData
        };
        ActivitySource.AddActivityListener(listener);
    }

Following this, introduce a trace around an operation. In the case of retrieving data, we mark the activity as `Client` cause we are retrieving data. `Server` and `Internal` are other markers we use. On top of that, we can store tags with associated info eg parameters.

    var activity = SharedTelemetryUtilities.Writer?.StartActivity("get_phone_data", ActivityKind.Client);
    //Some operation takes place for a length of time
    activity.AddTag("param", paramValue)
    activity?.Stop();

These end up building into existing traces with the capturing of additional data. 

In terms of accessing traces, the optimal way is to embed the trace id within your logs. Then you can see the associated traffic with the logs. Tooling such as grafana tempo or Zipkin can produce a flame graph with a pretty nice breakdown of the request.


### Metrics
Metrics allow for statistical data to be captured, this can be used in conjunction with tooling such as prometheus.


        services.AddOpenTelemetryMetrics(options =>
        {
            options.SetResourceBuilder(rb)
                .AddAspNetCoreInstrumentation().AddHttpClientInstrumentation();
            options.AddMeter(SharedTelemetryUtilities.MeterName);
            options.AddView(TelemetryProperties.RequestMeter, new ExplicitBucketHistogramConfiguration()
            {
                Boundaries = { 50, 75, 100, 250, 500, 1000, 2000, 5000, 10000 }
            });
            options.AddOtlpExporter(otlpOptions => { otlpOptions.Endpoint = new Uri(endpoint + "/api/v1/metrics"); });
        });


Setup is similar to traces. The one slightly different piece is the inclusion of the view. This allows for non standard buckets to be utilised to track our request duration, we include the name of the instrumentation so it knows which to apply it to. On top of that, we've introduced some tracking around duration of httpclient requests and other aspnet functionality.

            Histogram<double> histogram = null;
            var stopwatch = Stopwatch.StartNew();
            histogram = SharedTelemetryUtilities.Meter.CreateHistogram<double>(TelemetryProperties.RequestMeter);

            //Operation occurs here.
            histogram.Record(stopwatch.Elapsed.TotalMilliseconds);

Similar to the trace, we can access a static meter and create our prefered meter with that. Histograms make sense for statistical info but counters and other tools are included in meter depending on one's needs. Once again, we're still using built in dotnet functionality to measure this data.

Below  is such data being sent to Grafana, the particularly great thing about capturing metrics like this is your dashboards tend to load instantaneously cause it's not attempting to parse through logs.

![](img/sample-dashboard.png)


## Collecting the data
So one way in which we're being platform agnostic is that built-in dotnet tooling is utilised. The next is the collector, tools such as Grafana or Datadog have their own agents that tend to run alongside your containers. In the world of opentelemetry, we can let these agents die the deaths they deserve.


The [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector-contrib) needs to run alongside your application similar to the agents. In the example below, we send metrics to a prometheus instance and traces to Tempo, both based in Grafana Cloud, I've obfuscated api keys.



    receivers:
        prometheus:
            config:
            scrape_configs:
            - job_name: 'otel-collector'
                scrape_interval: 10s
                static_configs:
                - targets: ['0.0.0.0:8888']
        otlp:
            protocols:
            grpc:
            http:
    processors:
        batch:
            timeout: 10s
        resourcedetection:
            detectors: [ecs,system,env]
    exporters:
        otlp:
            endpoint: tempo-us-central1.grafana.net:443
            headers:
            authorization: Basic [ApiKey]
        prometheusremotewrite:
            endpoint: https://364320:api_key@prometheus-prod-10-prod-us-central-0.grafana.net/api/prom/push
            external_labels:
            env: dev
            region: us-west-2    
            app: sample-app
    extensions:
        health_check:
    service:
    extensions: [health_check]
    pipelines:
        traces:
            receivers: [otlp]
            processors: [batch, resourcedetection]
            exporters: [otlp]
        metrics:
            receivers: [otlp,prometheus]
            processors: [batch,resourcedetection]
            exporters: [prometheusremotewrite]

Now I'll break down this config to understand each piece of it.

### Receivers

Receivers is effectively the point where you configure what will be utilising this data. In this case, I've configured it to utilise prometheus, we're also utilising OTLP itself which is an opentelemetry standard. In addition to this we've configured prometheus to scrape data about the collector itself.

    receivers:
        prometheus:
            config:
            scrape_configs:
            - job_name: 'otel-collector'
                scrape_interval: 10s
                static_configs:
                - targets: ['0.0.0.0:8888']
        otlp:
            protocols:
            grpc:
            http:

If we wished to expand this to use other tooling, it would just be a matter of adding it. Some require no config eg `zipkin:` would be a default config for zipkin. The collected data will no go anywhere though unless it's configured at the exporter level.

#### Processors
My utilisation of processors is pretty basic. Processors operate on the data itself but also can further enrich based on information gathered.

 [`Batch`](https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/batchprocessor/README.md) allows for metrics, traces etc to be batched into bundles. The default is 8192 and then it will send, the timeout is the max time before sending regardless. I'd highly recommend utilising batches if you're dealing with high volumes of traffic.

        batch:
            timeout: 10s
        resourcedetection:
            detectors: [ecs,system,env]

[Resource detection](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/resourcedetectionprocessor/README.md) gathers data around the kind of system it's operating in. Eg in this case it will gather standard machine info but also Amazon ECS specific and any associated environment variables.