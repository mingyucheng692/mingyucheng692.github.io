---
title: "IIoT 接入路徑排查與修復紀錄：Podman Rootless 下 EMQX 5.8 部署故障復盤"
date: 2026-05-28T18:59:25+08:00
draft: false
tags: ["IIoT", "MQTT", "EMQX", "Podman", "Rootless", "Erlang", "HOCON", "Webhook", "CSRF", "Postmortem"]
categories: ["Tech", "Engineering", "Infrastructure", "Security"]
summary: "記錄一次 IIoT 接入路徑部署故障的完整排查：在 Podman Rootless 環境下，EMQX 5.8 同時暴露出 Erlang IPC 失效、HOCON Schema 驗證失敗、安全性群組未開放與 M2M 請求被 CSRF 誤攔截等問題，並整理最終修復方案與驗證結果。"
url: "/zh-tw/blog/tech/iiot-emqx-rootless-deployment-postmortem/"
---

這篇文章記錄一次 IIoT 接入路徑部署故障的排查與修復過程。目標是在雲端透過 Podman Rootless 容器執行 `EMQX 5.8.9`，完成設備 MQTT 接入、HTTP 動態鑑權、Webhook 轉發與時序資料寫入。

實際推進中，這條路徑先後暴露出四類問題：`emqx ctl` 在容器內無法穩定連回主節點、改用預寫 `cluster.hocon` 後 EMQX 在啟動階段因 Schema 驗證失敗而崩潰、外部設備連線持續逾時，以及後端對 EMQX 發起的 M2M HTTP 請求返回 `403 Forbidden`。這些現象分散在執行期、配置、網路與安全中介軟體四個層面，如果只盯著單一報錯，很容易誤判為某一處設定寫錯。

> 說明：本文中的專案名、服務名、網域、IP、路徑、帳號、資料表名、請求標頭、日誌片段與命令輸出均已脫敏、抽象或重寫，只保留與排查路徑相關的技術資訊。

## 背景與目標

- 容器執行環境：Podman Rootless
- Broker：`EMQX 5.8.9`
- 鑑權方式：HTTP AuthN
- 資料路徑：MQTT -> Rule Engine -> Webhook -> TimescaleDB
- 後端安全：瀏覽器側 CSRF 防護 + M2M Token 驗證

目標本身並不複雜：設備透過 MQTT 接入，EMQX 呼叫後端完成動態認證，再把遙測資料透過 Webhook 轉發給後端，最後寫入時序資料庫。

## 故障摘要

這次故障最後確認並不是單點問題，而是四條故障路徑疊加：

1. Podman Rootless 的隔離機制導致 Erlang EPMD IPC 不穩定，`emqx ctl` 無法作為可靠的動態載入入口
2. EMQX 5.8.9 對 Webhook Bridge 配置執行更嚴格的 HOCON Schema 驗證，歷史欄位觸發 `unknown_fields`
3. 雲端安全性群組未開放 `1883` 入站流量，外部 MQTT 連線直接被阻斷
4. 後端全域 CSRF 中介軟體預設保護所有 POST 請求，誤傷 EMQX 發起的 AuthN 與 Webhook 呼叫

最終修復也分別對應這四條路徑：

- 放棄依賴 `emqx ctl` 的動態載入，改為在 Entrypoint 前置階段渲染完整 `cluster.hocon`
- 移除不再相容的 `resource_opts` 歷史欄位
- 開放安全性群組 `1883`
- 對 `/api/v1/iot/*` 路徑繞過 CSRF，同時在 `/iot/ingest` 上增加常數時間 `X-EMQX-Token` 驗證

---

## 排查時間線

### 第一階段：先確認為什麼動態配置不可靠

最初部署方案是在 EMQX 主行程啟動後，透過腳本執行：

`emqx ctl conf load --merge`

把 HTTP AuthN 與 Webhook 規則動態注入 Broker。實際執行時，容器內持續出現：

```text
Node emqx@<service>-emqx not responding to pings
```

這一步的關鍵結論是：問題暫時還不在業務配置，而是在執行期控制層。只要 `emqx ctl` 不能穩定透過 Erlang 分散式節點協議連回主節點，後續的動態配置方案就沒有可靠性基礎。

#### 假設 A：節點 Hostname 綁定錯誤

第一反應是節點名 `emqx@<service>-emqx` 的解析異常，因此在 Compose 中顯式宣告 `hostname`。這一步修正後，節點命名確實更一致，但 `emqx ctl` 仍然時好時壞，說明 Hostname 只是干擾項，不是根因。

#### 假設 B：`sed` 佔位符替換被 Compose 提前展開

另一個已確認的問題是命令裡的 `${VAR}` 會被 `podman-compose` 提前展開，導致 `sed` 的替換目標被吃掉，Token 注入失效。將替換目標改為靜態佔位字串後，Token 渲染恢復正常，但 `emqx ctl` 依然不可靠。這說明模板替換確實有缺陷，但仍不是整條部署路徑失敗的主因。

