---
title: "Modbus-Tools 深度解析：高效輕量的工業協議調試利器"
date: 2026-02-28T12:00:00+08:00
lastmod: 2026-04-21T12:00:00+08:00
tags: ["Modbus", "Qt6", "C++20", "Open Source", "Tools", "Industrial"]
categories: ["systems"]
summary: "面向嵌入式開發與現場聯調的輕量級 Modbus 工具。介紹快速發幀、日誌查看、幀解析（倍率換算、寄存器註解、JSON / CSV 配置持久化）、Link to Analyzer 即時聯動分析等功能的實際用法。"
url: "/zh-tw/blog/tech/modbus-tools-intro/"
---

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)
**C++20 | Qt6 | Industrial Protocols**

> **本文基於 Modbus-Tools 最新版本撰寫，更新於 2026-04-21**

在工業自動化與嵌入式開發中，Modbus 幾乎是日常的一部分。真正進入現場聯調後，時間通常花在重複操作：手動組幀、反覆核對日誌、逐筆做倍率換算。

開發 Modbus-Tools 的目標很明確：把「發幀、看日誌、解幀」三件高頻操作做順手。它不追求功能堆疊，而是以調試效率和可用性為優先。實作上採用 channel / transport / session / parser 分層設計，並配套 CI/CD、多語言與自動更新能力，方便工具持續迭代。

以下依照實際使用順序，快速介紹核心操作方式。

---

### 1. 快速連線與報文建立

打開軟體後，不論使用 **Modbus RTU**（串口）或 **Modbus TCP**（網路），連線參數都集中在左側面板，設定很直觀。

![Modbus-TCP快速建幀](/images/blog/modbus-tools/modbus-tcp-frame-builder.png)

在聯調時，最需要的是快速驗證。Modbus-Tools 將參數與功能碼拆分得清楚易用：
- **一鍵發送常用功能碼**：填入 `從站地址 (Slave ID)`、`起始地址`、`數量/資料`，點選 `01/02/03/04/05/06/0F/10` 等按鈕即可發送。底層自動組幫並計算 CRC/LRC。功能碼涵蓋 0x01–0x04（讀）、0x05–0x06（單寫）、0x0F–0x10（多寫）。
- **HEX / DEC 智慧識別**：`Slave ID` 與 `起始地址` 支援 HEX（如 `0x10`、`10H`）與 DEC（如 `16`）兩種格式輸入，由 `parseSmartInt()` 統一解析並做範圍校驗。
- **寫入格式可切換（HEX / DEC / Binary）**：在 `Write Data` 旁透過 `Format` 下拉選單切換 `Hex`、`Decimal` 或 `Binary`，可依專案習慣輸入。
- **Raw 模式增強**：需要發送自定義 Hex 報文時，可直接切換 Raw 輸入並發送，適合非標流程測試。Raw 模式內建兩個輔助按鈕：
  - **Append CRC (RTU)**：自動計算並追加 CRC16 校驗值至輸入框。
  - **Add MBAP (TCP)**：自動封裝 Modbus TCP 主站報頭（Transaction ID / Protocol ID / Length / Unit ID）至輸入框。

![Modbus-TCP快速下發-DEC](/images/blog/modbus-tools/modbus-tcp-write-decimal.png)

#### 線圈 (Coils) 二進位下發互動

針對位元操作場景，工具提供了直覺的 Binary 輸入模式：
- **Binary 輸入**：支援直接輸入位元串（如 `1 0 1 1`），系統自動編碼並配合 `0x05`（單線圈寫入）或 `0x0F`（多線圈寫入）功能碼下發。
- **位元級讀取**：配合 `0x01`/`0x02` 讀取指令，實現對遠端設備線圈與離散輸入狀態的高效驗證。

---

### 2. 看日誌與一鍵複製 (Traffic Monitor)

問題定位通常從日誌開始。Traffic Monitor 的設計重點是可讀性與可分享性。

- **TX/RX 分離顯示**：發送與接收資料分色呈現，並附毫秒級時間戳，時序關係更清楚。
- **一鍵複製**：可快速複製關鍵報文，用於問題回報、團隊溝通與測試記錄。
- **方向過濾**：支援僅顯示 TX 或 RX，面對高頻輪詢時更容易聚焦。
- **日誌可匯出保存**：可將當前通訊記錄保存，方便現場問題留痕與測試歸檔。

---

### 3. 幀解析器 (Frame Analyzer)：把 Hex 變成可讀資料

Frame Analyzer 是日常使用頻率很高的模組。將 Hex 報文貼上後點擊解析，即可看到結構化欄位與可讀表格。

![建幀 → 複製 → 貼上解析流程](/images/blog/modbus-tools/demo.gif)

![幀解析助手主頁](/images/blog/modbus-tools/frame-analyzer-overview.png)

在現場最實用的是以下幾個能力：

#### 解碼模式切換 (Unsigned / Signed)
解析器工具列提供 `Decode Mode`，可在 `Unsigned` 與 `Signed` 間切換。切換後會立即刷新解析結果，十進位、Hex、二進位與換算值都會同步更新，查看有符號量更直觀。

#### 倍率換算 (Multiplier Scaling)
許多設備會將浮點值放大後再傳輸（例如 `220.5V -> 2205`）。  
在表格中可針對寄存器設定 `Scale`（如 `0.1`、`0.01`），系統會即時顯示換算後的工程值。

