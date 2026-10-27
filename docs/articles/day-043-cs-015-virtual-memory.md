---
layout: default
title: "Virtual Memory 虛擬記憶體完整解析：Demand Paging、Page Fault、FIFO／LRU、Thrashing"
description: "從 App 記憶體需求切入，說明 Demand Paging、Page Fault、TLB Miss、FIFO／LRU、Belady 異常、Working Set 與 Thrashing。"
date: 2026-10-27
categories: [計算機概論]
tags: [Operating Systems, Virtual Memory, Demand Paging, Page Fault, FIFO, LRU, Belady Anomaly, Thrashing, Working Set]
---

# 計概 #015｜RAM 不夠為什麼程式還能執行？從 Demand Paging、Page Fault 到 FIFO、LRU 與 Thrashing 看懂虛擬記憶體

> 同時打開瀏覽器、遊戲和編輯器，工作管理員顯示的程式記憶體需求可能非常大。為什麼程式還能順利切換，甚至在 RAM 緊張時繼續跑？關鍵不是「硬碟免費變成 RAM」，而是作業系統**只把眼前需要的資料頁留在實體記憶體，並按需調度其他頁面**。這就是理解 Virtual Memory（虛擬記憶體）的入口。

## 一、承接記憶體管理：位址空間不等於已占用的 RAM

上一篇討論 Paging、Page Table 與 MMU，回答「程式的虛擬位址如何對應實體頁框」。**Virtual Memory** 更進一步問：「一個行程所能使用的虛擬位址空間，是否每一頁都必須同時常駐 RAM？」答案是不必。

作業系統可先建立虛擬位址範圍及映射關係，再依實際存取分配實體頁、載入檔案內容，必要時回收或移出其他頁。**已保留的虛擬空間、已提交的配置、目前常駐的實體頁與交換空間使用量，是不同概念**；各作業系統的記憶體計量名稱也不完全相同。

