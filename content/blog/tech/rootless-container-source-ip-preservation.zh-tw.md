---
title: "Rootless 容器入向源 IP 保留：架構權衡與 Hybrid Host-Bridge 拓撲落地記錄"
date: 2026-07-11T20:00:00+08:00
draft: false
tags: ["Podman", "Rootless", "Nginx", "EMQX", "Networking", "NAT", "Source-IP", "Container", "Postmortem"]
categories: ["Tech", "Engineering", "Infrastructure", "Security"]
summary: "記錄一次 Rootless Podman 下入向源 IP 被 NAT 改寫問題的架構分析與方案落地。透過 Hybrid Host-Bridge 拓撲在邊緣服務保留客戶端真實 IP，同時維持內部服務的網路隔離，並給出方案選型權衡、實作細節與驗證方法論。"
url: "/zh-tw/blog/tech/rootless-container-source-ip-preservation/"
---

入向流量源 IP 被 Rootless 端口轉發器的 NAT 改寫為容器網橋閘道 IP，導致稽核、限流與診斷鏈路失效。這是 Rootless 使用者命名空間設計的結構性約束，無法透過設定規避。最終採用 Hybrid Host-Bridge 拓撲：邊緣服務（Nginx、EMQX）切換至 Host Network 保留客戶端真實 IP，內部服務（後端、DB、Redis）保留 Bridge Network 維持隔離，loopback 作為跨命名空間通訊通道。該方案在單機部署、邊緣服務數量有限、無 NetworkPolicy 需求的約束下成立。

> 說明：本文中的專案名稱、服務名稱、網域名稱、IP、帳號、路徑、日誌片段與命令輸出均已脫敏、抽象或改寫，僅保留與架構決策相關的技術資訊，不對應任何真實生產識別碼。

## 根因：Rootless NAT 機制

系統由四類服務構成：前端反向代理（Nginx）、MQTT Broker（EMQX）、業務後端（Go）、資料層（TimescaleDB + Redis）。容器執行階段為 Rootless Podman，部署在雲端主機上，外部客戶端包含瀏覽器與 IoT 裝置兩類。

問題表現為：應用存取日誌與 Broker 日誌中客戶端源 IP 統一被記錄為容器網橋位址（如 `10.89.0.7`），基於源 IP 的速率限制因單一網橋 IP 命中限流而導致全部客戶端被阻斷，稽核與診斷鏈路缺乏可信的客戶端地理來源資訊。

根因在於 Rootless 容器的網路隔離依賴使用者命名空間（User Namespace），非特權使用者無法修改宿主機路由表，也無法繫結特權埠（< 1024）。為繞開這一約束，Podman 透過使用者態端口轉發器（`rootlessport` 或 `slirp4netns`）將宿主機埠對映進容器網橋命名空間，流程如下：

1. 埠轉發器在宿主機側監聽目標埠
2. 外部連線到達後，轉發器在容器網橋命名空間內發起新連線
3. 新連線的來源位址被改寫為網橋閘道 IP（如 `10.89.0.7`）
4. 容器內的 `$remote_addr` 與後續轉發的 `X-Forwarded-For` / `X-Real-IP` 均基於改寫後的位址

```
[External Client] ---> (Host TCP Port 443/1883)
                             |
                   [Rootless Port Forwarder] (NAT Layer)
                             |
                             v  (Source IP rewritten to Gateway IP)
        [Edge Container (Bridge Namespace)]
                             |
                             v  (Propagates rewritten IP via HTTP Headers)
             [Backend Container (Bridge Network)]
```

NAT 發生在容器命名空間邊界之前，邊緣容器無法從連線本身取得原始客戶端 IP。只要流量經埠轉發器進入容器網橋，源 IP 改寫就不可避免，這是方案選型的前置約束。

---

## 方案選型與權衡

在確認根因後，候選方案有三類。下表從源 IP 保留、網路隔離、實作複雜度、維護成本四個維度對比：

| 方案 | 源 IP 保留 | 網路隔離 | 實作複雜度 | 維護成本 |
|---|---|---|---|---|
| A. Proxy Protocol | 需轉發器與上游均支援 | 保留 | 中 | 協議相容性依賴強 |
| B. 全量 Host Network | 保留 | 喪失 | 低 | 埠衝突風險隨服務數增長 |
| C. Hybrid Host-Bridge | 保留（邊緣） | 部分（內部） | 中 | 邊界清晰，耦合度低 |

### 方案 A：Proxy Protocol

透過在轉發器與上游之間引入 Proxy Protocol 傳遞原始 IP。該方案的前提是 Rootless 埠轉發器需支援輸出 PROXY protocol 標頭，且所有上游（Nginx、EMQX）需開啟對應接收設定。目前 `rootlessport` 對 PROXY protocol 輸出的支援有限，且 EMQX 的 MQTT 監聽器與 HTTP 監聽器的 Proxy Protocol 設定路徑不同，跨協議維護成本較高。

