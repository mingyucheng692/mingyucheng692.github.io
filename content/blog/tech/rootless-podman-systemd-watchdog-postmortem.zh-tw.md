---
title: "Rootless Podman + Systemd 託管失效復盤：一次恢復鏈路排查與修復紀錄"
date: 2026-04-03T20:00:00+08:00
draft: false
tags: ["Podman", "Systemd", "Rootless", "Container", "Nginx", "Redis", "Go", "Postmortem", "SRE"]
categories: ["Tech", "Engineering", "Infrastructure"]
summary: "一次 Rootless Podman + Systemd 使用者單元託管失效的復盤，梳理已確認的致因因素、實際修復動作、驗證方式與尚未覆蓋的風險。"
url: "/zh-tw/blog/tech/rootless-podman-systemd-watchdog-postmortem/"
---

這篇文章記錄一次容器恢復鏈路失效後的排查與修復過程。在一套由 Rootless Podman 託管的服務中，資料庫容器異常退出後沒有如預期被自動拉起，隨後又連帶暴露出啟動逾時、容器仍在線但服務顯示為 `inactive`，以及重啟多次後被頻率限制鎖死等問題。

> 說明：本文中的專案名、服務名、路徑、網域、使用者名稱、主機識別、時間點、PID、命令輸出與日誌片段皆為脫敏、抽象或重寫示例，只保留與排障鏈路相關的技術資訊，不對應任何真實生產識別。

- 執行環境：Rootless Podman 4.9.x
- 行程託管：Systemd User Units
- 業務元件：TimescaleDB、Redis、Go Backend、Nginx Frontend
- 目標能力：容器異常退出後由 Systemd 自動恢復

## 事故摘要

- 觸發背景：資料庫容器異常退出後，現場最先看到的是資料庫對應 Unit 變成 `inactive`，其餘服務對應 Unit 仍維持 `active`
- 過程表現：進入修復後，資料庫服務先恢復為 `active`，但其餘服務一度未被 Systemd 正確接管，於是陸續出現啟動逾時、容器狀態與 Unit 狀態不一致，以及部分服務因頻率限制被鎖死
- 已確認的致因因素：主 Unit 曾透過 `sed -i` 被腳本直接改寫，並在 PowerShell 跳脫截斷後被寫壞；Compose 與 Systemd 並存造成控制權分離；自動生成的 Unit 仍需後處理；恢復腳本對依賴清理與重啟失敗覆蓋不足
- 修復方向：停止用 `sed -i` 直接改主 Unit，改成重新生成 `.service` 後交由 Python 腳本修補，並顯式把執行期控制權交回 Systemd，再補上狀態輪詢與最終健康檢查
- 當前結果：在目前環境中，恢復鏈路已能按固定順序執行，失敗時也有明確診斷入口；但 readiness 判定與觀測能力仍待補強

## 現象與影響

最初現場並不是「所有服務一起掉線」，而是資料庫對應 Unit 先變成 `inactive`，其餘服務仍維持 `active`。真正把問題放大的，是進入修復與接管階段後又暴露出另一組現象：

- 有些服務在 `systemctl --user status` 中顯示逾時後重啟
- 有些容器已經 `Up`，但對應的 Systemd 服務仍是 `inactive`
- 有些服務因為連續失敗，被 Systemd 直接停止重試

這表示問題不是某一個容器的單點故障，而是「容器實際狀態、Systemd 託管狀態、部署腳本預設假設」三者之間已經出現偏差。本文聚焦在「恢復鏈路為什麼失效、腳本後來如何調整」，不展開資料庫掉線本身的業務觸發原因。

---

## 排查思路：先確認誰真正持有行程控制權

這次排查不是從單條報錯開始，而是先回答一個更基礎的問題：容器退出後，究竟是誰負責發現它失敗、判定它異常，並把它重新拉起。沿著這條線索，最後定位到四類彼此疊加的問題。

## 問題一：主 Unit 被直接修改，最終損壞

最先暴露的問題，是資料庫對應的主 Unit 被破壞成空檔案：

```text
systemd[<pid>]: /home/<deploy-user>/.config/systemd/user/container-<db-service>.service:1: Missing '='.
```

進一步檢查檔案本體時，可以直接看到它已被截斷為 `0` 位元組：

```bash
ls -l /home/<deploy-user>/.config/systemd/user/container-<db-service>.service
# -rw-rw-r-- 1 <deploy-user> <deploy-user> 0 <timestamp> container-<db-service>.service

file /home/<deploy-user>/.config/systemd/user/container-<db-service>.service
# container-<db-service>.service: empty
```

