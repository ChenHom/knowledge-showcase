# 規格即程式：用型別、狀態機與不變量定義需求

#specification-as-code #domain-modeling #state-machine #invariant #type-system

## 這是什麼

「用程式語言定義規格」不是提早寫正式系統，也不是把需求文件換成 class。它是用型別、狀態轉移、前置條件與不變量，把自然語言中含糊的規則改寫成機器可以檢查的模型。

## 核心結論

自然語言適合說明目的，結構化規格適合約束行為。兩者不能互相取代。一份有效的規格模型至少回答：系統有哪些穩定概念、合法狀態、命令、事件、不變量，以及失敗時如何處理。

## 五種核心構件

### Value Object

把「字串或數字」升級成有業務語意的值，並把合法範圍放進型別。

```php
final readonly class Money
{
    public function __construct(
        public int $minorUnits,
        public string $currency,
    ) {
        if ($minorUnits < 0) {
            throw new InvalidArgumentException('金額不可為負數');
        }
    }
}
```

規格訊息不是 PHP 語法，而是「金額使用最小貨幣單位、不可為負、必須帶幣種」。

### Entity 與識別

只有具備獨立生命週期、需要被持續追蹤的概念才需要 entity。地址快照、費率快照若只附屬於交易，不該因為有欄位就各自變成資料表與 entity。

### 狀態機

```text
Draft → Submitted → Paid
  └→ Cancelled
Submitted → Expired
```

狀態圖必須搭配轉移條件。只列 enum 不是狀態規格。

```yaml
transition:
  command: PayOrder
  from: Submitted
  to: Paid
  preconditions:
    - payment.amount == order.payable_amount
    - payment.currency == order.currency
  emits:
    - OrderPaid
```

### 不變量

不變量是系統無論走哪條路都必須成立的規則。

```text
INV-ORDER-001：已付款金額不得小於零。
INV-ORDER-002：訂單與付款幣種必須一致。
INV-ORDER-003：訂單一旦 Paid，不可直接回到 Submitted。
```

規格若只有正常流程而沒有不變量，AI 很容易生成「能跑但業務上錯誤」的設計。

### 命令與事件

命令表示意圖，可能被拒絕；事件表示已發生的事，不能用未來式命名。

```text
Command: ConfirmPayment
Event: PaymentConfirmed
Rejected: PaymentAmountMismatch
```

這個分離會逼出責任邊界：誰有權提出命令、哪個 aggregate 判斷、成功後誰接收事件。

## 規格檔的最低欄位

```yaml
id: ORDER-PAY-001
title: 確認訂單付款
actors: [merchant]
preconditions:
  - order.status == submitted
command:
  name: ConfirmPayment
inputs:
  order_id: OrderId
  amount: Money
postconditions:
  - order.status == paid
invariants:
  - INV-ORDER-002
errors:
  - code: ORDER_NOT_PAYABLE
  - code: PAYMENT_AMOUNT_MISMATCH
events:
  - OrderPaid
open_questions: []
```

`id` 讓規格、測試、程式與決策紀錄可以互相追蹤。沒有穩定識別碼，規格變更只能靠全文搜尋與記憶。

## 不該塞進規格模型的內容

- 尚未決定的假設：放入 `open_questions`。
- 純 UI 排版：放介面規格。
- 基礎設施細節：除非它會改變業務保證。
- AI 猜測的需求：標示來源與信心水準，不可偽裝成既定規則。
- 為了讓模型完整而捏造的欄位：刪除或提出問題。

## 完成判準

至少要能由另一位工程師根據規格回答合法狀態、拒絕條件、錯誤優先順序與副作用，且同一條規則沒有散落在多份文件中互相矛盾。

更新日期：2026-07-21
