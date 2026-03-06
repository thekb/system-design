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
- the system should support fetching the relative position of a player from
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

GET /v1/ranks/{user_id}?count=<>

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
- query all the shards and compute topk and cache with TTl = 5 seconds
- use single flight to query and update on cache miss

relative position
- query scores of user friends and sort scores by relative position in the appropriate
   global time window sorted set
- cache and serve

# Low level Design