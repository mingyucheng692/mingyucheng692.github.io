---
title: "Rootless Podman + Systemd Supervision Failure Postmortem: Diagnosing and Repairing a Broken Recovery Path"
date: 2026-04-03T20:00:00+08:00
draft: false
tags: ["Podman", "Systemd", "Rootless", "Container", "Nginx", "Redis", "Go", "Postmortem", "SRE"]
categories: ["Tech", "Engineering", "Infrastructure"]
summary: "A postmortem on a failed recovery path under Rootless Podman + Systemd user units, covering confirmed contributing factors, concrete remediation work, validation steps, and remaining risks."
url: "/en-us/blog/tech/rootless-podman-systemd-watchdog-postmortem/"
---

This post documents the investigation and repair of a failed container recovery path. In a service stack managed with Rootless Podman, a database container exited unexpectedly and was not restarted as intended. That initial miss then exposed several adjacent failures: startup timeouts, services reported as `inactive` while containers were still online, and rate-limit lockouts after repeated restart attempts.

> Note: project names, service names, paths, domains, usernames, host identifiers, timestamps, PIDs, command outputs, and log snippets in this article are all desensitized, abstracted, or rewritten placeholders. They preserve only the technical details relevant to the troubleshooting path and do not map to any real production identifiers.

- Runtime: Rootless Podman 4.9.x
- Process supervision: Systemd user units
- Service components: TimescaleDB, Redis, Go backend, Nginx frontend
- Intended capability: let Systemd recover containers automatically after abnormal exits

## Incident Summary

- Trigger context: after the database container exited unexpectedly, the first on-site state was that the database Unit became `inactive` while the other service Units still appeared `active`
- Observable symptoms: once the repair work began, the database service recovered to `active`, but the other services were not yet reliably handed over to systemd, which led to startup timeouts, container state drifting away from Unit state, and rate-limit lockouts on some services
- Confirmed contributing factors: the main Unit had been edited in place with `sed -i` and was later corrupted after a PowerShell escaping truncation; Compose and Systemd shared control of the runtime; generated Unit files still required post-processing; and the recovery scripts did not yet fully cover dependency cleanup and restart-failure handling
- Fix direction: stop editing the main Unit with `sed -i`, regenerate `.service` files and patch them in Python, explicitly hand runtime control back to systemd, and add state polling plus a final health check
- Current result: in the current environment, the recovery path now runs in a fixed order and exposes a clear diagnostic entry point on failure; readiness semantics and observability still need more work

## Symptoms and Impact

The initial on-site state was not "everything went down at once." The database Unit first turned `inactive` while the other services still looked `active`. The picture became more complicated during the repair and handoff phase:

- some services showed timeout-based restart loops in `systemctl --user status`
- some containers were already `Up` while the corresponding systemd services were still `inactive`
- some services had failed often enough that systemd stopped retrying entirely

That meant this was not just one container failure. The real issue was drift between three things: actual container state, systemd supervision state, and the assumptions baked into the deployment scripts. This write-up focuses on why the recovery path failed and how the scripts were adjusted afterward. It does not try to expand on the business-side reason the database exited in the first place.

---

## Investigation Approach: First Identify Who Actually Owns the Process

We did not begin with a single error line. The first question was more basic: when a container exits, who is actually responsible for noticing the failure, deciding it is unhealthy, and bringing it back. Following that thread eventually exposed four overlapping problems.

## Problem 1: The Main Unit Was Edited In Place and Eventually Corrupted

The first visible issue was that the database unit file had been damaged into an empty file:

```text
systemd[<pid>]: /home/<deploy-user>/.config/systemd/user/container-<db-service>.service:1: Missing '='.
```

Inspecting the file itself showed that it had been truncated to `0` bytes:

```bash
ls -l /home/<deploy-user>/.config/systemd/user/container-<db-service>.service
# -rw-rw-r-- 1 <deploy-user> <deploy-user> 0 <timestamp> container-<db-service>.service

file /home/<deploy-user>/.config/systemd/user/container-<db-service>.service
# container-<db-service>.service: empty
```

