---
layout: default
title: "Symmetric Matrix 對稱矩陣怎麼存？上下三角壓縮與位址公式｜資料結構"
description: "從 4×4 對稱矩陣出發，推導下三角與上三角壓縮儲存、Row-Major／Column-Major 位址公式、索引換算與考研常見陷阱。"
date: 2026-10-28
categories: [資料結構]
tags: [Symmetric Matrix, Data Structures, Packed Storage, Row-Major, Column-Major, Lower Triangle, Upper Triangle]
---

# 資結 #015｜對稱矩陣為什麼只要存一半？從 Lower Triangle、Row-Major 到一維位址公式看懂 Symmetric Matrix

> 想像學校有四棟教學大樓，紀錄「A 到 B 的直線距離」時，A 到 B 和 B 到 A 應該一樣。若把所有兩兩距離放進表格，上下兩側就會出現大量重複數字。**既然資料相同，為什麼要在記憶體裡存兩次？**這正是 Symmetric Matrix（對稱矩陣）壓縮儲存的出發點。

## 一、先認識對稱矩陣：不是所有方陣都對稱

對稱矩陣必須是 **n×n 方陣**，而且對任意索引 i、j，都符合 **A[i][j] = A[j][i]**，也就是矩陣轉置後不變（$A^T=A$）。主對角線上的元素是 A[0][0]、A[1][1] 等，它們在轉置後仍位於原處。

下面是一個 4×4 對稱矩陣。先不要背公式，觀察 2 和 2、7 和 7、5 和 5 如何沿著主對角線互為鏡像：

```text
       col0 col1 col2 col3
row0     4    2    7    1
row1     2    6    3    5
row2     7    3    8    9
row3     1    5    9   10
```

特別注意：**對稱不等於元素都一樣**，也不表示對角線必須是零；只要求 A[i][j] 與 A[j][i] 相等。數學上對稱矩陣的定義可參考 Wolfram MathWorld。

## 二、只留一半，空間到底省多少？

一般 4×4 陣列需要儲存 $4^2=16$ 個元素。如果只保留包含主對角線的**下三角**，每一列分別需要 1、2、3、4 個元素，共 $1+2+3+4=10$ 個。若保留上三角，數量也一樣。

延伸到 n×n 矩陣，所需元素數為：

$$
1+2+\cdots+n=\frac{n(n+1)}{2}
$$

因此，完整儲存使用 $n^2$ 格；三角壓縮則使用 $n(n+1)/2$ 格。n=4 時節省 6/16＝**37.5%**；n=1000 時從 1,000,000 格變成 500,500 格，節省 **49.95%**。常說「省一半」是大 n 的近似，不是每個尺寸都恰好減半。

## 三、下三角 Row-Major：公式從哪裡來？

承接上一篇 Matrix Storage：**Row-Major 是一列接著一列存**。下三角只保留 $i\ge j$ 的元素，將上面的 4×4 依列攤平：

```text
row0:  4
row1:  2, 6
row2:  7, 3, 8
row3:  1, 5, 9, 10

packed = [4, 2, 6, 7, 3, 8, 1, 5, 9, 10]
index  =  0  1  2  3  4  5  6  7  8   9
```

這裡**全部採 0-based index**，即列 i、欄 j 都從 0 開始。要找 A[i][j]，先數前面幾列總共放了多少元素：第 0 列 1 個、第 1 列 2 個，直到第 i−1 列 i 個，所以是 $1+2+\cdots+i=i(i+1)/2$。接著在第 i 列往右移 j 格：

$$
\boxed{offset_{lower,row}(i,j)=\frac{i(i+1)}{2}+j},\quad i\ge j
$$

例如 A[3][1]＝5：前面三列共 1+2+3＝6 格，再移 1 格，**offset＝7**，正是 packed[7]。

如果題目問 A[1][3] 呢？它位於沒有儲存的上三角；利用對稱性 A[1][3]＝A[3][1]，**先交換 i、j，再套公式**。

## 四、位址題：Offset 不等於實體位址

假設 packed 陣列起始位址 B＝1000，每個元素大小 w＝8 bytes。A[1][3] 先映射到 A[3][1]，其 offset＝7，所以：

$$
Address(A[1][3])=B+w\times offset=1000+8\times7=1056
$$

答 **1056**，不是 1007。考試最常見錯誤，是忘了乘上每個元素的 byte 數。

若題目使用 **1-based index**，且下三角按列儲存，元素 A(i,j) 在壓縮陣列中的 **1-based 位置**為 $i(i-1)/2+j$（$i\ge j$）。要算從 B 開始的位址，必須先將位置減 1。**先辨認索引起點，再決定用哪個公式。**

## 五、下三角 Column-Major，以及上三角怎麼辦？

如果改成 **Column-Major（逐欄儲存）**，下三角第 0 欄有 n 格、第 1 欄有 n−1 格……第 j 欄之前總共有 $jn-j(j-1)/2$ 格，而 A[i][j] 在此欄的相對位置為 i−j。因此：