### 第二階段：繞開 `emqx ctl`，改為啟動前寫配置

既然問題集中在執行期 IPC，排查方向自然收斂到「是否能在主行程啟動前就完成配置注入」。後續做法是把渲染後的配置直接寫入：

`/opt/emqx/data/configs/cluster.hocon`

再由 Entrypoint `exec` 啟動 EMQX。這個方向繞開了 `emqx ctl`，但立刻暴露出第二個阻塞點：EMQX 在啟動階段直接崩潰。

日誌中的關鍵訊息為：

```log
[error] failed_to_check_schema: emqx_conf_schema
[error] #{reason => unknown_fields,
           path => "bridges.webhook.<service>_backend_ingest_bridge.resource_opts",
           unknown => "pool_size,buffer_type,max_retries,..."}
```

這一步的結論也很清楚：預寫配置本身是正確方向，但模板中仍帶有舊版本可接受、當前版本已不再相容的歷史欄位。

#### 假設 C：只精簡部分 `resource_opts` 欄位即可恢復

初始做法是先刪除 `buffer_type`、`max_retries` 等欄位，保留看起來仍合理的 `pool_size`。結果 EMQX 仍然在 Schema 驗證階段失敗，說明這裡不是「個別欄位值不合法」，而是整個 `resource_opts` 子樹都已不再適配當前版本。最終將整塊完全移除後，EMQX 才恢復正常啟動。

#### 同一階段暴露出的配置覆蓋風險

這一階段還暴露出另一個隱患：`cluster.hocon` 改成覆蓋寫入後，如果模板只包含 Webhook 規則，就可能把先前透過 Dashboard 設定的 HTTP AuthN 一併覆蓋掉，導致 Broker 雖然能啟動，但設備接入鑑權路徑反而被切斷。

因此最終模板調整為一次性宣告三類配置：

- HTTP AuthN
- Rule Engine 規則
- Webhook Bridge

這一步不是效能或可維護性優化，而是為了讓 Broker 啟動配置保持原子性，避免不同入口寫同一份生效配置時互相覆蓋。

### 第三階段：Broker 啟動後，外部設備仍然連不上

當 EMQX 已能正常啟動，且容器日誌確認：

```text
Listener tcp:default on 0.0.0.0:1883 started
```

外部客戶端連線仍然持續逾時。此時排查對象已從 Broker 內部轉移到外圍網路層。

本地使用 `mosquitto_pub` 直接測試 `<IP>:1883` 後依然逾時，這一步將問題範圍縮小到 IaaS 層。最終確認是雲端安全性群組沒有開放 `1883` 入站流量。補齊規則後，設備到 Broker 的 MQTT 連通性恢復。

這一步非常典型：容器內連接埠監聽正常，不代表外部存取路徑已經打通。只要安全性群組或外層防火牆未放行，應用側日誌就會呈現「服務正常、客戶端逾時」的假象。

### 第四階段：M2M HTTP 請求被後端安全中介軟體攔截

MQTT 連線打通後，EMQX 發起的 HTTP AuthN 與 Webhook 請求又返回 `403 Forbidden`。繼續檢查後端邏輯後，問題定位到全域 CSRF 中介軟體。

後端原本面向瀏覽器流量啟用了嚴格的 `Origin` / `Referer` 驗證，這對 Web 管理端是合理的，但對 EMQX 發起的機器對機器請求並不適用。問題不在於這類請求「不能攜帶」相關標頭，而在於它們並不處於瀏覽器信任模型內，無法穩定滿足基於 `Origin` / `Referer` 的 CSRF 驗證前提，因此會被誤攔截。

這裡的處理不是簡單關掉 CSRF，而是做安全分流：

- `/api/v1/iot/*` 前綴繞過 CSRF
- 資料接收端點 `/iot/ingest` 額外驗證 `X-EMQX-Token`
- Token 比對使用 `crypto/hmac.Equal`，避免時序側信道

如此處理後，Web 瀏覽器端仍保留原有 CSRF 防護，而 IoT M2M 路徑則改用更適合機器呼叫場景的預共享密鑰驗證。

---

## 根因分析

### 1. Podman Rootless 與 Erlang EPMD 的執行期阻抗

`emqx ctl` 本質上依賴 Erlang Distribution Protocol 與 EPMD 進行節點發現與控制。在 Rootless 場景下，User Namespace 與虛擬網路堆疊改變了容器內回環與本地 IPC 的行為，使這條控制面路徑無法穩定建立。因此問題不在於 `emqx ctl` 命令本身寫錯，而在於它不適合作為目前執行環境中的核心配置入口。

### 2. EMQX 5.8.x 對歷史 HOCON 欄位收緊

