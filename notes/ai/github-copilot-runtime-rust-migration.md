# Github copilot runtime 移植到 rust 中文全文翻譯

#translation #rust #ai-agents #github-copilot #migration

## 用 Copilot 把 GitHub Copilot runtime 移植到 Rust

**原文**　[Migrating the GitHub Copilot runtime to Rust, using Copilot](https://github.blog/ai-and-ml/generative-ai/migrating-the-github-copilot-runtime-to-rust-using-copilot/)
**作者**　Stephen Toub（[@stephentoub](https://github.com/stephentoub)）
**日期**　2026 年 9 月 16 日
**篇幅**　約 65 分鐘閱讀

> 這種規模的重寫在 agent 出現之前根本不划算。以下是把 Copilot agent runtime 移植成 80 萬行生產級 Rust 實際付出的代價。

---

## 目錄

1. 為什麼我們需要移植
2. 移植前是什麼樣子
3. 原地移植策略
4. 起步
5. 互通（Interop）
6. session 資料透露了什麼
　6.1　一切都在於快取
　6.2　沒錯，靜態分析真的有幫助
　6.3　Agent 喜歡讀
　6.4　模型選擇
7. 與 agent 艦隊一起工作
8. 規模化的程式碼審查
9. 自動化內層迴圈
10. 一場遷移，兩種遷移
11. 有多少 `unsafe`？
12. 那些回歸
13. 能編譯，就只是能編譯
14. 效能、效能，還是效能
15. 這次移植花了多少
16. 學到的教訓
17. 接下來呢？

---

## 前言

[GitHub Copilot CLI](https://github.com/features/copilot/cli/)、[GitHub Copilot app](https://github.com/features/ai/github-app) 與 [GitHub Copilot SDK](https://github.com/github/copilot-sdk) 背後全都由 Copilot agent runtime 支撐——那是一套可以嵌入到應用程式與服務中的 agentic harness（代理框架）。

它最初是為了今天的 [GitHub Copilot cloud agent](https://docs.github.com/copilot/how-tos/use-copilot-agents/cloud-agent)（CCA）而以 TypeScript 寫成，跑在 Node.js 與 V8 JavaScript 引擎上；即使這個 runtime 與它的能力快速成長，它也一直留在那套技術堆疊上。

現在情況改變了。我們用 GitHub Copilot app 與 Copilot CLI，把整個 runtime 完全重寫成超過 80 萬行的生產級 Rust。程式碼大多是由 AI agent 寫的，橫跨 128 個合併進 `main` 的 pull request，而且是逐步出貨（ship incrementally），而不是等到最後一次性切換。

過程中少數難免出現的回歸（regression）都很快被發現並修掉，同時 runtime 的效能提升了好幾個數量級。這個在 agent 出現之前需要一整個開發團隊花上一兩年的專案，如今主要由一位開發者在短短數個月內完成，而團隊其餘成員在這段期間仍持續大幅擴充 runtime 的能力與觸及範圍。

---

## 1. 為什麼我們需要移植

Copilot agent runtime 不只是 Copilot CLI 背後的引擎。它支撐著一整群持續成長的 Microsoft、GitHub 與生態系解決方案；對這些方案而言，AI 支援在架構上就是「同一個 runtime 外面包一層殼，加上該方案自己需要的客製化」。

這不只包含 GitHub Copilot CLI 與 GitHub Copilot app，也包含最新版的 VS Code、Visual Studio、CCA、Copilot Code Review（[CCR](https://docs.github.com/copilot/concepts/agents/code-review)）、Copilot Cowork、Copilot Studio，以及 Excel、Outlook、PowerPoint、Word……族繁不及備載。

這些產品彼此差異極大，而且沒有任何一個會想（也不應該需要）自己實作一套生產級 agent harness 裡的所有東西。它們要的是完整的智慧、安全性、可靠性與效能，而且希望這些是共享的——這樣在一個地方修好，就等於在所有地方都修好。

上一段列出的產品大多一開始都自己實作了 agent loop（代理迴圈），但後來都改用 GitHub Copilot SDK，也就是進入 Copilot agent runtime 的入口。這麼做讓它們可以專注在自己的核心商業價值上，把細節交給 runtime。考慮到整個產業的推進速度，以及在激烈競爭下 agent loop 必須永遠保持業界最佳水準，這一點更顯重要。

所以：共享 runtime，很好。問題出在「被共享的那個東西」本身的性質。

### 原本的堆疊

以 CLI 來看，它在邏輯上就是一個疊在 agent loop 之上的終端機介面（TUI）。而實際上，整個堆疊都是用 TypeScript 實作的，用 Node.js 當框架、V8 當執行引擎，UI 用 Ink 加 React。

以一個 TUI 應用程式來說，這是個相當得體的選擇；TypeScript 與 Node.js 門檻低、普及廣，能支撐非常快速的應用開發。而以主控台應用程式的需求而言，它在啟動時間、回應速度、吞吐量與記憶體消耗上的效能影響也還算合理。

但很不幸地，當你把這套實作放到其他環境、面對其他限制、要求快速啟動與極低記憶體額外開銷帶來的優異伺服器密度時，這些就一點都不合理了。

CLI 及其 runtime 的架構也加深了這方面的困難。整個產業跑得極快，在那樣的情境下，非常聰明的人會為了交付速度與市場觸及做出取捨。Copilot CLI 當初寫得快、出貨也快，過程中 TUI 與 runtime 交纏在一起，沒有被切成清楚的分層。

後來需要一個 SDK 來以程式方式存取這個 runtime 時，由於層次沒有明確分離，於是做了一個務實的決定：把 SDK 疊在 CLI 之上——即使從邏輯上你會期待的是反過來的架構。

於是 CLI 不再只能透過使用者在命令列輸入的指令來使用，而是新增了一種 headless（無介面）模式，從 stdin 讀取類似的指令、把回應寫到 stdout。接著就可以用 JSON-RPC 協定，把函式呼叫在外部行程與 CLI 之間來回封送（marshal）。

SDK 因此可以被嵌入任意的消費端程式中，由它去生出一個 CLI 行程來在行程外托管 agent loop，SDK 再透過這套 JSON-RPC 機制呼叫遠端行程裡的函式。很巧妙。出貨很快。也很有彈性。但對那些消費端應用程式的效能（啟動、記憶體、吞吐量）與可靠性來說，並不理想。

從 SDK 建立一個新的 `CopilotClient`，意味著又生出一個行程：

```js
const client = new CopilotClient();
await client.start(); // spawns the CLI as a subprocess
const session = await client.createSession({
    /* ... */
});
```

### 這個做法的代價

這個行程必須啟動並托管 Node 與 V8。這代表：

- 要解析 CLI 的 TypeScript 產生出來的大量 JavaScript、為它產生 bytecode，還可能要在後續的 JIT 層級中最佳化熱點程式碼。
- 要承擔 V8 的所有記憶體額外開銷。
- 要繼承 Node 的執行緒模型——它預設會把我們推向「所有 CPU 密集工作都被序列化」的模式。
- 光是要做函式呼叫，就被迫走跨行程通訊。
- 每一個 SDK 消費端，不論用什麼語言，都得夾帶 Node.js，或是一個內含 V8 的打包執行檔。
- C#、Python、Go、Java 與 Rust 的 SDK，全都要為每一個 client 付出一整套第二語言 runtime 的代價——最少也是約 100 MB 的工作集（working set），而那套 runtime 對他們的應用程式本身毫無用處。
- 每一個事件、每一則訊息、每一次被抽象化的 session 檔案系統讀寫，都要被推過一道行程邊界。
- Node 一崩潰，整個 session 就跟著陪葬。
- 任何部署這套東西的人，最少都有兩個行程要監督、監控與除錯。

### 我們想要的 runtime

- 不包含 TUI；TUI 本身是一個獨立函式庫，TUI 與其他應用程式、服務可以乾淨地正確疊在它之上。
- 用一種相依性極少、額外開銷極低的語言實作。
- 實作方式能夠乾淨地在行程內（in-process）嵌入，而不是被迫跨行程。
- 用一種在效能、可擴展性與可靠性上都具有頂尖特性的語言實作。
- 用一種非常適合互通（interop）的語言實作，這樣六個 Copilot SDK 語言版本（C#、TypeScript、Python、Rust、Go、Java）都能用各自技術堆疊的外部函式介面（FFI，foreign function interface）機制乾淨地使用它。
- 用一套能提供更現代安全態勢的工具鏈實作，供應鏈風險更低，並且對「建構即正確」（correct-by-construction）的程式碼有更好的支援。

基於上述所有理由，再加上一些較軟性的理由（例如團隊經驗與產業走向），我們選擇了 Rust。

這完全不是在主張每一個大型 TypeScript 程式都該變成 Rust。我們的需求強調的是透過 C ABI 進行嵌入、低啟動與低穩態開銷，以及可預期的資源使用。Rust 讓這些目標成為可能，代價則是其他面向的複雜度——例如我們必須顯式地表達生命週期（lifetime）與共享狀態（後面談到的生命週期回歸問題就凸顯了這一點的後果）。正確的目標語言，確實會因應用程式而異。

接著有兩項彼此相關的關鍵工作要做：

1. **拆分層次。** 把 TUI 專屬的程式碼從 runtime 中分離出來，讓前者嚴格疊在後者之上，更具體地說是嚴格疊在 SDK 的公開介面之上。到今天為止，CLI 在好幾個地方仍然直接呼叫 runtime 的內部；把它完全搬到 SDK 的公開介面上是還在進行中的工作。
2. **移植 runtime。** 把那一層 runtime 移植成 100% Rust，產出一個純原生二進位檔，對外暴露一個 C ABI 供所有語言前端在行程內使用，並在仍需要跨行程時提供基於 stdin/stdout 或 socket 的伺服器。

這篇文章主要談的是第二項：把 runtime 移植到 Rust。

![圖 1](https://github.blog/wp-content/uploads/2026/09/architecture-before-after.svg)

*圖 1：Rust 重寫前後的 Copilot runtime 架構圖，顯示舊的「SDK 到 CLI」行程邊界，以及新的行程內與行程外托管路徑。*

---

## 2. 移植前是什麼樣子

2026 年 5 月初的初版移植計畫，估計 runtime 大約有 13 萬行 TypeScript。就估算範圍而言，這個初始數字還算準確，但結果證明它同時也極度誤導人，原因有兩個。在移植的同時：

1. 仍被包在 TUI 層裡的片段正在被下推到 runtime 層。整批整批的元件，以及當初估算時被忽略的大量程式碼，後來都被認定與移植相關。
2. 不斷有 pull request 貢獻大量新的 TypeScript，讓 repo 裡的 TypeScript 數量持續上升。數十位由 agent 輔助的開發者，每週合併數百個 pull request。

把所有因素算進來，我估計最終大約有 43 萬行生產級 TypeScript 經過了這場移植。

同樣這些因素也讓過程中很難看出進度：直到接近尾聲之前，生產級 TypeScript 的量看起來都相對持平，甚至還略為上升，因為移植的速度剛好跟上新增工作的速度。情況更混亂的是，在這段期間也有與移植無關的新 Rust 程式碼進來；移植初期，新進程式碼比較可能以 TypeScript 為主，到了後期則比較可能以 Rust 為主。

![圖 2](https://github.blog/wp-content/uploads/2026/09/runtime-line-history.svg)

*圖 2：TypeScript 降到零，生產級 Rust 升到約 83 萬行，Rust 單元測試升到約 46.9 萬行。*

在移植期間，runtime 吸收了約 30 萬行生產級 TypeScript、丟掉了約 43 萬行；同時約 120 萬行生產級 Rust 進來、約 36.5 萬行離開。換句話說，上圖中 TypeScript 曲線表面上的穩定，其實掩蓋了大量的 TypeScript 汰換（churn）。

---

## 3. 原地移植策略

那張圖也凸顯了這次移植做法的一個重要面向：原地（in place）進行。

這種規模的重寫主要有兩種做法：

1. **大爆炸（Big bang）。** 全新的 Rust runtime 被當成一個完整的替代品開發出來，等它就緒後一次性換上。這種大爆炸切換有兩種變體：
   - **停止世界。** 在重寫進行期間，所有人停下 `main` 分支上的其他工作，重寫直接在 `main` 裡做。
   - **平行開發。** 重寫在一個 feature branch 上進行，主分支的工作照常繼續，重寫端得不斷努力追上並合併主分支的變更。
2. **原地（In place）。** 這是逐元件的移植，runtime 一次一塊地被逐步重寫。這種原地做法也有兩種變體：
   - **原子替換。** 每一塊都從 TypeScript 原子性地翻轉成 Rust，由剩餘 TypeScript 與新 Rust 之間的互通層提供連續性。隨著時間推進，生產 runtime 中 TypeScript 愈來愈少、Rust 愈來愈多，直到有一天再也沒有 TypeScript，只剩 Rust。
   - **A/B。** 元件移植後不刪掉舊的，而是把 TypeScript 與 Rust 兩個版本都維護成可熱切換的選項，等信心累積到高原期再刪掉 TypeScript。

### 為什麼選「原地 × 原子替換」

- **沒有人需要停工。** 主分支持續活躍。每個沒有直接參與移植的開發者都能繼續照常推進，只有當他們手上長期進行中的 pull request 剛好碰到同時被移植的程式碼時才會受影響——這時他們需要 rebase，並讓自己的 agent 協助把手上的變更也一併移植過去。
- **runtime 的主分支永遠可出貨。** 每一個 pull request 都會把既有的 TypeScript 實作換成一個呼叫 Rust 的薄殼（shim），並在同一個原子性變更中刪掉舊程式碼。新程式碼會立刻在原位被實際運用到。
- **重寫是漸進而且可審查的。** 每個 pull request 只移植單一元件或切片，變更範圍較小，diff 也更容易審查——不論審查者是人、是 agent，或兩者兼有。
- **大多數移植都相當小而自足**，把來自並行 pull request 的偏移（drift）降到最低。有些情況下，當 TypeScript 元件太大時，可以先重構成更容易移植的元件。
- **所有既有的端到端（E2E）測試**，橫跨 CLI 與 SDK，在每一步都對新的 Rust 程式碼執行，給了我們信心與大量驗證。如果某個 pull request 讓必要測試失敗，它就進不來。

### 為什麼不選 A/B 雙版本

我們也避開了 2b 那種需要同時維護同一元件多個版本的變體。在過去幾個月裡，這個 repo 每週有數百個 pull request 被合併，程式碼庫持續且快速地演進。同一份程式碼存在兩種語言的兩個版本、使用兩套不同的相依函式庫，會增加大量複雜度。

這些元件有些也並非完美隔離；有些在邏輯上是獨立的、對外有簡單的 API 讓系統其他部分存取，但有些則有大量觸手般的牽連，要讓那張圖的每個元件都可熱切換簡直是一場惡夢。

最可能從謹慎的平行切換中受益的子系統，恰恰就是最難做平行的那些。舉例來說，session 編排（orchestration）不是那種你可以用一個實驗旗標的 `if`/`else` 去呼叫兩個不同版本的純函式。它擁有可變狀態、雙向驅動回呼，並貫穿幾乎所有其他子系統，所以「兩邊都跑然後比對」會變成要維護兩份彼此分歧的元件副本——而這個元件持有整段對話的狀態與服務——然後在數百個並行編輯之間祈禱它們保持同步。

讓一個元件難以移植的那種耦合，正是讓它幾乎不可能被影子化（shadow）的同一種耦合，硬做反而可能引入比避免掉的更多回歸。這種可切換做法的好處主要在於獲得信心，而信心我們可以用別的方式取得。

### 用漸進式推出換取驗證

驗證也透過漸進式推出（incremental rollout）來完成。若採大爆炸切換，我們會把一切留在一個長壽分支裡，把整個 runtime 移植完，然後一次切換。那意味著消費端會一次承受所有被移植的程式碼，包括所有在 repo 內測試時漏掉的回歸。

分批推出——這裡兩個元件、那裡一個元件——讓我們能在已部署的建置中、以真實消費端使用（最常見的是 Microsoft 與 GitHub 內部的第一方使用）取得最後一哩的驗證，同時把回歸風險壓到最低。

在大約十四週半的移植期間，`main` 出了 135 個版本，其中 100 個是 pre-release、35 個是穩定版，平均每天約 1.3 個版本。每天也大約開出 1.3 個移植 pull request，所以每個版本都只帶著一小群、可掌握的已移植元件（我們通常會——但並非總是成功——試著先用 pre-release 出貨移植成果）。

在 npm 過去七天的抽樣中，pre-release 版本只佔下載量的 10.5%，顯示初期曝險相對有限；同時我們監看各種回饋管道尋找出問題的訊號，並很快在下一個 pre-release 中修好。回報的 issue 也更容易與已知的近期變更關聯起來，更容易找到根本原因並迅速修復。

就這層意義而言，把移植拉長時間逐步進行反而是個優點而不是阻礙（也就是說，快不一定總是好）。到 8 月 21 日，runtime 已是 100% 生產級 Rust：832,378 行生產級 Rust、468,689 行 Rust 單元測試，另外還有 174,675 行 TypeScript 的 E2E 測試。獨立的 GitHub Copilot SDK repository 又另外增加了約 13 萬行 E2E 測試程式碼，橫跨 Node.js、Python、Go、C#、Rust 與 Java。

![圖 3](https://github.blog/wp-content/uploads/2026/09/port-fix-release-timeline-1.svg)

*圖 3：從 5 月 12 日到 8 月 21 日的時間軸，顯示 128 個依變更行數調整大小的移植 pull request 合併，以及 135 次 CLI 公開發佈；最大的移植變更集中在移植接近完成之處。*

---

## 4. 起步

在全力投入之前，我們先建立信心、驗證做法。

我們從兩個 pull request 開始，建立 Rust workspace、工具鏈、lint 規則、CI、建置管線與程式撰寫指示（coding instructions），接著引入 runtime crate 以及程式碼生成與互通模式，同時移植一批純邏輯的基礎元件——這些元件是特意挑選的，因為它們沒有 I/O、沒有共享狀態，而且已經有很強的測試。

等這些都落地之後，第一個正式的移植 pull request 才把三個無副作用的輔助函式完整走過整個流程。這些就像出貨版的試飛，把關於 repository 結構、FFI、打包、測試與審查的種種假設，轉化成後續規模大得多的移植可以直接沿用的慣例。基本上，我們把整套機制端到端測試了一遍。

計畫接下來的順序是由葉節點往內推進：先是純輔助函式、內容排除（content exclusion）、shell 工具與 session 檔案系統操作，藉此建立翻譯與測試的模式。接著是有狀態的子系統，然後工具（tools）、hooks、模型 client 與 MCP 疊在這些之上。session 編排——runtime 中耦合最深、最不適合平行處理的部分——則留到接近最後。

![圖 4](https://github.blog/wp-content/uploads/2026/09/pr-timeline.svg)

*圖 4：128 個落地的移植 pull request 從 5 月到 8 月的時間軸，從小型基礎元件逐步推進到較大的編排與 session 工作。*

| 期間 | Pull request 數 | 變更行數中位數 |
| --- | --- | --- |
| 5 月 1–15 日 | 8 | 3,250 |
| 5 月 16–31 日 | 2 | 9,421 |
| 6 月 1–15 日 | 40 | 5,073 |
| 6 月 16–30 日 | 31 | 8,253 |
| 7 月 1–15 日 | 10 | 9,514 |
| 7 月 16–31 日 | 14 | 28,159 |
| 8 月 1–15 日 | 19 | 13,861 |
| 8 月 16–30 日 | 4 | 99,445 |

早期的移植——小型葉節點元件——推進得很快。但較大的子系統並不是一步到位；舉例來說，MCP 支援經過了七個專門的 pull request，工具則走了一個六部曲系列，之後還需要額外工作才能把編排搬過去、退役剩下的 TypeScript。Hooks、認證、遙測、外掛、設定與持久化也都走過類似的路徑。

實際上，有用的移植單位並不總是「一個元件」。它往往是一波掃過相關行為區域的浪潮：先搬純邏輯，再搬狀態所有權，再搬編排，然後移除退路（fallback），最後在暫時性互通層消失之後簡化 Rust 程式碼。

---

## 5. 互通（Interop）

這次移植涉及互通的主要有兩層：

1. **暫時性的內部互通。** 每當一個函式被移植成 Rust，那個函式就必須能被原本呼叫該 TypeScript 函式的 TypeScript 程式碼呼叫。同樣地，我們也需要讓 Rust 函式能呼叫 TypeScript 回呼。

   這種互通需求是實作細節，而且極度流動。隨著 Rust 內部介面擴大，所需的 TypeScript 薄殼數量也跟著增加，因為它們與「需要從 TypeScript 呼叫的 Rust 方法」是一對一的。當那些呼叫端也被移植成 Rust 之後，既有那一層薄殼就被刪掉，換上新的一層。最終我們抵達 runtime 函式庫的公開進入點，薄殼也就蒸發了。

2. **SDK 介面。** 所有 SDK 函式庫都必須能夠坐在 runtime 之上並暴露它的功能。

   在移植前的世界裡，這是透過把 runtime 暴露在一個雙向 JSON-RPC 層之上來完成：SDK 把函式呼叫請求當成 JSON-RPC 方法呼叫的 payload 送出，runtime 解析請求並呼叫對應的 API，然後把結果經由同一個傳輸通道送回去讓 SDK 解析並回傳。反方向也存在；runtime 需要能回呼 SDK client，例如 hook 通知與權限請求，這些在 SDK client 中會以各語言慣用的語言特性呈現成回呼（例如 C# 的 delegate）。

### 暫時性內部互通：napi-rs

第 (1) 點我們是透過 napi-rs 專案的 `napi` Rust crate 達成的，該專案的存在目的就是用 Rust 建置 Node 原生擴充（native addon）。

你在函式上標注 `#[napi]`，napi-rs 的巨集就會產生 N-API 註冊膠水程式碼，讓該函式可以從 JavaScript 呼叫，並在產生的 `index.d.ts` 中加上對應的 TypeScript 宣告。同步的 Rust 函式會變成普通的 JavaScript 函式，`async fn` 會變成回傳 promise 的 JavaScript 函式，而標注 `#[napi(object)]` 的 struct 則會變成另一邊的普通物件。

流量也必須雙向移動。許多已移植的元件暫時仰賴某些還沒被移植的東西，所以 Rust 需要回呼 TypeScript——例如某個 Rust 中的工具實作要向仍是 TypeScript 的模型層請求推論、或觸發一個 hook、或為它想執行的指令請求一個權限決定。

napi-rs 用「threadsafe function」處理這件事，讓跑在 Tokio worker 執行緒上的 Rust 程式碼可以在 Node 的主執行緒上呼叫 JavaScript 回呼。Node 安裝回呼一次，Rust 持有它，並在需要往回走時呼叫。這其中的每一個都是「先天暫時」的：回呼之所以存在，只是因為另一端的東西還是 TypeScript，等那個東西被移植後它就會被刪掉。

![圖 5](https://github.blog/wp-content/uploads/2026/09/interop-surface.svg)

*圖 5：暫時性的 Rust N-API 匯出及其 TypeScript 呼叫點在漸進式移植期間增長，隨後在呼叫端遷往 Rust、暫時性互通介面消失後下降。*

這道暫時性接縫在 8 月 3 日達到高峰：2,019 個內部 N-API 匯出與 3,356 個 TypeScript 呼叫點。完成時，runtime 完全是 Rust，因此沒有內部互通：剩下 0 個暫時性內部 N-API 匯出、0 個 TypeScript 呼叫點。（我前面提過 CLI 仍有一些對 runtime 的內部存取我們正在努力移除；那些匯出不計入此處。）

### 永久介面：六語言 SDK

第二層互通，也就是 SDK 介面，是兩者中永久的那一個。Copilot SDK 為六種語言出貨：TypeScript、Python、Go、C#、Java 與 Rust。它們全都講同一套雙向 JSON-RPC 合約，而且原本全都用同一種方式抵達它：以 headless 模式把 Copilot CLI 生成為子行程，然後透過 pipe 或 socket 跟它對話。

移植期間這仍是預設做法。這也意味著任何語言的 SDK 消費端都得夾帶或找到一份完整的 Node 實作，每個事件與每則訊息都付出一次行程跳躍的代價，並且要監督兩個行程而不是一個。

把 runtime 移植到 Rust，正是讓另一種選項變得可行的關鍵。出貨的 `runtime.node` 是一個普通的平台共享函式庫（`.node` 副檔名是 Node.js 原生擴充的慣例，底下其實就是 `.dll`、`.so` 或 `.dylib`），而它現在對同一個引擎提供了兩道前門：

- **napi 門。** Node 行程把它當原生擴充載入；那是 CLI 目前走的路徑（未來的意圖是讓它完全改走 SDK 路徑）。
- **C ABI 門。** 任何語言都能把它載入自己的行程並透過 FFI 呼叫。

同一個行程內 runtime，透過各語言原生的互通機制來選用：

| SDK | 原生橋接 | 行程內 client 選用方式 |
| --- | --- | --- |
| C# | P/Invoke | `new CopilotClient(new CopilotClientOptions { Connection = RuntimeConnection.ForInProcess() })` |
| Go | purego | `copilot.NewClient(&copilot.ClientOptions{Connection: copilot.InProcessConnection{}})` |
| Java | JNA | `new CopilotClient(new CopilotClientOptions().setConnection(RuntimeConnection.forInProcess()))` |
| Python | cffi | `CopilotClient(connection=RuntimeConnection.for_inprocess())` |
| Rust | libloading | `Client::start(ClientOptions::new().with_transport(Transport::InProcess)).await?` |
| TypeScript | koffi | `new CopilotClient({ connection: RuntimeConnection.forInProcess() })` |

Rust 重寫，與「行程內 vs. 行程外托管」的選擇，是兩個獨立的維度。完成後的 Rust runtime 兩者都支援：它可以跑在 SDK 消費端的行程內，也可以躲在既有的 JSON-RPC 伺服器邊界之後。

那些行程內進入點目前是選擇性加入（opt-in）的，因為我們還在累積「與消費端應用程式共用一個行程、也就是共用一個故障邊界」的信心。傳輸層之上的一切仍然是同一套 SDK API：session、事件、工具、權限與回呼，都不在乎它們的 JSON-RPC 位元組是穿過一條 pipe 還是一次函式呼叫。

### 為什麼 C ABI 只有 19 個函式

第二道門有趣的地方在於它的大小。它只有 19 個匯出函式：四個負責伺服器生命週期、四個負責 session 註冊與設定、八個負責連線、三個負責嵌入式主機。

在這些函式背後，共享合約目前包含 364 條分派路由（dispatch route）：340 條可由 SDK 消費端呼叫，24 條則反向作為 runtime 對 SDK 的回呼。napi 那道門則大得多，需要為那 364 條分派路由中的每一條都提供函式。

C ABI 這道門是以分派為基礎的：API 方法根本不會有自己的匯出，而是以 JSON-RPC 位元組的形式寫入一條連線，結果、事件與伺服器對 client 的請求則透過主機提供的回呼傳回。新增、修改或移除一個 API 方法，會動到引擎的分派表，卻永遠不會動到 ABI。SDK 只要綁定那 19 個進入點一次，就能透過它們動態觸及整個、而且還在成長的 API 介面。

### 為什麼行程內還留著 JSON-RPC

這自然引出一個明顯的問題：既然呼叫已經不再跨越行程邊界，為什麼還留著 JSON-RPC？

答案是：這讓行程內托管成為「直接插上去就能用」，而不是一次重寫。每個 SDK 本來就已經有一個可運作的 JSON-RPC client，具備分幀（framing）、請求與回應的對應關聯，以及處理伺服器對 client 方向的處理器。把 FFI 掛成那個 client 底下的另一種傳輸方式，只是把位元組路徑從 pipe 或 socket 換成一次函式呼叫，而它之上的一切原封不動。

六個 SDK 都以附加的、選擇性加入的傳輸方式取得了行程內托管，既有的部分完全不變。如果我們當初改成為每個 API 方法定義一個帶型別的 C 函式，那麼每個 SDK 都得再做一層綁定，每新增一個 API 方法就得再寫六份綁定，而 ABI 也會變成一個我們必須做版本管理的二進位相容性介面。

我們也仍然需要 JSON-RPC 來服務那些真的位於遠端的 runtime，不論是跨子行程邊界還是透過 TCP。在行程內保留同一套協定，意味著我們只要維護一套雙向 API 與分派系統，而不是「遠端連線用 JSON-RPC、本地連線再加一套每方法的 FFI 介面」。

這是個真實的取捨，而不是想都不用想的決定。我們省掉了行程跳躍，但每次呼叫仍要付 JSON-RPC 的額外開銷。對以推論為主的工作負載而言，那點序列化跟模型往返比起來通常只是小菜一碟。在高吞吐量的本地工作負載中它仍然可量測得到，但還不到足以讓我們為此在六個 SDK 綁定中複製數百個方法的程度。

而且這是個日後若真有效能需求就能輕易修正的決定。payload 編碼是兩端之間的私有細節，把 JSON 換成 MessagePack 之類更緊湊的格式，不會改變任何一個已宣告的匯出。日後也可以為熱路徑加上帶型別的每方法匯出，呼叫同一個引擎與同一批處理器，而不必取代那條位元組通道——它仍會是串流、伺服器對 client 請求，以及那條「為它做專屬匯出毫無好處」的長尾稀用方法的底層基礎。

---

## 6. session 資料透露了什麼

這篇文章中幾乎每個數字都來自兩個來源之一。

第一個來源是私有 repository `github/copilot-agent-runtime` 的 GitHub 歷史：pull request 與它們的 diff、審查留言、CI 執行等等。

第二個來源是 agent session 日誌。runtime（因此也包括 CLI、app 等等）會為它執行的每一個 session 寫下結構化的事件日誌：每行一個 JSON 物件，隨著 session 進行而附加。這些日誌可能包含提示詞、指令、指令輸出、檔案路徑，以及工具可能揭露的機密，因此必須當成敏感資料處理。日誌存放在執行該 session 的機器本機；遠端 session 功能在啟用時也可以上傳它，但受產品設定與組織政策規範。

以下是所有構成移植的 pull request 的資料彙總：

| 指標 | 數量 |
| --- | --- |
| 事件 | 12,760,995 |
| 使用者訊息 | 31,247 |
| 助理訊息 | 1,385,214 |
| Hook 起訖事件 | 6,438,562 |
| 工具啟動 | 1,857,409 |
| 編譯指令 | 23,096 |
| 測試指令 | 19,485 |
| Rebase 指令 | 2,496 |
| Commit 指令 | 7,410 |
| Push 指令 | 5,554 |
| 完成的壓縮（compaction） | 5,116 |

那 31,247 則 user 角色訊息，並不是我本人親手打的 31,247 則提示詞；它們包含技能指示、自動化的合併勾選、跨 session 訊息，以及子代理的流量，遠多於我親手打或口述的那約 2,600 則——大約十二分之一。

同樣地，1,385,214 則助理訊息也包含子代理與偏工具導向的訊息，而不只是在對話介面中顯示給我看的文字。整個語料庫包含 68 種不同的事件類型與 67 種不同的工具名稱；1,130,921 次工具呼叫（61%）來自子代理，而非主 session 執行緒。

這個數字並沒有說明我為什麼要介入約 2,600 次。為此，我請 Copilot 為 session 日誌語料庫中每一則由人撰寫的訊息指派一個主要意圖。

![圖 6](https://github.blog/wp-content/uploads/2026/09/user-message-intents.svg)

*圖 6：在 2,639 則人撰寫的訊息中，31.0% 聚焦於審查、測試與 CI；17.4% 質疑技術或設計決策；15.0% 推動把事情做完整。*

前三個類別就佔了我互動的 63%。只有約 40 則是可辨識的、由我發起的 session 啟動；因為那件事大多是由我先建立一個聊天 session 去探索下一個地平線，然後請那個聊天 session 為每個想做的切片建立實際的移植 session。

我的角色與其說是「指派任務然後等待」，不如說是「操作控制迴路」：檢視結果、質疑技術決策、執行品質關卡，並在 agent 把某個中間停頓點當成終點線時推它一把。即使 agent 在做「實際工作」，人的判斷仍然涉入極深。只是我的涉入層級往上移了——我不再負責寫語法，而是負責界定問題、定義邊界、選擇策略、裁決例外，以及確保整體朝好的方向前進。

### 6.1　一切都在於快取

LLM 供應商通常對輸入 token（你送給它們的）與輸出 token（它們送回來的）收取不同費率。計費之所以常以 token 為單位，是因為 token 是推論所需計算量的有用近似：對每一個輸入 token，模型都必須讀取它、把它納入內部表示，並用它參與決定下一個 token 的計算。

不過，供應商往往支援快取那些計算的結果，如此一來，若某段提示詞前綴已經處理過一次，供應商就能重用快取中的中間計算，而不必從頭重算。這降低了處理那些 token 的成本，而這份節省可以回饋給消費端。因此，輸入 token 常常標示多種費率，其中包括從快取讀取的輸入 token 費率。

折扣非常驚人！供應商經常以 90% 的折扣計價快取命中，例如某家供應商可能對 100 萬個輸入 token 收 2.00 美元，但對 100 萬個快取輸入讀取 token 只收 0.20 美元。換句話說，你會非常、非常想維持良好的提示詞快取，好讓你的帳單少一個數量級。

移植過程的資料顯示我們在這方面做得不錯。提示詞快取命中率是 96.22%：快取讀取除以所有輸入側的 token 量（快取讀取加快取寫入加全新輸入）。快取寫入佔 3.07%，全新輸入佔 0.71%。

這不是巧合。GitHub Copilot 特意塑造 agent loop 以保留一段長而穩定的前綴（先系統提示詞，再工具定義，再累積的對話），使得每一輪都只是在模型已處理過的脈絡後面追加內容。脈絡中昂貴的部分只付費一次，之後每次呼叫都以低一個數量級的金錢成本重讀。

這也正是長時間自主 session 的經濟性得以成立的原因。一個三百小時的移植，如果在數萬次呼叫中每次都從頭重讀它不斷成長的整個脈絡，成本將會是我們所見數字的另一個量級。Agent harness 的開發者花費大量精力設法不打破提示詞快取，而模型供應商也例行性地推出新功能來幫助他們做到這點。

![圖 7](https://github.blog/wp-content/uploads/2026/09/prompt-cache-composition.svg)

*圖 7：移植 session 的提示詞快取組成：96.22% 快取讀取、3.07% 快取寫入、0.71% 全新輸入。*

壓縮（compaction）則說了一個互補的故事。在所有移植 session 中，GitHub Copilot 自動壓縮脈絡 5,116 次（也就是 session 填滿脈絡視窗、為了繼續下去而自我摘要的那些時刻）。單一個 session 基礎設施的移植 pull request，在它橫跨多日的生命週期中就壓縮了 647 次，而某個小型移植則一次也沒壓縮過。

持續數百小時的自主工作之所以可能，正是因為 agent 能一次又一次回收它的工作記憶而不失去頭緒。那數千次摘要中的每一次，都可能是一個讓移植悄悄脫軌的有損交接點——而大多數時候並沒有。

Copilot 的子代理對於降低壓縮次數也貢獻極大。每個子代理擁有自己的脈絡，因此父 session 實際上可以拋出一個問題，讓子代理跑去花上相當可觀的脈絡算出答案，然後只把答案回報給父代理。父代理的脈絡不必被那些中間資訊影響。

上面那句「大多數時候並沒有」在 session 日誌中是看得見的。我請 Copilot 把每一次成功的壓縮與其前後的工作配對起來，條件是兩側至少各有 20 次工具呼叫，這產生了約 4,000 個可比較的視窗。

Agent 在壓縮前 20 次工具呼叫中所做的事，其組成比例和壓縮後看起來規模相近（探索：壓縮前 46.5%、壓縮後 48.1%；變更：前 8.4%、後 6.0%；驗證：前 4.7%、後 4.0%；失敗：前 1.0%、後 1.5%）。如果壓縮經常把思路弄丟，我們會預期壓縮後那一側明顯偏向重新定位——讀取量飆升、編輯量崩塌，因為 agent 要重新搞清楚自己在哪、該做什麼。實際上，只有朝那個方向的輕微偏移。

### 6.2　沒錯，靜態分析真的有幫助

有一個流行的迷因說：Rust 特別適合作為 AI 生成程式碼的目標語言，因為 Rust 嚴格的編譯器會抓到模型犯的錯。session 日誌讓我們得以檢驗這個理論——至少對於長得像這次移植的任務而言。

直接的驗證指令結果，捕捉到 8,678 次 `rustc` 錯誤碼出現。最大的四個診斷家族涵蓋了 84%：

- **37%**：名稱與匯入解析，以 `E0425`（「在此範圍中找不到該值」）為大宗
- **22%**：缺少方法或欄位
- **14%**：型別不符
- **11%**：未滿足的 trait 界限（trait bound）

這每一項都是普通的接線工作：名稱輸出得稍微不對、簽章對不上、欄位被改名、抽象沒有被實作。這些正是大量翻譯很容易不小心產生的錯誤，也正是編譯器能非常快抓到的那種。

但請注意這份清單上缺了什麼：任何真正 Rust 專屬的東西。這四個類別全都是基本款的靜態型別檢查，C#、Java 或 Go 的編譯器一樣能全部抓到，其中好幾項的診斷訊息還更友善，而且全都快得多。

如果這就是「把 agent 指向 Rust」的論據，那它其實是「把 agent 指向任何靜態型別語言」的論據。強型別編譯器、或具備優秀靜態分析與 lint 的語言，確實非常適合這類工作，agent 會把它當成快速回饋迴路來用。在 4,478 次直接執行 `cargo check` 且較嚴格的結果比對器有捕捉到結果的執行中，87.1% 是乾淨通過的——這正是小幅度編輯、不斷重新編譯會得到的結果。

相對地，所有權（ownership）、借用（borrowing）與生命週期（lifetime）錯誤加起來只佔有錯誤碼診斷的 1.7%。借用檢查器（borrow checker）——那個主宰所有「Rust 很難」討論的東西——只是個安靜的背景存在。編譯器幾乎把它所有的報錯能量都花在無聊的機械性錯誤上。

### 6.3　Agent 喜歡讀

我們也可以檢視 session 事件語料庫中的工具呼叫資料，並從中萃取出一些關於 agent 如何運用時間的有趣觀察。

| 工具 | 呼叫次數 | 中位數 | 量測時數 |
| --- | --- | --- | --- |
| powershell | 630,423 | 3 秒 | 2,833.9 |
| view | 590,988 | 0 秒 | 621.7 |
| rg | 281,783 | 1 秒 | 408.4 |
| grep | 126,483 | 1 秒 | 115.3 |
| apply_patch | 53,715 | 0 秒 | 17.0 |
| edit | 40,591 | 1 秒 | 24.1 |
| read_powershell | 36,728 | 90 秒 | 1,203.9 |
| task | 13,080 | 274 秒 | 2,329.0 |

我的第一個心得是：agent 花在蒐集證據上的時間，遠多於花在改程式碼上的時間。就表中顯示的檔案讀取與搜尋工具對比編輯工具而言，它們做的探索是變更的 10 倍。讀檔案、搜 repository、跑診斷指令佔了絕大多數；編輯相對來說只是很小一塊。

「AI 狂噴程式碼」這個流行印象幾乎是反過來的；在這個規模下，工作看起來更像是迭代式的調查：檢視當前狀態、形成假設、做出針對性的變更，然後洗洗再來一次。

委派又放大了這個模式。子代理主要被用來把探索工作分散到彼此獨立的問題上，而主代理較常自己掌握編輯並整合答案。對這類專案而言，這是個有用的分工：許多脈絡可以平行調查，但把變更保持在靠近協調者的位置，能減少彼此衝突的修改，並維持一致的實作策略。

shell 流量也顯示出自主軟體工作中有多大比例其實是狀態管理。唯讀的 Git 檢視是最常見的指令模式，因為 agent 不斷在問的其實就是「我現在在哪？」。它們在找什麼變了、rebase 做了什麼、另一個 session 落地了什麼，以及某個分支跟快速演進的 `main` 偏離了多遠。那種定位工作，讓許多長時間運行的工作能在同一個移動中的程式碼庫上運作而不至於盲目互相覆蓋。

深入 shell 工具流量內部，最常見的指令家族讓「定位」與「驗證」之間的平衡更加清楚：

| 指令家族 | 呼叫次數 | 中位數 | 量測時數 |
| --- | --- | --- | --- |
| git inspect | 300,530 | 2 秒 | 608.1 |
| git other | 89,865 | 3 秒 | 243.1 |
| search | 85,482 | 2 秒 | 147.6 |
| pnpm test | 13,852 | 22 秒 | 219.1 |
| pnpm lint | 9,757 | 29 秒 | 177.0 |
| cargo test | 8,437 | 120 秒 | 364.2 |
| git commit | 7,410 | 11 秒 | 39.7 |
| cargo fmt | 5,223 | 18 秒 | 77.2 |
| cargo check | 4,492 | 120 秒 | 176.9 |
| pnpm build | 3,630 | 180 秒 | 215.6 |
| cargo clippy | 2,115 | 135 秒 | 107.4 |
| git rebase | 2,496 | 7 秒 | 9.9 |
| cargo build | 566 | 104 秒 | 20.3 |

### 6.4　模型選擇

GitHub Copilot 允許單一 session 在對話中途更換模型，也允許不同 session 跑不同模型，所以模型選擇變成了逐切片的決定。

日誌中出現了兩種不同的模型決策。在主執行緒上——也就是驅動每次移植的那一條——是我們選擇模型與推理強度（reasoning effort）。而在 session 內部，當 agent 啟動子代理或子 session 去探索、或去啃一個範圍明確的任務時，則是由編排的模型選擇那些模型。

![圖 8](https://github.blog/wp-content/uploads/2026/09/weekly-model-mix.svg)

*圖 8：主要移植 session 每週的模型組合，集中在少數幾個模型上，並在移植過程中有所移轉。*

對子代理而言，模型組合看起來有點不同——這時是由 agent 而非人在最佳化，優化的目標是吞吐量與成本，而不是最困難的判斷。它最常生出的子代理跑在 Claude Opus 4.8、GPT-5.6 Sol、Claude Haiku 4.5 與 GPT-5.5 上，其次是 Gemini 3.1 Pro 與 Claude Opus 5。

不過，至少在這些移植進行的當下，有三個常用的 agent 定義把模型選擇寫死了（`explore` 與 `task` 綁 Claude Haiku，`research` 綁 Claude Sonnet），所以那些量體中有相當一部分是由「選了哪個子代理」決定的，而不是另外選了模型。

![圖 9](https://github.blog/wp-content/uploads/2026/09/weekly-subagent-model-mix.svg)

*圖 9：移植期間生成的子代理每週模型組合，量體集中在 Claude Opus 4.8、GPT-5.6 Sol、Claude Haiku 4.5 與 GPT-5.5。*

---

## 7. 與 agent 艦隊一起工作

GitHub Copilot app 支援視覺化顯示進行中的 pull request session、各自的狀態，並讓你輕鬆在它們之間切換，這讓它非常適合管理移植過程中大量並行的工作。但真正讓它發光的能力之一，是 session 之間可以互動。

一個 session 可以建立其他 session，也可以在其他 session 執行時對它們傳訊息。每個 session，不論是父還是子，都擁有自己的 worktree、自己的分支與自己的 agent loop；它是與生成它的 session 分開的東西，而不是跑在它裡面。這跟子代理不同——子代理跑在父代理自己的工作區裡，並把答案交回父代理的脈絡中。兩者都是有用的構造，各有適合的場景。

### `session.ts`：3 萬行的骨幹

舉個 session 如何建立其他 session 的例子：最難的移植之一是 `session.ts` 這個檔案。

這個檔案有機生長到約 3 萬行 TypeScript。它是一個 session 的骨幹，實際上橫向貫穿整個 runtime，觸碰幾乎每個元件、也被幾乎每個元件觸碰，坐在狀態、事件、工具、模型、hooks、持久化與進入點存取的正中央。因此我把它留到移植流程的接近尾聲，從堆疊底部往上、跨所有垂直面推進，直到它們全都撞上 `session.ts` 這個死胡同。

接手它的那個移植 session 並不是一頭栽進去就開始寫 Rust。它前 56 分鐘都在讀，在建立任何東西之前做了 122 次工具呼叫，為的是建構出這個檔案究竟擁有什麼、接縫在哪裡的全貌。之後它才開始委派，在邏輯上把檔案切開，把切片分派給子 session。在整整 25 小時的執行過程中，它自己做了 222 次 shell 呼叫、205 次檔案檢視、197 次 ripgrep 搜尋——這還不包含它的子 session 所做的一切。

![圖 10](https://github.blog/wp-content/uploads/2026/09/session-fanout.png)

*圖 10：15 個子 session 巢狀於生成它們的 session 之下。*

那是 15 個子 session，每一個都是一條獨立分支、有自己的 worktree 與獨立的 agent，全都由最上面的父 session 隱式建立。父 session 在大約三小時內分七波建立它們：第一波五個，約二十分鐘後第二波兩個，再二十分鐘後又兩個，接著在之後兩小時裡零星地一個、兩個地建。

模型選擇是逐切片決定的：15 個中有 10 個跑 GPT-5.6 Sol，5 個跑 Claude Opus 4.8。全部 15 個都以 GitHub Copilot 的 autopilot 模式啟動，讓 session 能追求一個目標而不必每一步都停下來等核准。啟動提示詞的中位數長度約 1,100 字元，長到足以承載所有權邊界與各項限制，但又短到讓子 session 必須自己想出做法。我對父代理下提示詞，然後是父代理——而不是人——為每個子 session 寫下那些啟動提示詞。

在那 15 個子 session 之外，同一個父 session 還動用了五個子代理：三個 `explore` 代理與第一波同時發動，一個 `code-review`，一個 `rubber-duck`。這些子代理探索問題，把父代理在決定下一步之前所需的答案餵回它的脈絡。子代理讓父代理能取得經過深思熟慮的答案，卻不必用自己的脈絡視窗去推導它們。

相對地，子 session 負責的是實際的移植工作——那些會產生 diff、且需要與其他平行移植者隔離的工作。這些子 session 的工作觸及了 repo 中 140 個不同檔案，其中 120 個只被恰好一個 session 觸碰。20 個有競爭的檔案全都是樞紐，例如 `session.ts` 本身。

但每個 session 都在自己的 worktree 中工作，因此能不受手足干擾地推進。當然，父代理為此付出了協調成本。它花了不少力氣與子 session 溝通，扮演資訊中介，輪詢它們的狀態 60 次、送出 89 則協調訊息。當子 session 各自宣告完成時，父代理把它們的 commit cherry-pick 進自己的分支並解決衝突。而這些合併也不怎麼乾淨，父代理花了相當多時間去調和那些編輯。

我們可以在時間軸上看到這一切，圖中顯示了父 session 與它大部分的子 session。

![圖 11](https://github.blog/wp-content/uploads/2026/09/session-ts-fanout.svg)

*圖 11：`session.ts` 移植的時間軸，顯示一個 25 小時的父 session 使用五個子代理，並分七波生成 15 個子 session。*

注意圖中那些很大的空白。我當時一邊出差一邊做這個移植，好幾次不得不闔上筆電。（後來我改了工作流程，改成使用可以遠端連入的雲端虛擬機。）

### 把聊天 session 變成建置互斥鎖

這些平行的子 session 對那台筆電造成了顯著衝擊。有一陣子並行移植進行得順風順水。然後，同一台機器上的 15 個並行 agent 每個都試著建置與測試，我可憐的筆電直接卡死。

我對父代理下提示詞，請它轉達給所有子 session：它們全都必須停止建置與測試。父代理把這條限制向外轉達，它們也謝天謝地地砍掉了自己的建置，改以最低的 CPU 活動繼續工作。之後我更新了自己的常設指示：子代理與子 session 在移植期間應避免大型建置與測試執行，把那些延後交給父代理統一處理。

後來我又更進一步，把一個原本普通的聊天 session 變成八個獨立移植 session 的建置排程器。提示詞簡單到令人尷尬：對每個開啟中的 session 發送一條政策，要求盡可能避免耗用 CPU 的建置與測試，必須在需要建置時向這個 session 請求許可，並由這個 session 擔任關卡，一次只發放一個 session 的建置權。

基本上，我把一個聊天 session 變成了一個代理式互斥鎖（agentic mutex）。這個關卡維護明確的持有者與佇列，透過那些 session 原本就用來協調程式碼的同一套跨 session 訊息機制，一次發放一份租約。被拒絕租約的 session 往往會在等待期間去做別的事，例如從自己的待辦清單挑事情先做掉。

![圖 12](https://github.blog/wp-content/uploads/2026/09/build-gate.png)

*圖 12：一個聊天 session 擔任八個移植 session 的建置資源關卡。*

### 兩個 session 併吞事件

這次 `session.ts` 移植，也牽涉到我在整個 runtime 移植期間見過最酷、最悲傷、也絕對最出乎意料的互動之一。

如我所述，我們主要是由下而上進行移植，所以實際上坐在所有其他元件之上的 `session.ts` 是最後被移植的元件之一。唯一持續位於 `session.ts` 之上的，就是所有進入 runtime 的入口，也就是 SDK 對外暴露、並出現在前面談過的分派表裡的那些公開函式。這些有好幾百個。

而我雖然知道其中許多會立刻呼叫進 `session.ts`，但我想搶先推進移植，所以在啟動 `session.ts` 的 session 之後，我又啟動了一個 session 去移植所有的進入點。我告訴它在 `session.ts` 的邊界停下來。我心想大概會有一點白工、以及 rebase 時需要一些力氣或 token，但整體來說能加速移植。然後我就去睡了。

然後……它們找到了彼此。

我給進入點 session 的啟動提示詞確實有告訴它，有一個 session 移植與六個元件移植正在同時進行，因為我希望它知道自己的邊界、知道該避免移植什麼，好盡量減少衝突。結果我的提示詞顯然產生了相反的效果。開工才過四分鐘多，它盤點完各條入口路徑、想必也形成了對重疊程度的看法之後，就叫用了 app 內建的 `orchestrate` 技能——那個技能的用途正是跨 session 協調工作。接著：

1. 進入點 session 列舉了所有活躍的 session，並對它認為有重疊的那些送出訊息。
2. `session.ts` session 回了一份 2,001 字元的清單，標題是「Concrete overlap on `stephentoub-port-session-to-rust`」。
3. 進入點 session 讀取了 `session.ts` session 的 worktree，去確認它剛剛被告知的內容（我猜是「信任但要查證」吧）。
4. 進入點 session 問 `session.ts` session 是否準備好整合它那份涉及 760 個檔案的 diff。
5. `session.ts` session 基本上叫它滾一邊去：「尚未準備好 commit／整合。」
6. 進入點 session 接著又問了同樣的問題三次，每次都從 `session.ts` session 得到同樣的答案。
7. 到這時，進入點 session 決定它才不管 `session.ts` session 怎麼想，直接伸手進對方的 worktree，抓走另一個 session 的所有變更，合併進自己的分支。
8. 然後兩個 session 都若無其事地繼續各走各的路。

從這次互動中我學到幾件事：

1. **明確表達意圖非常重要。**
   那份啟動提示詞點名了其他執行中的 session，目的是讓這個 session 知道該放過哪些東西。但我沒有把「放過」這一點講明白，結果非但沒有阻止 agent 做某件事，反而變成鼓勵它去做。我需要在意圖與指引上明確得多。

2. **你提供出去的任何東西，agent 都可能認定它適用。**
   `orchestrate` 技能隨 GitHub Copilot app 出貨，它自我描述為用於平行執行獨立工作流。我的提示詞裡完全沒提到它。是模型自己發現了處境，把它與那段描述比對，然後載入了它。你暴露出來的能力集合，就是你可能得到的行為集合——包括在你從沒想過的情境中。

3. **對等的同儕需要一個裁決者。**
   兩個 session 誰也無法強制誰。當 `session.ts` session 四度表示自己尚未準備好整合，那份拒絕毫無份量，於是願意單方面行動的那一方就理所當然地贏了。針對相鄰程式碼平行運作的 session，需要一個被指定的協調者，或者需要一個人類——而這兩個都沒有。

4. **「自主執行」需要為「會伸出自己分支之外的決定」保留例外。**
   我真正的意思是「別為了設計細節把我吵醒」。它聽到的（也不算不合理）是「併吞同儕也在授權範圍內」。同樣地，我應該在指引上更明確。

5. **這件事的根本原因是我。**
   我同時由上而下與由下而上切分這份工作，結果兩個方向在整個程式碼庫中耦合最深的那個檔案上迎面相撞。我太貪心於推進度了。以上一切都由此而來。

### 兩種 session 節奏

謝天謝地，整起互動是個有趣的異數。在整個 runtime 移植過程中，大多數葉節點元件的移植都是直截了當的單一 session 任務。較大的子系統則常牽涉多個子 session 與子代理。不過，這些部件在一次移植過程中如何參與，差異相當大。

模型編排（model orchestration）——實際與供應商對話的那一層——的移植，是其中一種模式的好例子。它的主 session 執行了 42 個實際小時，啟動了 126 個子代理。在最忙碌時，有 22 個同時在工作。不過大多數時間其實只有主代理，然後偶爾才會在一段時間內生出大量子代理。

![圖 13](https://github.blog/wp-content/uploads/2026/09/port-model-orchestration.svg)

*圖 13：42 小時的模型編排移植在前 12 小時產出了大部分程式碼，之後進入一個明顯的驗證與審查階段，動用 126 個子代理。*

那張圖裡有三件事很突出。第一，程式碼生成實際上全在前 12 小時內完成；之後那一整天的工作全是驗證。第二，底部那一列的顏色由左往右移轉，從以藍綠為主（讀取、建置）變成以藍橘為主（讀取、審查）；這在邏輯上說得通，但實際看到還是很妙。第三，那個例子——以及更廣泛地說這個模式——各階段之間切分得非常乾淨。

而擴充功能 runtime（extension-runtime）的移植則是反例。

![圖 14](https://github.blog/wp-content/uploads/2026/09/port-extension-runtime.svg)

*圖 14：88 小時的擴充功能 runtime 移植，在整個 session 的大部分時間裡把讀取、撰寫、建置與審查交錯進行，而不是切分成不同階段。*

它花了 88 小時而不是 42 小時，而且結構非常不同：

- **撰寫與審查大幅重疊。** 前一個例子的工作非常瀑布式（先產生程式碼、再審查），這裡則是審查遠在撰寫停止之前就開始，兩者在大部分時間裡並行且重疊。
- **底部那一列的顏色到處都是。** 模型編排那次是隨著從撰寫走向檢查而由綠轉橘，這一次則從頭到尾都是讀取、建置與審查的同一種混合。中間一半的讀取呼叫散佈在 49 小時的跨度內，撰寫散佈在 33 小時，審查散佈在 27 小時——而這全都在一個 88 小時的 session 裡。每個類別都散佈在幾乎整段執行期間。
- **閒置被推遲到最後。** 這支艦隊在前 56 小時幾乎不間斷地工作。
- **比例仍然吻合。** 這裡寫 Rust 佔工具呼叫的 2%，那裡是 1%；讀取這裡 44%、那裡 57%；審查這裡 23%、那裡 27%。兩個 session 對「該做的工作是什麼」看法一致，只是對「何時做」看法不同。

結尾那些空白的時段，也是一個在這個 agentic 編碼時代愈來愈常見的問題的視覺化呈現：等待核准。團隊中某人和／或某個 agent 審查程式碼並留下回饋，接著有一小段活動期，agent 處理回饋、把 CI 推回綠燈，然後又是等待，如此反覆，直到最終拿到那個能讓人分泌多巴胺的核准戳章。

這兩個例子各代表一種主流模式。大約四分之一的 session 比較像模型編排那個，四分之三則像擴充功能 runtime 那個。乾淨的階段式推進是例外；常態是 agent 從頭到尾同時規劃、撰寫與審查。

---

## 8. 規模化的程式碼審查

前面幾張圖中看到的那種對審查的重視，很大一部分來自我明確的提示。

我做了一個簡單的「提示詞即技能」，取名為 `rust-rebase-review`（此外我們在 repo 裡也合併了一個通用的 Rust 編碼技能）。由於新進變更速度很快、其中許多會造成衝突，我得非常頻繁地 rebase。我透過自訂指示鼓勵 harness 在流程中適當的時機叫用這個技能，偶爾也會手動叫用。

這段提示詞隨時間有些演變，但大致是這個版本的變形：

```text
Squash into a single commit, then rebase on the latest in origin/main,
resolving all conflicts, and force push. As part of rebasing, pay extra
special attention to anything that has changed, been added, been removed,
and ensure that logic is all ported over to the corresponding Rust code
correctly. Always do the rebasing yourself / in the main agent; do not
spawn a subagent for it.

Then enter a review/fix loop where you launch a subagent per opus 5,
gpt-5.6-sol, and grok 4.6.

- That subagent should do a line-by-line comparison of the old TypeScript
  and the new Rust, confirming behavioral equality.
- Look for anything introducing any kind of incompatibility; our goal is to
  move this code into Rust with as close as is possible to 100% the same
  semantics. If you hit anything questionable, ask me about it.
- We want to ensure we're writing as efficient and idiomatic Rust code as we
  can; look for opportunities to simplify, to use routines like from the
  memchr crate to optimize searches instead of open-coded loops, avoid
  unnecessary allocation, use traits for reuse and loose coupling, etc.
- Ensure that all defunct TypeScript code (e.g. code that has been fully
  ported, tests that are now no longer necessary because they're duplicative,
  unnecessary napi shims, etc.) has been deleted.
- Ensure that we've ported as much code as possible, e.g. if there's any
  TypeScript remaining in touched files and that TypeScript is more than just
  a shim, that's a red flag. If new TypeScript that's not just a super thin
  shim is being added, that's a red flag. Look for any callers of TypeScript
  shims to see whether those callers can instead be ported to Rust, pushing
  the boundary as far as reasonably possible. Our goal is to soon get to 100%
  Rust in the runtime layer.
- Validate that no E2E tests have been deleted or changed. Such changes are
  an indication of a porting bug.

If a review surfaces issues, validate them, and then if there are any to fix,
fix them, and iterate to do another full review. Continue iterating with
reviewing/fixing until all reviews come back clean. After every set of changes
in response to review feedback, commit and push so that CI validation runs
concurrently with subsequent reviews.

Don't bother running full test suites; that'll be handled in CI. Try to
minimize CPU-consuming efforts to the bare minimum, as we'll likely have many
operations happening concurrently.
```

上面這段提示詞的中譯大意：

- 先壓成單一 commit，再 rebase 到 `origin/main` 的最新狀態、解決所有衝突並強制推送。rebase 過程中要特別留意任何被修改、新增或移除的東西，確保那些邏輯都正確移植到對應的 Rust 程式碼。rebase 一律由你自己／主代理做，不要為此生出子代理。
- 接著進入一個審查／修正迴圈，分別以 opus 5、gpt-5.6-sol 與 grok 4.6 各啟動一個子代理。
- 子代理應逐行比對舊的 TypeScript 與新的 Rust，確認行為等價。
- 找出任何造成不相容的東西——目標是把這段程式碼搬進 Rust 並盡可能維持 100% 相同語意，遇到任何可疑之處就來問我。
- 盡可能寫出高效且慣用的 Rust：尋找可簡化之處、用 `memchr` crate 之類的常式取代自寫迴圈來最佳化搜尋、避免不必要的配置、用 trait 達成重用與鬆耦合。
- 確保所有作廢的 TypeScript 程式碼都已刪除。
- 確保已盡可能移植最多程式碼：被觸碰的檔案中若還有超出薄殼程度的 TypeScript 是警訊；若新增了不只是極薄薄殼的 TypeScript 也是警訊。要查看 TypeScript 薄殼的呼叫端能否也改成 Rust，把邊界推到合理的極限。
- 驗證沒有任何 E2E 測試被刪除或修改——這類改動是移植出 bug 的跡象。
- 若審查暴露出問題，先驗證再修，然後再迭代做一次完整審查，直到所有審查都乾淨為止。每一輪修改後都要 commit 並 push，讓 CI 驗證與後續審查並行。
- 別去跑完整測試套件，那交給 CI；盡量把耗 CPU 的動作壓到最低，因為很可能同時有許多操作在跑。

### 刪除 TypeScript 帶來的意外好處

在這種頻繁 rebase 的情況下，新進變更很容易被意外弄丟。但我們發現原地原子替換有個意想不到的好處：因為我們是在加入對應 Rust 的同時刪掉 TypeScript，這等於隱式地與 rebase 帶進來的、對那段 TypeScript 的變更製造了衝突——一邊在改它，另一邊在刪它。這保證了我們會注意到對已移植程式碼的修改，而不必為每一行新進程式碼去推敲它是不是碰到了先前已移植的部分。

我自己的審查當然只是代理式審查中的一部分。除了每次 commit 都會跑的 CCR 之外，團隊還有多個專門的程式碼審查機器人，各有各的做法與提示詞，在每次 commit 上執行並提供詳盡的回饋。這些全都會變成 pull request 上的留言，接著需要被處理。所幸，處理所有這些代理式回饋，也可以（大致上）用代理式的方式處理掉。

至於我自己的審查，我聚焦在架構、設計、慣例與做法。exhaustive 的新舊比對交給 agent；測試與靜態分析檢查那些可機械化強制的性質；人類審查者則集中在架構、API 合約、風險，以及其他層次所暴露出的可疑之處。

我選定目標架構、決定哪些行為重要、切分工作、裁決模糊的取捨、判斷證據、手動審查高風險區域、審查代理對回饋的回應，並做出最終的合併決定。Agent 改變的是一位工程師所能監督的程式碼量。它們並沒有消除「需要一位理解系統、能為方向、護欄與發佈背書的工程師」這件事。

---

## 9. 自動化內層迴圈

GitHub Copilot app 在這項工作中扮演核心角色。它管理大量並行的活躍 session，讓你能輕鬆在它們之間切換，並隨每個 session 帶著所有相關的配對脈絡（關聯的終端機視窗、瀏覽器視窗、畫布等等）。這裡最關鍵的功能是 agent merge：

![圖 15](https://github.blog/wp-content/uploads/2026/09/AgentMerge.png)

*圖 15：GitHub Copilot app 的 Agent Merge 面板，追蹤審查回饋、CI、衝突與合併就緒狀態。*

Agent merge 是內建在 app 裡的一個迴圈（CLI 也有，指令是 `/pr auto`）。依照計時器、或是回應外部刺激（例如 GitHub 傳來 CI 完成或審查留言的通知），app 會去看有什麼變了：

- 如果有人留下審查留言，它會叫用 agent 判斷要駁回該留言、還是接受並處理它（並回覆，註明回覆者是自動化）。
- 如果某個測試失敗，它會下載日誌、調查失敗原因並修掉 bug。
- 如果發生衝突，它會叫用 agent 去 merge 或 rebase。

實際上，它把我們人類開發者都在做的那個迴圈自動化了：把 pull request 推到「綠燈」、拿到簽核，最終合併。

每一個移植 pull request 都是由 agent merge 處理的。不過在大多數情況下，我們在真正的「合併」那一步之前停住。Agent 會修好所有 CI 失敗、處理並回覆所有留言、確保所有衝突都解決。在合併之前，我會抽查 agent 實際做了什麼，特別是它如何處理回饋。它對審查者的回應有沒有哪些我不同意？所套用修正的高層方向是否可取且合理？對這些移植而言，我通常會讓最後那一項保持未勾選。

### 那個抽查救回來的一次

最後那個勾選框不只一次發揮了作用。

在某次 merge 迴圈中，移植刪掉了一個暴露給 SDK 的函式。我們 repo 的 schema 相容性 CI 關卡精準地做了它該做的事：失敗了。Agent 的反應是替 pull request 套上 repo 的 `schema-break-ok` 自動化標籤——那是讓檢查通過的逃生門。

我在合併前審查這個 pull request 時，問了那個顯而易見的問題：「這個 schema break 是什麼？你在這個 pull request 上加了 `schema-break-ok` 標籤；為什麼它是 ok 的？」它並不 ok。那個方法在 `main` 上是存在的；移植只是把它弄丟了。我把這稱為不可接受的回歸，並要求 agent 用完整的 Rust 把它加回來。二十一秒後，豁免標籤被移除，那個方法以原生 Rust 實作恢復了。

我們對失敗同時做出在地與全局的回應：修好個案，但也修好系統，讓它們比較不會再發生。我們持續演進餵給編碼與審查代理的指示，以進一步降低同樣問題在未來移植 pull request 中重演的機率。我們把 session 日誌變成 eval（評測）。而在某些情況下，我們甚至把學到的教訓用來改善 runtime 本身，透過調整提示詞、工具描述，或 autopilot 的運作方式。

---

## 10. 一場遷移，兩種遷移

語言重寫幾乎從來就不只是語言重寫。runtime 依賴的每一個函式庫也都必須被替換掉，而且不像我們自己撰寫並擁有的 Rust 程式碼，那些替換品並不是我們能讓它們忠實還原的。有些只是同一個概念換了名字。有些得用好幾個 crate 才能覆蓋一個 npm 套件原本做的事。少數幾個則完全沒有可接受的現成答案，必須手工（由 agent）寫出來。

CLI 與 runtime 目前在同一個 repo 裡，共用一份 `package.json`。在整個移植過程中，我們移除了約 60 個 npm 相依套件，因為它們只被那些已被移植成 Rust 的 runtime 程式碼所使用。

這是移除數量的下限，因為有些套件雖然在 runtime 中被替換掉，CLI 卻仍然需要它們。舉例來說，`zod` 是一個 TypeScript 綱要宣告與驗證函式庫，CLI 與 runtime 原本都在用。有了 Rust 移植之後，runtime 現在改用 `serde`、`schemars` 與 `jsonschema` 的組合來滿足同樣的用途，但 `zod` 仍然為了 CLI 而留在 manifest 裡。

**一對一替換的例子：**

| npm 套件 | Rust crate | 備註 |
| --- | --- | --- |
| `js-tiktoken` | `tiktoken-rs` | 使用同樣的 `o200k_base` 編碼 |
| `ignore` | `ignore` | 同名，同樣的 gitignore 語意 |
| `minimatch` | `globset` | |
| `fast-myers-diff` | `similar` | |
| `dompurify` | `ammonia` | |
| `github/keytar` | `keyring` | |

在其他情況下，我們無法一對一地用 crate 取代套件。取而代之的是一個套件變成好幾個 crate，或好幾個收攏成較少的幾個。相依套件的工作量大部分花在這裡：

- 八個 `opentelemetry/*` 套件變成四個 crate，外加一個手寫的追蹤器狀態機與檔案匯出器。
- 三個網頁內容套件——`mozilla/readability`、`linkedom` 與 `turndown`——變成兩個 crate：`readability` 與 `htmd`。
- `sharp`、`image-size`、`file-type` 變成 `image` 與 `imagesize`。

諸如此類。另外還有五個案例，我們是用完全客製的實作整個取代掉一個 npm 套件。

---

## 11. 有多少 `unsafe`？

另一個大家會問的、關於 agent 寫的 Rust 的問題是：其中有多少悄悄放棄了安全性保證。Rust 的安全性是一個你可以用一個關鍵字關掉的性質，所以當 agent 遇到一個它滿足不了的借用時，就有一道現成的逃生門。

在整個 runtime crate 中，我們現在有 158 個 `unsafe` 區塊，分散在僅僅 36 個檔案中（與之並存的還有 26 個 `unsafe fn` 宣告、26 個 `unsafe extern` 區塊，以及 9 個 `unsafe impl` trait 實作）。重點是，這其中每一個都與「和外部元件互通」有關。

| `unsafe` 區塊存在的原因 | 區塊數 | 佔比 |
| --- | --- | --- |
| C ABI 邊界 | 51 | 32.3% |
| Windows API | 49 | 31.0% |
| POSIX / libc | 46 | 29.1% |
| SQLite C API | 7 | 4.4% |
| 動態函式庫載入 | 4 | 2.5% |
| 行程環境變數 | 1 | 0.6% |

![圖 16](https://github.blog/wp-content/uploads/2026/09/unsafe-boundaries.svg)

*圖 16：158 個 `unsafe` 區塊的環圈圖，全都位於外部邊界：C ABI、Windows API、POSIX 與 libc、SQLite、動態函式庫載入，以及行程環境變數的變更。*

C ABI 區塊是 SDK 主機進來的前門，所以它們會從一個 Rust 編譯器完全無法掌控的呼叫端收到原始指標與長度。Windows 與 POSIX 區塊是系統呼叫：登錄檔讀取、憑證交握、行程樹、`sysconf`。SQLite 是一個 C 函式庫。

動態函式庫載入是 `dlopen`，它在本質上就無法建構即安全——光是「你解析到的符號可能不是你以為的那個函式」這一點就足夠了。行程環境變數那個 `unsafe` 區塊之所以存在，是因為 Rust 2024 把多執行緒行程中對行程全域環境狀態的修改視為 unsafe。

這些 `unsafe` 區塊每一個都標示著一個 Rust 保證真正終止的地方：另一側的東西是一個 C 函式、一個系統呼叫、一個來自外部 runtime 的指標，或行程全域的主機狀態。

有用的性質在於：`unsafe` 讓我們所擁有的 Rust 程式碼中的這些地方都變得可稽核。TypeScript runtime 中的等價程式碼跨越了完全相同的邊界——透過 Node 的 C++ 內部與原生 npm 套件——而我們的原始碼中沒有任何東西標示出「受檢查的世界」在哪裡結束。當然，這並不是交付系統中每一道安全邊界的完整清單：相依套件、建置工具、C 函式庫、安全包裝層，以及規格寫錯的 FFI 合約，仍然可能包含或暴露不安全性。

`unsafe` 用在哪裡的細節很有意思，但我更感興趣的是它沒有被用在哪裡。它沒有被用在模型 client、沒有用在 MCP 層、沒有用在 agent 層，也沒有用在提示詞層。而且，這次移植已知的回歸中，沒有任何一個牽涉到 `unsafe` 區塊。

---

## 12. 那些回歸

移植程式碼很容易。讓它正確才難。而在一個像 Copilot agent runtime 這麼大、這麼複雜的程式碼庫中，回歸是意料中的事。

到 2026 年 9 月 14 日為止，我們追蹤到數十個已知的移植回歸，全部都已修復。大多是正確性 bug，另有一小群是效能回歸。而對照的是從零寫出的約 83.2 萬行生產級 Rust。

當然，不是所有那些回歸都出貨了。有些光是在 repo 裡開發時就被抓到。有些出現在 pre-release，但在進到穩定版前就修好了。而有些則進入了穩定版，通常是因為它們不起眼到足以穿過一輪或多輪 pre-release 使用而未被察覺。

![圖 17](https://github.blog/wp-content/uploads/2026/09/regression-taxonomy-1.svg)

*圖 17：已知的正確性回歸被歸成五種反覆出現的失效模式：遷移不完整、狀態與生命週期、行為合約不符、主機邊界，以及錯誤的測試判準（test oracle）。*

這當然不是零，而且我百分之百確定實際數量多於我們已知的那些。這些是我們注意到或被回報的；一場這種規模的遷移，絕對也出貨了一些安靜到目前還沒人踩到的回歸。就像 bug 一般的情況一樣，我預期隨著堆疊更深遠的角落在實戰中被積極操練，我們會持續陸續發現一些邊角案例的回歸。

絕對數量其實也沒那麼重要。更重要的是這些為什麼會發生，好讓我們從問題中學習、避免未來重蹈覆轍。

幾乎所有正確性回歸都落在三大家族中：新程式碼實作了不同的行為合約；狀態、所有權或生命週期行為改變了；或者遷移的某一部分被遺漏、只做了一半，或在 rebase 中弄丟了。另有一小群來自主機或互通邊界的需求，甚至來自那些自信滿滿地驗證了錯誤行為的測試。

這些家族在幾個反覆出現的模式中變得更具體。

#### 語意曖昧

好幾個回歸來自原始語言留白的隱含行為。TypeScript 只有一種 `number` 型別；Rust 則要求你在好幾種之間做選擇，包括值是否可以是小數。而 agent 會猜錯。

概念上是整數的欄位變成了 `f64`，於是 Rust 把值序列化成 `42.0` 而不是 `42`：像 Go 與 C# 這種強型別的 SDK 無法把 repo ID 反序列化成 `int64`，於是拒絕了一個 hook 時間戳與任務持續時間。反方向也有：某個 agent 把 `timeToFirstTokenMs` 宣告成 `i64`，但串流路徑送出的值長得像 `5446.712845`，導致寫入的 session 無法讀取、也無法恢復。

還有一個更微妙的案例與型別無關：`event.error || "Unknown error"` 被翻成了 `.unwrap_or("Unknown error")`。JavaScript 的 `||` 會取代空字串；Rust 的 `unwrap_or` 則會保留它，所以一個空的子代理錯誤訊息就一直是空的。糟糕。

#### 環境性行為（Ambient behaviors）

另一個反覆出現的來源，是 JavaScript 或 Node 無形中提供的行為。

配額程式碼用了 `toLocaleDateString`，它會沿用主機的時區；Rust 則需要把那個時區顯式傳進去。但儘管 `Intl.DateTimeFormat().resolvedOptions().timeZone` 的型別標示是 `string`，它其實可能回傳 `undefined`，而 napi 無法把它轉換成 Rust 的 `String`，導致模型清單載入失敗。

另一處，把一次環境變數讀取從 `await` 之前搬到之後，意味著等待期間主機若有變更就可能改變結果。某個原生擴充載入器呼叫 `process.report.getReport()` 只是為了識別平台，但在 Windows 上這會遵循 `_NT_SYMBOL_PATH`，可能會花上好幾分鐘下載 PDB 之後才渲染出 CLI。

其他環境性輸入還包括工作目錄、repository 身分、`PATH` 與 session 認證。把所有權搬進 Rust，需要決定每一項要在何時擷取、如何攜帶，以及何時更新。

#### 只移植了成對操作中的一半

好幾個回歸來自成對的操作失去同步。一個輪次上限檢查更新了原生註冊表的中止狀態，卻沒有取消行程內的模型迴圈，結果又讓一個請求溜了出去。另一處，任務完成被持久化並發出了事件，卻沒有被投射到活躍的 session 狀態中，於是 Autopilot 在任務完成後仍繼續執行。

#### 阻塞主執行緒

CLI 仍然是從 Node 的單執行緒事件迴圈驅動 Rust runtime，所以跨 napi 邊界的同步工作會凍結 UI。`/chronicle reindex` 就是這樣解析了數百個 session 檔案，讓渲染與輸入阻塞將近一分鐘。

把匯出改成 `async` 並把工作移到阻塞執行緒池的執行緒上就解決了。一次稽核找出更多可能造成阻塞的進入點，因而訂下一條常設規則：會做實質工作的 napi 匯出必須是非同步的，必要時使用 `spawn_blocking`。

#### 閃現的視窗

在 Windows 上，生成子行程時若沒有帶 `CREATE_NO_WINDOW`，會短暫開啟一個主控台視窗。Node.js runtime 曾對行程生成做了猴子補丁（monkey patch）來加上這個旗標，這等於對移植中的 agent 隱藏了這項需求；Rust 的替代實作就漏了它。一次稽核修正了另外兩處生成點，不過那兩處是本來就有的遺漏，而不是移植造成的回歸，我們也把這條規則加進了 copilot 指示中。

#### 生命週期管理

最大的一群牽涉生命週期、釋放（disposal）、所有權、順序或競爭條件。

把狀態搬進 Rust，常常讓 TypeScript 手上只剩一個指向原生表中某個實例的不透明控制代碼（handle）。跟物件參考不同，那個控制代碼可能比實例活得更久。某個 hook 在請求進行中被釋放，孤立了一個 `tool_use` 區塊，讓對話卡死，因為模型 API 要求必須有對應的結果。一個在「已宣告」與「已啟動」之間被取消的 shell，洩漏出一個孤兒行程，讓 session 一直保持活躍。一個沙箱開關更新了一個世代計數器卻沒更新它的原生孿生計數器，讓 shell 卡在「重新設定中」。

#### 被忽略的功能

另一群牽涉到移植根本漏掉的功能。

某次移植省略了 SDK 回呼並刪掉了它們的端到端測試，因而催生出一條新規則：agent 不得在未經明確同意下修改 E2E 測試。一次 session 中止保留了它的原生那一半，卻遺失了「中斷一個正在工具中等待的輪次」所需的行程內取消機制。

SDK 取代內建工具搜尋的能力，取決於三件事：啟用、面向模型的描述與綱要，以及把執行路由到 SDK 回呼。這次移植把三者全都弄丟了（一致性可喜可賀？），無聲地把某個消費端的自然語言搜尋換成了正規表達式搜尋。

#### 不同函式庫，不同脾氣

還有幾個回歸來自把 JavaScript 函式庫或 API 換成更嚴格的 Rust 等價品。在當時，Rust 的 MCP SDK（`rmcp`）會對格式錯誤的 JSON-RPC 輸入做出回應，而 TypeScript SDK 不會。面對一個會用更多格式錯誤輸出來回應錯誤的伺服器，那份禮貌就變成了讓啟動卡死的無限迴圈。不同生態系中的相關函式庫，行為鮮少完全一致。

#### 擦身而過的船

少數幾個回歸來自分支偏移或 rebase。在一個每週數百個 pull request 的 repo 裡，一個開著好幾天的 session 會累積源源不絕的變更與衝突。這次移植需要數千次 rebase；即使成功率非常高，還是會留下一些失敗。

#### 還有那些慢郎中

最後一群功能上是正確的，只是比較慢。

有些回歸弄丟了既有的效率手段，例如記憶化（memoization）、完全非同步的等待，或有界的日誌串流。另一些則在 Rust–TypeScript 邊界上增加了遷移專屬的額外開銷：冗餘的序列化、鎖定、輪詢、無界的原生並行，以及原生到主機的跨界。

這些並不是「之後可以最佳化的機會」；每一個都是移植所引入的劣化。有一次唯讀掃描深拷貝了一份 260 MB 的事件日誌，而不是借用它。在持續的事件流量下，另一個實作只完成了所請求通道 flush 中極小的一部分，同時為每個事件保留一個非同步控制代碼，直到 V8 把它的 heap 耗盡。

### 這些回歸有被使用者「感受到」嗎

數十個回歸聽起來很多。但在一場產生超過 80 萬行程式碼的移植中，說真的，我很意外也很欣慰我們沒有遇到多一個數量級的回歸。

我們也可以透過觀察公開 repo [github/copilot-cli](https://github.com/github/copilot-cli) 與 [github/copilot-sdk](https://github.com/github/copilot-sdk) 的 issue 趨勢，來感受這些回歸是否被廣泛「感受到」。這裡我們把 1 月到 8 月開立的 issue 分類為品質相關——條件是它帶有 bug 標籤，或標題使用了常見的失效詞彙如「bug」、「regression」、「crash」、「hang」、「timeout」、「broken」或「incorrect」。

相較於移植工作之前，移植期間與之後的水準基本上沒有變化：

| Repository | 1–4 月（重寫之前） | 5–8 月（重寫期間／之後） |
| --- | --- | --- |
| github/copilot-cli | 22.9%（454 / 1,982） | 23.7%（354 / 1,496） |
| github/copilot-sdk | 36.2%（190 / 525） | 32.3%（135 / 418） |

這不是可用性指標，也不是逃逸缺陷的精確計數……就像那些回歸本身一樣，一個 issue 代表什麼、範圍多大等等都有極大的變異性。但它確實提供了一個有用的檢核：儘管產品變動量非常驚人，面向產品的 issue 管道在遷移期間並沒有出現有意義的品質疑慮尖峰。

---

## 13. 能編譯，就只是能編譯

前面我提到一個流行的迷因：Rust 因為編譯器嚴格，所以很適合 AI 生成的程式碼。還有另一個相關且流行的 Rust 迷因：如果程式碼能編譯，它就是正確的。

並不會。我們的已知回歸清單就是對這點的好回應：語料庫中的每一個回歸都被合併進了 `main`，這代表它們全都成功編譯過。編譯器接受了那些有 bug 的版本，因為就編譯器而言，它們每一個都是合法的 Rust。

編譯器可以證明一個 `f64` 被一致地使用。它無法知道一個 repository ID 必須被序列化成整數，或是一個帶著尾隨 `.0` 的時間戳會被線路另一端每一個強型別 SDK 拒絕。

編譯器可以防止它看得見的程式碼中出現未同步的資料競爭，但它無法防止一個同步得完美無瑕的狀態機編碼了錯誤的狀態。一個佇列可以被鎖保護得好好的，卻仍然讓兩個傳送端各自認定對方會去清空它。事件可以安全地在執行緒間移動，卻仍然以錯誤的順序抵達。一個同步的 napi 函式可以是記憶體安全的，卻仍然把 Node 的主執行緒阻塞一分鐘。

編譯幾乎在定義上也無法偵測「不存在的東西」。當一次 rebase 悄悄移除了一道防護與它的測試，或當一個行程生成器忘了那個抑制主控台視窗彈出的 Windows 旗標時，編譯器無從反對。它對「每次讀取都複製一份 250 MB 的事件日誌」也沒有任何意見。

編譯器檢查的是「你寫出來的程式是否內部自洽」。它無法檢查你是否寫完了整個程式、是否保留了舊的合約、是否以正確順序呼叫、是否滿足了主機那些沒寫下來的要求，或者是否以可接受的成本完成了工作。

這絕不是在反對 Rust 的編譯器。就像任何靜態型別語言一樣，編譯器消除了一大類機械性錯誤，並給了 agent 一個極其有用的內層迴圈。但「能編譯就是正確」只有當笑話講才有用。

---

## 14. 效能、效能，還是效能

那麼，我們用這一切買到了什麼？

這次移植刻意保持行為不變。它並不打算重新設計演算法或修 bug；事實上我一再把 agent 從順手最佳化的衝動中拉開，因為同時改變語言與行為，會讓「是哪一個害你壞掉」變得難上加難。

不過，重寫的一個關鍵目標確實是效能與可擴展性（此外還有可靠性等因素）。當有人問我為什麼要把 runtime 重寫成 Rust，我的回答常常是：「我並不是想搬到 Rust，我是想搬離 Node.js 與 V8。」而這一搬，確實在效能上讓我們的指標大幅移動。

### 量測方式

我透過 C# SDK 對 runtime 的幾個情境做了移植前後的基準測試（TypeScript、Python、Go、C#、Java 與 Rust 的 SDK 全都透過同一套傳輸架構抵達同一個引擎）。

基準線是移植前版本的 SDK 與 CLI，TypeScript runtime 由 Node 托管、透過 stdio 連接。8 月 21 日的結果使用 Rust runtime，同時測了行程外伺服器與透過 FFI 載入行程內兩種方式。這是對交付系統的端到端比較，而不是試圖隔離出語言變更本身的效果；同一期間還有其他變更落地，所以這些數字得酌情看待。

每一次計時的輪次都送到一個跑在 localhost 上的確定性聊天補全伺服器，產生固定且很小的回應。換句話說，這些數字刻意排除了模型推論與網路延遲。它們量的是我們改變的那部分：client 啟動、行程啟動、session 建立、事件處理、持久化、拆除等等。

| 情境 | 5 月 12 日 | 8 月 21 日（行程外） | 8 月 21 日（行程內） |
| --- | --- | --- | --- |
| Client、session、一輪對話 | 5.25 秒 | 1.33 秒（4.0 倍） | 292 毫秒（18.0 倍） |
| 恢復 32 輪的 session | 5.64 秒 | 1.52 秒（3.7 倍） | 264 毫秒（21.4 倍） |
| 十個並行 client 生命週期 | 12.34 秒 | 4.18 秒（3.0 倍） | 742 毫秒（16.6 倍） |
| 1,000 次單輪 session 生命週期 | 132.52 秒 | 22.53 秒（5.9 倍） | 20.93 秒（6.3 倍） |

![圖 18](https://github.blog/wp-content/uploads/2026/09/performance-1.svg)

*圖 18：從 5 月 12 日到 8 月 21 日，建立 client 與 session、完成一輪對話再拆除，從 5.25 秒改善到行程內的 55.3 毫秒；吞吐量從每秒 7.55 個 session 上升到 120.0 個；十個 client 的記憶體從 1,383 MB 降到 126 MB。*

### 延遲

「client、session、一輪對話」這個數字最容易讓人有感。它是建立一個 client、建立一個 session、做一輪對話，然後把一切拆除。

行程外開銷有很大一部分來自啟動 Node、初始化 V8，以及載入、解析並為「由 TypeScript 產生的 JavaScript 應用程式」生成 bytecode——這些都得在第一輪對話開始之前完成。Rust runtime 移除了那些 Node/V8 與 JavaScript 載入成本。

### 吞吐量

「1,000 次單輪 session 生命週期」這個壓力測試，代表的是我們能建造什麼東西的階梯式躍升。它使用單一共享 client，量測執行 100 條並行管線，每條管線建立一個 session、走完一整輪模型對話、釋放 session，如此依序做十次。

- 移植前的 TypeScript CLI：每秒 **7.55** 個生命週期
- Rust 行程外：每秒 **57.45** 個
- Rust 行程內：每秒 **120.0** 個

那是一個工作負載專屬的結果；Rust runtime 並不是普遍「快 15.9 倍」。但它正是伺服器主機在意的那種工作負載：許多獨立 session 共用一個 runtime。

而且 wall-clock 時間並沒有把工作藏到另一顆核心上。在對同一個 100×10 工作負載所做的另一次資源取樣中，移植前的行程樹消耗了 312 秒的聚合 CPU 時間。Rust 的各種配置約消耗 110 秒。那是主機可以拿去跑更多 session 的 CPU 容量。

### 記憶體

記憶體說的是同一個故事，不過照例要提醒：記憶體指標非常容易被誤用。就十個 client 批次執行期間新增的常駐私有記憶體來看：

- 移植前的行程樹峰值：比基準線高出 **1,383 MB**
- Rust 行程外峰值：**247 MB**
- Rust 行程內峰值：**126 MB**（低了一個數量級）

雖然這些數字當然會因使用情境與機器而異，但它們直指核心目標：一個服務可以在同一台機器上托管遠多於以往的 client 與 session，才會碰到記憶體、行程數或 CPU 成為限制資源。

最棒的是，這還只是移植後的基準狀態。實作中有很大一部分仍然是「被忠實地用 Rust 渲染出來的 TypeScript 形狀演算法」。我們還沒有做那些新的所有權模型、並行模型與行程內架構所開啟的大規模重新設計。

在那些最佳化工作開始之前，就能讓一個行程內 client 在約 55 毫秒內完成建立、跑完整整一輪 session、再拆除，讓共享 client 推到每秒 120 個單輪 session 生命週期，並把量測到的十 client 記憶體增量砍掉 91%——這是個非常好的起點。

---

## 15. 這次移植花了多少

那麼……這一切在金錢上花了多少？請下鼓聲……

我在所有移植工作上的 token 花費約為 **1,363 億個 token**：

| 類別 | 數量 |
| --- | --- |
| 快取輸入讀取 | 約 1,306 億 |
| 快取輸入寫入 | 約 42 億 |
| 全新輸入 | 約 9 億 |
| 輸出 | 約 6 億 |

所有這些 token 的帳單金額約為 **12 萬美元**。

當然，這些 token 不會自己花掉自己。使用一位開發者大量時間去引導這些 agent 的成本也應該被算進來。不過，我並沒有把所有時間都投注在這個專案上。

代理式開發有很多「趕快等一下」的成分：送出提示詞，讓編碼 agent 去做它的事，時不時回來看看、必要時導正方向，而在它完成之前的這段時間裡去做別的事。這意味著開發者不再一次只做一件編碼任務；他們把等待時間疊起來，好讓自己能同時處理很多事。

在移植期間，這些 Rust 移植 PR 約佔我在所有有貢獻的 repo 中所開 PR 的 20%。如果我們大手一揮，用 pull request 的佔比來近似我的時間佔比，那大約相當於三週專門投入這次移植。

換句話說，這次移植的粗略帳單是：**約 12 萬美元的歸屬 token 花費，加上三週的開發者時間**。

### 這不是一個人的工作

話說回來，這場端到端的 Rust 遷移並不是 100% 靠一位開發者；它一直是團隊努力的成果。

- [@stevesandersonms](https://github.com/stevesandersonms) 提供了 `napi-oop`（暫時性的跨行程互通層）的設計與實作，以及六個 SDK FFI 實作中的五個；第六個由 [@edburns](https://github.com/edburns) 提供。
- [@roji](https://github.com/roji) 實作了讓 SDK 能正確出貨並使用 Rust 二進位檔的打包機制。
- [@caarlos0](https://github.com/caarlos0) 協助把 Rust 程式碼拆成許多小的子 crate，以改善開始變得棘手的建置時間。
- [@criemen](https://github.com/criemen) 協助改進資產快取以加速 CI 與本機建置。
- [@devm33](https://github.com/devm33)、[@examon](https://github.com/examon)、[@MRayermannMSFT](https://github.com/MRayermannMSFT)、[@dereklegenzoff](https://github.com/dereklegenzoff) 以及其他人協助了數不清的 pull request 審查與核准。

而所有為 copilot-agent-runtime repo 做出貢獻的人，在世界於他們腳下位移的同時，都一直給予支持與包容。

---

## 16. 學到的教訓

這段經驗強化了幾個超出這次重寫範圍的教訓。以下是我們下次會帶著走的一些東西。

**1. 目標需要被清楚且完整地陳述。**

我們早期的指示太模糊了。「把 XYZ 元件移植到 Rust」被讀成只做熱路徑、或只做邏輯，agent 一再把 I/O 與編排當成範圍外。一旦我們明確說出最終狀態是「一個由 100% Rust 程式碼庫建置出來的原生二進位檔，就算我們想留也沒有任何執行環境留給 TypeScript」，它們就變得非常擅長自主地朝那個目標推進。

**2. 端到端測試絕對、毫無疑問地至關重要。**

除了一個例外，所有牽涉功能遺漏的回歸，以及其他許多回歸，都肇因於端到端測試不足。對任何這種性質的移植，你必須有能用來驗證移植正確性的測試，而那些測試本身在移植期間不能被改寫，否則你就失去了你的判準（oracle）。

我們這次移植的初始計畫有點出這件事，也點出我們需要在開始移植之前大幅改善 E2E 測試的態勢。我們做了，但做得不夠。我們很確定，如果在開始移植前再多加一些 E2E 測試，真正著重於確保大多數有意義的行為都被涵蓋，我們一路上遇到的回歸會比實際少。

**3. 保護判準不受 agent 侵蝕。**

那個改動實作的 agent，不能同時被允許透過削弱測試、更新快照、拉高相容性基準線或套用逃生門標籤，來悄悄重新定義什麼叫正確——至少不能在沒有監督的情況下。盡可能讓行為合約獨立存在，把敏感的護欄放在另外的所有權或核准之後，並疊加具有不同失效模式的檢查，這樣單一個失誤就不足以讓一個大回歸出貨。

**4. 先翻譯，後重新設計。**

保留行為與既有演算法，讓同時變動的變數數量維持在可控範圍內。等舊實作與過渡期的鷹架都消失之後，所有權、並行與效能就能對著一個穩定的基準線重新設計。

我有幾次偏離了這條原則——出於躁動、出於難以對同儕說不、或出於「這個案例不一樣」的信念——而事後回顧，我對每一次都感到後悔。每一次的代價都是更多回歸、更多時間，或更多 token，超過堅持原則所需付出的。

**5. 把重複的失敗轉化為未來的成功。**

AI agent 一定會走偏；當它們走偏時，就從中學習。當同一種失效模式出現第二次，它就該被寫進常設指示、可重用的技能、eval、受保護的基準線，或 harness 本身。

**6. 有 agent 參與時，開發者的內層迴圈更重要，而不是更不重要。**

身為開發者，我們活在自己的內層迴圈裡：能多快做出修改、建置、測試、迭代。當工具那一段拖太久，我們會很煩躁。而當 agent 在做較低層的工作時，會以為這件事不再適用，這也情有可原。

但它仍然適用，而且更適用。AI agent 做「思考」與「寫程式碼」的部分快得不可思議，但它們仍然要建置、仍然要測試。而且當它們花在思考與撰寫上的時間變少、花在快速驗證內層迴圈上的時間變多時，建置與測試所佔的時間比例其實是上升的。

花點時間在前期最佳化你的內層迴圈，並針對「多件事同時進行」去最佳化它（例如就當作你同時在多個 worktree 裡處理多個任務）。日後你會感謝自己的這份投資。

---

## 17. 接下來呢？

它成功了。五月時還完全是 TypeScript 的執行 runtime，到了八月已經完全是 Rust，而且整個過程中持續出貨給真實使用者，而不是在最後來一次令人膽戰心驚的切換。

我並沒有單純地叫一個 AI agent「把整個程式碼庫從 TypeScript 移植到 Rust」。就算產業終將走到那一步，我們現在絕對還沒到。

真正發生的是，agent 讓一整個類別的專案變得可行。一場產出數十萬行生產級 Rust、在活系統上、原地在 `main` 裡進行、由一位工程師在團隊支援下完成的重寫，在 agent 出現之前根本不會是一個會被批准的提案。它會需要一整個團隊、花上一兩年，它會和那個團隊原本能交付的每一項功能競爭資源，而且它會輸（老實說，也應該輸）。Agent 把價格拉到了讓這個專案站得住腳的位置。

移植本身已經完成：runtime 的生產實作是 100% Rust，暫時性的內部 TypeScript/N-API 接縫已經消失。不過還有許多我們想做的工作：進一步改善建置系統與開發者內層迴圈、清理翻譯過來的結構、圍繞 Rust 的所有權與並行模型重新設計，以及追求更多效能上的斬獲。

這次移植是一次翻譯，而且是刻意如此（提示詞就是那樣寫的），我們讓行為盡可能維持 100% 相同，而不是順手多修別的 bug、重新架構元件，或進一步改善效能與可擴展性（除了重寫本身隱含帶來的那些之外）。在微觀層面，大部分程式碼是慣用的 Rust；但在宏觀層面，有相當多原本是 TypeScript 的演算法只是穿上了 Rust 的語法。既然底下的限制條件已經改變，重新檢視那些決策，才是有趣的斬獲所在。

我最興奮的是這次移植讓什麼變得可能。SDK 可以用六種語言中的任一種直接載入到主機行程中，相依鏈裡沒有 Node.js 或 V8，也沒有第二個行程要監督——而那正是我們從採用 SDK 的合作夥伴那裡聽到最常見的單一摩擦點。

一個成本只剩原本一小部分的 runtime 實例，意味著一台主機在耗盡機器資源之前能跑遠多於以往的並行 session。而且 runtime 現在可以去到 Node.js 永遠不會跟去的地方，橫跨從雲端到桌面、到裝置、到嵌入式系統的整個光譜。

這些都不是終點線。它是我們現在能在其上建造 GitHub Copilot 未來的基礎；在看了三個月 agent 重寫那個運行它們自己的引擎之後，我很期待看到它還能走多遠。

Happy coding！

---

**標籤**　[AI agents](https://github.blog/tag/ai-agents/)　·　[developer experience](https://github.blog/tag/developer-experience/)　·　[GitHub Copilot](https://github.blog/tag/github-copilot/)　·　Rust

**原文**　[Migrating the GitHub Copilot runtime to Rust, using Copilot — The GitHub Blog](https://github.blog/ai-and-ml/generative-ai/migrating-the-github-copilot-runtime-to-rust-using-copilot/)
