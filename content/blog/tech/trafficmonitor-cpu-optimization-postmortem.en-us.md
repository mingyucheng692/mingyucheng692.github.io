---
title: "TrafficMonitor CPU Optimization Postmortem: An Engineering Simplification from 4% to 0.6%"
date: 2026-04-26T10:00:00+08:00
draft: false
tags: ["TrafficMonitor", "Qt", "C++", "Modbus", "Performance", "CPU", "Postmortem"]
categories: ["industrial-software"]
summary: "A postmortem on CPU optimization under a high-frequency polling workload: by shrinking the UI logging path, removing high-frequency state machines, and fixing the wait model, CPU usage dropped from 3.6%~4.2% to 0.1%~0.6%."
url: "/en-us/blog/tech/trafficmonitor-cpu-optimization-postmortem/"
---

In a `100ms` polling workload, CPU usage of the process hosting `TrafficMonitor` stayed around `3.6%~4.2%` for a long period. Beyond the elevated resource cost, the UI also showed visible refresh lag and less responsive scrolling during high-frequency traffic. After comparing two implementations, the optimized version brought CPU usage down to `0.1%~0.6%`, while noticeably reducing UI stutter under continuous polling.

The gain did not come from a single hot function. It came from a broader engineering simplification: cutting intermediate layers, state machines, and unnecessary wakeups from high-frequency paths so the system could return to a shape closer to its intended responsibility boundary.

The optimization discussed here comes from the open-source tool [Modbus-Tools](/en-us/blog/tech/modbus-tools-intro/). If you want the broader product context first, the introduction article is a better starting point.

---

## Background And Conclusion

This optimization can be reduced to two core conclusions:

- `TrafficMonitor` returns to being a lightweight traffic viewer instead of acting like a full logging system
- worker threads truly block when no event is present, rather than waking up on a fixed time slice

At the code level, the main changes fall into four areas:

- `TrafficMonitorWidget` moves from an event model back to direct text append
- `ModbusTcpView` and `ModbusRtuView` remove polling summary and suppression logic
- `ModbusClient` switches from `wait_for + 5ms slice` to `wait_until`
- `SerialChannel` and `TcpChannel` remove thread diagnostic logs from high-frequency paths

---

## Key Changes

### 1. Shrink The UI Logging Path

In the more complex version, `TrafficMonitor` introduced a full event abstraction. The logging path looked like this:

`caller -> build event object -> classify level/mode -> render text -> write to list`

The optimized version removes that intermediate layer and keeps only direct entry points such as `appendTx()`, `appendRx()`, `appendInfo()`, and `appendWarning()`. The path becomes:

`caller -> format text directly -> append to QListWidget`

This cuts object creation, event classification, second-pass rendering, and state synchronization cost. It is one of the main sources of the CPU reduction.

### 2. Move From Model/View Back To A Direct List

The complex `TrafficMonitorWidget` was not just a visual control. It also carried a full data layer, including:

- `QListView + QAbstractListModel`
- `pendingEvents_`
- `eventHistory_`
- `flushTimer_`
- `rebuildScheduled_`
- `pausedEventCount_`

This design is more feature-complete, but under `100ms` high-frequency refresh it keeps amplifying costs in:

- event enqueueing and batch flushing
- text formatting and filter checks
- history rebuilding and scroll synchronization

The optimized version goes back to direct append on `QListWidget`. In practice, that means removing a resident data-management layer. For high-frequency log display, the benefit is immediate.

### 3. Remove High-Frequency Extra Features And Polling State Machines

The complex version added several observability-oriented features:

- Pause View
- Raw Frames
- Level Filter
- paused counters
- visible list rebuild
- Poll Summary state machine

Each feature is reasonable on its own, but together they turn every log line into an event that must be evaluated and scheduled, instead of plain text that can be displayed directly.

This is especially visible in the Poll Summary logic inside `ModbusTcpView` and `ModbusRtuView`. It reduced visible log volume, but did not reduce total system work. Each polling cycle still had to update counters, maintain state, check timing windows, and format summary strings. Once this logic was removed, the high-frequency path became much shorter.

### 4. Make The Wait Model Truly Blocking

Outside the UI, the other key change was the wait strategy inside `ModbusClient`.

The complex version used `cv_.wait_for(lock, 5ms, ...)` together with an event pumping pattern. That means the thread woke up every `5ms` even when no real event existed. Under a `100ms` polling workload, that creates a large amount of useless wakeup traffic.

The optimized version standardizes on:

`cv_.wait_until(lock, deadline, predicate)`

The result is:

- threads stay asleep when no event is present
- wakeup happens only on timeout or when the condition is satisfied
- current-thread event pumping is no longer done periodically

This directly cuts worker-thread spinning and is the other main source of improvement.

### 5. Remove Debug Output From High-Frequency Paths

In `SerialChannel.cpp` and `TcpChannel.cpp`, the complex version kept thread-context diagnostics such as:

- `threadToken(...)`
- `logThreadContextOnce(...)`
- thread logs inside `open()`, `onReadyRead()`, and `onConnected()`

Each individual log is cheap, but these calls sit on high-frequency send/receive paths. Over time they amplify formatting and branch overhead. Removing them is a typical example of reducing weight on a hot path.

---

## Why The Gain Was So Visible

The value of this change is not that one function dropped from `10ms` to `2ms`. The real value is that multiple multiplicative terms were removed from a high-frequency path.

A single polling send/receive cycle in the complex version usually involved:

- I/O callback
- event object construction
- level and mode checks
- queue operations
- timer wakeup
- batch rendering
- filter checks
- history rebuild
- auto-scroll

The optimized version is much closer to:

- I/O callback
- text formatting
- direct append to the list

At a fixed `100ms` call frequency, this difference in path length keeps compounding. The final result is a CPU drop from `3.6%~4.2%` to `0.1%~0.6%`. Since the main thread no longer carries continuous pressure from event dispatch, batch refresh, and list rebuild, UI interaction also returns to a much steadier state.

---

## Engineering Takeaways

This optimization leaves a few clear lessons:

- in high-frequency scenarios, feature complexity itself is a performance cost
- GUI performance issues often come not from painting, but from event systems and state machines before painting
- if a wait can block, do not poll; if a wakeup can be condition-based, do not wake on time slices
- the clearer a logging component's responsibility is, the easier it is to keep both implementation and performance stable

In the end, this was not a story about clever micro-optimizations. It was a story about shrinking system boundaries again. Once `TrafficMonitor` returned to a lightweight viewer and `ModbusClient` returned to truly blocking waits, the system no longer paid high-frequency cost for low-frequency features, and the overall load dropped accordingly.

For a broader view of the tool itself, see the [Modbus-Tools introduction](/en-us/blog/tech/modbus-tools-intro/). Source code is available on GitHub: [mingyucheng692/Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools).