舊版本 Bridge 配置中常見的 `resource_opts.pool_size`、`buffer_type`、`max_retries` 等欄位，在 `5.8.x` 中已不再按原路徑生效。當前版本在啟動階段就會對未知欄位執行強驗證，因此這類歷史配置不會被忽略，而是直接阻斷 Broker 啟動。

### 3. 外圍網路策略與應用日誌之間存在錯位

EMQX 容器內成功監聽 `1883`，只能證明 Broker 完成本地綁定；它並不能證明外部設備到 Broker 的接入路徑已經打通。當客戶端逾時、而應用層沒有明顯異常日誌時，排查就必須繼續向外走到安全性群組、宿主機防火牆與入口轉發層。

### 4. 瀏覽器安全策略不能直接套用到 M2M 路徑

CSRF 是為瀏覽器上下文設計的防護模型，而 EMQX 與後端之間的 AuthN / Webhook 呼叫屬於 M2M 通信。若直接將同一套瀏覽器導向的驗證套在兩類流量上，就會把不符合瀏覽器來源判定前提的合法 Broker 請求誤判為可疑流量。因此正確做法不是「少做安全」，而是讓不同呼叫者類型使用不同的安全機制。

---

## 修復方案

最終落地方案包含四個部分。

### 1. 啟動前渲染完整 `cluster.hocon`

放棄執行期動態載入，在容器 Entrypoint 前置階段渲染最終配置並寫入，之後再啟動 EMQX。這樣做的目的，是把配置生效時機前移到 Broker 啟動前，避免繼續依賴不穩定的執行期 IPC。

### 2. 合併 AuthN、Rule Engine 與 Webhook 模板

最終模板統一包含：

- HTTP AuthN
- 遙測轉發規則
- Webhook Bridge

同時完全刪除不再相容的 `resource_opts` 區塊。

### 3. 打通外部網路入口

在確認 EMQX 已監聽 `0.0.0.0:1883` 後，補齊雲端安全性群組入站規則，恢復設備從外部存取 Broker 的通路。

### 4. 重構 M2M 安全策略

後端現在對 `/api/v1/iot/*` 繞過瀏覽器導向的 CSRF 驗證，同時在 `/iot/ingest` 上啟用共享密鑰驗證。最終形成兩套並行防護：

- 瀏覽器端：繼續使用 CSRF
- IoT M2M 端：使用 `X-EMQX-Token`

這讓後端安全邊界更清楚，也避免後續再把瀏覽器流量與設備流量混用同一套驗證模型。

---

## 驗證結果

修復後，驗證分三層完成。

### 1. 後端中介軟體驗證

針對 CSRF 與 Token 中介軟體的測試確認：

- 普通 Web/API 請求在缺失或偽造 `Origin` / `Referer` 時仍返回 `403`
- `/api/v1/iot/*` 前綴請求可繞過 CSRF
- `/iot/ingest` 在缺失或錯誤 Token 時被拒絕

### 2. MQTT 端到端驗證

使用外部客戶端執行發布測試：

```bash
mosquitto_pub -d -h <Domain> -p 1883 \
  -i "<Device_SN>" -u "<Device_SN>" -P "<Token>" \
  -t "telemetry/fms/<Device_SN>" \
  -m '{"fcs_speed_rpm": 7766, "fcs_soc": 88.9, "fms_fault_code": 0}'
```

驗證結果表明：

- 設備連線成功
- HTTP AuthN 返回通過
- 訊息被 Broker 接收並進入 Rule Engine

### 3. 資料寫入驗證

時序資料庫查詢確認資料寫入成功，`_iot` 資料面角色權限與控制面角色保持隔離，整條路徑形成閉環。

---

## 經驗與後續

這次排查沉澱出幾條比較明確的工程經驗：

- 在 Rootless Podman 下，凡是可以透過宣告式配置解決的問題，優先不要依賴啟動後的 Erlang IPC 控制路徑
- 中介軟體跨版本升級時，高級調優欄位比基礎功能配置更容易失效，配置應盡量從最小集開始驗證
- 「服務已監聽」不等於「外部可達」，網路連通性必須一路驗證到安全性群組或防火牆
- 瀏覽器安全機制與 M2M 安全機制應明確分流，避免彼此誤傷

從目前階段來看，單節點模板渲染方案已滿足 MVP 驗證需求，但若繼續演進到生產高可用架構，後續仍需要關注：

- Broker 叢集化與配置中心化
- Webhook 非同步化，避免後端被同步寫入流量放大
- 從共享密鑰逐步演進到雙向 TLS 或更強的設備身分體系

這次修復的價值，不在於消除了某一條啟動報錯，而在於把原本分散在執行期、配置、網路與安全層的故障路徑，重新整理成一條可解釋、可驗證、可重複執行的接入流程。
