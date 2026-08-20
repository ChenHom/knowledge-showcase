# tw-day-trading MCP：受控 CLI Wrapper 與 OpenClaw 接入紀錄

#tw-day-trading #MCP #OpenClaw #CLI-wrapper #交易系統 #AI協作

## 這篇文章在講什麼

這篇是 `tw-day-trading` 專案第一階 MCP 化的沉澱紀錄。重點不是重寫交易系統，而是把既有 CLI 包成一個受控 MCP 入口，讓 AI 可以查詢、對帳、整理報告與手動回填成交，同時避免任意 shell、危險重置、長任務或自動下單能力外露。

## 新手先記住這 5 件事

1. **MCP 是入口層，不是交易邏輯層**：實際交易邏輯仍由專案既有 CLI 處理，MCP 只負責把工具呼叫轉成安全 argv。
2. **allowlist 是安全邊界**：第一版只允許查詢、dry-run、報告、對帳，以及明確指定的 `record_fill`。
3. **record-fill 是成交事實回填，不是下單**：它記錄外部券商已發生的成交，預設帳號是「國泰」。
4. **summary/data 是 AI 使用體驗層**：保留原始 `stdout/stderr/argv/exit_code`，但常用輸出額外整理成結構化資料。
5. **查資料必須走 MCP 入口**：後續 agent 不應直接手打專案 CLI 查資料；即使 MCP 底層是 CLI wrapper，操作入口仍必須是 MCP。

## 系統定位

這次 MCP 的核心定位是「有安全欄杆的 CLI 遙控層」：

```text
AI / OpenClaw
  -> MCP tool call
  -> tw-day-trading MCP server
  -> static allowlist
  -> argv builder
  -> subprocess.run(argv, cwd=project root)
  -> stdout / stderr / exit_code / argv
  -> optional summary / data
```

它不做這些事：

- 不直接下單。
- 不重寫策略、帳務、回測或對帳邏輯。
- 不接受任意 shell command。
- 不在 P0/P2 階段開啟長任務或危險 mutation。

## 已完成的階段

### P0：專案內 MCP server 與受控 CLI wrapper

P0 建立最小可用架構：

- `src/mcp/server.py`：MCP tools 入口。
- `src/mcp/cli_registry.py`：P0 command allowlist 與 argv builder。
- `src/mcp/cli_runner.py`：`subprocess.run` wrapper。
- `tests/unit/test_mcp_cli.py`：allowlist、argv、record_fill、runner 測試。
- `requirements-mcp.txt`：MCP SDK 依賴。

P0 tools：

- `list_cli_capabilities`
- `run_cli`
- `record_fill`

`run_cli` 只允許白名單內的查詢或 dry-run 指令，例如：

- `approval list`
- `signal list`
- `portfolio reconcile`
- `report pnl`
- `report daily`
- `corporate-action list/check`

`record_fill` 對應既有 CLI：

```bash
python3 -m app trade record-fill ...
```

但 MCP 介面會把帳號預設為「國泰」，並要求最基本成交欄位：

- `symbol`
- `side`
- `quantity`
- `price`

### P1：接入 OpenClaw MCP client

OpenClaw managed MCP server：

```text
name: tw-day-trading-cli
command: /home/hom/services/stock/tw-day-trading/.venv/bin/python
args: -m src.mcp.server
cwd: /home/hom/services/stock/tw-day-trading
tools: list_cli_capabilities, run_cli, record_fill
```

驗證方式：

```bash
openclaw mcp probe tw-day-trading-cli --json
```

probe 看到三個 tools：

- `tw-day-trading-cli__list_cli_capabilities`
- `tw-day-trading-cli__run_cli`
- `tw-day-trading-cli__record_fill`

注意：正在運行中的 Codex tool registry 不一定會熱載新 MCP；需要下一個 runtime build 才會直接出現在工具搜尋裡。

### P2：常用輸出整理成 summary/data

P2 沒有改變 CLI 行為，只是在 MCP 回傳中增加 AI 更好使用的欄位。

所有結果仍保留：

