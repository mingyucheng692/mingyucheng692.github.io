---
title: "Modbus-Tools Deep Dive"
date: 2024-05-20
tags: ["Modbus", "Qt6", "C++", "Open Source"]
categories: ["Tech", "Project"]
summary: "A lightweight Modbus debugging tool designed for embedded developers. Supports RTU/TCP, Serial/Network debugging and packet parsing."
url: "/en-us/blog/tech/modbus-tools-intro/"
---

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)
**C++17 | Qt6 | Industrial Protocols**

A lightweight Modbus debugging tool designed for embedded developers.

- **Introduction**: An efficient industrial protocol debugging assistant aiming to solve field commissioning pain points.
- **Features**:
  - Supports **Modbus RTU** over Serial (UART)
  - Supports **Modbus TCP**
  - Supports **TCP Client**
  - Supports **Serial Port** (UART)
  - Real-time packet parsing and verification (CRC/LRC)
  - Platform support: Windows 10/11
- **Architecture**: Uses Qt6 Signal & Slot mechanism to decouple UI from protocol logic; underlying communication modules encapsulate QSerialPort and QTcpSocket.
