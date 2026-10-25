---
layout: default
title: "Matrix Storage 矩陣儲存怎麼算？Row-Major、Column-Major 位址公式與考研題一次看懂"
description: "二維陣列在記憶體裡如何排列？用 3×4 範例推導 Row-Major 與 Column-Major 位址公式，理解下界轉換、NumPy strides 和 Cache Locality。"
date: 2026-10-25
categories: [資料結構]
tags: [Data Structures, Matrix Storage, Row-Major, Column-Major, Address Calculation, Strides, Cache Locality]
---

# 資結 #014｜二維陣列明明是表格，記憶體卻是一條線？

> 同樣是一張 3×4 成績表，電腦在 RAM 中不會真的畫出三列四行的格子。記憶體位址是一條線，程式卻能寫出 A[2][1]。如果題目只給你起始位址、元素大小和矩陣形狀，能不能算出這個元素到底存在哪裡？**Matrix Storage（矩陣儲存）真正要理解的，不是背兩條公式，而是二維座標怎麼被展開成一維位移。**

## 一、矩陣有兩個索引，記憶體為何只有一條線？

先定義矩陣 A 有 **R 個橫向 row、C 個直向 column**；本文直接使用 row／column 術語，避免中文「行／列」在不同教材中的翻譯差異。每個元素占 **w bytes**，第一個元素的起始位址是 **B**。下面採 0-based index（索引從 0 開始）。

想像一張 3×4 的表格：

```txt
       col0 col1 col2 col3
row0     1    2    3    4
row1     5    6    7    8
row2     9   10   11   12
```

