---
title: "IIoT 平台 H.265 HTTP-FLV 瀏覽器取流排查記錄"
date: 2026-06-28T21:30:00+08:00
draft: false
tags: ["HTTP-FLV", "H.265", "WASM", "Jessibuca", "JavaScript", "Python", "AbortController", "Debugging"]
categories: ["backend-infra"]
summary: "記錄一次 IIoT 平台 H.265 HTTP-FLV 瀏覽器取流問題的排查過程：從播放連結驗證、桌面播放器相容性確認、本地 FLV 基線驗證，到將 `fetchError`、`WinError 10053` 與前端 `TypeError` 串聯定位，並沉澱出可複用的調試 SOP。"
url: "/zh-tw/blog/tech/h265-http-flv-browser-stream-debugging/"
---

本文記錄一次 IIoT 平台 H.265 視訊流瀏覽器接入問題的排查過程。約束是前端不暴露高權限存取憑證。現象是：上游可返回播放連結，桌面播放器可播放，本地 `.flv` 可播放，但瀏覽器線上流失敗。

> 說明：本文中的地址、路徑、檔名、參數、請求標頭、認證資訊、日誌內容和時間點均已做脫敏、抽象或重寫，僅保留與排查鏈路相關的技術資訊。

## 背景與約束

接入方案受以下約束限制：

1. 頁面側需要播放 H.265 視訊流
2. 前端不能直接持有螢石雲高權限 `AT`

因此未採用要求前端直接持有高權限憑證的私有協議方案，而是採用 `HTTP-FLV + Jessibuca` 鏈路：

- 後端向螢石雲請求播放地址
- 後端控制帶時效的播放連結及參數
- 前端僅消費受控後的播放地址或代理地址
- 瀏覽器透過 Jessibuca 在頁面內完成 WASM 解碼

簡化鏈路如下：

```text
[Ezviz Cloud] ---> [Backend gets playback URL] ---> [Local Proxy / Controlled URL]
                                                     |
                                                     v
                                           [Browser + Jessibuca + WASM]
```

## 結論

- 螢石雲返回的播放連結不等於瀏覽器可直接消費的最終連結，桌面播放器驗證階段仍需補齊 `&supportH265=1` 參數
- 瀏覽器可以播放同源本地 `.flv` 檔案，說明 `Chrome + Jessibuca + WASM` 這條 H.265 解碼鏈路在目前環境中可工作
- 瀏覽器線上流失敗時，`fetchError`、代理側 `WinError 10053` 和頁面側 `TypeError` 屬於同一條故障鏈，不是三個獨立問題
- 根因是播放器 `kBps` 事件上報的是字串，頁面監聽器按數字呼叫 `.toFixed()` 後拋出異常，異常繼續進入 `fetch` 讀取鏈路並觸發 `AbortController`
- 修復措施包括：由後端統一控制播放連結，前端對第三方事件載荷做型別轉換、邊界檢查和異常隔離

## 排查路徑

排查按以下順序進行：

1. 從螢石雲取得播放連結，確認上游可以返回線上流地址
2. 用 `PotPlayer`  `VLC Media Player‌` 驗證線上流可消費性，補齊必要參數
3. 在網頁中播放本地 `.flv` 檔案，建立瀏覽器 H.265 基線
4. 最小化代理透傳邏輯，排除代理格式干擾
5. 結合瀏覽器 Console、Network 和播放器日誌，定位前端事件鏈異常

該順序用於分層驗證，避免同時混入協議、代理、瀏覽器和播放器實作層的問題。

## 詳細過程

### T0：確定方案邊界

目標：明確瀏覽器側存取邊界。

約束如下：

- 後端向螢石雲請求播放地址
- 前端不直接向螢石雲申請播放連結
- 頁面只消費受控後的線上流地址
- 瀏覽器側使用 Jessibuca 播放 `HTTP-FLV`

結論：這一階段只定義存取邊界。

### T1：先驗證線上流本身是否可消費

目標：確認上游返回的播放連結是否具備用戶端消費條件。

方法：使用 `PotPlayer` 驗證線上流。

初始驗證未通過，後續補齊以下參數後，`PotPlayer` 可以播放線上流：

- `protocol=4`   // 請求指定取得 HTTP-FLV，預設為螢石雲私有視訊協議
- `&supportH265=1` 相關透傳參數

