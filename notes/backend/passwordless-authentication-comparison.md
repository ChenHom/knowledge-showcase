# Passwordless / Authentication 登入方式設計與安全性比較

#authentication #passwordless #passkey #webauthn #mfa #security #backend #knowledge

> 適用：Web / App 會員系統、SaaS、管理後台、金流與高權限系統的登入設計與 Security Review
> 整理日期：2026-09-23

## 這是什麼

比較 Password、Email / SMS Magic Link、Email / SMS OTP、TOTP、Push Approval、QR Login、Passkey（Synced / Device-bound）、三方登入與 Password + MFA 的原理、優缺點、適用情境與補強方式，並把 Recovery、Session、Step-up、Transaction Authorization、Factor Replacement 放進同一條 Security Boundary 評估。最後附資料模型建議與 Security Review Checklist。

Session 狀態與 Refresh Token 的細節另見 [Refresh Token 問題處理與架構設計指南](note.html?slug=backend/refresh-token-design-guide)。

## 核心結論

- 登入安全不是單一登入方式的屬性，而是整條帳號生命週期中**最弱一環**的屬性；Recovery Path 與 Factor Replacement 常是那個最弱點。
- Passwordless 不是安全等級：Email OTP 與 Passkey 都叫 Passwordless，安全模型差很多。
- **MFA ≠ Phishing Resistant**：Password / OTP / TOTP / 普通 Push 都可被即時 relay；只有 Passkey / WebAuthn 有 origin binding。
- Magic Link 是低摩擦登入，不是高安全登入；Email 被攻破 ≈ 帳號被攻破。
- 選方式前先問「這個帳號被接管的代價多高」，再決定需要多少 assurance；高風險系統不要讓 Email 或 SMS 成為唯一 Root of Trust。

---

## 1. 先建立正確觀念：登入方式不是單獨存在的功能

評估登入安全性時，不應只問：

> 「Magic Link 安不安全？」
> 「Passkey 安不安全？」
> 「有 MFA 就夠了嗎？」

真正需要評估的是整條帳號生命週期：

```text
Identity Proofing
        ↓
Authenticator Enrollment
        ↓
Authentication
        ↓
Session
        ↓
Authorization
        ↓
Sensitive Action
        ↓
Step-up / Transaction Authorization
        ↓
Recovery
        ↓
Factor Replacement
```

其中任一層可以繞過前面的強驗證，整體帳號安全性就會被那個最弱點限制。

例如：

```text
主要登入：Passkey
            ↓
Forgot Passkey?
            ↓
Email Magic Link
            ↓
直接重設 Passkey
```

表面上是 Passkey 系統，實際帳號接管門檻仍接近：

```text
攻破 Email
```

因此 **Recovery Path 是 Authentication Security 的一部分，不是附屬功能。**

---

## 2. 先把常被混在一起的概念拆開

### 2.1 Authentication

確認「目前操作的人」是否持有某個帳號的有效驗證憑證。

例如：

- Password
- Magic Link
- OTP
- TOTP
- Passkey
- Security Key
- 已綁定裝置

---

### 2.2 Identity Proofing

確認數位帳號背後現實中的人是誰。

例如：

- 身分證核驗
- KYC
- 銀行帳戶驗證
- 人工身分審核

Magic Link 只能證明：

```text
某人目前可以讀取 foo@example.com
```

不能證明：

```text
這個人現實身分就是某個特定自然人
```

---

### 2.3 Authorization

驗證完成後，決定使用者能做什麼：

```text
User
Admin
Finance
Merchant
Super Admin
```

例如：

- RBAC
- ABAC
- CASL
- Permission / Role

AuthN 與 AuthZ 不應混在一起。

---

## 3. Authentication 方式總覽

