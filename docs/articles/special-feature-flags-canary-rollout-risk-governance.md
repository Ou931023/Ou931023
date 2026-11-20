---
title: "新功能測試都通過，為什麼上線還是出事？從 Feature Flag、Canary Release 到 SLO 看懂發布風險"
date: 2026-11-20
description: "從 Feature Flag、Canary Release 到 SLO，理解新功能上線的技術風險與 MIS 治理。"
slug: feature-flags-canary-release-slo-risk-governance
categories: [MIS, CS, System Design]
tags: [Feature Flag, Canary Release, Progressive Delivery, SLO, DevOps, MIS]
status: original-article-experiment-unverified
wordpress_status: pending-site-restoration
---

# 新功能測試都通過，為什麼上線還是出事？從 Feature Flag、Canary Release 到 SLO 看懂發布風險

> 電商把新版結帳流程一次開放給全部顧客，測試環境明明通過，正式上線後卻出現部分手機無法付款、客服量暴增。工程師說「程式可以回滾」，營運主管卻問：「已經失敗的訂單怎麼辦？」這是部署（Deployment）和發布（Release）被混為一談時，容易忽略的問題。

## 一、先備知識與問題定義
理解 HTTP、Client/Server、基本 if/else、平均值與百分比即可。本文延伸 MIS 的資訊系統變革與專案風險、CS 的軟體部署、監控與可用性，不需要先學 Kubernetes。

企業不只要問「新功能做得對不對」，還要問「要讓誰先用、出了問題多久能發現、能不能把影響限制在少數人」。測試無法完全重現正式環境的流量、裝置、資料與第三方服務，因此發布需要一套可觀測、可暫停、可回退的控制流程。

## 二、部署不等於發布：Feature Flag 的核心
部署是把程式碼或服務版本放進執行環境；發布是讓使用者真正接觸新行為。Feature Flag（功能旗標）把兩者分離。例如程式中同時保留舊結帳與新結帳路徑，透過旗標決定使用哪一條。

```js
function checkout(user, cart, flags) {
  if (flags.newCheckoutEnabledFor(user.id)) {
    return newCheckout(cart);
  }
  return legacyCheckout(cart);
}
```

這是概念示意，不是完整可執行的付款系統。旗標評估服務若失效，必須預先決定安全預設值；不能讓付款結果依賴未受控的臨時切換。功能旗標也不會自動撤銷已完成的付款、資料庫結構變更或外部 API 副作用。

Microsoft Azure 的架構文件指出，功能旗標可用於在不重新部署程式碼的情況下，逐步向特定使用者或租戶開放功能。Unleash 文件也區分 Release、Experiment、Operational、Kill switch 等用途。

## 三、Canary Release：為什麼不是直接開放 100%？
Canary（小規模先行發布）先讓一小部分使用者接觸新版本，觀察錯誤率、延遲、業務指標，再決定是否擴大。假設一項變更先開放 5%，確認穩定後提高到 20%、50%、100%。這些數字是**示例發布階段，不是通用安全標準**；實際門檻須依流量與風險決定。

部署環（Deployment Ring）可以按照內部測試者、早期採用者、一般使用者分組。Feature Flag 是控制新行為的機制，Canary 是降低曝險的發布策略，兩者常搭配但不相同。Blue-Green Deployment 則是準備兩套可切換的執行環境，也不等於 A/B Test。

## 四、5% 是使用者還是請求？一致分組很重要
若每一次 HTTP 請求都隨機抽 5%，同一位顧客可能在結帳流程前半段看到新版、後半段卻落到舊版，造成狀態不一致。對需要連續互動的功能，通常應以穩定的 userId 或 accountId 做分組（Stickiness），並確保各服務使用相同的評估條件。

Unleash 的漸進發布策略會利用穩定識別碼進行一致分組；這與單純每次擲硬幣不同。若沒有登入帳號，可使用有明確有效期的匿名識別碼，但仍要考慮隱私、換裝置與登入後身分合併問題。

此外，5% 的使用者不一定代表 5% 的流量：高活躍客群可能貢獻大部分請求。因此要同時觀察「受影響使用者數」與「實際請求量」。

## 五、如何知道能不能繼續擴大？SLO 與護欄指標
只看「伺服器還活著」不足以判斷發布成功。可定義服務水準目標（SLO），例如結帳 API 成功率、P95 延遲與訂單完成率。SLO 是目標，SLI 是實際量測的指標；SLA 通常是對外的服務承諾，三者不能混用。

以新結帳功能為例，建議建立四類護欄：
- 技術可靠性：HTTP 5xx、逾時率、P95 延遲、錯誤追蹤。
- 業務結果：付款成功率、訂單完成率、重複訂單率。
- 使用者體驗：客服申訴、特定裝置／地區的失敗比例。
- 發布控制：目前旗標比例、受影響人數、回退耗時。

**只看整體平均值很危險。** 若新版只占 5% 的流量，整體錯誤率可能被舊版稀釋。應依新舊版本、裝置與客群分層觀察，並避免只挑有利指標。

