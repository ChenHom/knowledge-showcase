# 第二層 Agent 實作設計：固定工具下的受控自主工作流

#agent #agentic-workflow #ai-engineering #tool-calling #system-design #eval

## 這是什麼

這份文件說明如何實作「第二層 Agent 系統」。

所謂第二層 Agent，指的是：

```text
工具固定
權限固定
任務邊界固定
但步驟由 Agent 自己決定
```

也就是 production 場景中最常見、也相對安全的 agentic workflow 形態。

它介於兩者之間：

```text
第一層：Hardcoded Steps
→ 步驟全部寫死，LLM 只做局部判斷

第二層：Hardcoded Tools + Agent decides steps
→ 工具固定，但 Agent 自己決定下一步

第三層：Fully Autonomous
→ Agent 自己找工具、寫工具、決定完整策略
```

第二層的核心價值是：

> 人類控制工具邊界與安全規則，Agent 負責在邊界內規劃步驟、呼叫工具、處理例外。

## 核心結論

第二層 Agent 不應該是「完全放飛的 AI」，而是 **受控自主**。

最穩定的架構是：

```text
User Request
→ Agent Controller / Planner
→ Tool Registry
→ Tool Execution Layer
→ State Store
→ Guardrails / Policy Checks
→ Verifier
→ Response
```

實作原則：

- 工具由工程師定義，不讓 Agent 任意創工具。
- Agent 可以決定工具呼叫順序。
- 每個 tool 都要有 schema、驗證、權限控制。
- 高風險行為要 human approval。
- 系統要記錄每一步 action trace，而不是依賴不可審計的隱含推理。
- 必須有 component eval 與 end-to-end eval。

一句話：

> 第二層 Agent 是「固定工具箱 + 可控自主規劃 + 嚴格執行護欄」。

## 一、如何判斷 Agent 系統是哪一層

可以用三個問題判斷：

```text
1. 步驟是誰決定的？
2. 工具是誰決定的？
3. 高風險行為是否受控？
```

### 第一層：Hardcoded Steps

定義：

```text
步驟固定
工具固定
LLM 只處理局部任務
```

例子：客服改地址流程。

```text
1. extract_intent()
2. extract_order_id()
3. get_order()
4. check_policy()
5. update_address()
6. generate_reply()
```

每一步都由程式寫死。

特徵：

- 安全
- 好測試
- 好 debug
- 例外處理僵硬
- 不像真正 agent，比較像 LLM-enhanced workflow

適合：

- 高風險流程
- 金流
- 醫療
- 法務
- 固定 SOP

### 第二層：Hardcoded Tools + Agent Decides Steps

定義：

```text
工具固定
任務目標固定
安全邊界固定
Agent 自己決定下一步
```

例子：給 Agent 一組工具：

```text
get_order()
search_policy()
update_address()
send_email()
ask_user()
```

使用者說：

```text
我要改 A127 訂單地址，搬到建國南路。
```

Agent 自己決定：

```text
1. 先呼叫 get_order(A127)
2. 發現未出貨
3. 呼叫 search_policy(address_change)
4. 判斷可修改
5. 呼叫 update_address()
6. 呼叫 send_email()
7. 回覆使用者
```

如果使用者沒給 order ID，Agent 可以先問：

```text
請提供訂單編號。
```

特徵：

- 比第一層有彈性
- 比第三層安全
- 是多數 production agent 的合理起點
- 需要較完整的 tool guardrails 與 eval

### 第三層：Fully Autonomous

定義：

```text
工具不固定
步驟不固定
Agent 甚至可以自己寫 code、找 API、部署流程
```

例子：

```text
幫我做一套退貨自動化系統。
你可以搜尋文件、寫程式、改資料庫、部署服務。
```

特徵：

- 能力強
- 風險高
- 難測試
- 難預測
- 不適合直接接正式資料與外部副作用

適合：

- sandbox
- 研究
- 內部 coding assistant
- 有人類審查的任務

## 二、第二層 Agent 的系統架構

整體架構：

```text
                  ┌────────────────────┐
                  │    User Request     │
                  └─────────┬──────────┘
                            │
                            ▼
                  ┌────────────────────┐
                  │ Agent Controller    │
                  │ goal + state + LLM  │
                  └─────────┬──────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
     ┌─────────────────┐         ┌─────────────────┐
     │ Tool Registry   │         │ State Store      │
     │ schemas + docs  │         │ task/session     │
     └────────┬────────┘         └─────────────────┘
              │
              ▼
     ┌─────────────────┐
     │ Tool Executor   │
     │ validate + run  │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ Guardrails      │
     │ policy + auth   │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ Verifier        │
     │ check result    │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ Final Response   │
     └─────────────────┘
```

