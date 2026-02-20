# OpenTelemetry Certified Associate

## Fundamentals of Oberservability

**Telemetry Data**
Telemetry data are `traces`, `metrics`, `logs`, `baggage`

- *Traces*: traces gives us the big picture what is happening when a request is made to an application. Traces help us to understand the full path a request takes.
- *Metrics*: Is a measurement of a service captured in runtime
- *Logs*: Is the recording of an event, which can be structured or unstructured and with optional metadata
- *Baggage*: Is contextual information that is passed between signals. Its a key-value store. Definition of `context`: When Service A calls Service B, Service A includes a trace ID and a span ID as part of the context. Service B uses these values to create a new span that belongs to the same trace, setting the span from Service A as its parent. This makes it possible to track the full flow of a request across service boundaries.

**Semantic Conventions**
Are common names for different kinds of operations and data

- *Trace semantic conventions*: 