#### 多維位元序分析 (Byte Order)
不同廠商的 PLC 和儀表可能採用不同的資料排列方式。解析器支援 **四種位元組/字序模式**：
- **ABCD (Big Endian)**：大端模式，高位在前。
- **CDAB (Little Endian Byte Swap)**：小端位元組交換。
- **BADC (Big Endian Byte Swap)**：大端位元組交換。
- **DCBA (Little Endian)**：小端模式，低位在前。

切換位元序後，寄存器值會重新計算並顯示。

#### 寄存器功能註解 (Description)
可為寄存器地址補上說明，例如「A 相電壓」「電機轉速」。  
數值與語意同時呈現，減少來回查表時間。

![幀解析-地址倍率註解](/images/blog/modbus-tools/frame-analyzer-address-scale-description.png)

#### 配置持久化與 JSON / CSV 模板
配置可保存並重複利用。
- **自動保存**：保留最近使用的倍率與註解。
- **JSON / CSV 匯入/匯出**：可依設備建立專屬模板，切換機型時直接套用。

#### 其他實用細節
- **Format Hex 按鈕**：可一鍵清理並規範 Hex 輸入格式，貼上長報文後更易閱讀。
- **回應起始位址可設定**：可透過 `Start Address` 配合回應幀解析，對齊寄存器點位表。
- **協議模式可選**：支援 `Auto Detect / Modbus TCP / Modbus RTU`，方便混合抓包場景快速判斷。

#### 強制解析 (Force Parse)

現場抓包時，報文可能因截斷、中間設備修改等原因導致校驗不通過。當使用者在 Protocol 下拉選單中**手動指定 TCP 或 RTU**（而非 Auto Detect）時，解析器進入強制模式：

- **RTU 強制模式**：CRC 不匹配時不再直接報錯終止，而是標記 `checksumValid = false` 並在 warnings 中記錄 `"CRC Mismatch (Forced)"`，同時繼續解析 PDU 資料欄位。
- **TCP 強制模式**：MBAP length 欄位異常或幀尾有多餘位元組時，記錄對應 warning 後仍按實際位元組長度提取 PDU。

此機制適用於分析被閘道器/中繼修改過、或從串口抓包工具截取的不完整報文。Auto Detect 模式下校驗嚴格，適合正常通訊場景的精確驗證。

#### Link to Analyzer (即時聯動)

除手動貼上 Hex 報文外，Frame Analyzer 也支援從 Traffic Monitor 接收即時資料：

- **自動推送**：在 Modbus TCP / RTU 視圖中開啟 Linkage 開關後，RX 回應報文的 PDU 會自動送入解析器，無需手動複製。
- **暫停 / 恢復**：點擊 `Pause Refresh` 可駐留當前幀，方便編輯 Scale 或 Description；再次點擊 `Resume Refresh` 恢復自動重新整理。
- **停止聯動**：點擊 `Stop Link` 中斷資料流，解析器恢復為手動模式。
- **非同步執行**：解析邏輯運行於 `QThread` 背景執行緒，不阻塞 Traffic Monitor 的列表捲動。

如果你同時維護多種型號設備，依設備或專案儲存獨立模板後，切換調試物件時可直接匯入，減少重複配置。

---

### 4. 補充工具

除 Modbus 主流程外，也內建兩個輕量模組，方便臨時調試：
- **TCP Client**：快速驗證自定義網路報文。
- **Serial Port**：基本串口收發測試（ASCII/Hex）。

---

### 5. 工程品質與測試保障

作為持續迭代的開源專案，Modbus-Tools 在程式碼品質方面投入了相當精力：

#### 自動化測試
專案採用 **Google Test (GTest)** 與 **Google Mock (GMock)** 框架進行自動化品質檢測，覆蓋以下核心模組：
- **會話管理**：連線/斷線邏輯、請求逾時重試及異常狀態恢復。
- **協議傳輸**：TCP/RTU 報文封裝/解包、校驗和計算及完整性驗證。
- **解析邏輯**：針對多種有效指令及畸形報文的魯棒性驗證。
- **資料處理**：位元序轉換、工程量縮放及格式化演算法的計算準確性。

目前全量 **42 個自動化測試案例**（`TEST` + `TEST_F`），覆蓋會話管理、協議傳輸、解析邏輯、資料處理及格式化等模組，每次 Release 發布均執行迴歸測試。

#### CI/CD 整合
- GitHub Actions 流水線整合 **MSVC AddressSanitizer (ASan)**，用於記憶體損壞與洩漏的自動化監測。
- 支援自動建置、測試及 Release 解析包分發。

#### 自動更新 (OTA)
工具整合了基於 GitHub Releases 的自動更新機制：啟動時靜默偵測新版本 + 功能表列手動檢查。更新包透過 SHA256 校驗後執行替換，支援 UpdateOnly（增量）和 Full Package 兩種模式。

---

### 結語

Modbus-Tools 的設計目標是將高頻調試操作（發幀、看日誌、解幀）封裝為可複用的工作流，使開發者能更專注於業務邏輯與問題定位。

**功能概要**：
- **快速組幀**：HEX/DEC 智慧識別（`parseSmartInt`）+ Raw 模式 CRC/MBAP 輔助計算。
- **即時聯動**：Link to Analyzer 支援 RX 報文自動推送至解析器。
- **深度分析**：倍率換算（Scale Factor）+ 四種位元序（ABCD/BADC/CDAB/DCBA）+ 寄存器描述。
- **位元級控制**：線圈 Binary 輸入模式，支援 0x05/0x0F 功能碼。
- **品質保障**：42 個自動化測試 + CI/CD 整合 MSVC AddressSanitizer。

若你正在做嵌入式或上位機開發，歡迎前往 [GitHub 倉庫](https://github.com/mingyucheng692/Modbus-Tools) 體驗並提出建議。
