---
layout: default
title: "多項式 Polynomial 怎麼存？Array、Linked List、稀疏表示與加法複雜度一次看懂"
date: 2026-10-19
categories: [資料結構]
---

# 資結 #012｜多項式只有三項，為何陣列要存上千格？從 Array、Sparse Representation 到 Linked List 看懂 Polynomial

> 假設要儲存一個多項式：3x¹⁰⁰⁰ + 2x³ − 5。它只有三個非零項，但最高次方是 1000。如果依次方 0、1、2……建立 1001 個陣列位置，絕大多數都放 0。**電腦真的需要記住這 998 個沒有出現的項嗎？** 這就是多項式（Polynomial）能幫我們看懂「資料表示法決定效能」的原因。
## 一、先理解 Polynomial ADT：運算規格不等於儲存方式
單變數多項式可以寫成 P(x) = a₀ + a₁x + a₂x² + … + aₙxⁿ。每個項（Term）有 **Coefficient（係數）**與 **Exponent（指數）**；最高非零項的指數稱為 **Degree（次數）**。本篇先限定指數為非負整數，係數為一般數值。
**Polynomial ADT（多項式抽象資料型別）**描述加法、乘法、求值、微分等「應該能做什麼」；**Array、Sparse Term Array、Linked List** 決定這些操作「怎麼儲存、怎麼做」。普林斯頓大學的 Polynomial 實作就提供 plus、times、evaluate 等方法，展示 ADT 與實作的分工。[Princeton Algorithms—Polynomial](https://algs4.cs.princeton.edu/code/javadoc/edu/princeton/cs/algs4/Polynomial.html)
## 二、Dense Array：用「索引＝指數」直接定位係數
假設 P(x) = 5x⁶ + 2x² − 7，若定義 A\[i\] 儲存 xⁱ 的係數，陣列內容為：
- 指數（Index）：0、1、2、3、4、5、6
- 係數（Value）：−7、0、2、0、0、0、5
因此 A\[6\] = 5、A\[2\] = 2。已配置範圍內**讀寫指定次方的係數是 O(1)**，理由和前一篇 Array 完全相同：用索引直接取值，不必逐項搜尋。
這類每個次方都預留位置的表示法常稱 **Dense Representation（稠密表示）**；若最高次方為 d，空間為 **O(d+1)**，不管其中有多少係數其實是 0。假設 P(x)=3x¹⁰⁰⁰+2，要儲存 x⁰ 到 x¹⁰⁰⁰ 的係數，就得預留 1001 格。**空間複雜度不是 O(非零項數)，而是 O(最高次方+1)**。
這種方法很適合多數係數都非零、或必須頻繁依特定指數直接查表的情況；[Princeton IntroCS—Symbolic Methods](https://introcs.cs.princeton.edu/java/92symbolic/) 的多項式範例也採用係數陣列。
## 三、Sparse Representation：只記錄真正存在的項
若只存非零項，就可用 (Exponent, Coefficient) 配對來描述：
- P(x) = 5x⁶ + 2x² − 7
- P = \[(6, 5), (2, 2), (0, −7)\]
這就是 **Sparse Representation（稀疏表示）**。若 P(x)=3x¹⁰⁰⁰+2，僅需 \[(1000,3),(0,2)\] 兩組配對，不用為缺席的 999 個次方建係數格。
儲存稀疏項有至少兩種常見實作：
1. **Sparse Term Array（稀疏項陣列）**：把 (exp, coef) 配對依指數排序後放在連續空間；逐項走訪具快取優勢，也能在排序後進行二分搜尋，但中間插入可能需要搬移。
2. **Linked List（鏈結串列）**：每個節點儲存 exp、coef、next；新增節點可連接指標，若已知前驅位置通常 O(1)，但**尋找插入位置仍可能 O(k)**，且每個節點還要支付指標與配置成本。
所以**Sparse 不等於 Linked List**；「很稀疏」也不代表 Linked List 一定最快。資料量、操作型態、快取局部性與節點配置成本都要一起衡量。[UT Austin—Linked List Representation of Polynomials](https://www.cs.utexas.edu/~mitra/csSpring2021/cs313/assgn/assgn16.html) 說明以係數與指數對儲存非零項；[UMBC—Polynomial ADT](https://courses.cs.umbc.edu/undergraduate/CMSC201/fall98/projects/p5/index.shtml) 使用依指數排序的 Linked List。
## 四、最重要的不變條件：指數順序、唯一性與零係數
要讓兩個稀疏多項式能有效率地相加，常先維持三個 **Invariants（不變條件）**：
- 指數依遞減（或遞增）順序排列，而且全程使用同一種方向。
- 同一指數最多出現一次，重複項必須合併係數。
- 係數變成 0 的項立刻移除；零多項式可以表示為空序列。
例如 (4,3)+(4,−3) 應直接消失，而不是保留 (4,0)。因為零係數會讓「非零項數 k」變得沒有意義，也會浪費後續運算成本。
## 五、Polynomial Addition：跟合併兩個排序串列幾乎一樣
設 P(x)=5x⁶+2x²−7，Q(x)=−5x⁶+4x⁴+3。
- P：\[(6,5),(2,2),(0,−7)\]
- Q：\[(6,−5),(4,4),(0,3)\]
兩個游標從最高次方開始：
1. 同為指數 6，係數相加 5+(−5)=0，所以**不輸出**。
2. Q 的指數 4 較大，輸出 (4,4)，推進 Q。
3. P 的指數 2 較大，輸出 (2,2)，推進 P。
4. 同為指數 0，係數 −7+3=−4，輸出 (0,−4)。
得到 **P(x)+Q(x)=4x⁴+2x²−4**，結果 \[(4,4),(2,2),(0,−4)\]。
這就是資料結構常見的 **Merge（雙指標歸併）**。若兩個輸入分別有 k、m 個非零項，且都已排序，兩個游標都只會向前移動，所以加法 **O(k+m)**；若使用會依次方掃描的 Dense Array，則是 **O(max(dₚ,d_q)+1)**。[UMBC Polynomial ADT](https://courses.cs.umbc.edu/undergraduate/CMSC201/fall98/projects/p5/index.shtml) 的實作練習要求合併同次方係數，並處理零係數刪除。
## 六、Polynomial Multiplication：為何比加法麻煩？
乘法必須把每個 P 的項與每個 Q 的項相乘；兩項 (e₁,c₁) 與 (e₂,c₂) 會變成 **(e₁+e₂, c₁×c₂)**。例如 (2,3)×(1,4)=(3,12)，也就是 (3x²)(4x)=12x³。
即使只有 k、m 個非零項，**直接展開也會先產生 k×m 個項乘積**，而多個項可能落在相同指數，必須再次合併。例如 (x+1)(x+1) 先得到 x²、x、x、1，最後才合併為 x²+2x+1。
- **Dense Array 的基本巢狀迴圈**：若兩個多項式次數為 dₚ 與 d_q，時間 O((dₚ+1)(d_q+1))。
- **Sparse 的基本項對展開**：先計算 k×m 個配對，再依指數聚合，不能直接宣稱「完整演算法永遠 O(km)」。例如用雜湊表彙整，預期時間可接近 O(km)，但若最後要求有序輸出，還要排序；用未排序串列逐項搜尋同次方則可能更慢。
Princeton 的 [Polynomial.java](https://algs4.cs.princeton.edu/code/edu/princeton/cs/algs4/Polynomial.java.html) 展示基於分配律的基本多項式乘法，適合對照理解。
## 七、時間與空間複雜度：不要被「陣列一定快」誤導
| 操作 | Dense Array | 已排序 Sparse Terms |
| --- | --- | --- |
| 儲存空間 | O(d+1) | O(k) 個配對；鏈結節點另有指標開銷 |
| 依指數找係數 | 已配置範圍 O(1) | 排序配對陣列 O(log k) 二分；鏈結串列 O(k) |
| 逐項走訪 | 掃描所有次方 O(d+1) | 只看非零項 O(k) |
| 多項式加法 | O(max(dₚ,d_q)+1) | 雙指標歸併 O(k+m) |
| 中間插入 | 更新既有係數可 O(1) | 項陣列要搬移；鏈結串列先找位置 |

其中 d 是最高次方、k 是非零項數。**先釐清輸入規模用 d 還是 k 衡量，再比較 Big-O**；最高次方可能是百萬，但真正非零的係數只有兩個。
## 八、生活與實務情境：怎麼選表示法？
假如感測器校正公式只有 3 個非零係數，最高卻到 x⁵⁰⁰，Sparse 有利於節省空間；若一段數值分析公式的 0 到 200 次係數幾乎都有值，Dense Array 可能更直觀，且有利於連續走訪。
**常見陷阱**是看見公式稀疏就立即選 Linked List。若資料建立後幾乎不修改、需要大量掃描，**排序後的 Pair Array** 往往也值得比較；若必須反覆插入、合併項目，才應仔細評估 Linked List 的操作成本。
## 九、考研練習：四個容易失分的觀念
**題一：** P(x)=7x¹⁰⁰+3 的 Dense Array 需要幾格？若索引從 0 到 100，需 **101 格**；稀疏表示只需要 **2 個非零項**，但每項還得記指數。
**題二：** Sparse Linked List 加入一項一定是 O(1) 嗎？**不一定。** 已知正確插入位置才可 O(1) 修改鏈結；若要先搜尋，最壞可能 O(k)。
**題三：** 兩個依指數降冪排序、各有 k、m 項的多項式相加需要 O(km) 嗎？**不需要。** 雙指標歸併為 **O(k+m)**。
**題四：** P=\[(3,2),(1,4)\]，Q=\[(3,−2),(0,5)\]，相加後是否保留 (3,0)？**不應保留。** 結果為 \[(1,4),(0,5)\]。
**申論題：**「比較 Dense Array、Sparse Term Array 與 Linked List 表示多項式的優缺點，並設計多項式加法演算法。」建議順序：**ADT 定義 → d 與 k 的意義 → 空間比較 → 依指數排序與零項刪除 → Merge 流程 → O(k+m) 推導 → 適用場景**。
## 結論：儲存方式不是數學公式的一部分
多項式本身只有係數與指數的數學意義；讓演算法快或慢的，是你**選擇把缺席的項也存下來，還是只存真正存在的項**。Dense Array 以 O(d) 空間換得 O(1) 索引定位；Sparse 以記錄配對節省空間，並讓已排序非零項可 O(k+m) 歸併。理解這兩種設計，後面的 **Sparse Matrix（稀疏矩陣）**就能自然接上。
下一篇（Day 36）依三科輪替為 **MIS #012｜IS 投資效益與 ESG**。
---
## References
1. [Princeton Algorithms, 4th Edition—Polynomial API](https://algs4.cs.princeton.edu/code/javadoc/edu/princeton/cs/algs4/Polynomial.html)
2. [Princeton IntroCS—Symbolic Methods: Polynomial ADT 與 Horner's Method](https://introcs.cs.princeton.edu/java/92symbolic/)
3. [UT Austin CS313—Linked List Representation of Polynomials](https://www.cs.utexas.edu/~mitra/csSpring2021/cs313/assgn/assgn16.html)
4. [UMBC CMSC201—Polynomial ADT（Sorted Linked List）](https://courses.cs.umbc.edu/undergraduate/CMSC201/fall98/projects/p5/index.shtml)
5. [Princeton Algorithms—Polynomial.java 原始碼](https://algs4.cs.princeton.edu/code/edu/princeton/cs/algs4/Polynomial.java.html)