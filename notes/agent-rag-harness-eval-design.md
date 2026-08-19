# Agent、RAG、Harness 系統 Eval 功能設計

#eval #agent #rag #harness #system-design #ai-engineering #openclaw #regression-test

## 這是什麼

這份筆記整理一套用來評估 Agent / RAG / Harness 系統的 Eval 設計。重點不是做漂亮 demo，而是把系統裡容易退化、容易被改壞、但又很難靠肉眼長期盯住的規則，變成可重複執行的檢查。

核心原則：

> Eval 不是只拿來評分模型回答好不好，而是整個 AI 系統的防回歸機制。

對 Agentic system 來說，真正會壞的地方通常不是單一 prompt，而是：

- 工具選錯
- 權限越界
- RAG 找錯資料
- context packing 把重要片段擠掉
- Harness 沒有留下可恢復狀態
- 通知 / 寫入 / commit / push 等 workflow 規則被忘記
- fallback 表面可用，但實際輸出品質退化

所以 Eval 要從「單點答案評分」提升成「系統行為契約檢查」。

---

## 一、Eval 的分層

### 1. Regression Eval

Regression eval 用來保護已知規則與真實踩過的坑。

它的問題不是：

> 這次回答有多聰明？

而是：

> 以前修過的東西，有沒有又壞掉？

適合檢查：

- `know` / `knowledge-mirror` 寫入後是否 commit + push
- Ollama 是否固定透過 `ollama-presets` skill，而不是直接打 API
- Claw Notify 是否推送摘要而不是原文片段
- FCM sender 是否只推最新 device，避免重複通知
- workspace 偏好與合作規則是否仍寫在 memory / tools note

這類 eval 應優先 deterministic，不需要 LLM-as-Judge。

---

### 2. Component Eval

Component eval 用來測單一元件是否符合契約。

常見元件：

- tool adapter
- planner
- retriever
- reranker
- context packer
- summarizer
- approval gate
- state writer

例子：

```txt
輸入：一個需要查 know 的任務
預期：Agent 先查 knowledge-mirror，而不是直接憑記憶回答
```

```txt
輸入：一篇含下載連結、提取碼、雜訊文字的文章
預期：summary cleaner 移除 URL / pan.baidu / 提取碼，不把原文垃圾推到手機
```

Component eval 的價值是定位問題快：壞了就是某個元件的契約破了。

---

### 3. End-to-End Eval

End-to-end eval 測整條 workflow 是否完成。

例子：

```txt
輸入：一個 YouTube 影片 URL
預期：
1. 取得 transcript
2. 整理草稿
3. 使用者確認後寫入 know
4. git add / commit / push origin master
5. 回報 commit hash
```

E2E eval 可以捕捉 component eval 看不到的整合問題，例如：

- 每個元件單獨都對，但串起來漏掉 commit
- RAG 有找資料，但 context packing 把關鍵段落刪掉
- Agent 做完任務，但 Harness 沒留下 task-state / verification report

---

### 4. Live Eval

Live eval 直接檢查真實環境狀態。

例子：

- 查 Firestore 最新 `notifications`，確認 summary 不含 URL / 原文片段 / 提取碼
- 查 Firestore `devices`，確認至少有 device，且 sender 邏輯只推最新 token
- 查 repo working tree 是否乾淨
- 查最新 Harness task-state 是否有 lifecycle 欄位

Live eval 的好處是能抓到 mock 抓不到的問題；壞處是較依賴外部狀態，應把不可控失敗標成 warning 或診斷訊息。

---

## 二、為什麼優先 deterministic checks

第一版 eval 不應急著做複雜的 LLM-as-Judge。

原因：

1. **系統規則多半可被明確檢查**  
   例如是否含 `ollama-article`、是否有 `git status --short`、是否有 Firestore summary。

2. **deterministic eval 便宜、快、穩定**  
   可以頻繁跑，不會因模型漂移讓結果忽好忽壞。