| 方式 | 驗證依據 | 是否 Passwordless | Phishing Resistance | 主要外部依賴 | 整體定位 |
|---|---|---:|---:|---|---|
| Password | 使用者知道的秘密 | 否 | 低 | 無 | 基本型 |
| Email Magic Link | Email 控制權 + 一次性 URL | 是 | 低～中 | Email | 低摩擦登入 |
| SMS Magic Link | 手機門號 + 一次性 URL | 是 | 低 | SMS / 電信商 | 不優先 |
| Email OTP | Email 控制權 + 短碼 | 是 | 低 | Email | Magic Link 替代 |
| SMS OTP | 手機門號 + 短碼 | 是 | 低 | SMS / 電信商 | 常見 Possession Check |
| TOTP | Shared Secret + 時間 | 可是 | 低 | Authenticator | 常見第二因子 |
| Push Approval | 已綁定裝置 | 是 | 視實作而定 | Push Service | 適合已有 App |
| QR Trusted Device | 已登入裝置 | 是 | 視實作而定 | 已綁定裝置 | 桌機登入很好用 |
| Synced Passkey | 非對稱金鑰 | 是 | 高 | Passkey Provider | 現代主登入方式 |
| Device-bound Passkey / Security Key | 裝置綁定非對稱金鑰 | 是 | 高 | 硬體裝置 | 高安全 |
| Google / LINE / Apple Login | 外部 IdP 身分證明 | 是 | 依 IdP | 外部 IdP | Federated Login |
| Password + MFA | 多個不同因子 | 否 | 視第二因子 | 視 MFA | 傳統高強度方案 |

> Passwordless 不是安全等級。Email OTP 與 Passkey 都可以叫 Passwordless，但安全模型差非常大。

---

## 4. Email Magic Link

### 4.1 原理

```text
輸入 Email
    ↓
Server 產生高熵 random token
    ↓
DB 儲存 token hash
    ↓
寄送 URL

https://example.com/auth/magic?t=<random-token>

    ↓
使用者點擊
    ↓
Server 驗證並 consume token
    ↓
建立正式 Session
```

Magic Link 本身就是暫時性的 Bearer Credential：

```text
誰取得有效 URL
≈
誰有登入權限
```

---

### 4.2 優點

- 不需要管理密碼。
- 不存在弱密碼與密碼重複使用。
- 可大幅降低 credential stuffing。
- UX 低摩擦。
- 不需要 Password Reset 流程。
- Token 可以高 entropy、短效、single-use。
- 適合作為 Email ownership verification。

---

### 4.3 缺點

#### Email 被攻破 = 帳號通常也被攻破

```text
攻擊者控制 Gmail
        ↓
要求 Magic Link
        ↓
收到信
        ↓
登入
```

Magic Link only 本質上是一個單因子系統。

---

#### URL 是 Credential

可能洩漏到：

- Browser history
- Reverse proxy log
- Nginx access log
- APM
- Sentry
- Analytics
- Referer
- Browser extension
- Email forwarding
- Link preview
- Security scanner

---

#### Email Scanner 可能自動打開連結

如果：

```text
GET /magic?t=xxx
→ 直接 consume
```

企業郵件掃描器可能比使用者先開啟 URL，造成真正使用者點擊時 token 已失效。

---

### 4.4 適合

- 內容網站
- 一般 SaaS
- 論壇 / 社群
- 低～中風險會員系統
- 首次註冊
- Email 驗證
- Passkey 建立前的 bootstrap
- 某些低風險 recovery 流程

---

### 4.5 不適合單獨使用

- 金流後台
- 錢包 / 提領系統
- Super Admin
- 高權限管理員
- 高價值企業帳號
- 高敏感個資
- 任何不能接受「Email 被拿走 = 帳號被拿走」的系統

---

### 4.6 補強方式

#### Token

```text
CSPRNG
256-bit random token
5～15 分鐘 TTL
single-use
DB only stores SHA-256(token)
```

不要把 user ID / email 直接簽進一顆長效 JWT 當 Magic Link。

---

#### Consume 必須 atomic

```sql
UPDATE auth_challenges
SET consumed_at = NOW()
WHERE token_hash = ?
  AND consumed_at IS NULL
  AND expires_at > NOW()
RETURNING user_id;
```

避免兩個 request 同時通過。

---

#### GET 不 consume

推薦：

```text
GET /magic?t=xxx
        ↓
驗證 token
但不 consume
        ↓
建立 temporary challenge
        ↓
303 Redirect
        ↓
/auth/magic/confirm
        ↓
使用者確認
        ↓
POST /auth/magic/consume
        ↓
atomic consume
        ↓
create session
```

---

#### URL 洩漏防護

