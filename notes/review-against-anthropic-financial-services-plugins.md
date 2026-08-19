# Review：目前台股研究筆記相對於 Anthropic financial-services-plugins 的偏誤與修正方向

## 目的
這份 review 的目的不是否定目前已整理出的台股研究筆記，而是重新對照 Anthropic `financial-services-plugins` repo，確認目前輸出是否有：
- 抓到原 repo 真正強的地方
- 把不該放大的部分放太大
- 漏掉原 repo 最關鍵的方法論

---

## 一、先講總結
目前這批文件：
- `tw-stock-research-core-notes.md`
- `tw-stock-thesis-antithesis-catalyst-risk.md`
- `tw-stock-earnings-update-framework.md`
- `tw-stock-one-pager-structure.md`
- `tw-stock-valuation-framing.md`

**不是錯，也不是沒用。**
它們對台股研究其實很實用。

但如果拿來對照 Anthropic repo，會出現一個明顯傾向：

> **重研究內容，輕 workflow system。**

也就是說，目前整理出的內容更像：
- 一組不錯的台股研究筆記

但 Anthropic repo 更像：
- 一套金融工作流系統設計

這就是目前最主要的偏誤來源。

---

## 二、目前有對齊 repo 的地方

### 1. 有抓到「研究輸出需要固定骨架」
目前文件已經有明確的結構化輸出觀念：
- one-pager
- earnings update
- thesis / anti-thesis / catalyst / risk
- valuation framing

這點和原 repo 的精神一致，因為原 repo 非常重視：
- 不是只給答案
- 而是給可交付、可重複的 work product

### 2. 有抓到「thesis / catalyst / risk」這類高價值金融知識
這點和原 repo 裡 equity research / PE 的部分相當一致。

因為原 repo 並不只是蒐集資料，而是強調：
- 哪些變化重要
- 哪些事件會改變市場看法
- 哪些風險會破壞投資邏輯

### 3. 有抓到「估值不是數字，而是估值邏輯」
這點也和原 repo 的 financial analysis 層比較一致。

目前的估值筆記有整理：
- 市場現在怎麼定價
- 估值邏輯可能如何切換
- 為何不要只看便宜或昂貴

這種寫法比只寫目標價更接近原 repo 真正有價值的地方。

---

## 三、目前最主要的偏誤

### 偏誤 1：把 repo 誤縮成「台股 equity research 筆記」
Anthropic repo 的範圍其實很廣：
- financial analysis core
- investment banking
- equity research
- private equity
- wealth management
- partner-built data plugins
- connectors
- commands
- deliverables

但目前產出的內容幾乎都偏向：
- equity research / buy-side style thinking

#### 問題
這樣會讓人誤以為原 repo 的價值只在：
- thesis
- 財報更新
- 催化劑
- 估值

其實它更大的價值在於：
- **把不同金融職能的 workflow 做成模組化系統**

#### 修正方向
後續應補一份獨立說明，專講：
- original repo 的 system architecture
- core / add-on / connector / deliverable 分層
- 為什麼這些分層比單一研究框架更重要

---

### 偏誤 2：重分析內容，輕 workflow orchestration
目前文件內容已經有很多研究判斷框架，但對下面這些寫得還不夠：
- 任務如何被觸發
- 研究流程的前後順序
- 哪些步驟屬於 shared core
- 哪些步驟屬於 specific workflow
- 哪些 output 是最終交付物

Anthropic repo 真正厲害的地方，不只是內容知識，而是：

> **它知道在什麼場景下，Claude 應該怎麼走完整工作流。**

#### 修正方向
未來如果要更貼近原 repo，應該補：
- one-pager workflow
- earnings workflow
- decision brief workflow
- source-to-deliverable workflow

而不是只寫內容骨架。

---

### 偏誤 3：低估了 connector layer 的重要性
目前文件大多聚焦在：
- 研究要看什麼
- 分析要怎麼寫

但對資料層的抽象還不夠強。

Anthropic repo 很重要的一個設計是：
- workflow 不綁死單一資料供應商
- connector 可以替換
- 沒有某個資料源時，workflow 仍能部分運作

