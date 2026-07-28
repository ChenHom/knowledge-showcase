# 依賴注入原理、實作與設計模式：Pure DI、Container、生命週期與設計模式

> 草稿整理：依據目前關於 Dependency Injection（DI）、Pure DI、DI Container、Service Locator、生命週期管理、Composition Root 與設計模式的討論整理。

## 一句話總結

**依賴注入的本質，是把「使用物件」跟「建立物件」分離。**

物件本身不應該自己決定要建立哪個具體依賴，而是宣告自己需要什麼，由外部在組裝階段把依賴交給它。

```ts
// 不推薦：OrderService 自己建立依賴
class OrderService {
  private repo = new OrderRepository()
  private payment = new StripePaymentGateway()
}

// 推薦：依賴由外部注入
class OrderService {
  constructor(
    private repo: OrderRepository,
    private payment: PaymentGateway
  ) {}
}
```

DI 的價值不是少寫幾個 `new`，而是讓系統更容易：

- 測試
- 替換實作
- 隔離外部副作用
- 管理生命週期
- 降低業務邏輯與基礎設施的耦合

---

## DI、IoC 與依賴反轉

DI 常和 IoC 一起出現。

**IoC（Inversion of Control，控制反轉）**是一種設計思想：原本由物件自己控制依賴建立，改成由外部控制。

**DI 是實作 IoC 的一種方式。**

```txt
原本：物件自己 new 依賴
改成：外部建立依賴後注入物件
```

這也呼應依賴反轉原則：

> 高層業務邏輯不應依賴低層具體實作，兩者都應依賴抽象。

例如：

```ts
interface PaymentGateway {
  charge(amount: number): Promise<void>
}

class OrderService {
  constructor(private payment: PaymentGateway) {}
}
```

`OrderService` 不需要知道背後是 Stripe、PayPal，還是測試用 FakePaymentGateway。


---

## IoC、DI 與依賴反轉的差異

這三個概念常被混在一起，但它們其實位在不同層次：

```txt
IoC：控制權反轉的泛稱 / 思想
DI：實作 IoC 的一種手法，專門處理依賴取得
DIP：Dependency Inversion Principle，依賴反轉原則，SOLID 裡的設計原則
```

### IoC：誰控制流程或建立？

**IoC（Inversion of Control，控制反轉）**指的是原本由自己的程式主動控制流程、建立物件或呼叫邏輯，改成由外部框架、容器或環境來控制。

例如傳統程式：

```ts
const app = new App()
app.run()
```

這是你的程式主動控制流程。

Web framework 裡常見的寫法：

```ts
router.get('/users', userController.list)
```

你只是註冊 handler，真正什麼時候呼叫它，是 framework 決定。這就是 IoC。

IoC 可以出現在很多地方，不一定跟 DI 有關：

- framework 呼叫 callback
- event loop 觸發 handler
- lifecycle hook
- template method
- plugin system
- DI container 建立物件

### DI：依賴從哪裡來？

**DI（Dependency Injection，依賴注入）**是 IoC 的一種具體形式，專門處理「依賴怎麼取得」。

原本物件自己建立依賴：

```ts
class OrderService {
  private payment = new StripePaymentGateway()
}
```

改成外部注入：

```ts
class OrderService {
  constructor(private payment: PaymentGateway) {}
}
```

控制權從 `OrderService` 內部轉到外部組裝者。

所以：

```txt
DI 一定是某種 IoC。
但 IoC 不一定是 DI。
```

### DIP：程式碼依賴抽象還是具體實作？

依賴反轉通常指 **DIP（Dependency Inversion Principle，依賴反轉原則）**。

它不是工具，也不是注入方式，而是一條設計原則：

> 高層模組不應依賴低層模組；兩者都應依賴抽象。
> 抽象不應依賴細節；細節應依賴抽象。

壞例子：

```ts
class OrderService {
  private payment = new StripePaymentGateway()
}
```

`OrderService` 是高層業務邏輯，卻直接依賴低層 Stripe 實作。

比較好的版本：

```ts
interface PaymentGateway {
  charge(amount: number): Promise<void>
}

class OrderService {
  constructor(private payment: PaymentGateway) {}
}

class StripePaymentGateway implements PaymentGateway {
  charge(amount: number) {
    // call Stripe API
  }
}
```

這裡：

```txt
OrderService 依賴 PaymentGateway 抽象
StripePaymentGateway 也依賴 PaymentGateway 抽象
```