3. **真實回歸通常是工程契約破裂**  
   不是回答風格差一點，而是某條流程被漏掉。

4. **LLM-as-Judge 適合最後一層主觀品質**  
   例如摘要是否精煉、是否好讀、是否有洞察；但不適合拿來取代明確規則。

建議順序：

```txt
deterministic regression checks
→ component eval
→ end-to-end eval
→ live eval
→ subjective LLM-as-Judge
```

---

## 三、Agent Eval

Agent eval 要測的不是「模型有沒有想法」，而是它在固定工具與固定邊界內，是否做出正確決策。

### Tool Selection Eval

檢查 Agent 是否選對工具。

例子：

| 任務 | 預期工具 |
|---|---|
| 查 prior work / preference | memory_search / memory_get |
| 查 know 文件 | knowledge-mirror / RAG |
| 寫程式與跑測試 | read / edit / exec |
| 使用 Ollama | `ollama-presets` scripts |
| GitHub issue / PR | gh / github skill |

失敗型態：

- 應查 memory 卻憑印象回答
- 應用 skill 腳本卻直接 curl API
- 應先讀檔卻直接改檔

---

### Planner / Action Eval

檢查 Agent 的行動順序是否合理。

例子：

```txt
任務：整理影片成 know 文件
預期順序：
1. 取得 transcript / source
2. 整理草稿
3. 等使用者確認
4. 寫入 know
5. git diff
6. commit
7. push
8. 回報 commit hash
```

常見錯誤：

- 還沒確認就直接寫入 know
- 寫入後忘記 commit / push
- 沒有驗證檔案存在或 repo 狀態
- 最後只說「我會做」，但沒有真的做

---

### Boundary Eval

檢查 Agent 是否守住邊界。

例子：

- 對外發送訊息前是否確認
- 刪除遠端資料前是否確認
- 不把私人 memory 泄漏到群組
- 不把暫時 runtime 檔誤 commit
- 不在沒有依據時聲稱已驗證

Agent 越自主，boundary eval 越重要。

---

## 四、RAG Eval

RAG eval 不應只看最終答案，也要拆成 retrieval 與 generation 兩段。

### Retrieval Eval

核心問題：

> 使用者問這個問題時，系統有沒有找回該找的資料？

可測指標：

- Recall@K：正確文件是否出現在 top K
- Precision@K：top K 裡雜訊比例
- MRR：第一個正確結果排名
- source coverage：多文件問題是否涵蓋必要來源
- permission filter：不該看的文件是否被排除

測資格式可以是：

```json
{
  "query": "know 是什麼路徑？",
  "expected_sources": ["TOOLS.md"],
  "expected_terms": ["~/knowledge-mirror"]
}
```

---

### Context Packing Eval

很多 RAG 失敗不是 retrieval 沒找到，而是 context packing 時把重要段落丟掉。

要測：

- 關鍵段落是否進入 final context
- 是否重複塞入低價值片段
- 是否保留 source attribution
- 是否控制 token budget
- 是否把摘要、chunk、metadata 以正確順序組合

---

### Answer Grounding Eval

最終回答要測：

- 是否引用正確來源
- 是否避免 source 沒有支持的推論
- 是否承認查不到
- 是否把多來源衝突講清楚
- 是否沒有洩漏無關私人資訊

這一層可部分使用 LLM-as-Judge，但 grounding / citation coverage 仍應優先做 deterministic 檢查。

---

## 五、Harness Lifecycle Eval

Harness 是 run-level control plane，所以 eval 重點不是單一回答品質，而是任務治理是否完整。

### 必測項目

- task-state 是否建立
- task-state 是否有：
  - `task_id`
  - `title`
  - `goal`
  - `status`
  - `stage`
  - `steps_done`
  - `next_steps`
  - `artifacts`
  - `updated_at`
- workflow 是否有 verification report
- artifacts 是否符合 schema
- blocked 狀態是否說明 missing input
- completed 狀態是否有驗證證據

