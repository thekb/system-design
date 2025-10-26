# Distributed IP Blocking Service

Goal: Detect and block abusive IPs (or CIDRs/ASNs) across many edges (proxies, API gateways, app servers, clusters, CDNs) with <5s propagation, millisecond request path cost, low false positives, and auditable controls.

Scope: Both proactive (threat feeds, reputation) and reactive (rate/behavior signals) blocking. Support IPv4/IPv6, CIDR, ASN, geo, and time-bounded blocks with exceptions (allowlists).

Constraints: Multi-region, multi-cloud, multi-tenant isolation, thousands of edges, billions of daily requests. Prefer zero trust, signed policy, and HA.