- **下三角 Row-Major（i≥j）**：$i(i+1)/2+j$。
- **下三角 Column-Major（i≥j）**：$jn-j(j-1)/2+(i-j)$。
- **上三角 Row-Major（i≤j）**：$in-i(i-1)/2+(j-i)$。
- **上三角 Column-Major（i≤j）**：$j(j+1)/2+i$。

這四條公式無須死背四次：每次都先算「**前面完整儲存的列／欄有幾個元素**」，再加上「**當前列／欄內偏移**」。例如 n＝4，A[3][1] 的下三角 Column-Major offset＝$1\times4-0+(3-1)=6$，會不同於 Row-Major 的 7。兩者都正確，只因排列規則不同。

Intel oneMKL 與 oneAPI 規格明確區分 Upper／Lower 和 Row／Column-Major 的 Packed Storage；**不能看到對稱矩陣，就假設一定按下三角 Row-Major 儲存**。

## 六、實作：把「鏡像」當作存取規則

最簡單的做法，是把索引映射到下三角，再取出一維陣列的元素：

```python
def lower_row_major_index(i, j, n):
    if not (0 <= i < n and 0 <= j < n):
        raise IndexError("matrix index out of range")
    if i < j:
        i, j = j, i   # 上三角鏡射到下三角
    return i * (i + 1) // 2 + j

packed = [4, 2, 6, 7, 3, 8, 1, 5, 9, 10]
print(packed[lower_row_major_index(1, 3, 4)])  # 5
```

每次讀取只需固定次數的比較與計算，索引映射是 **O(1)**；但改用一維壓縮後，程式必須負責映射，不能直接當成一般二維陣列使用。若要修改 A[1][3]，也要更新那個**唯一保留**的位置，而不是存兩份。

NumPy 官方提供 tril_indices 等 API，可以取得下三角索引來做選取；不過 **「取得下三角索引」不代表 NumPy 一般 ndarray 自動變成壓縮儲存**，這是表示法與工具功能的差別。

## 七、真實應用：省空間未必每次都比較快

**距離矩陣、相關係數矩陣、無向圖鄰接矩陣**可能具有對稱性。例如無向圖中兩節點是否相連通常滿足 A[i][j]＝A[j][i]；但若資料是單行道行車時間，A 到 B 不一定等於 B 到 A，就不能硬套對稱壓縮。

科學運算中的 BLAS／LAPACK 也有 Packed Storage。不過壓縮後的排列方式不見得最適合高效矩陣乘法；因此某些程式會採用完整儲存，或採用 Rectangular Full Packed 等兼顧空間與計算效率的格式。Intel 官方文件也特別解釋了這類權衡。

## 八、考研最容易混淆的四件事

1. **Symmetric ≠ Triangular**：對稱矩陣的另一半是鏡像值；三角矩陣另一半則為零。它們可以使用相似壓縮方式，但數學意義不同。
2. **Symmetric ≠ Sparse**：對稱性來自重複資訊；稀疏性來自大量零值。不要把上一篇 Sparse Matrix 的 COO／CSR 公式直接套過來。
3. **Row-Major ≠ Column-Major**：即使都儲存下三角，同一元素的 offset 也可能不同。
4. **索引、位置與位址不同**：0-based offset、1-based 位置、byte address 是三個不同概念。

**考研題型練習**：一個 5×5 的對稱矩陣，採 0-based 下三角 Row-Major 壓縮，B＝2000、w＝4 bytes，求 A[1][4] 的位址。先映射 A[4][1]，offset＝4×5/2+1＝11，故位址＝2000+11×4＝**2044**。若題目改為 Column-Major，要重新計算，不能沿用 11。

## 結論：先理解結構，再推導位址

對稱矩陣壓縮的核心只有兩步：**利用 A[i][j]＝A[j][i] 消除重複資料，再依照儲存順序把二維索引轉成一維偏移**。掌握「前面存了幾格＋現在移動幾格」，遠比背四張公式表可靠。下一個 Band Matrix 主題會進一步討論：當矩陣只有主對角線附近的元素有用，該如何避免儲存大量零值？

## References

1. [Wolfram MathWorld — Symmetric Matrix](https://mathworld.wolfram.com/SymmetricMatrix.html)：對稱矩陣的定義。
2. [Intel oneMKL Developer Reference (2026) — Matrix Storage Schemes for LAPACK Routines](https://www.intel.com/content/www/us/en/docs/onemkl/developer-reference-c/2026-0/matrix-storage-schemes-for-lapack-routines.html)：Packed Storage 格式與列／欄位址映射。
3. [oneAPI Specification — Matrix Storage](https://oneapi-spec.uxlfoundation.org/specifications/oneapi/v1.1-rev-1/elements/onemkl/source/domains/matrix-storage)：上、下三角逐行／逐欄壓縮順序。
4. [NumPy v2.3 — numpy.tril_indices](https://numpy.org/doc/2.3/reference/generated/numpy.tril_indices.html)：程式取得下三角索引的公開文件。
5. [Oracle — Working With Matrices](https://docs.oracle.com/cd/E19059-01/stud.9/817-6701/plug_matrices.html)：LAPACK Packed Storage 的實例。

