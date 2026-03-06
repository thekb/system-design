# Design a system to rollout new versions of a mobile OS to devices worldwide

Design a system that can securely distribute mobile OS updates to millions of 
devices worldwide, handling different device types, network conditions, 
and rollout strategies like staged deployments.

# Functional Requirement:
1. system should support staged rollout to billions of devices
2. system should have the ability to pause rollout.
3. system should provide visibility into rollout progress. Visibility is near real
    time aggregates + 

# Non Functional Requirements:
1. availability >> consistency. System should provide an SLA of 99.99% uptime. 
2. versions should be delivered under varying n/w conditions.

# Assumptions:
1. No carrier specific opt outs. All the devices are assumed to be identical.


# Entities
Release
- id
- version_metadata
  - release_notes
- artifacts:
    - manifest_url
- constraints

RolloutPlan
- id
- stages: []RolloutPlanStage
- status:
  - stages: []RolloutPlanStageStatus

RolloutPlanStage
- id
- percentage

RolloutPlanStageStatus

DeviceState
- device_id
- device_info 
- version_info


# APIs

rpc CheckForUpgrade(CheckForUpgradeRequest) returns (CheckForUpgradeResponse)

CheckForUpgradeRequest:
- device_id
- device_info
    - battery %
    - storage left
    - current_version

CheckForUpgradeResponse:
- upgrade_offered: bool
- rollout_token: JWT token
- artifact_url: CDN url 
- retry_after: 

rpc SendUpgradeProgress(SendUpgradeProgressRequest) returns (Empty)

SendUpgradeProgressRequest
 - device_id
 - release_id
 - rollout_token
 - status: downloading| downloaded | install_pending | installed | failed

---
Core Problems

1. How to stage the rollout
    - stage 1 - N, each with a rollout percentage, ex 1, 4, 10, 20, etc
    - only one stage can be active at any given point of time
    - devices will periodically or on demand call the CheckForUpgrade
    - UpgradeService is stateless service which computes the upgrade eligibility
    - upgrade eligibility = rollout_predicate(device_info) AND hash(device_id, release_id) % 100 < current stage percentage
    - response with CheckForUpgradeResponse
    - emit upgrade offered event for device, stage, rollout
    - if multiple stages can be active
        - we need to add priority to stages and pick the matched stage with highest priority
    
2. How do we track the progress
    - devices send telemetry to the regional telemetry gateway
    - telemetry gateway is secured by mTLS
    - the gateway throttles the incoming telemetry based on device_id/ip
    - gateway pushes the incoming telemetry to a durable queue
    - gateway dedups the telemetry using (device_id, release_id, status) so that
        we see only on unique status per device
    - workers consume the telemetry from queue and keep per stage aggregates of
        devices status
    - this aggregates are used to track per stage progress of a rollout

3. How is the version delivered to the devices
    - when upgrade is offered to device, the response contains the artifact url and a rollout_token
    - rollout token is JWT which grants the device authorization to download the upgrade
    - rollout token is validated by the edge workers at the CDN edge
    - token validation:
        - signature check
        - expiry
        - audience -> download_url, release_id, device_id hash
    - edge checks
        - token validation
        - release_id not in paused_release_ids (from release config snapshots)    
    - after the edge check is completed, the device is allowed to download the upgrade metadata file, which
        contains the upgrade chunks and their signatures.
    - the device downloads the upgrade version, chunk by chunk and verified the
        downloaded chuck against the chunk signature. 
    - If the signature does not match, the chunk is downloaded again.
    - after all the chunks are downloaded, the device assembles the upgrade
        and verifies the signature of the assembled upgrade version.
    - the upgrade is applied after this.
    - device sends the upgrade progress to the regional telemetry gateway

4. How do we pause the rollout
    - we need to stop the progress in 2 places
        - CDN edge: stop new downloads
        - Upgrade service: don't offer the upgrade anymore
    - upgrade service maintains immutable rollout config snapshot with monotonically
        increasing versions
    - snapshots are pushed to a blob store
    - when rollout should be paused, a message will be pushed to all the upgrade
        service informing them to invalidate cache and pull the new version of 
        snapshot
    - Upgrade service will cache the rollout config with TTL of 1 minute and 
        all the new devices checking for upgrade after this will be denied
    - CDN workers can read config from a CDN KV store and reject downloads or they
        can pull the config from a blob store and cache it with TTL. 

---
Deep dive

abuse/adversarial clients

rate limiting strategy 
- edge
    - ip based rate limit
    - mTLS cryptographic proof of client identity (device_id)
    - verify JWT token against the client identity
    - JWT should be short lived ~1 hour
    - device can get a new token by calling CheckForUpgrade again
    - token is bound to hash(device_id) in audience
- UpgradeService
    - fronted by API Gateway which will do the below
        - ip based rate limit (might run into NAT issues)
        - mTLS cryptographic proof of client identity (device_id)
    - for genuine device 5/hour rate limit. If the device is unable to download
      for some reason, it can retry the next hour.
- telemetry gateway
    - fronted by API Gateway which will do the below
        - ip based rate limit
        - mTLS cryptographic proof of client identity (device_id)
        - enforce rate limit for genuine devices at 1/s. Device can retry if ratelimit
            is exceeded.
    
