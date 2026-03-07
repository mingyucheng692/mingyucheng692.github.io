---
title: "踩坑日记：Modbus 多线程死锁修复记录"
date: 2026-03-07T10:00:00+08:00
draft: false
tags: ["Modbus", "Qt6", "C++", "Deadlock", "Multithreading", "Industrial Software"]
categories: ["Tech", "Engineering"]
summary: "记录一次 Qt 多线程 Modbus 场景中的退出死锁与超时误判问题，以及对应的工程化修复方案。"
url: "/zh-cn/blog/tech/modbus-deadlock-fix-notes/"
---

在 `Modbus-Tools` 的迭代过程中，遇到过一次典型的并发问题：窗口已关闭，但进程未退出，调试会话也无法正常结束。后续修复中，又暴露出通信超时误判与重入竞争问题。本文记录排查过程与最终方案。

---

## 现象一：退出阶段卡住

- 主线程销毁 `MainWindow` 时会等待工作线程退出
- 工作线程可能仍阻塞于串口或网络读写流程
- UI 已消失，但进程保持存活

## 原因一：线程依附关系不一致

`QSerialPort` 与 `QTcpSocket` 的事件处理依赖其所属线程事件循环。若对象创建于主线程，但调用路径在工作线程，退出阶段容易出现互相等待。

## 处理一：明确对象归属线程

在通道创建后显式迁移至工作线程：

```cpp
stack.thread = std::make_shared<QThread>();
stack.channel->moveToThread(stack.thread.get());
```

同时保证对象创建时不绑定 `parent`，避免迁移失败。

---

## 现象二：连接后偶发 Timeout

- 监控抓包可见设备有响应
- 业务层仍返回超时

## 原因二：阻塞等待饿死事件循环

请求发送后使用 `condition_variable::wait_until` 阻塞等待，导致承载 IO 的线程无法及时处理 `readyRead` 事件，回包回调被延后。

## 处理二：改为可让渡事件的等待循环

```cpp
while (true) {
    QCoreApplication::processEvents(QEventLoop::AllEvents);
    if (signalReceived) break;
    if (timeout) return Error;
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
}
```

该方案在可控范围内维持响应性，同时避免忙等占满 CPU。

---

## 现象三：高频操作下偶发自锁

引入 `processEvents` 后，等待窗口期内可能触发同线程重入，导致同一把互斥锁被重复申请。

## 原因三：锁模型与重入路径不匹配

原有 `std::mutex` 无法支持同线程重复进入临界区。

## 处理三：改用递归锁保护请求序列

```cpp
std::recursive_mutex requestMutex_;
```

---

## 复盘结论

1. IO 对象所属线程必须在设计阶段固定并贯穿生命周期管理。
2. IO 线程中应避免长时间阻塞等待，必要时采用可处理中断事件的等待策略。
3. 引入事件让渡后，要同步评估重入路径与锁策略。

这次修复不仅解决了退出挂起，也提升了 Modbus 通信超时判定的一致性。对工业软件而言，稳定退出和时序可解释性与功能本身同等重要。
