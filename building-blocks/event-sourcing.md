# Event Sourcing

## Core Idea of Event Sourcing

### Traditional model:

We store the current state directly (e.g. SQL row: balance = 1000).

### Event Sourcing model:

We store every state change as a series of immutable events.

Current state is derived by replaying those events.

| Timestamp | Event          | Data                |
| --------- | -------------- | ------------------- |
| 10:00 AM  | AccountCreated | {id: 1, balance: 0} |
| 10:05 AM  | DepositMade    | {amount: 500}       |
| 10:10 AM  | WithdrawalMade | {amount: 200}       |


If you replay these events → you get balance = 300.

## Why do this?
* Auditability: full history of everything that happened.

* Reproducibility: you can rebuild state at any point in time.

* Temporal queries: “what was the balance on June 10th at 3 PM?”

* CQRS compatibility: easier to separate read/write models.

* Debugging: recreate bugs by replaying events in staging.

## How state reconstruction works
You define a reducer / materializer that knows how to apply each event to an in-memory object.

```python
class Account:
    def __init__(self):
        self.balance = 0
    
    def apply(self, event):
        if event.type == "AccountCreated":
            self.balance = 0
        elif event.type == "DepositMade":
            self.balance += event.data['amount']
        elif event.type == "WithdrawalMade":
            self.balance -= event.data['amount']
```

To get the current state:

```python
account = Account()
for event in event_store.get_events(account_id):
    account.apply(event)
```

## Where are events stored?
* Could be an append-only log (Kafka, Pulsar, EventStoreDB, custom).
* Could be a table (SQL, NoSQL) with strict append semantics.

Must guarantee ordering per entity/aggregate.

## Scaling Considerations
* Snapshotting: Periodically store the latest state after N events to avoid replaying millions of events.

* Sharding: Partition events per entity (aggregate ID) to scale horizontally.

* Compaction: In some systems, compact old events if full replay isn’t necessary.

* Schema evolution: Must handle versioning for events.

## Real World Analogs
* Kafka log compaction is event sourcing at infra level.
* Git is a kind of event sourcing (commits = events).
* Financial ledgers.
* CDC (Change Data Capture) pipelines.

## Common Architectures
Command Bus → Event Store → Materialized Views

* Separation of write model (command handlers) from read model (precomputed views for queries).
* You often see event sourcing combined with CQRS.

## Tradeoffs
| Pros                       | Cons                          |
| -------------------------- | ----------------------------- |
| Full audit trail           | Harder mental model           |
| Temporal queries           | Schema evolution complexity   |
| Debuggable                 | Event versioning              |
| Flexibility in read models | Can be overkill for CRUD apps |








