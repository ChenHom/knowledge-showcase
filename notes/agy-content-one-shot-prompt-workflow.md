# agy-content：從 one-shot prompt 工具到可驗證內容 workflow 的演進判斷

整理日期：2026-06-28

這篇不是單純記錄 `agy-content` 新增了哪些 command，而是整理一條比較可複用的判斷脈絡：什麼時候一個內容工具還停留在 one-shot prompt 階段，什麼時候已經值得升級成 content workflow，以及為什麼這次 `agy-content` 會自然長出 `base + overlay`、`checklist`、`self-review`、二段式 workflow 這些結構。

## 核心結論

`agy-content` 這次的演進，重點不是「多包幾個 prompt」，而是從「一次生成工具」往「可控的內容生產流程」跨了一步。

真正的定位改變有兩個：

1. 它不再只是幫忙快速翻譯、摘要、潤稿、發想。
2. 它開始承擔輸出品質、驗收與可回查的責任。

所以這次最重要的定案不是 token 省不省，而是：

- `agy-content` 的定位是**產文件工作器**
- 品質流程先固定在 **generate -> self-review**
- 暫時**不做自動重寫**

## 什麼時候 one-shot tool 還夠

維持單次 prompt 工具通常有這些特徵：

- 任務短、低風險、可快速人工掃過
- 失誤成本低，重跑一次就好
- 不需要固定驗收標準
- 任務重複率不高
- 產出不會成為後續流程的重要輸入

典型例子：

- 短訊息潤稿
- 一小段英文快速翻中
- 臨時摘要一篇文章
- 快速丟幾個標題或點子

這種情況下，one-shot 夠快，也夠省心。

## 什麼時候該升級成 content workflow

當內容工具開始碰到下面任兩項，就值得考慮升級：

1. **重複率高**：同類內容持續重做，代表規則值得顯式化。
2. **失誤成本高**：內容錯了會影響決策、對外溝通、求職、商業判斷、規格理解。
3. **有固定驗收標準**：例如一定要保留數字、先講結論、不能跑掉語氣。
4. **產出會被下游繼續使用**：例如摘要再變 briefing、翻譯再變正式文件、草稿再進下一輪編修。
5. **需要 explainability**：開始在意「這份內容為什麼長這樣」、「哪條規則在影響它」、「為什麼判 PASS / REVISE」。
6. **開始想加 routing / review / reject**：也就是不只是生成，而是想控制過程。

一句話講白：

如果要的是「幫我生一份」，one-shot 通常夠。

如果要的是「幫我穩定地一直生出可用的東西」，就該升級成 workflow。

## 為什麼 `agy-content` 這次會演變到 `base + overlay`

`base + overlay` 不是為了 prompt 架構漂亮，而是工具長大後很自然會被逼出來的結構。

原因是：

### 1. 共用規則開始穩定

像這些規則：

- 用正體中文
- 避免簡體
- 避免某些用詞
- 只輸出結果

一開始散在每個 template 還勉強可忍，但當它們變成整個工具的共同地板時，就需要單一來源，不然每個 task 會慢慢漂掉。

### 2. task 差異開始清楚

不同內容任務的價值點不一樣：

- `translate` 重 fidelity、術語與保結構
- `summarize` 重結論優先、重點排序、風險提示
- `polish` 重保留原意、流暢度與語氣
- `ideate` 重建議方向、trade-off 與下一步

如果把這些差異硬塞進同一大坨 prompt，很快就會變成難懂又難改的雜湊物。

所以共同地板與任務特化必須拆開。

### 3. 開始需要 debug

當輸出有問題時，會開始問：

- 是 style profile 太硬？
- 是共用規則設錯？
- 還是某個 task 的規則有偏？

`base + overlay` 的實際價值，就是讓這種問題有邊界，能定位是 base 問題還是 task 問題。

### 4. 後面準備接 workflow

只要下一步要做：

- `self-review`
- 生成後驗收
- review routing
- 多階段 workflow