這次對觸發鏈已經能描述得更具體：舊版腳本確實會直接以 `sed -i` 修改主 Unit；而在透過 PowerShell 下發這類命令時，跳脫處理出現偏差，導致目標檔沒有被正確寫回，最終把主 Unit 截斷成 `0` 位元組，使 Systemd 無法再解析。問題不只是「腳本用了 `sed -i`」，而是「把主 Unit 當成可原地修改的對象」，再疊加跨 Shell 跳脫差異，共同放大了損壞風險。

舊版腳本中的關鍵路徑大致如下：

```bash
# 舊版：直接修改主 Unit
sed -i 's/Restart=always/Restart=no/g' "$service"
```

問題本身已經足夠清楚：主服務檔同時承擔「執行定義」與「運維開關」兩種角色，本身就是高風險設計。一旦主 Unit 損壞，解析、啟動與重啟策略會一起失效。

後來的調整比較直接：

- 主服務檔只保留生成產物屬性，不再由 `sed -i` 直接修改
- 將重啟策略等開關下沉到 `.service.d/watchdog.conf`
- 生成後的 `.service` 相容性修補交給 Python 處理，Drop-in 則由腳本覆寫寫入

目前腳本中，Drop-in 的寫入方式大致如下：

```bash
cat > "$DROP_IN_DIR/watchdog.conf" << EOF
[Service]
Restart=always
RestartSec=10s
ExecStartPre=-/usr/bin/podman rm -f <db-service>
EOF
```

至少這樣把邊界拆清楚了：主 Unit 視為可重建產物，執行期開關統一放在 Drop-in 管理。從目前腳本與後續現場結果看，也沒有再重現「因原地修改主 Unit 而導致服務檔損壞」的問題。

## 問題二：Systemd 判定失敗，但容器其實已經啟動

第二類問題更隱蔽。故障階段，Systemd 會把某些服務標記為啟動逾時：

```text
Active: activating (auto-restart) (Result: timeout)
Main PID: <pid> (code=exited, status=0/SUCCESS)
```

但同時 `podman ps` 又能看到容器確實已經起來，本質上是行程契約出現分裂。

`podman generate systemd` 生成的 Unit 預設使用 `Type=notify`。這代表 Systemd 不會只因為行程存在就判定服務啟動成功，而是要求執行中的容器向宿主送出 `sd_notify` 就緒訊號。

故障階段尚未修補時，生成產物中的關鍵欄位如下：

```ini
Type=notify
NotifyAccess=all
```

在恢復後重新採集目前環境時，重新生成的產物仍保留這個預設值；而成功上線後實際生效的 Unit 已經改為 `Type=simple`：

```ini
# 故障階段對應的預設生成結果
Type=notify

# 修復上線後的生效 Unit
Type=simple
```

這次現場能確認的現象是：當服務仍使用預設生成的 `Type=notify` Unit 時，容器雖然已經 `Up`，對應 Unit 仍會在逾時後被 Systemd 判為失敗並重啟。後來的修復做法，是把生成產物中的 `Type=notify` 改成 `Type=simple`，把狀態判定切回到「由 Systemd 直接追蹤前景行程」。

這裡的邊界仍要說清楚：這並不等於已經證明 Rootless 場景下 `notify` 一定不可用，而是目前這套環境中，`notify` 沒有表現出穩定、可依賴的就緒語義。對這類長駐前景行程來說，`Type=simple` 是更保守、也更容易排障的選擇。

## 問題三：一部分容器其實不在 Systemd 控制之下

更準確地說，這個危險狀態出現在修復過程裡，而不是最初故障現場：資料庫對應 Unit 已恢復為 `active`，但其餘服務對應 Unit 仍顯示 `inactive (dead)`。

當時現場狀態大致如下：

```bash
podman ps
# ... 部分業務容器可能已重新起來

systemctl --user list-units 'container-*.service'
# ... container-<db-service>.service 為 active
# ... 其餘服務對應 Unit 為 inactive (dead)
```

原因是這些容器一開始並不是由 Systemd 拉起，而是更早以前透過 `podman-compose up -d` 直接啟動。對這類非託管容器來說，即使容器行程已經存在，Systemd 仍未真正持有控制權，自然也談不上穩定接管、失敗感知與自動恢復。

這暴露出一個很容易被忽略的前提：託管能力不只取決於 Unit 檔是否存在，更取決於行程是不是由 Systemd 真正持有。如果執行期控制權仍由 Compose 持有，那麼「已經寫了重啟設定」和「恢復能力真的生效」就不是同一回事。

