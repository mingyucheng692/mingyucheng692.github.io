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
▸ A wholly-owned subsidiary of Honghui Energy (leading flywheel energy storage enterprise), responsible for core system development at the Southern R&D and production base

- Participated in the 0-to-1 construction of the Southern base's energy storage software ecosystem (edge-side-cloud), aligning with group technical standards; delivered software modules directly serving production line equipment testing and on-site delivery.
- Collaborated closely with hardware and electrical protocol teams to resolve communication congestion and signal jitter issues in complex industrial environments, ensuring reliable interaction between power electronics devices and upper-layer systems.
- Facilitated team Git collaboration and code review practices; developed multiple universal debugging toolchains to replace manual lookups with visual parsing, significantly reducing on-site troubleshooting cycles.

## Key Projects

### Flywheel Energy Storage Monitoring System (FMS)
Core Developer ｜ C++ / Qt6 / Modbus-TCP / CAN / SQLite / IOCP ｜ 2025.06 - Present

- Refactored Modbus-TCP communication link with Windows IOCP and connection pooling, resolving UI blocking caused by high-frequency data acquisition, reducing CPU usage by ~15%.
- Developed dedicated debugging module for DSP control boards (based on ZLG CAN SDK), implementing bidirectional dynamic parsing of IEEE 754 floating-point numbers and HEX frames, supporting multi-byte order (CDAB/ABCD) auto-conversion and register semantic mapping, replacing manual CANTest workflows and significantly improving hardware-software integration efficiency.
- Designed sliding window algorithm for alarm signal debouncing; implemented high-frequency message persistence based on SQLite WAL mode, enabling efficient local retrieval and tracing of historical on-site data.
- Promoted CMake modular builds and integrated Crash Dump exception capture mechanism, compressing full compilation time from 5 minutes to under 50 seconds, significantly enhancing development iteration and on-site troubleshooting efficiency.

### Flywheel Energy Storage Edge Communication Gateway
Core Developer ｜ C / STM32F407 / FreeRTOS / MQTT / Modbus / IEC-104 ｜ 2025.12 - Present

- Responsible for edge gateway core business logic based on STM32F407 and FreeRTOS, polling underlying devices via Modbus/RS485 downwards and establishing time-series data channels to the cloud via MQTT upwards.
- Implemented protocol parsing and mapping conversion among Modbus, IEC-104, and MQTT, enabling a closed-loop data interaction across device-side, power protocol-side, and platform-side.
- Designed and implemented time-series data caching and breakpoint resume mechanism for weak network conditions, ensuring data integrity during network fluctuations.

### Flywheel Energy Storage Smart Cloud Platform
Backend Developer ｜ Golang / Docker / Podman / Redpanda / PostgreSQL / Redis ｜ 2025.06 - Present

- Participated in core backend service development for the energy storage cloud platform (based on Alibaba Cloud Linux), handling high-frequency device data ingestion, protocol parsing, and time-series storage.
- Led deployment environment configuration using Docker for local development and Podman rootless combined with Shell scripts for production, achieving secure isolated deployment with rootless accounts.
- Introduced Redpanda message queue to decouple data acquisition and storage layers, smoothing concurrent write peaks; independently designed authentication middleware maintaining secure session state via JWT (AT/RT) and Redis.

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools) (Personal Open Source Project)
Independent Developer ｜ C++20 / Qt6 / CMake / CI-CD ｜ 2025.12 - Present

- Developed cross-platform debugging tool using channel/transport/session/parser layered architecture with built-in visual frame builder, eliminating tedious manual table lookups and hexadecimal frame concatenation.
- Built Frame Analyzer core parser supporting automatic protocol recognition, custom scaling conversion, and register semantic annotation; supports JSON/CSV configuration import/export, compressing on-site troubleshooting time from minutes to seconds, improving integration efficiency by 5x+.
- Established fully automated CI/CD pipeline via GitHub Actions for automatic build and release upon code push; client features built-in multi-language switching and Auto-Updater mechanism, ensuring agile iteration in field deployments.

## Education

Chongqing College of Foreign Trade and Business ｜ Internet of Things Engineering ｜ Bachelor of Engineering

</div>
