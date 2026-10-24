---
layout: default
title: "Memory Management 記憶體管理是什麼？虛擬位址、MMU、Paging、Segmentation 與碎裂一次看懂"
description: "為什麼兩個程式使用相同記憶體位址卻不會互相覆蓋？從 Logical／Physical Address、Base／Limit、記憶體碎裂、Segmentation、Paging 到頁表、TLB 與 4 KiB 位址換算。"
date: 2026-10-24
categories: [計算機概論]
tags: [Operating Systems, Memory Management, MMU, Address Translation, Paging, Segmentation, Fragmentation, Page Table, TLB]
---

# 計概 #014｜兩個程式都用同一個記憶體位址，為何不會互相覆蓋？

> 你同時開啟瀏覽器、遊戲和通訊軟體，它們可能都在自己的程式裡使用「位址 0x00400000」。如果每個位址都直接指向同一條實體 RAM 位置，程式不就互相覆蓋了嗎？**作業系統的記憶體管理，核心不是只把 RAM 分出去，而是讓每個行程有可用、可保護、可轉換的位址空間。**

## 一、為什麼作業系統要管理記憶體？

CPU 執行指令時要取得程式碼和資料，RAM 就是重要的存放場所。但作業系統同時面臨三個問題：**配置（Allocation）**——誰可以用哪些空間？**保護（Protection）**——程式能否寫到別人的資料？**效率（Efficiency）**——空間不連續或程式不再使用時怎麼回收？

想像旅館：每位住客拿到自己的房號，不必知道實體鑰匙櫃放在哪裡；旅館要確保房間不被別人誤闖，也不能因退房留下零碎空房，就永遠拒絕新客人。記憶體管理也是類似的「抽象、隔離、重新安排」。

## 二、Logical／Virtual Address 與 Physical Address：相同數字不等於同一位置

**Logical Address（邏輯位址）**通常指程式或 CPU 產生、尚待轉換的位址；在一般作業系統教學中常與 **Virtual Address（虛擬位址）**一起討論。**Physical Address（實體位址）**則是存取實體記憶體所使用的位址。依硬體架構，邏輯位址與虛擬位址的精確術語可能不同，考題應以題目定義為準。

**MMU（Memory Management Unit）**是協助位址轉換與權限檢查的硬體。每個行程可以在自己的位址空間看見相似的地址，例如 A、B 都使用虛擬位址 0x1000，但它們的頁表可以分別映射到不同實體頁框；反過來，作業系統也能有意把同一實體頁映射給多個行程，提供共享記憶體。

先用簡化的 **Base／Limit（基底／界限）**理解：假設某行程合法邏輯位址介於 0 到 0x0FFF，base = 0x4000。存取 0x0A20 時，先確認小於 limit = 0x1000，再計算實體位址：

- 邏輯位址：0x0A20
- 實體位址：0x4000 + 0x0A20 = **0x4A20**
- 若嘗試 0x1200，已超出界限，應觸發保護例外，而不是照樣相加。

