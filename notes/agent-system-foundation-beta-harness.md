# Agent System Foundation：從多步任務骨架到可治理 Beta Harness

## 這是什麼

`agent-system-foundation` 是一套把 AI 助手從「只會回覆問題」往前推到「能穩定推進真實多步工作」的 foundation。

它的核心重點不是 prompt 本身，而是把以下幾件事制度化：
- workflow
- state
- verification
- run orchestration
- human handoff
- observability
- artifact governance

簡單講，它想解的不是模型夠不夠聰明，而是：

> 當任務跨多步、有副作用、需要驗證、需要之後接回來時，系統要怎麼不靠聊天前後文硬撐，而是有可追蹤、可驗證、可恢復、可演化的工作骨架。

---

## Harness 的定位修正：Runtime + Run Governance

早期這份文件把 `agent-system-foundation` 直接稱為 Harness，核心著重在 workflow、task-state、verification、handoff、audit、repair。這些內容沒有錯，但更精準地說，它目前最成熟的是 **Run Governance Harness**，不是完整 Agent Harness 的全部。

完整 Agent Harness 可拆成兩層：

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
    ├── Human Handoff
    ├── Observability
    ├── Artifact Governance
    ├── Audit
    ├── Repair
    └── Close-out
```

目前 `agent-system-foundation` 的強項主要在下半部，也就是：

```text
Run Governance
= 怎麼確保多步任務可靠完成
```

接下來若要往完整 Harness 演進，真正缺的不是再堆更多治理 artifact，而是把 Runtime 底層補齊。

---

## 它解決什麼問題

在一般聊天式 AI workflow 裡，常見問題有：
- 任務狀態只活在對話裡，過幾輪就漂掉
- 改了什麼、驗了什麼、還缺什麼不清楚
- 多步任務容易做到一半停住，之後難接回
- multi-agent handoff 常把主線搞髒
- run 結束後，學到的規則與摩擦沒有穩定沉澱
- tool call 和真正執行結果容易被混為一談
- 權限、approval、sandbox 分散在各工具裡，難以統一治理
- 多份 state / report 可能彼此 drift，沒有底層執行事實可重播

`agent-system-foundation` 的價值，就是把這些東西拆成能被明確管理的 artifacts 與流程；而完整 Harness Runtime 還需要進一步把執行事實、工具管線與 provider seam 制度化。

---

## 核心演進脈絡

這套 foundation 不是一開始就衝向重型 orchestration 平台，而是逐步補：

### Phase 1：Foundation skeleton
先建立：
- task-state
- verification report
- tool contract
- memory governance
- workflow recommendation

### Phase 2：Run-capable foundation
再補：
- run bootstrap
- capability routing
- multi-agent handoff protocol
- verify → revise → re-verify loop
- close-out 與 memory routing

### Phase 3：Governance layer
再往下補：
- tool orchestration
- human handoff lifecycle
- observability
- artifact schemas
- audit / repair / re-audit
- foundation-wide repair orchestration

截至 2026-04-10，這條線已經從「有骨架」推到：

> **具備 run orchestration + artifact governance + repair loop 的 beta Run Governance Harness**

下一階段不應繼續無限制擴大 Governance，而是補 Runtime Foundation。

---

## 現在已經有哪些核心能力

### 1. Workflow 與 state
系統已能把真實任務落成：
- workflow recommendation
- task-state
- run bootstrap
- bounded deliverable
- verification plan

也就是不再只是「先做看看」，而是任務一開始就有較明確的結構。

### 2. Run orchestration
已補上：
- execution / verification / revision / escalation routing
- tool orchestration plan
- run-level artifacts
- autonomous loop scaffolding

這代表 run 不只是「開始做」，也開始知道：
- 誰負責做
- 誰負責驗
- 驗失敗後誰修
- 哪些情況應升級

### 3. Human handoff lifecycle
高風險步驟已不只是在文件裡說「要問人」，而是有：
- must-confirm gate
- handoff payload
- approve / reject / safe-alternative
- blocked → resume lifecycle
- lifecycle consistency audit

也就是高風險 run 已能真的停下來，不會默默穿越。

### 4. Observability
目前已有：
- per-run observability
- run quality grading
- friction taxonomy
- multi-run aggregation

這讓系統不只知道有沒有做完，也開始知道：
- 卡在哪裡
- 是 routing 問題、tooling 問題還是 handoff 問題
- 某個 patch 之後摩擦是否真的下降

### 5. Artifact governance
這是 2026-04-10 之後成熟很多的一段。

目前已補：
- run artifact schema
- observability schema
- decision-log schema
- human-handoff schema
- task-state schema
- verification-report schema
- memory-closeout schema
- remediation log schema
- audit report schema
- repair batch report schema

也就是說，foundation 不只開始治理流程，也開始治理自己產出的資料結構。

### 6. Audit / Repair / Re-audit loop
現在這套已能：
- validate artifacts
- 用 audit 找 lifecycle / schema 問題
- 對常見 drift 做 safe repair
- 再重跑 audit 確認是否清乾淨

而且 repair 已不只限於 run lifecycle，還開始涵蓋：
- verification report
- task state
- memory closeout

這表示它已經從「有流程」走到「有基本治理閉環」。

---

## 接下來要補的 Harness Runtime

### 1. Append-only Session Event Log

目前已有大量治理 artifact，但還需要一個更底層的執行事實來源。

建議定義：

```text
Session Event Log = source of truth
```

最小事件：

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

然後：

```text
Session Event Log
    ├── derive model history
    ├── project task-state
    ├── build action trace
    ├── feed observability
    └── support resume / replay / audit
