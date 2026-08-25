---
title: "Windows Docker 端口幽灵映射排查：EMQX 1883 被 WinNAT 静默保留的复盘"
date: 2026-08-25T12:00:00+08:00
draft: false
tags: ["Docker", "Windows", "WinNAT", "Hyper-V", "EMQX", "MQTT", "Postmortem"]
categories: ["backend-infra"]
summary: "Windows 开发机 Docker Desktop 完整服务栈（backend + EMQX 5.8.9 + TimescaleDB + Valkey），后端 ping/pong 正常但 mosquitto_pub 持续 Connection refused。docker ps 中同栈其他端口正常映射，唯独 1883 缺 0.0.0.0:1883-> 前缀且不报错，根因为 Hyper-V/WinNAT 动态端口保留圈占 1883（落在 1802–1901 区间）。给出换端口/临时夺回/白名单/修正动态端口范围四级修复方案。"
url: "/zh-cn/blog/tech/docker-windows-port-ghost-mapping-postmortem/"
---

> 环境与命令均已脱敏：容器名、账号、密码、Token、Topic、payload 抽象为占位符。

环境：Docker Desktop（Windows）完整服务栈（backend + EMQX 5.8.9 + TimescaleDB + Valkey），链路为 WSL2 `mosquitto_pub` → 宿主机 `127.0.0.1:1883` → EMQX 容器。

## 现象

```bash
mosquitto_pub -d -h 127.0.0.1 -p 1883 -i "<Client_ID>" -u "<User>" -P "<Password>" -t "<Topic>" -m '<Test_Payload_JSON>'
# Error: Connection refused
```

容器内 1883 正常监听，宿主机映射缺失，Docker 全程不报错。

## 关键证据

完整服务栈同一次 `docker compose up` 起来，后端应用层正常响应：

```bash
curl 127.0.0.1:8080/ping
# {"msg":"pong"}
```

但 `docker ps` 中唯独 EMQX 的 1883 缺前缀：

```text
NAMES            PORTS
<svc>-backend    0.0.0.0:8080->8080/tcp
<svc>-emqx       0.0.0.0:8083->8083/tcp, 1883/tcp, 0.0.0.0:18083->18083/tcp
<svc>-db         0.0.0.0:5432->5432/tcp
<svc>-valkey      0.0.0.0:6379->6379/tcp
```

同一台机器、同一份 compose、同一次 up：backend（8080）、db（5432）、valkey（6379）、emqx 的 8083/18083 全部正常映射，唯独 1883 无 `0.0.0.0:1883->` 前缀（仅容器内监听）。问题被锁定到 1883 这个端口本身，而非 Docker 或 compose。

## 假设与验证

| # | 假设 | 验证命令 | 结果 | 结论 |
|---|---|---|---|---|
| A | 配置未重新加载 | `docker compose up -d --force-recreate <svc>-emqx` | 1883 仍无前缀 | 排除 |
| B | YAML 隐藏字符致解析失败 | `docker compose config` | 输出含 `published: "1883"` | 排除 |
| C | 宿主机端口被进程占用 | `netstat -ano \| findstr :1883` | 无输出 | 排除 |
| D | Windows 动态端口保留 | `netsh int ipv4 show excludedportrange protocol=tcp` | `1802  1901` 含 1883 | **确认根因** |

A–C 全部排除后，问题收敛到“Docker 读到了配置，但绑定宿主机失败且不报错”。D 命中。

## 根因

Windows Hyper-V / WinNAT 的动态端口起始范围被压到 1024 附近，随机保留大段端口给系统使用。1883 落在保留区间 `1802–1901` 内，Docker 绑定时被 Windows 底层静默拦截——不抛 `bind: address already in use`，直接丢弃该端口映射，形成“容器内监听正常、宿主机无映射、无报错”的幽灵状态。

关键输出片段：

```text
Start Port    End Port
----------    --------
      1802        1901      ← 1883 落在此区间
```

## 修复方案

方案一为推荐治本方案；二至四为无法采用方案一时的备选，按临时性递减排列。

### 方案一：修正动态端口范围（治本 · 推荐，管理员 PowerShell）

```bash
netsh int ipv4 set dynamicport tcp start=49152 num=16384
netsh int ipv6 set dynamicport tcp start=49152 num=16384
```

重启生效。此后保留段只在 49152+，不再干扰 1883/3306/6379/8080 等开发端口。开发机统一以此方案为基线。

### 方案二：换端口（止血）

```yaml
ports:
  - "11883:1883"
```

`mosquitto_pub` 改用 `-p 11883`。适合短期联调、设备不写死 1883。

### 方案三：临时夺回 1883（管理员 PowerShell）

```bash
net stop winnat
docker compose up -d --force-recreate <svc>-emqx
net start winnat
```

利用 winnat 停止时释放保留段，让 Docker 抢占。重启后可能复现，适合临时应急。

### 方案四：管理员白名单锁定 1883（管理员 PowerShell）

```bash
net stop winnat
netsh int ipv4 add excludedportrange protocol=tcp startport=1883 numberofports=1
net start winnat
```

验证：`netsh int ipv4 show excludedportrange protocol=tcp` 列表出现 `1883  1883  *`（`*` 为管理员保留）。适合必须用 1883 且长期的场景。

## 验证

```text
# docker ps
0.0.0.0:1883->1883/tcp, ...      ← 前缀出现，映射生效

# mosquitto_pub
（无 Connection refused，消息到达 EMQX Dashboard）
```

## 要点

- `docker ps` 的 `PORTS` 列缺 `0.0.0.0:port->` 前缀 = 映射未生效；不报错时优先查宿主机网络层而非 compose 文件
- 容器内 `0.0.0.0:1883 started` ≠ 宿主机可达，需分层验证到宿主机映射
- Windows 端口被保留不报错，必须主动 `netsh int ipv4 show excludedportrange protocol=tcp`
- 临时夺回 / 换端口治标，`set dynamicport tcp start=49152` 才是根治
- WSL2 → Windows Docker 排查顺序：`docker ps` → `netstat` → `netsh`，最后才考虑 WSL 转发（`host.docker.internal`）
- 团队开发机统一执行方案一，纳入 onboarding SOP；嵌入式设备写死 1883 场景叠加方案四双重保险
