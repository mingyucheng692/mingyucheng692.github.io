---
title: "Rootless Container Inbound Source IP Preservation: Hybrid Host-Bridge Topology and Tradeoff Analysis"
date: 2026-07-11T20:00:00+08:00
draft: false
tags: ["Podman", "Rootless", "Nginx", "EMQX", "Networking", "NAT", "Source-IP", "Container", "Postmortem"]
categories: ["infra"]
summary: "Architecture analysis and implementation of inbound source IP preservation under Rootless Podman. Uses a Hybrid Host-Bridge topology to retain real client IPs at edge services while preserving network isolation for internal services, with tradeoff analysis, implementation details, and verification methodology."
url: "/en-us/blog/tech/rootless-container-source-ip-preservation/"
---

Inbound source IPs are rewritten to the container bridge gateway IP by the Rootless port forwarder's NAT, breaking audit, rate limiting, and diagnostics. This is a structural constraint of the Rootless user namespace design and cannot be resolved via configuration. The adopted solution is a Hybrid Host-Bridge topology: edge services (Nginx, EMQX) switch to Host Network to retain the real client IP, while internal services (backend, DB, Redis) stay on Bridge Network to preserve isolation, with loopback as the cross-namespace communication channel. This approach holds under the constraints of single-host deployment, a limited number of edge services, and no NetworkPolicy requirement.

> Note: Project names, service names, domains, IPs, accounts, paths, log snippets, and command outputs in this article have been sanitized, abstracted, or rewritten. Only technical information relevant to the architecture decisions is retained and does not correspond to any real production identifier.

## Root Cause: Rootless NAT Mechanism

The system consists of four service categories: a reverse proxy (Nginx), an MQTT broker (EMQX), an application backend (Go), and a data layer (TimescaleDB + Redis). The container runtime is Rootless Podman on a cloud host, with external clients comprising both browsers and IoT devices.

The problem manifests as: application access logs and broker logs uniformly record the client source IP as the container bridge address (e.g., `10.89.0.7`), and source-IP-based rate limiting, when triggered by the single bridge IP, blocks all clients. Audit and diagnostic pipelines lack trustworthy client geographic origin information.

The root cause is that Rootless container network isolation depends on user namespaces, where non-privileged users cannot modify host routing tables or bind to privileged ports (< 1024). To bypass this constraint, Podman uses a user-space port forwarder (`rootlessport` or `slirp4netns`) to map host ports into the container bridge namespace, with the following flow:

1. The port forwarder listens on the target port on the host side
2. When an external connection arrives, the forwarder initiates a new connection within the container bridge namespace
3. The source address of the new connection is rewritten to the bridge gateway IP (e.g., `10.89.0.7`)
4. `$remote_addr` inside the container and the subsequent `X-Forwarded-For` / `X-Real-IP` headers are all based on the rewritten address

```
[External Client] ---> (Host TCP Port 443/1883)
                             |
                   [Rootless Port Forwarder] (NAT Layer)
                             |
                             v  (Source IP rewritten to Gateway IP)
        [Edge Container (Bridge Namespace)]
                             |
                             v  (Propagates rewritten IP via HTTP Headers)
             [Backend Container (Bridge Network)]
```

NAT occurs before the container namespace boundary, so edge containers cannot obtain the original client IP from the connection itself. As long as traffic enters the container bridge through the port forwarder, source IP rewriting is unavoidable — this is the precondition for solution selection.

---

## Solution Selection and Tradeoffs

After confirming the root cause, three candidate solutions were considered. The table below compares them across four dimensions: source IP preservation, network isolation, implementation complexity, and maintenance cost:

| Solution | Source IP Preservation | Network Isolation | Implementation Complexity | Maintenance Cost |
|---|---|---|---|---|
| A. Proxy Protocol | Requires forwarder and upstream support | Preserved | Medium | Strong protocol compatibility dependency |
| B. Full Host Network | Preserved | Lost | Low | Port conflict risk grows with service count |
| C. Hybrid Host-Bridge | Preserved (edge) | Partial (internal) | Medium | Clear boundaries, low coupling |

### Solution A: Proxy Protocol

Introduces Proxy Protocol between the forwarder and upstream to pass the original IP. The prerequisite is that the Rootless port forwarder supports emitting PROXY protocol headers, and all upstreams (Nginx, EMQX) enable the corresponding receive configuration. Currently, `rootlessport` has limited support for PROXY protocol output, and EMQX's MQTT listener and HTTP listener have different Proxy Protocol configuration paths, incurring higher cross-protocol maintenance cost.

### Solution B: Full Host Network

