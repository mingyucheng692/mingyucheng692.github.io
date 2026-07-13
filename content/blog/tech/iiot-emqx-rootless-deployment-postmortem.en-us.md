---
title: "IIoT Ingress Postmortem: Troubleshooting EMQX 5.8 Under Podman Rootless"
date: 2026-05-28T18:59:25+08:00
draft: false
tags: ["IIoT", "MQTT", "EMQX", "Podman", "Rootless", "Erlang", "HOCON", "Webhook", "CSRF", "Postmortem"]
categories: ["backend-infra"]
summary: "A postmortem of an IIoT ingress deployment failure: under Podman Rootless, EMQX 5.8 exposed unstable Erlang IPC, HOCON schema validation failures, a blocked security-group port, and M2M requests rejected by CSRF middleware. This post covers the confirmed causes, fixes, and validation path."
url: "/en-us/blog/tech/iiot-emqx-rootless-deployment-postmortem/"
---

This post documents the troubleshooting and repair of an IIoT ingress deployment. The target was straightforward: run `EMQX 5.8.9` in Podman Rootless on a cloud host, then complete MQTT device access, HTTP-based dynamic authentication, webhook forwarding, and time-series persistence.

In practice, the path exposed four separate problems: `emqx ctl` could not reliably reach the running node inside the container, EMQX crashed at startup after switching to a prewritten `cluster.hocon`, external MQTT clients kept timing out, and the backend returned `403 Forbidden` for M2M HTTP requests sent by EMQX. These symptoms belonged to four different layers: runtime control, configuration schema, network policy, and security middleware.

> Note: project names, service names, domains, IPs, paths, accounts, table names, request headers, log snippets, and command outputs in this article are all desensitized, abstracted, or rewritten placeholders. They preserve only the technical details relevant to the troubleshooting path.

## Background And Goal

- Container runtime: Podman Rootless
- Broker: `EMQX 5.8.9`
- Authentication mode: HTTP AuthN
- Data path: MQTT -> Rule Engine -> Webhook -> TimescaleDB
- Backend protection: browser-side CSRF defense + M2M token validation

The intended workflow was simple: devices connect over MQTT, EMQX calls the backend for dynamic authentication, then telemetry is forwarded through a webhook and persisted in the time-series database.

## Incident Summary

This was not a single-point failure. Four fault chains overlapped:

1. Podman Rootless isolation made Erlang EPMD IPC unreliable, so `emqx ctl` could not serve as a reliable runtime configuration path
2. EMQX 5.8.9 enforced stricter HOCON schema validation on bridge configuration, and legacy fields triggered `unknown_fields`
3. The cloud security group did not allow inbound traffic on `1883`, so external MQTT access was blocked
4. The backend CSRF middleware protected all POST requests by default and accidentally blocked AuthN and webhook calls from EMQX

The final remediation matched those four chains:

- stop relying on `emqx ctl` for dynamic loading and render a complete `cluster.hocon` before EMQX starts
- remove obsolete `resource_opts` fields
- open inbound `1883` in the security group
- bypass CSRF for `<iot-auth-prefix>` and add constant-time `X-M2M-Token` validation on `<iot-ingest-endpoint>`

---

## Troubleshooting Timeline

### Phase 1: Why Dynamic Configuration Was Unreliable

The initial deployment plan started EMQX first and then injected HTTP AuthN and webhook configuration with:

`emqx ctl conf load --merge`

What the container repeatedly returned was:

```text
Node emqx@<service>-emqx not responding to pings
```

The first useful conclusion was that the problem was not yet in business logic or payload format. It was in the runtime control plane. If `emqx ctl` could not reliably reconnect to the running Erlang node, the entire hot-load approach had no stable foundation.

#### Hypothesis A: The node hostname was wrong

The first guess was a node-name resolution issue, so the Compose file was updated with an explicit `hostname`. That made the node naming more consistent, but `emqx ctl` was still unreliable. Hostname alignment helped remove noise, but it was not the root cause.

#### Hypothesis B: Compose expanded the `sed` placeholder too early

