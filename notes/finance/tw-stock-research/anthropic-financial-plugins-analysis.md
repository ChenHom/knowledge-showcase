# Anthropic financial-services-plugins 分析：如何轉譯到台股研究場景

## 目的
這份筆記不是要直接照搬 Anthropic 的 `financial-services-plugins` repo，也不是先急著建 skill。

目標是先釐清：
1. 這個 repo 真正解決什麼問題
2. 哪些結構與方法論值得借鏡
3. 哪些內容不適合原樣照搬
4. 若放到台股研究場景，應該如何轉譯
5. 哪些金融知識與 workflow 最值得先整理

參考來源：
- https://www.blocktempo.com/anthropic-claude-financial-services-plugins-41-skills-11-data-providers/
- https://github.com/anthropics/financial-services-plugins

---

## 一、先講結論
這個 repo 最有價值的，不是「有 41 個 skills」，而是它把金融工作拆成了可重複使用的四層：

1. **Core financial analysis**
2. **Function workflows**
3. **Data connectors**
4. **Deliverable templates / commands**

也就是說，它不是單純提供很多 prompt，而是在做：

> 把金融專業流程模組化，讓 AI 能從資料一路走到成品。

這個方法論很值得借鏡。

---

## 二、這個 repo 在解什麼問題
傳統金融研究 / 投行 / PE / 財富管理的共同問題是：
- 資料分散
- 工作流高度重複
- 產出格式固定
- 但知識與流程常散在 analyst 腦袋裡

Anthropic 這個 repo 想解的是：
- Claude 不只是回答問題
- 而是知道該做哪種分析
- 知道應該先做哪一步
- 知道去哪抓資料
- 知道最後要交付什麼形式的 work product

所以它的核心不是聊天能力，而是：

> **workflow orchestration for financial work**

---

## 三、repo 結構真正值得學的地方

### 1. Core plugin 的概念很重要
它先把共用底層抽成一個 core：
- financial analysis
- comps
- dcf
- lbo
- 3-statement model
- connectors

這表示它在做的是：

> 先有「共用金融分析底座」，再讓不同部門 workflow 疊上去。

#### 借鏡點
如果未來要做自己的金融系統，也應先有：
- **研究核心層**
- 再往外長不同場景能力

---

### 2. 它不是只整理知識，而是整理流程
這個 repo 不是只告訴你：
- DCF 是什麼
- comps 是什麼

它更像是在定義：
- 使用者想做這件事時，Claude 應該如何走流程
- 應該先拿什麼資料
- 哪些步驟不能漏
- 最後應該輸出什麼

這種「流程知識」比靜態金融知識更有操作價值。

---

### 3. 它把 output 當一級公民
金融工作很多時候不是只要一個答案，而是要一個 deliverable：
- report
- memo
- deck
- tearsheet
- briefing
- note

這個 repo 很清楚知道：

> 金融工作最後要交的是 work product，不只是 insight。

這一點非常值得保留。

---

### 4. 它把 connector 跟 workflow 分開
這點很關鍵。

它沒有把 skill 寫死成：
- 一定要用 FactSet
- 一定要用 PitchBook

而是把資料來源獨立成 connector layer。

#### 好處
- workflow 可重用
- 資料源可替換
- 沒有高級資料也能先跑公開資料版

#### 對目前系統尤其重要
現在不該把研究流程綁死某個付費數據商，而應保留：

> **研究流程 ≠ 特定資料商**

---

## 四、repo 中隱含的重要金融知識

### A. Financial analysis 是共用底座
這層隱含的核心知識是：
- 如何理解公司經營
- 如何做估值
- 如何建模
- 如何 cross-check assumptions
- 如何把數據變成判斷

#### 關鍵知識類型
- valuation thinking
- model structure
- sensitivity / scenario thinking
- assumption discipline
- output formatting discipline

---

### B. Equity research 線
背後真正重要的知識是：
- earnings update 怎麼做
- thesis 怎麼寫
- catalyst 怎麼追
- morning note / initiation 類輸出如何組

#### 值得抽出的知識
- 哪些東西算 thesis
- 哪些東西算 risk
- 哪些東西算 catalyst
- 財報後哪些變化值得提

這非常適合轉成台股研究知識。

---

### C. Investment banking 線
本質是：
- deal materials production
- buyer list / process management
- merger model
- transaction workflow

#### 值得學的不是 jargon
而是：
- 高壓 deadline 下的 structured output
- 對標準 deliverable 的嚴格格式要求
- 把資料整理成可交付文件的能力

---

### D. Private equity 線
這條線的重要知識在於：
- sourcing
- diligence
- IC memo
- KPI tracking
- portfolio monitoring

#### 值得抽出的知識
- thesis / anti-thesis 對照
- diligence checklist 思維
- 風險與報酬拆開評估
- 投資決策要 memo 化