以遊戲載入大型地圖為例，程式可能需要訪問大量貼圖與資料，卻不必把所有檔案內容同時載入 RAM。這種把「程式看到的記憶體」與「當下實際所在的位置」分開管理的能力，是虛擬記憶體的價值。[Stanford CS111 — Paging](https://www.web.stanford.edu/~ouster/cs111-spring23/lectures/paging/)

## 二、Demand Paging：第一次碰到某一頁才處理

**Demand Paging（需求分頁／按需調頁）**的核心是：沒有被使用的頁，不一定要先放進 RAM。典型流程可拆成六步：

1. CPU 以虛擬位址讀寫資料，MMU 檢查頁表或快取的位址轉換資訊。
2. 若頁未常駐且這次存取需要作業系統介入，硬體觸發 **Page Fault（缺頁例外）**。
3. 核心確認位址和存取權限是否合法；違法存取不能靠「補一頁」解決。
4. 若存取有效，核心可能配置零填充的新頁，或從執行檔、記憶體映射檔案、交換區等來源取得資料。
5. 若缺乏可用頁框，可能先回收另一頁；必要時將被修改過、需要保留的內容寫回適當的後備儲存。
6. 更新映射與相關狀態，讓原本的指令重新執行或繼續。

**Page Fault 不一定會讀硬碟**。例如第一次存取匿名記憶體時可能只需分配並清零頁面；已在系統快取中的檔案頁也可能不需額外磁碟 I/O。反過來，非法存取也可能產生頁面錯誤，但會導向錯誤處理，而非成功分頁。MIT xv6 的 Lazy Allocation 實驗就是經典範例：請求堆空間時先不配實體頁，第一次存取才由 fault handler 建立映射。[MIT xv6 — Page Faults](https://mit-pdos.github.io/xv6-riscv-book/pgfault.html)

> 💡 **常考辨析：TLB Miss ≠ Page Fault。** TLB Miss 表示快速位址轉換快取沒有命中；硬體或核心仍可能從有效頁表找到映射，完全不需載入資料頁。Page Fault 則代表此次位址存取需要例外處理，原因可包括頁未常駐或權限錯誤。

## 三、缺頁為什麼貴？用 EAT 算一次

考研常見公式是 **Effective Access Time（EAT，有效記憶體存取時間）**。在簡化「每次正常存取 100 ns，發生一次需花 8 ms 處理的缺頁」的模型中，若缺頁率 p＝0.0001：

```txt
EAT = (1-p) × 100 ns + p × (8,000,000 ns + 100 ns)
    ≈ 900 ns
```

即使只有萬分之一的機率碰到高成本缺頁，平均時間仍可能從約 100 ns 增加到約 **900 ns，接近九倍**。這是**示意算式**，未計入多層快取、不同 fault 類型、佇列等待等真實系統因素；正式解題要先看題目有沒有把原本記憶體存取時間包含在 fault service time。

## 四、RAM 滿了怎麼選？FIFO、OPT、LRU

假設只有固定數量的實體頁框（Frame），遇到缺頁而且沒有空位，就得選一個 **Victim Page（被替換頁）**。三個觀念先抓住：

- **FIFO（First-In, First-Out）**：最早載入的頁先離開。簡單，但「住得久」不等於「最近不常用」；再次命中時**不**改變它的進入順序。
- **OPT／MIN（理論最佳）**：換掉「從現在起，最晚才會再次使用」的頁。可當最佳解下界，但通常無法預知實際未來存取。
- **LRU（Least Recently Used）**：換掉「距離上一次使用最久」的頁。再次命中**會**更新最近使用順序；精確追蹤所有存取可能有成本，實務常採近似法。

下面以 Cornell CS 4410 的經典 Reference String（頁面參考字串），從**全部頁框初始為空**開始計算：

```txt
1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5
```

| 演算法 | 3 個 Frame | 4 個 Frame |
| --- | --- | --- |
| FIFO | 9 次缺頁 | 10 次缺頁 |
| LRU | 10 次缺頁 | 8 次缺頁 |

這裡特別值得注意：**在 3 個 Frame 的例子中，FIFO（9 次）甚至比 LRU（10 次）少一次，不能說 LRU 對每一組參考字串必然都較好。** 但擴充到 4 個 Frame 時，FIFO 反而從 9 次變成 10 次，這就是 **Belady's Anomaly（Belady 異常）**。LRU 屬於具有包含性質的 Stack Algorithm，不會發生這個異常；這是關於增加頁框後缺頁數不會變多的性質，**不是保證 LRU 一定贏過 FIFO**。[Cornell CS4410 — Page Replacement](https://www.cs.cornell.edu/courses/cs4410/2015su/lectures/lec15-replacement.html)

## 五、Second Chance／Clock 為什麼常出現在課本？

FIFO 看不出頁最近是否有用；精確 LRU 又可能需要維護太多歷史資訊。**Second Chance（第二次機會）**會檢查參考位元（Reference Bit）：遇到被標記使用過的頁，先清除參考位元並略過，讓它暫時保留。**Clock** 常用環狀指標實現類似策略。

Clock 是近似使用近期性的經典教學演算法，**不是所有現代作業系統都照課本的 Clock 程式碼做回收**。例如 Linux 官方文件記載 Multi-Gen LRU 以不同世代近似存取新近程度、管理頁面回收；實際策略遠比三頁框手算題複雜。[Linux Kernel — Multi-Gen LRU](https://docs.kernel.org/mm/multigen_lru.html)

## 六、Thrashing：CPU 忙著換頁，卻沒空做正事

假設編輯影片、開瀏覽器與跑大型遊戲，每個程式都有近期反覆使用的資料頁。如果這些程式的 **Working Set（工作集合）**總需求超過系統能供給的實體記憶體，剛換出去的頁可能馬上又被需要，缺頁率和儲存 I/O 大幅上升。系統花大量時間調頁、有效運算反而下降，稱為 **Thrashing（抖動／顛簸現象）**。

它不是「所有卡頓都是 RAM 不足」，也不是「CPU 使用率越高就一定越 Thrashing」。可能的改善方法包括減少同時執行的高記憶體工作、控制多道程式程度、增加可用 RAM，或降低程式的實際工作集合。重點是讓經常被存取的頁有較高機會留在實體記憶體。[University of Iowa — Working Set and Thrashing](https://homepage.divms.uiowa.edu/~jones/opsys/fall12/notes/22.shtml)

## 七、考研常見陷阱與申論寫法

**題一：Page Fault 一定要從硬碟讀頁嗎？** 不一定。零填充、快取中的檔案頁、保護例外都可能涉及 fault 機制，需看具體原因。

**題二：FIFO 的 4 個 Frame，一定比 3 個 Frame 缺頁更少？** 錯。可用上面的參考字串說明 Belady's Anomaly；LRU 則不會有此異常。

**題三：LRU 和 FIFO 差在哪個動作？** 「命中頁面後」：FIFO 不改進入順序，LRU 要更新最近使用順序。算題時每次命中也必須記錄。

**題四：多配一些虛擬位址空間就等於馬上拿走 RAM 嗎？** 不一定。記憶體預留、提交與實體駐留分屬不同層次，需依 API 與系統語意判斷。

**申論題架構：** 先畫出虛擬頁與實體頁框 → 說明 Demand Paging 與 Page Fault 的合法／非法分流 → 在頁框不足時介紹 Victim 選擇 → 比較 FIFO、OPT、LRU → 用 Belady's Anomaly 和 Working Set 解釋效能問題 → 點出 Linux 真實回收策略與教學模型的差別。

## 結論：不是讓硬碟假裝成 RAM，而是讓熱門資料留在 RAM

記憶體管理解決「哪些位址能用、怎麼映射」，虛擬記憶體接著解決「哪些頁現在必須待在實體記憶體」。Demand Paging 延後不必要的載入，Page Fault 讓系統按需處理；FIFO、LRU、Clock 嘗試在頁框緊張時做取捨；Working Set 則提醒我們，**選對淘汰策略，也不能彌補長期不足的實體記憶體容量**。

下一篇三科輪替為 **Day 44｜資料結構 #015｜Symmetric Matrix（對稱矩陣）**。

---

## References

1. [Stanford CS111 — Paging and Virtual Memory](https://www.web.stanford.edu/~ouster/cs111-spring23/lectures/paging/)
2. [MIT xv6 Book — Page Faults, Lazy Allocation](https://mit-pdos.github.io/xv6-riscv-book/pgfault.html)
3. [Cornell CS4410 — Page Replacement Algorithms](https://www.cs.cornell.edu/courses/cs4410/2015su/lectures/lec15-replacement.html)
4. [University of Iowa — The Working Set Concept](https://homepage.divms.uiowa.edu/~jones/opsys/fall12/notes/22.shtml)
5. [Linux Kernel Documentation — Multi-Gen LRU](https://docs.kernel.org/mm/multigen_lru.html)
6. [University of Wisconsin — OSTEP Open Textbook Index](https://pages.cs.wisc.edu/~remzi/OSTEP/)
