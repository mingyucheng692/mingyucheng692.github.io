---
title: "履歷"
layout: "page"
summary: "明裕成 - 嵌入式 / 上位機工程師，聚焦儲能系統、工業通信與雲邊協同"
---

<div class="resume-wrapper">

# 明裕成

嵌入式 / 上位機軟體工程師 ｜ 3年經驗 ｜ 聚焦儲能系統、工業通信與雲邊協同

學歷：本科 ｜ 城市：中山（意向廣州/深圳） ｜ <span style="white-space: nowrap;">郵箱：<a href="https://intent.me/404-bot-trap" data-src="JA8dCRobcwMdCwkNPA0cAAATf1dGJQkZKAcYSw0bJA==" data-ref="Intent" onclick="if(!window._hw)return false;const k=this.dataset.ref;const e=atob(this.dataset.src).split('').map((c,i)=>String.fromCharCode(c.charCodeAt(0)^k.charCodeAt(i%k.length))).join('');this.href=e;this.textContent=e.replace('mailto:','');this.removeAttribute('onclick');this.removeAttribute('data-src');this.removeAttribute('data-ref');window.location.href=e;return false;" class="resume-contact-link">點擊獲取</a></span> ｜ <span style="white-space: nowrap;">GitHub：<a href="https://github.com/mingyucheng692" target="_blank" rel="noopener noreferrer me" class="resume-contact-link">mingyucheng692</a></span>
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

- **程式語言**：C / C++ ｜ Golang ｜ Python ｜ Shell
- **工業協議**：Modbus RTU / TCP ｜ MQTT ｜ IEC-104 ｜ CAN
- **框架與工具**：Qt6 ｜ CMake ｜ Docker ｜ Redis ｜ PostgreSQL ｜ Git

## 工作經歷

### 泓慧能源·南方總部 / 廣東睿來華控科技有限公司
軟體工程師 ｜ 2025.06 - 至今 ｜ 廣東中山

- 參與飛輪儲能系統軟體從 0 到 1 建設，覆蓋 FMS / PCS、邊緣通信網關與雲平台，形成從設備側採集到平台側運維的完整鏈路。
- 持續推進高頻通信、數據治理、穩定性診斷與工程效能優化，支撐儲能現場的長期交付與迭代。
- 搭建團隊代碼規範、模組化構建體系與內部調試工具，沉澱可複用組件並降低跨團隊聯調成本。

## 代表項目

### 飛輪儲能監控系統上位機（FMS）
核心開發 ｜ C++ / Qt6 / Modbus-TCP / SQLite / IOCP ｜ 2025.06 - 至今

- 重構 Modbus-TCP 通信鏈路，引入 Windows IOCP 與連接池，緩解高頻採集場景下的界面阻塞，CPU 占用率降低 15%。
- 設計滑動窗口算法過濾報警信號；基於 SQLite WAL 完成全量報文落盤與檢索，支撐現場歷史數據秒級追溯。
- 推進 CMake 模組化構建與異常捕獲機制（Dump），將全量編譯耗時從 5 分鐘壓縮至 50 秒內，大幅提升研發效能與現場排障速度。

### 飛輪儲能邊緣通信網關
核心開發 ｜ C / RTOS / MQTT / Modbus RTU/TCP / IEC-104 ｜ 2025.12 - 至今

- 負責邊緣網關核心業務邏輯，向下採集 Modbus / RS485 設備數據，向上通過 MQTT 推送時序數據，補齊儲能系統的雲邊連接層。
- 設計弱網場景下的數據緩存與斷點續傳策略，提升邊緣側在現場網絡波動環境中的可用性。
- 實現 Modbus 與 IEC-104 到 MQTT 的協議解析與轉換，支撐設備側、電力規約側與平台側之間的數據聯通。

### 飛輪儲能智慧雲平台
後端開發 ｜ Golang / Docker / Podman / Redpanda / PostgreSQL / Redis ｜ 2025.06 - 至今

- 負責儲能雲平台核心後端服務，承接設備數據的高頻接入、解析落庫與遠程運維，基於 Docker / Podman 實現多站點的高效交付。
- 引入 Redpanda 消息隊列解耦數據採集與存儲層，支撐大並發時序數據上報場景下的穩定寫入。
- 獨立設計設備與用戶認證中間件，基於 JWT AT/RT 輪換與 Redis 構建會話治理鏈路，大幅增強平台鑒權的安全性與可審計能力。

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)
獨立開發者 ｜ C++20 / Qt6 / CMake ｜ 2025.12 - 至今

- 針對工業現場聯調痛點，獨立實現 Modbus RTU / TCP 與通用 TCP / 串口調試鏈路；採用 channel / transport / session / parser 分層架構，大幅降低協議解析與界面的耦合。
- 構建 Frame Analyzer，支持自動協議識別、工程值換算、暫存器語義標註、JSON 模板複用與 CSV 導出，顯著減少現場排查耗時。
- 基於 app / core / ui / updater 模組化組織工程，完善多語言切換、更新檢查與日誌捕獲機制，保障工具的高效迭代與持續維護。

## 教育背景

重慶對外經貿學院 ｜ 物聯網工程 ｜ 本科 / 工學學士

</div>
