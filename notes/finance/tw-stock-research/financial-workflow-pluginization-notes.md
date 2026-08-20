# 金融工作流如何被 plugin 化、connector 化、deliverable 化

## 這份文件在講什麼
這份筆記不是在講某一種金融分析技巧，而是在整理一個更底層的系統問題：

> 一套金融專業工作，如何被拆成可重複使用、可維護、可擴充的 workflow system。

從 Anthropic `financial-services-plugins` repo 來看，真正值得學的，不只是金融內容，而是這三件事：

1. **plugin 化**：把不同角色 / 工作流封裝成獨立能力包
2. **connector 化**：把資料來源與工具接面獨立成可替換層
3. **deliverable 化**：把分析結果固定成專業交付物，而不是只有自由回答

這三件事加在一起，才構成它真正的方法論價值。

---

## 一、什麼是 plugin 化
plugin 化不是把 prompt 存成很多檔案而已。

它真正的意思是：
- 把某一類專業工作封裝成一個清楚的能力邊界
- 讓系統知道何時該用它
- 讓這個能力包可以被獨立安裝、維護、擴充

### 在金融工作流裡，plugin 化通常解決這些問題
- 不同職能（研究、投行、PE、財管）需要不同工作流
- 但它們又共享一些底層分析能力
- 如果不拆成 plugin，所有知識會混在一起、難以維護

### plugin 化的好處
1. **邊界清楚**
   - equity research 做什麼
   - investment banking 做什麼
   - private equity 做什麼

2. **易於擴充**
   - 後續要新增 partner-built plugin 或新 workflow，不必重寫整個系統

3. **可按場景裝載**
   - 使用者只載需要的那一層

### plugin 化最關鍵的觀念
不要把「整個金融分析宇宙」塞進單一 skill。

比較好的做法是：
- **core plugin** 放共享底座
- **add-on plugins** 放具體工作流

也就是：
> 先有 shared foundation，再疊 workflow overlays。

---

## 二、什麼是 connector 化
connector 化的核心在於：

> workflow 不應綁死在單一資料來源。

這在金融領域尤其重要，因為資料來源常常：
- 很昂貴
- 有授權限制
- 在不同團隊、公司之間並不相同

### connector layer 在解什麼問題
它把下面兩件事分開：
1. **我要做什麼工作流**
2. **我透過哪個資料源或工具取得資料**

例如：
- 做 earnings update 是 workflow
- 用 FactSet / LSEG / MOPS / 公司官網拿資料，是 connector 層問題

如果兩者混在一起，之後一換資料源，整個 workflow 就壞掉。

### connector 化的好處
1. **資料源可替換**
   - 有付費資料時走高級來源
   - 沒有時仍能退回公開資料版本

2. **workflow 可重用**
   - 核心分析邏輯不必重寫

3. **系統更容易擴展**
   - 新資料供應商可以新增 connector，而不是推翻既有 skill

### 在台股場景的意義
對台股來說，connector 化尤其重要，因為資料來源可能分成：
- 公開資訊觀測站
- 證交所 / 櫃買中心公告
- 法說簡報
- 月營收
- 公司官網
- 內部研究資料
- 付費在地或國際資料源

這些不應被寫死在同一份研究流程裡。

比較好的做法是：
- 先定 workflow
- 再定資料來源優先順序與替代路徑

---

## 三、什麼是 deliverable 化
deliverable 化的意思是：

> 分析的終點，不是「有答案」，而是「有可交付成果」。

金融工作最怕的是：
- 中途做了很多分析
- 但最後沒有固定產物
- 結果難以審閱、比較、更新、傳遞

### deliverable 化在金融場景中常見的形式
- one-pager
- earnings update
- initiation / long-form report
- memo
- briefing
- model workbook
- chart pack
- deck / slides

### deliverable 化的好處
1. **便於審閱**
   - 主管、PM、同事知道該看哪裡

2. **便於持續更新**
   - 下一次更新時，不是從零開始，而是覆寫既有交付物

3. **便於比較**
   - 不同公司、不同季度、不同投資主題可以套同一種輸出框架

4. **便於接 workflow**
   - one-pager 可以進一步延伸成決策 brief
   - earnings update 可以回寫 thesis tracker

### deliverable 化的真正價值
它迫使 workflow 在一開始就回答：
- 這次研究最後要交付什麼
- 用什麼格式
- 誰會看
- 哪些欄位不能缺

這比只問「要不要分析這家公司」成熟很多。

---

## 四、這三件事如何一起運作
plugin 化、connector 化、deliverable 化不是三件分開的事，它們應該一起工作。

### 最理想的關係
- **plugin** 定義「哪一類工作流」
- **connector** 定義「這個工作流可以接哪些資料 / 工具」
- **deliverable** 定義「這個工作流最後要交什麼成果」

可以把它想成：
- plugin = workflow 容器
- connector = data / tool interface
- deliverable = output contract

### 一個簡單例子：earnings update
- plugin 層：`equity-research`
- workflow：財報後更新
- connector 層：最新法說、財報、新聞、估值數據
- deliverable 層：一份 earnings update note / report

如果缺其中一層：
- 沒 plugin → 工作流邊界混亂
- 沒 connector → 資料來源寫死或不可替換
- 沒 deliverable → 研究沒有固定交付成果

---

## 五、為什麼這比單純 prompt library 更強
很多人第一次看這類 repo，會以為只是很多 prompt 檔案。

但真正成熟的地方在於，它不只告訴模型：
- 要做什麼

還會定義：
- 先做哪一步
- 資料優先順序是什麼
- 哪些錯法要避免
- 什麼情況要驗證
- 最後要交什麼格式

這代表它不是單純的文案集合，
而是：

> **把專業工作 SOP、資料接面與交付形式一起包進系統。**

---

## 六、如果要用在台股研究，應該怎麼借這套方法

### 可直接借的
1. **core + add-on 思維**
   - 先有共用研究底座
   - 再有財報更新、產業追蹤、決策簡報等不同 workflow

2. **workflow 與資料層分開**
   - 研究流程先獨立
   - 資料來源再做 mapping

3. **一開始就定義 deliverable**
   - one-pager
   - earnings update
   - decision brief
   - risk / catalyst tracker

4. **把 validation / anti-pattern 寫進 workflow**
   - 避免資料過期
   - 避免只抄新聞
   - 避免 thesis 與 catalyst 混淆

### 不該直接照搬的
1. 美式 sell-side 報告規格
2. Bloomberg / FactSet / LSEG 等資料假設
3. 機構級超重格式與頁數要求
4. 把台股研究直接包裝成固定 rating / price target 制式流程

---

## 七、最終結論
從這個角度看，Anthropic `financial-services-plugins` 最值得學的，不是某個金融分析細節，而是：

> **如何把金融專業工作拆成 plugin、connector、deliverable 這三層，形成可重複、可維護、可擴充的 workflow system。**

如果要應用到台股，最正確的方式不是直接複製美式內容，
而是先保留這三層結構，再重寫：
- 台股資料映射
- 台股研究 workflow
- 台股適合的交付物格式

這樣借到的才是方法論，不只是外殼。
