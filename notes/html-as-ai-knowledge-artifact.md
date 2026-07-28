# HTML 作為 AI 知識展示格式：Claude Code 的實務補充

#html #claude-code #knowledge-showcase #artifact #workflow

## 這是什麼

這份筆記整理 Thariq（Claude Code / Anthropic）文章〈Using Claude Code: The Unreasonable Effectiveness of HTML〉對 HTML artifact 的觀點，並轉成 `knowledge-mirror` / `knowledge-showcase` 的實作規則。

來源：

- Thariq X Article：<https://x.com/trq212/status/2052809885763747935>
- HTML examples：<https://thariqs.github.io/html-effectiveness/>

## 核心結論

- HTML 不是單純取代 Markdown，而是讓 AI 產出從「文件」升級成「可讀、可分享、可互動的工作介面」。
- Markdown 的優勢仍是可版本控管、好 diff、適合作為 canonical source；HTML 的優勢是資訊密度、視覺結構、互動與分享。
- 對 `know` 來說，Markdown 應繼續保存知識原始稿；HTML 則應作為主要閱讀、分享與操作層。
- 不要只是把 Markdown 包成 HTML；應依內容類型加入適合的視覺結構，例如表格、卡片、流程圖、時間線、tabs、互動控制。
- HTML artifact 的理想用法是雙向：人用 UI 選擇、調整、比較，最後能匯出 prompt / plan / spec，餵回 agent 繼續執行。

## 文章重點理解

Thariq 的主張是：Markdown 曾經很適合 agent 與人溝通，因為它簡單、可攜、容易手改，也能表示基本結構。但當 agent 產出越來越長、越來越複雜時，Markdown 會變成資訊密度太低的文字牆。

他觀察到幾個變化：

1. 長 Markdown spec 超過一百行後，很少人真的完整讀完。
2. 現在人通常不是直接手改 AI 產出的文件，而是把它當 spec、reference、brainstorming output，再要求 Claude 修改。
3. 既然人不常手改，Markdown「易手動編輯」的優勢就下降。
4. HTML 能表示表格、CSS 設計資料、SVG 圖、script snippets、互動元素、workflow、canvas 空間資訊與圖片。
5. HTML 更容易分享：上傳後一個連結就能打開，不需要收件人下載或使用特定 Markdown renderer。
6. HTML 可以做 two-way interaction：滑桿、選項、比較器、copy prompt button，讓人選完後把結果餵回 Claude Code。
7. Claude Code 適合產 HTML，因為它能讀取本地檔案、git history、MCP、瀏覽器等上下文，做出更貼合專案的 artifact。

## 對 know 的新規則

### 1. HTML 不是「轉檔」，而是「展示設計」

新增 know HTML 時，不應只把 Markdown 直譯成段落。至少要思考：

- 這篇內容的核心資訊結構是什麼？
- 使用者最想比較、查找、決策的是什麼？
- 有沒有適合用卡片、表格、流程圖、時間線、tabs 呈現的部分？
- 是否需要搜尋、篩選、展開/收合、複製 prompt 等輕互動？

### 2. 依內容類型選 HTML 型態

| 內容類型 | 建議 HTML 型態 |
|---|---|
| 架構 / 系統設計 | 架構圖、資料流、元件卡片、責任邊界表 |
| 流程 / SOP | stepper、流程圖、checklist、風險提示 |
| 多方案比較 | 比較矩陣、tabs、並排卡片、選項篩選 |
| 技術文章整理 | 摘要卡、重點區塊、範例程式碼、注意事項 |
| 專案狀態 / 報告 | timeline、狀態燈、風險表、指標卡 |
| Prompt / workflow | profile 卡片、輸入輸出範例、copy button |
| 研究 / 學習 | glossary、collapsible sections、tabbed examples |

### 3. HTML 要支援讀者快速掌握

每個分享頁至少要有：

- 標題與一句話摘要
- 核心結論
- 可掃讀的章節導航或清楚 heading
- 可讀連結文字，不顯示整串 URL
- 手機可讀的 responsive layout
- 來源與更新時間

### 4. 能互動時就加輕互動

若內容適合，優先加入：

- 搜尋 / filter
- tabs
- collapsible detail
- copy prompt / copy command button
- 比較卡片
- timeline
- checklist

但互動應保持單檔 HTML 可用，不要過度依賴外部套件。

### 5. Public showcase 要更克制

同步到 `knowledge-showcase` 前要確認：

- 不含 private path、token、credential、內部不可公開內容。
- 連結文字用短標籤，例如「原文」、「官方文件」、「GitHub repo」。
- 本機或內部 URL 不應公開；必要時改成說明性文字。

## 實作建議

目前 `knowledge-mirror` 的 HTML pipeline 應升級成：

```text
Markdown source
→ 擷取 title / tags / excerpt
→ 依內容類型選 HTML template
→ link label cleanup
→ index manifest
→ public sanitization
→ sync to knowledge-showcase
→ commit + push
→ probe Pages URL
```

短期可以先維持通用文章模板；之後遇到特定類型時逐步加入專用模板：

- `architecture.html`：架構圖 + 元件表
- `sop.html`：stepper + checklist
- `comparison.html`：比較矩陣
- `prompt-profile.html`：copy prompt + input/output examples
- `report.html`：timeline + 指標卡

## 可直接套用的 Prompt

```txt
請根據這份 Markdown 筆記產生單檔 HTML 分享頁。
要求：
1. 不只是 Markdown 轉 HTML，要重新設計閱讀結構。
2. 依內容加入適合的卡片、表格、流程圖、tabs 或 checklist。
3. 手機可讀，CSS 內嵌，不依賴外部 CDN。
4. URL 不要直接顯示完整字串，改用可讀連結文字。
5. 不包含任何 private path、token、credential。
6. 頁面需包含標題、一句話摘要、核心結論、來源與更新時間。
```
