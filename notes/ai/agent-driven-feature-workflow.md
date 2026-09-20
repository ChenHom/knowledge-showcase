# Agent 主導的功能開發流程：六步驟、五角度、機械化檢查

#playbook #ai-agents #workflow #feature-flags #claude-code

延伸自 [Copilot runtime 移植 Rust 的重點整理](note.html?slug=ai/copilot-rust-migration-takeaways) 的「套到產品開發」段落。那篇講機制，這篇定固定流程：每步的**進入條件 → 動作 → 判斷點 → 產出 → 退出條件**，以及步驟 1–3 怎麼用腳本和 hook 把判斷點機械化。

## 判斷角度（全流程共用）

| 代號 | 角度 | 判斷時問的問題 |
|---|---|---|
| **R** 可逆性 | 錯了退得回嗎 | 退回要幾步、由誰、多久？一鍵（旗標）／一個 revert／要人工修資料 |
| **O** 可觀測性 | 錯了多快知道 | 誰會先發現：CI、監控、內部使用者、外部使用者？間隔多久 |
| **B** 爆炸半徑 | 錯了影響多大 | 動到幾個模組、幾條呼叫路徑、幾個使用者群 |
| **A** 資訊不對稱 | 這一步誰知道的比較多 | agent 讀了程式碼我沒讀 → 我不該逐行審；我知道產品意圖 agent 不知道 → 我該寫進描述 |
| **M** 動機分岔 | agent 的目標跟我的目標會不會分開 | agent 的目標是「測試綠、PR 合併」；我的是「行為正確」。這一步兩者會不會用不同手段達成 |

每個判斷點只用其中一到兩個角度，全用等於沒用。R 和 B 決定「這步要多小心」，O 決定「要不要人看」，A 決定「人看什麼」，M 決定「人防什麼」。

## 流程總表

```text
0 判準 ──gate: 每模組有行為測試 + CI 擋合併──▶
1 功能卡 ──gate: 每片可獨立出貨、共用模組已標──▶
2 片卡 → agent 方案 ──gate: 範圍⊆片卡、測行為──▶
3 執行 [C1 測試手段 / C2 同錯二次 / C3 越界] ──gate: 一片一 PR、CI 綠──▶
4 審查 [回應分類 / 不准動 / 形狀] ──gate: 三項過──▶
5 發佈 [內部驗證 / 基線放量 / 關旗標] ──gate: 旗標刪──▶
6 回顧 → 改 0/1/2 的模板與檢查
```

人的判斷集中在四個地方：步驟 1 的切片、步驟 2 的方案審、步驟 3 的三個檢查點、步驟 4 的三件事。其他自動或交 agent。

## 步驟 0：建判準

**進入**：有一個功能要做。
**動作**：列出這功能會動到的模組；對每個模組確認現有的自動化檢查。

| 判斷點 | 角度 | 通過 | 不通過 → |
|---|---|---|---|
| 這模組的既有行為有測試守著嗎 | O | 改壞會有測試紅 | 補行為測試，列為片 0 |
| 測試測的是行為還是實作 | M | 重構不動測試 | 重寫成黑箱測試；agent 寫的測試一律用這條驗 |
| CI 會擋合併嗎 | O | 紅燈不能 merge | 開 branch protection |
| 有第二個獨立審查來源嗎 | A | 有非撰寫者的模型或人 | 開第二模型 review，最低配 |

**產出**：判準清單（模組 → 對應檢查）。
**退出**：每個會動到的模組都有至少一個行為測試、CI 擋合併。達不到就不進步驟 1。

## 步驟 1：功能規劃（人做，一次）

**進入**：判準到位。
**動作**：寫功能卡，三段：完成狀態、切片順序、不准動清單。

| 判斷點 | 角度 | 判斷方式 |
|---|---|---|
| 完成狀態寫的是終點還是動作 | A | 每句都能改寫成「使用者能……，驗證方式是……」才算終點。寫成「實作 X」的是動作，重寫 |
| 每片單獨合併後 main 能出貨嗎 | R | 逐片問。答否的片缺的一定是旗標或介面——補一片在它前面 |
| 哪些片共用模組 | B | 用檔案路徑對，不用「應該不會撞」。共用的標序列化 |
| 這波第一片是什麼 | R + B | 選可逆性最高、半徑最小的：通常是 schema 或純函式。不從 UI 開始 |
| 不准動清單完整嗎 | M | 固定四項起跳：tests/、CI 設定、旗標豁免、既有公開介面行為。再加功能特有的 |

**產出**：功能卡。
**退出**：每片都能回答「單獨合併後可出貨」；共用模組已標。

### 機械化：功能卡是 `feature.yaml`

