# Prompt

## Question
A server health monitoring system is an observability platform that collects heartbeats,
metrics, and logs from services; evaluates their health; and raises real-time alerts
when something goes wrong. 

Think of products like Prometheus, Datadog, and Amazon CloudWatch—operators use 
them to see dashboards, investigate anomalies, and get paged before incidents escalate.


## Context
Interviewers ask this to test if you can design high-ingest, low-latency, 
fault-tolerant systems for millions of sources.

They’re probing your ability to handle event streams at scale, 
control cardinality and cost, separate hot vs. cold paths, 
deliver timely and deduplicated alerts, and design for multi-tenancy 
and multi-region resilience. 

You should demonstrate clear trade-offs 

(push vs. pull, pre-aggregation vs. raw storage, at-least-once vs. exactly-once)
and an end-to-end plan from agents to alert delivery.

