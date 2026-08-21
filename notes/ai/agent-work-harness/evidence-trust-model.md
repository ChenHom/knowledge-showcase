# Evidence 可信度的四層模型

#ai #agent #evidence #verification #testing #knowledge

## 這是什麼

自動化系統判定「工作完成了」時，依據的是什麼？這篇把「evidence 有多可信」拆成四層，
並記錄一個真實的 false positive 案例與它的修法。

適用於任何「機器判定成功」的場景：CI 驗收、Agent 工作驗證、自動化部署 gate。

## 核心結論

> 大多數系統只解到第一層不等式就停了，但後面還有兩層。

```text
Agent 說完成    ≠  真的完成        ← 多數系統解到這裡
驗證通過        ≠  驗證完整
驗證完整        ≠  需求真的正確
```

第一層靠「不採信自述、自己收集 evidence」解決。第二層第三層需要不同的機制，
而且第三層基本上解不了 —— 重點是**知道它解不了並誠實標示**，而不是假裝解決了。

## 四層

| 層級 | 要回答的問題 | 觀察對象 |
|---|---|---|
| **E1 Integrity** | 這份 evidence 真的是執行出來的嗎？ | exit code、timeout、實際指令、執行環境、輸出是否完整 |
| **E2 Completeness** | 該跑的東西真的都跑了嗎？ | 執行數量、skip 數量、runner 是否降級 |
| **E3 Independence** | evidence 是不是被驗證方自己製造來證明自己的？ | 測試的來源：既存 / 本次新增 / 本次修改 |
| **E4 Sufficiency** | 這些 evidence 足以支撐這個完成宣告嗎？ | 單元測試通過能不能證明線上問題被修掉 |

四層是**遞進**的：E2 只在 E1 成立時有意義，E3 只在 E2 成立時有意義。
一份 timeout 的 evidence 談不上完整性；一份跳過一半測試的 evidence 談不上獨立性。

## 一個真實的 false positive

判定鏈原本是最直覺的那種：

```text
Repository 設定 → argv → 隔離執行 → exitCode == 0 → PASS
```

實際發生的事：Agent 為它的修正補了一條斷言，那條斷言在正常環境會失敗。
但該測試檔在隔離環境中被整個 skip 掉，`npm test` 仍然 exit 0，
evidence 記為 PASS，最終判定為成功。

**`exitCode == 0` 只能回答「這條命令沒有失敗」，不能回答「該跑的有沒有跑」。**

## E2 的解法：pre-flight baseline

要判斷「有沒有比動手前少跑」，需要一個比較基準。兩個直覺的來源都不能用：

- **從上一次成功的紀錄取** — 第一次執行沒有基準
- **由被驗證方靜態宣告** — 可以謊報，而且會隨測試增減立刻過期

可行的是 **pre-flight baseline**：在 Agent 動任何東西之前，先自己跑一次同一組檢查。

```text
動手前:  total 63, pass 63, skip 0
動手後:  total 63, pass 55, skip 8
         ↑ exit 仍然是 0，但少跑了 8 個
```

判定規則可以很小，不需要狀態機：

```text
timeout                          → INCONCLUSIVE
輸出超限（行程被殺）               → INCONCLUSIVE
exit != 0                        → FAIL
executed < baseline.executed     → INCONCLUSIVE
skipped  > baseline.skipped      → INCONCLUSIVE
其他                              → PASS
```

`exit != 0` 那條放在 baseline 比較之前，「基準本來就失敗、現在也失敗」自然就是 FAIL，
不必為它多寫任何規則。

### 兩條貫穿的原則

```text
未知不等於失敗    解析不出執行數量、或 baseline 本身跑不起來 → 退回原本行為
不完整不等於通過  timeout 與輸出超限一律 INCONCLUSIVE
```

第一條特別重要：如果解析能力的缺口會讓專案永遠無法通過驗收，
這個機制就會被關掉，等於沒做。**未知要標示，不要懲罰。**

### 成本

baseline 讓驗證時間翻倍。實測兩個真實專案分別是 12 秒與 0.6 秒，翻倍完全無感 ——
所以快取（需要持久化與失效邏輯）先不做，只提供一個關掉的開關。
等真的有跑幾分鐘的專案再說。

## E3：真正該問的問題

不是「Agent 有沒有改測試」，而是：

> **把這次新增和修改的測試全部拿掉之後，還剩下什麼證據？**

如果一個修正的證據全部來自 Agent 本次新寫的測試，那是自己出題自己答。
反過來，如果有一個**既存**的失敗測試在這次變成通過，那是強得多的證據 ——
它在 Agent 介入前就存在，不可能是為了配合這次修改而寫的。

```text
動手前:  test_login_500  FAIL
動手後:  test_login_500  PASS      ← 最強的單一指標
```

而且它是 pre-flight baseline 的**免費副產品**，不需要額外成本。

實務上不該把 E3 做成阻擋機制（正當的測試更新非常常見），而是把組成攤開：

```text
- npm test：PASS
    63 個測試全部執行，0 skip（baseline: 63 / 0）
    既存測試 58 通過，其中 1 個由 FAIL 轉 PASS
    本次新增測試 3 個通過
```

比單純一行 `PASS` 的資訊量高得多，而且判斷權留給人。
壓成一個 0.87 的信心分數反而丟失資訊。

## E4：知道它解不了

需求是「修掉登入偶發 500」，系統看到的是「單元測試通過、型別檢查通過」。
它能證明的是 `configured checks passed`，不能證明 `線上不再有 500`。

這一層的正確做法是**命名誠實**：

```text
ACCEPTED BY EVIDENCE CONTRACT     ✓ 準確
SUCCESS                           ✗ 過度承諾
```

實測中出現過一個漂亮的例子：Agent 被要求跑一個品質驗證腳本，服務連不上，
腳本回報 `stable_ranking=true`、`negative_zero_results=true`。
Agent 正確識別出「這兩個 true 只是**空結果的衍生值**，不能當品質通過證據」。

這正是 E4 的形狀 —— 指標為真，但它證明的不是你以為的那件事。

## 一個反直覺的觀察

Agent 會**迎合寫錯的測試**。

驗證場景裡曾經故意讓測試的期望值寫錯（預期 7，正確是 6），
需求只說「讓測試通過」。Agent 在實作裡加了 `+1` 去迎合那個錯誤斷言。

系統判定為成功 —— 機械上完全正確：測試通過、沒有越界、驗證都跑了。

這不是 evidence 的可信度問題，是 **evidence 的相關性問題**。
兩者很容易被混為一談，但前者可以改善，後者機械上不可判定。

## 該做與不該做

| 做 | 不做 |
|---|---|
| 把執行規模記進 evidence（執行數、skip 數） | 用第二個 LLM 判斷 evidence 可不可信 |
| 用 pre-flight baseline 當比較基準 | numeric confidence score |
| 未知就標未知 | 為了填滿欄位而猜測輸出格式 |
| 把 evidence 的組成攤開給人看 | 通用的 sufficiency engine |

最後一欄的共同點：它們都把「不可驗證性」搬到另一個地方，然後假裝解決了。

## 相關

- [Sandbox 隔離的實測邊界](note.html?slug=ai/agent-work-harness/sandbox-isolation-findings)：evidence 收集本身也要隔離執行
- [設計決策與它們的理由](note.html?slug=ai/agent-work-harness/design-decisions)：為什麼不做信心分數與 LLM 判定
