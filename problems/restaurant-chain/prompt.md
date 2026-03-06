# Problem: Design a Global Menu Update and Distribution System for a Restaurant Chain 

## Background & Requirements
The chain operates stores across many countries/regions.
Each store may have multiple types of devices that display or consume menu data, e.g.:
- digital menu boards
- POS/cash registers
- self-service kiosks
- staff/manager mobile devices
- other endpoints relying on menu data

Menus are published and controlled by headquarters.
A menu includes:
- global shared parts (global items)
- localized parts (country/region/store-specific items)

Updates are frequent:
- can change daily
- can differ by breakfast/lunch/dinner (and differ from the previous day)

## Goal

Design a system that reliably and efficiently distributes HQ menu updates to all stores and endpoints worldwide.

## Key Areas to Cover
- Scalability / Global scale
    - support many stores, multiple countries/regions, and multiple devices per store
    - strategy for mass updates
- Consistency & correctness
    - ensure devices render the HQ-published menu version (at least eventual consistency)
    - versioning, rollback, and optionally staged rollout
- Time windows & effective rules
    - menus effective by meal period (breakfast/lunch/dinner) across time zones
    - override/precedence rules for global vs localized content
- Offline & failure corner cases
    - what if a store is offline?
    - flaky networks, duplicate deliveries, partial device failures, stale versions
    - catch-up after long offline periods
- Data model & APIs (high-level)
    - how to model menu, items, prices, languages/currencies, effective time ranges
    - inheritance/overrides by country/store
    - device fetch APIs (push vs pull, incremental vs full)
- Observability & operations
    - monitoring update propagation to each store/device
    - audit logs (who published what and when)

---

# Clarifying Questions:
1. What is the scale of the system ? How many devices/stores will the system be serving ?
    - 1 Million 
2. Does the menu have only have text or does it have media ? What is the maximum size of the menu ?
    - Yes, contains media files. Can be several GB large.
3. Are there device specific variations of menu ?
    - Yes 

# Functional Requirements
1. The system should serve menu to end devices.
2. The system should support customizing the menu by region/store/device.

# Non Functional Requirements
1. Updates to menu should be visible to clients within 10s. The system is eventually
    consistent with <= 10s SLA. 
2. The devices should statically stable, i.e. if the menu service is unavailable, the devices
    should continue to display last known menu.

# Entities
Menu
- Metadata
- version

Menu will refer to assets (media)

Override
- overrides
- labels: map[string]string
- weight: uint

Device
- labels: map[string]string
    - region: <>
    - type: <>
    - store: <>
- current_menu_version: int

# APIs
-- ADMIN APIs -- 
CRUD APIs to create menu

-- Device APIs --
CheckForMenuRequest{
    current_version
    labels
}

CheckForMenuResponse {
    update_available: bool
    token: <>
    metadata_url: <>
    signature: <>
    retry_after: <>
}

CheckMenuVersion(CheckForMenuRequest) -> (CheckForMenuResponse)

MenuMetadata {
    page: []page
    assets: []asset{
        url
        signature
    }
}

# BOE

1 M devices * 5 GB -> 5000 TB of data transfer

intuition is too much data to transfer from the API servers, we should use CDN

# Architecture Sketch

## How do we create/update menu
### How should we structure / store the menu
- menu -> metadata (~KB) + assets (pictures, videos etc) (~GB)
- it does not make sense to store metadata and assets in the same store
- we should store the menu metadata in a strongly consistent data store like 
    Postgres, Spanner, DynamoDB etc
- we should store the assets in a blob/object store like S3
- menu changes should be versioned and we should create immutable snapshots.
- We should post process the menu changes and create readily consumable snapshots
    for the end user devices
- Overrides should be applied on top of the base menu.
- We will have as many unique snapshots for every version of menu as the number
    of unique label combinations.
    - TODO: risk is cardinality explosion, we need to keep the number of unique
        combinations in control
- After the post process, we will have store the final snapshots in a blob store.
- The access to the snapshots will be through CDN.
- TODO: deterministic mapping of labels -> snapshot

## How do devices get the menu
1. Devices periodically call the CheckMenuVersion API and present their
    current version and labels assigned to the device.
2. This API would be served by a stateless menu snapshot service, which will 
    determine if a new snapshot is available for a given version and labels.
3. If the new snapshot is available, the menu service will respond with a JWT
    token and metadata_url pointing to the CDN.
4. JWT token will have a short validity (~30 minutes), the CDN edge workers
    will validate the JWT token using pre published JWKs. 
5. After the token is validated, they allow the download. The metadata file
    is downloaded and the signature is verified. 
6. The metadata file contains references to the asset urls and signatures.
7. The device will download all the assets and verifies the signature. 
8. After successful download the menu pointer is updated from current to next.

## How do we post process the snapshots
1. Our goal is to maintain a postprocessed snapshot per unique label combination
2. A unique label combination is hash(sorted([]key=value)) we call this a 
    combination_id.
3. We pool all the labels from overrides and enumerate all the label combinations
    and created combination_ids for them.
4. To figure out which combination_ids are affected when an override is updated
    we need to maintain a index which has label -> []combination_id. 
5. When a label (key=value) changes, we will gather all the combinations that will
    be affected by it and build the snapshot.
6. Each combination has a monotonically increasing snapshot_id which is derived 
    from a menu_version and sorted(override_versions). 
7. When a new menu version or override version is created, we look up all the 
    combinations that are going to be affected by this and asynchronously trigger
    jobs to build immutable snapshots.
8. After the snapshot is build, it is stored in an blob store and an snapshot 
    available event is raised for a combination. This will be consumed by
    the menu service workers which update the latest snapshot in redis/memory
    for a given combination.
9. When devices make CheckMenuVersion request, the combination_id is derived
    from hash(sorted(labels)) and looked up in the cache to serve the response.
10. This ensures that the menu service is state less and can be scaled
    horizontally. 
11. We can also push notifications to the end devices letting them know of 
    of new versions if needed for faster menu updates.

## How do we rollback/invalidate menu versions 
1. We need to stop sending this version of menu to devices in CheckMenuVersion
    API.
    - add revoke API which will revoke this version and make a new version
        which is same as the last unrevoked version.
    - add revoked version to revoked_versions key in the cache. This will
        prevent the server from responding to devices asking for updates which
        instead will respond with check_later as the current_version is in the
        revoked set.
    - trigger jobs to update the current version for all combination_ids
    - JWT will have claims which contain the menu_version. 
2. We need to prevent client which are downloading this version from CDN.
    - step 1 will push config to CDN kv store and add the revoked version.
    - edge workers will check the menu_version claim against the config
        and reject the request.
    - as the request is rejected, the menu download won't be complete and the
        devices will continue using the current version.
3. Rollback devices which have already upgrade
    - Push notification to all the devices which are using the revoked version
        or respond in CheckMenuVersion api with the new version and they 
        will upgrade.

## Metrics
1. we need to track version for a device
    - expected_version for the device
    - current_version running in the device
    - we can collect both these metrics from the CheckMenuVersion API server
    - we can also track, which device we have offered a new version of the menu
    - we can track from CDN which device is downloading a version of menu
    - using a combination of the above metrics we can track, when a new version
        is offered, when did the device start downloading it, when did it finish
        and when did finish switching to the new version.
    - If we need more accurate metrics we can emit from the device instead of 
        deriving from other sources.

## Can we optimize the download
1. Instead of full download of snapshot every time, we can do delta downloads.
2. Most of the assets might not change between versions. We can exploit this
    fact and make the download less intensive. 
3. Device can only request assets which are not available locally on the device
    or whose signature does not match with local. 

---
