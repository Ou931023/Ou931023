---
layout: default
title: "Critical Section 臨界區是什麼？Race Condition、Mutex、Semaphore 與 Peterson 演算法一次看懂"
description: "兩個 Thread 同時更新共享變數，為何會漏算或超賣？從臨界區三大要求、Peterson 演算法、原子操作、Mutex 和 Semaphore，理解 OS 如何維護共享資料一致性。"
date: 2026-10-21
categories: [計算機概論]
tags: [Operating Systems, Critical Section, Race Condition, Mutex, Semaphore, Peterson Algorithm, Atomicity]
---

# 計概 #013｜多執行緒同時搶一個名額，為何會超賣？

> 票務網站只剩最後一張票，兩個人同時按「購買」。若兩個請求都讀到「剩餘 1 張」，也都判斷可以購買，系統就可能賣出兩張。問題不在伺服器算得不夠快，而在於**兩段看似合理的程式同時操作共享資料時，沒有被正確協調**。作業系統的 Critical Section（臨界區）要解決的，正是這種問題。

## 一、什麼是 Critical Section？先找出會互相干擾的資料

**Critical Section（臨界區）**是程式中存取共享資源，而且與其他執行流程並行執行可能破壞正確性的程式區段。共享資源可能是記憶體中的計數器、佇列、檔案，也可能是資料庫的一筆庫存。不是所有程式碼都需要加鎖；真正要保護的是「同時操作會違反不變條件」的部分。

經典臨界區問題可分成四段：

1. **Entry Section（進入區）**：取得進入臨界區的許可，例如 Lock。
2. **Critical Section（臨界區）**：讀取、判斷與修改受保護的共享狀態。
3. **Exit Section（離開區）**：釋放許可，讓其他執行流程進入。
4. **Remainder Section（剩餘區）**：不需要受同一把鎖保護的其他工作。

以售票為例，讀庫存、確認剩餘數量、扣庫存必須視為一個協調單位。畫面繪製、寄通知等耗時工作通常不該無故佔住同一把鎖。

## 二、Race Condition：兩個加一，結果為何可能只加一次？

假設共享變數 count = 5，Thread A 與 Thread B 都執行 `count++`。這個運算在一般情況下不是不可分割的一步，而可概念化為「讀取 → 加一 → 寫回」。可能的交錯順序：

1. A 讀到 5。
2. B 也讀到 5。
3. A 計算 6 並寫回。
4. B 計算 6 並寫回。

