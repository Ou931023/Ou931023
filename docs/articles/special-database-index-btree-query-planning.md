---
title: "資料庫加了索引為什麼查詢還是慢？從 B-tree、EXPLAIN ANALYZE 到 MIS 成本決策"
date: 2026-11-06
categories: [MIS, Computer Science, Database]
tags: [PostgreSQL, B-tree, EXPLAIN ANALYZE, Database Index, Query Optimization]
status: original_article
experiment_status: proposed_not_executed
---

# 資料庫加了索引為什麼查詢還是慢？從 B-tree、EXPLAIN ANALYZE 到 MIS 成本決策

> 電商客服要查詢「某位顧客最近 30 天的訂單」。資料庫有一百萬筆訂單，工程師建立索引後，查詢卻不一定變快；更糟的是，新增訂單的速度可能下降。索引不是免費的加速按鈕：它是一種用儲存空間與寫入維護成本，換取特定查詢效率的資料結構。

## 一、問題：查詢速度為什麼是管理議題？
想像企業的客服系統每次開啟顧客頁面，都要等待訂單清單。等待會影響服務效率，但為所有欄位建立索引，也會增加儲存與維護負擔。MIS 必須先定義服務水準（例如查詢延遲目標）、常見查詢、更新頻率與成本，再決定是否投入索引優化；CS 則負責說明資料結構與查詢計畫如何影響實際執行。

本篇先備知識是 SQL 的 SELECT、WHERE、ORDER BY，以及 Big-O 的基本概念。它與先前「Ordered List 與二分搜尋」不同：重點不只是理論搜尋次數，而是資料庫如何在磁碟頁面、快取、資料分布及讀寫負載之間選擇執行方式。

## 二、沒有索引：Sequential Scan 是什麼？
假設 orders 有 order_id、customer_id、created_at、total_amount 四個欄位。以下查詢要求某顧客的最新 20 筆訂單：