Another confirmed issue was that `${VAR}` in the command string was expanded too early by `podman-compose`, which broke the `sed` replacement target and caused token injection to fail. Replacing that pattern with a static placeholder fixed token rendering, but `emqx ctl` was still unstable. This showed that template replacement had a defect, but it still was not the main reason the deployment path failed.

### Phase 2: Bypass `emqx ctl` and Write Config Before Startup

Once the problem was narrowed to runtime IPC, the next step was to avoid that control path entirely. The rendered configuration was written directly to:

`/opt/emqx/data/configs/cluster.hocon`

and EMQX was started only after that file was in place. This bypassed `emqx ctl`, but immediately exposed a second blocker: EMQX crashed during startup.

The decisive log lines were:

```log
[error] failed_to_check_schema: emqx_conf_schema
[error] #{reason => unknown_fields,
           path => "bridges.webhook.<service>_backend_ingest_bridge.resource_opts",
           unknown => "pool_size,buffer_type,max_retries,..."}
```

This made the next conclusion clear. Pre-rendering the configuration was the right direction, but the template still contained legacy fields that were no longer accepted by the current EMQX version.

#### Hypothesis C: Removing only part of `resource_opts` would be enough

The first attempt removed fields such as `buffer_type` and `max_retries` while keeping `pool_size`. EMQX still failed schema validation. That showed the issue was not a bad value in one field; the `resource_opts` subtree itself no longer matched the current version. EMQX started normally only after the whole block was removed.

#### A second issue discovered at the same stage

This phase also exposed a configuration-overwrite risk. Once `cluster.hocon` was written as a full replacement, a template that only contained webhook rules could silently wipe previously configured HTTP AuthN settings. That would let the broker start but break device authentication.

The final template therefore declared three parts together:

- HTTP AuthN
- rule engine rules
- webhook bridge

That change was not cosmetic. It ensured startup configuration remained atomic instead of being split across multiple mutation paths.

### Phase 3: The Broker Was Up, but External Devices Still Timed Out

Once EMQX started normally and container logs showed:

```text
Listener tcp:default on 0.0.0.0:1883 started
```

external clients were still timing out. At that point, the problem was no longer inside the broker. The investigation moved outward to the network boundary.

`mosquitto_pub` from outside the host still timed out against `<IP>:1883`, which narrowed the problem down to the IaaS layer. The final cause was a missing inbound `1883` rule in the cloud security group. Once that rule was added, MQTT connectivity from devices recovered immediately.

This is a common failure pattern: a container can bind the port correctly while the real access path is still blocked farther out.

### Phase 4: M2M HTTP Requests Were Blocked by Backend Security Middleware

After MQTT connectivity was restored, EMQX still received `403 Forbidden` on HTTP AuthN and webhook requests. That led the investigation into the backend security layer, where the root cause turned out to be the global CSRF middleware.

The backend originally enforced strict `Origin` and `Referer` checks for browser traffic. That is a reasonable default for web-facing endpoints, but it does not fit machine-to-machine requests from EMQX. The issue is not that such requests can never carry those headers. It is that they do not operate within the browser trust model and therefore cannot reliably satisfy CSRF checks built around `Origin` and `Referer`.

The resolution was not to turn CSRF off globally, but to split the security model:

- bypass CSRF for the `<iot-auth-prefix>` path prefix
- validate `X-M2M-Token` on `<iot-ingest-endpoint>`
- compare the token with `crypto/hmac.Equal` to avoid timing side channels

This kept browser-facing protection in place while switching the IoT ingress path to a mechanism that actually matches M2M traffic.

---

## Root Cause Analysis

### 1. Runtime Friction Between Podman Rootless and Erlang EPMD

`emqx ctl` depends on Erlang distribution and EPMD node discovery. In a Rootless container environment, user namespaces and the virtual network stack alter loopback and local IPC behavior enough that this control path can become unreliable. The problem was not that the command syntax was wrong. The problem was that the chosen runtime control mechanism was not a stable fit for this environment.

### 2. EMQX 5.8.x Tightened Acceptance of Legacy HOCON Fields

