---
title: "TrafficMonitor CPU 占用优化复盘：从 4% 降到 0.6% 的工程化收缩"
date: 2026-04-26T10:00:00+08:00
draft: false
tags: ["TrafficMonitor", "Qt", "C++", "Modbus", "Performance", "CPU", "Postmortem"]
categories: ["Tech", "Engineering"]
summary: "记录一次高频轮询场景下的 CPU 优化复盘：通过收缩 UI 日志链路、删除高频状态机、改造线程等待模型，将进程 CPU 使用率从 3.6%~4.2% 降到 0.1%~0.6%。"
url: "/zh-cn/blog/tech/trafficmonitor-cpu-optimization-postmortem/"
---

在一次 `100ms` 轮询场景的性能排查中，`TrafficMonitor` 所在进程的 CPU 使用率长期稳定在 `3.6%~4.2%`。除资源占用偏高外，界面在高频收发期间也会出现可感知的刷新迟滞和滚动不跟手。对比两个版本实现后，优化版本最终将该指标降到 `0.1%~0.6%`，同时高频轮询下的 UI 卡顿现象明显缓解。

这次收益并非来自单点热点函数优化，而是一次典型的工程化收缩：减少高频路径上的中间层、状态机和无效唤醒，让系统回到更符合其职责边界的实现方式。

本文复盘的优化对象来自开源工具 [Modbus-Tools](/zh-cn/blog/tech/modbus-tools-intro/)。如果需要先了解工具的整体定位与核心能力，可先阅读该介绍文章。

---

## 背景与结论

这次优化的结论可以概括为两点：

- `TrafficMonitor` 回归轻量流量展示组件，不再承担复杂日志系统职责
- 工作线程在无事件时真正阻塞，而不是以固定时间片持续轮询

对应到代码层，主要改动集中在四个方向：

- `TrafficMonitorWidget` 从事件模型回退到直接文本追加
- `ModbusTcpView` 与 `ModbusRtuView` 删除轮询汇总和抑制逻辑
- `ModbusClient` 从 `wait_for + 5ms slice` 改为 `wait_until`
- `SerialChannel` 与 `TcpChannel` 移除高频路径上的线程诊断日志

---

## 关键改动

### 1. UI 日志链路收缩

复杂版本中，`TrafficMonitor` 引入了完整事件抽象，日志写入路径为：

`调用方 -> 构造事件对象 -> 组件判定级别/模式 -> 渲染文本 -> 写入列表`

优化版本删除这套中间层，保留 `appendTx()`、`appendRx()`、`appendInfo()`、`appendWarning()` 等直接入口，路径收缩为：

`调用方 -> 直接格式化文本 -> 追加到 QListWidget`

这一步减少了对象构造、事件分类、二次渲染和状态同步开销，是本次 CPU 下降的主要来源之一。

### 2. 组件实现从 Model/View 回退到直接列表

复杂版本的 `TrafficMonitorWidget` 不只是显示控件，还维护了完整的数据层，包括：

- `QListView + QAbstractListModel`
- `pendingEvents_`
- `eventHistory_`
- `flushTimer_`
- `rebuildScheduled_`
- `pausedEventCount_`

这类设计在功能上更完整，但在 `100ms` 高频刷新下，会持续放大以下成本：

- 事件入队与批量刷新
- 文本格式化与过滤判断
- 历史重建与滚动同步

优化版本改回 `QListWidget` 直接追加，本质上是删除一层常驻的数据管理系统。对高频日志场景，这种收缩带来的收益非常直接。

### 3. 删除高频附加能力与轮询状态机

复杂版本围绕“更强可观察性”增加了多项能力，包括：

- Pause View
- Raw Frames
- Level Filter
- 暂停计数
- 可见列表重建
- Poll Summary 汇总状态机

这些能力单独看都合理，但在高频场景下，会把每条日志都变成“需要判断和调度的事件”，而不是“直接展示的文本”。

尤其是 `ModbusTcpView` 与 `ModbusRtuView` 中的 Poll Summary 逻辑，虽然减少了可见日志条数，但并未减少系统总工作量。每次轮询仍需更新计数、维护状态、判断窗口并格式化摘要。优化版本删除这套逻辑后，链路明显缩短。

### 4. 等待模型改为真正阻塞

除 UI 外，另一处关键改动在 `ModbusClient` 的等待策略。

复杂版本使用 `cv_.wait_for(lock, 5ms, ...)` 配合事件泵机制，意味着线程即使没有真实事件，也会以 `5ms` 周期被唤醒一次。对 `100ms` 轮询场景，这会制造大量无效 wakeup。

优化版本统一改为：

`cv_.wait_until(lock, deadline, predicate)`

改造后的效果是：

- 无事件时线程真正休眠
- 仅在超时或条件满足时唤醒
- 不再周期性处理当前线程事件

这部分改动直接降低了工作线程空转，是本次优化的另一主要来源。

### 5. 清理高频调试输出

`SerialChannel.cpp` 和 `TcpChannel.cpp` 中，复杂版本保留了线程上下文诊断逻辑，例如：

- `threadToken(...)`
- `logThreadContextOnce(...)`
- `open()`、`onReadyRead()`、`onConnected()` 中的线程日志

单条日志成本不高，但它们位于高频收发路径，累计后会放大格式化和分支判断开销。优化版本将其移除，是一次典型的高频路径减负。

---

## 收益为什么明显

这次优化的价值，不在于把某个函数从 `10ms` 优化到 `2ms`，而在于删除了高频链路上的多项乘法项。

复杂版本的一次轮询收发，通常会经历：

- IO 回调
- 事件对象构造
- 级别与模式判断
- 队列操作
- 定时器唤醒
- 批量渲染
- 过滤判断
- 历史重建
- 自动滚动

优化版本则更接近：

- IO 回调
- 文本格式化
- 直接追加到列表

当调用频率固定为 `100ms` 时，链路长度差异会持续放大，最终直接反映为 CPU 使用率从 `3.6%~4.2%` 降到 `0.1%~0.6%`。由于主线程不再持续承受事件分发、批量刷新和列表重建压力，界面交互也恢复到更稳定的状态。

---

## 工程经验

这次复盘沉淀出几条比较明确的工程经验：

- 高频场景下，功能复杂度本身就是性能成本
- GUI 性能问题很多时候不在绘制，而在绘制前的事件系统和状态机
- 对等待模型来说，能阻塞就不要轮询，能按条件唤醒就不要按时间片唤醒
- 日志组件职责越清晰，实现越容易稳定，性能也越可控

从结果看，这次优化并不是“做了很多技巧性调优”，而是重新收缩了系统边界。当 `TrafficMonitor` 回归轻量视图、`ModbusClient` 回到真正阻塞等待后，系统不再为低频附加能力支付高频成本，整体负载自然回落。

如需进一步了解工具能力全貌，可参考 [Modbus-Tools 介绍文章](/zh-cn/blog/tech/modbus-tools-intro/)。项目源码见 GitHub 仓库：[mingyucheng692/Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)。
