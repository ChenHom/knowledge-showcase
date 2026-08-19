# 媒體補充：Claude Design vs Huashu Design

## 來源
- 標題：Claude Design 剛推出就被逆向開源！Huashu Design：一句 Prompt 就能生出原型、簡報與 MP4 動畫
- 網站：電腦王阿達
- 日期：2026-04-21
- 連結：https://www.koc.com.tw/archives/639936

## 這篇對原筆記的補充價值
這篇文章不是在補 Claude Design prompt 的逐段內容，而是在補**外部產品脈絡**：
- Claude Design 推出後，市場怎麼看它
- 為什麼 Huashu Design 會快速出現
- Claude Design 的設計流程邏輯，哪些部分已經被 skill 化、CLI 化、agent 化
- 原本看起來像「提示詞細節」的東西，實際上如何被轉成產品能力、安裝流程與開源資產結構

## 從文章抽出的關鍵補充

### 1. Claude Design 不只是 prompt，而是一整套產品化設計工作台
文章把 Claude Design 描述成一個雲端設計工具，而不只是「很長的系統提示詞」。

它被描述的能力包括：
- 用對話生成設計稿、互動原型、簡報與 Landing Page
- 從團隊 codebase 抽品牌色票、字型、元件規範
- 從 DOCX / PPTX / XLSX 直接生成視覺版本
- 匯出成 PDF、PPTX、Canva、HTML
- 甚至交給 Claude Code 接手轉成完整網頁產品

### 2. 提示詞裡的很多規範，其實是產品工作流的骨架
從文章回看原 prompt，可以更清楚理解：
- 為什麼它那麼重視設計系統、品牌資產、codebase、元件庫
- 為什麼它一直要求先拿上下文，再做設計
- 為什麼它會把 output format、驗證、匯出、handoff 寫得這麼細

換句話說，這份系統提示詞不像一般 prompt 那樣只是在教模型「回答得比較好」，而是在定義一個**設計工作台的操作系統**。

### 3. Huashu Design 顯示：Claude Design 的核心可被 skill 化
文章最重要的外部觀察是：花叔把 Claude Design 玩過後，抽象化成一支跨 agent 可安裝的 skill。

這意味著：
- Claude Design 的核心不只是 GUI
- 它的價值很大一部分其實在於 workflow spec
- 這份 spec 可以被蒸餾成 agent skill，而不必完整複製 Anthropic 的產品形態

這也呼應你前面說的：這種等級的提示詞，真正值錢的地方不是字面本身，而是它背後的**模型治理方式**與**任務編排方法**。

### 4. 品牌資產協議（Brand Asset Protocol）是值得單獨標記的概念
文章提到 Huashu Design 吸收了 Claude Design 的一個核心觀念：
- 執行時自動從官方品牌來源抓取色票與視覺規範
- 讓產出維持一致，不要出現風格打架

這件事很重要，因為它說明原 prompt 內那些「先找設計系統 / 品牌 / 元件庫」的規則，不是裝飾性的建議，而是為了建立**品牌一致性協議**。

## 對原 prompt 的再理解

### A. 原 prompt 的重點，不只是生成，而是「先對齊設計上下文」
文章證明這點不是理論，而是產品層面的真需求。

如果沒有：
- 品牌色票
- 字型系統
- 元件規範
- codebase
- 真實設計資產

那高保真設計很容易變成 AI slop。這也是 prompt 中為什麼對「先問問題、先找上下文、不要從零硬做」寫得那麼嚴。

### B. 原 prompt 的提問段落，本質上是在做設計需求採樣
文章提到 Huashu Design 可做：
- 高保真可點擊原型
- HTML 簡報
- 時間軸動畫
- 即時參數微調
- infographic
- 五維度設計評審

這些輸出差異很大，因此 prompt 必須在一開始就問清楚：
- 輸出形式
- 保真度
- 變體需求
- 使用者更在意流程 / 文案 / 視覺哪一項

所以那段「怎麼問好問題」的文字，其實是在做一件很產品化的事：**把模糊設計需求轉成可執行 spec**。

### C. 原 prompt 同時也是一份 agent skill 規格書
文章裡 Huashu Design 的 repo 結構包括：
- `SKILL.md`
- `assets/`
- `references/`
- `scripts/`
- `demos/`

這剛好能反推一件事：Claude Design 的 prompt 並不是孤立存在的，它天然就適合被拆成：
- 規則層
- 資產層
- 範例層
- 匯出層
- 驗證層

也就是說，這種一線 prompt 很值得拿來學的，不只是 wording，而是它如何天然對應到**可維護的 skill / asset / script 架構**。

## 我認為最值得補進你原本筆記的三個洞察

### 1. 系統提示詞 = 產品操作系統，不只是語言指令
這篇文章讓 Claude Design 更像一個「以 prompt 為核心的設計作業系統」。

### 2. 真正值錢的是 workflow，而不是表面功能清單
快速被逆向、蒸餾、開源，代表核心價值在於：
- 任務拆解方式
- 提問順序
- 品牌資產對齊
- 驗證與交付規範

### 3. 好 prompt 之所以強，是因為它可外溢成 skill 與產品能力
Huashu Design 這件事本身就是證據：
- 好 prompt 不只會讓模型「回答更好」
- 它還能變成 repo 結構、starter assets、references、scripts、demos、export pipeline

## 建議補讀角度
之後如果要繼續研究這份 Claude Design prompt，可以從這三條線再深挖：
1. **Prompt → Skill**：哪些段落可直接翻成 skill 的執行規格
2. **Prompt → Product**：哪些段落其實是在替 GUI / 匯出 / 驗證 / handoff 定 contract
3. **Prompt → Brand Protocol**：哪些段落實際上是在保證品牌一致性，而不是單純的設計偏好
