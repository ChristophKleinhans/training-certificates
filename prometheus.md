# Prometheus

## Observability Concepts

**Metrics:** Are numerical measurements that represents a state and behavior of a system. Often time-series data with a timestamp which gives us insights in performance and health. 
F.i. count of requests, CPU usage, how many requests per second

**Logs:** Are text-based records that contain detailed information of what happened in our system. Logs answer the "WHY" did something fail/succeed. F.i. Database connection failed, Payment successfull

**Events:** Are occurrences or state changes that happen at a specific point in time. Events are similar to logs but typically more structured and often mark important moments. F.i. Service deployed, alert triggered, Pod restarted

**What are tracings and spans:** Tracking the journey of a single request as it flows through multiple services in a distributed system. They show the complete path and timing of operations across different components.
A trace consists of **spans**, while each span is an operation with start and end time, f.i. a database query or an API call.
