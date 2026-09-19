---
title: "Linux Release CI/CD Postmortem: \"It Runs\" Is an Appearance, Not Evidence"
date: 2026-09-18T12:00:00+08:00
draft: false
tags: ["C++", "Qt", "Linux", "Windows", "Industrial", "CI/CD", "Multithreading", "Postmortem"]
categories: ["industrial-software"]
summary: "How the first automated Linux release pipeline for Modbus-Tools (Qt 6 / C++) converged over multiple CI rounds. Six of the seven cross-platform failure classes share one pattern: \"it ran elsewhere\" is merely a localized surface phenomenon, not proof of correctness. From GNU ld member stripping and epoll/Winsock event ordering, through undefined behavior from two Qt socket engines fighting over one fd, to ELF RUNPATH contract splitting — building a deterministic release pipeline."
url: "/en-us/blog/tech/linux-release-ci-postmortem/"
---

> **TL;DR**: The first automated Linux release pipeline for [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools) (Qt 6 / C++) took multiple CI rounds to converge. Six of the seven failure classes share one root cause — **"it ran elsewhere" is merely a localized surface phenomenon, not evidence of correctness; cross-platform delivery must rely on explicit contracts, not implicit environments.** End state: full cross-platform test suites passed reliably, self-contained packaging closed end-to-end, and release assets automated for distribution.

---

## A Release Pipeline That Had Never Closed End-to-End

Modbus-Tools's release pipeline is triggered by `v*` tags on GitHub Actions, running parallel jobs across Windows and Linux. The Linux job executes inside an `ubuntu:22.04` container on an `ubuntu-latest` host (pinning glibc 2.35 as the ABI baseline), installs Qt 6.10.3 via aqtinstall (gcc_64 / Ninja), executes the full unit test suite under `RelWithDebInfo` + ASan + `offscreen`, then feeds clean `Release` build artifacts to the `deploy-linux.sh` deployment script for self-contained packaging and static verification, generating a standalone `.tar.gz` bundle.

During the same timeframe, the Windows pipeline passed QA, packaging, and publishing seamlessly, uploading 4 release assets. The Linux pipeline, however, had never achieved end-to-end completion. Rehearsing the release required multiple CI rounds to converge, and failures cascaded strictly sequentially: a release pipeline is an elongated serial chain — once an upstream stage breaks, downstream logic never gets a chance to execute. Each blocker resolved merely allowed the next defect layer to execute for the first time. The process resembled peeling an onion, where every layer consumed a full CI cycle.

First, set aside two one-off infrastructure bootstrap issues: the CPython runtime embedded in GitHub's official action was linked against glibc 2.38 and crashed immediately in the glibc 2.35 container due to missing dynamic symbols (`GLIBC_2.38 not found`); and `libdbus-1-dev`, required by official Qt binaries, was missing from the container image. Once these two environment blockers were cleared, the remaining seven failure classes became the core subject of this postmortem. In hindsight, six of them answer the exact same question: why did they pass seamlessly on Windows CI and local dev machines before?

---

## Why Did They Pass on Windows CI and Local Environments?

Tracing the root causes of the six prior "passes," each relied on implicit fault tolerance in specific environments:

| Failure Class | Symptom | Root Cause of Prior Admission (Windows CI / Local) |
| --- | --- | --- |
| i18n Resource Loss | 5 cases of `QTranslator::load` returning `false` | **Windows CI (MSVC)** preserves static library members with CRT init segments by default; GNU ld's member-dropping behavior was never exposed. |
| Swallowed Socket Error | Warning logged, but signal assertion `false` | On **Windows CI**, `errorOccurred` arrives before `stateChanged`, clearing the guard; on Linux, the order is inverted. |
| Loopback Communication Deadlock | Data stream stalled, minimal repro SEGFAULT | Two engines contending for the same fd is undefined behavior; **Windows CI** merely tolerated redundant `close()` calls without crashing. |
| POSIX Portability Trap | Windows CI compilation failed | An inverted example: **Linux local** always provides `unistd.h`; the defect only surfaced under MSVC compile-time checks. |
| Qt Dependency Collection Failure | Step 2 reported `no Qt libraries collected` | The dynamic loader on the **Linux dev machine** (`ldconfig` cache, environment variables) was already configured with Qt install prefixes. |
| Wayland False Positive | Step 5 blocked `libwayland-cursor` | **Real desktop environments** always include Wayland sessions; the verifier mistook container minimalism for a bundle defect. |

A passing test merely proves that defects were not triggered within that specific semantic boundary. In cross-platform engineering, "running successfully" is only an unverified hypothesis; the comprehensive conditions under which it holds are the real engineering evidence.