```

核心規則：

```text
Event Log = facts
Task State = projection
Verification Report = derived evidence
Audit Report = governance judgment
```

這樣 `task-state` 就不再是另一份可能漂移的真相，而是從執行事實衍生出的狀態視圖。

### 2. Context Assembly

LLM 每一步看到的前後文不應直接靠一份 mutable `messages[]` 撐住。

應該是：

```text
Session Event Log
+ system prompt sections
+ current capability schemas
+ injected context
    ↓
Context Assembly
    ↓
Model Request
```

重要原則：

> Model-visible 的內容必須可以從 durable state 重建。

否則 crash / resume 後模型看到的狀態可能跟原本不同。

### 3. LLM Adapter Seam

Model 不應跟 Agent Loop 綁死。

```text
Agent Loop
    ↓
LLM Interface
    ↓
Adapter
    ↓
Provider
```

這樣 provider swap 不會牽動：
- tool registry
- session model
- governance
- verification
- agent interface

### 4. Agent Interface / Agent Loop Separation

Agent interface 是 contract；default loop 只是其中一個 implementation。

```text
Agent Interface
      ▲
      ├── Default Agent Loop
      ├── Deterministic Workflow Runner
      ├── Subagent
      └── External Agent Runtime
```

其他 Harness 元件應依賴 interface，不依賴某個固定 loop。

這能避免整個 foundation 被綁死在 ReAct、planner 或單一 Agent pattern。

### 5. Centralized Tool Execution Pipeline

任何 tool request 都不應直接：

```text
Agent → tool.execute()
```

應統一經過：

```text
tool/call
    ↓
pre-execute hooks
    ↓
permission / policy
    ↓
approval
    ↓
guards
    ↓
execution wrapper
(timeout / retry / metrics)
    ↓
tool.execute()
    ↓
side-effect / filesystem guard
    ↓
post-execute
    ↓
normalize result
    ↓
tool/result
```

這能把：

```text
Agent 想做什麼
```

和：

```text
這次到底允不允許真的做
```

完全拆開。

### 6. Capability / Provider Seam

Filesystem、subprocess、sandbox、subagent、LLM 等底層能力都應有：

```text
Service Definition
→ Provider
→ Consumer / Tool
```

例如：

```text
Filesystem Capability
├── Local Provider
└── Remote Sandbox Provider
```

Provider 換掉時，上層 tool / Agent / governance 不需要跟著重寫。

### 7. Runtime Recovery Invariants

完整 Harness 要能明確驗證：

```text
每個 tool/call 都有 authoritative tool/result
每個 step/start 最終都能對應 step/end / failure state
每個 turn 都能被 close 或標記 interrupted
resume 後 model history 可從 durable log 重建
權限拒絕時 tool body 沒有執行
副作用不能繞過 guard / sandbox
```

這些不是 UX 細節，而是 Runtime Integrity。

---

## Agent System Foundation 的更新後分層

建議把目前整體概念改成：

```text
Agent System Foundation
│
├── Harness Runtime
│   ├── session event model       ← 待補強
│   ├── context assembly          ← 待補強
│   ├── LLM adapter               ← 待補強
│   ├── agent interface / loop    ← 部分已有
│   ├── tool registry             ← 部分已有
│   ├── tool execution pipeline   ← 待集中
│   ├── permission / approval     ← 部分已有
│   └── sandbox / guards          ← 待補強
│
└── Run Governance
    ├── workflow                  ← 已有
    ├── task-state                ← 已有
    ├── verification              ← 已有
    ├── human handoff             ← 已有
    ├── observability             ← 已有
    ├── artifact governance       ← 已有
    ├── audit                     ← 已有
    ├── repair                    ← 已有
    └── close-out                 ← 已有
