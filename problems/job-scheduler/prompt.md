# Distributed Job Scheduler

Design a distributed job scheduler that allows users to submit, schedule, and monitor jobs across many worker nodes.

---
## below the line
The system should support one-time and recurring jobs, delayed execution, retries, and job cancellation.
It must guarantee that jobs execute at least once (or exactly once if possible) and scale to handle millions of jobs per day.

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