### 方案 B：全量 Host Network

所有容器切換至 `network_mode: "host"`。源 IP 保留問題消除，但資料層（TimescaleDB、Redis）埠直接暴露在宿主機介面上，喪失網路級隔離。後續若擴充多實例或引入網路策略，埠衝突與隔離缺失會增加維護成本。

### 方案 C：Hybrid Host-Bridge（最終採用）

僅將需要接收外部流量的邊緣服務（Nginx、EMQX）切換至 Host Network，內部服務（後端、資料庫、快取）保留在 Bridge Network。邊緣服務透過 loopback 與後端通訊，後端仍維持網路隔離。

該方案以邊緣服務的網路隔離換取源 IP 保留能力，同時透過 loopback 邊界限制內部服務的暴露面。在當前單機部署、邊緣服務數量有限的約束下，該權衡可接受。

---

## 落地實作：Hybrid Host-Bridge 拓撲

### 網路拓撲

```
                              HOST NETWORK NAMESPACE
+---------------------------------------------------------------------------------+
|                                                                                 |
|  [External Client] ---> (Port 443) ---> [Nginx (Host Mode)]                     |
|                                                |                                |
|  [IoT Gateway]     ---> (Port 1883) ---> [EMQX (Host Mode)]                     |
|                                                |                                |
|                                                v (Localhost Tunnel)             |
|                                          [127.0.0.1:<backend-port>]             |
|                                                |                                |
+------------------------------------------------|--------------------------------+
                                                 | (Port Publish, loopback only)
                                                 v
                               +----------------------------------+
                               |     CONTAINER BRIDGE NETWORK     |
                               |           (<bridge-net>)         |
                               |                                  |
                               |        [<backend-service>]       |
                               |          /           \           |
                               |         /             \          |
                               |        v               v         |
                               |   [<db-service>] [<redis-service>]|
                               |   (Port 5432)     (Port 6379)    |
                               +----------------------------------+
```

### 關鍵實作點

**1. Host Kernel 埠繫結閾值檢查**

Nginx 需在 Host Network 下監聽 80/443，但 Rootless 使用者預設無法繫結 < 1024 埠。該閾值是全域核心引數，非容器級設定。若前期 HTTPS 設定階段已下放該閾值，此處僅作確認檢查；若未下放，需在單機部署場景下評估全域下放的可接受性後執行。

```bash
# 檢查當前閾值（HTTPS 設定階段通常已下放至 80）
sysctl net.ipv4.ip_unprivileged_port_start

# 若輸出非 80，則下放（Nginx 需要 80 做 HTTP→HTTPS 重定向）
sudo sysctl net.ipv4.ip_unprivileged_port_start=80
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee -a /etc/sysctl.conf
```

多租戶環境下需評估是否引入更細粒度的埠許可權控制（見後續演進）。

**2. Compose 設定調整**

邊緣服務移除 `ports` 與 `networks` 欄位後，容器直接共享宿主機網路命名空間，繞開埠轉發器；後端服務僅發布到 loopback，避免在 Host Network 切換後暴露至宿主機介面。

```yaml
services:
  <frontend-service>:
    network_mode: "host"
    # ports 與 networks 欄位移除

  <emqx-service>:
    network_mode: "host"
    environment:
      - EMQX_AUTHORIZATION__SOURCES__1__URL=http://127.0.0.1:<backend-port>/api/v1/iot/auth

  <backend-service>:
    ports:
      - "127.0.0.1:<backend-port>:<backend-port>"  # 僅 loopback
    environment:
      - <EMQX_BASE_URL>=http://host.containers.internal:<dashboard-port>
```

`host.containers.internal` 是 Podman 提供的特殊 DNS 名稱，容器內可透過該名稱存取宿主機命名空間的服務，替代原 Bridge Network 內的容器間直連。該 DNS 僅用於容器→宿主機方向；反向（宿主機→容器）仍需透過埠發布。

**3. 反向代理與 Broker 設定收斂**

邊緣服務上游位址統一指向 loopback，確保跨命名空間通訊僅經 loopback 邊界。`$remote_addr` 在 Host Network 模式下即為客戶端真實 IP，無需依賴上游轉發標頭。

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:<backend-port>;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

```hocon
authentication = [
  { backend = "http", url = "http://127.0.0.1:<backend-port>/api/v1/iot/auth" }
]
bridges.webhook.<ingest-bridge> {
  url = "http://127.0.0.1:<backend-port>/api/v1/iot/ingest"
}
```

**4. 重新部署與 Supervisor 層狀態同步**

設定變更後不能直接 `podman-compose up` 拉起容器。原部署透過 systemd unit 管理容器生命週期，而 unit 檔案是基於原 Bridge Network 模式生成的。網路模式從 Bridge 切換至 Host 後，systemd unit 需由部署腳本重新生成以捕獲新的網路屬性，隨後將控制權移交 systemd。若跳過此環節，舊 unit 會以原 Bridge 模式拉起容器，導致設定與執行時不一致，且開機自啟可能失效。

