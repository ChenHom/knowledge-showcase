# 高效樂透比對系統：使用 Redis 的注單匹配與數據管理

### English Prompt

```
Build a highly efficient lottery system for comparing millions of lottery tickets with the winning numbers using Redis. The solution should meet the following requirements:

1. **Generate Random Lottery Tickets**:
   - Generate tickets with the following rules:
     - Each ticket contains exactly 6 unique numbers.
     - Numbers range from 1 to 49.
     - Convert each ticket into a bitmask representation, where each bit corresponds to a number (e.g., bit 0 for number 1, bit 1 for number 2, etc.).
     - Store the tickets in Redis with keys formatted as `bet:<id>`.

2. **Save Winning Numbers**:
   - Store the winning numbers as a bitmask in Redis under a specific key, e.g., `"draw:current"`.

3. **Perform Ticket Matching on Redis**:
   - Use Redis commands to perform ticket matching directly on the server:
     - Use `BITOP AND` to calculate the intersection of ticket numbers and winning numbers.
     - Use `BITCOUNT` to determine the number of matched numbers for each ticket.
   - Ensure all computations are handled server-side to minimize data transfer between PHP and Redis.

4. **Batch Processing with Redis Pipeline**:
   - Use Redis Pipeline to batch commands, including:
     - Ticket matching with `BITOP`.
     - Counting matched numbers with `BITCOUNT`.
     - Cleaning up temporary matching results.
   - This reduces network overhead and improves system performance.

5. **Group and Output Results**:
   - Group tickets by the number of matched numbers (e.g., tickets with 2, 3, 4, or more matches).
   - Output detailed results, including:
     - Ticket IDs.
     - Matched numbers for each ticket.
     - Statistical summaries for each group (e.g., total tickets with 2 matches).

6. **Clean Up Data**:
   - Include functionality to delete all tickets and temporary matching results from Redis after processing.

7. **Optimize for Production Use**:
   - Authentication: Ensure Redis access is secured with a password.
   - Memory Management: Use efficient Redis data structures (e.g., `Hash` or bitmaps) to reduce memory usage.
   - Scalability: Support Redis Cluster or Sentinel for distributed or high-availability setups.
   - Logging and Error Handling: Integrate robust logging (e.g., Monolog) and error handling to monitor and debug the system.
   - Performance Testing: Perform load testing to ensure the system can handle high concurrency and large datasets.

8. **Additional Notes**:
   - Provide a PHP implementation that demonstrates the full process:
     - Generating and storing random lottery tickets.
     - Storing and processing winning numbers.
     - Matching tickets and generating results in Redis.
     - Cleaning up after processing.
   - The system must be efficient and capable of handling millions of tickets within a reasonable time frame.
```

建立一個有效率的樂透系統，使用 Redis 進行數百萬注樂透注單與中獎號碼的比對。解決方案應滿足以下需求：

----------

1. **生成隨機樂透注單**：
   - 按以下規則生成樂透注單：
     - 每注包含 **6 個不重複的號碼**。
     - 號碼範圍為 **1 到 49**。
     - 將每注轉換為位元掩碼（`bitmask`）表示，其中每個位元對應一個號碼（例如，位元 0 表示號碼 1，位元 1 表示號碼 2，以此類推）。
     - 將注單存入 Redis，鍵名格式為 `bet:<id>`。

2. **儲存中獎號碼**：
   - 將中獎號碼轉換為位元掩碼，並儲存到 Redis 中，使用固定的鍵名（例如 `"draw:current"`）。

3. **在 Redis 伺服器上完成注單比對**：
   - 使用 Redis 指令完成伺服器端的比對計算：
     - 使用 `BITOP AND` 計算注單號碼與中獎號碼的交集。
     - 使用 `BITCOUNT` 計算每注匹配的號碼數量。
   - 確保所有計算在 Redis 伺服器內完成，減少 PHP 與 Redis 之間的數據傳輸。

4. **批量處理與 Redis Pipeline**：
   - 使用 Redis Pipeline 批量執行命令，包括：
     - 注單比對的 `BITOP` 操作。
     - 計算匹配數量的 `BITCOUNT` 操作。
     - 清理臨時比對結果的 `DEL` 操作。
   - 減少網路延遲並提升整體系統性能。

5. **分組與結果輸出**：
   - 根據匹配的號碼數量分組統計注單（例如，中 2 個號碼、中 3 個號碼等）。
   - 輸出詳細結果，包括：
     - 每注中獎注單的 ID。
     - 匹配的號碼數量與詳細號碼列表。
     - 各分組的統計摘要（如中 2 個號碼的注數總和）。

6. **清理數據**：
   - 提供清理功能，刪除 Redis 中的所有注單資料與臨時比對結果，釋放 Redis 記憶體。

7. **優化至生產環境**：
   - **安全性**：啟用 Redis 認證功能，確保數據訪問安全。
   - **記憶體管理**：使用高效的 Redis 結構（例如 `Hash` 或 Bitmap）來減少記憶體消耗。
   - **可擴展性**：支持 Redis Cluster 或 Sentinel 部署，確保分佈式與高可用性。
   - **錯誤處理與日誌**：增加錯誤處理與日誌記錄功能，方便系統監控與排錯。
   - **性能測試**：在生產部署前進行壓力測試，模擬高併發場景，驗證系統性能。

8. **其他補充**：
   - 提供一個完整的 PHP 實作範例，示範以下過程：
     - 隨機生成與儲存注單。
     - 儲存中獎號碼。
     - 在 Redis 上比對注單並生成結果。
     - 清理 Redis 數據。
   - 確保系統能在合理的時間內處理數百萬筆注單。

---

### **功能重點**
1. **隨機注單生成與儲存**：
   - 根據規則生成注單，並轉換為位元掩碼後儲存至 Redis。

2. **Redis 層進行比對**：
   - 使用 `BITOP AND` 與 `BITCOUNT` 完成比對與數量統計。

3. **批量與高效處理**：
   - 使用 Pipeline 批量執行操作，減少網路延遲。

4. **結果輸出與分組統計**：
   - 提供詳細的中獎結果與統計資料。

5. **清理與記憶體釋放**：
   - 提供自動清理功能，避免佔用過多記憶體。

6. **生產環境優化**：
   - 加強安全性與可擴充性，適合高併發業務場景。
