# AI-First Testing：收斂、探索與可重複執行

#ai-first-testing #decision-table #test-automation #mcp #accessibility #pest #playwright

## 這是什麼

這份文件整理安德魯〈AI-First Testing，以 AI 為核心重新設計測試流程〉的閱讀結果，以及閱讀過程中釐清的問題、工程補充與 Laravel 金流專案導入方式。

原文：https://columns.chicken-house.net/2025/11/10/ai-first-testing-workflow/

## 核心結論

這套流程不是讓 AI 取代測試框架，而是重新分配測試工作：

- 人腦負責判斷：決定什麼才是正確、哪些情境值得測。
- AI 負責收斂與探索：協助展開條件組合，找出實際操作與驗證步驟。
- 程式碼與 CPU 負責穩定重複：把已確認的流程轉成固定測試程式，由 CI 或排程反覆執行。

完整骨架：

```text
收斂該測什麼
→ 探索怎麼測
→ 轉化為可重複執行的資產
→ 持續維護規格與測試的一致性
```

文章主要處理前三段；第四段是正式導入後不可忽略的工程問題。

## Decision Table：把條件組合攤開

Decision Table 把含有多個判斷條件的驗收條件拆成：

- Criteria／Condition：會影響結果的條件。
- Action：系統在不同條件下應採取的行為。
- Rule：一組條件組合及其預期結果。

它的價值不是把需求寫得更長，而是把隱藏條件攤開，讓人檢查遺漏、衝突、不可能狀態與錯誤優先順序。

AI 可以產生第一版表格，但 Criteria、Action 與 Rule 是否符合領域規則，必須由領域專家確認。

### 建立與人工校正方向

- 所有條件皆成立：正常成功路徑。
- 所有條件皆不成立：全面失敗及錯誤優先順序。
- 部分條件成立：具有業務意義的混合組合。
- 邊界條件：等於上限／下限、低於下限、超過上限。
- 不可能狀態：標記並排除無意義組合。
- 多錯誤同時發生：確認應回傳哪一個錯誤。
- 條件相依：某條件是否只在另一條件成立後才需要判斷。

Decision Table 只能提高條件組合的可見性，不能自動保證涵蓋率。條件增加後仍會組合爆炸，需要搭配等價類別、邊界值分析、Pairwise、風險導向測試，以及歷史缺陷與正式環境事故。

人工校正的核心問題：

> 哪些條件真正影響結果、哪些組合實際可能發生，以及每種組合應產生什麼行為？

補充閱讀：https://ithelp.ithome.com.tw/articles/10375192

## 抽象 Test Case：把「測什麼」與「怎麼操作」分離

抽象 Test Case 只保留：

- Given：前置狀態。
- When：業務行為。
- Then：預期結果。

不寫死 API endpoint、HTTP Header、畫面按鈕、DOM selector 或測試框架。因此同一份 Test Case 可以搭配 OpenAPI、Web UI 或 App 操作規格，分別產生 API、Web 與 App 測試。

操作介面是 REST API、Web UI、Android、iOS；Pest、PHPUnit、Playwright、Appium 則是測試框架或工具，兩者不能混為一談。

抽象不等於模糊。測試仍必須明確描述前置條件、業務動作與預期結果，只移除技術操作細節。

## AI 所謂的探索

探索不是再次尋找漏掉的測試情境，而是針對已確定的 Test Case，實際找出：

- 如何建立測試資料。
- 如何登入或取得授權。
- 呼叫哪些 API 或操作哪些 UI。
- 結果應從哪裡驗證。
- 如何確認資料庫、事件、Queue 與外部呼叫等副作用。
- 如何清理及隔離測試資料。

AI 根據規格、系統回應與失敗結果反覆修正，直到找到可行流程。Session log 供人審查探索結果，也作為產生 Test Code 的操作範例。

探索屬於「測試步驟的驗證」，不是 CI/CD 每次執行的回歸測試。

## AI 臨場執行與固定測試程式的差異

Vibe Testing 是每次都由 AI Agent 重新閱讀 Test Case、操作系統、觀察結果並判斷成功或失敗。文章估算每次約兩分鐘與 USD 0.03，成本來自整個 Agent 推論與工具操作，不是單純啟動測試。

改良流程：

1. AI 對每個案例探索一次。
2. AI 將已驗證的步驟轉成 Pest、xUnit 或 Playwright 測試程式。
3. 後續由 CI、排程或人工指令直接執行固定程式。

文章的估算從 40,000 次 AI 任務，轉為 400 次探索、400 次程式碼產生，以及 40,000 次一般 CPU 測試。觸發者仍可都是 CI；真正改變的是執行者。

```text
舊：CI／排程 → AI Agent → AI 臨場操作與判斷
新：CI／排程 → 固定測試程式 → Assertion 判斷
```

傳統成熟的自動化測試本來就是由 CI 執行固定測試程式。文章修正的是作者先前「每次讓 AI 臨場測試」的 Vibe Testing 思路。

## AI 的角色與邊界

AI 可以：

- 協助展開可能需要測試的條件組合。
- 將選定規則整理為抽象 Test Case。
- 探索 Test Case 的實際操作與驗證流程。
- 將驗證過的流程產生為測試程式碼。

AI 不能自行決定業務上的正確答案。正確分工是：

> AI 提出與執行；人確認測試範圍、領域規則及預期結果。

測試也不能證明程式完全無誤，只能證明已定義且已涵蓋的條件目前符合預期。

## MCP 的建立理念

值得學的不是照抄作者的專案 MCP，而是工具設計：

