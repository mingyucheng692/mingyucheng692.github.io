---
title: "Windows Docker Ghost Port Mapping: EMQX 1883 Silently Reserved by WinNAT Postmortem"
date: 2026-08-25T12:00:00+08:00
draft: false
tags: ["Docker", "Windows", "WinNAT", "Hyper-V", "EMQX", "MQTT", "Postmortem"]
categories: ["backend-infra"]
summary: "Docker Desktop on Windows with a full stack (backend + EMQX 5.8.9 + TimescaleDB + Valkey). Backend ping/pong works, but mosquitto_pub keeps returning Connection refused. Other ports in docker ps map fine; only 1883 lacks the 0.0.0.0:1883-> prefix and Docker reports no error. Root cause: Hyper-V/WinNAT dynamic port reservation holds 1883 (in the 1802–1901 range). Four remediation options: swap port, temporary reclaim, admin allowlist, fix the dynamic port range."
url: "/en-us/blog/tech/docker-windows-port-ghost-mapping-postmortem/"
---

> Environment and commands are desensitized: container names, accounts, passwords, tokens, topics, and payloads are abstracted to placeholders.

Env: Docker Desktop (Windows) with a full stack (backend + EMQX 5.8.9 + TimescaleDB + Valkey); path is WSL2 `mosquitto_pub` → host `127.0.0.1:1883` → EMQX container.

## Symptom

```bash
mosquitto_pub -d -h 127.0.0.1 -p 1883 -i "<Client_ID>" -u "<User>" -P "<Password>" -t "<Topic>" -m '<Test_Payload_JSON>'
# Error: Connection refused
```

1883 listens inside the container; the host mapping is missing; Docker reports no error.

## Key Evidence

The whole stack came up from a single `docker compose up`; the backend app layer responds normally:

```bash
curl 127.0.0.1:8080/ping
# {"msg":"pong"}
```

But only EMQX's 1883 lacks the prefix in `docker ps`:

```text
NAMES            PORTS
<svc>-backend    0.0.0.0:8080->8080/tcp
<svc>-emqx       0.0.0.0:8083->8083/tcp, 1883/tcp, 0.0.0.0:18083->18083/tcp
<svc>-db         0.0.0.0:5432->5432/tcp
<svc>-valkey      0.0.0.0:6379->6379/tcp
```

Same machine, same compose, same `up`: backend (8080), db (5432), valkey (6379), and emqx's 8083/18083 all map normally — only 1883 lacks the `0.0.0.0:1883->` prefix (container-internal only). The problem is pinned to port 1883 itself, not Docker or compose.

## Hypotheses & Verification

| # | Hypothesis | Verification command | Result | Conclusion |
|---|---|---|---|---|
| A | Config not reloaded | `docker compose up -d --force-recreate <svc>-emqx` | 1883 still lacks prefix | Ruled out |
| B | Hidden YAML chars break parsing | `docker compose config` | Output contains `published: "1883"` | Ruled out |
| C | Host port held by a process | `netstat -ano \| findstr :1883` | No output | Ruled out |
| D | Windows dynamic port reservation | `netsh int ipv4 show excludedportrange protocol=tcp` | `1802  1901` contains 1883 | **Root cause confirmed** |

After A–C are ruled out, the problem narrows to "Docker read the config but the host bind failed silently". D hits.

## Root Cause

Windows Hyper-V / WinNAT pushes the dynamic port start down to ~1024 and randomly reserves large port ranges for system use. 1883 falls inside the reserved range `1802–1901`; when Docker tries to bind, Windows intercepts it silently — no `bind: address already in use` is thrown, the port mapping is dropped, producing the ghost state: listening inside the container, no host mapping, no error.

Key output snippet:

```text
Start Port    End Port
----------    --------
      1802        1901      ← 1883 falls in this range
```

## Remediation

Option 1 is the recommended root fix; Options 2–4 are fallbacks when Option 1 isn't viable, ordered by decreasing temporariness.

### Option 1: Fix dynamic port range (root fix · recommended, admin PowerShell)

```bash
netsh int ipv4 set dynamicport tcp start=49152 num=16384
netsh int ipv6 set dynamicport tcp start=49152 num=16384
```

Reboot to take effect. Reserved ranges now stay at 49152+, no longer interfering with 1883/3306/6379/8080. Dev machines use this as the baseline.

### Option 2: Swap port (stopgap)

```yaml
ports:
  - "11883:1883"
```

Use `-p 11883` with `mosquitto_pub`. Suitable for short-term debugging, devices not hard-coded to 1883.

### Option 3: Temporarily reclaim 1883 (admin PowerShell)

```bash
net stop winnat
docker compose up -d --force-recreate <svc>-emqx
net start winnat
```

Releases the reserved range while winnat is stopped so Docker can grab 1883. May recur after reboot, suited for temporary emergencies.

### Option 4: Admin allowlist for 1883 (admin PowerShell)

```bash
net stop winnat
netsh int ipv4 add excludedportrange protocol=tcp startport=1883 numberofports=1
net start winnat
```

Verify: `netsh int ipv4 show excludedportrange protocol=tcp` shows `1883  1883  *` (`*` = admin-reserved). Suited for cases that must use 1883 long-term.

## Verification

```text
# docker ps
0.0.0.0:1883->1883/tcp, ...      ← prefix present, mapping live

# mosquitto_pub
(no Connection refused; message reaches EMQX Dashboard)
```

## Takeaways

- `docker ps` `PORTS` column missing the `0.0.0.0:port->` prefix = mapping not live; when there's no error, suspect the host network layer before the compose file
- In-container `0.0.0.0:1883 started` ≠ host-reachable; verify layer by layer up to the host mapping
- Windows reserved ports fail silently; always run `netsh int ipv4 show excludedportrange protocol=tcp`
- Temporary reclaim / port swap are stopgaps; `set dynamicport tcp start=49152` is the root fix
- WSL2 → Windows Docker triage order: `docker ps` → `netstat` → `netsh`; consider WSL forwarding (`host.docker.internal`) only last
- Standardize dev machines on Option 1 and fold into onboarding SOP; for embedded devices hard-coded to 1883, stack Option 4 on top
