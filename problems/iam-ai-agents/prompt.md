# Design an Identity and Access Management system for AI agents
Design an IAM system for a cloud provider that allows customers and enterprises 
to create and manage multiple AI agents with proper access controls and permissions.

# Clarifying questions:
    1. What is the tenancy boundary for the cloud provider ?
        - account ?
            - yes
    2. Are all the resources created under a flat namespace under account or
        is there a sub tenant or project ?
            - yes resources are created under projects
            - an account can contain multiple projects
            - resources are under projects
    3. Can we treat an agent as an resource ?
        - yes
    4. Are the Access controls and permissions defined at project level or 
        can they be defined at the account level ?
        - yes and yes
    5. Do we want to allow grouping permissions into reusable roles ?
        - yes

# Functional Requirements
    - The system should allow creating Agent resources
    - The system should allow configuring roles and role bindings (role + agent(s))
        both at account and project level

# Non Functional Requirements
    - The system should support low latency (<= 10ms) enforcement of access control
    - The system should support low latency propagation (<= 5s ) of role/role binding changes.
        The system is eventually consistent.
    - The system should be highly available (>= 99.99 uptime SLA).
    
# Entities

All entities have
metadata
    - name (unique)
    - project (optional)
    - other fields

permissions
- resource
- allowed_actions: []string (get, create, list, delete)
- additional_actions: []string(set_status, logs etc)


AccountAgent (account scoped)
- metadata

Agent (project scoped)
- metadata


AccountRole (account scoped)
- metadata
- permissions

Role (project scoped)
- metadata
- permissions

AccountRoleBinding (account scoped)
- metadata
- binding
    - subject: -> agent
      role: -> role

RoleBinding
same as AccountRoleBinding but metadata has project

# APIs

CRUD /v1/project/{}/agent
CRUD /v1/project/{}/resource
CRUD /v1/project/{}/role
CRUD /v1/project/{}/rolebinding
CRUD /v1/accountrole
CRUD /v1/accountrolebinding

# High Level Design (problems we need to solve)

## How do we configure the roles and assign them to agents ?
- crud APIs 
## How do we enforce if the agent can access a resource ?
    - assumptions
     - micro services + high rps
    options:
    1. we can consult a central service which will give a go no go decision for
        given request
            - extract request context
            - extract caller context
            - pass to a central service and get the decision back
        - drawbacks
            - extra network round trip for every request, hard to hit our latency
                budget
            - central service is a SPOF, if the service is not available, none
                of the requests can be processed
            
    2. sidecar/embedded library which take the decision within the service
        - easier to hit out latency budget as we avoid n/w round trip
        - load is distributed evenly across the api server fleet, naturally
            load balanced (higher load service, will run more replicas)
        - drawbacks
            - we need to push the roles/rolebindings to the entire API fleet
        - sidecar vs lib
            - sidecar can keep the service agnostic of rbac implementation details
            - library needs, dependency bump + redeploy

    decision
    - side car

## How is the config distributed to side cars ?
    - control plane consumes the roles, account roles, role bindings and account
        role bindings and prepares resource level indexes of 
        project -> agent -> allowed actions + additional actions
        system -> agent -> allowed actions + additional actions
    - this index structure allows almost constant time lookup and enforcement
        given request context and caller context at the enforcement points
    - the per resource index is serialized and stored as immutable snapshots
      in a blob store fronted by CDN (because potentially millions of enforement
      points)
    - the enforcement points listen to events via a pub sub mechanism and are
        notified of updates 
    - the enforcement points pull the snapshot and load it into memory and
        atomically swap pointers
    - If the snapshots are getting big, we can send deltas which can be applied
      live. Snapshot are only used fetch the whole state on start up.
        - this is a little complicated as the deltas are only valid from a known
           reference point. We should use a distributed log like kafka to ensure
           that messages are processed in order and reliably. 
    - The control plane will sign the snapshots so that enforcement points can verify
        the downloaded snapshots. 


## How is the identity of an agent established ?
    - agents get JWT tokens from a central service ( if service is down, existing agents
        will continue to work until they are expired)
    - enforcement side cars pull the JWKs 
        (cached and periodically refreshed, if refresh fails, use stale JWKs) 
        and validate that they are signed by our private keys,
        - if signature cannot be verified, reject
        - if past expiry day, reject
    - extract claims and get subject which should be used for enforcement

    