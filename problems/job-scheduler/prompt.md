# Distributed Job Scheduler

Design a distributed job scheduler that allows users to submit, schedule, and monitor jobs across many worker nodes.
The system should support one-time and recurring jobs, delayed execution, retries, and job cancellation.
It must guarantee that jobs execute at least once (or exactly once if possible) and scale to handle millions of jobs per day.