Older webhook bridge configurations often used fields such as `resource_opts.pool_size`, `buffer_type`, and `max_retries`. In `5.8.x`, those paths are no longer accepted in the same way. The broker now performs strict startup validation and rejects unknown fields instead of ignoring them, so legacy configuration can block boot completely.

### 3. Network Policy and Application Logs Can Drift Apart

The fact that EMQX listened successfully on `1883` only proved that the broker had bound the local socket. It did not prove that the end-to-end ingress path from devices was open. When clients time out and the application stays quiet, the next checks need to move outward to security groups, firewall rules, and external routing.

### 4. Browser Security Logic Should Not Be Applied Directly to M2M Traffic

CSRF is designed for browser-originated traffic. AuthN and webhook calls from EMQX are M2M requests. If the same browser-oriented checks are applied to both classes of traffic, legitimate broker calls will be treated as suspicious simply because they do not carry browser semantics. The correct fix is not "less security," but a different security model for a different caller class.

---

## Resolution

The final solution had four parts.

### 1. Render A Complete `cluster.hocon` Before Startup

The hot-load approach was abandoned. Instead, the container entrypoint renders the final configuration and writes it before EMQX starts. This removes the dependency on unstable runtime IPC.

### 2. Merge AuthN, Rule Engine, and Webhook Into One Template

The final template contains:

- HTTP AuthN
- telemetry forwarding rules
- webhook bridge

and removes the incompatible `resource_opts` block entirely.

### 3. Open the External Network Path

After verifying EMQX was listening on `0.0.0.0:1883`, the cloud security-group rule was added so devices could actually reach the broker from outside.

### 4. Split the M2M Security Path from Browser Security

The backend now bypasses CSRF on `<iot-auth-prefix>` and enforces a shared-secret check on `<iot-ingest-endpoint>`. The two request classes are now protected differently:

- browser endpoints: CSRF remains enabled
- IoT M2M endpoints: `X-M2M-Token` is required

That makes the backend security boundary more explicit and avoids mixing browser and device traffic under one validation model.

---

## Validation

Validation was completed in three layers.

### 1. Backend Middleware Validation

Tests for CSRF and token middleware confirmed:

- ordinary web/API requests still return `403` when `Origin` or `Referer` is missing or incorrect
- the `<iot-auth-prefix>` prefix bypasses CSRF
- `<iot-ingest-endpoint>` rejects missing or invalid tokens

### 2. End-to-End MQTT Validation

An external publish test was executed with:

```bash
mosquitto_pub -d -h <Domain> -p 1883 \
  -i "<Device_SN>" -u "<Device_SN>" -P "<Token>" \
  -t "telemetry/device/<device_id>" \
  -m '{"fcs_speed_rpm": 7766, "fcs_soc": 88.9, "fms_fault_code": 0}'
```

The observed result confirmed that:

- device connection succeeded
- HTTP AuthN passed
- the message entered the broker and was processed by the rule engine

### 3. Persistence Validation

A database query confirmed successful writes into the time-series table, while the `_iot` data-plane role remained isolated from control-plane permissions.

---

## Takeaways And Next Steps

This troubleshooting cycle leaves a few clear engineering lessons:

- under Rootless Podman, prefer declarative configuration over runtime Erlang IPC whenever both are possible
- when middleware versions change, advanced tuning fields are more fragile than baseline functional settings, so validation should start from a minimal config
- "the service is listening" does not mean "the service is reachable"; ingress must be verified layer by layer out to the security group or firewall
- browser security and M2M security should be split explicitly instead of forced into one middleware model

At the current stage, the single-node template-rendering solution is sufficient for MVP verification. If deployment requirements expand beyond that, the next concerns are already clear:

- broker clustering and centralized configuration management
- asynchronous buffering between webhook and backend
- migration from shared-secret authentication toward mTLS or a stronger device identity model

The value of this repair is not that one startup error disappeared. The real value is that a failure chain spread across runtime, configuration, network, and security was turned into an ingress path that is understandable, testable, and repeatable.
