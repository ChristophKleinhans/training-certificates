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
