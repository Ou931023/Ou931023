---
title: "快取讓網站變快，為什麼庫存卻可能顯示錯誤？從 Cache-Aside、TTL 到 Cache Stampede 看懂系統取捨"
date: 2026-11-13
description: "快取讓網站變快，卻可能讓庫存顯示過時或造成熱門鍵過期時的資料庫尖峰。從 Cache-Aside、TTL、Single-flight 到資料庫交易，理解效能與正確性的取捨。"
slug: cache-aside-ttl-stampede-inventory-consistency
categories: [MIS, CS]
tags: [Caching, Redis, Cache-Aside, TTL, PostgreSQL]
status: original-article-experiment-unverified
---

# 快取讓網站變快，為什麼庫存卻可能顯示錯誤？從 Cache-Aside、TTL 到 Cache Stampede 看懂系統取捨

> 電商商品頁顯示「尚有 3 件」，結帳時卻告訴你已售完；另一邊，熱門商品剛好過期，數千名訪客同時讓資料庫重新查詢。快取（Cache）原本是為了加速系統，為什麼反而可能造成資訊不一致與尖峰流量？

## 一、先備知識與問題界定
理解 HTTP 請求、資料庫 SELECT、基本 Key–Value 結構即可。本文討論的是**應用程式資料快取**，不是 CPU Cache，也不是作業系統的頁面置換。企業在意的不是「所有資料都快取」，而是讓讀取延遲、資料正確性、營運成本符合業務需求。

想像電商每秒有大量商品詳情查詢，商品名稱、圖片和介紹變動很少，但可售庫存可能每秒變動。如果每次都查資料庫，讀取負載很高；如果所有資訊都快取十分鐘，消費者看到的庫存就可能過時。因此，**資料更新頻率與錯誤代價**，才是快取策略的起點。

## 二、Cache-Aside：誰決定要讀資料庫？
Cache-Aside（旁路快取）由應用程式負責查詢與填入快取：

1. 使用者請求商品資料，先用商品 ID 查快取。
2. Cache Hit：找到資料，直接回傳。
3. Cache Miss：查詢資料庫，再把結果放入快取，供後續請求使用。

它可以降低重複查詢，但新增了一個問題：資料庫更新後，舊快取怎麼辦？常見作法是在更新資料庫成功後**刪除對應快取鍵**，讓下一次讀取重新載入；也可搭配 TTL（Time to Live）限制資料存活時間。

注意：TTL 不是「保證資料最多只會錯幾秒」的萬靈丹。即使設定 30 秒，在讀取、更新、刪除交錯執行時，也可能發生舊值被重新填入快取的競態；而不同系統對 TTL 的精確度、過期刪除與複製延遲也各有差異。

## 三、用一張時間線看懂快取不一致
假設商品庫存從 3 件降到 2 件：

- 10:00:00：快取已有舊值 3。
- 10:00:01：資料庫交易成功，庫存改為 2。
- 10:00:02：若快取尚未失效，讀取仍可能得到 3。
- 10:00:03：刪除快取鍵後，下次 Miss 才重新查到 2。

這是**示意時間線，不是實測紀錄**。更棘手的是：請求 A 在更新前讀到 3，但尚未寫入快取；請求 B 更新資料庫為 2 並刪除快取；最後 A 才把舊值 3 寫回快取。單純「先更新 DB、再刪 Cache」並不能消除所有競態。

可用版本號、事件驅動失效、短 TTL 或版本化快取鍵改善，必要時讓關鍵查詢直接讀取權威資料來源。但這些方法增加系統複雜度，仍需根據一致性要求選擇。

## 四、Cache Stampede：過期時為何反而更慢？
假設熱門商品快取在同一秒過期，1,000 個請求同時 Cache Miss，全部直接查詢資料庫。原本一筆可重用的查詢，變成大量重複工作，可能拖慢資料庫，進而造成更多逾時和重試。這種現象常稱 Cache Stampede（快取驚群／擊穿）。

改善策略包含：
- **Single-flight / request coalescing**：同一鍵在同一時間只讓一個工作負責重新載入，其他請求等待或共享結果。
- **TTL jitter**：為不同鍵加入適度隨機過期偏移，避免大批鍵同時失效；對單一超熱門鍵仍需其他保護。
- **Stale-while-revalidate**：對允許短暫過時的內容先提供舊資料，背景更新。
- **限流與降級**：資料庫已接近負載上限時，保護核心交易流程。

這些策略不是越多越好。若商品庫存與付款決策需要強正確性，直接回傳過時的「可售數量」可能不合適；但文章、圖片或非關鍵推薦資訊通常更能容忍短暫過時。

