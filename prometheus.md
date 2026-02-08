# Prometheus

## Observability Concepts

**Metrics:** Are numerical measurements that represents a state and behavior of a system. Often time-series data with a timestamp which gives us insights in performance and health. 
F.i. count of requests, CPU usage, how many requests per second

**Logs:** Are text-based records that contain detailed information of what happened in our system. Logs answer the "WHY" did something fail/succeed. F.i. Database connection failed, Payment successfull

**Events:** Are occurrences or state changes that happen at a specific point in time. Events are similar to logs but typically more structured and often mark important moments. F.i. Service deployed, alert triggered, Pod restarted

**What are tracings and spans:** Tracking the journey of a single request as it flows through multiple services in a distributed system. They show the complete path and timing of operations across different components.
A trace consists of **spans**, while each span is an operation with start and end time, f.i. a database query or an API call.

**Push vs Pull:** 
- Push is when the Applications activly sent their data to a monitoring system, the App decides when to send
- Pull is when the monitoring system actively requests data from applications, often in intervals. The monitoring system in controling the timing

**Service Discovery:** Automatic detection of services and their network location. F.i. in Kubernetes services constantly start, stop, move, scale. Maintaining manually a list of IP addresses and ports would become impossible. Instead of telling Prometheus each target we tell Prometheus to f.i. discover all Pods and Prometheus will ask the Kubernetes API for the pods, services, resources in general.

**SLO:** Service Level Objective (SLO) are the definition of what level of performance or availability we promise to maintain.
**SLI:** Service Level Indecator (SLI) is the metric we measure, f.i. request success, latency, etc.
**SLA:** Service Level Agreement (SLA) is the contract with consequences if SLOs aren't met

## Prometheus Fundamentals

**System Architecture**

<img width="681" height="421" alt="image" src="https://github.com/user-attachments/assets/335bd178-07ad-4d72-a8ae-20fb66759fe5" />

- *Prometheus Server:*: Scrapes and stores the time-series data
- *Pushgateway:* Used for ephemeral and short-living jobs, which dont live long enough to get scraped. These can expose their metrics by pushing them to the Pushgateway
- *Jobs:* Prometheus Jobs are logical groupings of scrape targets that share the same purpose and configuration.
- *Exporters:* They help us exporting EXISTING metrics from a system as Prometheus metrics. Mostly used for systems where a direct instrumentalization is not feasible, f.i. HAProxy or Linux Sys Stats
- *Alermanager:* Simply handles the Alerts which are sent by the Prometheus Server. The Alertmanager takes care of dedublicating, grouping and routing


**Configuration and Scraping**

- *Configuration:* Prometheus is configured using command-line flags for immutable parameters (such as storage locations, amount of data to keep on disk and in memory, etc.) and a configuration file which describes everything related to scraping, jobs and which rule file to load. Prometheus is able to reload its configuration at runtime.
Example of a simple configuration including one job:
```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
```

Keep in mind, that the state of the system is scraped in this case in exactly 15 second intervals, when a peak is happening inbetween this 15 second interval, it will not be seen. It is always a snapshot of the system.

**Prometheus Limitations**
Prometheus is
- NOT FOR LOGS (Loki/Splunk for logs)
- NOT FOR EVENTS/TRACES (jaeger or zipkin for traces)
- pull-based means it can miss data between scrapes
- not suitable for billing or exact counts
- No built-in authentication/authorization
- No HA, single node
- No event-driven alerting, alerts are evaluated on intervals, not instantly

**Data Model and Labels:** 

- *Data Model:* Every metric is a timeseries. A timestamp and a value. The values are indentified by a metric name and labels (key-value pairs). Structured like this:
```
<metric_name>{<label_name>="<label_value>", ...} value timestamp

http_requests_total{method="GET", status="200", endpoint="/api"} 1547
```
While the Cardinality is the number of time series. Each Metric which differs in the name or label gets its own timeseries. So the Cardinality can reach millions (high cardinality) and is a performance problem.
=> We need to stick to Label Best Practices
=> `job` and `instance` (target address) is automatically labeled

