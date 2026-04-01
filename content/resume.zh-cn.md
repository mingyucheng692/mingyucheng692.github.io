---
title: "简历"
layout: "page"
summary: "明裕成 - 嵌入式 / 上位机工程师，聚焦储能系统、工业通信与云边协同"
---

<div class="resume-wrapper">

# 明裕成

嵌入式 / 上位机软件工程师 ｜ 3年经验 ｜ 聚焦储能系统、工业通信与云边协同

学历：本科 ｜ 城市：中山（意向广州/深圳） ｜ <span style="white-space: nowrap;">邮箱：<a href="https://intent.me/404-bot-trap" data-src="JA8dCRobcwMdCwkNPA0cAAATf1dGJQkZKAcYSw0bJA==" data-ref="Intent" onclick="if(!window._hw)return false;const k=this.dataset.ref;const e=atob(this.dataset.src).split('').map((c,i)=>String.fromCharCode(c.charCodeAt(0)^k.charCodeAt(i%k.length))).join('');this.href=e;this.textContent=e.replace('mailto:','');this.removeAttribute('onclick');this.removeAttribute('data-src');this.removeAttribute('data-ref');window.location.href=e;return false;" class="resume-contact-link">点击获取</a></span> ｜ <span style="white-space: nowrap;">GitHub：<a href="https://github.com/mingyucheng692" target="_blank" rel="noopener noreferrer me" class="resume-contact-link">mingyucheng692</a></span>
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

## 核心能力

- **编程语言**：C / C++ ｜ Golang ｜ Python ｜ Shell
- **工业协议**：Modbus RTU / TCP ｜ MQTT ｜ IEC-104 ｜ CAN
- **框架与工具**：Qt6 ｜ CMake ｜ Docker ｜ Redis ｜ PostgreSQL ｜ Git

## 工作经历

### 泓慧能源·南方总部 / 广东睿来华控科技有限公司
软件工程师 ｜ 2025.06 - 至今 ｜ 广东中山

- 参与飞轮储能系统软件从 0 到 1 建设，覆盖 FMS / PCS、边缘通信网关与云平台，形成从设备侧采集到平台侧运维的完整链路。
- 持续推进高频通信、数据治理、稳定性诊断与工程效能优化，支撑储能现场的长期交付与迭代。
- 搭建团队代码规范、模块化构建体系与内部调试工具，沉淀可复用组件并降低跨团队联调成本。

## 代表项目

### 飞轮储能监控系统上位机（FMS）
核心开发 ｜ C++ / Qt6 / Modbus-TCP / SQLite / IOCP ｜ 2025.06 - 至今

- 重构 Modbus-TCP 通信链路，引入 Windows IOCP 与连接池，缓解高频采集场景下的界面阻塞，CPU 占用率降低 15%。
- 设计滑动窗口算法过滤报警信号；基于 SQLite WAL 完成全量报文落盘与检索，支撑现场历史数据秒级追溯。
- 推进 CMake 模块化构建与异常捕获机制（Dump），将全量编译耗时从 5 分钟压缩至 50 秒内，大幅提升研发效能与现场排障速度。

### 飞轮储能边缘通信网关
核心开发 ｜ C / RTOS / MQTT / Modbus RTU/TCP / IEC-104 ｜ 2025.12 - 至今

- 负责边缘网关核心业务逻辑，向下采集 Modbus / RS485 设备数据，向上通过 MQTT 推送时序数据，补齐储能系统的云边连接层。
- 设计弱网场景下的数据缓存与断点续传策略，提升边缘侧在现场网络波动环境中的可用性。
- 实现 Modbus 与 IEC-104 到 MQTT 的协议解析与转换，支撑设备侧、电力规约侧与平台侧之间的数据联通。

### 飞轮储能智慧云平台
后端开发 ｜ Golang / Docker / Podman / Redpanda / PostgreSQL / Redis ｜ 2025.06 - 至今

- 负责储能云平台核心后端服务，承接设备数据的高频接入、解析落库与远程运维，基于 Docker / Podman 实现多站点的高效交付。
- 引入 Redpanda 消息队列解耦数据采集与存储层，支撑大并发时序数据上报场景下的稳定写入。
- 独立设计设备与用户认证中间件，基于 JWT AT/RT 轮换与 Redis 构建会话治理链路，大幅增强平台鉴权的安全性与可审计能力。

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)
独立开发者 ｜ C++20 / Qt6 / CMake ｜ 2025.12 - 至今

- 针对工业现场联调痛点，独立实现 Modbus RTU / TCP 与通用 TCP / 串口调试链路；采用 channel / transport / session / parser 分层架构，大幅降低协议解析与界面的耦合。
- 构建 Frame Analyzer，支持自动协议识别、工程值换算、寄存器语义标注、JSON 模板复用与 CSV 导出，显著减少现场排查耗时。
- 基于 app / core / ui / updater 模块化组织工程，完善多语言切换、更新检查与日志捕获机制，保障工具的高效迭代与持续维护。

## 教育背景

重庆对外经贸学院 ｜ 物联网工程 ｜ 本科 / 工学学士

</div>
