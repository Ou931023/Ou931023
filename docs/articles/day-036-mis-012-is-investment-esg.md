---
layout: default
title: "AI 系統省了錢，為何還不一定成功？從 ROI、TCO、NPV 到 ESG 看懂資訊系統投資效益"
date: 2026-10-20
categories: [MIS]
---

# MIS #012｜AI 系統省了錢，為何還不一定成功？從 ROI、TCO、NPV 到 ESG 看懂資訊系統投資效益

> 公司導入 AI 客服後，平均回覆時間縮短、值班人力減少。主管宣布專案成功，但財務部發現雲端帳單與維護費持續上升；永續部門也問：增加的運算用電算進去了嗎？**一個資訊系統「省錢」和「創造價值」，其實是兩種不同的判斷。** 本篇從可以計算的成本現金流、到不容易貨幣化的服務品質與 ESG，建立可真正檢驗的 IS 投資效益架構。
## 一、先區分「系統做出來」與「投資有回報」
資訊系統（Information System, IS）可以準時上線、功能正常，卻沒有改善組織績效。例如導入報表平台，操作畫面做得完整，但主管仍使用舊 Excel 決策；專案交付成功，**效益實現（Benefits Realization）**卻未必成功。
MIS 的投資評估可以分成三層：
1. **投入（Input）**：採購、開發、雲端、人員訓練與維運成本。
2. **產出（Output）**：系統上線、處理幾張訂單、生成多少份報表。
3. **成果（Outcome）與影響（Impact）**：訂單錯誤是否減少、顧客等待時間是否下降、風險與排放是否真的改善。
「系統每天自動處理 1,000 筆」只是產出；**相較原流程減少多少重工、以多大成本取得這個成效**，才接近效益。美國 GAO 的《Cost Estimating and Assessment Guide》也強調可靠估算須有清楚假設、成本範圍與持續更新，而非只拿最初報價當總成本。[GAO 成本估算指南（2020）](https://www.gao.gov/products/gao-20-195g)
## 二、TCO：買一套系統的價格，不是它真正的成本
**Total Cost of Ownership（TCO，總持有成本）**要求在指定期間與成本邊界內，計入取得、使用、維護以及最終退出所需資源。以三年期客服 AI 專案為例，除了第一筆軟體費，還可能有 API 使用量、資料整合、資安檢查、員工訓練、人工複核、備份、移轉與系統汰換。
尤其 SaaS 的低月費不等於低 TCO。若導入後必須雇用專人清理資料、修正 AI 誤答，這些**實際消耗的資源**就不能從比較中消失；反過來，也不能把原本就必須支出的費用無條件當成新系統成本。
實作評估表應至少列出：一次性費用、每年固定費用、依使用量變動的費用、替換舊系統的節省、風險準備金。比較兩個選項時，要使用**相同期間、相同服務範圍與相同幣別**，才能避免假性便宜。
## 三、ROI、Payback、NPV：三個數字回答三個問題
以下都是**虛構的教學情境，不是任何企業的實際營運數據**。
假設某公司一次投入 **90 萬元**建置客服系統；之後三年，每年產生可驗證的節省與收入改善 **50 萬元**，但每年另付維運與雲端費 **12 萬元**。假設各年現金流於年末發生、折現率為 **8%**，忽略稅負及殘值。
| 指標 | 本例算法 | 結果 |
| --- | --- | --- |
| TCO（3 年） | 初始 90＋維運 12×3 | **126 萬元** |
| 三年總效益 | 每年 50×3 | **150 萬元** |
| 簡化 ROI | （150－126）÷126 | **約 19.05%** |
| 簡單回收期 | 初始 90÷每年淨流入 38 | **約 2.37 年** |
| NPV | −90＋38÷1.08＋38÷1.08²＋38÷1.08³ | **約 +7.93 萬元** |

**ROI（投資報酬率）**呈現指定口徑下總利益相對成本的比例；**Payback Period（回收期）**回答多久拿回初始支出，卻通常忽略回收期以後的價值與貨幣時間價值；**NPV（淨現值）**將不同時間的淨現金流折成今日價值。OpenStax《Principles of Finance》的 NPV 教學亦以折現後現金流作為投資決策依據。[OpenStax—Net Present Value](https://openstax.org/books/principles-finance-2e/pages/16-2-net-present-value-npv-method)
注意：同樣的專案也可能因效益比預期少、維運成本提高或折現率改變，導致 NPV 轉負。**NPV 正值不是預言；它是基於假設的決策計算。** 必須做樂觀、基準、悲觀三種情境與敏感度分析。
## 四、不能直接換成錢的價值，也要用 KPI 驗證
有些資訊系統提高服務品質、資訊透明度、無障礙使用或資安治理，短期未必直接變成營收。不能因為難貨幣化就視為零，也不能憑一句「使用者滿意」便宣稱有成效。
建議使用「基準值 → 目標值 → 實測值 → 責任人 → 量測來源」五欄：客服平均解決時間、一次解決率、申訴率、員工採用率、資料錯誤率與系統停機時間都可追蹤。若只比上線前後，要警覺旺季、促銷活動、組織調整等其他因素；有條件時再使用分組測試或對照組強化因果判讀。
此外，**KPI 不可只有好消息**：AI 回答速度變快，若誤答率、個資事件或人員加班同時增加，就需要一起報告。
## 五、ESG：讓 IS 投資不只算財務報酬
**ESG** 分別代表 **Environmental（環境）**、**Social（社會）**與 **Governance（治理）**；可將其轉為設計與營運指標，而不是只在報告封面貼上三個字。
- **E 環境**：每千筆交易耗電量（kWh）、雲端服務估算排放（kgCO₂e）、設備使用年限、電子廢棄物處理。
- **S 社會**：身障或高齡使用者任務完成率、數位服務可及性、客服轉真人的成功率、員工培訓完成率。
- **G 治理**：存取權限稽核、資料刪除與留存規範、資安事件處理時間、模型決策可追溯性與責任分工。
這些是**可用的管理 KPI 設計範例**，不是 IFRS 強制每家企業都使用的統一清單。IFRS Foundation 對 **IFRS S2** 的公開說明指出，氣候相關揭露重點包含治理、策略、風險管理、指標與目標；是否具有適用的法定揭露義務，仍須依所在地規範、企業身分與適用時程判斷。[IFRS Foundation—ISSB Standards](https://www.ifrs.org/sustainability/knowledge-hub/introduction-to-issb-and-ifrs-sustainability-disclosure-standards/)
## 六、Scope 1、2、3：換成雲端不代表碳排自動歸零
GHG Protocol 將企業溫室氣體排放區分為三類：
1. **Scope 1**：企業擁有或控制來源的直接排放，例如自有鍋爐燃燒燃料。
2. **Scope 2**：購買電力、蒸汽、熱或冷卻等能源帶來的間接排放。
3. **Scope 3**：價值鏈中其他上游、下游間接排放，例如採購設備與使用外部服務所涉及的排放。
若把伺服器從公司機房移到雲端，公司的自有電力使用可能下降，但供應商的資料中心仍會耗能；**排放可能改變盤查位置，不等於自然消失**。在制定指標前，必須先訂組織邊界、盤查期間、排放係數與資料取得方式。[GHG Protocol—Standards](https://ghgprotocol.org/standards)；[GHG Protocol—Calculation Tools FAQ](https://ghgprotocol.org/calculation-tools-faq)
還有一個常考陷阱：**減少耗電量 ≠ 必然減少等比例排碳量**，因為用電時段與電網排放係數也會影響結果。
## 七、真實企業案例：Google Cloud Carbon Footprint
Google Cloud 提供 **Carbon Footprint**，協助客戶依專案、產品、區域等維度檢視其雲端服務所分攤的估算溫室氣體排放，並可結合資料匯出進行分析。它把「雲端成本看板」擴展成可追蹤環境影響的管理資訊，適合說明 **MIS 如何把資料變成可決策的指標**。[Google Cloud—Carbon Footprint 方法說明](https://docs.cloud.google.com/carbon-footprint/docs/methodology)；[Google Cloud—Custom Dashboard](https://docs.cloud.google.com/carbon-footprint/docs/custom-dashboard)
但**有儀表板不等於減碳已成功**。Google 官方說明亦指出，客戶層級的排放估算尚未經第三方獨立查證，方法或資料來源更新可能調整既有估計。進一步評估仍需區分原始觀測、估算分攤、真正減量與企業揭露。不能把平台提供估算功能，寫成「所有使用者都已減少多少噸碳」的證據。
## 八、考研常見的四組易混淆觀念
**TCO ≠ 初始採購價**：要包含指定期間內必要的維運與退出成本。
**ROI ≠ NPV**：ROI 看比例，NPV 使用折現後現金流；相同 ROI 的方案可能因回收時間不同而有不同 NPV。
**Output ≠ Outcome**：新系統上線、處理更多交易，不代表錯誤率、顧客體驗或實際成本一定改善。
**ESG 評量 ≠ 碳排單一數字**：ESG 涵蓋環境、社會、治理；碳排本身也必須說明範圍、方法、資料品質與期間。
**申論題：**「某零售商欲導入 AI 客服與雲端資料平台，請提出資訊系統投資效益評估架構，兼顧財務與 ESG。」
可依「**策略目標 → TCO 成本邊界 → ROI／NPV 與敏感度 → Baseline 與效益 KPI → ESG 三構面與碳盤查邊界 → 上線後追蹤、修正與治理**」答題。此架構同時回答**值不值得投資、投資後有沒有兌現、成本與外部影響由誰承擔**三個問題。
## 結論：價值必須在上線之後持續被驗證
對資訊系統而言，財務正報酬只是評價的一部分。TCO 告訴企業真正花了多少、ROI 和 NPV 衡量財務合理性；效益 KPI 檢查系統是否改善流程、顧客與員工體驗；ESG 則讓環境、社會及治理影響被看見。最關鍵的習慣是：**每一個宣稱的效益，都要對應明確的衡量邊界、基準值與資料來源。**
下一篇依三科輪替為 **Day 37｜計概 #013｜Critical Section（臨界區）**。
---
## References
1. [U.S. GAO—Cost Estimating and Assessment Guide（2020）](https://www.gao.gov/products/gao-20-195g)
2. [OpenStax—Principles of Finance, 16.2 Net Present Value](https://openstax.org/books/principles-finance-2e/pages/16-2-net-present-value-npv-method)
3. [IFRS Foundation—Introduction to ISSB and Sustainability Disclosure Standards](https://www.ifrs.org/sustainability/knowledge-hub/introduction-to-issb-and-ifrs-sustainability-disclosure-standards/)
4. [GHG Protocol—Standards](https://ghgprotocol.org/standards)
5. [GHG Protocol—Calculation Tools FAQ](https://ghgprotocol.org/calculation-tools-faq)
6. [Google Cloud—Carbon Footprint Reporting Methodology](https://docs.cloud.google.com/carbon-footprint/docs/methodology)
7. [Google Cloud—Create a Custom Carbon Footprint Dashboard](https://docs.cloud.google.com/carbon-footprint/docs/custom-dashboard)
## 發布與 SEO 資訊
**SEO 標題：** 資訊系統投資效益怎麼算？ROI、TCO、NPV 與 ESG KPI 一次看懂
**SEO 摘要：** 企業導入 AI 或雲端系統，省錢就算成功嗎？用三年案例算出 TCO、ROI、NPV 與回收期，並以 GHG Protocol、IFRS S2 和 Google Cloud Carbon Footprint 說明 ESG 效益的量化與限制。
**分類：** MIS／資訊管理
**Tags：** MIS、管理、ROI、NPV、TCO、ESG、資訊系統投資、效益實現、Carbon Footprint
**建議 GitHub slug：** `mis-012-it-investment-roi-tco-npv-esg`
## 視覺化構想
1. **三年現金流時間軸**：初始 90 萬與每年淨流入 38 萬，標出折現後 NPV。
2. **Output → Outcome → Impact**：系統上線、流程改善、永續影響逐層對照。
3. **ESG × KPI 矩陣**：每個構面對應量測單位、資料來源與管理責任。
4. **Scope 1／2／3 邊界圖**：自有設備、外購電力與雲端供應商的排放位置差異。