觀察：

1. 螢石雲返回的不是無效連結
2. 線上流本身在桌面播放器側可消費

結論：問題範圍縮小到瀏覽器鏈路。

### T2：網頁線上流失敗，但不直接歸因於瀏覽器解碼

目標：確認網頁線上流失敗發生在哪一層。

觀察如下：

- 頁面初始化正常
- 播放請求可以發出
- 線上流在短時間內失敗
- 頁面側出現 `fetchError`
- 代理側出現 `WinError 10053`

候選原因如下：

- 瀏覽器解碼能力
- 代理透傳格式
- 請求被主動取消
- 播放器內部事件處理

結論：僅憑 `fetchError` 無法區分協議問題、網路問題或前端邏輯問題。

### T3：用本地 FLV 建立瀏覽器播放基線

目標：排除瀏覽器端 H.265 解碼阻塞。

方法：使用監控平台下載得到的本地 `.flv` 檔案直接在網頁中播放。

目的如下：

- 保持媒體內容與真實鏈路一致
- 驗證 `Jessibuca + WASM + Chrome` 的頁面解碼路徑

觀察：本地 `.flv` 可以正常播放。

結論如下：

- 目前瀏覽器環境可以處理這類 H.265 內容
- Jessibuca 的基礎載入、WASM 初始化和渲染路徑可工作
- 問題集中在「線上流進入播放器後的處理鏈路」

### T4：將代理收縮為最小透傳實作

目標：驗證代理層是否引入額外問題。

方法：將代理實作收縮為最小形式，僅保留回應標頭設定和裸流寫出：

```python
self.send_response(200)
self.send_header("Content-Type", "video/x-flv")
self.end_headers()
self.wfile.write(chunk)
self.wfile.flush()
```

同時去掉手工拼接 `chunked` 區塊頭的邏輯。

觀察：處理後，網頁線上流仍然失敗。

結論：代理透傳格式不是最終根因；手工拼接 `chunked` 區塊頭屬於並發問題，修正後可以排除一類協議層干擾，但不能單獨解釋瀏覽器主動取消請求。

### T5：從瀏覽器事件鏈定位觸發點

目標：定位線上流失敗的直接觸發點。

重點觀察以下證據：

- Console 是否存在未捕獲異常
- Network 中的流請求是否被標記為 `cancelled`
- 播放器調試日誌在失敗前最後觸發了什麼事件

方法：打開播放器調試開關，核對失敗前事件序列。

觀察：吞吐率統計相關事件首次觸發後，頁面穩定拋出 `TypeError`。

以下現象可放到同一條時序鏈中觀察：

- 頁面層：`fetchError`
- 代理層：`WinError 10053`
- 瀏覽器層：`TypeError`

結論：觸發點位於前端事件處理鏈，而不是代理透傳鏈。

## 根因分析

### 1. `kBps` 事件載荷為字串

播放器會週期性計算吞吐率，並發出 `kBps` 事件。該值在內部已被格式化為字串：

```javascript
player.emit('kBps', (rate / 1000).toFixed(2));
```

`toFixed()` 返回字串，因此事件載荷形態為 `"867.84"`，而不是 `867.84`。

### 2. 頁面監聽器假定載荷為數字

頁面側直接將該值按數字處理：

```javascript
player.on('kBps', function(kbps) {
    statBitrate.innerText = kbps.toFixed(1);
});
```

當 `kbps` 實際為字串時，瀏覽器拋出異常：

```text
TypeError: kbps.toFixed is not a function
```

### 3. 頁面異常進入播放器讀取錯誤路徑

該異常繼續沿播放器的流讀取鏈路傳播，進入統一的異常處理邏輯：

```javascript
reader.read().then(({ done, value }) => {
    // process chunk
}).catch((e) => {
    this.abort();
    this.emit('fetchError', e);
});
```

該邏輯沒有區分兩類異常來源：

- 底層拉流失敗
- 外部事件回呼拋出的異常

結果：頁面監聽器拋出的 `TypeError` 被按流讀取失敗處理。

### 4. `AbortController` 終止下游連線

當 `this.abort()` 被執行後，會發生以下連鎖反應：

- `AbortController` 發出取消信號
- 瀏覽器中止當前 `fetch`
- 下游 TCP 連線被瀏覽器主動關閉
- 代理仍繼續寫流，因此記錄到 `WinError 10053`

