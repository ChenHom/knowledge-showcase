# Antigravity CLI 內容處理器：agy-content 的設計與封裝原則

整理日期：2026-06-27

這份筆記整理一次把 Antigravity CLI (`agy`) 包成內容處理工具的討論與實作重點。目標不是打造完整 agent，也不是把 OpenClaw 直接改成 Antigravity runtime，而是先做一個小而穩的 CLI wrapper，讓翻譯、總結、創意發想、潤稿這類低到中風險內容任務可以快速交給 `agy` 處理。

## 核心結論

`agy-content` 最適合先定位成「內容處理 CLI」，不是 agent plugin。

它應該做四件事：

1. 統一呼叫 `agy --print`
2. 根據任務和風險選模型階層
3. 讀取可替換的 style profile
4. 用固定模板產生可檢查、可重跑的 prompt

它不應該做這些事：

1. 自動 fallback 到昂貴雲端模型
2. 預設使用 dangerous permission
3. 讀取整個 workspace 或代理大型 coding 任務
4. 把私人語言偏好硬包進可分享版本

## 背景判斷

目前本機可直接使用：

```bash
/home/hom/.local/bin/agy
```

`agy --print` 可以做非互動的一次性內容處理。OpenClaw 文件和程式碼裡已有 `google-antigravity` 的支援痕跡，也可能在影像 / 影片理解 fallback 場景使用 `agy`，但 ACP harness target 並未列出 `antigravity`。因此現階段應視為：

- 可以直接用 `agy` CLI 做內容任務
- 可以包成外部 CLI wrapper
- 不應假設可用 `/acp spawn antigravity` 跑完整 coding agent

## 設計邊界

### Skill、Cheat、Profile 的分工

這次討論後，最好拆成三層：

| 層級 | 放什麼 | 是否可分享 |
| --- | --- | --- |
| Skill | 使用流程、任務模板、模型分級、驗證方式 | 可以 |
| Cheat / Quick Card | 常用命令、快速例子、注意事項 | 可以 |
| Style Profile | 個人語氣、用詞偏好、翻譯習慣 | 視內容而定，私人 profile 不直接分享 |

可分享 skill 應該描述「怎麼做」：

- 如何呼叫 `agy`
- 支援哪些任務
- tier 如何選模型
- template 如何插入 `{{STYLE_PROFILE}}` 和 `{{INPUT}}`
- 如何 dry-run、show-prompt、做 smoke test
- 安全邊界是什麼

私人 profile 則保存「要寫成什麼樣子」：

- 正體中文
- 台灣用語
- 特定禁用詞
- 語氣偏好
- 翻譯、總結、潤稿規則

這樣其他 AI 系統要用時，只要換自己的 style profile，就能沿用同一套工具與模板。

## MVP CLI 介面

目前 MVP 指令：

```bash
bin/agy-content translate article.txt
bin/agy-content translate --text 'English text'
printf '通知系統可以怎麼做' | bin/agy-content ideate --dry-run
bin/agy-content polish draft.md --tier high --save outputs/agy/polished.md
bin/agy-content summarize article.txt --show-prompt
```

支援任務：

- `translate`：翻譯成自然正體中文
- `summarize`：先結論，再整理重點與注意事項
- `ideate`：產生實用方案、取捨與下一步
- `polish`：依 style profile 潤稿、改寫

支援輸入：

- `--text`
- file path
- stdin

支援控制：

- `--tier auto|low|medium|high`
- `--model`
- `--save`
- `--timeout`
- `--show-prompt`
- `--dry-run`

## 模型分級

模型分級不要只看模型強弱，也要看任務風險。

| Tier | 預設模型 | 適合任務 |
| --- | --- | --- |
| low | `Gemini 3.5 Flash (Low)` | 短文翻譯、低風險摘要、簡單潤稿、格式整理 |
| medium | `Gemini 3.5 Flash (Medium)` | 長文翻譯、文章總結、一般創意發想、技術說明潤稿 |
| high | `Gemini 3.1 Pro (High)` | 公開文案、求職內容、商業 / 策略內容、決策摘要、高風險翻譯 |
| ultra | `Claude Opus 4.6 (Thinking)` | 預留，不預設啟用 |

`auto` 的初版規則應保持簡單可解釋：