這裡也需要補充實作細節：目前部署腳本並不是完全不用 Compose。它仍會先用 `podman-compose` 完成建置與冷啟動，再呼叫 `watchdog-enable.sh` 生成並修補 Unit，隨後顯式停止這些由 Compose 拉起的容器，最後才透過有順序的 `systemctl --user start` 將控制權交回 Systemd。問題不在於「腳本裡出現過 `podman-compose`」，而在於恢復能力應生效時，執行期控制權究竟是不是還握在 Compose 手上。後來的調整，就是把這個交接動作固定下來：

- 停掉現有非託管容器，重新由 `systemctl --user start` 接管
- 在交接前後做狀態檢查，避免再次留下「容器在跑，但 Systemd 不知情」的狀態

## 問題四：自動生成與恢復腳本仍需要補足細節

### 1）自動生成的 Unit 仍需後處理

重新回到 `podman generate systemd --new` 後，至少暴露出三類問題：

- 透過 `podman-compose` 等工具初始化的參數會污染生成結果。例如原容器若帶有 `-d`，`podman generate systemd --new` 也會原樣帶出，導致 Systemd 執行 `podman run -d` 後，從託管視角看行程立刻退出，無法追蹤真正的容器主行程
- Redis 是另一個典型案例：若原始設定透過空參數禁用危險命令，例如 `rename-command ""`，生成工具會靜默丟掉空字串，最後變成 `redis-server --rename-command FLUSHALL`，直接造成啟動參數錯誤
- 預設值 `Type=notify` 會再次把 Rootless 場景的逾時問題帶回來

Redis 這個問題，從原始容器參數和生成結果之間就能直接對照出來：

```yaml
# 原始容器參數
command:
  - --rename-command
  - "FLUSHALL"
  - ""
```

```text
# 未修補的 generate 結果
redis-server --rename-command FLUSHALL
```

即使在恢復後，重新執行 `podman generate systemd --new --name <redis-service>`，仍能看到空字串被吞掉；而實際生效的 Unit 已經補回：

```ini
# 恢復後重新生成的結果
--rename-command FLUSHALL

# 修復上線後的生效 Unit
--rename-command FLUSHALL ""
```

`-d` 也是同類問題。對照恢復後重新生成的產物與修復上線後的生效 Unit，可以直接看到預設生成結果仍包含 `-d`，而接管用的 Unit 已經把它移除：

```ini
# 恢復後重新生成的結果
ExecStart=/usr/bin/podman run \
  ...
  -d \
  --sdnotify=conmon \

# 修復上線後的生效 Unit
ExecStart=/usr/bin/podman run \
  ...
  --sdnotify=conmon \
```

因此目前 `watchdog-enable.sh` 的流程是：先重新執行 `podman generate systemd --new --name ...`，再用 Python 補丁腳本修補生成產物。實際落地包含三件事：

- 將 `Type=notify` 改成 `Type=simple`
- 移除 `ExecStart` 中殘留的 `-d`
- 對 Redis 的 `rename-command ""` 做回填，避免生成階段丟失空參數

這裡選擇 Python 也有很實際的理由：它更適合處理由 `\` 續行的多行命令，能讓生成產物的修補行為更可控。

### 2）依賴關係會影響失敗容器的清理

資料庫服務恢復前，需要先執行 `podman rm -f` 清理舊容器；但因為上層服務仍持有依賴引用，刪除動作回傳 `exit 125`。這代表單容器恢復不是孤立操作，依賴圖會反向影響清理過程。

直接報錯大致如下：

```text
Process: ExecStartPre=/usr/bin/podman rm -f <db-service> (code=exited, status=125)
```

目前腳本的處理方式，是把依賴清理邏輯寫進 Drop-in：

- 資料庫服務在 `ExecStartPre` 中會先停掉依賴它的上層服務，再執行 `podman rm -f <db-service>`
- 其他幾個服務只清理各自容器

這不是一套通用模板，而是針對這次這組服務依賴關係做的腳本化處理。

### 3）恢復鏈路不能只依賴固定 `sleep`

早期腳本確實存在靠固定 `sleep` 等待依賴的做法。

目前部署腳本在 Compose 冷啟動階段仍保留少量固定等待；但在控制權交給 Systemd 之後，已經改成按依賴順序逐個 `systemctl --user start`，並配合 `wait_for_service()` 輪詢 `is-active` / `is-failed`，在失敗或逾時時輸出對應的 `journalctl`。這比盲等更容易定位失敗點。

因此最後把啟動順序改成顯式編排：

- `<db-service>` -> `<redis-service>` -> `<backend-service>` -> `<frontend-service>`

在全部服務接管完成後，腳本還會再補一次應用層健康檢查，用來確認「服務 active」已進一步接近「對外可用」。這一步是對 `Type=simple` 取捨的補償，不是每個服務啟動時都執行的深度 readiness 檢查。

交接給 Systemd 後，等待邏輯大致如下：

```bash
systemctl --user start container-<db-service>.service
for i in $(seq 1 60); do
  systemctl --user is-active --quiet container-<db-service>.service && break
  systemctl --user is-failed --quiet container-<db-service>.service && exit 1
  sleep 1
