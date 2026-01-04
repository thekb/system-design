# Prompt

# Question
A real-time game leaderboard is a ranking service built into an online game that
continuously ingests players' scores and lets them see how they stack up globally
and among their friends. 

Think of the leaderboards you see in Fortnite, Call of Duty, or a coding competition platform.
Players expect their rank to update within seconds after each match.


# Considerations
Interviewers ask this question to see if you can scale ultra-high write throughput,
avoid hot shards, and deliver low-latency reads such as K-neighbors around a user.

They’re testing your ability to combine streaming ingestion, in-memory indexing,
sharding, and cache recovery while making the right consistency and correctness
trade-offs. 

Expect to justify how you’ll keep updates near real-time without sacrificing
availability or blowing up costs.