- 短文、低風險：`low`
- 長文、結構敏感、技術內容：`medium`
- 公開、求職、商業、法律、金融、決策支援：`high`
- 手動指定 `--tier` 或 `--model` 永遠優先

## Template 設計

每個任務模板都應由四段組成：

1. 任務角色
2. style profile
3. 任務規則
4. 使用者輸入

概念如下：

```text
You are translating content for Boss.

Follow this Traditional Chinese style profile:
{{STYLE_PROFILE}}

Task:
Translate the source text into natural Traditional Chinese.

Rules:
- Be faithful to the source meaning.
- Preserve structure when possible.
- Avoid translationese.
- Output only the translation.

Source text:
{{INPUT}}
```

這裡最重要的是讓 style profile 可替換，而不是把個人偏好寫死在模板裡。

## Style Profile 維護規則

style profile 是輸出品質的中心。

在這次實作中，profile 放在：

```text
references/style/boss_zh_tw_style.md
```

它負責保存：

- 核心語言：正體中文、台灣用語、保留必要英文術語
- 語氣：直接、有效率、少廢話，但要保留溫度
- 互動風格：先結論，再重點
- 避免句型：例如固定 AI 反差句
- 用詞偏好表
- 翻譯、總結、創意發想、潤稿規則

更新判斷很簡單：

```text
A(X) B(O)
不要用 A，改用 B
以後寫成 B
任何直接修正用詞、語氣、翻譯習慣的句子
```

遇到這些訊號，就應更新 profile，而不是散落到對話、記憶或臨時 prompt。

例子：

```text
體現(X) 呈現、展現(O)
```

應寫成：

```text
Avoid: 體現
Prefer: 呈現 / 展現
```

## 實作原則

這次 MVP 採用最小可行設計：

- 單檔 Python CLI：`bin/agy-content`
- 使用 Python 標準庫，不新增依賴
- 模板放在 `references/agy-content/templates/`
- 模型分級放在 `references/agy-content/models.json`
- style profile 獨立
- 真正執行時才呼叫 `agy`
- dry-run 和 show-prompt 可在不花模型成本的情況下檢查行為

這符合幾個工程原則：

- 單一職責：CLI 負責輸入、路由、組 prompt、呼叫 `agy`
- 開放封閉：新增模板或改模型分級時改檔案，不改主流程
- 依賴倒置的簡化版：個人風格透過 profile 注入，不硬寫在程式裡
- 最小實作：先能用、能測、能重跑，再談 plugin 化

## 驗證方式

MVP 應至少通過這些檢查：

```bash
python3 -m unittest tests.test_agy_content
python3 -m py_compile bin/agy-content tests/test_agy_content.py
bin/agy-content translate --text 'Artificial intelligence helps software teams review code.' --dry-run
bin/agy-content polish --text '這個功能可以體現產品價值' --show-prompt
printf '通知系統可以怎麼做' | bin/agy-content ideate --dry-run
```

再跑一次真實 `agy` smoke：

```bash
bin/agy-content translate \
  --timeout 60s \
  --text 'Artificial intelligence helps software teams review code, test ideas quickly, and reduce repetitive work.'
```

可接受輸出例：

```text
人工智慧能協助軟體團隊審查程式碼、快速測試想法，並減少重複性工作。
```

## 可分享化建議

穩定後可以抽成一個 shareable skill：

```text
agy-content-skill/
  SKILL.md
  scripts/agy-content
  references/templates/*.md
  references/models.json
  references/style-profile.example.md
  evals/smoke_cases.md
  CHEATSHEET.md
```

`SKILL.md` 應寫：

- 何時使用
- 何時不要使用
- CLI 使用步驟
- tier routing 規則
- 模板與 profile 的關係
- 驗證方式
- 安全邊界

`CHEATSHEET.md` 則只放短版命令：

```bash
agy-content translate article.md --tier auto
agy-content summarize article.md --tier medium
agy-content polish draft.md --tier high
agy-content translate article.md --show-prompt
```

## 後續路線

建議順序：

1. 保留 workspace 版，實際用幾次
2. 依真實使用修正 templates
3. 把私人 style profile 與 shareable skill 分離
4. 補 `CHEATSHEET.md`
5. 再整理成可安裝 / 可分享的 skill
6. 最後才考慮 OpenClaw 自動選用

先不要急著做 OpenClaw plugin。現階段價值在於：快速、可控、低成本地處理內容任務。