就很難接受 prompt 還是一堆彼此複製貼上的字串。因為之後每多一層流程，維護成本都會一起爆。

## 這次 `base + overlay` 的真實價值

這次實作後有一個很重要的現象：

runtime token 沒有省，甚至還微幅增加。

但這不代表這步是錯的。

這步真正換來的是：

1. **共用規則有單一來源**
2. **task 差異有清楚邊界**
3. **debug 可拆 base 與 task 來看**
4. **之後比較容易接上 workflow**

也就是說，這不是 token optimization，而是 quality engineering / workflow engineering。

## 為什麼先做 checklist，而不是直接做自動重寫

這次的品質控制是刻意分三層長出來的：

1. **Checklist**
2. **Self-review prompt**
3. **Workflow integration**

這個順序比一開始就做「生成後不滿意就自動重寫」穩很多。

原因有三個：

### 1. 先定驗收標準，再定 review 行為

如果連什麼叫好輸出都還沒講清楚，就讓系統自動改第二輪，最後只會變成模型自己評自己、自己改自己，但人類其實不知道它是根據什麼在改。

### 2. 先讓 review 可見

`self-review` 先做成顯式 command，再嵌進 workflow，價值在於你可以看到：

- 它怎麼判 PASS / REVISE / REJECT
- 它抓到哪些問題
- 它的審稿邏輯有沒有跑偏

這樣之後要不要自動化，決策才有根。

### 3. 先 review，不直接 revise

這次刻意停在：

```text
generate -> self-review
```

而不是：

```text
generate -> self-review -> auto revise
```

因為對文件型輸出來說，第二輪自動重寫的風險很明確：

- 可能把原意改掉
- 可能加入模型自己腦補的內容
- 可能讓語氣變得更一致，但內容更不忠實

所以比較穩的邊界是：

- 先 review
- 先讓人看
- 真的需要時，再決定要不要把 review 建議餵進下一輪

## `agy-content` 目前的成熟度在哪裡

如果把內容工具成熟度粗分，大概可以這樣看：

1. **One-shot prompt**
2. **Template tool**
3. **Template + shared base**
4. **Template + review loop**
5. **Content workflow**
6. **Production content pipeline**

`agy-content` 現在大概在第 4 與第 5 之間：

- 已有模板
- 已有 base + overlay
- 已有品質 checklist
- 已有 self-review command
- 已能跑 `generate -> self-review`

但還沒到 fully automated production pipeline，因為它目前刻意不做：

- `REVISE` 後自動重寫
- review 決策後的下一步 routing
- task / risk 導向的專屬 review routing

這不是缺點，而是邊界選擇。

## 這次定下來的工程判斷

這次我認為值得固定下來的判斷有這些：

1. `agy-content` 的定位是產文件工作器，不是省 token 工具。
2. `base + overlay` 的主要價值是規則集中、品質穩定、debug 邊界與 workflow 接軌。
3. 品質控制先走 `checklist -> self-review prompt -> workflow integration`。
4. `self-review` 先做成顯式 command，再嵌進內容 command。
5. 二段式 workflow 先固定在 `generate -> self-review`。
6. `REVISE` 暫時交給人看，不讓工具默默第二輪亂改。

## 對未來的啟發

這次的經驗不只適用於 `agy-content`。

它其實給了一個很通用的升級判斷：

當一個內容工具開始承擔：

- 穩定輸出
- 明確驗收
- 下游依賴
- review 可見性
- 決策可回查

它就不再只是 prompt wrapper，而會自然演變成 workflow。

而當它演變成 workflow 之後，最先長出來的通常不是更花的模型 routing，而是：

- shared base
- task overlay
- quality checklist
- review loop

這些才是後面自動化的地基。

## 一句話總結

`agy-content` 這次不是單純多了幾個 prompt，而是從「幾個可用的內容命令」跨到「有規則分層、有品質驗收、有自評流程的內容產線雛形」。