## 六、停止與回退：Kill Switch 不是萬能
Kill Switch 可以停止新請求進入有問題的功能路徑，縮短曝險時間；但已寫入的訂單、外部扣款與資料遷移不會因此自動復原。真正的回退計畫應包括：
1. 停止擴大並關閉新路徑；
2. 保留錯誤日誌、請求 ID 與版本資訊；
3. 對已受影響的交易執行狀態查詢、補償或人工處理；
4. 若涉及資料庫 Schema 變更，事先設計向後相容的 Expand–Migrate–Contract 流程；
5. 事後檢討偵測時間、影響範圍與根因。

還要為旗標設定負責人、用途、到期日與移除計畫。永久留下大量舊路徑會增加測試組合與維護成本；旗標本身也成為需要治理的技術負債。

## 七、MIS × CS：用風險而不是直覺決定發布速度
CS 提供分流、監控、日誌、回退與測試機制；MIS 則需要決定哪些風險可接受、由誰批准擴大、哪些 KPI 構成停止條件。這是資訊系統治理，不只是工程團隊的設定。

假設新結帳功能能提高轉換率，但也可能增加付款錯誤。管理者不能只比較功能開發成本，還要評估故障時的退款處理、客服人力、品牌信任與法遵成本。高風險付款功能可能需要較長觀察期；低風險的文字版面調整則可使用不同策略。發布政策應依變更風險分級，而非所有功能套用相同百分比。

## 八、可重現小型實驗：模擬逐步發布（提案／待驗證）
建立 Node.js 模擬器，產生 10,000 位固定 userId，使用穩定雜湊將使用者分到新舊功能。設定 5%、20%、50% 的發布比例，並**人工注入**新版錯誤率較高的情境，觀察分流比例、相同使用者是否保持同一路徑、各組錯誤率與停止條件。

實驗必須明確標記「模擬故障」，不能把人為注入的錯誤率當成真實企業資料。可以先驗證以下不變條件：
- 同一 userId、相同設定下，多次請求分組結果一致；
- 由 5% 擴至 20% 時，原本已分到新版的使用者不應無故回到舊版（需使用一致的分桶方法）；
- 若新版錯誤率超過預先定義的護欄，模擬控制器應停止擴大並記錄決策；
- 停止新流量不會刪除已產生的業務紀錄。

保留程式碼、固定亂數種子、設定檔、CSV、執行日誌與圖表，並在報告說明樣本數、觀察時間、統計波動及無法代表真實生產流量的限制。**目前未執行實驗，也沒有任何實測結果。**

## 九、研究所考題與解析
**題目：** 某電商準備上線新版結帳流程，工程師提議直接部署到全部伺服器，主管希望降低營運風險。請比較 Feature Flag、Canary Release 與 Blue-Green Deployment，並提出兩項可量測的停止條件。

**參考答案：** Feature Flag 控制誰能使用新行為，部署後也可關閉；Canary Release 先對少量流量或客群開放，觀察後逐步擴大；Blue-Green Deployment 以兩套環境降低切換與版本回退難度，但未必自動完成使用者分流或資料回復。可設定新版 5xx 錯誤率與 P95 延遲的護欄，搭配付款成功率等業務指標。指標必須區分新舊版本，設定觀察窗與最低樣本量，避免因少量隨機波動做錯決策。

## 十、結論
成熟的資訊系統不應把「程式部署成功」視為「業務變更成功」。Feature Flag 決定功能對誰可見，Canary Release 限制初期影響，SLO 與業務指標提供是否擴大的依據，Kill Switch 與補償流程則協助處理異常。技術措施能降低風險，卻不能取代企業對資料正確性、顧客影響與責任分工的治理。

## References
1. [Microsoft Azure — Safe deployment practices](https://learn.microsoft.com/en-us/azure/well-architected/operational-excellence/safe-deployments)
2. [Microsoft Azure — Multitenant solution updates](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/updates)
3. [Microsoft Azure — Feature management](https://learn.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management)
4. [Unleash — Feature flags](https://docs.getunleash.io/concepts/feature-flags)
5. [Unleash — Activation strategies](https://docs.getunleash.io/concepts/activation-strategies)
6. [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)

## SEO 與書審資訊
- SEO title：Feature Flag 與 Canary Release 怎麼降低上線風險？SLO、回退與 MIS 決策
- Excerpt：新功能測試通過，上線仍可能失敗。從 Feature Flag、Canary、SLO 與 Kill Switch，理解逐步發布、技術監控和企業營運風險。
- Slug：feature-flags-canary-release-slo-risk-governance
- Categories：MIS、CS、系統設計
- Tags：Feature Flag、Canary Release、Progressive Delivery、SLO、DevOps、MIS、Release Management
- 書審作品標題：以功能旗標與漸進發布降低資訊系統變更風險：可重現的流量分組與回退驗證
- 書審亮點摘要：針對新功能上線可能造成付款失敗與營運損失的問題，提出結合 Feature Flag、Canary Release 與監控護欄的風險控制設計。規劃以固定使用者分組與故障注入模擬逐步發布，驗證分流穩定性、錯誤偵測與停止條件，並保留程式、日誌及圖表，呈現技術取捨與資訊系統治理能力。實驗尚待執行。
- 代表作評估：中高（待完成實驗）；建議置於「自主學習／MIS × CS 跨域作品」，與資料庫索引、快取專題形成可靠系統設計作品群。