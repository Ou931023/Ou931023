---
layout: default
title: "Sparse Matrix 稀疏矩陣怎麼存？COO 三元組、CSR 與 Fast Transpose 複雜度一次看懂"
description: "稀疏矩陣為何不該把 0 全存下來？以 4×5 範例比較 COO 三元組、CSR 的 indptr、indices、data，推導 Fast Transpose 時間複雜度。"
date: 2026-10-22
categories: [資料結構]
tags: [Data Structures, Sparse Matrix, COO, CSR, Triplet, Fast Transpose, Prefix Sum, Time Complexity]
---

# 資結 #013｜為什麼矩陣裡幾乎都是 0，電腦卻不必全部存？

> 假設有一張 10,000 × 10,000 的使用者互動表：每個人只和少數人有連結，其餘位置都是 0。若硬把一億個位置全部存起來，電腦多半是在替「沒有關係」浪費空間。**Sparse Matrix（稀疏矩陣）**的核心問題是：能不能只記下真正有資料的位置，又保留矩陣運算需要的行、列資訊？

## 一、Dense Matrix 與 Sparse Matrix，差別不是長相

一般二維陣列把每個位置都配置出來。若矩陣有 r 列、c 行（本文採「r 個 row、c 個 column」的記法），Dense Representation（稠密表示）需要 **r × c 個元素位置**，即使其中多數值為 0。