- HTTPS only
- `Referrer-Policy: no-referrer`
- Token 不寫 log
- APM / Sentry redact
- 不載入不必要 third-party scripts
- 驗證後迅速 redirect 到乾淨 URL
- 禁止 arbitrary redirect URL

---

#### Rate Limit

至少做：

```text
IP
Email
IP + Email
```

並限制 resend。

---

#### Step-up

Magic Link 可以負責一般登入，但：

```text
修改 Email
修改手機
新增提款帳戶
提款
變更 MFA
新增 Passkey
API Key 管理
管理員操作
```

再要求 Passkey / TOTP / Security Key。

---

## 5. SMS Magic Link

### 原理

跟 Email Magic Link 完全相同，只把 transport 換成 SMS：

```text
手機號碼
   ↓
Server 產生 random token
   ↓
SMS：

https://example.com/magic?t=...
```

---

### 優點

- 點擊即可登入。
- Token 可以高 entropy。
- 比手動輸入 OTP 少一步。

---

### 缺點

同時承受兩組問題：

#### SMS 本身

- SIM Swap
- Number Porting
- 電信商社交工程
- 門號回收
- SMS interception

#### URL Credential

- Browser history
- Link forwarding
- URL preview
- Log leakage
- redirect 問題

---

### 判斷

如果已經使用 SMS：

```text
SMS OTP
```

通常會比：

```text
SMS Magic Link
```

更單純。

Magic Link token entropy 更高，但 SMS Magic Link 把 URL credential 的額外風險也一起帶入。

---

## 6. Email OTP

### 流程

```text
Email
  ↓
Server 建 challenge
  ↓
Email：

Code: 381921

  ↓
回原網站
  ↓
輸入 code
  ↓
Server 驗證
  ↓
建立 Session
```

---

### 優點

- 不會被 Email link scanner 自動 consume。
- Credential 不出現在 URL。
- 不需要 deep link。
- 跨 browser / App 通常比 Magic Link 穩定。
- 實作相對直接。

---

### 缺點

- Email 被攻破仍可登入。
- 6 位 OTP entropy 很低。
- 必須防 brute force。
- 仍可被 phishing。
- UX 比 Magic Link 多一步。

---

### 必要補強

```text
6 digits
5 分鐘 TTL
最多 5 次嘗試
成功即 consume
challenge-bound
IP rate limit
Email rate limit
resend rate limit
```

不要只用：

```text
email + otp
```

查詢。

應使用：

```text
challenge_id + otp
```

避免 OTP 汙染不同 challenge。

---

## 7. SMS OTP

### 優點

- 使用者熟悉。
- 手機場景方便。
- Credential 不在 URL。
- 可做手機 possession verification。
- 適合做 fallback 或中等風險 step-up。

---

### 缺點

- SIM Swap。
- 電信商 account takeover。
- 門號重新配發。
- 漫遊 / 收訊問題。
- SMS 成本。
- OTP 可 phishing。
- OTP entropy 低。

---

### 適合

- 手機為核心識別資料的服務。
- 中低風險會員登入。
- 手機驗證。
- Recovery 的其中一個信號。
- 無 Passkey 時的 fallback。

---

### 不適合單獨承擔

- 高額提款
- 超級管理員
- 高權限 DevOps
- 高價值企業帳號
- 大額交易確認

---

### 補強

- SIM change / phone-number change 後冷卻期。
- 新裝置額外驗證。
- 異常 IP / ASN / 國家觸發 step-up。
- 修改手機號碼必須使用現有 factor 重新驗證。
- 不允許只靠新手機 OTP 直接接管帳號。

---

## 8. TOTP

例如：

```text
Google Authenticator
1Password
Microsoft Authenticator
```

TOTP 不等於 Google Login。

Authenticator App 只是保存 shared secret。

```text
Server secret
      +
Current Time
      ↓
382591
```

使用者 App 有相同 secret，所以能產生同一組 code。

---

### 優點

- 不依賴 SMS。
- 可離線。
- 成本低。
- 生態成熟。
- 適合當第二因子。

---

### 缺點

- 仍可 phishing。
- Shared secret 必須安全保存。
- 換機 / 遺失裝置要處理 recovery。
- Seed 被偷後可長期複製 OTP。
- UX 比 Passkey 差。