done
```

### 4）Systemd 的頻率限制會把服務鎖死

當依賴尚未就緒時，Frontend 會反覆失敗並重啟。例如 Nginx 在啟動時會強制解析 `<backend-service>` 的 upstream 名稱。若在設定視窗內重啟次數超過 `StartLimitBurst`，Systemd 就會停止重試。這時即使後續依賴恢復正常，服務也不會自動起來，除非顯式重設失敗狀態：

```text
container-<frontend-service>.service: Start request repeated too quickly.
container-<frontend-service>.service: Failed with result 'exit-code'.
```

在 Nginx 這一側，應用層證據通常更直接：

```text
[emerg] 1#1: host not found in upstream "<backend-service>" in default.conf:31
```

```bash
systemctl --user reset-failed container-<frontend-service>.service
```

`deploy-all.sh` 現在也已經在拉起 Frontend 前補上一個 `reset-failed`，用來清掉此前可能殘留的鎖死狀態。

---

## 本次故障中已確認的致因因素

從表面看，這次故障像是資料庫異常後，現場狀態與接管狀態一路分裂；回頭看，其實是幾類問題疊加：

1. 主 Unit 被腳本直接改寫，服務定義本身存在被破壞的風險
2. `podman-compose` 直接啟動與 Systemd 託管並存，導致容器狀態與託管狀態分離
3. `podman generate systemd` 的產物不能直接拿來用，仍需補齊 `Type`、`-d` 與參數修補
4. 恢復腳本最初對依賴清理、頻控恢復與最終健康檢查覆蓋不足

這不是某一個開關寫錯造成的單點問題，而是恢復鏈路在定義、接管與執行三個層面都出現斷點。因此修復也不只是改一條報錯，而是把執行期控制權、生成後修補與失敗後診斷入口重新串成一條可用的鏈路。

---

## 已落地的修復動作

結合 `watchdog-enable.sh` 與 `deploy-all.sh`，目前實際落地的動作大致如下：

- 部署腳本先檢查執行使用者、`HOME`、`XDG_RUNTIME_DIR` 與 `DBUS_SESSION_BUS_ADDRESS`，降低 Rootless User Units 因執行環境不一致而失效的機率
- 部署階段仍使用 `podman-compose` 完成建置與冷啟動，但在交接階段會顯式停掉這些容器，再由 Systemd 按順序接管
- `watchdog-enable.sh` 會先備份舊 Unit，再重新生成 `.service`
- 生成後的 `.service` 改由 Python 腳本做三項修補：改成 `Type=simple`、刪除 `-d`、補回 Redis 空參數，同時避開先前 `sed -i` 路徑裡 PowerShell 跳脫截斷的問題
- Drop-in 中統一寫入 `Restart=always`、`RestartSec`、`StartLimit*` 以及各服務對應的 `ExecStartPre`
- 交接給 Systemd 後，腳本按 `<db-service> -> <redis-service> -> <backend-service> -> <frontend-service>` 順序啟動，並用 `wait_for_service()` 檢查 `is-active` / `is-failed`；若失敗或逾時，直接輸出對應 `journalctl`
- 在拉起 Frontend 前補一次 `reset-failed`，避免殘留的頻率限制狀態阻斷恢復
- 全部服務接管完成後，再追加一次應用層健康檢查，確認系統不只「服務 active」，也更接近「對外可用」

這套做法仍然是在單機、Rootless、User Units 這組限制下的工程化方案，解決的是「誰持有行程、如何恢復、失敗後如何排查」這些具體問題，並不等同於更廣義的高可用設計。

---

## 修復後如何驗證

- 在交接前後對照 `podman ps` 與 `systemctl --user list-units`，確認不再出現「容器在跑，但對應 Unit 為 `inactive`」的狀態
- 透過 `wait_for_service()` 輪詢 `is-active` / `is-failed`，讓服務在啟動階段就能暴露失敗點，而不是交給固定 `sleep`
- 若服務失敗或逾時，立即輸出對應 `journalctl`，把排查入口固定到具體 Unit
- 全部服務接管完成後補一次應用層 `/health` 檢查，避免只用 `Type=simple` 的行程在線語義來推斷外部可用性

恢復後再次核對時，四個容器與四個 User Unit 已重新對齊：

```text
podman ps
<db-service> / <redis-service> / <backend-service> / <frontend-service> 均為 Up