---

## Static Library Linker Semantics: Unreferenced Symbols Cause Embedded Resources to be Dropped

Five i18n tests failed on assertion because `QTranslator::load` returned false — neither the Qt resource system `:/i18n/` nor the filesystem fallback matched. The identical test suite passed completely on Windows CI, yet reproduced deterministically on local Linux.

The root cause stems from differing static library member extraction semantics between linkers:

```text
qrc compilation output .o: Contains only Q_CONSTRUCTOR_FUNCTION initializers (calling qRegisterResourceData at runtime);
                           defines no unresolved symbols referenced by external code.
GNU ld (Linux):           Follows single-pass archive scanning, extracting .o members from libui.a only if they
                           resolve currently undefined symbols -> These .o files are dropped entirely.
MSVC (Windows):           Retains object files containing CRT initialization sections (.CRT$XC*) by default -> Never exposed.
```

Root-cause verification requires just one command; when failing, it outputs nothing (note: `strings` cannot find resource paths because rcc encodes internal qrc paths in UTF-16 binary format by default):

```bash
nm -C build_qa/tests/test_ui_widgets | grep -i qRegisterResourceData
```

Before applying a fix, three facts dictated our technical direction. First, the blast radius extended beyond tests: the production binary also statically links `libui.a`; without a clean fix, runtime language switching in the Linux release would fail silently — test failure was the front-line sentinel of a production defect. Second, the 7 local failures matched the CI failure set 1:1 in count and category, proving this was a deterministic toolchain behavior rather than flaky CI container jitter. Third, platform conditional branches were rejected: alternatives included OBJECT libraries, explicit `Q_INIT_RESOURCE` calls, and moving qrc files up to executable targets. We ultimately chose CMake 3.24+'s native `$<LINK_LIBRARY:WHOLE_ARCHIVE,ui>` — a zero-intrusion, unified abstraction covering both the production app and 3 test targets, automatically mapping to `/WHOLEARCHIVE:ui.lib` on MSVC with identical semantics and zero `#ifdef`:

```cmake
target_link_libraries(Modbus-Tools PRIVATE $<LINK_LIBRARY:WHOLE_ARCHIVE,ui>)
```

---

## Event Ordering is Not Guaranteed; Descriptor Ownership Must Be Exclusive

### Transient State Race Conditions Swallowing Error Events

Concrete evidence had long been recorded in the logs: `[warning] socket error ... Connection refused` was clearly output, confirming `onSocketError` was invoked, yet the test assertion `errorEmitted` remained false — the underlying network error occurred, but the outward domain signal was never dispatched. The test elapsed 2018ms, exhausting its timeout window without receiving the expected signal.

The root cause lies in fundamental timing differences in how Winsock and epoll dispatch underlying events into the Qt event loop:

* Windows (Winsock): Upon a network error, `errorOccurred` typically arrives before `stateChanged(UnconnectedState)`.
* Linux (epoll): When a connection is refused, epoll reports error events immediately; Qt's internal state machine transitions ahead of time, with `stateChanged(UnconnectedState)` resetting the state to `Closed` before `errorOccurred` is delivered.

The original guard relied on transient state:

```cpp
// Defective code: relies on transient state, swallowing errors under Linux event ordering
if (state_ != State::Opening && state_ != State::Open) {
    return; // Discards as historical noise from an inactive connection
}
```

On Linux (epoll), the error signal arrived one step late when the state was already `Closed`; the guard discarded the event as stale historical noise. Consequently, if a Linux user attempted to connect to an unreachable port, the UI would stall silently with zero error feedback — a genuine product defect, not a test harness anomaly.

The key architectural fix was reframing the state arbitration model. Instead of asking "what is the instantaneous state right now?", the guard must arbitrate "does this error belong to the current connection session?". We introduced a session flag `socketDropIsError_`: set upon initiating or adopting a connection, and reset upon active close, timeout, or error consumption. As long as an error belongs to the active session, it is dispatched reliably even if the state machine has already transitioned to `Closed`:

```cpp
// Corrected logic: based on session lifecycle arbitration
if (state_ != State::Opening && state_ != State::Open) {
    if (state_ == State::Closed && socketDropIsError_) {
        // Pending drop belonging to the current session; dispatch error normally
    } else {
        return;
    }
}
```

### Multi-Engine Descriptor Contention: From ::dup() Portability Traps to Object Adoption