### 1. Agent Controller

負責：

- 接收使用者請求
- 讀取目前狀態
- 決定下一步
- 選擇要呼叫哪個 tool
- 判斷是否需要問使用者
- 判斷是否完成任務

Controller 不直接做事，只能透過 tool。

### 2. Tool Registry

保存 Agent 可用工具。

每個工具應包含：

```json
{
  "name": "get_order",
  "description": "查詢訂單狀態與基本資訊",
  "input_schema": {
    "order_id": "string"
  },
  "side_effect": false,
  "risk_level": "low",
  "requires_approval": false
}
```

工具 metadata 很重要，因為 Agent 要靠 description 判斷何時使用。

### 3. Tool Executor

負責真正執行工具。

它不能盲目信任 Agent 傳來的參數。

必須做：

- schema validation
- type validation
- permission check
- rate limit
- idempotency check
- timeout
- retry policy
- error normalization
- audit log

### 4. State Store

記錄任務狀態。

例如：

```json
{
  "task_id": "task_123",
  "user_intent": "change_address",
  "order_id": "A127",
  "new_address": "建國南路...",
  "order_status": "not_shipped",
  "policy_checked": true,
  "address_updated": false,
  "email_sent": false,
  "current_state": "READY_TO_UPDATE"
}
```

State Store 的價值：

- 可恢復
- 可 debug
- 可追蹤
- 可 eval
- 可避免 Agent 忘記前面做過什麼

### 5. Guardrails

Guardrails 是第二層 Agent 的安全核心。

#### 權限護欄

```text
這個使用者能不能查這筆資料？
這個 agent 能不能改這張訂單？
```

#### 業務規則護欄

```text
已出貨訂單不得改地址
超過 7 天不得退款
退款金額不可超過付款金額
```

#### 副作用護欄

```text
寄 email 前是否需要確認？
退款前是否需要 approval？
刪除資料是否禁止？
```

#### 輸出護欄

```text
不得承諾政策外事項
不得捏造處理時間
不得輸出內部錯誤細節
```

### 6. Verifier

Verifier 負責檢查 Agent 的結果是否合理。

例如：

```text
Agent 說地址已更新
→ verifier 查 DB 確認地址真的更新

Agent 說訂單不可退款
→ verifier 檢查政策來源是否支持

Agent 要寄信
→ verifier 檢查 email 內容是否包含必要資訊
```

Verifier 可以是：

- deterministic code
- rule engine
- another LLM
- human approval

## 三、第二層 Agent 的實作流程

### Step 1：定義任務邊界

先不要寫 prompt，先定義 agent 可以處理什麼。

例如客服 agent：

```text
可以處理：
- 查訂單
- 改地址
- 查退款政策
- 草擬客服回覆

不能處理：
- 直接退款
- 刪除訂單
- 修改付款資料
- 承諾政策外補償
```

這一步決定安全邊界。

### Step 2：定義工具

工具要小而清楚。

不要做一個巨大工具：

```text
handle_customer_request()
```

而是拆成：

```text
extract_order_info()
get_order()
search_policy()
update_address()
draft_email()
send_email()
ask_user()
```

工具越清楚，Agent 越容易正確組合。

### Step 3：替每個工具標註風險

| Tool | Side Effect | Risk | Approval |
|---|---:|---:|---:|
| get_order | no | low | no |
| search_policy | no | low | no |
| draft_email | no | low | no |
| update_address | yes | medium | maybe |
| send_email | yes | medium | yes |
| refund_payment | yes | high | yes |
| delete_order | yes | critical | forbidden |

這張表是 production agent 的基本安全設計。

### Step 4：設計 Agent System Prompt

範例：

```text
你是客服任務處理 agent。

你的目標：
協助使用者處理訂單相關問題，包括查訂單、改地址、查政策、草擬回覆。

限制：
- 你只能使用提供的工具。
- 不得猜測訂單狀態。
- 不得承諾政策外的退款或補償。
- 如果缺少必要資訊，必須先詢問使用者。
- 高風險操作必須等待 approval。
- 若工具結果與使用者說法衝突，以工具結果為準。
- 任務完成前，請持續根據工具結果決定下一步。
```

### Step 5：設計 Agent Loop

第二層 Agent 通常是一個 loop。

```text
while not done:
    1. read state
    2. ask LLM what next action to take
    3. validate action
    4. execute tool
    5. update state
    6. verify result
    7. decide continue / ask user / finish
```

Pseudo code：

