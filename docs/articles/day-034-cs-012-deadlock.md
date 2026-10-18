---
layout: default
title: "Deadlock 死結是什麼？四必要條件、資源配置圖、銀行家演算法與 PostgreSQL 實例"
description: "兩個 Thread 都沒當機，為何永遠等不到對方？從 Deadlock 四必要條件、Resource Allocation Graph、Banker's Algorithm 到 PostgreSQL 交易死結，整理作業系統考研重點。"
date: 2026-10-18
categories: [計算機概論]
tags: [Computer Science, Operating System, Deadlock, Coffman Conditions, Resource Allocation Graph, Banker's Algorithm, PostgreSQL]
---

# 計概 #012｜兩個程式都沒當機，為何永遠等不到？從 Deadlock 四條件、資源配置圖到 PostgreSQL 看懂死結

> 兩位廚師同時做菜：甲拿著平底鍋，等乙手上的鍋鏟；乙拿著鍋鏟，又等甲的平底鍋。兩人都沒當機，廚房也正常，卻誰也完成不了。**電腦中的兩條 Thread 也可能各自持有一把鎖，卻同時等待對方的鎖。** 這就是 Deadlock（死結），不是把 CPU 換快就能解決。

## 一、Deadlock 和一般 Waiting 的差別

**Deadlock（死結）**是兩個或以上的 Process 或 Thread 形成互相依賴的資源等待，使相關工作無法繼續。常見資源包括 Mutex、檔案鎖及資料庫的列鎖。**Waiting 不等於 Deadlock**：下載檔案等待網路封包，封包仍可能抵達；死結卻是等到的資源只能由同樣被卡住的另一方釋放。

從上一篇「行程五狀態」來看，Waiting 只是目前缺少事件，並未說明事件是否永遠不會發生。觀察 Deadlock 應先問「誰已持有什麼、誰又在等誰」，而不是只看執行多久。鎖等待中的 Thread 往往被阻塞，不一定持續使用 CPU。

## 二、兩把鎖如何讓程式停止前進？

假設 Thread A 先取得 L1 再請求 L2，Thread B 卻先取得 L2 再請求 L1。若 A、B 各自先拿到第一把鎖，之後兩者都等對方放下第二把鎖，就出現 `A → B → A` 的等待環。

```text
Thread A                     Thread B
lock(L1)                     lock(L2)
lock(L2)  ← 等 B              lock(L1)  ← 等 A
處理工作並釋放鎖              處理工作並釋放鎖
```

**這段程式可能死結，但不是每次執行都必然死結。** 如果 A 在 B 取得 L2 前便完成兩項工作並釋放鎖，B 就可能正常繼續。這個差異與執行交錯（Interleaving）有關。[Oracle Java 官方 Deadlock 範例](https://docs.oracle.com/javase/tutorial/essential/concurrency/deadlock.html)提供類似情境，但該教學以 JDK 8 為背景，不代表最新 Java 全部語法特性。

## 三、Coffman 四必要條件

典型資源死結模型必須同時滿足：

1. **Mutual Exclusion（互斥）**
2. **Hold and Wait（占有並等待）**
3. **No Preemption（不可搶奪）**
4. **Circular Wait（循環等待）**

破壞任一條件，就能阻止典型死結。

## 四、Resource Allocation Graph

- **P → R（Request Edge）**：P 正在請求資源 R。
- **R → P（Assignment Edge）**：資源 R 已配置給 P。

若每種資源只有一個實例，圖中出現 Cycle 可判定死結；若資源有多個實例，Cycle 只代表可能死結。

## 五、Prevention、Avoidance、Detection & Recovery

| 方法 | 核心思想 | 典型實務 |
| --- | --- | --- |
| Prevention | 破壞必要條件 | 固定順序取得鎖 |
| Avoidance | 僅允許安全狀態 | Banker's Algorithm |
| Detection & Recovery | 先允許，再偵測解除 | 回滾交易 |
| Ignore | 接受低機率風險 | 依情境取捨 |

## 六、Banker's Algorithm：Unsafe 不等於 Deadlocked

基本式：`Need = Max - Allocation`。

Safe State 表示存在能讓所有 Process 完成的安全序列；Unsafe State 僅表示無法保證，不等於已經 Deadlocked。

## 七、PostgreSQL 為什麼會主動終止交易？

[PostgreSQL 18 官方文件](https://www.postgresql.org/docs/current/explicit-locking.html)說明，資料庫會自動偵測死結並中止其中一個交易，讓其他交易繼續。官方建議多個資源應按**一致順序取得鎖**，並在死結中止時**重試完整交易**。

`deadlock_timeout` 是啟動死結檢查前的等待時間；官方預設為 **1 秒**，不是 SQL 查詢只能執行 1 秒。

## 八、Deadlock、Starvation、Livelock、Race Condition

- **Deadlock**：形成無法自行解除的資源等待圈。
- **Starvation**：某工作長期拿不到資源，但其他工作可能持續完成。
- **Livelock**：工作不停退讓或重試，卻沒有實質進展。
- **Race Condition**：結果取決於操作時序。

## 九、考研重點

- Waiting ≠ Deadlock
- RAG 有 Cycle 不一定永遠等於 Deadlock
- 固定取鎖順序可破壞 Circular Wait
- Unsafe State ≠ Deadlocked State

## 結論

Deadlock 是**資源依賴與取得順序**問題，不是單純效能不足。先看清誰持有什麼、誰等待什麼，再用四必要條件與資源配置圖判讀。實務上可固定鎖順序、檢查安全序列，或在資料庫中偵測後回滾並受控重試。

---

## References

1. [OSTEP — Common Concurrency Problems](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-bugs.pdf)
2. [Oracle Java Tutorials — Deadlock](https://docs.oracle.com/javase/tutorial/essential/concurrency/deadlock.html)
3. [PostgreSQL 18 — Explicit Locking / Deadlocks](https://www.postgresql.org/docs/current/explicit-locking.html)
4. [PostgreSQL 18 — Lock Management](https://www.postgresql.org/docs/current/runtime-config-locks.html)