---

### 適合

- MFA。
- 管理後台第二因子。
- 既有系統漸進式提升安全性。

---

### 不應誤認為

```text
有 TOTP
=
phishing-resistant
```

攻擊者可建立即時 phishing proxy：

```text
假網站
 ↓
password
 ↓
TOTP
 ↓
即時 relay 到真網站
```

---

## 9. Passkey / WebAuthn

### 原理

```text
Server
  ↓
random challenge
  ↓
Browser / OS
  ↓
private key sign
  ↓
Server 使用 public key verify
```

Server 不持有 private key。

Credential 綁定網站的 RP ID / Origin。

---

### 優點

- Phishing-resistant。
- Credential stuffing 幾乎失去作用。
- Server 沒有 password database。
- challenge-response，具 replay resistance。
- Private key 不需要傳給 Server。
- 使用 Face ID / Touch ID / Windows Hello 時 UX 很低摩擦。

原文記載 WebAuthn Level 3 已於 2026-08-25 成為 W3C Recommendation（未查證，見文末「限制與未查證事項」）。

---

### 缺點

- Account recovery 必須重新設計。
- 使用者可能不了解 Passkey。
- 多裝置 / 共用電腦 UX 需要設計。
- Passkey provider account 會成為 synced passkey 的一部分信任鏈。
- 系統仍需要處理裝置遺失、credential replacement、session theft。

---

## 10. Synced Passkey 與 Device-bound Passkey

### Synced Passkey

例如 credential 由：

- iCloud Keychain
- Google Password Manager
- 1Password

跨裝置同步。

#### 優點

- 換手機較容易。
- 多裝置體驗好。
- Recovery 比 device-bound 容易。
- 一般會員系統非常適合。

#### 代價

信任鏈包含：

```text
Passkey Provider Account
```

例如 iCloud 帳號安全也會影響 passkey availability。

---

### Device-bound Passkey / Hardware Security Key

Credential 綁在：

- YubiKey
- TPM
- Hardware authenticator
- 特定裝置 secure hardware

不跨裝置同步。

#### 優點

- Assurance 更高。
- Credential 不容易因雲端帳號接管而擴散。
- 適合高權限身份。

#### 缺點

- 裝置遺失處理困難。
- 必須註冊備援 security key。
- 使用者 onboarding 成本較高。

---

### 適合

#### Synced Passkey

- 一般會員
- SaaS
- 電商
- 中高安全系統
- Password replacement

#### Device-bound

- Super Admin
- Finance
- DevOps
- Infrastructure access
- 高價值企業帳戶

---

## 11. Push Approval

### 流程

```text
Desktop request login
       ↓
Server 建 login challenge
       ↓
Push 到已綁定 App
       ↓
使用者確認
       ↓
App approve challenge
       ↓
Desktop 建立 Session
```

---

### 優點

- UX 好。
- 適合已有 App 的產品。
- 可以顯示登入裝置、地點、時間。
- 可搭配 device key 簽章。

---

### 缺點

單純：

```text
Approve / Deny
```

容易產生 MFA fatigue / push bombing。

使用者可能一直收到：

```text
允許登入？
允許登入？
允許登入？
```

最後誤按允許。

---

### 補強

不要只有：

```text
Approve
```

至少加入：

```text
登入裝置
大概位置
Browser
時間
```

更好：

```text
Number Matching
```

例如桌機顯示：

```text
42
```

手機必須選：

```text
42
```

或使用裝置 private key 對 challenge 簽章。

---

## 12. QR Code Login

適合：

```text
Desktop
+
已登入手機 App
```

---

### 正確模型

QR 不應直接包含 Access Token。

應只包含：

```text
login_challenge_id
+
random nonce
```

流程：

```text
Desktop 建 challenge
      ↓
QR
      ↓
已登入手機掃描
      ↓
手機顯示登入資訊
      ↓
Approve
      ↓
Server 標記 challenge approved
      ↓
Desktop 建立自己的 session
```

---

### 優點

- 桌機 UX 很好。
- 不需輸入密碼。
- 可以利用既有 trusted device。
- Credential 不需直接傳給桌機。

---

### 缺點