最後是 6，不是預期的 7；B 覆蓋掉 A 的更新，稱為 **Lost Update（遺失更新）**。Oracle 官方教學用相同原理解釋 Thread Interference。這類錯誤不一定每次發生，因為不同執行時序可能得到不同結果。[Oracle — Thread Interference](https://docs.oracle.com/javase/tutorial/essential/concurrency/interfere.html)

**Race Condition（競爭狀況）**泛指正確性或結果依賴不受控的時序；更精確的 **Data Race** 通常涉及缺乏必要同步的衝突記憶體存取。兩者相關，但不能一律畫上等號。

## 三、合格的臨界區解法，要同時滿足哪三個條件？

**Mutual Exclusion（互斥）**：在同一時間，最多只能有一個競爭者執行同一受保護資源的臨界區。這避免兩人同時改寫庫存。

**Progress（進展）**：如果沒有人在臨界區、又有人想進去，不應讓無關的流程無限拖延誰能進入的決定。否則雖然「沒有人同時進去」，系統仍可能停滯。

**Bounded Waiting（有限等待）**：當某個流程提出進入要求後，其他流程可以搶先進入的次數必須有上界，避免某個等待者無止境被插隊。此處強調公平性，不能單靠「有鎖」就保證。

考題常把三者混淆：**互斥是安全性（Safety），進展與有限等待是活性／公平性的要求**。一個解法即使保證互斥，也不必然滿足有限等待。

## 四、Peterson's Algorithm：只靠軟體協調兩個流程

經典 Peterson 演算法使用兩個共享陣列旗標 `flag[0]`、`flag[1]`，加上變數 `turn`。流程 i 想進入時，先將自己的旗標設為 true，再把優先權讓給對方 j；當對方也想進入且輪到對方時，就等待：

```txt
flag[i] = true
turn = j
while (flag[j] && turn == j) { /* wait */ }
// critical section
flag[i] = false
```

若只有 A 想進入，條件不成立就能進去；若雙方都想進入，`turn` 可以打破僵局。**它的價值在於練習互斥、進展與有限等待的推理，而不是推薦直接用這段程式寫正式系統**。經典正確性依賴兩個競爭者、原子讀寫及適當記憶體順序等假設；現代 CPU 和編譯器可能重新排序記憶體操作，不能用一般變數直接假定程式可攜而正確。

## 五、為何「先檢查 lock，再把 lock 設為 1」仍不安全？

想像兩個 CPU 幾乎同時看到 `lock == 0`，於是都把它改成 1，都以為自己取得了鎖。問題是**檢查與設定分成兩步**；這段檢查本身也會遭遇 Race Condition。

現代硬體與程式庫提供 **Atomic Read-Modify-Write** 操作，例如 Test-and-Set、Swap 或 Compare-and-Swap（CAS），把競爭判斷和狀態修改組成不可被其他核心插入的原子步驟，再搭配正確的記憶體順序。MIT xv6 的鎖實驗刻意把原子設定改成普通讀寫，正是要觀察為何互斥會失效。[MIT 6.1810 — Locks Lab](https://pdos.csail.mit.edu/6.S081/2026/labs/lock.html)

但是**原子操作不等於整個演算法自動正確**。單一計數器可以用 Atomic Increment；若必須一起維護「餘額不得為負、兩個帳戶總額守恆」，仍要協調一組操作的整體不變條件。

## 六、Mutex、Semaphore、Spinlock：名稱相近，目的不同

**Mutex（互斥鎖）**重點在唯一持有者：需要修改同一共享狀態的流程先取得同一把鎖，結束後釋放。這也可以建立跨執行緒的記憶體可見性關係。例如 Java 的 `synchronized`，不只避免同一物件上的相關操作交錯，也讓解鎖與之後上鎖之間建立 happens-before 保證。[Oracle — Synchronized Methods](https://docs.oracle.com/javase/tutorial/essential/concurrency/syncmeth.html)

**Semaphore（號誌）**持有的是可用許可數；計數號誌初值 3，適合限制「最多三個工作同時使用某資源」。二元號誌也能實作類似互斥的效果，但其所有權規則、使用慣例不應直接與 Mutex 混為一談。只有一個廁所隔間的鑰匙像 Mutex；停車場只剩三個空位的計數器像 Counting Semaphore。

**Spinlock（自旋鎖）**在無法取得時反覆檢查，短暫等待可能省掉睡眠／喚醒成本，卻會耗用 CPU；需要長時間等待的區段通常更適合會阻塞／睡眠的鎖。MIT xv6 核心教學用實際的自旋次數衡量鎖競爭，並探討縮小鎖範圍以改善並行度。[MIT xv6 Locks Lab](https://pdos.csail.mit.edu/6.S081/2026/labs/lock.html)

**不要以為 Lock 越多越安全。** 鎖範圍過大會犧牲平行度；多把鎖取得順序不一致，又可能產生上一篇學過的 Deadlock（死結）。

## 七、實務案例：Node.js 後端的庫存與 PostgreSQL

假設 Node.js 伺服器用兩個請求處理同一筆庫存，先各自 `SELECT stock`，再用舊值計算後 `UPDATE`，依交易隔離與寫法，仍可能發生遺失更新。**Node.js 單一事件迴圈不代表跨非同步請求、跨多個伺服器實例、跨資料庫交易都沒有競爭**。

一種可行做法是把「檢查剩餘量並扣一」合成資料庫中的條件式更新：

```sql
UPDATE products
SET stock = stock - 1
WHERE id = $1 AND stock > 0
RETURNING stock;
```

這個單一 SQL 更新使扣庫存與檢查條件在資料庫的並行控制下完成；沒有回傳資料列代表沒有符合條件的商品。若是**跨多筆資料的一致性操作**，則要依需求使用交易、適當的隔離等級或 `SELECT ... FOR UPDATE`，並處理死結／交易重試。PostgreSQL 官方文件詳細列出 Row-Level Lock 與其他鎖模式。[PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)

此案例是概念示範，不表示任何特定專案已採用此實作。

## 八、考研容易失分的五個判斷

1. **「單行 ****`x++`**** 一定是 Atomic？」錯。** 單行語法不代表底層單一步驟。
2. **「只要 Mutual Exclusion 就一定沒有 Starvation？」錯。** 互斥不保證有限等待。
3. **「Peterson 可不經修改直接放在任何多核心程式？」錯。** 要檢查競爭者數量與記憶體模型。
4. **「Semaphore 初值 4 等於只能一個人進入？」錯。** 在正確使用下，可同時發出四個許可。
5. **「使用 Mutex 就不可能 Deadlock？」錯。** 相反取鎖順序可能製造循環等待。

**申論題建議順序：** 先定義共享資料與 Race Condition → 標出 Entry／Critical／Exit／Remainder → 說明三大正確性條件 → 用互斥鎖或原子操作提出解法 → 討論等待、公平性、效能與死結風險。遇到比較題，先區分「互斥」和「可同時通行名額」。

## 結論：並行真正難的不是同時跑，而是維持不變條件

Critical Section 的核心不是讓所有工作排隊，而是找出不能被任意交錯的共享狀態修改，選擇剛好足夠的同步範圍。理解 Lost Update、三項臨界區要求、Peterson 假設以及 Mutex、Semaphore、Atomic 操作的差別，才能回答「為什麼電腦明明兩次加一，最後卻只加了一次？」下一步將以此觀念銜接記憶體管理與更多並行程式問題。

---

## References

1. [Oracle Java Tutorials — Thread Interference](https://docs.oracle.com/javase/tutorial/essential/concurrency/interfere.html)（以 Java 8 為背景的教學）
2. [Oracle Java Tutorials — Synchronization](https://docs.oracle.com/javase/tutorial/essential/concurrency/sync.html)
3. [Oracle Java Tutorials — Synchronized Methods](https://docs.oracle.com/javase/tutorial/essential/concurrency/syncmeth.html)
4. [Oracle Java Tutorials — Atomic Variables](https://docs.oracle.com/javase/tutorial/essential/concurrency/atomicvars.html)
5. [MIT 6.1810 / Fall 2026 — Lab: locks](https://pdos.csail.mit.edu/6.S081/2026/labs/lock.html)
6. [PostgreSQL Documentation — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
