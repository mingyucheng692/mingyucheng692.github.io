---
title: "Pitfall Log: Fixing Modbus Multithread Deadlocks"
date: 2026-03-07T10:00:00+08:00
draft: false
tags: ["Modbus", "Qt6", "C++", "Deadlock", "Multithreading", "Industrial Software"]
categories: ["Tech", "Engineering"]
summary: "A field note on shutdown deadlocks and timeout misjudgment in a Qt multithreaded Modbus workflow, with practical fixes."
url: "/en-us/blog/tech/modbus-deadlock-fix-notes/"
---

During iteration of `Modbus-Tools`, we met a classic concurrency issue: the UI window closed, but the process stayed alive and the debug session could not exit cleanly. After addressing that, timeout misjudgment and re-entrancy contention also surfaced. This note records the diagnosis and fixes.

---

## Symptom 1: Hang During Shutdown

- Main thread waits for worker thread termination while destroying `MainWindow`
- Worker thread may still be blocked on serial or socket I/O
- UI is gone, but process remains

## Cause 1: Inconsistent Thread Affinity

`QSerialPort` and `QTcpSocket` rely on the event loop of their owning thread. If created in the main thread but effectively used from worker-side flow, shutdown can fall into cross-thread waiting.

## Fix 1: Make Ownership Explicit

Move the channel object to the worker thread right after construction:

```cpp
stack.thread = std::make_shared<QThread>();
stack.channel->moveToThread(stack.thread.get());
```

Also keep `parent` unset at creation time so migration remains valid.

---

## Symptom 2: Intermittent Timeout After Connect

- Packet monitor shows device responses
- Business layer still returns timeout

## Cause 2: Blocking Wait Starves Event Loop

After sending requests, `condition_variable::wait_until` blocked the I/O thread. As a result, `readyRead` callbacks were delayed even when data had already arrived.

## Fix 2: Replace With Event-Yielding Wait Loop

```cpp
while (true) {
    QCoreApplication::processEvents(QEventLoop::AllEvents);
    if (signalReceived) break;
    if (timeout) return Error;
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
}
```

This keeps responsiveness within a controlled loop and avoids busy-spin.

---

## Symptom 3: Self-Deadlock Under High-Frequency Operations

After introducing `processEvents`, re-entrant calls could happen within the same thread during the waiting window, causing lock contention on the same request path.

## Cause 3: Lock Type Mismatch With Re-entrancy

`std::mutex` does not allow repeated acquisition by the same thread.

## Fix 3: Use Recursive Mutex for Request Serialization

```cpp
std::recursive_mutex requestMutex_;
```

---

## Takeaways

1. Fix I/O object thread ownership early and keep it consistent through lifecycle.
2. Avoid long blocking waits in I/O threads; if waiting is required, use an event-aware strategy.
3. Whenever event yielding is introduced, re-check re-entrancy paths and lock model together.

This fix not only removed shutdown stalls, but also improved timeout consistency in Modbus communication. For industrial software, deterministic shutdown and explainable timing are as important as functional completeness.