- 必須已有登入裝置。
- QR phishing / session swapping 仍需要防。
- challenge lifecycle 比單純登入複雜。

---

### 補強

- QR challenge 短效。
- Single-use。
- 顯示登入裝置資訊。
- 手機必須確認。
- Approval 綁定 desktop challenge。
- 不允許 QR 本身直接成為 Session Token。

---

## 13. Google / LINE / Apple 三方登入

這跟 Magic Link、OTP、Passkey 是不同類型。

Magic Link：

```text
你的系統驗證：
使用者控制這個 Email
```

Passkey：

```text
你的系統驗證：
使用者能用網站專屬 private key 簽章
```

三方登入：

```text
Google / LINE / Apple 告訴你的系統：
我已經驗證這個使用者
```

---

### 優點

- 不需要自己保存 Password。
- Login UX 好。
- 外部 IdP 通常已有 MFA / risk engine。
- Account recovery 很大部分由 IdP 處理。

---

### 缺點

- Vendor dependency。
- IdP account 被接管會影響你的帳號。
- IdP outage 可能影響登入。
- Account linking 設計容易出安全問題。
- 使用者可能失去第三方帳號。

---

### Identity Mapping

不要直接把：

```text
email
```

當第三方 Identity Primary Key。

推薦：

```text
user_identities

user_id
provider
provider_subject
```

例如：

```text
123 / google / 108123...
123 / line   / U89abc...
123 / apple  / 001928...
```

OpenID Connect 主要使用：

```text
issuer + subject
```

識別外部 Identity。

---

## 14. Password + MFA

傳統但仍然有效。

例如：

```text
Password
+
TOTP
```

或：

```text
Password
+
Passkey / Security Key
```

---

### 優點

- 使用者熟悉。
- Enterprise 支援成熟。
- Recovery 模型已有大量經驗。
- 第二因子可大幅降低 Password 被偷的風險。

---

### 缺點

Password 本身仍帶來：

- Password reuse
- credential stuffing
- Password reset
- Hash database
- phishing
- Support 成本

而：

```text
Password + OTP
```

雖是 MFA，仍不是 phishing-resistant。

---

## 15. Phishing Resistance 比「有沒有 MFA」更重要

以下方式通常仍可能被 phishing relay：

```text
Password
Email OTP
SMS OTP
TOTP
普通 Push Approval
```

攻擊流程：

```text
fake-example.com
      ↓
使用者輸入 Password
      ↓
輸入 OTP
      ↓
攻擊者即時 relay
      ↓
example.com
```

Passkey / WebAuthn 的差別在於 credential 有 verifier / origin binding。

```text
example.com credential
```

不能正常拿去：

```text
examp1e.com
```

使用。

因此：

```text
MFA
≠
Phishing Resistant
```

---

## 16. Replay Resistance

登入方式還要問：

```text
憑證被截到後能不能再次使用？
```

---

### Password

```text
abc123
```

除非修改，否則可以一直 replay。

---

### OTP

應該：

```text
成功一次
↓
立即 consume
```

即使 TTL 尚未過。

---

### Magic Link

必須：

```text
single-use
+
atomic consume
```

---

### Passkey

每次使用 Server 產生新的 challenge：

```text
random challenge
↓
signed response
```

舊 response 無法直接拿去完成新的 authentication ceremony。

---

## 17. Authentication Intent

需要區分：

```text
credential 存在
```

跟：

```text
使用者真的想進行這次登入
```

例如不建議：

```text
收到 GET
↓
直接登入
```

較好的 Magic Link：

```text
Link
↓
確認頁
↓
「繼續登入 example.com」
↓
POST
```

Passkey：

```text
網站 challenge
↓
Face ID / PIN
↓
使用者主動確認
```

可以更明確表達 intent。

---

## 18. Step-up Authentication

不需要所有操作都要求最高強度 authentication。

可以根據行為風險提高驗證強度：

```text
一般瀏覽
↓
既有 Session

修改暱稱
↓
既有 Session

修改 Email
↓
Passkey

新增提款帳戶
↓
Passkey

提款
↓
Passkey + Transaction Authorization
```

---

### 常見 Step-up Trigger

