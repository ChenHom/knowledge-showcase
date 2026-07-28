# 完整操作範例：以訂單與付款流程建立規格

#specification-example #order-domain #payment-domain #ai-workflow #decision-table

## 這是什麼

這份範例示範如何把一句模糊需求，透過 AI 發散、人工追問、模型收斂與可執行驗證，轉成可交付的規格資產。

原始需求：商戶可以建立訂單。顧客付款成功後訂單改為已付款，失敗可以重試，系統要保留紀錄。

## 第一步：禁止 AI 直接設計資料庫

先列出名詞、動作、規則與未知項目。未知包含：一張訂單可有幾次付款嘗試、是否允許部分付款、付款成功後能否取消、重試是否沿用識別碼，以及「紀錄」用於稽核、對帳或客服查詢。此時建立 `open_questions`，不讓 AI 用一般電商慣例自行補答案。

## 第二步：取得領域答案

假設人工確認：一張訂單可有多次付款嘗試但只能成功一次；不允許部分付款；每次重試都有新的 `payment_attempt_id`；成功 callback 可能重複且必須冪等；紀錄需支援客服與對帳；已付款取消屬退款流程，不是狀態倒退。

## 第三步：產生候選再收斂

AI 第一版可能產生十五張表。不要逐張修欄位，先依已確認規則分類。

| 概念 | 決定 | 原因 |
|---|---|---|
| Merchant | 保留 | 訂單所有者，有獨立生命週期 |
| Customer | 延後／外部識別 | 沒有管理顧客生命週期的需求 |
| Order | 保留 aggregate | 維護只能成功付款一次等不變量 |
| PaymentAttempt | 保留 entity | 每次重試有獨立識別、結果與對帳資料 |
| StatusHistory | 刪除 | 先由 domain event log 滿足追蹤 |
| PaymentCallback | 併入收件紀錄 | 用外部事件 ID 唯一鍵處理冪等 |
| Currency | 值／設定 | 不維護交易生命週期 |
| Refund | 延後 | 已知存在，但不在本次交付範圍 |

## 第四步：寫出規格與不變量

```yaml
id: ORDER-PAY-001
title: 接受付款成功通知
preconditions:
  - order.status == submitted
  - callback.amount == order.payable_amount
  - callback.currency == order.currency
command:
  name: ConfirmPaymentAttempt
idempotency_key: gateway_event_id
postconditions:
  - attempt.status == succeeded
  - order.status == paid
events:
  - PaymentAttemptSucceeded
  - OrderPaid
```

```text
INV-PAY-001：同一訂單最多只有一個 succeeded payment attempt。
INV-PAY-002：成功付款的金額與幣種必須等於訂單應付金額與幣種。
INV-PAY-003：相同 gateway_event_id 不得重複改變狀態。
```

## 第五步：建立 Decision Table

| Rule | event 已處理 | 訂單可付款 | 金額相符 | 幣種相符 | 動作 |
|---|---:|---:|---:|---:|---|
| R1 | 是 | - | - | - | 回傳既有結果，不新增事件 |
| R2 | 否 | 否 | - | - | ORDER_NOT_PAYABLE |
| R3 | 否 | 是 | 否 | - | PAYMENT_AMOUNT_MISMATCH |
| R4 | 否 | 是 | 是 | 否 | PAYMENT_CURRENCY_MISMATCH |
| R5 | 否 | 是 | 是 | 是 | 標記成功並發布事件 |

表格揭露：冪等檢查必須先於狀態檢查，否則相同成功 callback 第二次抵達時，會因訂單已 Paid 而錯誤回覆 `ORDER_NOT_PAYABLE`。

## 第六步：用反例攻擊

```text
使用重複 callback、兩次付款同時成功、資料庫寫入後發布事件失敗、
外部逾時後重送、舊資料缺少 currency 五種反例攻擊 ORDER-PAY-001。
只能指出規格缺口，不可自行決定業務答案。
```

可能找到：需要 aggregate version 或唯一約束、transactional outbox、舊資料遷移決策，以及 callback 收件與業務處理分離。這些結果必須回寫規格與 ADR。

## 第七步：產生可執行驗證

```php
it('重複成功通知不會產生第二次付款事件', function () {
    $first = confirmPayment(eventId: 'evt-001');
    $second = confirmPayment(eventId: 'evt-001');

    expect($second)->toEqual($first)
        ->and(recordedEvents(OrderPaid::class))->toHaveCount(1);
});
```

測試語法只是示意。真正的驗證重點是測到不變量、冪等與競爭條件，不是只檢查 HTTP 200。

## 第八步：完成追蹤

```text
REQ-ORDER-007
  → ORDER-PAY-001
  → INV-PAY-001 / 002 / 003
  → ADR-001 payment boundary
  → PAY-001 / PAY-002 / PAY-003
  → OrderPaymentSpecificationTest
```

AI 的作用不是替團隊畫 ERD，而是把假設攤開、用反例破壞模型、產生可執行檢查。最後的業務決定仍由人負責。

更新日期：2026-07-21