All containers switch to `network_mode: "host"`. The source IP preservation problem is eliminated, but data layer (TimescaleDB, Redis) ports are directly exposed on host interfaces, losing network-level isolation. When extending to multi-instance or introducing network policies, port conflicts and missing isolation increase maintenance cost.

### Solution C: Hybrid Host-Bridge (Adopted)

Only edge services receiving external traffic (Nginx, EMQX) switch to Host Network; internal services (backend, database, cache) remain on Bridge Network. Edge services communicate with the backend via loopback, and the backend retains network isolation.

This solution trades edge-service network isolation for source IP preservation capability, while limiting the internal services' exposure surface via the loopback boundary. Under the current constraints of single-host deployment and a limited number of edge services, this tradeoff is acceptable.

---

## Implementation: Hybrid Host-Bridge Topology

### Network Topology

```
                              HOST NETWORK NAMESPACE
+---------------------------------------------------------------------------------+
|                                                                                 |
|  [External Client] ---> (Port 443) ---> [Nginx (Host Mode)]                     |
|                                                |                                |
|  [IoT Gateway]     ---> (Port 1883) ---> [EMQX (Host Mode)]                     |
|                                                |                                |
|                                                v (Localhost Tunnel)             |
|                                          [127.0.0.1:<backend-port>]             |
|                                                |                                |
+------------------------------------------------|--------------------------------+
                                                 | (Port Publish, loopback only)
                                                 v
                               +----------------------------------+
                               |     CONTAINER BRIDGE NETWORK     |
                               |           (<bridge-net>)         |
                               |                                  |
                               |        [<backend-service>]       |
                               |          /           \           |
                               |         /             \          |
                               |        v               v         |
                               |   [<db-service>] [<redis-service>]|
                               |   (Port 5432)     (Port 6379)    |
                               +----------------------------------+
```

### Key Implementation Points

**1. Host Kernel Port Binding Threshold Check**

Nginx must listen on 80/443 under Host Network, but Rootless users cannot bind ports < 1024 by default. This threshold is a global kernel parameter, not a container-level configuration. If the threshold was already lowered during the HTTPS configuration phase, this step serves only as a confirmation check; if not, evaluate the acceptability of a global lowering under single-host deployment scenarios before executing.

```bash
# Check current threshold (typically already lowered to 80 during HTTPS setup)
sysctl net.ipv4.ip_unprivileged_port_start

# If output is not 80, lower it (Nginx needs 80 for HTTP→HTTPS redirect)
sudo sysctl net.ipv4.ip_unprivileged_port_start=80
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee -a /etc/sysctl.conf
```

For multi-tenant environments, evaluate whether to introduce finer-grained port permission controls (see follow-up evolution).

**2. Compose Configuration Adjustment**

After edge services remove the `ports` and `networks` fields, the container directly shares the host network namespace, bypassing the port forwarder; the backend service publishes only to loopback to avoid exposure on host interfaces after the Host Network switch.

```yaml
services:
  <frontend-service>:
    network_mode: "host"
    # ports and networks fields removed

  <emqx-service>:
    network_mode: "host"
    environment:
      - EMQX_AUTHORIZATION__SOURCES__1__URL=http://127.0.0.1:<backend-port>/api/v1/iot/auth

  <backend-service>:
    ports:
      - "127.0.0.1:<backend-port>:<backend-port>"  # loopback only
    environment:
      - <EMQX_BASE_URL>=http://host.containers.internal:<dashboard-port>
```

`host.containers.internal` is a special DNS name provided by Podman. Containers can use it to access services in the host namespace, replacing the original inter-container direct connection in Bridge Network. This DNS is one-directional (container→host only); the reverse direction (host→container) still requires port publishing.

**3. Reverse Proxy and Broker Configuration Convergence**

Edge service upstream addresses uniformly point to loopback, ensuring cross-namespace communication passes only through the loopback boundary. `$remote_addr` under Host Network mode is the real client IP, requiring no upstream forwarding headers.

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:<backend-port>;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

```hocon
authentication = [
  { backend = "http", url = "http://127.0.0.1:<backend-port>/api/v1/iot/auth" }
]
bridges.webhook.<ingest-bridge> {
  url = "http://127.0.0.1:<backend-port>/api/v1/iot/ingest"
}
```

**4. Redeployment and Supervisor State Synchronization**

After configuration changes, containers cannot be brought up directly via `podman-compose up`. The original deployment manages container lifecycles via systemd units, and the unit files were generated based on the original Bridge Network mode. After switching the network mode from Bridge to Host, the systemd units must be regenerated by the deploy script to capture the new network properties, and control is then handed back to systemd. Skipping this step causes the old units to bring up containers in the original Bridge mode, resulting in configuration/runtime inconsistency and potential failure of auto-start on boot.

