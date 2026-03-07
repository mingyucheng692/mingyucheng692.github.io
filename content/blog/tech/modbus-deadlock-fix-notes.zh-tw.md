---
title: "踩坑日記：Modbus 多執行緒死鎖修復記錄"
date: 2026-03-07T10:00:00+08:00
draft: false
tags: ["Modbus", "Qt6", "C++", "Deadlock", "Multithreading", "Industrial Software"]
categories: ["Tech", "Engineering"]
summary: "記錄一次 Qt 多執行緒 Modbus 場景中的退出死鎖與超時誤判問題，以及對應的工程化修復方案。"
url: "/zh-tw/blog/tech/modbus-deadlock-fix-notes/"
---

在 `Modbus-Tools` 的迭代過程中，遇到過一次典型的並發問題：視窗已關閉，但程序未退出，調試會話也無法正常結束。後續修復中，又暴露出通信超時誤判與重入競爭問題。本文記錄排查過程與最終方案。

---

## 現象一：退出階段卡住

- 主執行緒銷毀 `MainWindow` 時會等待工作執行緒退出
- 工作執行緒可能仍阻塞於串口或網路讀寫流程
- UI 已消失，但程序保持存活

## 原因一：執行緒依附關係不一致

`QSerialPort` 與 `QTcpSocket` 的事件處理依賴其所屬執行緒事件循環。若物件建立於主執行緒，但調用路徑在工作執行緒，退出階段容易出現互相等待。

## 處理一：明確物件歸屬執行緒

在通道建立後顯式遷移至工作執行緒：

```cpp
stack.thread = std::make_shared<QThread>();
stack.channel->moveToThread(stack.thread.get());
```

同時確保物件建立時不綁定 `parent`，避免遷移失敗。

---

## 現象二：連線後偶發 Timeout

- 監控抓包可見設備有回應
- 業務層仍返回超時

## 原因二：阻塞等待餓死事件循環

請求送出後使用 `condition_variable::wait_until` 阻塞等待，導致承載 IO 的執行緒無法即時處理 `readyRead` 事件，回包回調被延後。

## 處理二：改為可讓渡事件的等待循環

```cpp
while (true) {
    QCoreApplication::processEvents(QEventLoop::AllEvents);
    if (signalReceived) break;
    if (timeout) return Error;
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
}
```

該方案在可控範圍內維持回應性，同時避免忙等佔滿 CPU。

---

## 現象三：高頻操作下偶發自鎖

引入 `processEvents` 後，等待窗口期內可能觸發同執行緒重入，導致同一把互斥鎖被重複申請。

## 原因三：鎖模型與重入路徑不匹配

原有 `std::mutex` 無法支持同執行緒重複進入臨界區。

## 處理三：改用遞歸鎖保護請求序列

```cpp
std::recursive_mutex requestMutex_;
```

---

## 復盤結論

1. IO 物件所屬執行緒必須在設計階段固定並貫穿生命週期管理。
2. IO 執行緒中應避免長時間阻塞等待，必要時採用可處理中斷事件的等待策略。
3. 引入事件讓渡後，要同步評估重入路徑與鎖策略。

這次修復不僅解決了退出掛起，也提升了 Modbus 通信超時判定的一致性。對工業軟體而言，穩定退出與時序可解釋性與功能本身同等重要。
