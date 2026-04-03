---
title: "Modbus-Tools 深度解析：高效輕量的工業協議調試利器"
date: 2026-02-28T12:00:00+08:00
tags: ["Modbus", "Qt6", "C++", "Open Source", "Tools"]
categories: ["Tech", "Project"]
summary: "面向嵌入式開發與現場聯調的輕量級 Modbus 工具。本文以實作流程為主，重點介紹如何快速發幀、查看並複製日誌，以及使用幀解析器完成倍率換算、寄存器註解與 JSON / CSV 配置持久化。"
url: "/zh-tw/blog/tech/modbus-tools-intro/"
---

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)
**C++20 | Qt6 | Industrial Protocols**

在工業自動化與嵌入式開發中，Modbus 幾乎是日常的一部分。真正進入現場聯調後，時間通常花在重複操作：手動組幀、反覆核對日誌、逐筆做倍率換算。

開發 Modbus-Tools 的目標很明確：把「發幀、看日誌、解幀」三件高頻操作做順手。它不追求功能堆疊，而是以調試效率和可用性為優先。實作上採用 channel / transport / session / parser 分層設計，並配套 CI/CD、多語言與自動更新能力，方便工具持續迭代。

以下依照實際使用順序，快速介紹核心操作方式。

---

### 1. 快速連線與報文建立

打開軟體後，不論使用 **Modbus RTU**（串口）或 **Modbus TCP**（網路），連線參數都集中在同一區塊，設定很直觀。

![Modbus-TCP快速建幀](/images/blog/modbus-tools/modbus-tcp-frame-builder.png)

在聯調時，最需要的是快速驗證。Modbus-Tools 將參數與功能碼拆分得清楚易用：
- **一鍵發送常用功能碼**：填入 `從站地址 (Slave ID)`、`起始地址`、`數量/資料`，點選 `01/03/06/10` 等常用按鈕即可發送。
- **自動組幀與校驗**：底層自動計算 CRC/LRC，降低手動計算與輸入錯誤。
- **寫入格式可切換（HEX / DEC）**：在 `Write Data` 旁透過 `Format` 下拉選單切換 `Hex` 或 `Decimal`，可依專案習慣輸入。
- **Raw 模式**：需要發送自定義 Hex 報文時，可直接切換 Raw 輸入，適合非標流程測試。

![Modbus-TCP快速下發-DEC](/images/blog/modbus-tools/modbus-tcp-write-decimal.png)

---

### 2. 看日誌與一鍵複製 (Traffic Monitor)

問題定位通常從日誌開始。Traffic Monitor 的設計重點是可讀性與可分享性。

- **TX/RX 分離顯示**：發送與接收資料分色呈現，並附毫秒級時間戳，時序關係更清楚。
- **一鍵複製**：可快速複製關鍵報文，用於問題回報、團隊溝通與測試記錄。
- **方向過濾**：支援僅顯示 TX 或 RX，面對高頻輪詢時更容易聚焦。
- **日誌可匯出保存**：可將當前通訊記錄保存，方便現場問題留痕與測試歸檔。

---

### 3. 幀解析器 (Frame Analyzer)：把 Hex 變成可讀資料

Frame Analyzer 是日常使用頻率很高的功能。將報文貼上後點擊解析，即可看到結構化欄位與可讀表格。

![幀解析助手主頁](/images/blog/modbus-tools/frame-analyzer-overview.png)

在現場最實用的是以下幾個能力：

#### 解碼模式切換 (Unsigned / Signed)
解析器工具列提供 `Decode Mode`，可在 `Unsigned` 與 `Signed` 間切換。切換後會立即刷新解析結果，十進位、Hex、二進位與換算值都會同步更新，查看有符號量更直觀。

#### 倍率換算 (Multiplier Scaling)
許多設備會將浮點值放大後再傳輸（例如 `220.5V -> 2205`）。  
在表格中可針對寄存器設定 `Scale`（如 `0.1`、`0.01`），系統會即時顯示換算後的工程值。

#### 寄存器功能註解 (Description)
可為寄存器地址補上說明，例如「A 相電壓」「電機轉速」。  
數值與語意同時呈現，減少來回查表時間。

#### 配置持久化與 JSON / CSV 模板
配置可保存並重複利用。
- **自動保存**：保留最近使用的倍率與註解。
- **JSON / CSV 匯入/匯出**：可依設備建立專屬模板，切換機型時直接套用。

#### 其他實用細節
- **Format Hex 按鈕**：可一鍵清理並規範 Hex 輸入格式，貼上長報文後更易閱讀。
- **回應起始位址可設定**：可透過 `Start Address` 配合回應幀解析，對齊寄存器點位表。
- **協議模式可選**：支援 `Auto Detect / Modbus TCP / Modbus RTU`，方便混合抓包場景快速判斷。

![幀解析地址倍率註解](/images/blog/modbus-tools/frame-analyzer-address-scale-description.png)

---

### 4. 補充工具

除 Modbus 主流程外，也內建兩個輕量模組，方便臨時調試：
- **TCP Client**：快速驗證自定義網路報文。
- **Serial Port**：基本串口收發測試（ASCII/Hex）。

---

### 結語

Modbus-Tools 的價值不在於功能數量，而在於把高頻操作做得穩定、直接、可複用。  
當發幀、看日誌、解幀更流暢，開發者就能把精力放回業務邏輯與問題根因分析。

若你正在做嵌入式或上位機開發，歡迎前往 [GitHub 倉庫](https://github.com/mingyucheng692/Modbus-Tools) 體驗並提出建議。
