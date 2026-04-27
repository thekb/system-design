# Prompt

Design a service that can migrate large datasets from existing data sources to 
Google Cloud Platform, including task scheduling capabilities and 
efficient handling of very large file transfers.

# Clarifying Questions 
- how is the source data store ? blob store, file system, database ?
    - we should be able to support multiple source storage formats
- can we assume source and destination have identical access patterns or does
    the source access pattern is different from destination access pattern ? example
    is source and destination both blob stores or is source database and destination
    blob store ?
    - we can assume that both source and destination are in the same format 
- does the source support any kind of read from an offset kind of access ?
    - some of them do some of them don't
- how large can the files be ?
    - 10s of TBs
- is the migration one off or should the system detect changes and re sync
    the data ?
    - both, we can support watch functionality
- is there any ordering requirement between transfers ?
    - yes, some transfers have dependencies

# Functional Requirements
- the system should support transfers between source and destination (in gcp)
- source and destination will have logically similar storage abstractions
- transfers should support dependencies
- transfers can be triggered either as one off or they can automatically scheduled
    by watching the source for changes

# Non Functional Requirements
- the transfer should be at least once, with best effort idempotent overwrite
- the system should be available, 99.99 SLA
- when source and destination supports interruptable/resumable transfers, the system
    should support resuming an interrupted transfer
- the system should support < 30s latency for detecting changes in the source

# Entities 
Connector
- id

Transfer
- source: connector_id
- destination: connector_id

TransferStage
TransferStageStatus
TransferTrigger

TransferTask
- trigger_id
- transfer_id
- task_id
- stages: []TransferStage
- status: []TransferStageStatus

Trigger

# APIs

-- control plane apis --
CRUD /source/{id}
CRUD /destination/{id}
CRUD /transfer/{id}

POST /transfer/{id}/trigger
POST /transfer/{id}/task/{id}/{action=pause/resume/cancel}
GET  /transfer/{id}/task/{id} -> returns TransferTask object 

-- task workers apis --

# High level design

Control Plane API Server
- serves all the entity CRUD APIs
- persists data in the Database
- handles triggers APIs
- exposes webhook for receiving connector changes for connectors that support it
- idempotency
    - client supplied idempotency key
    - Create idempotency can be implemented using fail insert if exists
    - Update can be handled idempotently using conditional updates

DB
- authoritative source of truth for the state of the system
- The qualities we need in the datastore used for persistence is support
    for atomic and serializable transactions. The reason for this is to prevent
    lost updates during concurrent/conflicting writes. We can relax the 
    serializable transactions if the datastore supports atomic compare and set.
- Another consideration is the support for the kind of query patterns we need. In
    this case we need mostly point look ups for most of the operations served easily
    by either relational or non relations DBs. 
- Relational DBs make the GET List style APIs easier, although we can do the same
    with denormalizing data and range scans in nosql db like dynamoDB. 
- choice dynamodb as we can satisfy all of our requirements and can rely on a 
    managed service for partitioning/scaling/caching. 


Cron Service
- handles trigger schedules
- handles periodic connector health checks
- handles periodic connector polling for changes
- stateless
- uses DB as source of truth and for acquiring lock/lease on partitions of above
    for processing

Message Queue (kafka)
- for queueing messages for async processing of triggers / tasks / health checks / polling
- topics are partitioned are replicated across multi region for availability

ControlPlaneProcessor
- per topic consumer groups
- consumers process update state/status idempotently. even if duplicate messages
    are delivered or consumers crash, state will not get corrupted.
- after tasks are processed, the next task stages which need to be executed are
  put in an outbox table and a reference to the stage_id is put in a pub/sub
- 

TransferTask Worker
- workers pick up tasks from pubsub and acquire a lock/lease on the taskstage. This
    is done to prevent multiple workers from processing the same taskstage.

- the workers while it is executing the taskstage has to periodically renew the lease.
    This is used to monitor task execution liveness. If lease is not extended for
    an inactivity timeout, the taskstage/worker is expected to dead and the lease
    is free to be acquired by other free workers.

- push/pull
    - pull has natural back pressure control
    - pull, even if messages are lost or worker crashes after receiving message before
        ack, it will recover in the next poll.
    - push has low latency
    - more prone to message loss
    - hybrid is best to balance latency vs fault tolerance


# Low level design
- how to handle a large transfer
    - when is trigger is processed, the source connector is queried to find out the
        size of transfer
    - based on this if the connector supports resumable / ranged fetches, the task
        is split into chunks. Small chunks incur excessive metadata / processing
        overhead. Large chunk should be started from the beginning in case of
        failure. Pick a chunk size of ~100MB as a compromise between the two. 
    - each chunk is a transfer stage and can be processed concurrently.
    - we can define task level parallelism which controls how many chunks can 
        be downloaded in parallel.
    - each chunk should have a signature so that the chunk transfer can be 
        verified to mark completion.
    - after all the chunks are assembled, we can verify the signature of the
        final artifact to ensure that the transfer is successful.
    - if the connector does not support resumable / ranged fetches, we have to
        transfer the whole file at one. This means that if case of any errors
        we need to start from the beginning. 
    - each transfer task can be used for updating progress of the transfer or we
        need to periodically update the progress as a part of updating task lease.
    

# observability
- queue depth for the topics for consumer group autoscaling
- 