```

這個分層比單純說「Foundation = Harness」更精確，也能看出真正的工程缺口在哪。

---

## 目前的成熟度怎麼看

比較準的描述是：

### Run Governance 已經是 Beta
因為現在它已具備：
- 多步 run 進場方式
- verification / revise 節奏
- human handoff control
- observability
- artifact schema validation
- audit / repair / repair orchestration

### 完整 Agent Harness Runtime 還沒到 Product-Grade
目前仍缺或未完全制度化：
- append-only session event model
- model-visible state reconstruction
- swappable LLM adapter
- Agent interface / loop separation
- centralized guarded tool execution pipeline
- capability / provider abstraction
- sandbox execution boundary
- runtime recovery invariants
- 更完整的 policy coverage
- 更產品化的 dashboard / batch UX
- 更成熟的語意修復能力

所以最適合的定位是：

> **Run Governance 已有可用 beta baseline；完整 Agent Harness 仍需要 Runtime Foundation 補齊。**

這不是把原本成熟度往下調，而是把評估範圍從「治理層」擴大到「完整 Harness」。

---

## 現在最好的使用策略

截至目前，最合理的策略不是再無限擴 Governance feature，而是：

1. 先把現有 Run Governance baseline 拿去跑真實任務
2. 觀察實際 friction
3. 保留已有效的 task-state / verification / audit / repair
4. Runtime 優先補 `event log → tool execution pipeline → recovery invariant`
5. 再補 adapter / capability portability
6. 不為了 completeness 繼續堆平行 artifact

Runtime 的優先順序建議是：

```text
1. Session Event Log
2. Tool Execution Pipeline
3. Runtime Integrity Eval
4. Context Reconstruction
5. Agent / LLM Adapter Seam
6. Capability Provider Seam
7. Sandbox portability
```

原因是前四項先解決「執行事實是否可信」，後三項再解決「架構是否可替換」。

---

## 適合記住的核心觀點

### 1. 模型能力不等於系統能力
一個會寫、會答的模型，不代表它已經是一個能穩定交付多步工作的系統。

### 2. 真正的可靠，不只來自 prompt
而來自：
- durable execution facts
- workflow
- state projection
- guarded execution
- verification
- governance
- repair

### 3. State 不能有多個互相競爭的真相
如果 session、task-state、action trace、verification report 都能各自描述「發生了什麼」，就會產生 drift。

應該優先建立：

```text
Event Log = fact source
其他 artifacts = projection / evidence / judgment
```

### 4. Tool call 不是執行證據
模型輸出：

```text
tool/call
```

只代表「要求執行」。

真正可以作為證據的是 Harness Runtime 留下的：

```text
policy decision
approval result
execution result
tool/result
side-effect observation
```

### 5. human handoff 不是例外，而是系統的一部分
當風險牽涉到 production、權限、外部影響時，真正好的系統不是偷偷做完，而是能在正確的時間停下來，把高品質決策材料交給人。

### 6. 治理不只是抓錯，也要能修
只有 audit 沒有 repair，系統還是會越用越髒。真正成熟的一步，是開始建立：
- schema
- validator
- repair policy
- remediation log
- repair orchestration

### 7. Harness 的邊界不是「包住 Agent」而已
更精確的理解是：

```text
Harness Runtime = Agent 執行環境
Run Governance = 任務治理環境
Agent Decision Policy = 下一步決策
```

三者分工清楚，才能真正替換 model、loop、tool provider，而不破壞整個系統。

---

## 一句話總結

`agent-system-foundation` 的本質，是把 AI 從聊天代理逐步推向可治理工作系統；目前 workflow、task-state、verification、handoff、observability、artifact governance、audit 與 repair 已形成可用的 **Run Governance beta baseline**，下一階段應停止繼續橫向堆治理 artifact，轉而補齊 **Session Event Log、Context Reconstruction、LLM / Agent Adapter、Centralized Tool Execution Pipeline、Capability Seam 與 Runtime Recovery Invariants**，才會成為更完整的 Agent Harness。

---

## 參考架構

本次分層補強參考：

- DeepSeek Harness Architecture：`deepseek-ai/deepseek-harness/docs/architecture.md`
- DeepSeek Harness Tool Execution Pipeline：`deepseek-ai/deepseek-harness/docs/tool-execution-pipeline.md`

吸收的是可泛化的 Harness 原則，不綁定 Cordis 或 DeepSeek 的具體 plugin 實作。

#AI #AgentSystemFoundation #OpenClaw #Workflow #Orchestration #Verification #Observability #KnowledgeManagement #MultiAgent #Governance #Harness #Runtime #EventLog