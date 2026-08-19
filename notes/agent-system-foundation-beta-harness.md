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

> 當任務跨多步、有副作用、需要驗證、需要之後接回來時，系統要怎麼不靠聊天上下文硬撐，而是有可追蹤、可驗證、可恢復、可演化的工作骨架。

---

## 它解決什麼問題

在一般聊天式 AI workflow 裡，常見問題有：
- 任務狀態只活在對話裡，過幾輪就漂掉
- 改了什麼、驗了什麼、還缺什麼不清楚
- 多步任務容易做到一半停住，之後難接回
- multi-agent handoff 常把主線搞髒
- run 結束後，學到的規則與摩擦沒有穩定沉澱

`agent-system-foundation` 的價值，就是把這些東西拆成能被明確管理的 artifacts 與流程。

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

> **具備 run orchestration + artifact governance + repair loop 的 beta harness**

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

## 目前的成熟度怎麼看

比較準的描述是：

### 已經是 Beta Harness
因為現在它已具備：
- 多步 run 進場方式
- verification / revise 節奏
- human handoff control
- observability
- artifact schema validation
- audit / repair / repair orchestration

### 但還不是 Product-Grade Platform
因為它仍缺：
- 更完整的 policy coverage
- 更產品化的 dashboard / batch UX
- 更成熟的語意修復能力
- 更完整的 control plane / adapter portability

所以最適合的定位是：

> **可以先穩定使用的 beta harness baseline**

而不是一個要無止盡繼續擴功能的半成品。

---

## 現在最好的使用策略

截至目前，最合理的策略不是再無限擴 feature，而是：

1. 先把這套 baseline 拿去跑真實任務
2. 觀察實際 friction
3. 只針對重複、結構性缺口補 patch
4. 不為了 completeness 繼續堆規格

這也是為什麼目前已有一份 freeze note，建議把這個狀態先視為可收的 beta 基線。

---

## 適合記住的核心觀點

### 1. 模型能力不等於系統能力
一個會寫、會答的模型，不代表它已經是一個能穩定交付多步工作的系統。

### 2. 真正的可靠，不只來自 prompt
而來自：
- workflow
- state
- verification
- governance
- repair

### 3. human handoff 不是例外，而是系統的一部分
當風險牽涉到 production、權限、外部影響時，真正好的系統不是偷偷做完，而是能在正確的時間停下來，把高品質決策材料交給人。

### 4. 治理不只是抓錯，也要能修
只有 audit 沒有 repair，系統還是會越用越髒。真正成熟的一步，是開始建立：
- schema
- validator
- repair policy
- remediation log
- repair orchestration

---

## 一句話總結

`agent-system-foundation` 的本質，是把 AI 從聊天代理，逐步推向一個具備 workflow、run orchestration、verification、handoff、observability 與 artifact governance 的可治理工作系統；截至 2026-04-10，它已經到達值得先 freeze、先真實使用、再依摩擦演化的 beta harness 狀態。

#AI #AgentSystemFoundation #OpenClaw #Workflow #Orchestration #Verification #Observability #KnowledgeManagement #MultiAgent #Governance #Harness