```yaml
feature: two-factor-auth
done_when:
  - user: 在設定頁開啟兩步驗證並綁定 TOTP
    verify: e2e/2fa_setup.spec.ts
  - user: 登入時被要求輸入 code
    verify: e2e/2fa_login.spec.ts

frozen:                      # 不准動，預設四項 + 功能特有
  - tests/**
  - .github/**
  - src/auth/login.ts

slices:
  - id: s0-tests
    gate: test-only
    touches: [tests/auth/**]
  - id: s1-schema
    gate: additive-schema
    touches: [db/migrations/**, src/models/user.ts]
    after: [s0-tests]
  - id: s2-totp-core
    gate: interface-only
    touches: [src/auth/totp.ts, tests/auth/totp.test.ts]
    after: [s0-tests]
  - id: s3-api
    gate: flag
    flag: FF_2FA
    touches: [src/api/2fa/**, src/auth/service.ts]
    after: [s1-schema, s2-totp-core]
  - id: s5-login-hook
    gate: flag
    flag: FF_2FA
    touches: [src/auth/service.ts, src/auth/login.ts]
    unfreeze: [src/auth/login.ts]   # 附理由
    after: [s3-api]
  - id: s6-cleanup
    gate: cleanup
    flag: FF_2FA
    after: [s5-login-hook]
```

`lint_feature.py` 在 commit 功能卡時跑：

| 檢查 | 對應判斷點 | 規則 | 結果 |
|---|---|---|---|
| `done_when[].verify` 存在或列在某片 `touches` | 完成狀態是終點 | 沒 verify 的條目 = 動作 | 擋 |
| `gate` ∈ {test-only, additive-schema, interface-only, flag, cleanup} | 單獨合併可出貨 | `gate: flag` 必須有 `flag:` | 擋 |
| `touches` 兩兩交集 | 共用模組 | 交集非空且無 `after` 關係 → 自動加 `serialize` | 自動補 + 警告 |
| `after` 圖無環、第一片 gate 是 test-only 或 additive-schema | 第一片半徑最小 | — | 擋 |
| `touches` ∩ `frozen` | 不准動 | 非空 → 那片要顯式 `unfreeze` 附理由 | 擋 |
| `frozen` 含預設四項 | 清單完整 | — | 擋 |

腳本擋不住的：切片順序的 R 判斷、touches 有沒有漏。後者半機械：`git log --since=6.month -- <touches>` 看這些檔歷史上跟誰一起改，高頻共改但不在 touches 的印出來提醒。

## 步驟 2：片規劃（人定邊界，agent 細切）

**進入**：功能卡、這一波要做哪幾片。
**動作**：人寫片卡 → agent 讀程式碼提方案 → 人審方案。

片卡的判斷：

| 判斷點 | 角度 | 判斷方式 |
|---|---|---|
| 「不改什麼」寫了嗎 | M | 沒寫 = 授權順手重構。至少列相鄰但不在範圍的檔 |
| 介面簽名和錯誤型態定了嗎 | A | agent 不知道產品意圖，介面是唯一傳遞方式。沒定它會自己發明，而且發明得很合理 |
| 判準對到具體測試名稱了嗎 | O | 「要有測試」不算，「`verify_rejects_bad_code` 要綠」才算 |

方案的判斷：

| 判斷點 | 角度 | 通過 | 不通過 → |
|---|---|---|---|
| 要動的檔 ⊆ 片卡範圍 | M | 子集 | 超出的每個檔要它解釋；解釋不了就退 |
| 它發現了我沒想到的相依嗎 | A | 有，且合理 | 這是 agent 比我知道多的地方，要認真看，通常代表功能卡切片要改 |
| 測試計畫測行為嗎 | M | 從外部入口驗結果 | 測內部函式或 mock 太多 → 退，指定測試入口 |
| 片太大 | B | — | 它不會主動說。動超過 5 個檔或跨兩個模組 → 再切 |

**產出**：確認過的方案。
**退出**：方案範圍 ⊆ 片卡範圍，測試計畫是行為測試。

### 機械化：片卡由功能卡生成，方案有固定格式

`slice_card.py s3-api` 產出片卡，人只填 `interface` 和 `acceptance`：

```yaml
slice: s3-api
allow: [src/api/2fa/**, src/auth/service.ts]          # 從 touches 來
deny:  [tests/**, .github/**, src/auth/login.ts]      # 從 frozen 來
flag: FF_2FA
interface:                                             # 人填
  - "POST /2fa/setup -> {secret, qr} | 401"
  - "POST /2fa/verify {code} -> 204 | 400 InvalidCode | 429"
acceptance:                                            # 人填
  - tests/api/2fa.test.ts::setup_returns_secret
  - tests/api/2fa.test.ts::verify_rejects_bad_code
```

