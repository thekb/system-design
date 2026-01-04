# Problem
Trending Hashtags is a real-time analytics feature built into social platforms that surfaces the most popular hashtags over recent time windows, like "last 1/5/15/60 minutes" or the past hour. Think of Twitter Trends or Instagram Explore showing the top topics globally, by country, or within categories like sports and politics.

Interviewers ask this because it exercises your ability to design high-throughput streaming systems, compute top-K over sliding windows, and handle skewed traffic (hot hashtags) at global scale. It probes your understanding of event-time processing, approximate algorithms, partitioning to avoid hotspots, fault tolerance, and low-latency serving. Expect to balance accuracy vs freshness vs cost while supporting filters (geo/category) and user-specified windows.

