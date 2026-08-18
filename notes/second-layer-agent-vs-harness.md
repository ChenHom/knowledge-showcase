# 第二層 Agent 與 Harness 系統的差異：Runtime、任務治理與自主決策的分工

#agent #harness #agentic-workflow #system-design #workflow #verification #runtime #event-log #tool-execution

## 這是什麼

這份文件說明「第二層 Agent」與「Agent Harness」的差異與分工，並補上完整 Harness 應包含的 Runtime 層。

早期版本把 Harness 主要描述成「管理整個任務生命週期的外層執行框架」。這個描述對 `agent-system-foundation` 目前實作的 **Run Governance** 是成立的，但若拿來描述完整 Agent Harness，範圍太窄。

更完整的定義應該是：

```text
Agent Harness
= 支撐 Agent 執行、限制副作用、保存執行事實、治理任務生命週期的整套 runtime / control system

第二層 Agent
= 在既定能力與邊界內，自主決定下一步行動的 decision loop
```

因此 Harness 可以再拆成兩層：

```text
Agent Harness
├── Harness Runtime
│   ├── Session / Event Log
│   ├── Context Assembly
│   ├── LLM Adapter
│   ├── Agent Interface
│   ├── Agent Loop
│   ├── Tool Registry
│   ├── Tool Execution Pipeline
│   ├── Permission / Approval
│   └── Sandbox / Guard
│
└── Run Governance
    ├── Workflow
    ├── Task State
    ├── Verification
    ├── Audit
    ├── Repair
    ├── Human Handoff
    ├── Close-out
    └── Artifact Governance
```

第二層 Agent 則是 Harness Runtime 裡的一種執行策略：

```text
read state
→ choose next action
→ request tool
→ evaluate result
→ continue / ask / finish
```

兩者不是競爭關係，也不是簡單的「Harness 在外、Agent 在內」就能完整描述；更精準的理解是：

> Harness 提供 Agent 可以安全、可恢復、可觀測地運作的執行環境與治理機制；Agent Decision Policy 負責在這個環境裡決定下一步。

---

## 核心結論

原本的說法：

```text
Harness = run-level control plane
第二層 Agent = task-level decision loop
```

需要修正成：

```text
Agent Harness = runtime + execution control + run governance
Run Governance = run-level control plane
第二層 Agent = task-level decision loop
```

所以：

```text
Harness Runtime 管「怎麼執行」
Run Governance 管「怎麼確保可靠完成」
Agent Decision Policy 管「下一步做什麼」
```

Harness 不一定需要 LLM 自主決策；它可以管理固定流程、人工流程、腳本任務、subagent 任務。第二層 Agent 則一定有「Agent 自己選下一步」這件事。

---

## 一、核心差異表

| 面向 | Agent Harness | Run Governance | 第二層 Agent |
|---|---|---|---|
| 主要目的 | 提供 Agent 可執行、可限制、可恢復的 runtime | 讓任務可追蹤、可驗證、可收尾 | 讓 Agent 在固定能力內自主決定步驟 |
| 管理範圍 | session、model、agent loop、tools、policy、sandbox、runtime state | 整個 run / task lifecycle | 單一業務任務或工作流程內部 |
| 重點 | event log、context、LLM adapter、tool execution pipeline、capability seams | bootstrap、state、contracts、verification、close-out、audit、repair | choose action、tool use、local state、result evaluation |
| 自主性 | 不要求一定自主 | 不要求一定自主 | 明確要求 Agent 自己決策 |
| 產物 | session events、tool call/result、runtime trace | task-state、verification report、audit、handoff、memory target | action sequence、final response、decision trace |
| 失敗處理 | cancellation、timeout、retry、deny、resume、sandbox failure | repair、blocked handoff、re-run、close-out review | ask user、choose alternative、stop / finish |
| 適用層級 | Agent 執行基礎設施 | 任務治理層 | 任務內決策策略 |