指示檔寫死：agent 讀完程式碼先產 `plan.yaml`，不動手：

```yaml
files_modify: [src/auth/service.ts]
files_create: [src/api/2fa/setup.ts, src/api/2fa/verify.ts, tests/api/2fa.test.ts]
deps_found:                          # 範圍外但相關的
  - path: src/auth/session.ts
    why: verify 成功後要寫 session flag
tests_planned:
  - name: setup_returns_secret
    entry: "POST /2fa/setup"
    mocks: []
  - name: verify_rejects_bad_code
    entry: "POST /2fa/verify"
    mocks: [clock]
```

`check_plan.py`：

| 檢查 | 對應判斷點 | 規則 | 結果 |
|---|---|---|---|
| `files_*` ⊆ `allow` 且 ∩ `deny` = ∅ | 範圍一致 | — | 擋，印超出的檔 |
| `deps_found[].path` | agent 發現的相依 | 在 allow → 忽略；在 deny → 標紅；都不在 → 標黃 | 人看 |
| `tests_planned[].name` ⊇ `acceptance` | 判準對得上 | 缺的印出 | 擋 |
| `tests_planned[].entry` ∈ interface | 測行為 | entry 不在 interface → 測的是內部 | 擋 |
| `mocks` > 2 | 測行為 | — | 標黃 |
| 檔案 > 5 或跨 > 2 個頂層目錄 | 片太大 | — | 標黃 |

人只看標紅標黃的；綠的方案直接放行。

## 步驟 3：執行（agent 主導，人設檢查點）

**進入**：確認過的方案、獨立 worktree。
**動作**：agent 執行；人不盯過程，只在三個檢查點介入。

| 檢查點 | 觸發 | 角度 | 判斷 |
|---|---|---|---|
| **C1 第一次測試跑完** | agent 回報測試結果 | M | 它讓測試綠的手段：改實作（正常）／改測試、加 skip 或 ignore、改判準定義（都退） |
| **C2 同一錯誤第二次** | agent 卡在同一個錯 | A | 這是資訊缺口不是能力問題。停，問片卡少了什麼、判準是不是本身錯的。不讓它試第三次 |
| **C3 想動範圍外的檔** | agent 提出或已動 | B + M | 問「不動這個檔這片能完成嗎」。能 → 退回；不能 → 切片錯，回步驟 1，不在這片硬做 |

固定做法：一片一 session；探索交子代理，主代理只編輯；序列化的片等前一片合併才開；agent 回報完成時先看 diff 形狀再看內容。

**產出**：PR。
**退出**：PR 只含一片、CI 綠。

### 機械化：三個檢查點各對一個 hook

**C3 越界 → PreToolUse hook 直接擋**（Claude Code，`Edit|Write|MultiEdit`）：

```bash
path=$(jq -r '.tool_input.file_path')
python3 scripts/in_scope.py "$path" .slice/card.yaml || {
  echo "out of scope for slice: $path" >&2
  exit 2   # 擋下，訊息回給 agent
}
```

agent 被擋且看到原因，下一步只能回報不能硬做。

**C1 測試手段 → diff 檢查**（pre-commit + CI 各一次）：

```bash
git diff base..HEAD -- 'tests/**' --diff-filter=M | grep -q . && fail "modified existing test"
git diff base..HEAD | grep -E '^\+.*(\.skip\(|@ts-ignore|eslint-disable|# noqa|xit\(|xdescribe)' && fail "suppression added"
[ "$(count_tests HEAD)" -lt "$(count_tests base)" ] && fail "test count dropped"
git diff base..HEAD | grep -E '^\+.*expect\(true\)|assert True' && fail "vacuous assertion"
```

`--diff-filter=M` 是關鍵：新增測試檔允許，修改既有測試檔擋。功能卡 `unfreeze` 過的片放行。四條擋的是 M 角度的四種典型手段，agent 要讓測試綠只剩改實作一條路。

**C2 同錯二次 → 錯誤簽章計數**（PostToolUse hook，`Bash`）：

```bash
out=$(jq -r '.tool_response.stderr // .tool_response.stdout' | tail -20)
sig=$(echo "$out" | grep -E 'Error|FAIL|error\[' | head -3 | sed 's/[0-9]//g' | md5sum | cut -c1-8)
[ -z "$sig" ] && exit 0
n=$(grep -c "$sig" .slice/errors.log 2>/dev/null || echo 0)
echo "$sig" >> .slice/errors.log
[ "$n" -ge 1 ] && echo "same error seen $((n+1)) times — stop and report what is missing from the slice card" >&2 && exit 2
```

去掉數字再 hash，行號變了也算同一錯。第二次直接把訊息塞回 agent 要它回報缺什麼，不是再試。

**PR 形狀 → CI 自動標籤**：

