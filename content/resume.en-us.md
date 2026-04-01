---
title: "Resume"
layout: "page"
summary: "YUCHENG MING - Embedded / PC Software Engineer, focusing on Energy Storage Systems, Industrial Communication, and Cloud-Edge Synergy"
---

<div class="resume-wrapper">

# YUCHENG MING

Embedded / PC Software Engineer ｜ 3 Years Experience ｜ Energy Storage, Industrial Comm & Cloud-Edge Synergy

Education: Bachelor ｜ Location: Zhongshan (Intent: Guangzhou/Shenzhen) ｜ <span style="white-space: nowrap;">Email: <a href="https://intent.me/404-bot-trap" data-src="JA8dCRobcwMdCwkNPA0cAAATf1dGJQkZKAcYSw0bJA==" data-ref="Intent" onclick="if(!window._hw)return false;const k=this.dataset.ref;const e=atob(this.dataset.src).split('').map((c,i)=>String.fromCharCode(c.charCodeAt(0)^k.charCodeAt(i%k.length))).join('');this.href=e;this.textContent=e.replace('mailto:','');this.removeAttribute('onclick');this.removeAttribute('data-src');this.removeAttribute('data-ref');window.location.href=e;return false;" class="resume-contact-link">Click to View</a></span> ｜ <span style="white-space: nowrap;">GitHub: <a href="https://github.com/mingyucheng692" target="_blank" rel="noopener noreferrer me" class="resume-contact-link">mingyucheng692</a></span>
<script>
  (function() {
      if (typeof window._hw !== 'undefined') return;
      window._hw = false;
      const onUserAction = () => { window._hw = true; };
      ['mousemove', 'touchstart', 'scroll', 'keydown'].forEach(evt => 
          window.addEventListener(evt, onUserAction, {once: true, passive: true})
      );
  })();
</script>

## Core Competencies

- **Languages**: C / C++ ｜ Golang ｜ Python ｜ Shell
- **Protocols**: Modbus RTU / TCP ｜ MQTT ｜ IEC-104 ｜ CAN
- **Frameworks & Tools**: Qt6 ｜ CMake ｜ Docker ｜ Redis ｜ PostgreSQL ｜ Git

## Work Experience

### Honghui Energy (South HQ) / Guangdong Ruilai Huakong Technology Co., Ltd.
Software Engineer ｜ 2025.06 - Present ｜ Zhongshan, Guangdong

- Participated in the 0-to-1 construction of Flywheel Energy Storage System software, covering FMS/PCS, edge communication gateways, and cloud platforms, forming a complete loop from device-side acquisition to platform-side O&M.
- Continuously promoted high-frequency communication, data governance, stability diagnosis, and engineering efficiency optimization to support long-term delivery and iteration at energy storage sites.
- Built team code specifications, modular build systems, and internal debugging tools, accumulating reusable components and reducing cross-team joint debugging costs.

## Key Projects

### Flywheel Energy Storage Monitoring System (FMS)
Core Developer ｜ C++ / Qt6 / Modbus-TCP / SQLite / IOCP ｜ 2025.06 - Present

- Refactored the Modbus-TCP communication link, introducing Windows IOCP and connection pools to alleviate UI blocking in high-frequency acquisition scenarios, reducing CPU usage by 15%.
- Designed a sliding window algorithm to filter alarm signals; achieved full message storage and retrieval based on SQLite WAL, supporting second-level historical data tracing on-site.
- Promoted CMake modular builds and exception capture mechanisms (Dump), compressing full compilation time from 5 minutes to under 50 seconds, significantly improving R&D efficiency and on-site troubleshooting speed.

### Flywheel Energy Storage Edge Communication Gateway
Core Developer ｜ C / RTOS / MQTT / Modbus RTU/TCP / IEC-104 ｜ 2025.12 - Present

- Responsible for the core business logic of the edge gateway, collecting Modbus/RS485 device data downwards and pushing time-series data via MQTT upwards, completing the cloud-edge connection layer of the energy storage system.
- Designed data caching and breakpoint resume strategies in weak network scenarios, improving availability in fluctuating network environments on-site.
- Implemented protocol parsing and conversion from Modbus and IEC-104 to MQTT, supporting data interconnection among devices, power protocols, and the cloud platform.

### Flywheel Energy Storage Smart Cloud Platform
Backend Developer ｜ Golang / Docker / Podman / Redpanda / PostgreSQL / Redis ｜ 2025.06 - Present

- Responsible for core backend services of the energy storage cloud platform, handling high-frequency device data access, parsing/storage, and remote O&M, achieving efficient multi-site delivery via Docker/Podman.
- Introduced Redpanda message queue to decouple the data acquisition and storage layers, supporting stable writing in high-concurrency time-series data reporting scenarios.
- Independently designed device and user authentication middleware, building a session governance link based on JWT AT/RT rotation and Redis, significantly enhancing platform authentication security and auditability.

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)
Independent Developer ｜ C++20 / Qt6 / CMake ｜ 2025.12 - Present

- Addressed pain points in industrial field joint debugging by independently implementing Modbus RTU/TCP and general TCP/Serial debugging links; adopted a channel/transport/session/parser layered architecture, drastically reducing coupling between protocol parsing and UI.
- Built Frame Analyzer, supporting automatic protocol recognition, engineering value conversion, register semantic annotation, JSON template reuse, and CSV export, significantly reducing on-site troubleshooting time.
- Modularized the project based on app/core/ui/updater, perfecting multi-language switching, update checking, and log capture mechanisms to ensure efficient iteration and continuous maintenance of the tool.

## Education

Chongqing College of Foreign Trade and Business ｜ Internet of Things Engineering ｜ Bachelor of Engineering

</div>