- 新裝置
- 異常 IP
- 異常國家
- Tor / Proxy
- 帳號剛完成 Recovery
- 修改 Email
- 修改手機
- 新增 Passkey
- 刪除 MFA
- 新增 API Key
- 修改付款資料
- 提款
- 管理員操作

---

## 19. Transaction Authorization 不等於再次登入

例如：

```text
User 123 已登入
```

不能直接推導：

```text
User 123 確認要轉 100,000 TWD 給 ABC
```

高風險交易應把交易資訊納入確認內容：

```text
amount
currency
recipient
transaction_id
nonce
```

而不是只：

```text
請再次輸入 OTP
```

對金融、錢包、支付系統特別重要。

---

## 20. Session Security

登入完成之後，真正最重要的 Credential 常變成：

```text
session cookie
```

如果攻擊者偷到：

```text
session=XYZ
```

可能完全不需要再次突破 Password / Passkey。

---

### 必做

Cookie：

```text
HttpOnly
Secure
SameSite
```

登入成功：

```text
rotate session id
```

避免 Session Fixation。

---

### 高風險事件

例如：

- Recovery 完成
- Password change
- MFA change
- Passkey enrollment
- 異常登入

應考慮：

```text
rotate session
revoke old sessions
require re-authentication
```

---

## 21. Recovery 是整套系統最容易出問題的地方

錯誤案例：

```text
主要登入：
Passkey + Security Key

Recovery：
Email Magic Link → 全部重設
```

攻擊者根本不用研究 WebAuthn：

```text
攻 Email
```

即可。

---

### 高安全 Recovery 可以使用

```text
Email
+
Recovery Code
```

或：

```text
Email
+
另一個已註冊 Passkey
```

或：

```text
另一台 Trusted Device
```

或：

```text
兩支 Security Key 中任一支
```

企業：

```text
Admin-assisted recovery
+
Identity verification
+
Audit
+
cooldown
```

---

## 22. Factor Replacement 比 Login 更敏感

假設攻擊者偷到一個仍有效的 Session。

如果可以直接：

```text
新增新的 Passkey
刪除舊 Passkey
修改手機
```

那強登入完全失去意義。

---

### 敏感操作前重新驗證

至少：

```text
新增 Passkey
刪除最後一個 Passkey
新增 TOTP
停用 MFA
修改 Email
修改手機
產生 Recovery Codes
```

必須重新驗證既有 Factor。

不要：

```text
目前有 Session
=
可以修改 Authentication Factors
```

---

## 23. 各方式的適用情境比較

| 場景 | 建議主方式 | 補強 |
|---|---|---|
| 內容網站 | Magic Link / OAuth | Rate Limit |
| 一般會員 | Passkey / OAuth | Email OTP fallback |
| 電商 | Passkey / OAuth | 高風險付款 Step-up |
| SaaS | Passkey / OIDC | Recovery + Session 管理 |
| B2B SaaS | Enterprise OIDC / Passkey | MFA Policy |
| 管理後台 | Passkey | TOTP / Security Key fallback |
| Super Admin | Device-bound Passkey / Security Key | 兩把 Key + Recovery 流程 |
| 金流後台 | Passkey / Security Key | Step-up + Audit |
| 錢包 / 提領 | Passkey | Transaction Authorization |
| 手機服務 | Passkey / SMS OTP | SIM change risk control |
| 沒有 App 的低風險產品 | Email Magic Link | Step-up |
| 已有 App 的產品 | Passkey / Push / QR | Device binding |

---

## 24. 各方式缺點與補強對照

| 問題 | 常見於 | 補強 |
|---|---|---|
| Email 被攻破 | Magic Link / Email OTP | Passkey、第二因子、Recovery 不只靠 Email |
| URL 洩漏 | Magic Link | 短效、single-use、redact log、clean URL |
| Mail scanner | Magic Link | GET 不 consume、POST confirm |
| OTP brute force | Email/SMS/TOTP | attempt limit、rate limit、短 TTL |
| OTP phishing | OTP / TOTP | Passkey / Security Key |
| SIM Swap | SMS | Passkey、裝置驗證、SIM change cooldown |
| Push Bombing | Push MFA | number matching、context、rate limit |
| QR hijacking | QR Login | challenge-bound、短效、手機確認 |
| Password reuse | Password | Password manager、MFA、Passkey |
| Session theft | 所有方式 | HttpOnly、Secure、rotation、re-auth |
| Recovery bypass | 所有方式 | 多因子 recovery、cooldown、audit |
| Factor takeover | 所有方式 | 修改 Factor 前 re-auth |
| OAuth account linking | Social Login | provider issuer + subject、顯式 linking |
| Device loss | Passkey | synced passkey、第二把 key、recovery code |

