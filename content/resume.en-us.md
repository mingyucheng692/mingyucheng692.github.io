---
title: "Resume"
layout: "page"
summary: "YUCHENG MING (Intent) - Embedded Software Engineer / PC Software (ESS/EMS)"
---

# YUCHENG MING (Intent)

> **Embedded Software Engineer | 3 Years Experience | Focus on IIoT & ESS**

- **Email**: [Hidden for Privacy]
- **GitHub**: [mingyucheng692](https://github.com/mingyucheng692)
- **Location**: Zhongshan, Guangdong (Open to Shenzhen)

---

## 🛠 Skills

- **Languages**: **C/C++** (Qt6, STL, C++11/14/17), **Golang**, Python, Shell
- **Protocols**: **Modbus (RTU/TCP)**, **MQTT**, CAN, IEC-104, IEC-61850
- **Database/Middleware**: SQLite, PostgreSQL (TimescaleDB), Redis, Redpanda (Kafka compatible)
- **Tools**: CMake, Docker/Podman, GDB, Git, STM32, RTOS, Windows IOCP

---

## 💼 Work Experience

### **Honghui Energy (Southern HQ) | Guangdong Ruilai Huakong Technology Co., Ltd.**
**Software Engineer** | 2025.06 - Present | Zhongshan, Guangdong

> **Company Background**: Beijing Honghui International Energy Technology Development Co., Ltd. is a leading enterprise in the flywheel energy storage industry. I am a core member of the Southern R&D Center team.

- **Responsibilities**: Responsible for the 0-to-1 software selection and core business development of the flywheel energy storage system (covering FMS/PCS, Edge Gateway, Cloud Platform). Bridged the data link between the underlying device side and the cloud platform side, and established the team's code standards and modular build system from scratch.
- **Team Efficiency**: Developed and maintained internal general-purpose communication debugging tools, precipitated reusable components, effectively reducing the costs of cross-department collaboration and on-site equipment commissioning.

---

## 💻 Project Experience

### **Flywheel Energy Storage Monitoring System (FMS)**
**Role**: Core Developer (C++/Qt6) | **Period**: Honghui Energy

- **Communication Optimization**: Introduced Windows **IOCP** asynchronous network model and connection pooling to refactor Modbus-TCP communication. Reduced CPU usage by 15% under high-frequency throughput scenarios, completely solving UI freezing issues.
- **Data Governance**: Designed a sliding window algorithm to filter jitter signals, achieving 100% effective alarm extraction. Utilized **SQLite WAL** mode to optimize full message logging, supporting second-level historical data queries.
- **Engineering Efficiency**: Led **CMake** modularization (precompiled headers, parallel builds), compressing full build time from >5 minutes to <50 seconds (600% efficiency increase).
- **Stability**: Built C++ exception capture and automatic Dump generation mechanisms, combined with user authentication, significantly shortening fault localization cycles and enhancing system security.

### **Flywheel Energy Storage Smart Cloud Platform**
**Role**: Backend Developer (Golang/Microservices) | **Period**: Honghui Energy

- **Architecture Design**: Adopted **Docker/Podman** containerization to achieve one-click service deployment and migration.
- **High Concurrency**: Introduced **Redpanda (Kafka compatible)** message queues for peak shaving, decoupling the acquisition layer from the storage layer to ensure zero data loss under high concurrency.
- **Security**: Designed device authentication middleware and user identity authentication systems to ensure industrial data security.

---

## 🚀 Open Source Projects

### **[Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)**
**Individual Developer** | 2025.12 - Present
- **Intro**: An efficient industrial protocol debugging assistant designed to solve field commissioning pain points.
- **Tech**: C++17, Qt6, Protocol Parser Design.
- **Core Features**:
  - Independently designed Modbus protocol parsing core, supporting RTU/TCP message assembly, validation, and parsing.
  - Supports TCP Client and UART debugging modes.
  - *Highlight*: Clear code structure, high modularity, easy for secondary development.

---

## 🎓 Education

- **Chongqing College of International Business and Economics** |  B.Eng in IoT Engineering

