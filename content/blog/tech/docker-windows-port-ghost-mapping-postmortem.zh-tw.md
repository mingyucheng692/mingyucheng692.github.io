---
title: "Windows Docker 連接埠幽靈映射排查：EMQX 1883 被 WinNAT 靜默保留的覆盤"
date: 2026-08-25T12:00:00+08:00
draft: false
tags: ["Docker", "Windows", "WinNAT", "Hyper-V", "EMQX", "MQTT", "Postmortem"]
categories: ["backend-infra"]
summary: "Windows 開發機 Docker Desktop 完整服務棧（backend + EMQX 5.8.9 + TimescaleDB + Valkey），後端 ping/pong 正常但 mosquitto_pub 持續 Connection refused。docker ps 中同棧其他連接埠正常映射，唯獨 1883 缺 0.0.0.0:1883-> 前綴且不報錯，根因為 Hyper-V/WinNAT 動態連接埠保留圈佔 1883（落在 1802–1901 區間）。給出換連接埠/臨時奪回/白名單/修正動態連接埠範圍四級修復方案。"
url: "/zh-tw/blog/tech/docker-windows-port-ghost-mapping-postmortem/"
---

> 環境與命令均已脫敏：容器名稱、帳號、密碼、Token、Topic、payload 抽象為佔位符。

環境：Docker Desktop（Windows）完整服務棧（backend + EMQX 5.8.9 + TimescaleDB + Valkey），鏈路為 WSL2 `mosquitto_pub` → 宿主機 `127.0.0.1:1883` → EMQX 容器。

## 現象

```bash
mosquitto_pub -d -h 127.0.0.1 -p 1883 -i "<Client_ID>" -u "<User>" -P "<Password>" -t "<Topic>" -m '<Test_Payload_JSON>'
# Error: Connection refused
```

容器內 1883 正常監聽，宿主機映射缺失，Docker 全程不報錯。

## 關鍵證據

完整服務棧同一次 `docker compose up` 起來，後端應用層正常回應：

```bash
curl 127.0.0.1:8080/ping
# {"msg":"pong"}
```

但 `docker ps` 中唯獨 EMQX 的 1883 缺前綴：

```text
NAMES            PORTS
<svc>-backend    0.0.0.0:8080->8080/tcp
<svc>-emqx       0.0.0.0:8083->8083/tcp, 1883/tcp, 0.0.0.0:18083->18083/tcp
<svc>-db         0.0.0.0:5432->5432/tcp
<svc>-valkey      0.0.0.0:6379->6379/tcp
```

同一台機器、同一份 compose、同一次 up：backend（8080）、db（5432）、valkey（6379）、emqx 的 8083/18083 全部正常映射，唯獨 1883 無 `0.0.0.0:1883->` 前綴（僅容器內監聽）。問題被鎖定到 1883 這個連接埠本身，而非 Docker 或 compose。

## 假設與驗證

| # | 假設 | 驗證命令 | 結果 | 結論 |
|---|---|---|---|---|
| A | 配置未重新載入 | `docker compose up -d --force-recreate <svc>-emqx` | 1883 仍無前綴 | 排除 |
| B | YAML 隱藏字元致解析失敗 | `docker compose config` | 輸出含 `published: "1883"` | 排除 |
| C | 宿主機連接埠被處理序佔用 | `netstat -ano \| findstr :1883` | 無輸出 | 排除 |
| D | Windows 動態連接埠保留 | `netsh int ipv4 show excludedportrange protocol=tcp` | `1802  1901` 含 1883 | **確認根因** |

A–C 全部排除後，問題收斂到「Docker 讀到了配置，但綁定宿主機失敗且不報錯」。D 命中。

## 根因

Windows Hyper-V / WinNAT 的動態連接埠起始範圍被壓到 1024 附近，隨機保留大段連接埠給系統使用。1883 落在保留區間 `1802–1901` 內，Docker 綁定時被 Windows 底層靜默攔截——不拋 `bind: address already in use`，直接丟棄該連接埠映射，形成「容器內監聽正常、宿主機無映射、無報錯」的幽靈狀態。

關鍵輸出片段：

```text
Start Port    End Port
----------    --------
      1802        1901      ← 1883 落在此區間
```

## 修復方案

方案一為推薦治本方案；二至四為無法採用方案一時的備選，按臨時性遞減排列。

### 方案一：修正動態連接埠範圍（治本 · 推薦，管理員 PowerShell）

```bash
netsh int ipv4 set dynamicport tcp start=49152 num=16384
netsh int ipv6 set dynamicport tcp start=49152 num=16384
```

重啟生效。此後保留段只在 49152+，不再干擾 1883/3306/6379/8080 等開發連接埠。開發機統一以此方案為基線。

### 方案二：換連接埠（止血）

```yaml
ports:
  - "11883:1883"
```

`mosquitto_pub` 改用 `-p 11883`。適合短期聯調、裝置不寫死 1883。

### 方案三：臨時奪回 1883（管理員 PowerShell）

```bash
net stop winnat
docker compose up -d --force-recreate <svc>-emqx
net start winnat
```

利用 winnat 停止時釋放保留段，讓 Docker 搶佔。重啟後可能復現，適合臨時應急。

### 方案四：管理員白名單鎖定 1883（管理員 PowerShell）

```bash
net stop winnat
netsh int ipv4 add excludedportrange protocol=tcp startport=1883 numberofports=1
net start winnat
```

驗證：`netsh int ipv4 show excludedportrange protocol=tcp` 列表出現 `1883  1883  *`（`*` 為管理員保留）。適合必須用 1883 且長期的場景。

## 驗證

```text
# docker ps
0.0.0.0:1883->1883/tcp, ...      ← 前綴出現，映射生效

# mosquitto_pub
（無 Connection refused，訊息到達 EMQX Dashboard）
```

## 要點

- `docker ps` 的 `PORTS` 欄缺 `0.0.0.0:port->` 前綴 = 映射未生效；不報錯時優先查宿主機網路層而非 compose 檔案
- 容器內 `0.0.0.0:1883 started` ≠ 宿主機可達，需分層驗證到宿主機映射
- Windows 連接埠被保留不報錯，必須主動 `netsh int ipv4 show excludedportrange protocol=tcp`
- 臨時奪回 / 換連接埠治標，`set dynamicport tcp start=49152` 才是根治
- WSL2 → Windows Docker 排查順序：`docker ps` → `netstat` → `netsh`，最後才考慮 WSL 轉發（`host.docker.internal`）
- 團隊開發機統一執行方案一，納入 onboarding SOP；嵌入式裝置寫死 1883 場景疊加方案四雙重保險