```text
files_changed  vs  plan.files_*    不一致 → label: scope-drift
loc_changed    vs  中位數 × 3      超過   → label: oversized
touches deny                        → label: frozen-touched，擋合併
```

### 三步串起來的資料流

```text
feature.yaml ──lint_feature──▶ 綠
     │
     ├─slice_card──▶ card.yaml（人填 interface / acceptance）
     │                    │
     │            agent 讀碼 → plan.yaml ──check_plan──▶ 紅/黃給人，綠放行
     │                    │
     │            執行：guard_scope(hook) 擋越界
     │                    loop_guard(hook) 擋同錯
     │                    check_test_integrity(pre-commit) 擋改測試
     │                    │
     └────────────▶ PR ──CI 標籤──▶ 步驟 4
```

人的輸入只有三處：feature.yaml、card.yaml 的兩個欄位、標紅標黃的裁決。

## 步驟 4：審查

**進入**：CI 綠的 PR。
**動作**：自動審查先跑（第二模型、合約檢查）→ agent 回應 → 人看三件事。

| 判斷點 | 角度 | 判斷方式 |
|---|---|---|
| 它怎麼回應自動審查 | M | 每條意見分類：修了（diff 有對應變更）／解釋掉（只有文字）／繞過（加豁免、放寬型別、改測試）。解釋掉的人判斷成不成立；繞過的直接退 |
| 有沒有碰不准動清單 | M | grep diff 檔案路徑對清單，機械化 |
| 方向對不對 | A + B | 不逐行讀。看檔案清單跟方案一致嗎、新增公開介面跟片卡一致嗎、diff 大小跟片的預期成比例嗎 |

人不做的事：逐行看邏輯。那是測試和第二模型的工作。

**退出**：三項通過 → 合併。任一不過 → 退回 agent 附具體理由，不自己修（自己修會讓回顧抓不到模式）。

## 步驟 5：發佈與放量

**進入**：合併到 main，旗標關。
**動作**：自動 pre-release。整波到齊後內部開 → 分階段放量 → 刪旗標。

| 判斷點 | 角度 | 判斷方式 |
|---|---|---|
| 可以開內部旗標了嗎 | O | 功能卡 `done_when` 的 verify 內部走一次通過 |
| 放量到下一階段 | O + B | 前一階段錯誤率、關鍵指標沒偏離基線。基線在開旗標前就寫下，不是看到數字再決定 |
| 出問題：關旗標還是 revert | R | 永遠先關旗標。revert 只在旗標沒包住問題時（代表切片時 R 判斷錯了，記進回顧） |
| 刪旗標 | B | 100% 放量穩定一個週期後。刪旗標是一片，走步驟 2–4 |

**退出**：旗標刪除，功能卡標完成。

## 步驟 6：回顧

**片級（即時）**

| 觸發 | 動作 | 寫到哪 |
|---|---|---|
| C1 抓到改測試／加豁免 | 加一條明確禁止 | 指示檔 |
| C3 發生 | 切片改法記一條 | 功能卡附註 |
| 步驟 4 「繞過」 | 加 lint rule 或 CI check | CI |
| 同類 bug 第二次 | 補行為測試 | 測試 |

能寫成機械檢查的不寫成文字指示。文字指示 agent 會忘，CI 不會。

**功能級（結束時一次）**

| 看什麼 | 角度 | 產出 |
|---|---|---|
| 哪種退回重複出現 | M | 片卡模板加固定段落 |
| 哪個判準沒接住 | O | 步驟 0 清單加項目 |
| 切片估錯在哪 | B | 下個功能的切片順序調整 |
| revert 過嗎 | R | 有 → 切片時 R 判斷的規則要改 |

## 機械化不了的判斷

| 判斷 | 為什麼 | 折衷 |
|---|---|---|
| 切片順序 R 角度對不對 | 需要知道退回代價，腳本不知道業務 | gate 分類逼人選，選了就有預設可逆性 |
| deps_found 在 deny 裡該怎麼辦 | 是 agent 的洞見還是想繞路 | 標紅，一律人看 |
| 測試「測行為」 | entry + mock 數只是代理指標 | 兩個都過再加人抽 1/5 |
| 片太大要不要切 | 5 個檔是經驗值 | 標黃不擋，回顧調閾值 |

## 限制與未驗證

- 流程和腳本規格是設計，尚未在真實專案跑過；閾值（5 個檔、mock > 2、LOC 中位數 × 3）是起始值，靠步驟 6 調。
- hook 片段以 Claude Code 的 PreToolUse / PostToolUse 介面為準，其他 harness 要換對應機制。
- C2 的錯誤簽章用「去數字後 hash 前三行」是粗略啟發式，不同語言的錯誤格式可能要調 grep pattern。
