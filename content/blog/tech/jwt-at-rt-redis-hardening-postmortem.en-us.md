---
title: "JWT Dual-Token Hardening Postmortem: From Stateless Refresh to Revocable Redis Sessions"
date: 2026-03-23T10:00:00+08:00
draft: false
tags: ["JWT", "Access Token", "Refresh Token", "Redis", "Go", "Gin", "Security", "Zero Trust", "Postmortem"]
categories: ["Tech", "Engineering", "Security"]
summary: "A security hardening postmortem for JWT AT/RT architecture: treating Redis reservation as completed and implementing RT rotation, replay detection, and revocable sessions."
url: "/en-us/blog/tech/jwt-at-rt-redis-hardening-postmortem/"
---

This post documents a practical JWT dual-token (AT/RT) hardening effort.  
The real question was simple: if a refresh token is replayed in an abnormal scenario, can the system detect it quickly and contain impact.

> Note: all domains, IPs, account identifiers, session IDs, and log samples are desensitized placeholders.

System boundary:

- Frontend: Web SPA (AT stays in memory)
- Backend: Go + Gin (auth and refresh)
- Session layer: Redis (token state index)
- Gateway: HTTPS reverse proxy

---

## Symptom and Trigger

In routine security exercises for a utility-scale energy cloud platform, we ran penetration tests on the authentication path across large volumes of edge gateways and dashboard traffic, including an extreme case where an old RT was intercepted and replayed.  
The previous design had AT/RT separation, but the refresh path did not fully enforce state transitions, so old RTs could still be abused in narrow concurrency windows.

Desensitized event sample:

```json
{
  "event": "auth.refresh.reuse_detected",
  "user_id_masked": "u-***39",
  "session_id": "s-***b1",
  "ip_masked": "10.**.**.21",
  "action": "session_revoked"
}
```

---

## Strategy: Keep AT Stateless, Strengthen RT Governance

We kept the existing AT/RT architecture and focused on hardening the RT refresh path to make it revocable, auditable, and operationally manageable.

## Layer 1: Keep Token Responsibilities Clear

- AT handles high-frequency access via `Authorization: Bearer`
- RT is delivered only through HttpOnly Cookie
- AT validation remains stateless for performance and scalability

This keeps the access path lightweight while concentrating control in the refresh path.

## Layer 2: Make RT Refresh a Single-Use Traceable Flow

Refresh follows a strict sequence: validate old RT -> mint new RT -> update Redis state -> invalidate old RT.

Redis key model (desensitized):

```text
rt:active:{jti}           -> { user_id, session_id, exp }  // primary state, TTL follows RT expiry
rt:session:{session_id}   -> Set<jti>                      // secondary index for one-shot session cleanup
rt:deny:{jti}             -> 1 (TTL=remaining lifetime)    // deny-list for replayed legacy tokens
user:sessions:{user_id}   -> Set<session_id>               // user-level control, supports "sign out other devices"
```

Controls:

- one RT can refresh successfully only once
- old RT is deny-listed immediately after rotation
- short refresh lock prevents race-based double refresh

### Concurrency Debounce Design (Result Reuse Window)

Modern SPA clients often emit multiple requests in parallel. Without coordination, one legacy RT can trigger multiple refresh attempts and cause valid traffic to be mistaken as replay. We introduced a 5-second reuse window:

- the first request acquires the lock and completes rotation
- the generated AT/RT pair is cached for 5 seconds
- concurrent requests with the same key reuse that exact pair within the window, with no second rotation
- after the window expires, the flow returns to strict single-use rotation

## Layer 3: Handling Replay Events

When RT replay is detected, the system revokes the related session, writes an audit event, and drives the client back to login.  
The goal is to terminate risky sessions quickly instead of trying to patch them in place.

## Layer 4: Tie Logout and High-Risk Operations to Revocation

Session invalidation is linked to:

- user logout
- admin password reset
- account lock/disable transitions

This upgrades logout from browser-only cookie clearing to actual server-side session invalidation.

---

## Root Cause

This was a lifecycle governance gap rather than a single coding mistake:

1. Early implementation focused on issuance and verification.
2. Session-state capability existed technically, but was not fully integrated into the refresh critical path.

---

## Outcomes

Three practical improvements after hardening:

1. sessions can be revoked quickly server-side
2. RT replay becomes detectable and attributable
3. security incidents are easier to triage by user and session scope

User experience remains stable: normal traffic keeps silent refresh, while risky paths degrade to explicit re-login.

---

## Process Improvements

### 1) Add Token Lifecycle Checks to Release Gate

Pre-release checks now include:

- RT single-use validation
- refresh concurrency consistency
- session invalidation checks after logout/password reset

### 2) Standardize Security Audit Fields

Audit logs use desensitized fields consistently: `user_id_masked`, `session_id`, `jti_prefix`, `ip_masked`, `ua_hash`, `risk_level`.

### 3) Run Regular Replay Drills

A monthly RT replay drill verifies that detection, revocation, and recovery paths still work end to end.

---

## Phase 2 Roadmap: From Session Control to Contextual Zero Trust

After this round, Phase 1 goals are in place: Redis-backed session revocation and concurrency-safe anti-replay controls. The refresh path already enforces real-time user status checks and blocks token issuance immediately for locked or disabled accounts.

Phase 2 focuses on advanced hijack scenarios under contextual risk:

- Context-aware risk controls: introduce environment fingerprint comparison (IP range + UA hash). On abnormal geographic RT jumps, the system will revoke all AT/RT, freeze the account, and enforce third-party step-up identity verification.
- False-positive management: because industrial field networks may switch 4G/5G links frequently and legitimately, the policy will be promoted into the critical path only after risk-model tuning and controlled rollout.

---

## Closing

Authentication resilience is less about token issuance speed and more about containment speed during abnormal events.  
The key value of this hardening effort is moving AT/RT from “works” to “governable”.
