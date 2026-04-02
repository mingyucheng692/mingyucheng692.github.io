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
▸ 所在公司为泓慧能源（飞轮储能头部企业）全资子公司，承担南方研发与量产基地核心系统开发

- 参与南方基地储能软件体系（端-边-云）从 0 到 1 建设，负责衔接集团技术规范，交付的软件模块直接服务于本地产线设备的整机测试与现场出厂交付。
- 深度对接硬件与电气规约团队，解决复杂工业现场的通信拥塞与信号抖动等痛点，保障底层电力电子设备与上层系统的可靠交互。
- 协助推行团队 Git 协同与代码审查机制；针对跨部门设备联调耗时长的痛点，主导开发多套通用联调工具链，以可视化解析取代手工查表与拼接，显著缩短现场排障周期。

## 代表项目

### 飞轮储能监控系统上位机（FMS）
核心开发 ｜ C++ / Qt6 / Modbus-TCP / CAN / SQLite / IOCP ｜ 2025.06 - 至今

- 重构 Modbus-TCP 通信链路，引入 Windows IOCP 与连接池机制，解决高频数据采集导致的界面阻塞问题，CPU 占用率降低约 15%。
- 开发底层 DSP 控制板专属调试模块（基于 ZLG CAN SDK），实现 IEEE 754 浮点数与 HEX 报文的双向动态解析，支持多字节序（CDAB/ABCD）自动转换与寄存器语义映射，替代 CANTest 手动抓包换算流程，显著提升软硬件联调效率。
- 设计滑动窗口算法实现报警信号防抖；基于 SQLite WAL 模式实现高频报文落盘，支持现场历史数据的高效本地检索与追溯。
- 推进 CMake 模块化构建，并集成 Crash Dump 异常捕获机制，将全量编译耗时从 5 分钟压缩至 50 秒内，显著提升日常开发迭代与现场排障效率。

### 飞轮储能边缘通信网关
核心开发 ｜ C / STM32F407 / FreeRTOS / MQTT / Modbus / IEC-104 ｜ 2025.12 - 至今

- 负责基于 STM32F407 与 FreeRTOS 的边缘网关核心业务逻辑，向下通过 Modbus/RS485 轮询底层设备，向上通过 MQTT 建立与云端的时序数据通道。
- 实现 Modbus、IEC-104 与 MQTT 之间的协议解析与映射转换，打通设备端、电力规约侧与平台侧的数据交互闭环。
- 针对现场弱网工况，设计并实现时序数据缓存与断点续传机制，保障网络波动下的数据完整性。

### 飞轮储能智慧云平台
后端开发 ｜ Golang / Docker / Podman / Redpanda / PostgreSQL / Redis ｜ 2025.06 - 至今

- 参与储能云平台（基于 Alibaba Cloud Linux）核心后端服务开发，承接设备数据的高频接入、协议解析与时序入库。
- 主导部署环境配置，采用 Docker 兼顾本地开发，生产环境应用 Podman rootless 结合 Shell 脚本，实现Rootless账号的安全隔离部署。
- 引入 Redpanda 消息队列解耦数据采集与存储层，平抑并发写入峰值；独立设计认证中间件，基于 JWT (AT/RT) 与 Redis 维护安全的会话状态。

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)（个人开源项目）
独立开发者 ｜ C++20 / Qt6 / CMake / CI-CD ｜ 2025.12 - 至今

- 采用 channel / transport / session / parser 分层架构研发跨平台调试工具，内置可视化报文构建器，彻底免除繁琐的手动查表与十六进制帧拼接。
- 开发 Frame Analyzer 核心解析器，支持协议自动识别、自定义倍率换算与寄存器语义标注；支持 JSON / CSV 配置导入导出，将现场排障耗时从分钟级压缩至秒级，联调提效 5 倍以上。
- 基于 GitHub Actions 跑通全自动 CI/CD 流水线，实现代码推送后的自动构建与 Release 发布；客户端内建多语言切换与自动更新（Auto-Updater）机制，保障工具在现场的敏捷迭代。

## 教育背景

重庆对外经贸学院 ｜ 物联网工程 ｜ 本科 / 工学学士

</div>