這才是依賴反轉。

### 有 IoC 就算有 DI 嗎？

不一定。

例如：

```ts
button.onClick(() => {
  console.log('clicked')
})
```

這是 IoC，因為不是你主動呼叫 callback，而是 UI framework 在事件發生時呼叫你。

但這不是 DI，因為它沒有處理物件依賴注入。

```txt
有 IoC ≠ 有 DI
```

### 有 IoC 就算有依賴反轉嗎？

也不一定。

例如：

```ts
router.get('/pay', () => {
  const stripe = new StripePaymentGateway()
  stripe.charge(...)
})
```

路由 handler 被 framework 呼叫，所以有 IoC。

但 handler 裡還是直接依賴 Stripe 具體實作，所以沒有做好 DIP。

```txt
有 IoC ≠ 有 DIP
```

### 有 DI 就算有依賴反轉嗎？

也不一定。

你可以把具體類別注入進去：

```ts
class OrderService {
  constructor(private payment: StripePaymentGateway) {}
}
```

這是 DI，因為依賴從外部注入。

但它仍然依賴具體 Stripe 類別，不是依賴抽象。

比較符合 DIP 的版本是：

```ts
class OrderService {
  constructor(private payment: PaymentGateway) {}
}
```

所以：

```txt
有 DI ≠ 一定有 DIP
```

DI 是「怎麼給依賴」。
DIP 是「應該依賴誰」。

### 有 DIP 一定要 DI 嗎？

嚴格說也不一定，但實務上常搭配。

可以用 Factory、Plugin、Event 等方式達成依賴抽象，不一定非得 constructor injection。

但在大多數業務程式中：

```txt
DIP 通常靠 DI 實作起來最自然。
```

### 關係圖

```txt
IoC：控制權交給外部
 └─ DI：依賴取得的控制權交給外部

DIP：高層與低層都依賴抽象
```

更精準地說：

```txt
IoC 是大概念
DI 是技術手法
DIP 是設計原則
```

一句話記法：

```txt
IoC 問：誰控制流程或建立？
DI 問：依賴從哪裡來？
DIP 問：程式碼依賴抽象還是具體實作？
```

結論：

```txt
有 IoC，不一定有 DI。
有 IoC，不一定有依賴反轉。
有 DI，也不一定有依賴反轉。
但好的 DI 設計，通常會用來實現依賴反轉。
```
---

## Pure DI 是什麼？

**Pure DI** 指的是不用 DI Container，而是直接用程式碼手動組裝物件圖。

```ts
const db = new PostgresDatabase()
const userRepo = new UserRepository(db)
const email = new SendGridEmailService()
const userService = new UserService(userRepo, email)
const controller = new UserController(userService)
```

這仍然是 DI，因為依賴是從外部注入的；只是組裝方式是手寫，而不是交給容器。

### Pure DI 優點

- 明確，沒有魔法
- 好 debug
- 不依賴框架
- 小型專案很乾淨
- 測試時容易控制
- 依賴關係一眼可見

### Pure DI 缺點

當依賴圖變大時，手動接線會變得冗長：

```ts
const a = new A()
const b = new B(a)
const c = new C(a, b)
const d = new D(c)
const e = new E(b, d)
```

當專案開始有大量模組、request scope、transaction scope、provider override 時，Pure DI 的組裝成本會上升。

---

## DI Container 是什麼？

**DI Container** 是幫你建立物件、解析依賴、管理生命週期的工具。

例如 NestJS、Spring、Angular 都有 DI Container。

```ts
@Injectable()
class UserService {
  constructor(private repo: UserRepository) {}
}
```

你不需要自己寫：

```ts
new UserService(new UserRepository(...))
```

容器會根據註冊規則或 metadata 自動組裝。

DI Container 通常負責：

- 建立物件
- 解析 constructor 需要哪些依賴
- 自動注入
- 管理 singleton / scoped / transient
- 處理 request scope
- 綁定 interface 到 implementation
- 測試時 override provider
- 組裝 decorator、pipeline、多實作集合

### Pure DI vs DI Container

```txt
Pure DI：依賴怎麼組裝，你看得見。
DI Container：依賴怎麼組裝，交給容器處理。
```

### 什麼時候用 Pure DI？

適合：

- 小到中型專案
- CLI 工具
- library
- 核心 domain logic
- 依賴圖不複雜
- 不需要 request scope
- 重視明確性與低魔法