---

## 二、完整的 Harness 是什麼

完整 Harness 不是只有任務治理。

它至少要回答五類問題：

```text
1. Model 看到了什麼？
2. Agent 現在處於什麼執行狀態？
3. Model 要求的 tool call 能不能真的執行？
4. 執行後留下了什麼不可否認的事實？
5. 整個任務是否被正確驗證、收尾與治理？
```

因此 Harness 的完整責任可以分成：

### Harness Runtime

```text
Session / Event Log
Context Assembly
LLM Adapter
Agent Interface
Agent Loop
Tool Registry
Tool Execution Pipeline
Permission / Approval
Sandbox / Guard
```

它解決：

> Agent 到底如何被執行。

### Run Governance

```text
bootstrap
workflow selection
task-state
contracts
verification
verify-revise loop
close-out
audit
human handoff
repair orchestration
memory target recommendation
```

它解決：

> 任務到底有沒有可靠完成。

目前 `agent-system-foundation` 最成熟的其實是第二塊，也就是 **Run Governance Harness**。

---

## 三、Session Event Log：執行事實的 Source of Truth

如果 Harness 同時維護：

```text
messages[]
task-state
action trace
tool results
verification report
audit report
```

卻沒有定義哪一份才是真相，時間一久一定會 drift。

比較穩定的做法是：

```text
Append-only Session Event Log
        │
        ├── derive model history
        ├── project task-state
        ├── build action trace
        ├── build observability data
        └── support resume / replay / audit
```

最小事件可以是：

```text
turn/start
step/start
user/message
assistant/message
tool/call
tool/result
step/end
turn/end
```

核心原則：

> Event Log 記錄 facts；其他 state / report 是 projection 或 derived artifact。

因此：

```text
Event Log = 執行事實
Task State = 任務狀態投影
Verification Report = 從結果與證據導出的驗證產物
Audit Report = 對 lifecycle / artifacts 的治理判定
```

這樣才能真正支援：

```text
resume
replay
fork
crash recovery
UI rendering
telemetry
audit
```

而不是靠聊天紀錄或多份 JSON 猜現在發生到哪裡。

---

## 四、第二層 Agent 是什麼

第二層 Agent 關心的是某個任務內部怎麼跑。

定義：

```text
能力固定
權限邊界固定
任務邊界固定
但步驟由 Agent 自己決定
```

例如客服改地址 Agent：

```text
可用工具：
- get_order
- search_policy
- update_address
- send_email

Agent 自己決定：
- 先查訂單？
- 先問使用者？
- 要不要查政策？
- 是否提出具有副作用的 tool request？
```

它關心的是：

> 在一組固定能力和規則內，Agent 如何自主決定下一步？

但要注意：

```text
Agent 決定「想做什麼」
≠
Agent 有權直接「真的做」
```

實際副作用仍必須經過 Harness 的 execution control。

---

## 五、Tool Execution Pipeline：Model 想做，不代表可以直接做

簡單 Agent 常寫成：

```text
model tool call
→ tool.execute(args)
→ result
```

這會把 permission、approval、timeout、sandbox、filesystem guard 全塞進各工具自己處理，最後很難治理。

Harness 應該統一成：

```text
tool/call
    ↓
pre-execute hooks
    ↓
permission / policy
    ↓
approval（需要時）
    ↓
guards
    ↓
execution wrapper
  timeout / retry / metrics
    ↓
tool.execute()
    ↓
filesystem / side-effect guard
    ↓
post-execute hooks
    ↓
result normalization
    ↓
tool/result
```

因此邊界應該是：

```text
Agent
  ↓ request
Tool Runtime
  ↓
Policy
  ↓
Approval
  ↓
Guard / Sandbox
  ↓
Execute
  ↓
Observe / Log
```

任何具有副作用的能力都不應讓 Agent 直接繞過這條管線。

這個設計的價值是把：