這是**教學用的連續配置模型**，現代主流系統常用分頁機制，不是每次存取都只做一次 base 加法。美國威斯康辛大學的開放教材 OSTEP 把 Address Spaces、Address Translation、Segmentation、Paging 依先備觀念安排，適合照此順序理解。[OSTEP—Virtualization 章節索引](https://pages.cs.wisc.edu/~remzi/OSTEP/)

## 三、一個行程的記憶體長什麼樣？

典型概念圖會畫出 **Text／Code（程式碼）、Data／BSS（資料）、Heap（動態配置）和 Stack（函式呼叫堆疊）**。Heap 和 Stack 的成長方向常被畫成相對，但實際配置、成長方式與位址布局取決於平台、執行檔、函式庫、ASLR 等因素，**不能把課本示意圖當成所有電腦固定的實體 RAM 排列**。

更重要的是：看到一大片「可用的虛擬位址範圍」，不等於 RAM 已經立刻分配相同大小。Linux 的 `mmap()` 會建立行程虛擬位址空間中的映射，可以對應檔案或匿名映射；位址保留、實體頁配置與資料載入是不同階段。[Linux man-pages—mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html)

## 四、連續配置為什麼產生 Fragmentation？

如果每個行程都必須占有一大塊**連續實體記憶體**，設計直觀，但長期配置與釋放會遇到碎裂。

**Internal Fragmentation（內部碎裂）**：已配置區塊內有用不到的空間。例如固定分配 16 KiB、行程只用 10 KiB，剩下 6 KiB 在該區塊內閒置。**External Fragmentation（外部碎裂）**：空閒總量可能足夠，但分散成不相鄰的小洞。例如兩處空閒分別 5 KiB 和 7 KiB，合計 12 KiB，卻無法直接滿足需要單一連續 10 KiB 的要求。

First Fit、Best Fit 等方法決定如何挑選空洞；**Compaction（緊縮）**試圖移動配置讓洞連在一起，卻要付出搬移及位址更新成本。請注意：內部與外部碎裂描述的是不同浪費位置，不是「剩幾 KiB」就能互換稱呼。

## 五、Segmentation 與 Paging：解決問題的方法不同

**Segmentation（分段）**依程式的邏輯單位分配變動大小的區段，例如 Code、Heap、Stack 各自具有 base、limit 及權限。這樣可以分別保護、共享不同區段，但區段大小不固定，仍可能遇到外部碎裂。

**Paging（分頁）則把虛擬位址空間切成固定大小的 Page（頁），把實體記憶體切成相同大小的 Frame（頁框）。一個行程的第 0、1、2 頁可以分別放在不連續的實體頁框，因此不必替整個行程找到一塊連續的 RAM。分頁可避免此類連續配置造成的外部碎裂，代價包括頁表空間、位址轉換成本，以及最後一頁可能未用滿的內部碎裂**。

### 算給你看：4 KiB 頁面如何轉換位址？

以下假設頁面大小是 **4 KiB = 4096 = 2¹² bytes**；它是常見設定，不代表所有平台都只有這個頁面大小。

虛擬位址可拆成 **Page Number（頁號）＋Offset（頁內位移）**。對位址 `0x2ABC`：

1. 頁號 = 2，頁內位移 = `0xABC`（因為每頁大小 `0x1000`）。
2. 假設頁表把虛擬第 2 頁映射到實體頁框 **9**。
3. 實體位址 = `9 × 0x1000 + 0xABC = 0x9ABC`。

**頁號可變、Offset 不變**，因為轉換的是頁面所在位置，不是頁面內的每個 byte。MIT xv6 的 Page Tables Lab 會讓學生實際檢視 VA、PTE、PA 與 permission bits；它也提醒虛擬頁面不必連續對應到實體頁框。[MIT 6.1810—Page Tables Lab（2025）](https://pdos.csail.mit.edu/6.1810/2025/labs/pgtbl.html)

## 六、Page Table 不只是查表，還負責保護

**Page Table（頁表）**紀錄虛擬頁與實體頁框的對應及必要的狀態／權限資訊。除了轉換位址，也可以限制是否可讀、可寫、可執行；因此相同虛擬位址在不同行程中，不會自然取得對方的私有資料。

想像簡化的 **32-bit 虛擬位址＋4 KiB 頁面**：Offset 是 12 bits，頁號便是 20 bits，最多有 **2²⁰ 個虛擬頁**。若天真地為每頁都準備一筆 4-byte PTE，單一完整線性頁表就需要 **2²⁰ × 4 bytes = 4 MiB**，即使大量位址沒有使用。這正是為何多層頁表與只為需要的部分建立對應有價值。

**TLB（Translation Lookaside Buffer）**則快取最近使用的位址轉換，減少頻繁走訪頁表的成本。它不是拿來快取一般程式資料的 L1 Data Cache。Linux 官方說明了頁表層級、MMU、TLB 及權限檢查的關係；其五層頁表是 Linux 使用的抽象層級，實際硬體可依架構折疊層數，不能說每顆 CPU 都一定走五層。[Linux Kernel—Page Tables](https://cdn.kernel.org/doc/html/latest/mm/page_tables.html)

## 七、實務案例：一份檔案，兩種映射語意

Linux 的 `mmap()` 能將檔案對應到行程的虛擬位址空間。若使用 **MAP\_SHARED**，對共享映射的更新可被其他相應映射看見，並涉及回寫檔案的語意；使用 **MAP\_PRIVATE** 則是私有的 Copy-on-Write 映射，寫入不會直接修改底層檔案。搭配 `PROT_READ`、`PROT_WRITE` 等權限，能描述哪些操作應被允許。

這不表示共享映射自動解決所有並行更新問題：**記憶體能共享，不等於存取已同步**。上一篇 Critical Section 的 Mutex／Atomic 問題依然存在。[Linux man-pages—mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html)

## 八、考研五個常見陷阱

1. **虛擬位址等於實體位址？** 不一定；需看位址轉換機制與映射。
2. **Paging 完全沒有 Fragmentation？** 錯。典型分頁避免行程連續配置的外部碎裂，最後一頁仍可能有內部碎裂。
3. **Segmentation 是固定大小？** 錯。典型分段依邏輯區段，大小可以不同。
4. **4 KiB 頁面，Offset 幾 bits？** **12 bits**。若位址是 32 bits，頁號為 20 bits。
5. **Page Fault 一定代表硬碟太慢？** 錯。原因可能是尚未映射、權限違規或延遲配置，不一定需要磁碟 I/O；這將在虛擬記憶體篇進一步討論。

**申論題寫作順序：** 先寫配置／保護目的 → 區分虛擬與實體位址 → 用 Base／Limit 解釋最基本的轉換 → 比較 Segmentation 與 Paging 的碎裂 → 拆解 Page Number／Offset → 補充頁表與 TLB 的時間及空間取捨。若考到多工，再說明每個行程的位址映射如何維持隔離。

## 結論：程式不必知道 RAM 真正放在哪裡

記憶體管理的巧妙之處，是**程式看見一致、可操作的位址；硬體與作業系統負責把它轉成合法的實體存取**。從連續配置的碎裂，到分段、分頁與頁表的映射，每個設計都在權衡彈性、保護、空間與速度。理解這些基礎，下一篇的 Virtual Memory（虛擬記憶體）才能接上 Demand Paging、Page Fault 與 Swap，而不會把虛擬位址誤認成硬碟空間。

---

## References

1. [Arpaci-Dusseau & Arpaci-Dusseau, Operating Systems: Three Easy Pieces（OSTEP），章節 13–20](https://pages.cs.wisc.edu/~remzi/OSTEP/)
2. [MIT 6.1810, Lab: Page Tables（2025）](https://pdos.csail.mit.edu/6.1810/2025/labs/pgtbl.html)
3. [The Linux Kernel Documentation, Page Tables](https://cdn.kernel.org/doc/html/latest/mm/page_tables.html)
4. [Linux man-pages, mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html)
5. [The Linux Kernel Documentation, Memory Management](https://kernel.org/doc/html/v6.16/mm/index.html)