systemctl --user list-units 'container-*.service'
對應四個 Unit 均為 active (running)
```

這些驗證主要覆蓋「是否由 Systemd 持有行程」「失敗時是否能及時暴露」「接管完成後是否對外可用」，尚未擴展到更細的 readiness 訊號與持續觀測。

---

## 這次排障後明確下來的約束

- 主 Unit 只保留為生成產物，執行期開關統一放到 Drop-in；這不是形式上的規範，而是為了避免主檔一旦損壞，就同時打斷解析、啟動與重啟策略
- `podman generate systemd` 的產物不能直接視為可執行結果；生成、修補與重載應被視為同一個步驟。就這次腳本而言，`Type=notify`、`-d` 與空參數問題都屬於生成後必須處理的相容性項
- 部署期控制面與執行期控制面需要明確分開；Compose 仍可用於建置與冷啟動，但進入執行期後，控制權必須顯式交回 Systemd，否則「已寫重啟設定」與「恢復能力真的生效」會繼續混在一起
- 恢復鏈路不能只寫 `Restart=always` 就算完成，還必須覆蓋依賴清理、頻率限制恢復、狀態輪詢與最終健康檢查，讓故障後的恢復與排查可重複執行

---

## 後續演進方向

這次修復解決了恢復鏈路中的幾個確定問題，但仍有幾類風險尚未完全覆蓋，後續仍需要繼續推進：

- `Type=simple` 配合 `is-active` / `is-failed` 與最終 `/health` 檢查，已補上「行程在線」與「實際可用」之間的缺口，但仍缺少更細粒度的 readiness 訊號；若服務短暫變成 `active`，之後才暴露內部初始化失敗，現有腳本仍可能較晚才發現
- 目前方案仍依賴 `podman generate systemd` 加腳本修補；只要生成產物格式、參數展開方式或 Podman 版本行為改變，現有補丁邏輯就可能失效，因此是否改用 Quadlet 或其他宣告式方式，仍值得單獨評估
- 目前驗證仍偏向腳本內診斷；失敗次數、退出碼、容器狀態與頻率限制命中情況尚未接入採集、上報與告警，代表系統外層仍缺少持續觀測能力，類似問題更可能在故障後才被動暴露

## 這次排障帶來的直接改善

- 由誰啟動行程、由誰負責重啟，邊界比之前清楚
- 自動生成的 Unit 是否可直接使用，如今有了明確判準與後處理步驟
- 接管失敗時該看哪一層日誌、在哪一步退出，腳本內已有穩定入口
- 服務 active 與對外可用之間的差異，已補上一層應用層驗證

這些改動並不代表整套方案已成為更廣義的高可用設計，但至少把原本「重啟配置存在、恢復卻不可靠」的狀態，整理成一條更容易驗證、也更容易重複執行的恢復鏈路。

## 結語

這次排查最後留下的結論其實很直接：設定裡寫了 `Restart=always`，不代表恢復鏈路就已經完整。主 Unit 是否穩定、容器是不是由 Systemd 啟動、生成產物有沒有修補、依賴與頻率限制有沒有被腳本覆蓋，這些細節都會影響最終結果。

至少在這次這套單機 Rootless Podman 場景裡，把這些斷點補進腳本之後，行程控制權、排查入口與故障處置路徑都比之前清楚很多。這未必表示方案已經「完美」，但至少把一次原本高度依賴經驗判斷的故障恢復，整理成一條更可解釋、也更容易驗證的工程鏈路。

---

## 附錄：關於 Linger 機制的補充說明

若 Rootless Podman 要搭配 `systemd --user` 託管服務，通常需要先為目標使用者啟用 `linger`。否則一旦該使用者登出，使用者層級的 Systemd 實例可能被回收，相關服務也就無法在「無人登入」的情況下持續運行。本次事故發生時，環境已滿足 `Linger=yes`，因此 `linger` 不是本次故障根因，這裡僅作為前置檢查項補充。

```bash
# 檢查目前狀態
loginctl show-user <deploy-user> --property=Linger

# 若尚未啟用，請由具 sudo 權限的使用者執行：
sudo loginctl enable-linger <deploy-user>
```
