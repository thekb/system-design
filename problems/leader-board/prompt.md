# Prompt

# Question
A real-time game leaderboard is a ranking service built into an online game that
continuously ingests players' scores and lets them see how they stack up globally
and among their friends. 

Think of the leaderboards you see in Fortnite, Call of Duty, or a coding competition platform.
Players expect their rank to update within seconds after each match.

# Clarifications
 - is there any time dimension for the leader board ? daily, weekly etc ?
    - yes, we want to display daily, weekly and monthly leaderboard
 - should we consider the daily, weekly etc as tumbling windows or sliding windows ?
    - tumbling windows 
 - how many daily active users and how many games are played per day ?
    - 100M DAU, playing 10 games a day on avg
 - what is the size of leader board ?
    - the leader board includes all the players

# Functional Requirements
- the system should support querying top k players from the global leader board
- the system should support fetching the relative position +- 5 of a player from
    their friends list

# Non Functional Requirements
- the leader board should be updated within 5 seconds after a game is completed
- the system should be highly available

# BOE
avg 10^9 games per day =~ 10^4 writes/s =~ 20k writes/s peak
assuming users look at the leader board after the game, potentially 
same amount of reads

# Entities
User

UserFriends

Game

GameScore
   - game_id
   - user_id
   - score
   - completed_at


# APIs

GET /v1/top_k?window=<>,&day=<>&k=<>

GET /v1/ranks/{user_id}/friends

# Design


# How to store the leader board
   - considerations
      1. ~10k rps
      2. leader board is not the system of record for the game scores, it should
         just keep the leader board

   - options
      1. write optimized DB columnar DB / OLAP
         - pros 
            - easy to absorb write volume
            - up to date data
         - cons
            - full table aggregation for every query. 
            - unable to support 10k rps
      2. in memory data structure (?redis) representing leader board
         - pros
            - fast access
         - cons
            - need to maintain sorted data structure (log(N) best case) for write. 
      
      decision:
      - use redis for storing the leader board
   
   keep N redis shards partitioned by user_id

   store in a redis sorted set for a given time window

   rank tie breaking with in the same time window
   - score > latest completed_at > user_id

# How to handle writes
- game completed events are produced to a message queue topic partitioned by
   user_id
- completion processor consumes the message and updates the appropriate redis
   shard in the appropriate time window
- keep the highest score for the user in the time window


# how to handle reads
## topk 
- query all the shards top k + delta and compute topk for the combined resuls
- cache the results with ttl of 5 seconds
- cache should be read through
   - use single flight to query and update on cache miss, to reduce calls to
      upstream
- cache can be replicated asynchronously to to scale reads, at the expense of
   potentially state results or there can be multiple instances of cache
   servers independently serving results
- clients can be assigned to cache shards via consistent hashing to ensure that
   the results they see don't flip on refresh as a result of replication lag
- measure rps, p99 latency, cache hit rate, upstream refresh duration
- Another approach is to have a service periodically update the cache with
   updated topk. This will ensure that the system is available and serving
   stale topk even when upstream is unavailable.
# relative position
- for a give user, 
   - find all friends
   - group friends by partitions
   - issue one request per shard to find score of all the friends
   - gather and sort the results and cache with ttl

- read through cache
- avg rps will be low, cache hit rate might be low due to low avg rps
- when a user's get a new score, broadcast invalidation to all the friends
- measure hit rate, 