```python
while True:
    state = load_state(task_id)

    action = planner.decide(
        user_message=user_message,
        state=state,
        tools=tool_registry,
        constraints=policy
    )

    if action.type == "ask_user":
        return ask_user(action.message)

    if action.type == "finish":
        return final_response(action.message)

    validate_action(action, state, user)

    if requires_approval(action):
        return request_approval(action)

    result = tool_executor.run(action)

    state = update_state(state, action, result)

    verification = verifier.check(state, action, result)

    if not verification.ok:
        state = handle_failure(state, verification)
```

## 四、第二層 Agent 的工具設計

一個好 tool 應該：

- 名稱清楚
- description 明確
- input schema 嚴格
- output schema 穩定
- error 格式一致
- side effect 明確
- 可重試
- 可追蹤
- 可測試

### get_order

```json
{
  "name": "get_order",
  "description": "根據訂單編號查詢訂單狀態、收件地址、付款狀態與是否已出貨。",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string" }
    },
    "required": ["order_id"]
  },
  "side_effect": false,
  "risk_level": "low"
}
```

Output：

```json
{
  "order_id": "A127",
  "status": "paid",
  "shipping_status": "not_shipped",
  "address": "台北市...",
  "customer_email": "user@example.com"
}
```

### search_policy

```json
{
  "name": "search_policy",
  "description": "查詢公司政策，例如退款、改地址、出貨後處理方式。",
  "input_schema": {
    "policy_type": "refund | address_change | shipping"
  },
  "side_effect": false,
  "risk_level": "low"
}
```

Output：

```json
{
  "policy_type": "address_change",
  "allowed": true,
  "conditions": [
    "訂單尚未出貨",
    "地址格式有效"
  ],
  "source": "退款與出貨政策 v2026，第 4 頁"
}
```

### update_address

```json
{
  "name": "update_address",
  "description": "更新尚未出貨訂單的收件地址。",
  "input_schema": {
    "order_id": "string",
    "new_address": "string"
  },
  "side_effect": true,
  "risk_level": "medium",
  "requires_approval": true
}
```

Executor 內部必須再檢查：

```text
order exists
order belongs to user
shipping_status == not_shipped
new_address is valid
policy allows address change
```

不能只因為 Agent 說可以就真的更新。

## 五、狀態機設計

即使第二層 Agent 讓模型決定步驟，也建議保留狀態機。

範例：

```text
START
→ NEED_ORDER_ID
→ NEED_ADDRESS
→ CHECK_ORDER
→ CHECK_POLICY
→ READY_FOR_APPROVAL
→ UPDATE_ADDRESS
→ SEND_CONFIRMATION
→ DONE
```

例外狀態：

```text
ORDER_NOT_FOUND
ADDRESS_INVALID
POLICY_DENIED
TOOL_FAILED
NEED_HUMAN
```

好處：

- 限制 Agent 亂跳
- 每一步可觀察
- 可恢復
- 可測試
- 可做 component eval

## 六、錯誤處理與恢復

第二層 Agent 一定要設計錯誤處理。

常見錯誤：

```text
缺少必要資訊
工具 timeout
工具回傳格式錯
政策查不到
資料庫查無訂單
Agent 選錯工具
Agent 重複呼叫同一工具
Agent 嘗試未授權操作
```

處理策略：

| 錯誤 | 處理 |
|---|---|
| 缺少資訊 | ask_user |
| 查無訂單 | 要求使用者確認 |
| 工具 timeout | retry with limit |
| 政策查不到 | escalate human |
| 未授權操作 | block + log |
| 高風險操作 | request approval |
| 重複失敗 | stop loop |

Loop 防爆一定要設：

```text
max_steps
max_tool_calls
max_retries
timeout
cost budget
```

例如：

```json
{
  "max_steps": 12,
  "max_tool_calls": 8,
  "max_retries_per_tool": 2,
  "timeout_seconds": 60
}
```

## 七、Human Approval 設計

第二層 Agent 可以自主，但不能對所有事情自主。

需要 approval 的操作：

- 寄外部 email
- 退款
- 修改正式資料
- 刪除資料
- 對外發文
- 下單 / 付款
- 高金額操作

Approval request 應該包含：

```json
{
  "action": "update_address",
  "reason": "訂單尚未出貨，政策允許改地址",
  "tool_input": {
    "order_id": "A127",
    "new_address": "建國南路..."
  },
  "risk_level": "medium",
  "source_evidence": [
    "訂單查詢結果",
    "地址修改政策 v2026"
  ]
}
```

人類批准後才執行。

## 八、Observability：要記錄什麼

Production agent 必須留下 trace。

但不一定要記錄模型完整 chain-of-thought；應記錄可審計的 action trace。

