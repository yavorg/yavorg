---
title: 'Analyzing SPCS application telemetry with Snowflake Trail'
date: 2025-05-12 10:10:00-08:00
featured_image: '/images/posts/spcs-observability.png'
excerpt: Let's walk through an example of how to use some awesome new Snowflake Trail observability capabilities, which make developing, debugging, and monitoring SPCS applications a breeze
tags:
- Snowflake
---

**Observability** is a key requirement we hear about a lot from Snowpark Container Services (SPCS) customers, whether they are *developing and debugging* their containerized workload, or when they are *monitoring* it in production or *troubleshooting* an outage. Let's walk through an example of how to use some awesome new Snowflake Trail observability capabilities with SPCS.

There are multiple types of telemetry that can be useful when working with containerized applications:
- **Application logs** - logs emitted by your application code 
- **Application metrics** - quantitative metrics assessing the performance of the user's code. This complements SPCS **Platform metrics**, which provide CPU/GPU/Memory usage, network/storage throughput, and other metrics about the infrastructure that hosts your containers
- **Application traces** - provide a detailed record of how requests flow through your application, helping you identify performance bottlenecks, understand dependencies between services, and troubleshoot complex issues that logs alone might not reveal.

Snowflake makes it easy to collect and durably store all this telemetry within a single central observability solution called [Snowflake Trail](https://www.snowflake.com/en/product/features/snowflake-trail/), and you can find a great overview [here](https://medium.com/snowflake/snowflake-trail-for-snowpark-is-now-generally-available-b769e6ab5007). There are two main Trail capabilities that we'll refer to:
- *Event Tables* - dedicated account-level tables where SPCS sends all collected telemetry
- *Traces & Logs* tab in Snowsight - a convenient UI for exploring traces and logs stored in Event Tables

In this post, we'll use a [stock API example](https://github.com/Snowflake-Labs/spcs-templates/tree/main/application-observability) hosted in SPCS, which offers the stock price, top gainers, and lets you look up the exchange where an individual stock is traded. Check out this video, or keep reading for more detail on how the app was instrumented:

<iframe width="560" height="315" src="https://www.youtube.com/embed/v6KkzfSaD8Y?si=bVO98CwipK0M00uA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Application logs
By default, SPCS collects any logs sent by your application to `stdout` and `stderr`. For example, in Python you may set up logging like so: 

```python
import logging

# Set up logger
logger = logging.getLogger("stock_snap_py")
logger.setLevel(logging.INFO)
ch = logging.StreamHandler()
ch.setFormatter(logging.Formatter("%(asctime)s;%(levelname)s:  %(message)s", "%Y-%m-%d %H:%M:%S"))
logger.addHandler(ch)

# Emit log line
logger.info(f"GET /stock-price - 400 - Invalid symbol")
```

Logs can be retrieved using the [`SYSTEM$GET_SERVICE_LOGS`](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#using-system-get-service-logs) function in SQL. Snowflake Trail offers a handy visual view of the same log data, with the ability to sort and filter by time period, as well as search the logs. 

![](/images/posts/spcs-observability-trail-logs.png)

*To use the Trail logs viewer, ensure your [account-level Event Table](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#using-event-table) is selected in the `Event Table` filter, and you have selected the correct Compute Pool and Service Name in the `Filters` drop-down.*

For more information on application logs, check out our [documentation](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#accessing-container-logs).

## Application metrics
At a high-level, customers frequently want to get a quantitative  view of how their containerized workload is doing, either for troubleshooting performance issues, or for getting an idea of utilization. 

SPCS supports automatic collection of [OpenTelemetry metrics](https://opentelemetry.io/docs/specs/otel/metrics/), *as long as your application is instrumented to emit them*. Instrumentation is generally pretty straightforward, using [standard OTLP clients](https://opentelemetry.io/docs/languages/). SPCS will emit the necessary environment variables that the clients use to know where to publish telemetry. Check out this Python example:

```python
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics._internal.export import PeriodicExportingMetricReader
from opentelemetry.sdk.resources import Resource
from opentelemetry.metrics import set_meter_provider, get_meter_provider

# OpenTelemetry setup for metrics
metric_exporter = OTLPMetricExporter(insecure=True)
metric_reader = PeriodicExportingMetricReader(exporter=metric_exporter, export_interval_millis=5000)
meter_provider = MeterProvider(metric_readers=[metric_reader], resource=Resource.create({"service.name": SERVICE_NAME}))
set_meter_provider(meter_provider)
meter = get_meter_provider().get_meter(SERVICE_NAME)

# Set up custom metrics
request_counter = meter.create_counter(
    name="request_count",
    description="Counts the number of requests"
)
response_histogram = meter.create_histogram(
    name="response_latency",
    description="Response latency",
    unit="ms"
)

# Capture application metrics
start_time = time.time()
# Do work
response_time = (time.time() - start_time) * 1000

request_counter.add(1, {"endpoint": STOCK_PRICE_ENDPOINT})
response_histogram.record(response_time, {"endpoint": STOCK_PRICE_ENDPOINT})
```

In certain cases, customers favor [Prometheus-style metrics](https://prometheus.io/docs/concepts/metric_types/) instrumentation, where metrics are being collected from the container, versus being published. To support this scenario, SPCS supports a [Prometheus sidecar collector](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#publishing-prometheus-application-metrics) to pull these metrics and store them in the account Event Table. 

In certain cases, application level metrics (how many times did my API get invoked) need to be paired with platform-level resource metrics (what's the network throughput) to get a complete picture and troubleshoot a performance bottleneck. In those cases, Application Metrics can be complementary with built-in SPCS Platform Metrics, such as CPU/GPU/Memory usage, and network/storage throughput ([full list](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#available-platform-metrics)). Enabling these metrics is as easy as updating the serivce spec. You can specify a metrics group instead of individual metrics, choosing between `system`, `network`, `storage`, [more info here](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#accessing-event-table-service-metrics).

```yaml
spec:
    containers:
      ...
    platformMonitor:
      metricConfig:
        groups:
        - system
```

Metrics can be complex to aggregate and display, so we enable easy querying directly against Event Tables. Here is how to query for Application metrics ([more info](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#accessing-application-metrics-and-traces-in-the-event-table)):

```sql
SELECT timestamp, cast(value as int)
  FROM <current_event_table_for_your_account>
  WHERE timestamp > dateadd(hour, -1, CURRENT_TIMESTAMP())
    AND resource_attributes:"snow.service.name" = 'STOCK_SNAP_PY'
    AND scope:"name" != 'snow.spcs.platform'
    AND record_type = 'METRIC'
    AND record:metric.name = 'request_count'
  ORDER BY timestamp DESC
```

Here is how to query for a specific Platform metric ([more info](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#accessing-event-table-service-metrics)):

```sql
SELECT timestamp, cast(value as float)
  FROM <current_event_table_for_your_account>
  WHERE timestamp > DATEADD(hour, -1, CURRENT_TIMESTAMP())
    AND resource_attributes:"snow.service.name" = 'STOCK_SNAP_PY'
    AND scope:"name" = 'snow.spcs.platform'
    AND record_type = 'METRIC'
    AND record:metric.name = 'container.cpu.usage'
  ORDER BY timestamp DESC
```

## Application traces
Similar to Metrics, SPCS supports collecting [OpenTelemetry traces](https://opentelemetry.io/docs/concepts/signals/traces/) out of he box, as this Python example shows:

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from snowflake.telemetry.trace import SnowflakeTraceIdGenerator

# Service name
SERVICE_NAME = "stock_snap_py"

# OpenTelemetry setup for tracing
trace_id_generator = SnowflakeTraceIdGenerator()
tracer_provider = TracerProvider(
    resource=Resource.create({"service.name": SERVICE_NAME}),
    id_generator=trace_id_generator
)
span_processor = BatchSpanProcessor(
    span_exporter=OTLPSpanExporter(insecure=True),
    schedule_delay_millis=5000
)
tracer_provider.add_span_processor(span_processor)
trace.set_tracer_provider(tracer_provider)
tracer = trace.get_tracer(SERVICE_NAME)

# Emit traces
with tracer.start_as_current_span("get_stock_exchange") as span:
    with tracer.start_as_current_span("validate_input") as child_span:
        # If validation fails
       span.add_event("response", {"response: Invalid symbol")})
 
    with tracer.start_as_current_span("fetch_exchange") as child_span:
        # Fetch stock exchange for given symbol from Snowflake table, traces will propagate between SPCS and the Warehouse

    span.add_event("response", {"response: Successfully fetched exchange for symbol"})
```

*To correctly display SPCS traces in the Trail Traces viewer, please ensure the `SnowflakeTraceIdGenerator` provided in `snowflake.telemetry.trace` is configured as the trace ID generator*

Once the application is instrumented correctly, traces will be stored in the account-level Event Table, just like the application logs and metrics. You can use the Trail Traces viewer to analyze these traces. Snowflake supports trace propagation between SPCS and some workloads that run in the Warehouse, so you will see continuous trace information if you call into Python/Java stored procedures, and others coming soon.

![](/images/posts/spcs-observability-trail-traces.png)

For more information, see [this section of our docs](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#publishing-otlp-application-metrics-and-traces).


### Next steps
👩‍💻 Check out the [stock API code sample used above](https://github.com/Snowflake-Labs/spcs-templates/tree/main/application-observability)<br/>
📚 Review our  [documentation](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services)<br/>
📣 Let us know what you think by responding to this post