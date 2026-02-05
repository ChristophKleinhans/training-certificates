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

Continue

