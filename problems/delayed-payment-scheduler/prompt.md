# Delayed Payment Scheduler
Design a system that lets a user schedule a virtual-currency transfer (Robux) 
from one account to another, to be executed automatically at a specified future time. 

---
*Below the line*

The core deep-dive is on durable scheduling under failure, 
exactly-once execution despite retries, and how to keep the scheduler accurate 
when millions of payments are queued for the same wall-clock minute.

---

# Clarifying Questions
- Can the user cancel a scheduled payment ?
    - yes
- If the payment is scheduled for say 12.30, what is the acceptable margin for the 
    executing the actual transfer ? +- 5 seconds ?
    - yes
- What is the scale of scheduled transfers ?
    - 1 M/day
- How many unique users ?
    - 10 M
---

# Functional Requirements
The system should support,
- Scheduling payments between accounts
- Scheduled payments can be cancelled
- monitoring the status of a scheduled transfer

# Non Functional Requirements
- CAP -> consistency (system is not available during n/w partitions)
    - Payments transfers should be consistent, i.e. once a transfer is executed, 
        all the callers should see correct account balances.
    - API should be available (99.99 SLA)
- Fault tolerant - transfers should be executed correctly despite n/w, s/w failures
    - the system should make progress (correctly) despite n/w, s/w failures
- Scheduled transfer should be executed withing 5 seconds of the scheduled time.



# Entities and APIs

ScheduledPayment
- user_id (req headers)
- idempotency_key (req headers)
- payment_id (PK, server generated)
- source_account_id
- dest_account_id
- amount: decimal
- currency
- transfer_at
- status (PENDING, SCHEDULED, REJECTED, SUCCESS, FAILED)
- current_attempt_number

UNIQUE(user_id, idempotency_key)


ScheduledPaymentHold
- payment_id
- account_id
- amount
- currency
- status

PaymentScheduleOutbox
- payment_id
- created_at

If we want a detailed audit of the transfer attempts, we need this
ScheduledPaymentAttempt
- payment_id
- attempt_number
- claimed_by
- version
- status (PENDING, SUCCESS, FAILED, RETRY)

PK(payment_id, attempt_number)


POST /payment/schedule
-> {
 source_account_id
 dest_account_id
 amount
 currency
 transfer_at

}
<-
202 Accepted
{
    payment_id
}

GET /payments/schedule/{payment_id}
<- PaymentSchedule

# BoE
1 M/day -> 10/s
Assuming peak is 10x avg, peak QPS -> 100/s

Each payment scheduled results in around 10 queries to complete -> 1000 qps peak

This can be safely handled by one Postgres instance

Storage
data + index ~ 1 KB -> Daily 1 GB, so the DB size grows by ~400 GB per year
We can safely handle 1-5 TB with on PG instance

so for the sake of the design we don't need to think about sharding

however, if the scale grows by 10x, we will see ~10k QPS peak, which will tend to 
saturate a postgres instance, so we should look into sharding it

The core of the problem is efficiently figuring out and executing the transfers

# High Level Design



ScheduleAPI
- expose HTTP API
- stateless
- perform basic validations, accounts exists, input valid etc
- handles idempotency
- persist in DB, create outbox entry within the same TX
- outbox ensures that once the request is pending, the API can return safely
    without worrying about successfully producing to Kafka

PaymentScheduleOutboxPoller
- poll the payment schedule outbox, and produce to a kafka message queue (payment.scheduled.pending)
- removes the entries from outbox after successfully producing to kafka (RF=3, ACKS=ALL)
- kafka queue is partitioned by source_account_id
- pending queue will see peak 100 messages/sec and at 1KB pe message we are at 100 KB/s which
    is well below the conservative partition throughput of 10 MB/s.
- We should still partition the topic for consumer parallelism (num_partitions = 32 or 64 for headroom)
- There is only active poller, ensured by leader election, 

PaymentSchedulePendingConsumer
- create PaymentHold
    - deduct the amount from source_account_id and create a hold
    - ledger should reflect, deduct from source account and credit to hold account
    - if we cannot safely deduct the account, update the status to REJECTED with reason
    - if successful, update the status to SCHEDULED

PaymentSchedulePoller
- queries the DB and fetches payments which are supposed to be scheduled before the t + 5 minutes.
- peak QPS is 100/s -> we can have up to 3k schedules, we need to pick up
- store this in a redis sorted set
- poll every 30 seconds
- in a separate thread/process ZRANGE -inf to now, run this every second or 500 ms. 
- queue the payment_id in payments.scheduled.execute topic (RF=3, ACKS=ALL)
    - peak 100 qps, topic needs to be partitioned for for consumer parallelism
        - 32 or 64 partitions
        - partition on payment_id
        - ~~produce(payment_id,attempt_number)~~
        - produce(payment_id) -> hash(payment_id) should distribute the payments
            equally among all the partitions

PaymentTransferConsumer
- consumes from execute topic and execute the transaction and update status
- consumer has to claim the PaymentTransferAttempt to execute the transaction
- this ensures that only one consumer executes any payment transfer attempt
- as the transfer executions take very little time, the overhead for claims by workers
    might be over kill. but it preserves the retry attempts which is good for audit.
- claim process happens via OCC on status = PENDING and claimed_by is null, this ensures
    that only one consumer can claim any transfer attempt. This is useful if a consumer
    picks up a transfer and crashes before committing the offset to Kafka. 
- after claim, the consumer also updates a redis set which contains active transfers attempts
    being handled by the consumer.
- once the transfer is processed the status is updated and the active transfer attempt is removed
    from the set.
- consumer updates the heartbeat key in redis every 30s.
- Payment transfer can fail transiently (n/w, s/w issues) or permanently (account_id does not exist or
    something to that effect)
- transient errors create a new transfer attempt
- permanent errors fail scheduled transfer and create a new transfer to transfer from
    hold account to origin account.

PaymentTransferConsumerReaper
- A reaper runs every 30s and finds consumers which have not updated heartbeats for
    over 90s (3x missed heart beats). This worker is assumed dead and all the pending
    transfer attempts are marked failed and the attempt number if incremented and a new
    transfer attempt is created to be picked up the a new consumer.
- One instance is ensured via leader election.



# Low Level Design

# Observability
- DLQ
- Retries
- Redis CPU/Memory
- DB CPU/Memory/Storage
- Transfer Retries
- Transfer diff between transfer_at and attempt claimed_at -> schedule latency
- diff between kafka topic current offset - last committed offset (consumer lag)
- kafka partition IO (produce bps)
- kafka storage