---

## 25. 安全強度不要用單一排名看

一個比較實際的思考方式是看多個維度：

| 方式 | Credential Stuffing | Phishing | Replay | Server Secret 洩漏 | Recovery 複雜度 |
|---|---:|---:|---:|---:|---:|
| Password | 差 | 差 | 差 | 中 | 低 |
| Email Magic Link | 好 | 中～差 | 好* | 中 | 低 |
| Email OTP | 好 | 差 | 好* | 中 | 低 |
| SMS OTP | 好 | 差 | 好* | 中 | 低 |
| TOTP | 好 | 差 | 好* | 差（seed） | 中 |
| Passkey | 好 | 好 | 好 | 好 | 中 |
| Device-bound Passkey | 好 | 好 | 好 | 好 | 高 |
| OAuth / OIDC | 取決於 IdP | 取決於 IdP | 通常好 | 取決於 IdP | 由 IdP 分攤 |

`*` 前提是正確實作 single-use / challenge consume。

---

## 26. 一般會員系統建議模型

比較平衡的做法：

```text
Primary

Passkey
Google / Apple / LINE

        ↓

Fallback

Email OTP

        ↓

Sensitive Action

Passkey re-authentication

        ↓

Recovery

Email
+
Recovery Code / Existing Passkey / Trusted Device
```

Email 不再是唯一的 Root of Trust。

---

## 27. 高權限系統建議模型

例如：

```text
Admin
Finance
Wallet
Infrastructure
Super Admin
```

推薦：

```text
Primary:
Passkey / Security Key

        ↓

Step-up:
Passkey user verification

        ↓

Sensitive Action:
Transaction / Action confirmation

        ↓

Recovery:
Second registered credential
+
Recovery Code / Admin process

        ↓

Audit:
所有 Factor / Recovery / Session 事件留紀錄
```

避免：

```text
Email Magic Link only
```

以及：

```text
SMS OTP only
```

---

## 28. Authentication 資料模型建議

不要把登入方式寫死在 User table：

```text
users
google_id
line_id
apple_id
totp_secret
...
```

推薦拆開：

```text
users

id
status
...
```

```text
user_identities

id
user_id
provider
provider_subject
provider_email
created_at
```

```text
user_credentials

id
user_id
type
credential_id
public_key
metadata
created_at
last_used_at
revoked_at
```

```text
auth_challenges

id
type
user_id
target
token_hash
attempt_count
expires_at
consumed_at
created_at
```

```text
sessions

id
user_id
device_id
created_at
last_seen_at
expires_at
revoked_at
```

---

## 29. Authentication Challenge 應抽象化

不要每個功能自己寫：

```text
magic_link_tokens
sms_codes
email_codes
reset_codes
```

可以抽象：

```text
auth_challenges

type:

MAGIC_LINK_LOGIN
EMAIL_OTP_LOGIN
SMS_OTP_LOGIN
EMAIL_VERIFY
PHONE_VERIFY
PASSWORD_RESET
ACCOUNT_RECOVERY
CHANGE_EMAIL
CHANGE_PHONE
DEVICE_APPROVAL
```

共同處理：

```text
expiration
attempts
consume
revocation
audit
rate limit
```

降低認證邏輯分散造成的漏洞。

---

## 30. Security Review Checklist

### Authentication

- [ ] Credential 是否足夠高 entropy？
- [ ] 是否 single-use？
- [ ] 是否短效？
- [ ] 是否 replay-resistant？
- [ ] 是否 phishing-resistant？
- [ ] 是否綁定 origin / challenge？
- [ ] 是否有 authentication intent？

### Rate Limit

- [ ] IP
- [ ] Account
- [ ] Email / Phone
- [ ] IP + Account
- [ ] resend
- [ ] verification attempt