## 五、最重要的區分：快取顯示不等於交易判定
商品頁上的「庫存 3」可以是近似的展示資訊，**是否准許下單**卻應由權威庫存狀態與原子性交易判定。例如 PostgreSQL 可以在單筆 UPDATE 中加入 `stock >= quantity` 條件，檢查更新的列數是否為 1：

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = 42 AND stock >= 1
RETURNING stock;
```

若沒有回傳列，就代表這次扣庫存未成功。這段 SQL 是核心概念示例，真實訂單仍需考慮交易範圍、訂單狀態、支付與退款流程，不能直接當成完整電商系統。

這也說明：**Cache 是加速讀取的衍生資料，不能替代業務正確性的最後防線。** 快取與資料庫之間允許多少不一致，是產品與風險管理決策，不只是程式設計問題。

## 六、MIS × CS：如何選擇不同資料的快取策略？
將資料依更新頻率、讀取熱度及錯誤成本分類：

| 資料 | 可能策略 | 核心考量 |
|---|---|---|
| 商品圖片與說明 | 較長 TTL、版本化網址 | 更新少，重複讀取多 |
| 熱門商品排行榜 | 短 TTL、背景刷新 | 可容忍短暫落後 |
| 顯示用庫存提示 | 短 TTL 或主動失效 | 需清楚揭露可能變動 |
| 最終扣庫存與付款 | 權威資料庫與交易驗證 | 錯誤代價高，不以快取作最後判定 |

MIS 角度要衡量顧客體驗、錯誤訂單、客服成本、資料庫費用與系統維護複雜度；CS 角度則評估 Hit Ratio、P95 延遲、資料庫 QPS、Stale Read Rate、更新失效延遲與尖峰負載。單看快取命中率可能誤導決策：即使命中率很高，少數過時的關鍵資料仍可能造成嚴重業務問題。

## 七、考研觀念比較
**Cache Hit Ratio** 是命中次數除以總查詢次數，但不能直接推論正確性；**TTL** 限制快取項目的有效期，不保證更新立刻可見；**Cache-Aside** 是應用程式控制快取載入的模式；**Write-Through** 則是寫入流程同步經過快取與後端儲存，仍需考慮失敗與一致性語意。**Cache Stampede** 是大量 Miss 同時造成後端重複負載，不等於一般單次 Cache Miss。

### 自測題
某購物網站商品頁使用 60 秒 TTL 的 Cache-Aside，熱門商品每秒有大量讀取。請解釋：(1) 為什麼庫存顯示可能不正確？(2) 為什麼 TTL 到期時可能造成效能下降？(3) 提出兩項技術措施及一項 MIS 管理指標。

**參考答案：** (1) DB 更新與快取失效之間存在時間差，也可能有讀取舊值再填回的競態。(2) 同一熱門鍵同時失效，大量請求都 Miss，造成 Cache Stampede。(3) 使用 Single-flight 限制同鍵重建併發，並採用更新後失效／版本控制；最終庫存扣減仍由資料庫原子操作判定。管理指標可選過時庫存顯示率或因此引發的下單失敗率，而非只追求 Hit Ratio。

## 八、可重現的小型實驗（提案／待驗證）
設計一個 Node.js + Redis + PostgreSQL 的虛擬商品查詢 API，分成無快取、Cache-Aside、Cache-Aside 加 Single-flight 三組。在固定資料集下產生穩定與突發讀取負載，測試熱門鍵同時過期，另模擬資料庫更新與快取刪除交錯。

每組至少重複多輪，記錄 Hit Ratio、P50/P95 延遲、DB 查詢次數、過時讀取比例、測試環境資源使用率。測試應固定硬體、資料量、並發數及負載產生器，分開比較冷快取與暖快取。保留程式碼、SQL、Redis 設定、測試指令、原始 CSV、監控截圖及失敗案例，並用表格清楚區分「預期」與「實測」。

**目前沒有執行任何實驗；沒有實測數據，也不代表使用者已實作這個系統。**

## 結論
快取不只是「把資料放在記憶體裡」。真正的系統設計必須回答：哪些資料可以過時？過時多久可接受？快取失效時能否保護資料庫？哪一個元件擁有最終交易判定權？把一致性、延遲、成本與顧客風險放在同一張決策表，才是 MIS 與 CS 的整合。

## References
1. [Redis — Key expiration](https://redis.io/docs/latest/develop/use/keyspace/)
2. [AWS Builders' Library — Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)
3. [MDN — Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
4. [PostgreSQL — UPDATE](https://www.postgresql.org/docs/current/sql-update.html)
5. [PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)


## SEO 與作品集資訊
- SEO Title：Cache-Aside 與 TTL 怎麼設計？從快取不一致到 Cache Stampede｜MIS × CS
- SEO Description：快取讓網站變快，卻可能讓庫存顯示過時或造成熱門鍵過期時的資料庫尖峰。從 Cache-Aside、TTL、Single-flight 到資料庫交易，理解效能與正確性的取捨。
- Slug：cache-aside-ttl-stampede-inventory-consistency
- Categories：MIS × CS、系統設計、考研強化
- Tags：MIS、CS、Caching、Redis、Cache-Aside、TTL、Cache Stampede、PostgreSQL
- 書審作品標題：從庫存資訊落差到快取架構取捨：以 Cache-Aside 與 Single-flight 設計可重現驗證
- 書審摘要：針對電商商品庫存顯示延遲與熱門快取同時失效問題，提出 Cache-Aside、TTL 與 Single-flight 的比較設計。規劃以 Redis、Node.js 與 PostgreSQL 模擬穩定及突發流量，驗證延遲、資料庫負載與過時讀取比例，並評估效能與業務正確性的取捨。實驗待執行，預期藉可重現證據呈現跨領域系統分析能力。
- 圖表構想：快取失效與資料庫更新時間線；三種架構的 P95 延遲、DB QPS 與過時讀取比例比較（待實測）
- 建議書審位置：自主學習／專業作品集；代表作候選（須有實驗證據後再升級）
