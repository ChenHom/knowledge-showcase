# 第二層 Agent 與 Harness 系統的差異：任務治理與自主決策的分工

#agent #harness #agentic-workflow #system-design #workflow #verification

## 這是什麼

這份文件說明「第二層 Agent」與「Harness 系統」的差異與分工。

簡單講：

```text
Harness = 管理整個任務生命週期的外層執行框架
第二層 Agent = 某個任務內部的 agent 自主決策模式
```

比喻：

```text
Harness 是片場導演 / 製片流程
第二層 Agent 是其中一個演員怎麼在規則內自由發揮
```

兩者不是競爭關係，而是上下層關係：

```text
Harness 可以包住第二層 Agent，讓 Agent 的每次工具呼叫、狀態轉移、驗證結果都成為可追蹤、可審計的 run artifact。
```

## 核心結論

**Harness 解決「任務治理」；第二層 Agent 解決「工具內自主行動」。**

更精準地說：

```text
Harness = run-level control plane
第二層 Agent = task-level decision loop
```

Harness 不一定需要 LLM 自主決策；它可以管理固定流程、人工流程、腳本任務、subagent 任務。  
第二層 Agent 則一定有「Agent 自己選下一步」這件事。

## 一、核心差異表

| 面向 | Harness 系統 | 第二層 Agent |
|---|---|---|
| 主要目的 | 讓任務可追蹤、可驗證、可恢復、可收尾 | 讓 Agent 在固定工具內自主決定步驟 |
| 管理範圍 | 整個 run / task lifecycle | 單一業務任務或工作流程內部 |
| 重點 | bootstrap、state、contracts、verification、close-out、audit | tool registry、agent loop、guardrails、state、approval |
| 自主性 | 不一定要求 Agent 自主；可以管理人工流程、固定腳本、subagent | 明確要求「工具固定，但步驟由 Agent 決定」 |
| 產物 | task-state、verification report、audit、handoff、memory target | tool calls、state transitions、final response、action trace |
| 失敗處理 | repair、blocked handoff、re-run、close-out review | retry、ask user、approval、tool error handling |
| 適用層級 | 外層治理 / 任務作業系統 | 內層 agent workflow pattern |

## 二、Harness 是什麼

Harness 是任務外層的治理框架。

它關心的是：

> 這個任務有沒有被正確開始、執行、驗證、修正、收尾、沉澱？

典型 Harness 會管理：

```text
任務開始前：bootstrap / 判斷 workflow
任務進行中：維護 task-state / contracts / orchestration
任務完成前：verify-revise loop
任務完成後：close-out / audit / memory target 判斷
卡住時：handoff / repair / blocked state
```

Harness 不一定管「Agent 下一步要查訂單還是查政策」。

它管的是：

```text
這個任務是否有狀態
是否有成功標準
是否有驗證報告
是否可以恢復
是否有可審計的紀錄
是否有 close-out
```

## 三、第二層 Agent 是什麼

第二層 Agent 關心的是某個任務內部怎麼跑。

定義：

```text
工具固定
權限固定
任務邊界固定
但步驟由 Agent 自己決定
```

例如客服改地址 Agent：

```text
工具固定：
- get_order
- search_policy
- update_address
- send_email

Agent 自己決定：
- 先查訂單？
- 先問使用者？
- 要不要查政策？
- 要不要送 approval？
```

它關心的是：

> 在一組固定工具和規則內，Agent 如何自主決定下一步？

## 四、Run-level Control Plane vs Task-level Decision Loop

### Harness：Run-level Control Plane

Harness 的層級比較高。

它像任務控制塔：

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

第二層 Agent 的層級比較低。

它像任務內部的決策引擎：

```text
read state
→ choose next action
→ validate action
→ execute tool
→ update state
→ verify result
→ continue / ask user / finish
```

它的核心問題是：

```text
下一步該使用哪個工具？
```

## 五、兩者如何搭配

最合理的關係是：

```text
Harness
└── Task: 建立客服改地址 Agent
    ├── bootstrap task-state
    ├── 定義 tool contracts
    ├── 執行第二層 Agent loop
    ├── 收集 action trace
    ├── 跑 component eval
    ├── 跑 end-to-end eval
    ├── 產生 verification report
    └── close-out / audit
```

也就是：

```text
Harness 包住第二層 Agent
第二層 Agent 是 Harness 裡的一種可執行工作單元
```

Harness 可以把第二層 Agent 的內部事件收斂成外層 artifact：

| 第二層 Agent 內部事件 | Harness 可收斂成 |
|---|---|
| tool call | action trace / tool contract result |
| state transition | task-state update |
| approval request | human handoff artifact |
| verifier result | verification report |
| repeated failure | blocked state / repair request |
| final response | deliverable |

## 六、具體例子：客服改地址 Agent

### 任務：做客服改地址 Agent

Harness 會管：

```text
- 這個任務叫什麼？
- 成功標準是什麼？
- 需要哪些 tool contract？
- 要跑哪些測試？
- 有沒有 verification report？
- 失敗要不要 repair？
- 完成後要不要寫入 know / memory / RAG？
```