### Magic Link

- [ ] DB 不存 raw token
- [ ] GET 不 consume
- [ ] atomic consume
- [ ] URL log redact
- [ ] Referrer Policy
- [ ] redirect allowlist
- [ ] 新 token 是否 revoke 舊 token

### OTP

- [ ] challenge-bound
- [ ] max attempts
- [ ] TTL
- [ ] successful consume
- [ ] resend cooldown

### Passkey

- [ ] RP ID 正確
- [ ] Origin validation
- [ ] challenge single-use
- [ ] credential lifecycle
- [ ] multiple passkeys
- [ ] lost-device recovery
- [ ] factor deletion re-auth

### Session

- [ ] Secure
- [ ] HttpOnly
- [ ] SameSite
- [ ] login rotate session
- [ ] session revoke
- [ ] device/session list
- [ ] suspicious event re-auth

### Recovery

- [ ] Recovery 是否比登入方式弱太多？
- [ ] Factor replacement 是否要求 existing factor？
- [ ] 是否有 cooldown？
- [ ] 是否通知既有聯絡方式？
- [ ] 是否 Audit？
- [ ] 高權限帳號是否禁止單一 Email recovery？

---

## 31. 最後的判斷原則

不要問：

```text
哪個登入方式最安全？
```

應該問：

```text
這個帳號被接管的代價多高？
```

然後決定需要多少 assurance。

---

### 低風險

```text
Magic Link
Email OTP
Social Login
```

可以合理使用。

---

### 中風險

```text
Passkey
OAuth / OIDC
+
Risk-based Step-up
```

通常是更好的平衡。

---

### 高風險

```text
Passkey / Security Key
+
強 Recovery
+
Session Security
+
Step-up
+
Transaction Authorization
+
Audit
```

不要把 Email 或 SMS 當成唯一 Root of Trust。

---

## 32. 總結

Magic Link 解決的主要問題是：

```text
弱密碼
Password reuse
Credential stuffing
Password reset 成本
登入摩擦
```

它沒有解決：

```text
Email account takeover
Phishing
Session theft
Recovery bypass
Factor replacement attack
```

因此：

```text
Magic Link
≠
高安全 Authentication
```

比較正確的定位是：

```text
低摩擦 Passwordless Authentication
```

Passkey 則是不同層級的設計：

```text
asymmetric cryptography
+
origin binding
+
challenge-response
+
phishing resistance
```

而真正成熟的帳號安全設計，不會只選一個「最強登入方式」，而是把：

```text
Primary Authentication
Recovery
Session
Step-up
Sensitive Action
Factor Replacement
Audit
```

全部視為同一條 Security Boundary。

---

## 限制與未查證事項

- 文中的 TTL（5～15 分鐘）、OTP 嘗試次數（5 次）、token 長度（256-bit）是常見實務值，不是規範硬性要求；實際數值依風險與 NIST SP 800-63B 等規範自行校準。
- 「WebAuthn Level 3 於 2026-08-25 成為 W3C Recommendation」與對應公告連結沿用原始資料，整理時未查證。
- 各方式的 Phishing / Replay 評級是定性比較，前提是正確實作；OAuth / OIDC 的實際強度取決於 IdP 與 account linking 設計。
- SQL 範例使用 `RETURNING`（PostgreSQL / SQLite 語法），MySQL 需改用交易內 `SELECT ... FOR UPDATE` 或檢查 affected rows。

## 參考資料

1. NIST Digital Identity Guidelines — Authentication and Authenticator Management
   https://pages.nist.gov/800-63-4/sp800-63b/

2. OWASP Authentication Cheat Sheet
   https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html

3. OWASP Multifactor Authentication Cheat Sheet
   https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html

4. OWASP Session Management Cheat Sheet
   https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html

5. FIDO Alliance — Passkeys
   https://fidoalliance.org/passkeys/

6. W3C — Web Authentication Level 3 Recommendation
   https://www.w3.org/TR/webauthn-3/

7. W3C — WebAuthn Level 3 Recommendation announcement, 2026-08-25
   https://www.w3.org/news/2026/web-authentication-an-api-for-accessing-public-key-credentials-level-3-is-now-a-w3c-recommendation/