### 成功標準

一個任務完成時，不只要有產出，還要能回答：

```txt
做了什麼？
目前在哪個 stage？
驗證了什麼？
留下哪些 artifacts？
如果要接手，下一步在哪？
```

這就是 Harness lifecycle eval 的核心。

---

## 六、目前系統的第一版 Regression Eval

目前 OpenClaw workspace 已落地第一版 runner：

```txt
/home/hom/.openclaw/workspace/evals/runners/system_regression_eval.py
```

執行：

```bash
python3 evals/runners/system_regression_eval.py
```

產報告：

```bash
python3 evals/runners/system_regression_eval.py --write-report --no-fail-exit
```

第一版檢查範圍：

### know / knowledge-mirror

- README 是否記錄 commit + push 規則
- repo working tree 是否乾淨
- `TOOLS.md` 是否有 know alias

### Ollama

- `MEMORY.md` 是否記錄 `ollama-presets` 規則
- `ollama-article` / `ollama-tech` 是否存在
- workspace/bin 是否沒有直接打 Ollama HTTP API

### Claw Notify

- summary 是否透過 `ollama-article`
- FCM push body 是否使用 `doc.summary`
- sender 是否只推最新 device token
- frontend 是否使用 stable device id
- summary cleaner 是否移除 URL / pan.baidu / 提取碼等原文雜訊
- live Firestore notifications 是否有合格 summary
- live Firestore devices 是否存在且 sender policy 正確

### Harness

- runtime scaffold 是否存在
- task-state / verification report 是否通過 schema validation
- 最新 task-state 是否有 lifecycle fields
- verification reports 是否存在

### Preferences

- workspace memory 是否保留正體中文、預設、天生知道等穩定偏好

---

## 七、Eval Report 格式

建議 eval report 至少包含：

```json
{
  "status": "pass",
  "score": 18,
  "max_score": 18,
  "generated_at": "2026-05-07T...+08:00",
  "checks": [
    {
      "id": "notify.live_latest_summary_quality",
      "status": "pass",
      "description": "latest Firestore notifications have summary-like summaries",
      "details": "",
      "severity": "error",
      "metadata": {
        "inspected": ["..."]
      }
    }
  ]
}
```

Report 要能被人讀，也要能被 script 消費。

---

## 八、失敗等級設計

不是所有失敗都應該讓任務中斷。

建議分級：

| 等級 | 用途 | 是否 fail exit |
|---|---|---|
| error | 明確違反系統契約 | 是 |
| warning | 外部狀態不可控或需要人工確認 | 視情況 |
| info | 診斷資訊 | 否 |

例子：

- 直接打 Ollama API：error
- Firestore 暫時無法連線：warning
- eval report 數量偏少：info / warning

---

## 九、後續擴充方向

### 1. RAG retrieval eval dataset

建立一組固定 query / expected source：

```txt
query → expected file / expected section / expected facts
```

先測 recall，再測 answer grounding。

---

### 2. Agent planner eval

建立一組任務情境，檢查 Agent 是否產生合理 action sequence。

例如：

- 寫 know 文件
- 修程式並跑測試
- 查 prior decision
- 發通知前需要確認

---

### 3. Harness smoke run eval

建立最小任務，要求 Harness 完成：

```txt
start task
→ update task-state
→ produce artifact
→ validate artifact
→ write verification report
→ close task
```

---

### 4. LLM-as-Judge 補充主觀品質

只放在 deterministic eval 之後，用來評估：

- 摘要是否好讀
- 結案報告是否清楚
- 文章筆記是否有結構
- 回答是否自然、不囉嗦

但不能用它取代明確規則。

---

## 十、一句話總結

Agent / RAG / Harness 的 Eval，第一步不是追求高深模型評分，而是把真實工作中已經踩過的規則、邊界與 workflow 契約變成可重跑的防回歸檢查。

先讓系統不要重犯錯，再談更細的主觀品質評估。
