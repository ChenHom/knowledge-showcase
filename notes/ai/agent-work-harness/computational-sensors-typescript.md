# TypeScript Computational Sensors：AI 程式碼的機械檢測 Harness

#ai #agent #harness #typescript #testing #verification #computational-sensors

## 這是什麼

這篇整理 Martin Fowler 網站上 Harness Engineering 系列的 Computational Sensors 概念，並落到 TypeScript 專案：哪些問題應該交給型別檢查、Lint、架構規則、測試、Mutation Testing、安全掃描等工具機械判定，以及這些結果怎麼成為 Agent 可重跑、可比較的 evidence。

參考文章：

- [Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html)
- [Maintainability sensors for coding agents](https://martinfowler.com/articles/sensors-for-coding-agents.html)

## 核心結論

機械檢測的目的不是證明「AI 寫得很好」，而是先攔掉所有**可以客觀證明不符合規則**的東西。

```text
AI 寫 code
   ↓
Computational Sensors
   ├─ 型別
   ├─ Lint / 靜態規則
   ├─ 架構依賴
   ├─ Dead code
   ├─ Tests / Coverage
   ├─ Mutation Testing
   ├─ Security / Secret Scan
   └─ Dependency vulnerability
   ↓
任一 FAIL → Agent 修正 → 重跑
   ↓
全部通過
   ↓
Inferential Review
   ├─ abstraction 合不合理
   ├─ domain model 對不對
   ├─ coupling 是否真的有害
   └─ 需求本身是否正確
   ↓
Human / AI 語意判斷
```

分界很重要：

> **能寫成 deterministic rule 的，盡量機械化；需要理解「為什麼」的，不要硬塞成分數或規則。**

## TypeScript 可用的工具

| 檢測面 | 工具 | 主要抓什麼 | 在 Harness 的角色 |
|---|---|---|---|
| 型別 | `tsc --noEmit` | 型別錯誤、null/undefined、介面不一致 | 最基本 hard gate |
| Lint | ESLint + typescript-eslint | 危險 pattern、Promise 誤用、複雜度、過長函式 | 程式規則 sensor |
| 快速 Lint / Format | Biome | 格式與部分靜態規則 | 可替代或補充 ESLint，不必重複檢一樣的事 |
| Dead code | Knip | 未使用檔案、export、dependency | 防 AI 重構後留下垃圾 |
| 架構規則 | dependency-cruiser | 循環依賴、跨層引用、依賴方向 | 防功能正確但架構腐化 |
| 靜態 / 安全分析 | Semgrep | Bug pattern、安全問題、自訂團隊規則 | 把 Code Review 規則 executable 化 |
| Unit / Integration | Vitest | 已定義行為是否符合預期 | 行為 evidence |
| Coverage | Vitest Coverage | 哪些 statement / branch / function 沒被測到 | 找測試空洞，不代表測試品質 |
| Mutation Testing | StrykerJS | 測試是否真的能抓出邏輯被改壞 | 檢查測試有效性 |
| E2E | Playwright | 真實瀏覽器流程與整合行為 | 高層行為 gate |
| Secret Scan | Gitleaks | API key、token、private key、密碼 | commit / CI gate |
| Dependency 漏洞 | OSV-Scanner / `npm audit` | 第三方套件已知漏洞 | supply-chain sensor |

不需要為了「工具多」全部裝。每一個 sensor 都應回答一個不同問題；如果兩個工具只是在重複同一類規則，維護成本會高於價值。

## 1. `tsc`：先把型別問題變成硬失敗

最基本：

```bash
npx tsc --noEmit
```

建議至少把 TypeScript 的嚴格檢查打開：

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

這層可以直接機械判定：

```text
user.profile.name
        ↓
profile?: Profile
        ↓
Object is possibly undefined
        ↓
FAIL
```

這種錯誤不需要交給 LLM Review。

## 2. ESLint + typescript-eslint：把常見 Review 意見寫成規則

除了格式，真正有價值的是 type-aware rules 與複雜度限制，例如：

```text
no-floating-promises
no-misused-promises
no-explicit-any
complexity
max-lines
max-lines-per-function
max-params
```

原本 Code Review 的：

```text
「這個 function 太複雜」
```

如果團隊真的能定義門檻，就改成：

```text
complexity > 10 → FAIL
function > 80 lines → FAIL
params > 5 → FAIL
```

但門檻不是自然定律。錯誤的數字只會逼 Agent 拆出一堆無意義的小函式，所以規則要來自真實維護問題，而不是為了讓 dashboard 看起來完整。

## 3. Knip：抓 AI 重構後的殘留物

Agent 很容易：

```text
修改 A
→ 新增 B
→ caller 全部改用 B
→ A 已無用途
→ 忘記刪掉 A
```

Knip 可抓：

```text
unused files
unused exports
unused dependencies
unused devDependencies
unused types
```

這類問題人工 Review 很浪費時間，而且非常適合 deterministic sensor。

## 4. dependency-cruiser：檢查架構，不只檢查單一檔案

例如架構規定：

```text
controller
    ↓
application
    ↓
domain

infrastructure → 實作 domain 定義的 port
```

可以直接禁止：

```text
domain → infrastructure
domain → controller
production → test
```

如果 AI 寫出：

```ts
// src/domain/order.ts
import { prisma } from "../infrastructure/prisma";
```

不需要人看到才說架構反了：

```text
domain-must-not-depend-on-infrastructure → FAIL
```

這一層對 coding agent 特別重要，因為 Agent 很常做到：

```text
功能完成
TypeScript 通過
Tests 通過
但依賴方向逐次腐化
```

## 5. Semgrep：把團隊知識變 executable rule

適合處理：

```text
禁止 eval
禁止直接組 SQL
API route 不得直接碰 DB
特定資料必須先 validation
禁止 catch 後吞掉錯誤
某類敏感資料不得寫 log
```

這不是要取代完整 SAST，而是把「團隊已經知道不能這樣做」變成可重複執行的規則。

判斷標準：

```text
Reviewer 每次都會留下同一句 comment
        ↓
而且能精確描述 AST / pattern / dependency rule
        ↓
考慮轉成 sensor
```

## 6. Vitest：測的是行為，不是程式碼長相

Unit / Integration Test 回答：

> 在這組 Given / When 下，Then 是否仍成立？

這是必要 evidence，但要注意：

```text
Tests PASS
```

只代表：

```text
目前存在的 assertion 全部通過
```

不代表：

```text
需求正確
測試完整
測試本身沒有寫錯
正式環境一定沒問題
```

這跟 [Evidence 可信度的四層模型](note.html?slug=ai/agent-work-harness/evidence-trust-model) 是同一件事：`exitCode == 0` 只是一層訊號。

## 7. Coverage：找「沒測到」，不能證明「測得好」

Coverage 可以設 threshold：

```text
Statements ≥ 90%
Branches   ≥ 85%
Functions  ≥ 90%
Lines      ≥ 90%
```

低於門檻直接 FAIL。

但：

```text
Coverage 100%
```

只能證明程式碼被執行過，不能證明 assertion 有能力抓錯。

因此 coverage 的用途應理解成：

> **找測試空洞，不是測試品質分數。**

## 8. StrykerJS：檢查測試有沒有真的咬住邏輯

假設：

```ts
function canBuy(age: number) {
  return age >= 18;
}
```

測試只有：

```ts
expect(canBuy(20)).toBe(true);
```

Coverage 可以是 100%。

Mutation Testing 會故意改程式：

```ts
return age > 18;
return age < 18;
return true;
```

再重新跑測試。

結果：

```text
Killed    → 測試能抓到這個錯誤
Survived  → 程式被改壞，測試仍然通過
```

所以：

```text
Tests PASS
+ Coverage 高
+ Mutation score 合理
```

比單純 `Coverage 100%` 強得多。

但 mutation testing 通常較慢，不適合 Agent 每改一行就全量執行；可以做 changed-file / incremental，或放在較深的 CI gate。

## 9. Gitleaks 與 dependency vulnerability

這兩類規則不需要 LLM 判斷：

```text
API key 被 commit     → FAIL
Private key 被 commit → FAIL
已知 critical CVE     → FAIL / policy 判定
```

需要注意 dependency vulnerability 的結果仍有 policy 問題，例如：

```text
漏洞是否真的可達？
有沒有官方修正版？
只是 devDependency 還是 production runtime？
```

掃描結果可以機械產生，但最後是否阻擋要有明確政策，不要讓 Agent 臨場決定。

## 建議的 Harness 分層

不要每次存檔都跑全部工具。

### Fast loop：Agent 每輪修改後

目標是幾秒內回饋：

```text
tsc --noEmit
ESLint
Vitest affected / targeted tests
```

### Structural gate：準備交付前

```text
Knip
dependency-cruiser
Semgrep
完整 Vitest
Coverage
Gitleaks
```

### Deep gate：CI / 高風險變更

```text
StrykerJS
Playwright E2E
dependency vulnerability scan
完整安全掃描
```

概念：

```text
             Agent 修改
                 │
                 ▼
        ┌─────────────────┐
        │ Fast Sensors    │
        │ tsc / lint/test │
        └────────┬────────┘
                 │ FAIL
                 └──────────────→ Agent 修正
                 │ PASS
                 ▼
        ┌─────────────────┐
        │ Structural      │
        │ arch/dead/SAST  │
        └────────┬────────┘
                 │ PASS
                 ▼
        ┌─────────────────┐
        │ Deep Evidence   │
        │ mutation / E2E  │
        └────────┬────────┘
                 │
                 ▼
          Inferential Review
```

## 一套實用的 TypeScript 最小組合

如果先做第一版，不要一口氣裝十幾套：

```text
必做
├─ tsc --noEmit
├─ ESLint + typescript-eslint
├─ Vitest
└─ Coverage

第二批
├─ Knip
├─ dependency-cruiser
├─ Semgrep
└─ Gitleaks

測試開始大量由 AI 產生後
└─ StrykerJS

有真實 Web 流程
└─ Playwright
```

其中最值得補在一般 `tsc + lint + tests` 之外的是：

```text
dependency-cruiser → 防架構腐化
Knip               → 防殘留 dead code
StrykerJS          → 防「為了 PASS 而 PASS」的弱測試
Semgrep            → 把團隊 Review 規則機械化
```

## package scripts 可以長這樣

```json
{
  "scripts": {
    "verify:fast": "tsc --noEmit && eslint . && vitest run",
    "verify:structure": "knip && depcruise src --config .dependency-cruiser.cjs",
    "verify:test": "vitest run --coverage",
    "verify:mutation": "stryker run",
    "verify": "npm run verify:fast && npm run verify:structure && npm run verify:test"
  }
}
```

Semgrep、Gitleaks、OSV-Scanner 通常比較適合由 CI / Harness runner 統一執行，不必硬塞成 npm package dependency。

## Evidence 不要只存 PASS / FAIL

Harness 如果最後只記：

```text
npm test: PASS
```

資訊仍然太少。

至少應保留：

```text
sensor: vitest
command: npm test
exitCode: 0
durationMs: 1820
tests:
  total: 63
  passed: 63
  failed: 0
  skipped: 0
baseline:
  total: 63
  skipped: 0
```

其他 sensor 也一樣：

```text
tsc
  errors: 0

eslint
  errors: 0
  warnings: 2

dependency-cruiser
  violations: 0
  cycles: 0

coverage
  statements: 94.2%
  branches: 88.7%

mutation
  killed: 146
  survived: 9
  score: 94.2%

gitleaks
  leaks: 0
```

這樣人看到的是**工具真正執行出的 evidence**，不是 Agent 自己整理一句「所有檢查都通過」。

## 機械檢測仍然會被騙

這是最重要的限制。

### 測試本身可能是錯的

Agent 會迎合錯誤 assertion：

```text
需求正確答案 = 6
測試錯寫 = 7
Agent 為了 PASS 在正式 code +1
```

結果可能是：

```text
tsc PASS
lint PASS
tests PASS
coverage PASS
mutation PASS
```

但系統仍然錯。

所以 Computational Sensors 能證明的是：

> **實作符合目前可執行的規則與測試。**

不能證明：

> **這些規則與測試代表真正需求。**

### Agent 可能讓測試少跑

`exitCode == 0` 不代表所有測試有跑。

應比較：

```text
pre-flight: 63 tests / 0 skip
post:       55 tests / 8 skip
```

這時即使 exit 0，也不能算完整 PASS。

### 指標不等於判斷

例如：

```text
fan-out = 14
complexity = 11
file = 430 lines
```

這些數字可以機械產生。

但：

```text
這個 coupling 在這個 domain 是否合理？
這個 abstraction 值不值得拆？
```

不是數字自己能回答。

## 最後的邊界

```text
可以機械化
────────────────────────
型別
Lint rule
複雜度門檻
Dependency direction
Circular dependency
Dead code
Unit / Integration / E2E
Coverage
Mutation
Secret
已知 vulnerability
明確安全 pattern

不該假裝能完全機械化
────────────────────────
需求是不是對的
測試 Oracle 是不是對的
Domain model 合不合理
Abstraction 好不好
高 coupling 是否真的有害
商業風險是否可接受
```

因此完整 Harness 不是：

```text
AI 寫 code
→ AI Review
→ AI 說 OK
```

而是：

```text
AI 寫 code
→ Computational Sensors 產生硬 evidence
→ Harness 確認 evidence 完整性與來源
→ Inferential Review 處理語意問題
→ 人只看仍需要判斷的部分
```

機械檢測不是用來取代工程判斷，而是把工程師從「機器本來就能判斷的事」裡移出去。

## 相關

- [Evidence 可信度的四層模型](note.html?slug=ai/agent-work-harness/evidence-trust-model)：sensor 跑出結果後，怎麼判斷 evidence 的完整性、獨立性與充分性
- [開發過程與每階段真正抓到的問題](note.html?slug=ai/agent-work-harness/development-log)：驗證活動應該以「實際抓到什麼」衡量價值
- [AI-First Testing](note.html?slug=ai/ai-first-testing-workflow)：AI 探索測試、固定 Test Code 與 Test Oracle 的分工
