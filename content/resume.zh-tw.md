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
▸ 所在公司為泓慧能源（飛輪儲能頭部企業）全資子公司，承擔南方研發與量產基地核心系統開發

- 參與南方基地儲能軟體體系（端-邊-雲）從 0 到 1 建設，負責銜接集團技術規範，交付的軟體模組直接服務於本地产線設備的整機測試與現場出廠交付。
- 深度對接硬體與電氣規約團隊，解決複雜工業現場的通信擁塞與信號抖動等痛點，保障底層電力電子設備與上層系統的可靠交互。
- 協助推行團隊 Git 協同與代碼審查機制；針對跨部門設備聯調耗時長的痛點，主導開發多套通用聯調工具鏈，以可視化解析取代手工查表與拼接，顯著縮短現場排障週期。

## 代表項目

### 飛輪儲能監控系統上位機（FMS）
核心開發 ｜ C++ / Qt6 / Modbus-TCP / CAN / SQLite / IOCP ｜ 2025.06 - 至今

- 重構 Modbus-TCP 通信鏈路，引入 Windows IOCP 與連接池機制，解決高頻數據採集導致的界面阻塞問題，CPU 占用率降低約 15%。
- 開發底層 DSP 控制板專屬調試模組（基於 ZLG CAN SDK），實現 IEEE 754 浮點數與 HEX 報文的雙向動態解析，支援多位元組序（CDAB/ABCD）自動轉換與暫存器語義映射，替代 CANTest 手動抓包換算流程，顯著提升軟硬體聯調效率。
- 設計滑動窗口算法實現報警信號防抖；基於 SQLite WAL 模式實現高頻報文落盤，支援現場歷史數據的高效本地檢索與追溯。
- 推進 CMake 模組化構建，並整合 Crash Dump 異常捕獲機制，將全量編譯耗時從 5 分鐘壓縮至 50 秒內，顯著提升日常開發迭代與現場排障效率。

### 飛輪儲能邊緣通信網關
核心開發 ｜ C / STM32F407 / FreeRTOS / MQTT / Modbus / IEC-104 ｜ 2025.12 - 至今

- 負責基於 STM32F407 與 FreeRTOS 的邊緣網關核心業務邏輯，向下通過 Modbus/RS485 輪詢底層設備，向上通過 MQTT 建立與雲端的時序數據通道。
- 實現 Modbus、IEC-104 與 MQTT 之間的協議解析與映射轉換，打通設備端、電力規約側與平台側的數據交互閉環。
- 針對現場弱網工況，設計並實現時序數據緩存與斷點續傳機制，保障網絡波動下的數據完整性。

### 飛輪儲能智慧雲平台
後端開發 ｜ Golang / Docker / Podman / Redpanda / PostgreSQL / Redis ｜ 2025.06 - 至今

- 參與儲能雲平台（基於 Alibaba Cloud Linux）核心後端服務開發，承接設備數據的高頻接入、協議解析與時序入庫。
- 主導部署環境配置，採用 Docker 兼顧本地開發，生產環境應用 Podman rootless 結合 Shell 腳本，實現 Rootless 帳號的安全隔離部署。
- 引入 Redpanda 消息隊列解耦數據採集與存儲層，平抑並發寫入峰值；獨立設計認證中間件，基於 JWT (AT/RT) 與 Redis 維護安全的會話狀態。

### [Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)（個人開源項目）
獨立開發者 ｜ C++20 / Qt6 / CMake / CI-CD ｜ 2025.12 - 至今

- 採用 channel / transport / session / parser 分層架構研發跨平台調試工具，內建可視化報文構建器，徹底免除繁瑣的手動查表與十六進制幀拼接。
- 開發 Frame Analyzer 核心解析器，支援協議自動識別、自定義倍率換算與暫存器語義標註；支援 JSON / CSV 配置導入導出，將現場排障耗時從分鐘級壓縮至秒級，聯調提效 5 倍以上。
- 基於 GitHub Actions 跑通全自動 CI/CD 流水線，實現代碼推送後的自動構建與 Release 發布；客戶端內建多語言切換與自動更新（Auto-Updater）機制，保障工具在現場的敏捷迭代。

## 教育背景

重慶對外經貿學院 ｜ 物聯網工程 ｜ 本科 / 工學學士

</div>