### 什麼時候用 DI Container？

適合：

- 使用 NestJS、Spring、Angular 這類框架
- 大型後端 API
- 模組很多
- 依賴圖很大
- 有 request scope / transaction scope
- 有大量 provider override
- 有 logging、metrics、tracing、interceptor 等橫切需求
- 團隊已熟悉 container 規則

實務建議：

```txt
核心邏輯用 Pure DI 思維設計。
外層 framework / infrastructure 可以用 DI Container 組裝。
```

核心業務類別不要知道 container 存在。

---

## Service Locator 跟 DI Container 的差別

差別在於：**誰主動去拿依賴。**

### DI Container 正常用法

物件只宣告自己需要什麼，由外部注入。

```ts
class UserService {
  constructor(private repo: UserRepository) {}
}
```

`UserService` 不知道 container 存在。

### Service Locator

物件自己去 locator / container 裡查詢依賴。

```ts
class UserService {
  createUser() {
    const repo = ServiceLocator.get(UserRepository)
    repo.save(...)
  }
}
```

或：

```ts
class UserService {
  constructor(private container: Container) {}

  createUser() {
    const repo = this.container.get(UserRepository)
  }
}
```

這就是 Service Locator。

```txt
DI Container：依賴從外面被注入進來
UserService ← repo

Service Locator：物件自己去查詢依賴
UserService → locator.get(repo)
```

Service Locator 常被視為反模式，因為它會把依賴藏起來：

- constructor 看不出真正依賴
- 測試時才發現缺東西
- 型別系統較難協助檢查
- 物件與 container 耦合
- 錯誤容易到 runtime 才爆
- 容易變成全域變數式設計

少數可以接受 Service Locator 味道的地方：

- Composition Root
- plugin system
- framework adapter
- middleware / controller factory
- 動態 handler 載入邊界

但不要放進核心業務邏輯。

---

## Composition Root 是什麼？

**Composition Root** 是整個程式裡集中建立物件、接好依賴關係的地方。

通常位於：

```txt
main.ts
app.ts
bootstrap.ts
server.ts
program.cs
Application.java
```

它負責：

- 建立 concrete implementation
- 決定 interface 綁哪個實作
- 設定生命週期
- 接好 decorator / adapter / factory
- 初始化 DI container
- 啟動最外層 app / controller / handler

Pure DI 的 Composition Root 可能長這樣：

```ts
const db = new Database()
const repo = new OrderRepository(db)
const payment = new StripePaymentGateway()
const orderService = new OrderService(repo, payment)
const controller = new OrderController(orderService)
```

DI Container 的 Composition Root 則可能是：

```ts
container.register(OrderRepository)
container.register(OrderService)
container.register(OrderController)

const app = container.resolve(App)
```

核心原則：

```txt
業務類別：我需要什麼。
Composition Root：我要給你什麼。
```

Composition Root 是組裝區，不是業務邏輯區。

---

## 副作用邊界是什麼？

**副作用邊界**是程式從「單純計算」走出去，開始碰到外部世界或可變狀態的地方。

純計算：

```ts
function calculateTotal(price: number, qty: number) {
  return price * qty
}
```

同樣輸入永遠得到同樣輸出，不碰外部世界。

副作用：

```ts
await db.save(order)
await email.send(...)
await payment.charge(...)
const now = new Date()
const id = crypto.randomUUID()
```

常見副作用邊界：

- DB / Repository
- Cache
- Queue
- HTTP API
- Payment gateway
- Email / SMS / Push
- File system
- Clock / time
- UUID / random
- Logger / metrics / tracing
- Environment config
- Current user / request context

這些地方容易慢、失敗、不穩定、測試困難，所以特別適合用 DI 包起來。

正式環境：

```ts
const payment = new StripePaymentGateway()
const email = new SendGridEmailService()
const clock = new SystemClock()
```

測試環境：

```ts
const payment = new FakePaymentGateway()
const email = new FakeEmailService()
const clock = new FixedClock(new Date('2026-01-01'))
```

業務邏輯不用改。

---

## 生命週期管理

DI 裡的生命週期管理，是回答：

> 一個依賴物件應該什麼時候被建立？建立幾份？什麼時候釋放？

常見生命週期：

```txt
Singleton：整個應用程式共用同一個實例
Scoped：某個範圍內共用同一個實例
Transient：每次需要都建立新的實例
```

