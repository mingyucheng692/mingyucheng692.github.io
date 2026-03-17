---
title: "HTTPS Upgrade Triggered 403: A Deep Postmortem from Security Middleware to Container Isolation"
date: 2026-03-17T12:00:00+08:00
draft: false
tags: ["HTTPS", "Nginx", "Go", "Gin", "CORS", "CSRF", "Podman", "Rootless", "Postmortem"]
categories: ["Tech", "Engineering", "Security"]
summary: "A postmortem on a persistent 403 after HTTPS migration, traced to both missing CSRF allowlist updates and Podman Rootless image namespace isolation."
url: "/en-us/blog/tech/https-upgrade-403-postmortem/"
---

While hardening infrastructure for a core business line, we migrated the global protocol from HTTP to HTTPS. SSL was configured on Nginx and all traffic was redirected to 443, but regression tests consistently hit `403 Forbidden` on the login API.

Response payload:

```json
{"code":403,"msg":"forbidden: invalid origin"}
```

System architecture:

- Frontend: Vue
- Backend: Go + Gin
- Gateway: Nginx reverse proxy
- Runtime: Podman (Rootless mode)

---

## Troubleshooting Strategy: Top-Down Layer Isolation

We followed a strict top-down path with clear boundaries: gateway layer, application layer, and runtime layer.

## Layer 1: Gateway (Nginx)

The first suspicion was header forwarding. Configuration review confirmed `Origin` and related headers were already forwarded correctly.

During debugging, we found this workaround could make requests pass:

```nginx
proxy_set_header Origin "http://business-domain";
```

But this approach forges the request origin and effectively bypasses backend origin verification, weakening CSRF protection. We rejected it and continued to root-cause analysis in the application layer.

## Layer 2: Application (CORS vs CSRF Misalignment)

Packet inspection showed this response header was correct:

```http
Access-Control-Allow-Origin: https://business-domain
```

That proved CORS already allowed HTTPS. The key was that:

- CORS controls whether browsers can read cross-origin responses
- CSRF validates whether backend should trust the request origin

They are parallel but independent defenses.

Source review exposed the real gap: during protocol migration, only the CORS allowlist was updated. The CSRF middleware still trusted the old `http://` origin list. Requests passed CORS but were blocked by CSRF with `invalid origin`.

Fix: add `https://` domains to CSRF trusted origin allowlist in sync with CORS.

## Layer 3: Runtime (Podman Rootless Isolation)

After code fix and build, production still failed. We compared image hashes via `podman inspect` and found a hidden operational trap:

- Production services run under `deploy` in a Rootless namespace
- Some images were built in the Root namespace
- Under Rootless isolation, `deploy` cannot see newly built images in Root namespace

Deployment scripts kept restarting containers from stale images, creating a false state where code was fixed but runtime never updated.

---

## Root Cause Summary

This incident was not a single bug, but two systemic gaps stacked together:

1. Architecture governance gap: CORS and CSRF allowlists were configured separately without centralized security config ownership.
2. Engineering loop gap: image build authority was not fully constrained to standardized CI/CD, allowing cross-namespace dirty state.

---

## Action Items and Process Evolution

### 1) Strengthen Security Release SOP and Checklist

Add mandatory “CORS-CSRF policy alignment” checks for new services and protocol upgrades.

### 2) Converge Production Operations Baseline

Enforce that application image build and lifecycle are handled only by CI/CD pipelines. Manual cross-privilege namespace intervention is prohibited.

### 3) Standardize Observability for Microservices

Make middleware order an architecture rule: logging must be loaded before security interception, ensuring blocked requests are always auditable.

---

## Closing

The value of this 403 investigation was not just restoring one API. It revealed weak points in security governance and delivery reliability. Code fixes solve the current outage; process constraints prevent the next one.