```text
能力本身
```

和：

```text
能力能不能在這一次被使用
```

拆開。

---

## 六、Agent Loop 與 Harness Runtime 的關係

Agent Loop 不是整個 Harness；但 Agent Loop 可以是 Harness Runtime 的可替換元件。

最小 loop：

```text
claim input
→ assemble context + tool schemas
→ call model
→ append assistant output
→ execute requested tools through pipeline
→ append tool results
→ determine continuation
→ next step / turn end
```

Harness 不應假設一定是：

```text
ReAct
某種 Planner
某一家 LLM
某個固定 agent implementation
```

比較好的結構是：

```text
              Agent Interface
                    ▲
        ┌───────────┼───────────┐
        │           │           │
 Default Loop   Workflow    External Agent
        │                       │
     Subagent                delegated runtime
```

其他元件依賴的是 Agent / Runner contract，而不是某個具體 loop implementation。

---

## 七、Capability Seam：能力與 Provider 解耦

Tool 不應直接綁死底層執行世界。

更可替換的結構是：

```text
Consumer / Tool
      ↓
Capability Interface
      ↓
Provider
```

例如：

```text
filesystem tool
      ↓
Filesystem Capability
      ↓
Local Provider / Remote Sandbox Provider
```

```text
subagent tool
      ↓
Subagent Capability
      ↓
Local Agent / External Agent / Other Runtime
```

```text
LLM call
      ↓
LLM Adapter
      ↓
Provider A / Provider B
```

這種 seam 的重點不是抽象化本身，而是：

> 換 Provider 時，不需要重寫 Agent Loop、Tool Schema、Governance 與上層流程。

---

## 八、Run-level Control Plane vs Task-level Decision Loop

### Run Governance：Run-level Control Plane

Run Governance 的層級比較高。

```text
start run
→ choose workflow
→ create task-state
→ prepare contracts
→ execute work
→ verify output
→ revise if needed
→ audit
→ close out
```

它的核心問題是：

```text
這個任務是否被正確治理？
```

### 第二層 Agent：Task-level Decision Loop

第二層 Agent 是任務內部的決策引擎：

```text
read state
→ choose next action
→ request tool
→ receive authoritative result
→ update local decision state
→ continue / ask user / finish
```

它的核心問題是：

```text
下一步該做什麼？
```

### Harness Runtime：Execution Runtime

Harness Runtime 則負責：

```text
如何把這次 decision 變成真正、受控、可追蹤的執行。
```

三者不能混為同一件事。

---

## 九、兩者如何搭配

完整關係應該是：

```text
Agent Harness
├── Harness Runtime
│   ├── Session Event Log
│   ├── Model / Context
│   ├── Agent Loop
│   └── Tool Execution Pipeline
│
└── Run Governance
    └── Task: 建立客服改地址 Agent
        ├── bootstrap task-state
        ├── 定義 tool contracts
        ├── 執行第二層 Agent decision loop
        ├── 收集 action trace
        ├── 跑 component eval
        ├── 跑 end-to-end eval
        ├── 產生 verification report
        └── close-out / audit
```

第二層 Agent 的內部事件可以投影成治理 artifact：

| Runtime / Agent 事件 | Governance 可收斂成 |
|---|---|
| tool/call + tool/result | action trace / tool contract result |
| state transition | task-state projection |
| approval request/result | human handoff / approval artifact |
| verifier result | verification report |
| repeated failure | blocked state / repair request |
| final response | deliverable |

---

## 十、具體例子：客服改地址 Agent

### 任務：做客服改地址 Agent

Run Governance 會管：

```text
- 這個任務叫什麼？
- 成功標準是什麼？
- 需要哪些 tool contract？
- 要跑哪些測試？
- 有沒有 verification report？
- 失敗要不要 repair？
- 完成後要不要寫入 know / memory / RAG？
```

第二層 Agent 會判斷：

