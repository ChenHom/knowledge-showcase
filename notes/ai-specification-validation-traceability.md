# 驗證與追蹤：在開發前發現規格錯誤

#specification-validation #decision-table #property-based-testing #traceability #architecture-review

## 這是什麼

規格工程的價值不在文件數量，而在規格能否被攻擊、被檢查、被重播。驗證要在開發前找出矛盾、缺口、不可達狀態與錯誤優先順序。

## 核心結論

每條重要規則至少需要三種證據：來源證據、Given／When／Then 或 Decision Table 的行為證據，以及 schema validation、狀態機測試、property test 或 model simulation 的執行證據。

## 驗證層次

### 靜態一致性

- 每個規格 ID 唯一。
- 每個狀態轉移引用已定義的命令、事件與錯誤。
- 每條不變量至少被一個情境覆蓋。
- 已刪除欄位未被其他文件引用。
- `open_questions` 不可被實作當成已確認規則。

### Decision Table

| Rule | 訂單可付款 | 金額相符 | 幣種相符 | 結果 |
|---|---:|---:|---:|---|
| R1 | 是 | 是 | 是 | 確認付款 |
| R2 | 否 | - | - | ORDER_NOT_PAYABLE |
| R3 | 是 | 否 | 是 | PAYMENT_AMOUNT_MISMATCH |
| R4 | 是 | 是 | 否 | PAYMENT_CURRENCY_MISMATCH |

`-` 代表該條件在前一條件失敗後不再影響結果。這同時定義錯誤優先順序。

### 狀態可達性

檢查是否存在永遠到不了的狀態、離不開的非終止狀態、錯誤終止狀態，以及重複或併發命令如何處理。

### Property-based Testing

```php
it('任何命令序列都不會讓已付金額為負', function (array $commands) {
    $order = runModel($commands);

    expect($order->paidAmount()->minorUnits)->toBeGreaterThanOrEqual(0);
})->with('generated command sequences');
```

### 反例與失敗注入

要求 AI 攻擊重複 callback、寫入成功但事件發布失敗、外部逾時後其實成功、跨時區、併發更新、舊資料違反新規則與流程中途撤銷權限。AI 提出的反例仍需人工判定是否屬於系統承諾。

## 追蹤矩陣

| Spec ID | Invariant | Scenario | Test | Decision |
|---|---|---|---|---|
| ORDER-PAY-001 | INV-ORDER-002 | PAY-001、PAY-004 | OrderPaymentSpecTest | ADR-001 |

追蹤矩陣用來回答變更衝擊。當不變量改變時，可以明確找到受影響的情境、測試與架構決策。

## AI 審查 Prompt

```text
你是敵對式規格審查者。只能使用 repository 內已確認規則。
找出：
1. 互相矛盾的前置條件與後置條件。
2. 沒有任何情境覆蓋的不變量。
3. 沒有合法入口或出口的狀態。
4. 未定義的錯誤優先順序。
5. 被當成事實使用的 open question。
6. 無法追蹤來源的欄位與資料表。
每項問題必須引用規格 ID，給出最小反例，不可自行補需求。
```

## 核可門檻

- 詞彙表無同義詞混用。
- 核心正常與拒絕路徑完整。
- 不變量與錯誤優先順序由領域人員核可。
- 高風險狀態轉移有可執行驗證。
- 待決問題有負責人，不會被默默實作。
- 模型已收斂到有證據支持的最小集合。
- 規格變更能透過 CI 重新執行檢查。

更新日期：2026-07-21