- *Labels:* Are key-value pairs. Labels beginning with two underscores are reserved. Empty values in the labels are treated as if the label is not present.

**Exposition Format**

Metrics can be exposed to Prometheus using a simple text-based exposition format. All the client libraries implement this for us. This text-base format is line oriented and lines are seperated by the line feed character `\n`.
When prometheus scrapes `myapp:8080/metrics` it gets plain text:
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1543
http_requests_total{method="POST",status="200"} 421
http_requests_total{method="GET",status="404"} 23

# HELP memory_usage_bytes Current memory usage
# TYPE memory_usage_bytes gauge
memory_usage_bytes 524288000

# HELP request_duration_seconds Request duration
# TYPE request_duration_seconds histogram
request_duration_seconds_bucket{le="0.1"} 100
request_duration_seconds_bucket{le="0.5"} 250
request_duration_seconds_bucket{le="1.0"} 300
request_duration_seconds_bucket{le="+Inf"} 305
request_duration_seconds_sum 145.3
request_duration_seconds_count 305
```
while `# HELP` is a description and the `# TYPE` is the metric type (counter, gauge, histogram, summary).

The HTTP content type is: `text/plain; version=0.0.4` (A missing version value will lead to a fall-back to the most recent text format version.)


## PromQL

**Selecting Data:**

*Example:* We want to return all timeseries for the metric name `http_requests_total`
*Solution:* `http_requests_total`

*Example:* Return all timeseries for the metric `http_requests_total` and the filter by label `method=GET` 
*Solution:* `http_requests_total{method="GET"}`

*Example:* Return all timeseries for the metric `http_requests_total` and the filter by label `method=GET` and `status=200` 
*Solution:* `http_requests_total{method="GET", status="200"}`

**Rates and Derivates**

*Example:* Calculate the rate (change of the value per second) of the `http_requests_total` for the last 5 minutes
*Solution:* `rate(http_requests_total[5m])`

*Example:* Calculate the rate (change of the value per second) of the last two datapoints in a timewindow of 5 minutes of the `http_requests_total`
*Solution:* `irate(http_requests_total[5m])`

*Example:* Calculate the increase of the `http_requests_total` within the timerange of 2 hours
*Solution:* `increase(http_requests_total[2h])`

*Example:* Calculate the `rabbitmq_queue_messages` rate of change (value change in seconds) within the last 10 minutes.
*Solution:* deriv(rabbitmq_queue_messages[10m])

> [!NOTE]  
> the deriv is mainly used on gauge metrics, while rate, irate and increase mainly on count metrics.

**Agregation over time**

Lets us apply functions across a time range for each time series and return a single value
- avg_over_time(cpu_usage_percent[5m]) returns the average CPU over the last 5 minutes
- max_over_time(response_time_seconds[1h]) and min_over_time(disk_free_bytes[24h]) gives us the peak max/min over time
- count_over_time()
- stddev_over_time() or stdvar_over_time() gives us the variability
- quantile_over_time(), f.i. quantile_over_time(0.95, response_time_seconds[5m]) gives us the P95 latency over the 5 minute time window

**Aggregation over dimensions**

Instead of aggregating over time, we aggregate across different time series at a SINGLE POINT of time, which is the most recent value.

As example without label preservation:
- `sum(http_requests_total)`, `avg(cpu_usage_percent)`, `max(memory_usage_bytes)`, `min(disk_free_bytes)` adds all the time series together, removes all labels. Here f.i. all the HTTP requests or the load of all servers, etc.

Aggregation WITH label preservation:

Use the `by` or `without` syntax
- `sum(http_requests_total) by (status)` groups by status code and sums everything else
- `sum(http_requests_total) without (instance)` removes the instance label but keep everything else

**Aggregation at a specific point of time**

Using the `@` modifier
- `sum(http_requests_total) @ 1704067200` using the Unix timestamp
- `sum(http_requests_total) @ start()` at the beginning of time range
- `sum(http_requests_total) @ end()` at the end of the time range
- `sum(http_requests_total) @ (time() - 3600)` one hour ago