第二層 Agent 會管：

```text
- 使用者沒給 order_id，要問使用者
- 有 order_id，要查 get_order
- 未出貨，要查 search_policy
- 政策允許，要 request_approval
- approval 後才能 update_address
- 完成後 draft / send email
```

### Harness 流程

```text
1. 建立 task-state：customer-address-agent
2. 定義成功標準：
   - 不更新已出貨訂單
   - 缺 order_id 時會詢問
   - 高風險操作需要 approval
   - end-to-end case 通過
3. 建立 tool contracts：
   - get_order
   - search_policy
   - update_address
   - send_email
4. 執行 Agent loop
5. 收集 action trace
6. 跑 eval
7. 建 verification report
8. close-out
```

### 第二層 Agent 流程

```text
1. 收到使用者：我要改 A127 地址到建國南路
2. 呼叫 get_order(A127)
3. 發現未出貨
4. 呼叫 search_policy(address_change)
5. 判斷允許修改
6. 送出 update_address approval
7. approval 通過後呼叫 update_address
8. 草擬通知 email
9. 送出 send_email approval
10. 寄出 email
11. 回覆使用者完成
```

## 七、什麼情況只需要 Harness

如果任務本身步驟固定，不需要 Agent 自主選擇下一步，只需要外層追蹤和驗證，那只需要 Harness。

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

但不需要 Agent 自己決定工具順序。

## 八、什麼情況需要第二層 Agent

如果任務內部有分支、例外、工具選擇，而且工具邊界明確，就適合第二層 Agent。

例子：

```text
- 客服 Agent
- IT helpdesk Agent
- 內部資料查詢 Agent
- 訂單處理 Agent
- RAG + tool action 的業務助理
```

這些任務需要 Agent 自己判斷：

```text
資訊夠不夠？
要不要問使用者？
要先查哪個資料源？
要不要使用 RAG？
要不要呼叫外部工具？
是否需要 approval？
```

## 九、什麼情況兩者都需要

當任務同時具備：

```text
1. 多步驟、需要驗證、需要可恢復
2. 任務內部又有 Agent 自主決策
```

就應該兩者都用。

例子：

```text
- 建立 production 客服 Agent
- 做一套 RAG + tool-calling 的企業知識助理
- 自動化資料修復 Agent
- 半自動交易研究 Agent
- 多工具 DevOps Agent
```

此時：

```text
Harness 負責治理整個 run
第二層 Agent 負責 run 裡的自主決策工作單元
```

## 十、與 workspace agent-system-foundation 的對照

目前 workspace 裡的 `agent-system-foundation/` 比較接近 Harness。

它已經具備的方向包括：

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

所以它的定位不是「某個業務 Agent」，而是：

```text
多步任務的外層治理框架
```

它可以管理很多種任務：

```text
固定腳本任務
文件整理任務
RAG 寫入任務
coding 任務
subagent 任務
第二層 Agent 任務
```

其中第二層 Agent 只是它可承載的一種任務形態。

## 十一、實作建議

### 如果要做第二層 Agent，不要把 Harness 和 Agent loop 混在一起

建議分層：

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

Harness 不應該直接塞滿業務工具細節。  
Agent loop 也不應該自己處理整個任務生命週期治理。

### Harness 應該看到什麼

Harness 需要的是摘要化 artifact：

```text
action trace
tool contract result
state transition summary
verification result
approval record
final deliverable
```

### 第二層 Agent 應該看到什麼

第二層 Agent 需要的是任務內部上下文：

```text
current state
available tools
tool descriptions
business constraints
user request
recent tool results
approval status
```

### 邊界原則

```text
Harness 管「任務是否可靠完成」
Agent 管「下一步該做什麼」
Tool Executor 管「能不能真的做」
Verifier 管「做完是否正確」
Human Approval 管「高風險副作用是否允許」
```

## 十二、常見混淆

### 混淆 1：有 Harness 就等於有 Agent

不對。

Harness 可以管理完全 deterministic 的流程，不需要 Agent 自主決策。

### 混淆 2：有 Agent loop 就等於有 Harness

也不對。

Agent loop 可能能完成任務，但如果沒有 task-state、verification、audit、close-out，就很難治理。

### 混淆 3：第二層 Agent 可以取代 Harness

不建議。

第二層 Agent 解決的是任務內的決策彈性，不負責整個 run 的治理閉環。

### 混淆 4：Harness 應該控制 Agent 每一步

也不一定。

如果 Harness 把每一步都寫死，Agent 就退回第一層 workflow。比較好的做法是：Harness 設邊界與驗證，Agent 在邊界內自主決策。

## 十三、總結

兩者分工如下：

```text
Harness：
任務生命週期治理
bootstrap → execution → verification → close-out → audit

第二層 Agent：
任務內部自主決策
state → choose action → validate → execute tool → update state → finish
```

最重要的一句話：

> Harness 是讓任務可靠落地的外層控制系統；第二層 Agent 是在固定工具與規則內做彈性決策的內層工作單元。