```sql
SELECT order_id, created_at, total_amount
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

若缺少合適索引，資料庫可能逐頁掃描資料表（Sequential Scan），找出符合條件的列，再排序並取前 20 筆。這不代表 PostgreSQL 一定逐筆讀取磁碟：作業系統與資料庫快取可能讓資料頁已在記憶體中。真正的執行成本要看查詢計畫與量測。

## 三、B-tree：為什麼索引不只是排序清單？
PostgreSQL 預設的 B-tree 索引適合等值與範圍比較，也能支援符合條件的排序。可把它想成多層的路標：先在上層節點縮小範圍，再前往包含索引項目的葉節點。相較於從頭掃描整張表，它可能大幅減少要檢查的資料頁。

但不要把資料庫 B-tree 直接等同於程式課上的「二元搜尋樹」。B-tree 每個節點可容納多個鍵值，設計上適合頁面式儲存；資料庫索引還涉及 MVCC 可見性、Heap 存取、快取與維護成本。教科書常以對數高度理解搜尋路徑，真實延遲卻不能只用 O(log n) 預測。

參考 [PostgreSQL Index Types](https://www.postgresql.org/docs/current/indexes-types.html) 與 [Introduction to Indexes](https://www.postgresql.org/docs/current/indexes-intro.html)。

## 四、複合索引：欄位順序決定適用情境
針對前面的查詢，可以考慮：

```sql
CREATE INDEX idx_orders_customer_created
ON orders (customer_id, created_at DESC);
```

這個複合索引先依 customer_id 分組，再依 created_at 的降冪方向組織索引順序。當 WHERE 先限定 customer_id，且 ORDER BY created_at DESC，規劃器可能利用索引順序，減少額外排序，並在找到足夠的列後停止。是否真的使用索引，仍要由規劃器根據統計資訊與成本估計判斷。

反過來，如果查詢只有 `WHERE created_at > ...`，這個索引不一定適合，因為第一個欄位是 customer_id。PostgreSQL 在某些條件下仍可能使用多欄 B-tree 的進階掃描策略，因此不能簡化成「沒有最左欄就絕對不能用」。

## 五、為什麼有索引仍可能選擇全表掃描？
第一，選擇率（Selectivity）不同：如果查詢要回傳全表很大比例的資料，走索引再回表可能比順序掃描更貴。第二，統計資訊可能過舊，導致規劃器估計錯誤。第三，查詢的條件、排序方向或型別轉換可能讓索引不適用。第四，索引本身也要占空間，INSERT、UPDATE、DELETE 可能必須維護索引項目。

因此，「索引存在」和「索引被使用」是兩件事；「索引被使用」與「端到端使用者體驗改善」又是兩件事。

## 六、EXPLAIN 與 EXPLAIN ANALYZE：如何拿證據說話？
`EXPLAIN` 顯示規劃器估計的執行計畫；`EXPLAIN ANALYZE` 會真正執行查詢並回報實際執行資訊。可用：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, created_at, total_amount
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

觀察 Seq Scan、Index Scan、Index Only Scan、Sort、實際列數、估計列數、Buffers 與 Execution Time。注意 EXPLAIN ANALYZE 本身有量測額外開銷，且執行計畫可能因資料量、統計資訊與快取狀態改變。它不是效能提升的保證書。請參考 [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)。

## 七、實驗設計：用數據決定索引是否值得（待驗證）
提案建立 10 萬與 100 萬筆虛擬訂單資料，設計三組測試：A 無索引；B 只有 customer_id 單欄索引；C 使用 (customer_id, created_at DESC) 複合索引。對相同查詢各重複執行至少 30 次，控制資料量、資料分布、硬體與資料庫設定，記錄延遲中位數、P95、計畫節點及 Buffer 命中情況。

另外以固定批次 INSERT 比較寫入耗時、索引大小與資料表大小。分別觀察冷／暖快取條件，避免只挑最快一次。使用 `ANALYZE orders;` 更新統計資訊後再測試，並記錄 PostgreSQL 版本、測試 SQL 與隨機資料種子。所有數值目前**尚未測量**，不能宣稱「索引讓查詢快十倍」。

可視覺化為一張「查詢 P95 延遲 vs. 批次寫入耗時」雙指標比較圖，再附查詢計畫截圖。若讀取改善卻使寫入負擔增加，MIS 應依業務高峰、SLA 與維運成本做取捨，而不是一味追求索引數量。

## 八、考研自測題與解析
**題目：** 已有 `CREATE INDEX ON orders (customer_id, created_at DESC)`，為何下列兩種查詢未必都能同樣受益？(A) `WHERE customer_id=42 ORDER BY created_at DESC LIMIT 20`；(B) `WHERE created_at >= '2026-01-01'`。

**參考答案：** A 的等值條件符合複合索引首欄，且排序方向與第二欄一致，通常較有機會使用索引順序快速取得前 20 筆。B 沒有首欄限制，資料庫可能需要掃描大量索引項目，或直接選擇 Sequential Scan；是否使用索引取決於資料分布、選擇率與成本估計。不能僅憑索引存在就推定查詢更快。

**延伸考點：** 比較 B-tree 與 Hash Index 的適用操作、Clustered 與 Nonclustered Index 的概念差異，以及 Big-O 理論與 I/O 成本模型的不同。

## 九、結論：好的索引設計是可驗證的取捨
索引的價值不在於建立了幾個，而在於是否改善重要的查詢，同時維持合理的寫入成本。MIS 提供需求與績效目標，CS 提供資料結構、SQL 與量測方法。將查詢計畫、效能實驗和業務成本放在一起，才是可向管理者解釋的資訊系統設計。

## References
1. [PostgreSQL — Introduction to Indexes](https://www.postgresql.org/docs/current/indexes-intro.html)
2. [PostgreSQL — Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
3. [PostgreSQL — Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
4. [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
5. [PostgreSQL — Indexes and ORDER BY](https://www.postgresql.org/docs/current/indexes-ordering.html)
6. [PostgreSQL — ANALYZE](https://www.postgresql.org/docs/current/sql-analyze.html)

## 書審作品定位
**作品標題：** 從客服查詢瓶頸到資料庫索引取捨：以 PostgreSQL 執行計畫設計可重現的效能驗證。

**作品亮點摘要（提案）：** 針對企業客服查詢延遲問題，規劃以 PostgreSQL 建立無索引、單欄索引與複合索引三組對照，透過 EXPLAIN ANALYZE 與重複測試觀察查詢延遲、I/O 與寫入成本。預計保留 SQL、測試日誌及圖表，呈現從業務需求、資料結構選型到效能取捨的整合分析能力。實驗尚未執行，成果待驗證。

**證據保留：** SQL 建表及產生資料腳本、EXPLAIN 輸出、版本與硬體環境、30 次以上測試原始紀錄、繪圖程式、分析限制。建議放在書審「自主學習／專業作品」章節，完成實測後再評估升格代表作。