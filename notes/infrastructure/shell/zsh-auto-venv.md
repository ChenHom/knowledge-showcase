# Zsh：進入專案自動載入 `.venv`

這裡放的是純 `zsh` hook 版，不依賴 `direnv`。

## 目的

當你 `cd` 進某個專案資料夾時，如果該目錄或其上層有：

```bash
.venv/bin/activate
```

就自動啟用該 Python 虛擬環境；離開那個專案樹時則自動 `deactivate`。

## 使用方式

把腳本放到你慣用的位置，然後在 `~/.zshrc` 只保留一行 `source`：

```zsh
[[ -f "$HOME/.config/zsh/auto-venv.zsh" ]] && source "$HOME/.config/zsh/auto-venv.zsh"
```

腳本本體可直接用這份：

- `cli/zsh/auto-venv.zsh`

## 行為特性

- 會往目前目錄一路往上找 `.venv/bin/activate`
- 找到後自動啟用
- 離開原本專案樹時自動停用
- 若已在正確的 venv 內，不會重複 activate
- 若切到另一個有 `.venv` 的專案，會自動切換

## 適合情境

- 多個 Python 專案切來切去
- 不想安裝額外工具
- 想把 `~/.zshrc` 保持乾淨，只 `source` 外部檔案
