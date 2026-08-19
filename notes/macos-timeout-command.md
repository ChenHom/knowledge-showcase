# 在 macOS 中使用 timeout 指令：解決無限迴圈問題

這篇文章主要討論如何在 Bash 腳本中使用 `timeout` 指令來防止無限迴圈的問題。作者在工作中遇到一個腳本，它會持續檢查 Web 伺服器是否啟動，但當伺服器在啟動過程中崩潰時，`until` 迴圈就會無限執行。`timeout` 指令可以設定時間限制，讓超時的指令被終止，但它無法直接與 `until` 搭配使用，因為 `until` 是一個 Shell 關鍵字，而不是可以被 `SIGTERM` 終止的指令。

為了解決這個問題，作者提供了兩種解決方案：
1. **使用 `bash` 進行包裝**：
   ```bash
   timeout 1m bash -c "until curl --silent --fail-with-body 10.0.0.1:8080/health; do sleep 1; done"
   ```
2. **將 `until` 迴圈移到一個獨立的 Bash 腳本**：
   ```bash
   timeout 1m ./until.sh
   ```

### 在 macOS 上使用 `timeout`
macOS 並未內建 `timeout` 指令，但可以使用 **Homebrew** 安裝 `coreutils` 來取得 `timeout`：
```bash
brew install coreutils
```
安裝完成後，`timeout` 指令名稱變成 `gtimeout`，你可以使用以下指令讓它與 Linux 的 `timeout` 一致：
```bash
alias timeout=gtimeout
```
或者，建立符號連結：
```bash
ln -s /usr/local/bin/gtimeout /usr/local/bin/timeout
```

### macOS 使用範例
當 `timeout` 設定超過 3 秒時，就會終止 `sleep`：
```bash
timeout 3s sleep 10
echo $?  # 顯示非零的返回值
```

### 小結
- `timeout` 指令可用來限制某些指令的執行時間。
- 它可以與 `bash -c` 方式包裝 `until` 迴圈來達到超時控制。
- macOS 系統需透過 `coreutils` 套件安裝 `timeout`，並使用 `gtimeout` 來替代。


[source](https://heitorpb.github.io/bla/timeout)