在連續儲存的情況下，12 個元素會放在一連串記憶體位置。差別在於：**先存完整的一列，再存下一列；還是先存完整的一欄，再存下一欄？** C 語言的真正二維陣列，例如 `int a[3][4]`，屬於列優先的巢狀陣列布局。這與 `int **` 指向多個獨立配置的陣列不同；後者不保證所有元素構成同一段連續的 3×4 區塊。[cppreference — C Array Declaration](https://en.cppreference.com/c/language/array)

## 二、Row-Major：把一整列寫完，才輪到下一列

**Row-Major Order（列優先）**的線性順序是：

```txt
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
```

想找 A[i][j]，先跳過前面 i 列；每一列都有 C 個元素，所以跳過 **i×C** 個，再向右移動 j 個。若從 0 開始編號，公式自然是：

```txt
Row-major offset (elements) = i × C + j
Address(A[i][j]) = B + w × (i × C + j)
```

假設 B = 1000、w = 4 bytes、C = 4，A[2][1] 的線性位置是 2×4+1=9，位址就是 **1000+9×4=1036**。這裡的 1000 是為了練習而假設的位址，不代表實際程式必然從此處配置。

可以用「圖書館書架」理解：每層放四本書，要找第三層第二本，先越過前兩層的八本，再往右走一本。這比死記「乘欄數」更容易回推公式。

## 三、Column-Major：先存一整欄，為何要乘列數？

**Column-Major Order（欄優先）**改成先向下儲存：

```txt
[1, 5, 9, 2, 6, 10, 3, 7, 11, 4, 8, 12]
```

A[i][j] 前面已經有 j 個完整欄，而每欄有 R 個元素，所以要跳過 **j×R** 個，再往下移動 i 個：

```txt
Column-major offset (elements) = j × R + i
Address(A[i][j]) = B + w × (j × R + i)
```

沿用 R=3、B=1000、w=4，A[2][1] 的偏移是 1×3+2=5，位址為 **1020**。數學上的 A[2][1] 仍是值 10；變的是物理儲存順序，不是矩陣內容。Fortran 官方教學說明，多維陣列採欄優先，因此**最左側索引**相鄰變化時，通常就是連續記憶體存取。[Fortran-lang — Multidimensional Arrays](https://fortran-lang.org/learn/best_practices/multidim_arrays/)

## 四、考試最常變形：索引不是從 0 開始怎麼辦？

如果列索引從 Lr 開始，欄索引從 Lc 開始，就先把座標歸零。不是把原本的 i、j 直接帶入：

```txt
Row-major:
B + w × ((i − Lr) × C + (j − Lc))

Column-major:
B + w × ((j − Lc) × R + (i − Lr))
```

例如 A[1..3][1..4] 是 3×4、元素大小 4 bytes、B=1000，問 A[3][2]。把索引改成零起算：i−Lr=2，j−Lc=1。Row-Major 位址是 **1036**，Column-Major 位址是 **1020**。兩種題目只差布局；如果混淆 R 和 C，就會算錯。

再延伸到三維 A[D0][D1][D2]：Row-Major 的位移量是 `i×D1×D2 + j×D2 + k`。思路完全一樣：**先跳過完整的平面，再跳過完整的一列，最後定位單一元素**。對更高維的陣列，也可以逐層乘上後續維度大小。

## 五、Strides：同一個公式在 NumPy 裡真的看得見

在 NumPy 中，**strides** 表示沿某個維度前進一格時，底層資料指標需要跳過幾個 bytes。假設矩陣形狀為 (3,4)、每個 `int32` 占 4 bytes，Row-Major 的 strides 是 **(16,4)**；也就是「row +1 前進 16 bytes，column +1 前進 4 bytes」。[NumPy — ndarray.strides](https://numpy.org/doc/2.3/reference/generated/numpy.ndarray.strides.html)

```py
import numpy as np

a = np.arange(1, 13, dtype=np.int32).reshape(3, 4)
print(a.shape)      # (3, 4)
print(a.strides)    # (16, 4)
print(a[2, 1])      # 10

t = a.T
print(t.shape)      # (4, 3)
print(t.strides)    # (4, 16)
```

此處 `a.T` 通常建立**不同 strides 的視圖（view）**，不需要立即複製元素；但如果後續函式要求連續儲存，可能需要額外配置或複製。這表示「轉置矩陣」在數學上交換索引，與「將每個位元組重新排列」不是永遠同一個操作。NumPy 的 N-dimensional array 文件也把 Row-Major、Column-Major 都視為一般化 strides 的特殊案例。[NumPy — The N-dimensional Array](https://numpy.org/doc/1.25/reference/arrays.ndarray.html)

## 六、為什麼矩陣走訪順序會影響速度？

假設 `a` 是很大的 Row-Major 稠密矩陣，兩層迴圈都會訪問 R×C 個元素，理論上的遍歷時間複雜度都是 **O(R×C)**；但兩者不一定一樣快。

```c
/* C 的 int a[R][C]：內層改變 column，傾向連續存取 */
for (int i = 0; i < R; ++i)
    for (int j = 0; j < C; ++j)
        use(a[i][j]);

/* 內層改變 row：跨列跳躍，步距通常較大 */
for (int j = 0; j < C; ++j)
    for (int i = 0; i < R; ++i)
        use(a[i][j]);
```

第一種通常更符合快取（Cache）的**空間區域性 Spatial Locality**，因為相鄰元素容易落在同一個 Cache Line。第二種若每次跨越很大步距，可能增加快取失誤。這是常見的效能趨勢，**不能說不論矩陣大小、硬體或編譯器最佳化，第一種永遠比較快**。在 Column-Major 的 Fortran 中，反而應讓最左側索引在內層快速變動。這也是實務上為何要理解資料布局，而非只看 Big-O 的原因。[Fortran-lang — Multidimensional Arrays](https://fortran-lang.org/learn/best_practices/multidim_arrays/)

## 七、與上一篇 Sparse Matrix 如何銜接？

前一篇介紹 COO、CSR，目的是在 **nnz 遠小於 R×C** 時避免儲存大量沒有必要的 0；本篇討論的是**稠密或規則連續陣列中，座標到位址的映射**。兩者處理不同問題：

- **Dense Row-Major / Column-Major**：有固定且規則的位址公式，任意元素索引通常 O(1)，但儲存需要 O(R×C) 個位置。
- **Sparse COO / CSR**：只保存部分座標與資料，空間與非零項數相關；若要讀任意 A[i][j]，可能要搜尋索引結構，不能直接套用稠密陣列的位址公式。
- **Transpose**：稠密矩陣可以透過 strides 描述轉置視圖；CSR/COO 的轉置則牽涉其索引及儲存格式，是否重排資料依實作而定。

因此，「選哪一種矩陣表示法」應先問資料是否稀疏，再問後續主要是依列、依欄掃描，還是隨機查詢。

## 八、考研五個高頻判斷

1. **3×4 Row-Major A[2][1]，從 0 開始，w=4，B=1000？** 公式用 C=4，答案是 **1036**；不是把列數 3 乘進去。
2. **同元素的 Row-Major / Column-Major 位址一定相同？** 錯，矩陣值相同不代表線性順序相同。
3. **索引從 1 開始時，直接算 i×C+j？** 錯，必須先減掉各維下界。
4. **二維陣列一定等於 ****`int **`****？** 錯，C 的真正二維陣列是巢狀的連續陣列，指標陣列不保證同布局。
5. **兩種迴圈都 O(R×C)，效能一定相同？** 錯，步距與快取區域性可能影響常數成本。

**申論題順序：** 定義 R、C、w、B 與索引下界 → 畫出儲存順序 → 把目標索引改成零起算 → 計算前面跨過幾個完整 row 或 column → 乘每元素 bytes → 加回 base address → 最後檢查是否落在合法索引範圍。

## 結論：記住跨過多少元素，比背公式更重要

Row-Major 的關鍵是「**前面有幾列 × 每列幾格**」；Column-Major 是「**前面有幾欄 × 每欄幾格**」。只要清楚矩陣維度、元素大小、索引下界與儲存順序，就能自己推導位址；再連到 NumPy strides 與 CPU Cache，才會理解為什麼同樣的二維資料，在不同語言與走訪方式下有不同的記憶體行為。下一篇資料結構將依 Roadmap 接續 **Symmetric Matrix（對稱矩陣）**的壓縮儲存。

---

## References

1. [cppreference — C Array Declaration: Multidimensional Arrays](https://en.cppreference.com/c/language/array)（C 語言的多維陣列布局與指標轉換）
2. [Fortran Programming Language — Multidimensional Arrays](https://fortran-lang.org/learn/best_practices/multidim_arrays/)（欄優先與連續存取）
3. [NumPy — ndarray.strides](https://numpy.org/doc/2.3/reference/generated/numpy.ndarray.strides.html)（索引與 byte stride）
4. [NumPy — The N-dimensional Array](https://numpy.org/doc/1.25/reference/arrays.ndarray.html)（多維位址映射與 stride 的一般形式）
5. [NumPy — ndarray Reference](https://numpy.org/doc/1.24/reference/generated/numpy.ndarray.html)（C/F order、shape 與 itemsize）

## 發布資訊

**SEO 標題：** Matrix Storage 矩陣儲存怎麼算？Row-Major、Column-Major 位址公式與考研題一次看懂

**SEO 摘要／Excerpt：** 二維陣列在記憶體裡如何排列？用 3×4 範例推導 Row-Major 與 Column-Major 位址公式，理解下界轉換、三維陣列、NumPy strides、Cache Locality 與考研常見陷阱。

**建議 slug：** `ds-014-matrix-storage-row-major-column-major`

**分類：** 資料結構

**Tags：** Data Structures、Matrix Storage、Row-Major、Column-Major、Address Calculation、Strides、Cache Locality

**書審代表作建議：** 暫不列入。本文適合作為基礎教學篇；若後續以 NumPy 與 C 實測不同走訪順序的執行時間，再搭配效能分析，可提升作品集價值。

## 視覺化構想

1. **矩陣到 RAM 的箭頭圖：** 同一張 3×4 表，分別依 Row-Major 與 Column-Major 展成兩條線。
2. **A[2][1] 位址分解圖：** 標註被跳過的完整 row／column、元素位移及 bytes。
3. **NumPy strides 動態圖：** 對照 (16,4) 和轉置視圖的 (4,16)。
4. **Cache Locality 比較圖：** 標出 Row-Major 逐列與逐欄掃描所跨越的記憶體距離。