### Singleton

整個 application 只有一份。

適合：

- Logger
- Config
- DB connection pool
- HTTP client
- Metrics client
- Cache client

不適合放 request-specific mutable state。

例如 `CurrentUser` 不應該是 singleton，否則不同 request 可能共用到錯誤使用者狀態。

### Scoped

在某個範圍內共用一份，最常見是 request scope。

適合：

- CurrentUser
- RequestContext
- UnitOfWork
- TransactionManager
- per-request cache
- correlation id / trace id
- tenant context

```txt
Request A
  ├─ CurrentUser #A
  ├─ Transaction #A
  └─ RequestLogger #A

Request B
  ├─ CurrentUser #B
  ├─ Transaction #B
  └─ RequestLogger #B
```

### Transient

每次需要就建立新的。

適合：

- 無狀態小物件
- command handler
- validator
- formatter
- builder
- 短生命週期工作物件

### 常見生命週期錯誤：Captive Dependency

**Captive Dependency** 是指長生命週期物件持有短生命週期依賴。

錯誤例子：

```ts
class SingletonOrderService {
  constructor(private currentUser: CurrentUser) {}
}
```

如果 `OrderService` 是 singleton，而 `CurrentUser` 是 request scoped，singleton 可能抓住第一次 request 的 user，導致後續 request 用錯狀態。

危險方向：

```txt
Singleton -> Scoped / stateful Transient
```

比較安全的方向：

```txt
Scoped -> Singleton
Transient -> Singleton
```

簡單說：短生命週期可以依賴長生命週期，但長生命週期不要直接持有短生命週期狀態。

---

## DI 與設計模式

DI 不取代設計模式，而是讓很多設計模式更容易落地。

```txt
設計模式：定義物件之間怎麼合作。
DI：負責把這些物件接起來。
```

### Strategy Pattern：策略模式

策略模式讓演算法或行為可替換。

```ts
interface DiscountStrategy {
  calculate(price: number): number
}

class VipDiscount implements DiscountStrategy {
  calculate(price: number) {
    return price * 0.8
  }
}

class NormalDiscount implements DiscountStrategy {
  calculate(price: number) {
    return price
  }
}

class CheckoutService {
  constructor(private discount: DiscountStrategy) {}
}
```

DI 負責決定注入哪個策略。

適合：付款策略、折扣策略、排序策略、風控策略、通知渠道、推薦演算法。

### Adapter Pattern：轉接器模式

Adapter 把外部 API 包成系統內部想要的介面。

```ts
interface PaymentGateway {
  charge(amount: number): Promise<void>
}

class StripePaymentAdapter implements PaymentGateway {
  async charge(amount: number) {
    await stripe.paymentIntents.create({
      amount,
      currency: 'usd'
    })
  }
}
```

業務服務只依賴 `PaymentGateway`，不依賴 Stripe SDK。

這是 DI 在實務上最常見、最有價值的用法之一。

### Decorator Pattern：裝飾器模式

Decorator 在不改原本類別的情況下增加行為。

```ts
interface UserRepository {
  findById(id: string): Promise<User>
}

class SqlUserRepository implements UserRepository {
  async findById(id: string) {
    return db.query(...)
  }
}

class CachedUserRepository implements UserRepository {
  constructor(private inner: UserRepository) {}

  async findById(id: string) {
    const cached = await cache.get(id)
    if (cached) return cached

    const user = await this.inner.findById(id)
    await cache.set(id, user)
    return user
  }
}

class LoggingUserRepository implements UserRepository {
  constructor(private inner: UserRepository) {}

  async findById(id: string) {
    logger.info('find user', id)
    return this.inner.findById(id)
  }
}
```

組合：

```ts
const repo =
  new LoggingUserRepository(
    new CachedUserRepository(
      new SqlUserRepository()
    )
  )
```

適合：logging、metrics、tracing、caching、retry、transaction、authorization、rate limit。

### Abstract Factory：抽象工廠

有些依賴無法在 app 啟動時決定，而是 runtime 才知道。

```ts
interface PaymentGatewayFactory {
  create(method: PaymentMethod): PaymentGateway
}

class CheckoutService {
  constructor(private factory: PaymentGatewayFactory) {}

  checkout(order: Order) {
    const gateway = this.factory.create(order.paymentMethod)
    return gateway.charge(order.amount)
  }
}
```