- MCP 是 AI 與受測系統之間的受控工具層，不是測試框架。
- 工具應提供穩定、結構化、可判讀的輸入輸出。
- 登入、API 呼叫、狀態查詢、資料準備與清理應封裝為明確能力。
- 每次操作留下 session log、API 規格快照與稽核紀錄。
- 探索紀錄供人審查，並成為產生 Test Code 的規格輸入。
- 限制可操作環境、允許動作、資料範圍與敏感資訊。
- 不把沒有邊界的管理員 API 直接交給 Agent。

MCP 實作應另開專題，討論工具粒度、權限、可觀察性、錯誤格式、測試資料生命週期及安全邊界。

## 無障礙與 AI 可操作性

AI 操作網頁時，通常依賴 DOM、語意化 HTML、ARIA 與 Accessibility Tree，而不只是視覺辨識與座標點擊。

Playwright MCP 會從無障礙資訊提取精簡結構，降低完整 HTML 佔用大量前後文的問題。語意化結構越清楚，Agent 越容易找到元件並理解用途。

實務重點：

- 使用正確的 `button`、`label`、`input`、`nav` 等語意元素。
- 表單欄位具有明確 label。
- 按鈕與連結有可理解的名稱。
- 錯誤與狀態能以 ARIA 表達。
- 不只依賴顏色、座標或圖示傳遞功能。
- DOM 結構與實際操作意義一致。

無障礙的主要服務對象仍是人；AI Friendly 是相同語意基礎帶來的延伸效益，不應把無障礙縮減成 AI 可讀性工程。

## 從探索轉成 Test Code

轉化不能只是逐步抄寫探索紀錄。探索可能包含繞路、誤操作及偶然成功，應提取：

- 最小且確定的操作流程。
- 固定、可建立、可清理的測試資料。
- 明確的 assertion。
- 必要副作用與「不應發生」行為的驗證。
- 不依賴測試順序的隔離機制。
- 失敗時可以定位原因的輸出。

## 正式導入必須補足的問題

### Test Oracle

「怎樣才算正確」必須有可靠依據。HTTP 200 不代表完整成功，還可能需要驗證訂單、錢包、事件、Queue、外部呼叫、回滾與冪等性。若 Oracle 錯誤，AI 只會精準產生錯誤測試。

### 可重複性

一次探索成功不代表能穩定回歸。還要處理時間、亂數、非同步工作、外部服務、測試隔離、資料清理及環境差異。

### 規格漂移

需求、API、UI 或領域規則變更後，必須判斷：

- Decision Table 是否改變。
- 抽象 Test Case 是否仍成立。
- 只有操作方式改變，還是預期結果也改變。
- 哪些 Test Code 需要重新探索或產生。

### 維護責任

AI 可以降低首次建立測試的成本，但無法消除後續維護。正式流程需要追蹤 AC、Decision Table、Test Case、操作規格、session log 與 Test Code 的關聯。

## 套用到 Laravel 金流專案

先選擇條件多、風險高、結果明確的垂直流程，例如「代付訂單建立前置驗證」。

可能條件：

- 商戶是否啟用。
- IP 是否在白名單。
- 簽章是否正確。
- timestamp 是否在允許範圍。
- nonce 是否重複。
- 錢包與幣種是否有效。
- 可用餘額是否足夠。
- 通道是否啟用。
- 金額是否符合限額。

最小導入流程：

1. 選一項 AC。
2. 由 AI 產生第一版 Decision Table。
3. 由領域人員確認條件、錯誤優先順序與預期行為。
4. 收斂為約 5～15 個抽象 Test Case。
5. 探索其中 2～3 個高價值案例。
6. 讓 AI 依 Laravel route、Form Request、Service、Factory、Seeder 與既有 Pest 測試產生 Test Code。
7. 人工審查 assertion、資料隔離及副作用。
8. 納入 CI。
9. 經過數次需求變更後，再判斷是否值得建立專用 MCP。

第一階段不應先開發 TestKit 或 MCP。只有當登入、簽章、建立商戶、調整錢包、查詢事件及清理資料等探索工作反覆出現，才值得封裝。

## 最終判斷

這篇文章真正有價值的地方，不是提出新的測試框架，而是重新配置 Brain、GPU 與 CPU：

- Brain 用於領域判斷。
- GPU 用於不確定的探索與轉化。
- CPU 用於確定、廉價、穩定的重複執行。

需要保持的批判是：Decision Table、AI 探索與程式碼生成都只是降低工作成本，無法取代領域知識、Test Oracle、測試設計、維護策略與安全邊界。

## 參考資訊

1. [原文：AI-First Testing](https://columns.chicken-house.net/2025/11/10/ai-first-testing-workflow/)
2. [安德魯 2025/11/03 Facebook：AI Native 流程與新工具套舊流程](https://www.facebook.com/share/p/177qgafB5Z/)
3. [安德魯 2025/10/21 Facebook：新工具需要新流程](https://www.facebook.com/share/p/1BYCtCqZod/)
4. [楊大成 2025/10/11 Facebook：別拿新工具優化舊戰術](https://www.facebook.com/share/p/1HoM5rq2D9/)
5. [A16Z AI Era 閱讀心得：Accessibility as the universal interface](https://columns.chicken-house.net/2025/09/28/reading-a16z-emerging-developer-patterns-for-the-ai-era/)
6. [Decision Table Testing：Day 21 開立範例的方式－DTT](https://ithelp.ithome.com.tw/articles/10375192)
7. [Sam Altman：Young people use ChatGPT as an operating system](https://www.youtube.com/watch?v=uVEjlRK0VWE)
8. [安德魯 2025/09/08 Facebook：前後文雜訊對 Agent 能力的影響](https://www.facebook.com/share/p/1D7ZeAAZPT/)
9. [Context Rot：How Increasing Input Tokens Impacts LLM Performance](https://www.trychroma.com/research/context-rot)

更新日期：2026-07-17
