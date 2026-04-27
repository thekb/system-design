# Prompt

Design a smart alarm system that can be installed across home devices, mobile phones,
and smart watches, where alarms can be set or deleted from any device 
but will only ring on the designated effective device.

# Clarifying questions 
- one off or recurring ?
    - both
- is one alarm to device mapping 1:1 or 1:many ?
    - 1:many
- are alarms only configured in the user context or are there any global alarms ?
    - only user context
- can the alarms overridden on the device ?
    - no
- scale ?
    - 1B users x 10 devices per user x 10 alarms per device
    - 100B active alarms
---

## Functional Requirements
1. users should be able to register devices
2. alarm can be set/deleted on 1 or more devices
3. alarm should be activated on a device in the configured time zone 
## Non Functional Requirements
1. system should maintain static stability, i.e. when there is a n/w partition between
    the device and the alarm service, the device should continue to hold its 
    last known state
2. the alarm service should be highly available, 99.99 SLA
3. the alarm service is eventually consistent with bounded SLA, i.e. changes made to the alarms should
    be propagated to the devices within a bounded (TBD) SLA. The SLA is only for
    devices online devices. Offline devices should catch up to the 
## Entities

Device
- id
- timezone
- user_id

Alarm
- id
- user_id
- time
- schedule
- devices
- version
- deleted_at

DeviceAlarmStatus
- device_id
- alarm_id
- version


## APIs
register_device(device)
set_alarm(Alarm)
delete_alarm(alarm_id)

get_alarms_for_device(device_token, []{alarm_id, version}) -> []Alarm

## HLD/LLD
### Alarm Service
- handles APIs and persistence
- adds a monotonically incrementing version whenever alarm changes
- this is needed to figure out if clients should be sent alarms which have
    changed
- persistence
    - partition devices/alarms on user_id using hash based partitioning
        - this ensures that the alarms and devices are distributed evenly across
            all the partitions
        - It will also ensure that devices/alarms for the same user will be in the
            same partition
        - alarm should have a version which should be increased monotonically when
            it is updated/deleted. 
            - We will use this to implement optimistic concurrency control to 
                prevent lost updates when concurrent requests try to update the
                same alarm.
            - Alarm clients will apply a alarm update on if the incoming version 
                is > the active version. This will ensure that an alarm read
                is alway monotonic from the clients POV, i.e. the client will never
                apply a value which is from the past compared to the current value.
        - Database choice
            - We are going to deal with 100B record potentially
            - The configured alarms are read more than they are updated
                - read >> write
            - we can use a partitioned relational datastore like postgresql or
                a NoSQL DB like dynamoDB
            - I would pick dynamo DB as it supports all the features we need
                - automatic partitioning of data
                - conditional updates
                - optionally we can enable DAX (caching) to speed up reads
                - use quorum writes, reads, w + r > n to ensure availability
- persistent connection considerations
    - Persistent
        - Pros
            - single channel to do all the necessary bidirectional IO per device
            - can satisfy < 1s latency SLA
            - send messages only on state change
        - Cons
            - potentially 100B active connections
            - connections are idle most of the time, we need to send periodic
                message to handle make sure that the client has not lost any
                messages.
            - Need stateful LB to ensure connections are evenly distributed
            - Need reasonably stable n/w connection
    - Polling
        - Pros
            - Completely stateless stack LB/app servers, simpler design
            - works in poor/unstable n/w conditions
        - Cons
            - Should process 100B requests every 30s/minute, due to stateless nature
              of the request/connection
    Should probably adopt hybrid approach, 
    - persistent connections for powerful clients with good n/w
    - fallback to polling
    - same RPC can be exposed over both persistent/polling

- metrics considerations
    - metrics we should be tracking
        - current alarm version vs last acked alarm version per alarm, device
            - bubble up disconnected client
            - device lag
        - alarm to_be_fired_at, fired_at per alarm, device
            - track firing delay
        - number of connected devices
        - get_alarms_for_device p99 Latency, cache hit rate
        - set_alarm p99 latency, contention
    - metrics are batched and sent by the device periodically

- security considerations
    - devices should present JWT for every request ->
        - get_alarms_for_device
        - metrics
    - device refresh JWT by presenting the device cert

- caching considerations
    - mostly small data, no need to CDN based distribution
    - If devices are spread across the globe, we can setup dynamodb replication
        across regions and use GSLB or anycast to steer clients to their nearest
        region
    - write operations are infrequent, so they can be sent to only one region. Data
        from that regions can be replicated to other regions.

- other execution models (not considered)
    - trigger alarms in the alarm service and notify client
        - does not work when client is offline, i.e. violates static stability
        - computationally intensive, client should always be online


### Alarm Client
- periodically polls for alarms
- updates local alarms and calls os apis to create/update/delete alarms
- alarm timezone is converted to devices timezone when alarm is created/updated
- for lower latency, we can keep a persistent connections between client and service
    to push changes
- should have local persistence to store currently active alarms
- all client side apply operations should be idempotent,
    - only apply alarm updates/deletes if incoming version > active version
    - when calling os alarm apis, ensure that state of the os alarm matches the
        desired state
- is os alarm supports callbacks, use them to notify the client so that we can
    send metrics on alarm triggers back to the alarm service
- if the os alarm service does not support callbacks, implement alarm subsystem
    in the client and call os apis to trigger notifications.
- 