At this point we can describe the trigger chain more concretely. The older script did in fact modify the main Unit with `sed -i`. When that command was dispatched from PowerShell, escaping was handled incorrectly, the target file was not written back successfully, and the main Unit was truncated to `0` bytes. The issue was not merely that the script used `sed -i`; it was that the main Unit was treated as something safe to edit in place, and that choice was then amplified by cross-shell escaping differences.

The older script path looked roughly like this:

```bash
# old approach: edit the main Unit directly
sed -i 's/Restart=always/Restart=no/g' "$service"
```

The design problem is clear enough on its own: the main service file was being used both as the runtime definition and as an operational toggle surface. Once the main Unit breaks, parsing, startup, and restart policy all fail together.

The later adjustment was straightforward:

- keep the main service file as a generated artifact only, and stop editing it with `sed -i`
- move restart-policy switches and similar runtime knobs into `.service.d/watchdog.conf`
- handle compatibility patches for generated `.service` files in Python, while letting the script overwrite Drop-ins explicitly

In the current script, the Drop-in write path looks roughly like this:

```bash
cat > "$DROP_IN_DIR/watchdog.conf" << EOF
[Service]
Restart=always
RestartSec=10s
ExecStartPre=-/usr/bin/podman rm -f <db-service>
EOF
```

This at least restores a cleaner boundary: the main Unit is treated as a rebuildable artifact, while runtime switches live in Drop-ins. Based on the current scripts and subsequent field checks, we have not seen the same "main Unit damaged by in-place edits" problem recur.

## Problem 2: Systemd Reported Failure Even Though the Container Was Already Running

The second issue was subtler. During the incident, systemd marked some services as startup timeouts:

```text
Active: activating (auto-restart) (Result: timeout)
Main PID: <pid> (code=exited, status=0/SUCCESS)
```

At the same time, `podman ps` showed that the container itself was already up. In other words, the process contract had split.

`podman generate systemd` uses `Type=notify` by default. That means systemd does not consider process existence alone to be enough for startup success; it expects an `sd_notify` readiness signal from the running container back to the host.

Before the recovery path was patched, the generated Unit still contained the default fields:

```ini
Type=notify
NotifyAccess=all
```

When we re-collected the environment after the repair, freshly generated output still carried that default, while the Unit that actually went live after the fix had already been changed to `Type=simple`:

```ini
# default generated result during the incident path
Type=notify

# effective Unit after the repair went live
Type=simple
```

What we can confirm from the incident is this: while services were still using the default generated Unit with `Type=notify`, containers could already be `Up` and yet the corresponding Unit would still be treated as timed out and restarted by systemd. The repair changed generated Units from `Type=notify` to `Type=simple`, shifting status recognition back to "systemd tracks the foreground process directly."

The boundary here matters. This is not proof that `notify` is categorically unusable in all Rootless scenarios. It only shows that, in this environment, the readiness semantics delivered by `notify` were not stable enough to rely on. For these long-running foreground processes, `Type=simple` was the safer and easier-to-debug choice.

## Problem 3: Some Containers Were Not Actually Under Systemd Control

More precisely, this dangerous state showed up during the repair process rather than in the initial incident snapshot: the database Unit had already recovered to `active`, while the other service Units still showed `inactive (dead)`.

The state on the host looked roughly like this:

```bash
podman ps
# ... some service containers might already be back up

systemctl --user list-units 'container-*.service'
# ... container-<db-service>.service was active
# ... the other service units were inactive (dead)
```

The reason was that those containers had not been started by systemd in the first place. They had been launched earlier through `podman-compose up -d`. In that state, even if the container process already exists, systemd still does not own it and therefore cannot take over cleanly, detect failure consistently, or recover it.

That exposed an easy-to-miss prerequisite: supervision capability depends not only on whether a Unit file exists, but on whether the process is actually held by systemd. If runtime control still belongs to Compose, then "restart configuration exists" and "recovery really works" are not the same thing.

