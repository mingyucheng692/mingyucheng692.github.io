---
title: "Modbus-Tools 深度解析"
date: 2024-05-20
tags: ["Modbus", "Qt6", "C++", "Open Source"]
categories: ["Tech", "Project"]
summary: "一个专为嵌入式开发人员设计的轻量级 Modbus 调试工具。支持 RTU/TCP、串口/网络调试及报文解析。"
url: "/zh-cn/blog/tech/modbus-tools-intro/"
---

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)
**C++17 | Qt6 | Industrial Protocols**

一个专为嵌入式开发人员设计的轻量级 Modbus 调试工具。

- **简介**: 一个高效的工业协议调试助手，旨在解决现场联调痛点。
- **特性**:
  - 支持 **Modbus RTU** over Serial (UART)
  - 支持 **Modbus TCP** 
  - 支持 **TCP Client**
  - 支持 **Serial Port** (UART)
  - 实时报文解析与校验 (CRC/LRC)
  - 平台支持 Windows 10/11
- **架构**: 采用 Qt6 信号槽机制实现 UI 与 协议逻辑解耦，底层通信模块封装了 QSerialPort 与 QTcpSocket。