The `NetworkDebuggerLoopback` test harbored two distinct defect layers. The surface layer was a thread affinity violation: `QTcpServer server_` was a value member without a parent; when worker threads invoked `moveToThread`, the server remained pinned to the main thread, causing `nextPendingConnection` to spawn child objects across threads and spamming Qt thread-affinity warnings. The deeper layer was severe undefined behavior (UB): even with warnings silenced, the data stream deadlocked. A minimal isolated repro proved: the original implementation retained `sourceSocket` while handing its raw native fd to `TcpChannel` — two internal Qt Socket Engines simultaneously listened for read-readiness on the same system fd, inducing read starvation; the isolated repro triggered a segmentation fault (SIGSEGV) outright. Windows CI passed previously only because the OS kernel tolerated duplicate releases on closed handles without crashing.

The first iteration used POSIX `::dup()` on local Linux to duplicate a private fd and retire the original socket, passing 100% of local tests. Once pushed to Windows CI, MSVC failed at compile time:

```text
TcpServerHandle.cpp(15,10): error C1083: Cannot open include file: 'unistd.h'
```

The `_dup()` path was equally a dead end: on Windows, `socketDescriptor()` returns a Winsock `SOCKET` handle rather than a CRT file descriptor; switching to `WSADuplicateSocket` would require including `<windows.h>` and littering the codebase with platform `#ifdef` branches, violating our project constraints.

In cross-platform C++, when a platform-specific syscall lacks a clean equivalent on the other side, the abstraction level is almost always wrong.

We abandoned descriptor-level replication and destruction patching in favor of Object Adoption:

```cpp
// TcpServerHandle delivers object ownership:
channel->adoptSocket(socket);

// TcpChannel safely adopts: detaches from QTcpServer tree, owns native fd, and rewires signals
socket->setParent(nullptr);
delete socket_;
socket_ = socket;
wireSocketSignals();
```

Managing lifecycle handover purely through the Qt object tree eradicated engine contention, POSIX portability traps, and platform conditional branches simultaneously.

---

## Passing Locally Doesn't Count: Implicit Loader Dependencies, Verification Contracts, and the Offscreen Blind Spot

### Implicit Dependencies of the Dynamic Loader

The first time the packaging script executed inside the CI container, Step 2 exited with `no Qt libraries collected`. A chain of cascading misses: `cmake --install` stripped the binary's build rpath; the container had neither `LD_LIBRARY_PATH` nor the Qt prefix registered in ldconfig; and at that instant, the bundle's `lib/` directory was still empty — `ldd` dependency resolution deadlocked.

The dev machine succeeded purely due to uncontrolled implicit dependencies in the local environment. Our fix adhered to explicit declaration and minimal scoping: rather than polluting the global container environment, we injected a process-scoped isolation prefix exclusively during the `ldd` static analysis step:

```bash
LD_LIBRARY_PATH="${QT_LIBS}" ldd "${TARGET_BIN}"
```

### Verification Logic Failure: Auditing the Verifier's Own Specification Contract

Step 5 hermeticity checks intercepted `libwayland-cursor.so.0 not found`, declared by `libQt6WaylandClient.so.6`.

Three technical paths emerged: the most careless choice was running `apt install libwayland-cursor0` in the CI container, which merely patches an environment to appease flawed verification semantics; packaging only xcb plugins would narrow the dependency surface, but abandoning native Wayland support is a product decision that should not be usurped by a CI fix; packaging Wayland system libraries into the bundle would directly violate the design principle that host system libraries are not collected. All three were rejected. In real operating scenarios, user desktops running Wayland sessions are guaranteed to have this library; in minimal X11 environments, Qt degrades gracefully to xcb. A container is not a desktop; therefore, the verifier's contract was refactored with strict ownership boundaries:

1. Prevent prefix leakage: No ELF binary may resolve back to the build machine's Qt installation prefix.
2. Release integrity: Among unresolved items, any shipped with official Qt distribution packages (`$QT_LIBS`) must be bundled; missing host-provided system libraries belong to the host baseline and are tolerated.

While adding a 4-branch test harness for this validation logic, we uncovered a long-dormant bug in the legacy script:

```bash
# Actual ldd output sample: "libwayland-cursor.so.0 => not found"
# Field breakdown: $1="libwayland-cursor.so.0", $2="=>", $3="not", $4="found"

# Legacy script (Vacuously True bug: space-delimited, $3 can never equal "not found"):
awk '$3 == "not found"'

# Fixed (strict multi-field match):
awk '$2 == "=>" && $3 == "not" && $4 == "found"'
```

Untested verification logic is production code. The legacy illusion of passing silently was merely validation blindness (Unexercised Validation).

### RUNPATH Depth: The Structural Blind Spot of Offscreen Smoke Testing