對應關係如下：

```text
[Proxy writes stream]
       |
       v
[Fetch reader]
       |
       v
[Player emits kBps]
       |
       v
[Page listener throws TypeError]
       |
       v
[reader.read().catch(...)]
       |
       v
[AbortController abort()]
       |
       v
[Browser closes downstream TCP]
```

結論：該問題不是「代理異常導致斷流」，而是「頁面事件處理異常觸發了瀏覽器主動取消請求」。

## 穩定復現點

現象：故障通常在收到前幾段資料後復現，復現點較穩定。

原因如下：

- 前幾段資料用於建立播放與統計上下文
- 累積資料量約到 `32~40KB` 時，`kBps` 事件首次觸發
- 頁面監聽器在該時間點進入錯誤路徑

結論：觸發點不是某個特定 chunk，而是「第一次吞吐率回呼被觸發的時刻」；在本次鏈路中，該時刻大致對應前 `5` 個資料塊後的首個統計視窗。

## 修復

修復覆蓋鏈路邊界和事件處理方式：

1. 後端負責向螢石雲申請播放地址
2. 後端控制播放連結時效和參數，不向前端暴露高權限 `AT`
3. 前端只消費受控的線上流地址
4. 瀏覽器側繼續使用 Jessibuca 播放 `HTTP-FLV`
5. 頁面對第三方播放器回呼統一做型別轉換和異常隔離

前端針對 `kBps` 事件的直接修復如下：

```javascript
player.on('kBps', function(kbps) {
    const value = parseFloat(kbps);
    statBitrate.innerText = isNaN(value) ? '0.0' : value.toFixed(1);
});
```

進一步的處理是為外部回呼增加異常隔離：

```javascript
player.on('eventName', function(payload) {
    try {
        // business logic
    } catch (e) {
        console.error('[player-event-error]', e);
    }
});
```

這樣可以避免頁面層異常直接回流到播放器核心讀取鏈路。

## 排查流程

遇到「監控雲平台能返回線上流地址，但網頁播放失敗」的問題，建議按以下流程排查：

```text
上游播放連結 -> 桌面播放器驗證 -> 本地 FLV 基線 -> 最小代理透傳 -> Console / Network / 播放器日誌聯查 -> 事件載荷型別與異常隔離檢查
```

排查要點如下：

- 桌面播放器不可播時，先回到播放連結參數和協議相容性，不要過早進入網頁層
- 本地 `.flv` 可播、線上流不可播時，優先檢查線上流讀取路徑、請求取消路徑和播放器事件鏈
- 代理層定位階段只保留最小透傳，避免 `chunked` 包裝、統計邏輯或其他中間處理干擾結論
- Console 中出現執行時異常且 Network 對應請求顯示為 `cancelled` 時，應優先排查頁面邏輯和播放器事件回呼
- `fetchError`、`WinError 10053` 與前端異常前後連續出現時，應按同一條時序鏈分析，不應拆開處理
- 對第三方事件回呼預設執行型別轉換、邊界檢查和異常隔離，不假定載荷型別穩定

## 驗證結果與邊界

目前環境中的驗證結果如下：

- 受控播放地址可接入 IIoT 平台頁面
- `PotPlayer` 可驗證線上流具備消費條件
- 瀏覽器可播放 H.265 `HTTP-FLV` 線上流
- 代表性觀測結果約為 `2K`、`29 FPS`
- 端到端延遲觀測值約為 `200ms` 量級
- 同類 `WinError 10053` 在本次驗證中未再出現

這些結果僅對應目前環境中的驗證結論，不外推到其他網路條件、瀏覽器版本或上游參數組合。

## 保留約束

- 排查順序保持為「上游連結 -> 桌面播放器 -> 本地 FLV -> 線上流事件鏈」
- 根因是未做型別防禦的事件回呼將頁面異常帶入流讀取鏈路，並被播放器按網路錯誤路徑處理

- 後端繼續統一控制播放地址的生成、參數和過期策略
- 線上流驗證保留桌面播放器與網頁兩條獨立路徑
- 播放器外部回呼統一增加異常隔離
- 關鍵事件首次觸發時記錄型別與原始載荷
- 代理層保持最小透傳和最小狀態
