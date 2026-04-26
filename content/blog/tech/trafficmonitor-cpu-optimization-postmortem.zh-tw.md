---
title: "TrafficMonitor CPU 佔用優化復盤：從 4% 降到 0.6% 的工程化收縮"
date: 2026-04-26T10:00:00+08:00
draft: false
tags: ["TrafficMonitor", "Qt", "C++", "Modbus", "Performance", "CPU", "Postmortem"]
categories: ["Tech", "Engineering"]
summary: "記錄一次高頻輪詢場景下的 CPU 優化復盤：透過收縮 UI 日誌鏈路、刪除高頻狀態機、改造執行緒等待模型，將程序 CPU 使用率從 3.6%~4.2% 降到 0.1%~0.6%。"
url: "/zh-tw/blog/tech/trafficmonitor-cpu-optimization-postmortem/"
---

在一次 `100ms` 輪詢場景的效能排查中，`TrafficMonitor` 所在程序的 CPU 使用率長期穩定在 `3.6%~4.2%`。除資源佔用偏高外，介面在高頻收發期間也會出現可感知的刷新遲滯與滾動不跟手。對比兩個版本實作後，優化版本最終將該指標降到 `0.1%~0.6%`，同時高頻輪詢下的 UI 卡頓現象也明顯緩解。

這次收益並非來自單點熱點函式優化，而是一次典型的工程化收縮：減少高頻路徑上的中間層、狀態機與無效喚醒，讓系統回到更符合其職責邊界的實作方式。

本文復盤的優化對象來自開源工具 [Modbus-Tools](/zh-tw/blog/tech/modbus-tools-intro/)。如果想先了解工具的整體定位與核心能力，可先閱讀該介紹文章。

---

## 背景與結論

這次優化的結論可以概括為兩點：

- `TrafficMonitor` 回歸輕量流量展示元件，不再承擔複雜日誌系統職責
- 工作執行緒在無事件時真正阻塞，而不是以固定時間片持續輪詢

對應到程式碼層，主要改動集中在四個方向：

- `TrafficMonitorWidget` 從事件模型回退到直接文字追加
- `ModbusTcpView` 與 `ModbusRtuView` 刪除輪詢彙總與抑制邏輯
- `ModbusClient` 從 `wait_for + 5ms slice` 改為 `wait_until`
- `SerialChannel` 與 `TcpChannel` 移除高頻路徑上的執行緒診斷日誌

---

## 關鍵改動

### 1. UI 日誌鏈路收縮

複雜版本中，`TrafficMonitor` 引入了完整事件抽象，日誌寫入路徑為：

`調用方 -> 構造事件物件 -> 元件判定級別/模式 -> 渲染文字 -> 寫入列表`

優化版本刪除這套中間層，保留 `appendTx()`、`appendRx()`、`appendInfo()`、`appendWarning()` 等直接入口，路徑收縮為：

`調用方 -> 直接格式化文字 -> 追加到 QListWidget`

這一步減少了物件構造、事件分類、二次渲染與狀態同步開銷，是本次 CPU 下降的主要來源之一。

### 2. 元件實作從 Model/View 回退到直接列表

複雜版本的 `TrafficMonitorWidget` 不只是顯示控制項，還維護了完整的資料層，包括：

- `QListView + QAbstractListModel`
- `pendingEvents_`
- `eventHistory_`
- `flushTimer_`
- `rebuildScheduled_`
- `pausedEventCount_`

這類設計在功能上更完整，但在 `100ms` 高頻刷新下，會持續放大以下成本：

- 事件入隊與批量刷新
- 文字格式化與過濾判斷
- 歷史重建與滾動同步

優化版本改回 `QListWidget` 直接追加，本質上是刪除一層常駐的資料管理系統。對高頻日誌場景，這種收縮帶來的收益非常直接。

### 3. 刪除高頻附加能力與輪詢狀態機

複雜版本圍繞「更強可觀察性」增加了多項能力，包括：

- Pause View
- Raw Frames
- Level Filter
- 暫停計數
- 可見列表重建
- Poll Summary 彙總狀態機

這些能力單獨看都合理，但在高頻場景下，會把每條日誌都變成「需要判斷和調度的事件」，而不是「直接展示的文字」。

尤其是 `ModbusTcpView` 與 `ModbusRtuView` 中的 Poll Summary 邏輯，雖然減少了可見日誌條數，但並未減少系統總工作量。每次輪詢仍需更新計數、維護狀態、判斷窗口並格式化摘要。優化版本刪除這套邏輯後，鏈路明顯縮短。

### 4. 等待模型改為真正阻塞

除 UI 外，另一處關鍵改動在 `ModbusClient` 的等待策略。

複雜版本使用 `cv_.wait_for(lock, 5ms, ...)` 配合事件泵機制，意味著執行緒即使沒有真實事件，也會以 `5ms` 週期被喚醒一次。對 `100ms` 輪詢場景，這會製造大量無效 wakeup。

優化版本統一改為：

`cv_.wait_until(lock, deadline, predicate)`

改造後的效果是：

- 無事件時執行緒真正休眠
- 僅在超時或條件滿足時喚醒
- 不再週期性處理當前執行緒事件

這部分改動直接降低了工作執行緒空轉，是本次優化的另一主要來源。

### 5. 清理高頻調試輸出

`SerialChannel.cpp` 和 `TcpChannel.cpp` 中，複雜版本保留了執行緒上下文診斷邏輯，例如：

- `threadToken(...)`
- `logThreadContextOnce(...)`
- `open()`、`onReadyRead()`、`onConnected()` 中的執行緒日誌

單條日誌成本不高，但它們位於高頻收發路徑，累積後會放大格式化與分支判斷開銷。優化版本將其移除，是一次典型的高頻路徑減負。

---

## 為什麼收益這麼明顯

這次優化的價值，不在於把某個函式從 `10ms` 優化到 `2ms`，而在於刪除了高頻鏈路上的多項乘法項。

複雜版本的一次輪詢收發，通常會經過：

- IO 回調
- 事件物件構造
- 級別與模式判斷
- 佇列操作
- 定時器喚醒
- 批量渲染
- 過濾判斷
- 歷史重建
- 自動滾動

優化版本則更接近：

- IO 回調
- 文字格式化
- 直接追加到列表

當調用頻率固定為 `100ms` 時，鏈路長度差異會持續放大，最終直接反映為 CPU 使用率從 `3.6%~4.2%` 降到 `0.1%~0.6%`。由於主執行緒不再持續承受事件分發、批量刷新與列表重建壓力，介面互動也恢復到更穩定的狀態。

---

## 工程經驗

這次復盤沉澱出幾條比較明確的工程經驗：

- 高頻場景下，功能複雜度本身就是效能成本
- GUI 效能問題很多時候不在繪製，而在繪製前的事件系統與狀態機
- 對等待模型來說，能阻塞就不要輪詢，能按條件喚醒就不要按時間片喚醒
- 日誌元件職責越清晰，實作越容易穩定，效能也越可控

從結果看，這次優化並不是「做了很多技巧性調優」，而是重新收縮了系統邊界。當 `TrafficMonitor` 回歸輕量視圖、`ModbusClient` 改回真正阻塞等待後，系統不再為低頻附加能力支付高頻成本，整體負載自然回落。

如需進一步了解工具能力全貌，可參考 [Modbus-Tools 介紹文章](/zh-tw/blog/tech/modbus-tools-intro/)。專案原始碼見 GitHub 倉庫：[mingyucheng692/Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)。
