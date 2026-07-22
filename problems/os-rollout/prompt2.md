# Design a system to rollout new versions of a OS to VM/Nodes

# Clarifying Questions
- Is the rollout staged ?
    - yes
- Do the node have n/w access ? Can they reach internet ?
    - yes, yes
- Can nodes opt out of OS upgrades ?
    - yes
- Do we need to support pausing/cancel the rollout ?
    - yes
- Scale ?
    - 10M nodes
- Can nodes opt out of upgrades ?
    - yes

# Functional Requirements
- the system should support rolling out new versions of OS to nodes
- the rollout should be staged 
- rollout can be paused or cancelled
- nodes can opt out of rollout
- we should have visibility into progress of the rollout

# Non Functional Requirements
- The system should be highly available (99.99 SLA). availability >> consistency 
    from CAP POV. 
- The system should be fault tolerant and scalable

# Entities / APIs
Release
- id
- version_metadata
- artifacts
    - urls

RolloutPlan
- id
- release_id
- []plan_stage {
    - id
    - percentage
    - constraints
}

Node
- id
- info
    - os_version
    - hardware_info



rpc CheckForUpgrade(CheckForUpgradeRequest) returns (CheckForUpgradeResponse)



# BOE

Data Tx
Avg size of OS upgrade ~1 GB
Number of nodes = 10 M
Total data transferred = 10 PB ( should use CDN for distribution)

Push vs Pull
Push
- stateful orchestrator has to coordinate upgrades on 10 M hosts
- complex and too many moving parts at 10 M scale
    - have to maintain 10M active stateful sessions
    - we can stagger them and do it in batches

Pull
- let agent in the host, check if the host is eligible for upgrade and 
    initiate and manage the upgrade
- 10 M x poll every 30s = ~ 300k rps (non trivial, but manageable), drop it to 300s + jitter
    30k rps (trivial)
    

Decision: Pull

# High Level Design

# Low Level Design