需要動態建立物件時，優先注入明確 factory，而不是把整個 container 注入進業務服務。

### Composite Pattern：組合模式

Composite 把多個實作包成一個實作。

```ts
interface Notifier {
  send(message: string): Promise<void>
}

class CompositeNotifier implements Notifier {
  constructor(private notifiers: Notifier[]) {}

  async send(message: string) {
    for (const notifier of this.notifiers) {
      await notifier.send(message)
    }
  }
}
```

業務服務只依賴一個 `Notifier`，不需要知道背後有 Email、Slack、SMS 幾種通知渠道。

### Chain of Responsibility：責任鏈

常見於 middleware、validator、handler pipeline。

```ts
interface OrderValidator {
  validate(order: Order): void
}

class OrderValidationPipeline {
  constructor(private validators: OrderValidator[]) {}

  validate(order: Order) {
    for (const validator of this.validators) {
      validator.validate(order)
    }
  }
}
```

適合：middleware、validation pipeline、event handlers、command handlers、rule engine、filters。

---

## 如何不過度工程化？

核心原則：

> 先讓程式碼清楚，再讓它可抽換；不要為了「可能未來會換」先抽一堆介面。

### 1. 不要每個 class 都配一個 interface

不一定要這樣：

```ts
interface IUserService {}
class UserService implements IUserService {}
```

如果目前只有一個實作，而且沒有明確替換需求，直接用 concrete class 就好。

值得抽 interface 的情況：

- 會打外部 API
- 會碰 DB / cache / queue
- 測試需要 fake
- 真的有多個實作
- 是跨模組邊界
- 要隔離第三方套件

### 2. 純計算與小 helper 不要硬 DI

```ts
function calculateDiscount(price: number, level: UserLevel) {
  return ...
}
```

這種純函式通常直接寫最乾淨，不需要硬包成 `DiscountCalculator`。

### 3. 抽象要來自痛點，不要來自想像

先問：

```txt
現在有第二個實作嗎？
測試真的需要替換嗎？
這是外部服務或副作用邊界嗎？
這個依賴是否跨越架構邊界？
```

如果答案都是否，就先不要抽。

### 4. 副作用邊界用 DI，核心邏輯少抽象

最值得 DI 的地方：

```txt
DB / HTTP API / 金流 / Email / Queue / Cache / File system / Clock / UUID / Logger / Config
```

最不需要重 DI 的地方：

```txt
純計算 / 資料轉換 / 小 helper / value object / 簡單 validator
```

### 5. 先 Pure DI，痛了再 Container

```txt
只有一個實作、無副作用、小專案
→ 直接 new / concrete class

有副作用、需要測試替換
→ constructor injection

有多個實作，需要 runtime 切換
→ Strategy / Factory + DI

依賴圖很大，scope 很複雜
→ DI Container

核心 domain logic
→ 優先簡單、純函式、明確資料流
```

---

## 實務設計準則

1. **業務核心只依賴明確依賴，不依賴 container。**

```ts
class OrderService {
  constructor(
    private repo: OrderRepository,
    private payment: PaymentGateway
  ) {}
}
```

不要：

```ts
class OrderService {
  constructor(private container: Container) {}
}
```

2. **生命週期由外層組裝層決定。**

業務類別不要自己決定要 new 哪個具體依賴。

3. **長生命週期不要抓短生命週期。**

尤其 singleton 不要持有 `CurrentUser`、`RequestContext`、`Transaction`。

4. **外部系統用 Adapter 包起來。**

業務邏輯不要直接依賴 Stripe SDK、SendGrid SDK、資料庫 driver。

5. **橫切需求用 Decorator / Pipeline。**

例如 logging、cache、retry、metrics、tracing。

6. **動態選擇用 Strategy / Factory。**

不要把 container 當 Service Locator 塞進業務邏輯。

---

## 最後整理

DI 的主軸不是工具，而是設計邊界：

```txt
核心業務：
  Constructor Injection + 明確依賴 + 少魔法

外部基礎設施：
  Adapter 包外部服務

橫切需求：
  Decorator 加 logging/cache/retry/metrics

多實作集合：
  Composite / Chain of Responsibility

動態選擇：
  Strategy / Abstract Factory

大型 app 組裝：
  DI Container 管生命週期與依賴圖
```

一句話：

**DI 不是只為了少寫 `new`，而是為了讓物件的建立、替換、組合、生命週期，從業務邏輯裡分離出去。**