Sparse Representation（稀疏表示）改為記錄非零值及其座標。用 **nnz（Number of Non-Zero Entries）**表示非零項數，密度定義為 nnz ÷ (r × c)。當 nnz 遠小於 r × c 時，才有機會節省大量空間。Cornell 大學的數值分析課也指出，稀疏矩陣除了節省儲存，還能改善矩陣向量乘法等計算的成本。[Cornell CS 4220 — Data sparsity](https://www.cs.cornell.edu/courses/cs4220/2026sp/lec/2026-01-23.html)

**但「有很多 0」不等於一定省空間。** 因為每個非零值還得記錄位置，當矩陣不夠稀疏，座標成本可能吃掉節省的空間。

## 二、Triplet／COO：每個非零元素記三件事

先看這個 4 × 5 矩陣 A，索引由 0 開始：

```txt
       c0 c1 c2 c3 c4
row 0   0  0  8  0  0
row 1   5  0  0  0 -2
row 2   0  0  0  0  0
row 3   0  7  4  0  0
```

整張表有 20 格，真正非零只有 5 格。因此可依 row、column 順序儲存五個 **(row, col, value)**：

- (0, 2, 8)
- (1, 0, 5)
- (1, 4, −2)
- (3, 1, 7)
- (3, 2, 4)

這叫 **Triplet（三元組）**，也常稱 **COO（Coordinate Format）**。教科書的三元組表可能額外在首列放「總列數、總行數、非零項數」；實務函式庫通常把矩陣 shape 當獨立 metadata。兩種寫法不能混算。

從數量粗估：Dense 需要 20 個值；三元組要 5 × 3＝15 個欄位，看似較省。但實際位元組數還看數值型別、索引型別、記憶體配置與額外 metadata。**不要只用欄位數斷言壓縮比。**

SciPy 官方 COO 文件使用三個陣列分別存 row、col、data；它也允許相同座標重複出現，轉成 CSR／CSC 時通常會把重複位置的值加總。[SciPy — COO](https://docs.scipy.org/doc/scipy/reference/generated/scipy.sparse.coo_matrix.html)

## 三、CSR：為什麼每個項都記 row 可能有點浪費？

如果非零元素已依列排序，其實不必替每一項重複寫 row。**CSR（Compressed Sparse Row，壓縮稀疏列）**將矩陣拆成：

1. **data**：非零值，依列串接。
2. **indices**：每個值所在的 column。
3. **indptr**：各 row 在前兩個陣列中的起始偏移量，長度為 r+1。

剛才矩陣的 CSR 可以寫成：

```txt
data    = [8, 5, -2, 7, 4]
indices = [2, 0,  4, 1, 2]
indptr  = [0, 1,  3, 3, 5]
```

讀 row 1 時，查 indptr[1]＝1、indptr[2]＝3，就知道該列的資料位於半開區間 [1, 3)：(col 0, value 5) 與 (col 4, value −2)。row 2 的起訖都是 3，代表它是空列。**最後一個 indptr 數值必須等於已儲存項數。**

因此 CSR 儲存量在常見實作下是 O(nnz+r)：兩個長度 nnz 的陣列，加上一個長度 r+1 的列偏移量陣列。SciPy 文件明確定義第 i 列的索引範圍為 indices[indptr[i]:indptr[i+1]]，也指出 CSR 適合快速列切片與矩陣向量乘法，但對改動稀疏結構不友善。[SciPy — CSR](https://docs.scipy.org/doc/scipy/reference/generated/scipy.sparse.csr_matrix.html)

**COO 適合建立資料，CSR 適合重複依列計算**；若頻繁做 column 切片，可考慮把壓縮方向改成 CSC（Compressed Sparse Column）。Cornell 的稀疏矩陣課程也以 COO 與 CSR 比較建構成本和運算用途。[Cornell CS 4787 — Lecture 12](https://www.cs.cornell.edu/courses/cs4787/2023fa/lectures/lecture12.html)

## 四、Transpose：交換 row、column 就算完成了嗎？

轉置（Transpose）使 B[j, i]＝A[i, j]，因此原本 4 × 5 的矩陣會變成 5 × 4。對三元組而言，**交換座標值很容易，維持結果順序才是關鍵**。

原矩陣依 row-major 排序的三元組為：

```txt
(0,2,8) (1,0,5) (1,4,-2) (3,1,7) (3,2,4)
```

直接把 row 與 col 對調，雖然數學上已轉置，但順序變成 (2,0)、(0,1)、(4,1)、(1,3)、(2,3)，**並未依新 row 排列**。如果後續演算法假設資料已排序，就必須另外整理。

典型考題有兩種做法：

**普通轉置（Simple Transpose）**：從原矩陣 column 0 掃到 column c−1；每輪都走訪全部 nnz 項，挑出屬於該 column 的資料。結果自然按轉置後的 row 排序，但在這種寫法下時間為 **O(c × nnz)**。

**快速轉置（Fast Transpose）**：先統計每個原 column 的非零個數，再以 Prefix Sum 算出它在轉置結果中的起始位置，最後只走訪原三元組一次完成放置。若 c 為原 column 數、k＝nnz，時間為 **O(c+k)**，輔助計數與位置陣列為 O(c)，輸出本身為 O(k)。這個界限以已取得三元組作為輸入，不含從稠密矩陣讀入全部 r×c 格的成本。

以上例而言，各原 column 的項數為：

```txt
count = [1, 1, 2, 0, 1]
start = [0, 1, 2, 4, 4]
```

於是 B 的新 row 0 從位置 0 開始，row 2 從位置 2 開始，row 3 沒有資料。按每個 column 的下一個可寫位置填入，得到依新 row 排序的結果：

```txt
(0,1,5) (1,3,7) (2,0,8) (2,3,4) (4,1,-2)
```

這裡的 **count → prefix offset → scatter** 觀念，也是在許多資料重排與平行處理演算法中反覆出現的設計。請注意：如果 COO 本身已有重複座標或未先排序，輸出是否符合特定 canonical format 還取決於額外整理步驟。

## 五、Sparse 不代表任何操作都變快

假設要做 y＝Ax，x＝[1,2,3,4,5]，上述矩陣得到：

- y[0]＝8×3＝24
- y[1]＝5×1−2×5＝−5
- y[2]＝0
- y[3]＝7×2＋4×3＝26

用 CSR 可以逐列走訪非零項，不必對每列的五個 column 全掃。一般 CSR 乘稠密向量的計算可寫成 **O(nnz+r)**（包含初始化或輸出 r 個結果）；若只計非零元素上的乘加，則為 O(nnz)。但如果需要在 CSR 中間反覆加入新座標，維護連續陣列可能需要大量搬移。

另外，SciPy 的 nnz 在 API 語意上是**儲存的項數，可能包含明確存入的 0**；「儲存項數」不一定等於數學上的非零項數。若要得到乾淨的表示，還需注意 explicit zeros、重複座標及排序條件。[SciPy Sparse Tutorial](https://docs.scipy.org/doc/scipy/tutorial/sparse.html)

## 六、生活應用：社群圖、推薦與感測器

社群平台把使用者當節點、互動當邊，形成鄰接矩陣；若一個人只與少數人互動，多數位置是 0。推薦系統的使用者 × 商品互動表也是類似概念。稀疏表示只記錄存在的互動，適合進一步處理圖運算或矩陣向量乘法，但**未知評分不能不經定義就當成「明確評了 0 分」**。

另一個情境是感測器：若一萬個裝置只在少數時間槽產生事件，可記錄 (裝置、時間、讀值)。不過要先定義「沒有回報」表示 0、缺值還是離線，避免節省空間卻損害資料語意。

## 七、考研常見陷阱與題型

**題一：** 100 × 200 矩陣有 40 個非零元素，COO 至少記多少個三元組？**40 組**，另須保存 shape；不是 20,000 組。

**題二：** CSR 的 indptr 長度等於 nnz 嗎？**錯**，常見 CSR 是 row 數＋1；空列也必須由相鄰偏移量相等來表示。

**題三：** COO 每項 row、column 交換後就必然符合排序要求？**錯**，數學轉置正確不等於儲存順序正確。

**題四：** Fast Transpose 一定是 O(nnz)？**不精確**，傳統計數與 prefix-sum 版本還要初始化與掃描 c 個 column，應寫 O(c+nnz)。

**申論題寫法：** 先定義 r、c、nnz → 畫原矩陣 → 寫 COO 三元組 → 求 CSR 三陣列 → 展示轉置後索引 → 分析時間與空間 → 指出各格式適用的操作。比起死背程式碼，能清楚說明 indptr 和 Prefix Sum 的功能更重要。

## 結論：省下 0 不是終點，選對操作才是重點

前一篇 Polynomial 教我們只儲存「真正存在的項」；Sparse Matrix 把這個思路擴展成二維位置。**COO 擅長逐項建立，CSR 用 row pointer 壓縮列資訊，Fast Transpose 用計數與前綴和免去反覆掃描。** 資料結構選型不只看儲存量，也必須看後續要搜尋、插入、逐列計算，還是轉置。

下一篇將依三科輪替進入 **Day 39｜MIS #013｜IS 倫理與新興法規**。

---

## References

1. [SciPy — COO / coo\_matrix 官方文件](https://docs.scipy.org/doc/scipy/reference/generated/scipy.sparse.coo_matrix.html)
2. [SciPy — CSR / csr\_matrix 官方文件](https://docs.scipy.org/doc/scipy/reference/generated/scipy.sparse.csr_matrix.html)
3. [SciPy — Sparse Arrays Tutorial（Canonical Format 與 Explicit Zeros）](https://docs.scipy.org/doc/scipy/tutorial/sparse.html)
4. [Cornell CS 4787 — Lecture 12: Storing Sparse Matrices](https://www.cs.cornell.edu/courses/cs4787/2023fa/lectures/lecture12.html)
5. [Cornell CS 4220 — Data Sparsity and Fast Matrix-Vector Products（2026-01-23）](https://www.cs.cornell.edu/courses/cs4220/2026sp/lec/2026-01-23.html)