#### 問題
如果只寫研究內容，不寫資料層抽象，之後系統很容易退化成：
- 一組漂亮的研究筆記
而不是：
- 一套可持續擴充的研究 workflow

#### 修正方向
應補一份資料層筆記，說明：
- 公開資料
- 內部資料
- 高級數據源
- 這三者如何分層，而不是混成同一件事

---

### 偏誤 4：對 deliverable discipline 還不夠重
目前文件提到 one-pager、earnings update、valuation framing，已經是好的開始。

但和原 repo 相比，還少了一層：
- 這些輸出是要交給誰看
- 在什麼決策場景用
- 輸出結構如何穩定化
- 哪些欄位是每次都不能缺的

Anthropic repo 真正強的是：
- output 不是附屬品
- output 本身就是 workflow 的終點

#### 修正方向
未來應加強：
- deliverable-oriented writing
- decision brief
- monitoring memo
- update memo
- research note lifecycle

---

## 四、目前文件各自的評價

### 1. `tw-stock-research-core-notes.md`
#### 優點
- 很適合當台股研究入門骨架
- 抓到研究的五個關鍵問題
- 有助於避免只抄新聞

#### 偏誤
- 比較像研究筆記的總綱
- 還不像 Anthropic repo 那種「workflow core layer」

#### 建議
保留，但後續應補一份更偏 system / workflow 的核心說明，與這份分開。

---

### 2. `tw-stock-thesis-antithesis-catalyst-risk.md`
#### 優點
- 內容實用
- 對台股研究判斷很有幫助
- 很適合當研究輸出的骨架

#### 偏誤
- 它比較像研究內容模組
- 不能代表原 repo 的整體設計價值

#### 建議
保留，並明確標記這是「內容模組」，不是 repo 的系統層總結。

---

### 3. `tw-stock-earnings-update-framework.md`
#### 優點
- 很像原 repo equity research 線中真正有用的部分
- 有明確的 post-earnings update 思維

#### 偏誤
- 仍偏內容分析，還沒寫成完整 workflow（資料來源 → 變化判讀 → 更新輸出）

#### 建議
可視為高價值內容文件，但未來應補 workflow 版。

---

### 4. `tw-stock-one-pager-structure.md`
#### 優點
- 跟原 repo 的 deliverable thinking 最接近
- 已經開始把研究輸出視為固定交付物

#### 偏誤
- 還可以更強調：這份 one-pager 是在什麼決策節點使用

#### 建議
這份很值得保留，並擴充成更明確的 deliverable 規格。

---

### 5. `tw-stock-valuation-framing.md`
#### 優點
- 把估值拉回估值邏輯，而不是只看目標價
- 對台股研究有很高實用性

#### 偏誤
- 比較像研究知識說明
- 和原 repo 的 connector / workflow / output chain 沒有直接對齊

#### 建議
保留，但不要把它誤當成 repo 系統設計的主軸。

---

## 五、如果要更貼近原 repo，下一步應補什麼

### 1. `workflow-system-notes.md`
專門說明：
- core / addon / connector / deliverable 的分層
- original repo 到底怎麼看待金融工作流

### 2. `research-data-layering.md`
說明：
- 公開資料層
- 使用者內部資料層
- 付費資料層
- 如何避免 workflow 綁死在單一來源

### 3. `deliverable-lifecycle.md`
說明：
- one-pager
- earnings update
- decision brief
- monitoring note

這些輸出之間如何接續，不只是各自獨立存在。

---

## 六、最終結論
目前產出的文件：
- **作為台股研究內容筆記是成立的**
- **作為 Anthropic repo 的完整方法論映射則仍不完整**

### 最核心的偏誤可以濃縮成一句話：

> 目前這批文件，把 Anthropic repo 的價值整理得太像「研究內容框架」，而還沒有完整保留它作為「金融 workflow system」的核心設計。

### 所以最正確的定位應該是：
- 現有文件 = **內容層 / 研究層**
- 尚待補的部分 = **系統層 / workflow 層 / connector 層 / deliverable chain**

這樣分類之後，整體會更準，也更不容易偏離原 repo 的真正強項。
