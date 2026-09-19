---
title: "Linux 發佈 CI/CD 多輪收斂復盤：「能跑」是表象，不是證據"
date: 2026-09-18T12:00:00+08:00
draft: false
tags: ["C++", "Qt", "Linux", "Windows", "Industrial", "CI/CD", "Multithreading", "Postmortem"]
categories: ["industrial-software"]
summary: "開源工業除錯工具 Modbus-Tools（Qt 6 / C++）在建立 Linux 自動化發佈管線時，透過多輪迭代收斂出七類跨平台深坑。其中六類共享同一模式：「在別處能跑」往往只是特定環境下的局部表象，而非邏輯正確。本文從 GNU ld 符號剔除、epoll/Winsock 事件時序、Qt 雙 Socket 爭搶 fd 的未定義行為，到 ELF RUNPATH 契約切分，復盤一個高確定性發佈管線的建立過程。"
url: "/zh-tw/blog/tech/linux-release-ci-postmortem/"
---

> **TL;DR**：[Modbus-Tools](https://github.com/mingyucheng692/Modbus-Tools)（Qt 6 / C++）的首條 Linux 自動化發佈管線多輪 CI 才收斂：七類故障中六類同根——**「在別處能跑」只是特定環境下的局部表象，不是正確性證據；跨平台交付必須依賴顯式契約，而不是隱式環境。** 最終雙平台全量測試用例穩定通過，自包含打包流程端到端閉環，跨平台發佈資產自動化建置就緒。

---

## 一條從未端到端閉環的發佈流水線

Modbus-Tools 的發佈管線由 `v*` tag 觸發 GitHub Actions，Windows 與 Linux 雙平台並行。Linux 側跑在 `ubuntu-latest` 宿主機啟動的 `ubuntu:22.04` 容器裡（鎖定 glibc 2.35 作為 ABI 基線），用 aqtinstall 裝 Qt 6.10.3（gcc_64 / Ninja），先以 `RelWithDebInfo` + ASan + `offscreen` 執行全量單元測試套件，再將純淨 `Release` 建置產物交給部署腳本 `deploy-linux.sh` 執行自包含打包與靜態校驗，生成獨立的 `.tar.gz` 發佈包。

同一時期，Windows 側 QA、打包、發佈一路順利通過，4 項資產順利上傳；Linux 側卻從未實現端到端完整閉環。發佈預演前後跑了多輪 CI 才收斂，且阻塞點呈現嚴格的時序級聯：管線作為串行長鏈路，前序階段一旦中斷，後續邏輯根本沒有執行機會——每修復一個阻斷點，下一層的缺陷才第一次真正執行。排查過程如同剝洋蔥，而每剝一層都需要消耗一次完整的 CI 週期。

先排除兩個基礎設施初始化階段的一次性阻塞項：GitHub 官方 action 內嵌的 CPython 基於 glibc 2.38 動態連結，在 glibc 2.35 的容器內因缺失動態符號直接崩潰（`GLIBC_2.38 not found`）；Qt 官方二進位依賴的 `libdbus-1-dev` 在容器映像檔中未預裝。解決上述兩個環境阻斷項後，剩下的七類故障才是本文的主體。事後審視，其中六類都在回答同一個問題：它們此前在 Windows CI 或本地開發機上，為何能一路通過？

---

## 為什麼此前在 Windows CI 與本地能通過？

對前六處故障此前能夠「順利放行」的成因逐一溯源，每一處都依賴了特定環境的隱式容錯：

| 故障類別 | 故障表象 | 此前環境（Windows CI / 本地）放行的真實成因 |
| --- | --- | --- |
| i18n 資源遺失 | 5 例 `QTranslator::load` 回傳 false | **Windows CI (MSVC)** 預設保留帶 CRT 初始化段的靜態庫成員，GNU ld 的丟棄行為從未暴露 |
| Socket 錯誤被吞 | 告警日誌在，訊號斷言 false | **Windows CI** 下 `errorOccurred` 先於 `stateChanged` 到達，guard 放行；Linux 順序恰好反轉 |
| 迴路通訊死鎖 | 資料流不通，最小重現區段錯誤 | 雙引擎爭搶同一 fd 是未定義行為，**Windows CI** 僅僅容忍了重複 close 的報錯 |
| POSIX 移植陷阱 | Windows CI 編譯失敗 | 方向相反的一例：**Linux 本地**永遠有 `unistd.h`，缺陷只在 MSVC 編譯期顯形 |
| Qt 依賴收集落空 | Step 2 報 `no Qt libraries collected` | **Linux 開發機**的動態載入器（ldconfig 快取、環境變數）預設了 Qt 安裝前綴 |
| Wayland 假陽性 | Step 5 攔截 `libwayland-cursor` | **真實桌面環境**必有 Wayland 會話，驗證器把「CI 容器極簡」誤判為 bundle 缺陷 |

測試通過，只說明在那個特定的語意邊界下，缺陷恰好未被觸發。在跨平台工程中，「能夠執行」只是待驗證的假設，其成立的完備條件才是工程證據。

---

## 靜態庫連結語意：無引用符號導致嵌入資源被整塊剔除

5 例國際化測試斷言失敗，根源在於 `QTranslator::load` 回傳 false——Qt 資源系統 `:/i18n/` 和檔案系統兩條路徑都沒命中。同一套測試，Windows CI 全部通過，本地 Linux 穩定重現。

根因是靜態庫在兩種連結器下的語意差異：

```text
qrc 編譯產物 .o ：只含 Q_CONSTRUCTOR_FUNCTION 初始化器（執行時呼叫 qRegisterResourceData），
                  不定義任何可被外部引用的未解析符號。
GNU ld (Linux)  ：按傳統單遍掃描演算法，僅當 .o 能解析當前未決符號（Undefined Symbol）時才提取該成員 → 這些 .o 被整塊丟棄。
MSVC (Windows)  ：預設保留包含 CRT 初始化段（.CRT$XC*）的目標檔案 → 從未暴露。
```

根因驗證僅需一條命令，失敗時它沒有任何輸出（注：`strings` 查不到資源路徑，因為 rcc 生成的 qrc 內部路徑預設以 UTF-16 格式儲存）：

```bash
nm -C build_qa/tests/test_ui_widgets | grep -i qRegisterResourceData
```

在實施修復前，三個事實決定了技術方向。第一，波及面不僅限於測試：生產可執行檔同樣靜態連結 `libui.a`，若不徹底修復，Linux 發佈包的執行時多語言切換將靜默失效——單測失敗正是生產缺陷的前哨警報。第二，本地 7 敗與 CI 的 7 敗在數量和類別上嚴格對應，證明這是確定性的工具鏈與程式碼行為差異，而非 CI 容器的環境偶發抖動（Flaky Environment）。第三，拒絕平台條件分支：備選方案還有 OBJECT 庫、顯式 `Q_INIT_RESOURCE`、qrc 上移可執行目標，最終選中 CMake 3.24+ 原生的 `$<LINK_LIBRARY:WHOLE_ARCHIVE,ui>`——零侵入、單一語意同時覆蓋生產 app 與 3 個測試目標，MSVC 端自動映射為 `/WHOLEARCHIVE:ui.lib`，行為等價，零 `#ifdef`：

```cmake
target_link_libraries(Modbus-Tools PRIVATE $<LINK_LIBRARY:WHOLE_ARCHIVE,ui>)
```

---

## 事件順序不保證，fd 所有權必須唯一

### 瞬時狀態競爭導致錯誤事件被吞沒

日誌中早已留存確鑿證據：`[warning] socket error ... Connection refused` 已明確輸出，表明 `onSocketError` 回呼已被觸發，但測試斷言 `errorEmitted` 卻為 false——底層網路錯誤雖已發生，對外業務訊號卻未派發。用例執行耗時 2018ms，直至超時視窗耗盡仍未擷取到預期訊號。

根因在於 Winsock 與 epoll 將底層事件傳遞至 Qt 事件循環的時序存在根本差異：

* Windows (Winsock)：網路錯誤發生時，`errorOccurred` 通常先於 `stateChanged(UnconnectedState)` 到達。
* Linux (epoll)：連線被拒時，epoll 迅速回報錯誤事件，Qt 內部狀態先躍遷，`stateChanged(UnconnectedState)` 先將狀態機置為 `Closed`，`errorOccurred` 隨後才傳遞。

原程式碼的 guard 判定的是瞬時狀態：

```cpp
// 缺陷程式碼：依賴瞬時狀態判定，在 Linux 亂序下會吞錯
if (state_ != State::Opening && state_ != State::Open) {
    return; // 判定為非作用中連線的歷史噪音，直接吞噬
}
```

在 Linux (epoll) 架構下，錯誤訊號晚一步到達，此時狀態機已前置遷入 `Closed` 態，導致後續到達的連線拒絕事件被守衛邏輯作為「過期噪音」靜默丟棄。換言之，若 Linux 使用者嘗試連線一個不可達連接埠，UI 將陷入無回應的靜默斷開狀態——這是真實的產品缺陷，絕非測試斷言的設計偏差。

設計改進的關鍵在於重構狀態判定模型。守衛邏輯不應依賴瞬時連線狀態，而應仲裁「該錯誤是否屬於當前連線會話」。引入會話旗標 `socketDropIsError_`：連線發起或接管時置位，主動 close、超時或錯誤消費後重設。只要屬於本輪會話的未決異常，即便狀態機已先一步遷入 `Closed`，錯誤照常派發：

```cpp
// 修正邏輯：基於生命週期會話判定
if (state_ != State::Opening && state_ != State::Open) {
    if (state_ == State::Closed && socketDropIsError_) {
        // 屬於本輪會話未決斷線，錯誤照常派發
    } else {
        return;
    }
}
```

### 描述符多頭競爭：從 ::dup() 移植困局到物件所有權接管

`NetworkDebuggerLoopback` 測試中交織著兩層不同維度的缺陷。表層缺陷屬於執行緒親和性違背：`QTcpServer server_` 為未宣告 parent 的值成員，工作執行緒執行 `moveToThread` 時其滯留主執行緒，導致 `nextPendingConnection` 跨執行緒建立子物件，觸發大量跨執行緒警告。深層缺陷則是嚴重的未定義行為（UB）：即便消除警告，資料流依然死鎖。基於獨立用例的最小重現證明：原實現保留 `sourceSocket`，同時將其底層 native fd 交給 `TcpChannel` 採納——Qt 的兩個內部 Socket Engine 同時監聽同一個系統 fd 的讀就緒事件，引發嚴重的讀飢餓（Read Starvation），脫離框架的裸重現直接觸發區段錯誤（SIGSEGV）。Windows CI 此前能夠通過，僅僅是由於底層系統容忍了對已關閉控制代碼的重複釋放報錯。

第一版修復在 Linux 本地透過 POSIX `::dup()` 複製私有 fd 並註銷原 Socket，本地測試全量通過。然而推送到 Windows CI 後，MSVC 編譯期直接攔截：

```text
TcpServerHandle.cpp(15,10): error C1083: Cannot open include file: 'unistd.h'
```

`_dup()` 路線同樣走不通：Windows 下 `socketDescriptor()` 回傳的是 Winsock `SOCKET` 控制代碼而非 CRT 檔案描述符；若換用 `WSADuplicateSocket`，不僅需要引入 `<windows.h>`，更會堆砌大量平台 `#ifdef` 分支，直接違反專案工程紅線。

在跨平台 C++ 工程中，當一個平台專有的系統呼叫在另一端找不到乾淨的等價物時，多半是抽象層次選錯了。

最終放棄在描述符層面的複製與析構修補，轉向物件所有權採納（Object Adoption）：

```cpp
// TcpServerHandle 交付物件所有權：
channel->adoptSocket(socket);

// TcpChannel 內部安全接管：脫離 QTcpServer 父子樹，獨佔原生 fd 並重綁訊號
socket->setParent(nullptr);
delete socket_;
socket_ = socket;
wireSocketSignals();
```

透過純 Qt 物件樹完成生命週期接管，雙引擎爭搶、POSIX 相容陷阱與平台條件分支一併消除。

---

## 本地能過不算數：載入器隱式依賴、驗證契約與離屏盲區

### 動態載入器的隱式依賴

打包腳本在 CI 容器首次執行時，Step 2 拋 `no Qt libraries collected` 退出。一串連鎖落空：`cmake --install` 剝掉了二進位的 build rpath；容器內既無 `LD_LIBRARY_PATH`，Qt 前綴也未註冊進 ldconfig；而此刻 bundle 內部的 `lib/` 尚為空白——`ldd` 解析依賴陷入死鎖。

開發機之所以能夠跑通，全靠本地特定環境未受控的隱式依賴。修復原則遵循顯式宣告與最小作用域：不污染全域容器環境，僅在 `ldd` 靜態分析階段注入行程級隔離前綴：

```bash
LD_LIBRARY_PATH="${QT_LIBS}" ldd "${TARGET_BIN}"
```

### 驗證邏輯失效：審視校驗器自身的規格契約

Step 5 密封性檢查攔截了 `libQt6WaylandClient.so.6` 所宣告的系統依賴 `libwayland-cursor.so.0 not found`。

三條技術路線擺在面前：最輕率的做法是向 CI 容器執行 `apt install libwayland-cursor0`，但這屬於「為迎合錯誤的驗證語意而修補環境」；只打包 xcb 外掛可收窄依賴面，但剝離原生 Wayland 支援屬於產品決策，不應由一次發佈建置修復來逾越；若將 Wayland 系統庫打包進 bundle，則直接破壞了「宿主基礎系統庫不隨包分發」的架構設計。三條路線均被否決。在真實執行場景中，執行 Wayland 會話的桌面環境必預裝此庫；在純 X11 極簡環境中，Qt 亦可安全降級回落至 xcb。容器並非最終桌面，因此應重構的是校驗器的契約，嚴格落實所有權切分：

1. 防前綴洩漏：任何 ELF 產物不得解析回建置機的 Qt 安裝前綴。
2. 發佈完整性：未解析依賴項中，凡屬 Qt 官方發行包自帶者（`$QT_LIBS`），必須全部收錄進 bundle；系統級動態庫缺損屬於宿主基線範疇，予以容忍。

為這套校驗邏輯補全 4 分支測試套件時，還順帶排查出舊腳本中一個潛伏已久的 Bug：

```bash
# 真實 ldd 輸出樣例: "libwayland-cursor.so.0 => not found"
# 欄位解析: $1="libwayland-cursor.so.0", $2="=>", $3="not", $4="found"

# 舊腳本（恒假 Bug：空格切分下，$3 永遠不可能等於包含空格的 "not found"）:
awk '$3 == "not found"'

# 修復後（嚴格欄位比對）:
awk '$2 == "=>" && $3 == "not" && $4 == "found"'
```

未經測試的驗證邏輯本身也是生產程式碼。舊版本此前靜默放行的假象，本質上是檢驗邏輯長期處於「失明」狀態（Unexercised Validation）。

### RUNPATH 深度：離屏冒煙測試的結構性盲區

第七類故障不在「此前能跑」的模式內——它源於部署腳本覆寫 RUNPATH 時，將原本正確的路徑破壞。

腳本曾粗暴地給所有外掛統一注入 `$ORIGIN/../lib`。然而外掛物理分布於 `plugins/<category>/lib*.so` 兩級目錄下，該相對路徑將解析至不存在的 `plugins/lib`。Qt 官方預編譯外掛原生燒錄的 RUNPATH 本就是符合目錄層級的 `$ORIGIN/../../lib`。

自動化冒煙測試（Step 6）為何毫無感知？因為 CI 冒煙測試運行於 `QT_QPA_PLATFORM=offscreen` 模式下，離屏平台外掛所依賴的底層核心庫（QtCore / QtGui）早已被主程式預先載入進記憶體。真實使用者的 X11 桌面環境則截然不同：動態載入 `libqxcb.so` 必須解析其獨有的 `libQt6XcbQpa.so.6`（主程式並未直接連結此庫）。一旦 RUNPATH 破損，使用者在桌面按兩下啟動應用，行程將因無法載入平台外掛而直接異常退出。

修一層，露一層。離屏測試因符號預先載入而存在結構性盲區，基於 ELF 依賴圖的靜態斷言成為了阻斷此類啟動崩潰的最後一道防線。修復後的整包解包稽核：26/26 外掛 RUNPATH 規範統一、零 Qt 前綴洩漏、裸 `ldd` 零未解析，該套斷言隨後在 Step 5 的 CI 端複驗中順利通過。

---

## 哪些已有防禦機制按預期發揮了作用（What Went Well）

復盤不能只列失敗項。這一輪裡，有幾處已有機制按設計發揮了作用：

- 本地基準與 CI 失敗集 7/7 嚴格對應：絕大多數修復在本地實現閉環驗證，CI 週期僅消耗在驗證真實的環境差異上，避免了針對已知故障反覆空耗流水線。
- Windows CI 充當了 POSIX 污染的編譯期哨兵：`unistd.h` 在 Linux 本地編譯期靜默放行，Windows CI (MSVC) 在初次提交時即實施硬性攔截。
- 全量測試套件（開啟 ASan 動態插樁）在發佈前精準攔截了真實的未定義行為（雙引擎爭搶 fd）與產品邏輯缺陷（錯誤訊號吞沒）——高品質的測試體系本身就是最敏銳的報警哨位。
- Step 5 在驗證語意重構後的首次端到端演練中，立即擷取了深層的外掛 RUNPATH 深度缺陷：分層防禦機制按預期生效。

## 沉澱下來的四條防線

從首輪流水線失敗到雙平台全量測試穩定通過、多平台發佈資產自動化建置就緒。事後審視，逐輪暴露的故障只是表徵，根源在於兩個長期事實——Linux 發佈鏈路此前從未被完整執行過，以及多處「能夠執行」建立在局部環境的特例表現之上。多輪收斂最終凝結為四條工程防線，每條均有已落地的技術機制作為保障，可直接複用到其他跨平台 C++/Qt 交付工程：

1. 「在別處能跑」往往只是逃脫了未定義行為的懲罰。連結器歸檔裁剪策略、非同步事件傳遞時序、系統級原生控制代碼抽象差異——每一處「在另一端看似正常」，往往只是那一側的環境偶然容忍了未定義行為。硬性工程約束隨之建立：雙平台全量單測套件（帶 ASan 動態插樁）出現任何斷言失敗即無條件熔斷流水線，徹底杜絕「帶傷交付、上線再補」的倖存者偏差。
2. 當系統迫使你編寫 `#ifdef` 時，首先審視抽象層次。醜陋的平台分支往往源於過早陷入作業系統專用控制代碼的細節糾纏；將控制權提升至物件生命週期所有權，平台差異便不復存在。落地成果：網路控制代碼接管徹底收斂至 `adoptSocket` 單一通道，專案自有程式碼庫保持零平台條件編譯。
3. 驗證邏輯本身也是生產程式碼，必須經過嚴格自我測試。恒真放行（Vacuously True）的校驗腳本比顯式報錯更具欺騙性——靜默放行往往只是掩蓋了檢驗邏輯的徹底失效。落地成果：`check_ldd` 核心函式本體從生產腳本中抽離，直接驅動 4 分支測試 Harness（包含正向對照組），實現校驗邏輯與其自我測試套件的零漂移。
4. 捍衛靜態不變量，而非修補環境表象。面對建置容器回報的依賴缺失，深究一步即可明確契約邊界：釐清「哪些依賴必須隨二進位包分發」與「哪些依賴本應由宿主發行版提供」，交付鏈路方具備確定性。落地成果：Step 5 確立的兩項靜態不變量（零前綴絕對路徑洩漏、Qt 隨包依賴零缺失）固化為打包門禁，容器極簡基線不再引發假陽性誤報。

文末附上一份可直接複用的排查檢查清單。文中所用手段不與本專案繫結：使用 `nm` 驗證關鍵資源註冊符號是否存留；透過最小重現隔離未定義行為；以行程級 `LD_LIBRARY_PATH` 替代對全域環境的污染；對 `ldd` 輸出實施嚴格的欄位級比對而非正規表示式模糊比對；打包後執行整包解包稽核，全面校驗 RUNPATH 與前綴洩漏。若你的流水線亦陷入「本地能過、CI 必掛」的困境，不妨對照此清單逐項排查。
