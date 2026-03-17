---
title: "記一次 HTTPS 升級引發的 403 排查：從安全中介軟體到容器隔離機制的深度復盤"
date: 2026-03-17T12:00:00+08:00
draft: false
tags: ["HTTPS", "Nginx", "Go", "Gin", "CORS", "CSRF", "Podman", "Rootless", "Postmortem"]
categories: ["Tech", "Engineering", "Security"]
summary: "一次 HTTPS 協議升級後穩定復現 403 的排障復盤，最終定位為 CSRF 白名單遺漏與 Podman Rootless 映像隔離的疊加問題。"
url: "/zh-tw/blog/tech/https-upgrade-403-postmortem/"
---

近期在推進核心業務線基礎架構安全演進時，我們將全域 HTTP 協議升級為 HTTPS。Nginx 閘道層完成 SSL 配置並啟用 443 強制跳轉後，回歸測試穩定復現登入介面的 `403 Forbidden`。

返回報文如下：

```json
{"code":403,"msg":"forbidden: invalid origin"}
```

系統架構：

- 前端：Vue
- 後端：Go + Gin
- 閘道：Nginx 反向代理
- 執行環境：Podman（Rootless 模式）

---

## 排查策略：自頂向下，逐層剝離

本次排查遵循「邊界清晰、逐層驗證」原則，依序穿透閘道層、應用層與容器執行時層。

## 第一層：閘道層（Nginx）

第一反應是 Nginx 未正確透傳請求標頭。檢查配置後確認 `Origin` 等關鍵標頭已正常透傳。

除錯過程中發現，只要在閘道中強行注入：

```nginx
proxy_set_header Origin "http://業務域名";
```

介面就能「恢復正常」。

但這個方案本質是在偽造請求來源，用 HTTP 值去冒充 HTTPS 請求，等同削弱後端 CSRF 防線。該 workaround 被明確廢棄，繼續向應用層深挖。

## 第二層：應用層（CORS 與 CSRF 的錯位）

抓包確認回應標頭中已返回：

```http
Access-Control-Allow-Origin: https://業務域名
```

說明 CORS 中介軟體對 HTTPS 來源已放行。問題在於：

- CORS 負責「瀏覽器能否讀取跨域回應」
- CSRF 負責「後端是否信任請求來源」

兩者並行但獨立。

最終在程式碼中定位到核心遺漏：協議升級時只更新了 CORS 白名單，CSRF 中介軟體仍保留舊的 `http://` 信任列表。請求通過了 CORS，卻被 CSRF 攔截並返回 `invalid origin`。

修復動作：同步補齊 CSRF 中介軟體中的 `https://` 域名白名單。

## 第三層：執行時層（Podman Rootless 隔離）

程式碼修復並觸發建置後，線上現象仍未消失。進一步透過 `podman inspect` 比對映像雜湊，發現一個工程化陷阱：

- 生產服務運行在 `deploy` 帳號的 Rootless 命名空間
- 部分映像曾在 Root 命名空間建置
- Rootless 隔離下，`deploy` 無法感知 Root 空間中的新映像

結果是部署腳本反覆重啟舊映像，形成「程式碼已修復、環境未生效」的假象。

---

## 根因總結

這次故障不是單點 Bug，而是兩類系統性遺漏疊加：

1. 架構規範缺口：CORS 與 CSRF 白名單分散配置，協議升級時缺乏統一治理入口。
2. 工程閉環缺失：生產映像建置權限未完全收斂到標準化 CI/CD，導致跨命名空間髒狀態。

---

## 流程改進與行動項

### 1）完善安全發版 SOP 與 CheckList

將「CORS 與 CSRF 策略一致性校驗」納入強制發布檢查項，新專案上線與協議升級必須通過該關卡。

### 2）收斂生產環境運維準入

明確應用容器僅允許透過 CI/CD 流水線建置與發布，禁止人工跨權限命名空間干預線上服務。

### 3）推進可觀測性規範

將中介軟體載入與路由註冊順序納入架構規範，確保日誌層前置於安全攔截層，避免「被攔截但無日誌」的幽靈請求。

---

## 結語

這次 403 排查的價值不在於「把介面調通」，而在於暴露了安全配置治理與交付鏈路上的薄弱點。技術修復解決當下，流程與制度收斂才是避免同類問題重演的關鍵。
