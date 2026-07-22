# Rank by category

- ecommerce website has rankings
- items have rankings and one or more categories
- every time someone buys an item ecommerce would send a signal category, count
- ranking is number of items sold in 30 days period rounded to the nearest hour
- every time a ranking appears api call get the ranking

rank by category

- items can have one or more categories
(item, count)
10^9 items sold per month
10^8 per day
10 categories per item
10k total categories
10 x 10^6 unique items
global rankings

---------
# Functional Requirements
1. the system should show popular products in a category for the last 30 days (rounded to hour)
2. the system should show popular categories for the last 30days (rounder to hour)
# Non Functional Requirements
1. The system is eventually consistent. Rankings will be lag the sales by at most
    1 hour.
2. Ranking should be fault tolerant. Rankings should reflect the sales accurately.
3. The system should be highly available (4 nines SLA for ranking service).

# Entities and APIs
SaleEvent
- order_id
- item_id
- []category_id

A sale might contain multiple items and each item can be part of multiple categories.
The contract 
# BOE

10^8 sales per day
10k unique categories
10^7 unique products

we need,
1. top k products per category (last 30 days, rounded to hour)
2. top k categories (last 30 days, rounder to hour)

last 30 days means that we need to aggregate by 30 days sliding window

