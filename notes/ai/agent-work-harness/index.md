# Agent Work Harness：把 Coding Work 安全交給外部 Agent

#ai #agent #governance #evidence #sandbox #index

## 這是什麼

一次完整的系統開發紀錄：設計一個 **Work Governance Layer**，把 coding work 交給外部
Agent Runtime（Codex）執行，但**不採信 Agent 的自述**，改以獨立收集的 evidence 決定
接受、重試、詢問或阻擋。

核心定位是一句話：

> Harness 管 Work，不管 Agent loop。

它不重新實作 Agent 已經擅長的 repository discovery、tool selection 與多步推理，
只負責治理：需求 → authority → context → prompt → attempt → evidence → outcome。

實作是 TypeScript on Node 24，零 runtime 依賴，程式碼在
[ChenHom/agent-work-harness](https://github.com/ChenHom/agent-work-harness)。

## 這裡收錄什麼

| 筆記 | 類型 | 內容 |
|---|---|---|
| [開發過程與每階段真正抓到的問題](note.html?slug=ai/agent-work-harness/development-log) | Research | 從設計文件到 MVP 到跨 repo 驗證，每個驗證活動實際抓到什麼 |
| [Sandbox 隔離的實測邊界](note.html?slug=ai/agent-work-harness/sandbox-isolation-findings) | Knowledge | codex sandbox 與 bubblewrap 的真實行為，文件假設與實測不符的地方 |
| [Evidence 可信度的四層模型](note.html?slug=ai/agent-work-harness/evidence-trust-model) | Knowledge | 為什麼「測試通過」不等於「驗證完整」，以及怎麼分層處理 |
| [TypeScript Computational Sensors：AI 程式碼的機械檢測 Harness](note.html?slug=ai/agent-work-harness/computational-sensors-typescript) | Knowledge | 用 tsc、ESLint、Knip、dependency-cruiser、Vitest、Stryker、Semgrep 等工具，把可客觀判定的問題轉成硬 evidence |
| [設計決策與它們的理由](note.html?slug=ai/agent-work-harness/design-decisions) | Decision record | 每個決定背後的取捨，以及什麼情況下該反過來做 |

## 最值得帶走的三件事

**1. 驗證活動的價值等於它抓到什麼。**
E2E 場景抓到 3 個缺陷、dogfood 抓到 3 個、跨 repo 驗證抓到 2 個。
每一輪都不是「跑過就好」，而是先問「這一輪要證偽什麼假設」。

**2. 隔離要實測，不能照文件假設。**
三個安全假設裡有兩個實測後不成立 —— 沙箱模式本身不擋網路，Agent 的 HOME 完全暴露。
如果照文件寫程式，會得到一個「宣稱隔離但沒有隔離」的系統。

**3. 「沒有專用欄位」不等於「表達不了」。**
遇到 schema 表達不了的需求時，先確認現有機制真的不夠 ——
一個 `env(1)` 就解決的問題，差點變成新的 config 欄位。

## 相關

- [Agent 執行流程 SOP](note.html?slug=ai/agent-flow-sop)：Agent 端的決策、循環與停止條件
- [Agent、RAG、Harness 系統 Eval 功能設計](note.html?slug=ai/agent-rag-harness-eval-design)：評估面
