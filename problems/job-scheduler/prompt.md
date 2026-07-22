# Distributed Job Scheduler

Design a distributed job scheduler that allows users to submit, schedule, and monitor jobs across many worker nodes.

---
## below the line
- The system should support one-time and recurring jobs, delayed execution, retries, and job cancellation.
- It must guarantee that jobs execute at least once (or exactly once if possible) and scale to handle millions of jobs per day.

---
## Clarifications
1. Are the jobs scheduled on demand or can they be scheduled ahead of time ?
2. How precisely should the jobs be executed ahead of time ?
3. Should the jobs be retried on failure ?
4. Can the jobs be cancelled ?
5. What is the job execution semantics ? at least once, exactly once or at most once ?
6. What is the scale of jobs ?
7. How is the job status monitored ?
8. How is the job described ? Can we assume that the job is packaged in an executable 
    format ?
9. Do the jobs specify any affinity for the nodes they should be executed on ?
10. Can the jobs have any dependency between them ?
11. Are the scheduled jobs recurring ?
12. Are the workers and the scheduler within the same network ?
13. Should the users be able to see all the jobs they have submitted ?
---

## Functional Requirements
1. The system should support one time, recurring jobs.
2. Jobs can be scheduled into the future.
3. Scheduled Jobs can be cancelled.
## Non Functional Requirements
1. Job submission should be highly available (SLA 4 nines).
2. Job execution should be at least once (failed jobs are ). 
3. Jobs should be scheduled for execution within 5 seconds of their expected
    execution time.
## BoE
### scale numbers
- 1000 jobs submitted per second
- 1 year of execution history
- 1 KB per job (configuration + status)
### Storage requirements
- 1 MB/s write
- 10 GB per day, ~ 5 TB per year

## Entities and APIs
### Job (configuration)
- id (pk, client generated)
- version (server generated monotonically increasing)
- metadata

### Job Version (configuration)
- id
- version (id, version -> PK)
- metadata
- spec
### Task (execution instance)
- job_id
- job_version
- task_id
- current_attempt_number
- run_at
- status (PENDING|IN_PROGRESS|SUCCESS|FAILED|CANCELLED)

### TaskAttempt
- task_id
- attempt_number {pk = (task_id, attempt_number)}
- status (PENDING|IN_PROGRESS|SUCCESS|FAILED)
- claimed_by (executor_id)

## High Level Design
### Admission
- handles job configuration + config validation
- reject invalid configurations
- manage monotonic job versions
- create -> idempotent create on client supplied job_id; version = 1
- update -> create new version, OCC on current version
- versioned job config, ensure that the configuration referred by a task is always
    stable
- http api server + lb in front
- backed by strongly consistent DB (need multi table/collection TX)
    - Postgres/Dynamo/Spanner
- ensure consistent job configurations

### Scheduling
- task is a runnable instance of a Job
- task refers to a stable version of job
- responsible for creating tasks for jobs which are ready to run
- kinds of scheduling
    - on demand (run now)
    - delayed scheduling (run at)
    - recurring / periodic scheduling (run every)
- all of the different kinds of scheduling can be normalized to run this task if
    run_at <= current time.
- manages the number of attempts/retries for a task
- ensures that only one executor can claim/execute the task at any given point of 
    time
- task state needs to be persisted in a strongly consistent DB to guarantee 
    the execution semantics
    - Postgres/Dynamo/Spanner

### Execution
- Executors are responsible for executing a pending task
- executors should claim a task before they can execute the task
- executors need to periodically update the progress of the task to distinguish
    between a task which a taking a long time to complete vs a executor which
    has gone offline(fault)
- the scheduler will release the claim of 

## Low Level Design / Deep Dives

