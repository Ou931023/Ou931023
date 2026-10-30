---
layout: default
title: "API 重試為什麼可能產生重複訂單？從 Idempotency Key、ACID 到 PostgreSQL 看懂可靠系統"
description: "API 逾時重試為何會重複建立訂單？比較 Idempotency Key、ACID、PostgreSQL 與企業營運風險。"
date: 2026-10-30
categories: [MIS, Computer Science]
tags: [Idempotency, API, PostgreSQL, Transactions, MIS, CS]
---

# API 重試為什麼可能產生重複訂單？從 Idempotency Key、ACID 到 PostgreSQL 看懂可靠系統

> 消費者按下購買，伺服器已建立訂單，回應卻在網路途中遺失。畫面顯示逾時，消費者再次點擊，結果收到兩筆訂單。系統真的「當機」了嗎？不一定：兩次請求可能各自成功，問題是系統沒有辨識「這其實是同一次業務意圖」。

## 一、技術正確不代表業務正確
在企業資訊系統中，重複訂單可能帶來退款成本、客服工作與庫存錯誤。HTTP 用戶端遇到逾時，無法直接知道遠端操作「尚未執行」「正在執行」或「已成功但回應遺失」。因此「請求逾時」不等於「交易沒有發生」。此問題同時涉及 CS 的分散式系統可靠性，以及 MIS 的營運風險與內部控制。

## 二、Idempotency（冪等性）到底是什麼？
冪等操作重複執行，不會持續增加相同操作的業務效果。例如對一筆訂單建立請求使用 `Idempotency-Key: operation-123`：第一次處理成功後，使用相同 Key 與相同參數重試，應取得先前結果或被安全辨識，而非再新增一筆訂單。

關鍵是 **Key 代表一次業務意圖，而不只是相同商品內容**。一個人真正要買兩杯同樣價格的咖啡，應使用兩個不同操作識別碼；相同 Key 配上不同內容則應拒絕或回報衝突。

根據 [RFC 9110 的 HTTP 方法語意](https://www.rfc-editor.org/rfc/rfc9110.html)，GET、PUT、DELETE 具有規範上的冪等性；POST 預設不保證。因此「POST 重送是否安全」必須由應用程式與伺服器契約共同設計。

## 三、為什麼 ACID 還不足以防止重複交易？
ACID 中 Atomicity（原子性）、Consistency（一致性）、Isolation（隔離性）、Durability（持久性）約束資料庫交易處理。但假設兩次 POST 各自啟動獨立 Transaction、分別 INSERT 一筆合法訂單，即使兩筆交易都符合 ACID，業務上仍可能多出一筆。

所以 **ACID 保證交易內的正確性，不會自動辨識跨請求的同一意圖**。可參考 [PostgreSQL Transaction 官方教學](https://www.postgresql.org/docs/current/tutorial-transactions.html)。

## 四、如何設計安全的防重機制？
最小可行設計可以使用 Node.js API 與 PostgreSQL：

1. 用戶端為每一次「建立訂單意圖」產生唯一 Key，重試必須重用該 Key。
2. 後端以帳戶或商戶範圍與 Key 組成唯一約束，記錄請求參數指紋、執行狀態、結果或訂單 ID。
3. 在資料庫交易中原子性地取得處理權與建立訂單；對重複 Key 進行衝突處理，不做第二次業務寫入。
4. 對已完成的相同操作回傳結果；對相同 Key 但不同參數拒絕；對仍在執行的請求明確定義等待或衝突策略。
5. 設計逾期、失敗復原、監控與重試政策。外部金流若不在同一資料庫交易範圍內，仍需金流業者的冪等機制與對帳流程。

PostgreSQL 的 `UNIQUE` 約束和 `INSERT ... ON CONFLICT` 能處理並行衝突，但只有唯一約束並不足以實作完整的結果重播、參數比對與外部付款一致性。[PostgreSQL INSERT 文件](https://www.postgresql.org/docs/current/sql-insert.html)

一個簡化的鍵表可以是：

```sql
CREATE TABLE request_keys (
  account_id BIGINT NOT NULL,
  request_key TEXT NOT NULL,
  request_hash TEXT NOT NULL,
  status TEXT NOT NULL,
  order_id BIGINT,
  PRIMARY KEY (account_id, request_key)
);
```

這是**資料結構示意，不是可直接投入正式金流的完整實作**。真正的交易流程還需處理並行鎖定、失敗狀態與業務操作提交順序。

## 五、容易混淆的三個觀念
**相同內容 ≠ 相同意圖**：重複購買相同商品可能是合法需求。**資料庫交易 ≠ API 冪等性**：單筆交易成功不會阻止另一筆相同意圖的交易。**冪等性 ≠ 零成本或絕對恰好一次執行**：即使對外業務效果只產生一次，系統仍可能接受多次請求、做多次查詢或執行恢復流程。

AWS 在 [Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) 中也強調辨識呼叫者意圖及安全重試的重要性。

## 六、MIS × CS 小型驗證實驗（提案，尚未實作）
規劃兩種 Node.js + PostgreSQL 訂單 API：A 為一般 INSERT，B 為具 Key、唯一約束與交易的設計。建立 10 組獨立購買意圖，每組模擬 10 次重送，包含同時送達及「資料已提交但 HTTP 回應遺失」情境。

**待測指標**：資料庫中實際訂單數、重複寫入數、HTTP 狀態與 P95 延遲。依正確性目標，每一組購買意圖最多應建立一筆訂單；但實際實驗結果、延遲與成本尚未量測，不能宣稱測試已通過。保留 SQL schema、測試腳本、log、測試輸出與錯誤案例，才能構成可供書審查驗的證據。

可做成一張「同一 Key 多次重試：無保護／有保護」的序列圖，並附上實際測試統計表。若延遲顯著增加，還應說明資料一致性與效能之間的取捨。

## 七、考研自測
**題目：** 訂單 API 已使用 ACID Transaction，為何在 HTTP 逾時重試後仍可能建立兩筆訂單？請提出兩層改善方法。

**參考答案：** ACID 保障每筆資料庫交易的性質，但兩次請求可能各自合法提交。API 層使用 Idempotency Key 與請求參數比對辨識同一意圖；資料庫層以 UNIQUE 約束及交易確保同 Key 只能對應一次業務建立，並處理並行請求與結果回傳。若有外部扣款，還需對接金流冪等機制及對帳。

## 結論
可靠系統的目標不只是「請求成功」，而是即使網路逾時、使用者重試、伺服器並行執行，也能守住業務不變條件。Idempotency 將 HTTP、Database 與 MIS 的風險控制串成同一項設計工作。本文提出可檢驗的方法，但**不代表作者已完成 API 實作、量化測試或真實企業導入**。

## References
1. [IETF RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
2. [AWS Builders' Library — Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
3. [PostgreSQL — Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)
4. [PostgreSQL — INSERT / ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html)
5. [Stripe — Idempotent requests](https://docs.stripe.com/api/idempotent_requests)

## 發布與書審資訊
**SEO 標題：** API 重試與冪等性怎麼設計？Idempotency Key、ACID、PostgreSQL 一次看懂
**摘要：** API 逾時後重新送出，為何會產生重複訂單？從冪等鍵、ACID、唯一約束到 Node.js 與 PostgreSQL 小型實驗提案，理解系統可靠性與企業風險控制。
**Tags：** MIS、Computer Science、Idempotency、API Design、PostgreSQL、Transaction、System Reliability
**書審定位：** MIS × CS 跨領域技術作品候選；驗證實驗尚未完成。
