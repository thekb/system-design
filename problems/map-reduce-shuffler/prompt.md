# Prompt
Design a shuffler component for MapReduce that enables efficient communication 
and data transfer between Mapper and Reducer nodes. 
The system should handle data partitioning, sorting, and distribution across the cluster.

# Basic Idea

MapReduce Job has mapper and reducer tasks.

mapper 
- called once for every input record
- emits a list of key value pairs per input
- stateless
- mappers are partitioned by input file/file blocks

reducer
- takes key value pairs produced by mapper and collects all the values belonging
    to the same key
- reducer number of partitions are defined job author
- hash of the key is used to assign a key value to a reducer
- key value pairs are presented in sorted order 

shuffler
- partition the output of mapper 
- sort per partition
- send the output to reducers
- shuffle is a cross cutting concern, part of it happens on the mapper and part
    of it happens on the reducer
- map side concern
    - partitioning per reducers
    - sort per partition
- reducer side concern
    - copy partitioned data from mappers
    - k way merge the sorted data from mappers


# Functional Requirements
- partition mapper output in R (num reducers) partitions (map side)
- sort data by key within each reducer partition (map side)
- distribute mapper output to reducers (reduce side)
- merge and sort partitioned data (reduce side)

# Non Functional Requirements
- system should be scalable to support large mapper outputs and reducers
- system should be fault tolerant, i.e., the mapper output should be reliably delivered
    to reducers

# Entities
MapReduceJob
- id

MapReduceTask
- id
- job_id
- task_type: ENUM{Map | Shuffle | Reduce}
- current_attempt_number: int
- winning_attempt_id: None | str
- state: ENUM{ PENDING | RUNNING | SUCCEEDED | FAILED }

MapReduceTaskAttempt
- attempt_id: str
- task_id: str
- job_id: str
- attempt_number: int
- state: ENUM{ ASSIGNED | RUNNING | SUCCEEDED | FAILED | EXPIRED }
- worker_id: str
- started_at: datetime
- assigned_at: datetime
- lease_expires_at: datetime
- completed_at: datetime
- reason: None | str
- output: None | JSON

Why separate task from attempt
- same task may be launched multiple times due to worker crash, timeout, coordinator
    failure, network issues, if we don't have attempt history we cannot authoritatively
    answer,
    - which worker currently owns this task ?
    - did this task already succeed on another worker ?
    - which output should reducer trust ?


Mapper

Reducer

# APIs
-- control plane apis --
Note: attempt_id will be used as a fencing token to ensure that the task status
    is not updated because of a expired/previous attempts
GetTask(worker_id, capabilities) -> MRTask + attempt_id
HeartBeat(worker_id, job_id, task_id, attempt_id, status)
UpdateTaskStatus(worker_id, job_id, task_id, attempt_id, output)

-- data plane apis -- 
we need to bulk transfer data, hash/checksum would be read as part of the mapper
output

# High level design

Coordinator (Control Plane API)

Coordinator -> Worker Job assignment
- we can handle handle the task assignment to workers in two ways
    - push
        - pros
            - global view of worker assignments, potentially more efficient 
                scheduling decisions
            - lower scheduling latency, when a task becomes pending, it can be
                scheduled on a worker immediately
        - cons
            - scheduling message can lost, need explicit worker acks to ensure
                reliable delivery
            
    - pull
        - pros
            - natural back pressure control, worker only picks up task if it has
            capacity to do so.
            - scheduling is simpler, maintain a queue/priority queue of pending
                tasks and simply assign it to a worker when it asks
            - if message are lost, we can simply retries again
        - cons
            - scheduling latency is dependent on the frequency of worker polling
                for next tasks
    hybrid push + pull is best of both worlds, at the expense of more implementation
    complexity. I will start with pull and implement push later for improving
    scheduling latency.


Worker (Physical)

Mapper (Logical)

Shuffler (Logical/Reducer side)
- pull all the sorted partitioned mapper output for current reducer
    - control concurrency / bandwidth
    - verify with checksum after download is complete
    - if fails retry and fetch the partition output from beginning. 
    - write to scratch file and verify the signature. If passes move to 
        correct location
    - update status of shuffle for reducer when all the fetches complete successfully.
- merge all the output
- update state


Reducer (Logical)

# Low level design