這對台股中長期研究也有幫助。

---

### E. Wealth management 線
這條線偏向：
- client review
- planning
- portfolio rebalance
- reporting

#### 值得保留的不是財管產品知識本身
而是：
- 如何針對不同對象生成可溝通的輸出
- 如何做定期 review
- 如何把建議轉成可理解內容

這比較像「如何把分析轉成可讀決策輸出」。

---

## 五、對目前系統最值得先保留的 5 類知識

### 1. Thesis / Anti-thesis
這很重要，因為很多研究只剩單邊敘事。

最值得標準化的是：
- thesis
- anti-thesis
- catalyst
- risk

---

### 2. Earnings update framework
高價值內容之一。

財報後不只是抄數字，而是要回答：
- 什麼變了？
- 哪個變化最重要？
- 是短期噪音還是中期轉折？
- 市場最可能在意什麼？

---

### 3. Catalyst / Risk tracking
這對台股尤其重要。

因為很多標的不是純估值問題，而是：
- 法說催化
- 月營收催化
- 新產品出貨
- 客戶砍單
- 匯率變動
- 原料價格
- 產業 cycle

---

### 4. Deliverable discipline
應保留這些輸出觀念：
- one-pager
- memo
- briefing
- note

因為真正有價值的不是分析片段，而是：

> **可交付、可比較、可持續追蹤的格式**

---

### 5. Connector independence
這不是金融知識本身，但對系統設計非常重要。

未來即使資料來源不同：
- 公開資訊觀測站
- 公司公告
- 法說會資料
- 新聞
- 使用者內部筆記

都應該被視為可替換的資料層，而不是把研究流程綁死在特定資料商。

---

## 六、哪些內容不適合直接搬進來

### 1. 美股 institutional workflow 原樣照抄
台股與美股 sell-side / IB / PE 的資料環境、節奏、輸出習慣不同。

### 2. 高級資料商依賴
如果沒有授權，不應先把流程寫成依賴 Bloomberg / FactSet / PitchBook。

### 3. plugin / command 生態完整照搬
現在不是在 Claude plugin marketplace 裡執行，因此要借的是方法論，不是檔案結構 1:1 複製。

### 4. 過度強調輸出像專業報告
比起「像 report」，更重要的是：
- 框架正確
- 缺口明確
- 觀點可追蹤

---

## 七、放到台股場景，應該怎麼轉譯

### 第一層：先抽研究框架
先不要急著建 skill，先整理出最值得重複使用的研究骨架：
- one-pager 框架
- earnings update 框架
- thesis / anti-thesis 框架
- catalyst / risk board
- valuation framing

這些才是高價值知識模組。

---

### 第二層：再決定哪些要變 skill
等真的常用、常重複，再把其中某幾個封裝成 skill。

最可能優先變 skill 的會是：
- 台股公司 one-pager
- 財報後更新
- catalyst / risk tracker

---

### 第三層：資料來源最後再接
先跑公開資料版，
未來如果有更好的資料源，再外掛 connector 層。

這順序比一開始就碰 connector 健康很多。

---

## 八、如果套到你現在的系統，最合理的使用方式

### 單 agent
適合先做：
- 公司研究摘要
- 財報後更新
- thesis / risk / catalyst 梳理
- 決策前 briefing

流程應是：
1. 定義研究任務
2. 收斂可用資料
3. 依固定骨架分析
4. 明示不確定性
5. 最後再用 prose 做輸出整形

---

### 多 agent
更適合的方式是：
- 某個 agent 負責資料蒐集與初稿
- evaluation-qa 檢查論證、缺口、scope 漂移
- orchestrator 收斂
- prose 做最後輸出整形

這樣比直接把整份工作塞給單一 skill 更穩。

---

## 九、台股場景下最值得先整理的知識模組
依優先序，我會先整理這些：

1. 台股公司研究 one-pager
2. 財報後更新框架
3. thesis / anti-thesis / catalyst / risk 骨架
4. 估值 framing 模板
5. 決策前 briefing 格式
6. 法說重點整理框架
7. 月營收追蹤框架
8. 供應鏈 / 產業鏈脈絡整理方式
9. 風險清單標準化格式
10. 研究缺口與不確定性標記方式

---

## 十、最後結論
這個 repo 對目前系統最有價值的，不是 41 個 skills 本身，而是：

> **把金融專業知識變成可重複工作流 + 可交付輸出的方法。**

如果要用在台股場景，正確順序不是先建 skill，而是：
1. 先抽知識骨架與研究 workflow
2. 先從公開資料可達版本開始
3. 把 deliverable format 定清楚
4. 真正常用後，再封裝成 skill / sub-workflow

這樣比較務實，也比較能跟目前 OpenClaw + multi-agent 系統接得起來。
