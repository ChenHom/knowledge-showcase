# Laravel 專案 403：Apache 使用者與目錄穿越權限

#laravel #apache #macos #filesystem-permissions

權限錯誤的問題總結如下：

---

### 問題描述
訪問 Laravel 專案時，伺服器返回 `403 Forbidden` 錯誤，Apache 錯誤日誌顯示：
```plaintext
Permission denied: access to / denied (filesystem path '/Users/hom/code')
```
此錯誤是由於 Apache 使用者 `daemon` 無法存取專案所在目錄 `/Users/hom/code` 及其上層目錄，缺少目錄的「搜尋權限」（執行權限 `x`）。

---

### 解決方案
1. **確認 Apache 使用者**：
   使用 `ps aux | grep httpd` 確認 Apache 的執行使用者為 `daemon`。

2. **確認目錄權限與群組**：
   - 使用 `groups hom` 確認 `hom` 使用者的群組為 `staff`。
   - 使用 `ls -ld` 檢查專案目錄及其上層目錄的權限。

3. **將 `daemon` 加入 `staff` 群組**：
   - 使用以下命令將 `daemon` 加入 `staff` 群組：
     ```bash
     sudo dseditgroup -o edit -a daemon -t user staff
     ```
   - 確認 `daemon` 已成功加入：
     ```bash
     groups daemon
     ```

4. **只調整必要目錄的穿越權限**：
   先逐層檢查 `/Users`、`/Users/hom`、`/Users/hom/code`。目錄需要 `x` 才能穿越，不要對整個程式碼目錄遞迴套用 `770`：
   ```bash
   chmod g+x /Users/hom
   chmod g+x /Users/hom/code
   ```

   專案內檔案與目錄權限應分開處理。只有 Laravel 需要寫入的 `storage/`、`bootstrap/cache/` 才授予寫入權限。

5. **重啟 Apache**：
   變更完成後，重新啟動 Apache：
   ```bash
   sudo apachectl restart
   ```

---

### 結果
完成後，`daemon` 可以穿越上層目錄並讀取網站檔案；Laravel 可寫目錄則維持最小必要權限。

## 限制

把 Web Server 使用者加入開發者主要群組會擴大可讀範圍。正式環境應改用獨立部署群組、明確 owner/group 與最小權限，不應直接沿用本機開發設定。

---
