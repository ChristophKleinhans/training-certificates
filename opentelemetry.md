# OpenTelemetry Certified Associate

## Fundamentals of Oberservability

**Telemetry Data**
Telemetry data are `traces`, `metrics`, `logs`, `baggage`

- *Traces*: traces gives us the big picture what is happening when a request is made to an application. Traces help us to understand the full path a request takes.
- *Metrics*: Is a measurement of a service captured in runtime
- *Logs*: Is the recording of an event, which can be structured or unstructured and with optional metadata
- *Baggage*: Is contextual information that is passed between signals. Its a key-value store. Definition of `context`: When Service A calls Service B, Service A includes a trace ID and a span ID as part of the context. Service B uses these values to create a new span that belongs to the same trace, setting the span from Service A as its parent. This makes it possible to track the full flow of a request across service boundaries.

**Semantic Conventions**
Are common names for different kinds of operations and data. They are one of the most important concepts in OpenTelemetry. Semantic Conventions define a standardized set of names and values for telemetry attributes (metadata) attached to spans, metrics, and logs. They ensure that different teams, libraries, and tools all "speak the same language" when describing telemetry data.
*Example*: Instead of one developer calling an attribute `http.url` and another calling it `request.url`, Semantic Conventions say: "For HTTP, you must call it `url.full`." This makes data portable and interoperable across the ecosystem.
 
- HTTP —> http.request.method, http.response.status_code, url.full, server.address
- Databases —> db.system, db.name, db.statement, db.operation, db.stored_procedure.name
- Messaging —> messaging.system, messaging.destination.name, messaging.operation
- RPC —> rpc.system, rpc.service, rpc.method
- Exceptions —> exception.type, exception.message, exception.stacktrace
- Resources —> service.name, service.version, host.name, cloud.provider, k8s.pod.name

- Stability Levels:
   - *Stable* —> production-ready, won't change in breaking ways.
   - *Experimental* —> may change; use with caution
   - *Deprecated* —> being phased out
 
- Based on the signal, the semantic convention aplly differently:
   - Resource attributes describe what is producing the telemetry (e.g., service.name, host.id)
   - Span attributes describe what happened during an operation (e.g., http.request.method)
   - Metric attributes (dimensions) describe how to slice/aggregate metrics
 ```yaml
 http.request.method  = "GET"
 url.full             = "https://example.com/api/users"
 server.address       = "example.com"
 server.port          = 443
 http.response.status_code = 200
 ```

- Semantic Conventions are defined by the OTel specification, not by individual SDKs


 **Instrumentation**

Instrumentation is the process of adding observability code to our application. For a system to be observable, it must be instrumented. The code from the system's components must emit signals (traces, metrics, logs). There are two ways to do that in OTEL, but both ways can also be used simultaneously:

1. *Code-based*: Use the OTEL APIs and SDKs to get telemetry from the applications directly.
2. *Zero-code*: When we cannot modify the application where we want to get the telemetry from. It provides telemetry data from the libraries we use. We can think about it lile "what's happending at the edges of the application". An agent or library hooks into our runtime/framework and generates telemetry automatically. Works by patching or wrapping well-known libraries (HTTP clients, DB drivers, frameworks, etc.)

<img width="556" height="385" alt="image" src="https://github.com/user-attachments/assets/205687ca-c57b-48d1-9061-881e34ced063" />

(3.) There are more ways to get telemetry data but the both methods above are the main approaches.