```bash
# Execute via the original deploy script, which regenerates systemd units and hands over management
./<deploy-script>  # CI: pipeline should verify old units are stopped before generating new ones, to avoid coexistence of old/new units causing duplicate instances
```

---

## Verification Strategy

Verification must cover the full link, done in three layers. The logical relationship: Nginx logs confirm the edge layer has obtained the real IP; backend logs confirm the IP is correctly propagated via HTTP headers; EMQX logs confirm the Broker-layer AuthN/Webhook link has converged to loopback. All three are indispensable — verifying only Nginx cannot confirm whether the backend correctly parses headers; verifying only the backend cannot confirm whether the Broker link has been adjusted.

### 1. Nginx Log Verification

```bash
podman logs <frontend-service>
```

Expected: the log entry's leading IP is the client public IP, not the bridge gateway address.

```
<client-public-ip> - - [<timestamp>] "GET /api/v1/projects HTTP/1.1" 200 ...
```

### 2. Backend Structured Log Verification

```bash
podman logs <backend-service>
```

Expected: the `"ip"` field in the JSON log matches the client IP in the Nginx log.

```json
{"level":"INFO","ts":"<timestamp>","caller":"middleware/logger.go:<line>","msg":"Inbound Request","ip":"<client-public-ip>","status":200,"method":"GET","path":"/api/v1/<resource>"}
```

### 3. EMQX Integration Verification

```bash
podman logs <emqx-service>
```

Expected: listeners bind to `0.0.0.0`, and AuthN and Webhook requests route to `127.0.0.1`.

```
EMQX_AUTHORIZATION__SOURCES__1__URL [authorization.sources.1.url]: http://127.0.0.1:<backend-port>/api/v1/iot/auth
Listener tcp:default on 0.0.0.0:1883 started.
Listener http:dashboard on :<dashboard-port> started.
```

---

## Engineering Constraints and Follow-up Evolution

### Validity Boundaries of the Current Solution

The Hybrid Host-Bridge topology holds under the following constraints:

- **Single-host deployment**: Host Network port conflict risk is amplified in multi-host/cluster scenarios
- **Limited edge service count**: Each additional Host Network service linearly increases host port management complexity
- **No network policy requirement**: Rootless Podman's Host Network does not support NetworkPolicy; isolation depends on the loopback boundary

### Identified Follow-up Evolution Directions

**1. Migrate from Host Network to Pasta**

The pasta mode was introduced in Podman 4.x as a replacement for slirp4netns, and became the default rootless network stack in Podman 5.0 (alongside netavark + nftables). The key improvement over slirp4netns is that it no longer performs SNAT on inbound connections, making source IP preservation feasible in Bridge mode as well. When the runtime version is upgraded, evaluate whether to revert edge services to Bridge Network and restore network isolation.

**2. Fine-grained Port Binding Permission Control**

Currently, the port threshold is lowered via global `sysctl`. As more services are introduced, evaluate systemd's `AmbientCapabilities=CAP_NET_BIND_SERVICE` or RootlessKit's port whitelist mechanism to avoid global threshold lowering.

**3. Monitoring and Alerting Coverage**

Current verification relies on manual log inspection. Source IP preservation failure does not produce obvious errors at runtime; monitoring items (such as bridge IP frequency in logs, rate limit hit anomalies) must be established for passive detection capability.

**4. Multi-instance and Horizontal Scaling**

Under the current single-host topology, Host Network port binding is exclusive. If edge services later require multi-instance or canary release, an upstream load balancer layer must be introduced, at which point the source IP preservation issue shifts to the LB layer and requires reevaluating Proxy Protocol or client IP preservation solutions at the LB layer.

---

## Engineering Constraints Summary

- **Inherent tension between network isolation and source IP preservation**: Host Network preserves IP but loses isolation; Bridge Network preserves isolation but rewrites IP. The core work of solution selection is drawing the boundary between the two.
- **Hybrid topology boundaries should be drawn along the data flow direction**: Edge services receiving external traffic use Host Network; internal services for inter-service communication use Bridge Network; loopback serves as the cross-namespace communication channel.
- **Verification must cover the full link**: Single-point logs are insufficient to confirm end-to-end IP propagation correctness; layer-by-layer verification from edge to backend is required.
- **Network mode changes require supervisor state synchronization**: After switching the container network mode, the supervisor's (e.g., systemd) unit files must be regenerated to capture the new properties; otherwise, configuration/runtime inconsistency and auto-start failure may occur.

The constraints distilled here can serve as a reference baseline for future runtime upgrades or topology changes.