An implementation detail also matters here: the current deployment script does not eliminate Compose entirely. It still uses `podman-compose` for build and cold start, then calls `watchdog-enable.sh` to generate and patch Units, explicitly stops the Compose-launched containers, and finally hands control over through ordered `systemctl --user start` calls. The issue was never "Compose appeared in the script"; it was whether Compose still owned the runtime process when recovery was expected. The later change made that handoff explicit:

- stop existing non-supervised containers and let `systemctl --user start` take over
- add state checks around the handoff so the system does not silently fall back into "container is running but systemd does not know it"

## Problem 4: Generated Units and Recovery Scripts Still Needed More Detail Work

### 1) Generated Units Still Required Post-Processing

After switching back to `podman generate systemd --new`, at least three categories of issues showed up:

- parameters introduced by tools such as `podman-compose` can leak into the generated result; for example, if the original container was created with `-d`, `podman generate systemd --new` will carry that forward, which makes `podman run -d` exit immediately from systemd's point of view and prevents it from tracking the real container process
- Redis is another typical case: if the original configuration disables dangerous commands with an empty argument such as `rename-command ""`, the generator silently drops the empty string and turns the final command into `redis-server --rename-command FLUSHALL`, which causes startup-parameter errors
- the default `Type=notify` value would reintroduce the same Rootless timeout problem

The Redis case is directly visible by comparing original container arguments with the generated output:

```yaml
# original container arguments
command:
  - --rename-command
  - "FLUSHALL"
  - ""
```

```text
# unpatched generated result
redis-server --rename-command FLUSHALL
```

Even after recovery, rerunning `podman generate systemd --new --name <redis-service>` still shows the empty string being dropped, while the effective Unit already has it restored:

```ini
# regenerated output after recovery
--rename-command FLUSHALL

# effective Unit after the fix went live
--rename-command FLUSHALL ""
```

`-d` is the same kind of problem. Comparing regenerated output with the effective Unit after the fix shows that the default generated result still contains `-d`, while the Unit used for supervision removes it:

```ini
# regenerated output after recovery
ExecStart=/usr/bin/podman run \
  ...
  -d \
  --sdnotify=conmon \

# effective Unit after the fix went live
ExecStart=/usr/bin/podman run \
  ...
  --sdnotify=conmon \
```

The current `watchdog-enable.sh` workflow is therefore: rerun `podman generate systemd --new --name ...`, then patch the generated result in Python. In practice, it does three things:

- change `Type=notify` to `Type=simple`
- remove any leftover `-d` from `ExecStart`
- restore Redis `rename-command ""` so empty arguments are not lost during generation

Python was chosen for a specific reason: it handles multi-line commands split with trailing `\` more safely and predictably.

### 2) Dependency Relationships Affect Cleanup of Failed Containers

Before the database service could recover, the script first had to run `podman rm -f` to remove the old container. But because upstream services still held dependency references, the delete returned `exit 125`. That means single-container recovery is not isolated; the dependency graph can push back on cleanup.

The direct error looked like this:

```text
Process: ExecStartPre=/usr/bin/podman rm -f <db-service> (code=exited, status=125)
```

The current script handles this by putting dependency cleanup into the Drop-in:

- for the database service, `ExecStartPre` first stops dependent upper-layer services, then runs `podman rm -f <db-service>`
- for the other services, each one only cleans up its own container

This is not a generic template. It is just the scripted handling required by this service graph.

### 3) The Recovery Path Cannot Rely on Fixed `sleep`

The earlier script did in fact rely on fixed `sleep` windows to wait for dependencies.

The current deployment script still keeps a small amount of fixed waiting in the Compose cold-start phase. But after control is handed to systemd, startup is now ordered explicitly through `systemctl --user start`, with `wait_for_service()` polling `is-active` and `is-failed`, and printing the relevant `journalctl` output when a failure or timeout occurs. That is far easier to debug than blind waiting.

The final startup order is therefore made explicit:

- `<db-service>` -> `<redis-service>` -> `<backend-service>` -> `<frontend-service>`

After all services are handed over, the script also adds one application-level health check to verify that "`active`" is at least moving closer to "externally usable." That is a compensating measure for the `Type=simple` tradeoff, not a deep readiness probe on every service startup.

The wait logic after handoff looks roughly like this:

```bash
systemctl --user start container-<db-service>.service
for i in $(seq 1 60); do
  systemctl --user is-active --quiet container-<db-service>.service && break
  systemctl --user is-failed --quiet container-<db-service>.service && exit 1
  sleep 1
