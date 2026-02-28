---
title: "Modbus-Tools 深度解析"
date: 2024-05-20
tags: ["Modbus", "Qt6", "C++", "Open Source"]
categories: ["Tech", "Project"]
summary: "一個專為嵌入式開發人員設計的輕量級 Modbus 調試工具。支持 RTU/TCP、串口/網絡調試及報文解析。"
url: "/zh-tw/blog/tech/modbus-tools-intro/"
---

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)
**C++17 | Qt6 | Industrial Protocols**

一個專為嵌入式開發人員設計的輕量級 Modbus 調試工具。

- **簡介**: 一個高效的工業協議調試助手，旨在解決現場聯調痛點。
- **特性**:
  - 支持 **Modbus RTU** over Serial (UART)
  - 支持 **Modbus TCP** 
  - 支持 **TCP Client**
  - 支持 **Serial Port** (UART)
  - 實時報文解析與校驗 (CRC/LRC)
  - 平台支持 Windows 10/11
- **架構**: 採用 Qt6 信號槽機制實現 UI 與 協議邏輯解耦，底層通信模塊封裝了 QSerialPort 與 QTcpSocket。