```text
- 使用者沒給 order_id，要問使用者
- 有 order_id，要查 get_order
- 未出貨，要查 search_policy
- 政策允許，提出 update_address request
- 完成後提出 draft / send email request
```

Harness Runtime 則負責：

```text
- 記錄 user/message
- 呼叫 model
- 記錄 tool/call
- 判斷 update_address 是否需要 approval
- approval 不成立時不得執行
- 執行工具並記錄 tool/result
- 失敗時保留可恢復狀態
```

### Run Governance 流程

```text
1. 建立 task-state：customer-address-agent
2. 定義成功標準：
   - 不更新已出貨訂單
   - 缺 order_id 時會詢問
   - 高風險操作需要 approval
   - end-to-end case 通過
3. 建立 tool contracts
4. 執行 Agent loop
5. 收集 / 投影 action trace
6. 跑 eval
7. 建 verification report
8. close-out
```

### 第二層 Agent 決策流程

```text
1. 收到使用者：我要改 A127 地址到建國南路
2. request get_order(A127)
3. 收到未出貨結果
4. request search_policy(address_change)
5. 判斷允許修改
6. request update_address
7. Harness 執行 approval / policy / guard
8. 通過後真正執行 update_address
9. request send_email
10. Harness 再做副作用控制
11. 回覆使用者完成
```

---

## 十一、什麼情況只需要 Run Governance

如果任務本身步驟固定，不需要 Agent 自主選擇下一步，只需要外層追蹤和驗證，就不需要第二層 Agent。

例子：

```text
- 寫入 know 文件後 commit & push
- 跑固定測試流程
- 產生固定報表
- 執行固定部署 checklist
- 整理文件、驗證格式、close-out
```

這些任務可能需要：

```text
task-state
verification report
audit
memory target
```

底下仍可跑在 Harness Runtime 上，但不需要自主 Agent decision loop。

---

## 十二、什麼情況需要第二層 Agent

如果任務內部有分支、例外、工具選擇，而且工具邊界明確，就適合第二層 Agent。

例子：

```text
- 客服 Agent
- IT helpdesk Agent
- 內部資料查詢 Agent
- 訂單處理 Agent
- RAG + tool action 的業務助理
```

它需要判斷：

```text
資訊夠不夠？
要不要問使用者？
要先查哪個資料源？
要不要使用 RAG？
要不要呼叫外部工具？
是否提出需要 approval 的 action？
```

---

## 十三、什麼情況 Runtime、Governance、第二層 Agent 都需要

當任務同時具備：

```text
1. 有真正工具或外部副作用
2. 多步驟、需要驗證、需要可恢復
3. 任務內部有 Agent 自主決策
```

三層都需要。

例如：

```text
- production 客服 Agent
- RAG + tool-calling 的企業知識助理
- 自動化資料修復 Agent
- 半自動交易研究 Agent
- 多工具 DevOps Agent
```

---

## 十四、與 workspace agent-system-foundation 的對照

目前 workspace 裡的 `agent-system-foundation/` 最成熟的是 **Run Governance**。

已具備：

```text
bootstrap
workflow selection
task-state
contracts
verification
verify-revise loop
close-out
audit
human handoff
repair orchestration
memory target recommendation
```

所以它不是某個業務 Agent，而是：

```text
多步任務的治理框架
```

它可以管理：

```text
固定腳本任務
文件整理任務
RAG 寫入任務
coding 任務
subagent 任務
第二層 Agent 任務
```

接下來真正需要補齊的不是更多治理 artifact，而是 Harness Runtime 底層：

```text
append-only session event model
LLM adapter seam
Agent interface / loop separation
centralized tool execution pipeline
capability / provider abstraction
sandbox execution boundary
runtime recovery invariants
```

---

## 十五、實作建議

### 資料夾概念不要再切成「Harness vs Agent Runtime」

原本：