done
```

### 4) Systemd Rate Limiting Can Lock Services Out

When dependencies are still not ready, Frontend may repeatedly fail and restart. For example, Nginx tries to resolve the upstream name for `<backend-service>` at startup time. Once restart attempts exceed `StartLimitBurst` within the configured window, systemd stops retrying. At that point, even if dependencies become healthy later, the service will not come back automatically unless we reset the failure state explicitly:

```text
container-<frontend-service>.service: Start request repeated too quickly.
container-<frontend-service>.service: Failed with result 'exit-code'.
```

On the Nginx side, the application-level evidence is often more explicit:

```text
[emerg] 1#1: host not found in upstream "<backend-service>" in default.conf:31
```

```bash
systemctl --user reset-failed container-<frontend-service>.service
```

`deploy-all.sh` now inserts a `reset-failed` call before starting Frontend to clear any residual lockout state.

---

## Confirmed Contributing Factors in This Incident

On the surface, the incident looked like a database failure followed by drifting service and supervision states. Looking back, it was really a stack of contributing issues:

1. the main Unit was edited directly by scripts, creating a risk of service-definition corruption
2. `podman-compose` startup and systemd supervision coexisted, splitting container state from supervision state
3. the output of `podman generate systemd` was not directly runnable and still needed `Type`, `-d`, and argument patching
4. the recovery scripts initially did not cover dependency cleanup, rate-limit recovery, and final health validation thoroughly enough

This was not one flag set incorrectly. The recovery path had breaks across definition, handoff, and execution. The repair therefore was not about fixing one error line; it was about reconnecting runtime ownership, generation-time patching, and failure-time diagnostics into a single workable path.

---

## Remediation Already Landed

Combining `watchdog-enable.sh` and `deploy-all.sh`, the practical changes now in place are roughly:

- the deployment script validates execution user, `HOME`, `XDG_RUNTIME_DIR`, and `DBUS_SESSION_BUS_ADDRESS` first, reducing the chance that Rootless user units fail because of an inconsistent runtime environment
- the deployment phase still uses `podman-compose` for build and cold start, but the handoff phase explicitly stops those containers and gives ordered control back to systemd
- `watchdog-enable.sh` backs up old Units before regenerating `.service` files
- generated `.service` files are now patched in Python in three ways: change to `Type=simple`, remove `-d`, and restore Redis empty arguments; this also avoids repeating the PowerShell escaping issue that broke the earlier `sed -i` path
- Drop-ins now write `Restart=always`, `RestartSec`, `StartLimit*`, and service-specific `ExecStartPre`
- after handoff to systemd, services start in the fixed order `<db-service> -> <redis-service> -> <backend-service> -> <frontend-service>` and `wait_for_service()` checks `is-active` and `is-failed`; failures and timeouts print the corresponding `journalctl`
- before Frontend is started, the script runs `reset-failed` once to clear any leftover rate-limit lockout
- after all services are under systemd control, the script adds one application-level health check so the system is verified beyond mere process liveness

This is still an engineering solution under a specific set of constraints: single host, Rootless Podman, and user units. It solves who owns the process, how recovery is executed, and where to look when it fails. It is not a claim of generalized high availability.

---

## How the Fix Is Verified

- compare `podman ps` with `systemctl --user list-units` before and after handoff, confirming that we no longer leave the system in a state where containers are running but the corresponding Units are `inactive`
- use `wait_for_service()` to poll `is-active` and `is-failed`, exposing failures during startup instead of hiding them behind fixed `sleep`
- print the relevant `journalctl` immediately on timeout or failure so each problem maps back to a specific Unit
- add a final application-level `/health` check after all services are handed over, so external availability is not inferred from `Type=simple` process liveness alone

A later verification pass showed that all four containers and all four user units were back in sync:

```text
podman ps
<db-service> / <redis-service> / <backend-service> / <frontend-service> all Up