- `stdout`
- `stderr`
- `exit_code`
- `argv`

額外增加：

- `summary`：一句人類可讀摘要。
- `data`：結構化資料。

目前解析三類常用輸出：

1. `report pnl` + `by_strategy=true`
   - 帳號
   - 日期
   - 現金
   - 策略分組
   - 持倉明細
   - 長期標記
   - 未實現損益

2. `signal list`
   - 訊號數量
   - 表格列資料

3. `portfolio reconcile`
   - 帳號
   - reconcile 成功狀態

`record_fill` 也回傳 normalized data：

- account
- symbol
- side
- quantity
- price
- strategy_id
- long_term
- date

## 被明確延後的階段

### P3：低風險寫入 tools

只記錄，不實作。

候選項目：

- `reject_signal`
- `un_reject_signal`
- `set_long_term`
- corporate-action record/apply

原則：等真的需要時，優先做專用 tool，不要急著做通用 mutation runner。

### P4：長任務 operator tools

只記錄，不實作。

候選項目：

- `simulation run-daily`
- `market sync`
- `market backfill`
- `backtest run`
- `market build-universe`

原則：這類工具需要 timeout、log path、重複執行保護與 operator intent，不應混進查詢工具。

## 本輪重要修正

實作過程中也修掉一個既有 bug：

`approval list` 會因為 strategy manifest 的 `expires_at` 是 date-only，而 current time 是 timezone-aware，導致 Python 比較時噴 `TypeError`。

修正方向：

- date-only manifest time 若遇到 aware current time，補上同樣 tzinfo。
- 補單元測試覆蓋 date-only expiry。

## 驗證紀錄

本輪驗證包含：

```bash
pytest tests/unit/test_mcp_cli.py tests/unit/test_manifest_validator.py -q
```

結果：18 passed。

```bash
python -m py_compile src/mcp/*.py src/approval/validator.py tests/unit/test_mcp_cli.py tests/unit/test_manifest_validator.py
```

結果：passed。

```bash
pytest tests/ -q
```

結果：344 passed, 1 warning。

MCP client probe：

```bash
openclaw mcp probe tw-day-trading-cli --json
```

結果：tools=3，diagnostics empty。

## 可重用設計原則

這次設計有幾個可重用原則：

1. **先包現有 CLI，不急著 import 內部 handler**
   - 對既有專案風險最低。
   - CLI contract 已經是人類熟悉的操作界面。

2. **先做 allowlist，不做通用 command runner**
   - 避免 AI 可以組出任意危險指令。
   - 新能力需要明確加進 registry。

3. **常用輸出再解析，不預付完整 parser 成本**
   - 先保留 stdout。
   - 只對高頻查詢補 `summary/data`。

4. **寫入工具用專用 tool，不做大而全 mutation 平台**
   - `record_fill` 是明確需求，所以獨立。
   - 其他寫入等 P3 再逐一討論。

5. **長任務獨立成 operator layer**
   - `run-daily`、backtest、market sync 都不是普通查詢。
   - 需要不同 timeout、log、狀態與重入保護。

## 下次入口

確認 MCP 是否可連：

```bash
openclaw mcp probe tw-day-trading-cli --json
```

查國泰損益應從 MCP tool 入口：

```json
{
  "command": "report pnl",
  "args": {
    "account": "國泰",
    "by_strategy": true
  }
}
```

手動回填成交應用 `record_fill` tool，不要直接手打 CLI：

```json
{
  "symbol": "2330",
  "side": "BUY",
  "quantity": 10,
  "price": 1000,
  "strategy_id": "trend_breakout"
}
```

未指定 `account` 時，MCP 會預設為「國泰」。

## 後續沉澱規則

這次起，工程任務 close-out 的沉澱若具有可回查價值，除了寫入專案 docs、workspace memory 或 `.learnings/`，也要同步整理一份到 `know`。

`know` 版本應該是可獨立閱讀的知識文章，不只是聊天紀錄；內容要避開秘密、token、private key 與不適合公開的帳務細節。