```text
/harness
  task-state
  verification report
  audit
  close-out

/agent-runtime
  planner
  tool registry
  tool executor
  state machine
  verifier
```

實體資料夾可以暫時保留，但概念應改成：

```text
/agent-harness
  /runtime
    session
    context
    llm-adapter
    agent-interface
    agent-loop
    tool-registry
    tool-runtime
    permission
    sandbox

  /governance
    task-state
    verification
    audit
    repair
    handoff
    close-out
```

`planner / decision policy` 則是 Runtime 中可替換的 Agent implementation，不應跟整個 Harness 對立。

### Harness 應看到什麼

Runtime 層應保存真實事件：

```text
user/message
assistant/message
tool/call
tool/result
approval result
turn / step boundary
runtime error / cancellation
```

Governance 層則使用摘要化 artifact：

```text
action trace
tool contract result
state transition summary
verification result
approval record
final deliverable
```

### 第二層 Agent 應看到什麼

Agent 需要的是任務內部前後文：

```text
current state
available tools
tool descriptions
business constraints
user request
recent authoritative tool results
approval status（若與下一步決策相關）
```

### 邊界原則

```text
Harness Runtime 管「執行如何發生」
Run Governance 管「任務是否可靠完成」
Agent 管「下一步該做什麼」
Tool Runtime 管「這次要求能不能真的做」
Verifier 管「做完是否正確」
Human Approval 管「高風險副作用是否允許」
Event Log 管「實際發生了什麼」
```

---

## 十六、常見混淆

### 混淆 1：有 Harness 就等於有 Agent

不對。Harness 可以管理 deterministic workflow，不需要自主 Agent。

### 混淆 2：有 Agent loop 就等於有完整 Harness

也不對。只有 loop，但沒有 durable session、guarded tool execution、recovery、governance，就只是能跑，不代表可治理。

### 混淆 3：Run Governance 就等於完整 Harness

不完整。Run Governance 是 Harness 的重要一層，但還需要 Runtime 才能回答「執行事實從哪裡來、工具如何受控、如何 resume/replay」。

### 混淆 4：第二層 Agent 可以取代 Harness

不建議。Agent 解決的是任務內決策彈性，不負責整個 runtime 與 run governance。

### 混淆 5：Harness 應該控制 Agent 每一步

不一定。如果 Harness 把 decision sequence 全寫死，Agent 就退回 deterministic workflow。Harness 應控制邊界、執行規則與驗證；Agent 在邊界內自主決策。

### 混淆 6：Tool call 就等於已執行

不對。

```text
tool/call = Agent / Model 的執行意圖
tool/result = Harness Runtime 記錄的執行結果
```

中間必須經過 policy / approval / guard / execution。

---

## 十七、總結

新版分工：

```text
Agent Harness
├── Harness Runtime
│   └── session → context → model → agent loop → tool pipeline → event log
│
└── Run Governance
    └── bootstrap → execution governance → verification → repair → close-out → audit

第二層 Agent
└── state → choose action → request tool → evaluate result → continue / finish
```

最重要的一句話：

> **Agent Harness 是控制 Agent 如何取得前後文、如何呼叫模型、如何執行工具、如何限制副作用、如何保存執行事實，以及如何讓任務可恢復、可驗證、可治理的執行系統；第二層 Agent 則是在這個系統內負責下一步決策的工作單元。**

---

## 參考架構

- DeepSeek Harness Architecture：`deepseek-ai/deepseek-harness/docs/architecture.md`
- DeepSeek Harness Tool Execution Pipeline：`deepseek-ai/deepseek-harness/docs/tool-execution-pipeline.md`

本文件吸收的不是特定實作或 Cordis plugin 機制，而是三個可泛化原則：

```text
1. durable event log 作為執行事實來源
2. tool execution 必須集中經過 policy / guard / sandbox pipeline
3. agent loop、model、capability provider 應透過 interface / adapter seam 可替換
```