The seventh defect fell outside the "passed elsewhere" pattern — it was introduced when the deployment script rewrote RUNPATH, corrupting a previously valid value.

The script blindly assigned `$ORIGIN/../lib` to all plugins. However, plugins are organized two levels deep under `plugins/<category>/lib*.so`; this relative path resolved to non-existent `plugins/lib`. Official Qt precompiled plugins natively baked in `$ORIGIN/../../lib`, which was correct all along.

Why did automated smoke testing (Step 6) fail to notice? CI smoke tests ran under `QT_QPA_PLATFORM=offscreen`, where base libraries (QtCore / QtGui) needed by the offscreen platform plugin were already preloaded into memory by the main application. Real X11 desktops behave fundamentally differently: dynamically loading `libqxcb.so` requires resolving its exclusive `libQt6XcbQpa.so.6` (which the main application does not link). Once RUNPATH was broken, double-clicking the application on a desktop crashed immediately due to a failed platform plugin load.

Fix one layer, expose the next. Offscreen testing suffered a structural blind spot due to symbol preloading; static assertions based on the ELF dependency graph served as the final safety net against launch crashes. Post-fix bundle audits verified: 26/26 plugins had consistent RUNPATHs, zero Qt prefix leaks, and zero unresolved dependencies under bare `ldd`, which Step 5 subsequently re-verified in CI.

---

## Which Existing Defense Mechanisms Worked as Intended (What Went Well)

A postmortem must not only list failures. Several existing mechanisms functioned exactly as designed:

- Local baseline matched CI failure sets 7/7: Almost all fixes converged locally, ensuring CI cycles were spent validating genuine environmental differences rather than burning pipelines on known issues.
- Windows CI served as a compile-time sentinel for POSIX contamination: `unistd.h` passed silently on local Linux, but Windows CI (MSVC) blocked it on the very first commit.
- The full test suite (with ASan dynamic instrumentation) caught genuine undefined behavior (dual-engine fd contention) and product defects (swallowed errors) prior to release — a high-quality test suite is the earliest alarm.
- Step 5 immediately caught the deep plugin RUNPATH defect on the first end-to-end rehearsal following verification refactoring: layered defenses functioned as designed.

## Four Defenses That Stuck

From the initial pipeline failure to full cross-platform test suites passing reliably and multi-platform release assets automated for delivery. In hindsight, sequentially exposed failures were merely symptoms; the root causes were two long-standing realities — the Linux release chain had never been executed end-to-end, and multiple instances of "working" rested on local environmental anomalies. Multi-round convergence crystalized into four architectural defenses, each backed by landed mechanisms directly reusable in other cross-platform C++/Qt delivery pipelines:

1. "It ran elsewhere" usually means it escaped undefined behavior penalties. Linker archive pruning, async event delivery timing, and native OS handle abstractions — every "appears normal on the other side" often means that environment accidentally tolerated undefined behavior. Hard engineering constraints followed: full test suites with ASan on both platforms, where any assertion failure unconditionally gates the release, eliminating any compromise on shipping broken builds.
2. When the system pushes you toward `#ifdef`, examine the abstraction level first. Clumsy platform branching typically stems from prematurely tangling with OS-specific handles; elevating control to object lifecycle ownership eliminates platform divergence. Landed outcome: socket takeover converged to a single `adoptSocket` channel, preserving zero platform conditional branches in project code.
3. Verification logic is production code and must be thoroughly tested. Vacuously true checking scripts are far more hazardous than explicit failures — silent passage often masks total validation failure. Landed outcome: the `check_ldd` core function was extracted from production scripts to directly drive a 4-branch test harness (including positive controls), ensuring zero drift between verification logic and its test suite.
4. Defend static invariants rather than patching environmental symptoms. When containers report missing dependencies, probing deeper clarifies contract boundaries: distinguishing "which dependencies must ship with the application bundle" from "which must be provided by host distributions" gives delivery pipelines determinism. Landed outcome: Step 5's two static invariants (zero prefix leakage, zero missing Qt-bundled dependencies) became hard packaging gates; minimalist container baselines no longer trigger false positives.

Finally, here is a reusable release checklist. None of the diagnostic techniques depend on this project: use `nm` to verify that resource registration symbols are preserved; isolate undefined behavior via minimal repros; replace global environment pollution with process-scoped `LD_LIBRARY_PATH`; enforce strict multi-field comparison on `ldd` outputs rather than regex fuzzy matching; and perform full-bundle unpack audits to inspect RUNPATH and prefix leakage. If your pipeline is currently stuck in "passes locally, dies on CI," walk through this checklist first.
