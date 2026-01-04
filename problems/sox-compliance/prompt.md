# SOX Compliance

## Context
- Hubspot is moving from subscription based billing to usage based billing
- Historically only financial/fintech infra is under SOX controls.
- With usage based billing product infra (services and APIs) that can **affect 
    or record usage events** will be under SOX scope.
## Scale
- The product infra has **~10^3** of services and **~10^4** endpoints.
- Centralized usage meter that aggregates service level usage data into
    billable metrics
- Service graph that maps dependencies and relationships between 
    **deployables**.
- Request tracing and API cataloging tools (with varying completeness)
- Messaging and queuing systems
- Autonomous teams, who independently deploy and evolve services

## Task
- Determine which services and endpoints have a material impact on customer
    invoicing, so that we can apply right level of SOX controls - 
    only where necessary; minimizing friction while maintaining compliance
    and trust.