systemctl --user list-units 'container-*.service'
all four corresponding Units active (running)
```

These checks mainly cover whether systemd truly owns the process, whether failures are exposed in time, and whether the system becomes externally usable after handoff. They still do not cover finer-grained readiness signals or ongoing telemetry collection.

---

## Constraints Made Explicit After This Incident

- the main Unit should remain a generated artifact only, while runtime switches belong in Drop-ins; this is not ceremonial separation, but a way to avoid breaking parsing, startup, and restart behavior all at once when the main file is damaged
- `podman generate systemd` output cannot be treated as ready to run; generation, patching, and daemon reload must be considered one continuous step, and in this case `Type=notify`, `-d`, and empty-argument handling all fall into mandatory post-generation compatibility work
- deployment control and runtime control must be separated; Compose may still be useful for build and cold start, but runtime ownership must be handed back explicitly to systemd, otherwise "restart config exists" and "recovery actually works" remain conflated
- a recovery path is not complete just because `Restart=always` exists; it must also cover dependency cleanup, rate-limit recovery, state polling, and a final health check so the failure path remains repeatable and diagnosable

---

## Follow-Up Work

The current repair closes several known breaks in the recovery path, but some risks are still not fully covered and need follow-up:

- `Type=simple` plus `is-active` / `is-failed` and the final `/health` check close the gap between "process is alive" and "service is actually usable," but there is still no finer-grained readiness signal; if a service becomes `active` briefly and only then exposes an internal initialization failure, the current scripts may still detect it late
- the solution still depends on `podman generate systemd` plus script-level patching; if generated output format, argument expansion, or Podman behavior changes across versions, the current patch logic can break, which is why a move to Quadlet or another declarative path still deserves separate evaluation
- current validation remains mostly inside scripts; failure counts, exit codes, container state, and rate-limit hits are not yet collected, shipped, or alerted on, which means the broader system still lacks continuous observability and similar problems may remain reactive instead of proactively visible

## Direct Improvements from This Troubleshooting Cycle

- process ownership and restart responsibility are now much clearer
- there is now an explicit standard for deciding whether generated Units are directly usable
- the script now provides a stable entry point for where to inspect logs and where execution should stop on failure
- the gap between "service is active" and "service is externally usable" is now covered by an application-level check

These changes do not mean the whole setup has become a generalized HA design. They do mean the previous state, where restart configuration existed but recovery was still unreliable, has been turned into a path that is more testable, more explainable, and more repeatable.

## Closing Note

The core takeaway from this incident is simple: writing `Restart=always` in configuration does not mean the recovery path is complete. Main Unit stability, who actually starts the container, whether generated output is patched, and whether dependency and rate-limit behavior are handled in script all affect the final result.

At least in this single-host Rootless Podman setup, once those breaks were repaired in script, process ownership, diagnostic entry points, and the failure-handling path all became much clearer. That does not make the solution "perfect," but it does turn recovery from something driven largely by operator intuition into a path that is easier to explain and easier to verify.

---

## Appendix: A Note on the Linger Mechanism

If Rootless Podman is supervised through `systemd --user`, the target user usually needs `linger` enabled first. Otherwise, once that user logs out, the user-level systemd instance may be reclaimed and the related services may no longer stay alive without an interactive session. In this incident, the environment already had `Linger=yes`, so `linger` was not part of the root cause. It is included here only as a prerequisite check.

```bash
# check current state
loginctl show-user <deploy-user> --property=Linger

# if it is not enabled, run this as a sudo-capable user:
sudo loginctl enable-linger <deploy-user>
```