```bash
# 透過原部署腳本執行，內部重新生成 systemd unit 並移交管理
./<deploy-script>  # CI: 部署流水線應校驗舊 unit 已停止後再生成新 unit，避免新舊 unit 並存導致雙實例
```

---

## 驗證策略

驗證需覆蓋全鏈路，分三層完成。三層驗證的邏輯關係：Nginx 日誌確認邊緣層已取得真實 IP；後端日誌確認 IP 透過 HTTP 標頭正確傳播；EMQX 日誌確認 Broker 層 AuthN/Webhook 鏈路已收斂至 loopback。三者缺一不可——僅驗證 Nginx 無法確認後端是否正確解析標頭，僅驗證後端無法確認 Broker 鏈路是否已調整。

### 1. Nginx 日誌驗證

```bash
podman logs <frontend-service>
```

預期：日誌條目起始 IP 為客戶端公網 IP，而非網橋閘道位址。

```
<client-public-ip> - - [<timestamp>] "GET /api/v1/projects HTTP/1.1" 200 ...
```

### 2. 後端結構化日誌驗證

```bash
podman logs <backend-service>
```

預期：JSON 日誌中 `"ip"` 欄位與 Nginx 日誌中的客戶端 IP 一致。

```json
{"level":"INFO","ts":"<timestamp>","caller":"middleware/logger.go:<line>","msg":"Inbound Request","ip":"<client-public-ip>","status":200,"method":"GET","path":"/api/v1/<resource>"}
```

### 3. EMQX 整合驗證

```bash
podman logs <emqx-service>
```

預期：監聽器繫結至 `0.0.0.0`，AuthN 與 Webhook 請求路由至 `127.0.0.1`。

```
EMQX_AUTHORIZATION__SOURCES__1__URL [authorization.sources.1.url]: http://127.0.0.1:<backend-port>/api/v1/iot/auth
Listener tcp:default on 0.0.0.0:1883 started.
Listener http:dashboard on :<dashboard-port> started.
```

---

## 工程約束與後續演進

### 當前方案的有效邊界

Hybrid Host-Bridge 拓撲在以下約束下成立：

- **單機部署**：Host Network 的埠衝突風險在多機/叢集場景下放大
- **邊緣服務數量有限**：每增加一個 Host Network 服務，宿主機埠管理複雜度線性增長
- **無網路策略需求**：Rootless Podman 的 Host Network 不支援 NetworkPolicy，隔離依賴 loopback 邊界

### 已識別的後續演進方向

**1. 從 Host Network 遷移至 Pasta**

pasta 模式在 Podman 4.x 引入作為 slirp4netns 的替代，在 Podman 5.0 中已成為 rootless 預設網路棧（配合 netavark + nftables）。相對 slirp4netns 的關鍵改進是不再對入向連線做 SNAT，源 IP 保留因此在 Bridge 模式下也具備可行性。當執行階段版本升級時，可評估是否回退邊緣服務至 Bridge Network，恢復網路隔離。

**2. 埠繫結許可權的細粒度控制**

當前透過全域 `sysctl` 下放埠閾值。在引入更多服務時，可評估基於 systemd 的 `AmbientCapabilities=CAP_NET_BIND_SERVICE` 或 RootlessKit 的埠白名單機制，避免全域閾值下放。

**3. 監控與告警覆蓋**

當前驗證依賴手動日誌檢查。源 IP 保留的失效在執行期不會產生明顯報錯，需透過監控項（如日誌中網橋 IP 出現頻率、速率限制命中率異常）建立被動發現能力。

**4. 多實例與橫向擴充**

當前單機拓撲下，Host Network 的埠繫結是獨佔的。若後續邊緣服務需要多實例或灰度發布，需引入上游負載均衡層，屆時源 IP 保留問題會轉移至 LB 層，需在 LB 層重新評估 Proxy Protocol 或保留客戶端 IP 的方案。

---

## 沉澱的工程約束

- **網路隔離與源 IP 保留存在固有張力**：Host Network 保留 IP 但喪失隔離，Bridge Network 保留隔離但改寫 IP。方案選型的核心工作是在兩者之間劃定邊界。
- **Hybrid 拓撲的邊界應沿資料流向劃分**：接收外部流量的邊緣服務用 Host Network，服務間通訊的內部服務用 Bridge Network，loopback 作為跨命名空間通訊通道。
- **驗證需覆蓋全鏈路**：單點日誌不足以確認端到端 IP 傳播正確性，需從邊緣到後端逐層核對。
- **網路模式變更需同步 supervisor 層狀態**：容器網路模式切換後，supervisor（如 systemd）的 unit 檔案需重新生成以捕獲新屬性，否則會出現設定與執行時不一致、開機自啟失效等問題。

本文沉澱的約束可作為後續執行階段升級或拓撲變更的參考基線。