建議記錄：

```json
{
  "task_id": "task_123",
  "step": 4,
  "state": "CHECK_POLICY",
  "agent_action": "search_policy",
  "tool_input": {
    "policy_type": "address_change"
  },
  "tool_output_summary": "尚未出貨可改地址",
  "duration_ms": 320,
  "success": true,
  "timestamp": "2026-05-07T10:00:00Z"
}
```

不建議記錄：

- 完整 private credentials
- 使用者敏感資訊全文
- 模型隱含推理全文
- 無遮罩的個資

## 九、Eval：第二層 Agent 要怎麼測

第二層 Agent 至少要測三層。

### 1. Tool Eval

每個 tool 單獨測：

```text
schema validation
permission check
error handling
idempotency
timeout
retry
```

例：

```text
update_address 不可更新已出貨訂單
refund_payment 不可超過付款金額
send_email 必須有 recipient
```

### 2. Planner / Action Eval

測 Agent 是否選對下一步。

測資：

```json
{
  "user_message": "我要改 A127 地址到建國南路",
  "state": {
    "order_id": "A127",
    "new_address": "建國南路",
    "order_status": null
  },
  "expected_next_action": "get_order"
}
```

要測：

```text
缺 order_id 時會 ask_user
有 order_id 會 get_order
未查政策前不 update_address
已出貨不 update_address
```

### 3. End-to-End Eval

完整跑任務。

測：

```text
任務是否完成
工具呼叫順序是否合理
是否違反政策
是否需要 approval
使用者回覆是否清楚
latency 是否可接受
```

Eval 指標：

```text
task success rate
tool call accuracy
policy violation rate
unnecessary tool call rate
human escalation rate
approval precision
latency
cost per task
user satisfaction
```

最重要的是：

```text
policy violation rate 必須接近 0
```

## 十、第二層 Agent 的實作範例：客服改地址

使用者輸入：

```text
我要改 A127 訂單的地址，因為我搬家到建國南路了。
```

Agent 流程：

```text
1. 抽出 order_id=A127, new_address=建國南路
2. get_order(A127)
3. 發現 shipping_status=not_shipped
4. search_policy(address_change)
5. 發現政策允許未出貨訂單改地址
6. request_approval(update_address)
7. approval 通過
8. update_address(A127, 建國南路)
9. draft_email()
10. request_approval(send_email)
11. send_email()
12. 回覆使用者完成
```

為什麼這是第二層：

```text
工具是固定的
Agent 自己決定下一步
高風險操作需要 approval
業務規則由 executor / guardrails 檢查
```

不是第一層，因為步驟不是完全寫死。  
不是第三層，因為 Agent 不能自己創工具、不能自己繞過權限。

## 十一、常見錯誤

### 1. 把 Agent 當萬能大腦

錯誤：

```text
讓 LLM 自己決定所有事，工具也隨便用
```

正確：

```text
工具固定，權限固定，副作用受控
```

### 2. Tool 太大

錯誤：

```text
handle_order_problem()
```

正確：

```text
get_order()
check_policy()
update_address()
send_email()
```

### 3. 沒有狀態

錯誤結果：

- Agent 忘記查過什麼
- 重複呼叫工具
- 無法恢復
- 無法 debug

### 4. 只靠 prompt 約束

錯誤：

```text
Prompt 寫「不要退款超過付款金額」
```

正確：

```text
Executor 裡用程式檢查退款金額
```

Prompt 不是安全機制。

### 5. 沒有 approval

Agent 不應直接做高風險副作用。

尤其是：

- 發送外部訊息
- 修改資料
- 刪除資料
- 金流操作

### 6. 沒有 eval

沒 eval 的 agent 只是 demo。

Production 至少要有：

- tool tests
- planner tests
- end-to-end tests
- regression cases

## 十二、實作優先順序

### MVP

先做：

```text
固定工具集
tool schema
tool executor
basic state store
max_steps
approval gate
action trace
少量 end-to-end eval
```

### 第二階段

再加：

```text
planner eval dataset
component eval
policy verifier
retry strategy
error recovery
observability dashboard
```

### 第三階段

最後加：

```text
multi-agent
dynamic routing
LLM-as-Judge
online feedback
自動 regression suite
```

## 十三、總結

第二層 Agent 的設計重點是：

```text
不要寫死全部步驟
也不要完全放飛

工具固定
規則固定
權限固定
副作用受控
步驟讓 Agent 自己決定
```

最重要的一句話：

> 第二層 Agent 是 production agent 的實用起點：用工程系統控制邊界，用 LLM 在邊界內做彈性決策。
