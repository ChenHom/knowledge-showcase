# Zeabur 2026 資安事件：攻擊鏈、信任邊界、偵測與事故應變

#security #cloud-security #zero-trust #least-privilege #secrets #kubernetes #incident-response #research

> 文件類型：Research / Knowledge / Security Review  
> 資料截點：2026-10-20。10 月事故仍屬初步調查，後續若有新證據須重新核對。

## 這是什麼

以 Zeabur 2026 年 8 月、10 月的兩次資安事件為案例，分析：PaaS 為何執行客戶容器、RCE 如何造成權限擴散、AWS Root Key 為何不應交給日常服務、CI/CD 與 Runtime 權限如何區隔、Secrets 與備份如何分權、Runtime/API 如何偵測異常，以及稽核紀錄與事故處理的工程要求。本文不是聊天逐字稿；「官方確認」「官方推定／推測」「工程建議」刻意區分。

## 核心結論

1. **Zero Trust：能連上內網，不代表有權讀取資源。** 網路可達、身分驗證、資源授權是三道不同的控制。
2. **最小權限必須同時限制動作、資源範圍、時間與身分。** 「唯讀」但能讀取全平台 Secrets，仍然是高風險權限。
3. **Secrets 可以集中管理，但不能集中暴露。** 單一 Admin Token 或維運帳號不應能還原所有租戶機密；短期憑證與完整輪替比存放位置更關鍵。
4. **備份也是資安邊界。** 避免不必要的明文機密，備份與解密金鑰分權，並把歷史副本納入憑證撤銷與輪替。
5. **監控不只看 5xx、CPU 和 Shell。** 要偵測管理 API、Token、Secrets 讀取、Kubernetes 管理操作、雲端高權限身分、資料庫查詢與跨環境連線。
6. **Audit Log 要能回答誰、何時、以什麼身分、從哪裡、對哪個資源、做什麼、結果如何。** 尤其要記錄敏感資料的「讀取」，而非只有修改。
7. **RCE 不是平台全陷落的充分條件。** 真正擴大影響的，是客戶工作負載、Control Plane、雲端帳號及正式環境間連續失效的安全邊界。
8. **事故處理不等於換密碼。** 必須找到全部副本、撤銷舊權限、切斷入侵路徑、清查持續存取方式，並從可信任環境重建。

## 一、Zeabur 的服務與信任邊界

Zeabur 是 PaaS，提供 GitHub／容器部署、資料庫、網域、HTTPS、環境變數與伺服器管理。開發者把 Laravel、NextChat、Node.js 等自己的程式交給平台執行；「使用者容器」因此是正常業務的一部分，不是 Zeabur 自己的管理程式。

```text
客戶／開發者
   │ 提交程式／部署設定
   ▼
Zeabur Control Plane（專案、部署、Secrets、資源管理）
   │ 建立／管理工作負載
   ▼
Data Plane（客戶 Container、Database、Queue）
   │
   ▼
底層 IaaS（AWS、Google Cloud 等）
```

- **Control Plane**：平台管理 API、身分、叢集與機密設定，權限高。
- **Data Plane**：不受平台信任的客戶程式碼與流量，應被限制。
- **Shared Cluster**：客戶共用底層叢集或節點，不代表允許客戶互通，更不代表可以碰平台管理服務。
- **Namespace 不是完整隔離機制**：還要 NetworkPolicy、RBAC、Service Account、Pod Security 與必要時的叢集／帳號隔離。
- **跨雲端不等於零信任**：AWS 叢集若能透過維運通道存取另一雲端的正式 DB，該通道仍是攻擊面。

## 二、8 月 27 日事件：RCE → Redis → API Key → AWS Root → 正式 DB

### 官方確認的攻擊鏈

```text
使用者部署的 NextChat v2.16.1
  └─ CVE-2026-7644／MCP 授權檢查不足造成 RCE
       ▼
客戶 Container 取得程式執行能力
       ▼
共享叢集內網連到平台路由 Redis
       ▼
讀取快取中的 Zeabur Legacy API Key
       ▼
挑出有效的高權限管理員 Key
       ▼
Zeabur 管理 API 列舉內部專案並讀取環境變數
       ▼
取得內部設定中長期保留的 AWS Root Access Key
       ▼
使用 AWS Root 身分操作雲端、控制東京共享叢集
       ▼
利用原有維運任務的私有網路與過大 DB 憑證
       ▼
讀取位於另一雲端的正式資料庫／使用者環境變數
```

這裡有兩個必須釐清的因果關係：

- **Redis 裡被讀到的是 Legacy API Key，不是 AWS Root Key。** Root Key 是攻擊者後續利用平台管理 API 讀到的內部專案設定。
- **RCE 不會自動取得 Root。** 攻擊者最初只取得 NextChat 容器的執行能力，是後續網路可達與授權過大，才形成逐層權限提升及橫向移動。

### 原本缺陷、應阻斷的位置與後續處理

| 跨越的邊界 | 原本失效的控制 | 理論上應如何限制 | Zeabur 8 月後宣告的處理 |
|---|---|---|---|
| 客戶 Container → 平台 Redis | 共享叢集跨專案內網可達、網路限制不足 | 預設拒絕客戶工作負載存取平台內部服務 | 停用相關舊服務、分離路由資料與憑證、清除舊快取 |
| Redis → 管理 API | 快取存高權限 Legacy API Key；Key 無 Scope 與明確期限 | 短期、限制資源與操作的 Token；憑證不進路由快取 | 全面停用 Legacy API Key，補強 Access Token 權限與紀錄 |
| 管理 API → 內部 Secrets | 管理員身分可列舉內部專案並讀機密設定 | 高權限操作獨立授權、敏感讀取稽核、專案隔離 | 補強 API 授權、敏感操作紀錄；專用伺服器支援環境變數改存本機 |
| Secrets → AWS | 內部設定含長期 AWS Root Key，超過日常需要 | Workload IAM Role／STS／短期憑證，禁止日常使用 Root | 撤銷已辨識憑證、檢查權限與共享叢集環境 |
| AWS 共享叢集 → 核心 DB | 維運工作同時有私有網路連線及過大 DB 讀取權 | 網路路徑、DB 權限、工作身分都要限縮 | 切斷共享叢集連回正式 DB 的路徑、撤銷帳號 |
| 入侵 → 告警 | 監控未辨識不正常的 API／DB 行為 | 跨系統日誌、關聯偵測、立即告警 | 補強叢集、網路、DB、快取紀錄與告警 |

官方事後回查：**8/27 07:54 UTC** 出現第一筆確認的正式資料庫入侵紀錄；**8/28 02:34 UTC** 收到外部 API Key 濫用通報才確認事故，相隔約 **18 小時 40 分鐘**。這是已知資料庫入侵至通報的時間差，不是整場攻擊的絕對起始時間。

## 三、10 月 14–15 日事件：合法憑證遭濫用

**10 月事件不是已證實再次從 NextChat RCE 開始。** 官方 10/19 初步報告指出，攻擊者持有有效內部 AWS Key、無期限的共享叢集管理 Token 與舊 GCP Service Account Key；**最初如何取得這些憑證仍未確認**。8 月外洩的 GitHub App 私鑰可能相關，屬官方推測，不能當成定論。

### 路徑 A：AWS／共享叢集（部分關節為推定）

```text
已持有內部 AWS Key、共享叢集管理 Token（取得方式未知）
    ▼
公網可達的 Kubernetes 管理介面 + 無期限 Token
    ▼
9 個區域共享叢集／服務 Secrets／容器指令
    ▼
[官方推定] 用過權限 AWS Key 讀取舊叢集備份
    ▼
備份含可讀取機密管理服務的 Key
    ▼
[確認] 讀取約 53 筆正式環境 Secrets
    └─ 含內網管理憑證、DB 帳號、GitHub App 私鑰
```

### 路徑 B：GCP／專用伺服器

```text
舊 GCP 服務帳號 Key（來源未知）
    ▼
[官方推定] 讀取初始化設定中的明文 Root 密碼
    ▼
[確認] 以 Root 登入 101 台 GCP 專用伺服器
    ├─ 其中 50 台被加入攻擊者 SSH 公鑰
    └─ 有客戶磁碟複本被掛載至新建 VM
    ▼
使用路徑 A 取得的內網管理憑證加入內部網路
    ▼
[確認] 內部網路規則允許存取平台正式 DB
    ▼
[確認] 以唯讀 DB 帳號大量讀取平台資料
```

### 確認影響（10/19 初步報告）

| 項目 | 數量／狀態 | 如何解讀 |
|---|---|---|
| 平台儲存的環境變數名稱與值 | 3,218,965 筆 | 確認遭讀取；不是 321 萬個獨立使用者 |
| 帳號資料 | 321,810 筆帳號 | 確認遭讀取 |
| zSend API Key | 566 把 | 明文遭讀取，官方已撤銷 |
| 平台 Access Token | 11,536 筆 | 讀到的是雜湊值，非可直接還原的明文 Token；已撤銷 |
| 使用者資料庫容器 | 865 個服務執行過讀取命令 | **不代表 865 個 DB 都完整外洩**，部分執行未成功、紀錄不全 |
| GCP 專用伺服器 | 101 台、92 名使用者 | 確認遭 Root 登入；其中 50 台被加 SSH Key |
| 正式環境 Secrets | 約 53 筆 | 確認讀取 |

### 為什麼 8 月處理後仍會再出事？

官方 10 月報告更正：8 月的憑證輪替**沒有涵蓋**私有設定儲存庫裡明文保存的共享叢集管理 Token 與 AWS Key。此外，GCP 初始化設定保存 Root 密碼的缺陷早在 9/3 就已被發現，但未在 10 月事件前修正。

第二次事故暴露三個額外問題：

1. **Rotate 不是只改目前的環境變數。** Git 歷史、CI/CD、備份、舊 Deploy Key、已退役服務、初始化設定都可能有可用的副本。
2. **備份可能把新憑證再次暴露。** 官方確認事故處理期間新輪替憑證也進入叢集備份，隨後遭攻擊者使用；是否就是由該備份取得新憑證，仍屬推定。
3. **合法 API 行為也可能是攻擊。** 從首次確認使用內部憑證（10/14 01:36 UTC）至團隊開始調查（10/15 13:56 UTC），約 **36 小時 20 分鐘**未觸發有效警報。

官方後續宣布：限制管理介面連線來源、撤銷受影響管理 Token／雲端 Key、限制 SSH、重設 Root 密碼、分離備份機密、限制平台 DB 存取、讓全部使用者重新登入。這些是官方處置**宣告**，不是外部獨立驗證已完成全面安全改善。

### 停止營運

10/17 官方宣布逐步停止營運；12/1 起部分服務停止或停止續費，12/31 前所有專用伺服器與共享叢集停止運作。資料刪除與最後匯出期限當時仍待公告。這是服務狀態與遷移背景，與上述技術缺陷分開判斷。

## 四、概念釐清：RCE、CI/CD、Runtime、Root、Secrets

| 概念 | 正確認知 | 常見誤解 |
|---|---|---|
| RCE | 攻擊者透過網路在目標程式的權限範圍內執行程式碼；命令注入、反序列化等都可能導致 RCE | 有 RCE 就一定取得作業系統 Root |
| Privilege Escalation | 取得更高的身分／權限，可能是盜用 Token 或雲端憑證，不必是 Kernel 漏洞 | 只會發生在 Linux 權限提升 |
| Lateral Movement | 從已控制服務沿網路、憑證或信任關係移到另一系統 | 只有同一網段才叫橫向移動 |
| AWS Root Access Key | 以 AWS 帳號最高身分呼叫 API，沒有日常維運必要性 | 必須登入 Console 才能使用 Root |
| CI/CD | Build／Test／Deploy 階段；應使用部署專用 Role／OIDC | 部署金鑰也給 Runtime 使用 |
| Runtime | Container／EC2／Pod 正式運作期間；只應取得業務必要權限 | Runtime 需要有基礎設施管理權 |
| Environment Variables | 將設定交給程式的管道，可在部署平台／容器執行環境中出現 | 放在環境變數就代表加密、安全 |
| Secrets Manager | 集中管理、授權與稽核機密；可配合短期憑證 | 集中管理本身就是錯 |
| Backup | 可供復原，但副本仍可能含密碼、簽章金鑰、Token | 備份是離線檔案就不會遭竊 |

**關於 8 月的 AWS Root Key：** 官方確認 Root Key 儲存在內部管理專案的環境變數**設定**，可透過平台管理 API 讀取。沒有證據可以直接等同於「攻擊者從執行中的容器記憶體讀到 Root Key」。Zeabur 也未公布原本保留這組 Root Key 的具體業務理由。

建議的身分分工：

```text
GitHub Actions ── OIDC ──► 部署專用 IAM Role（短期）
Laravel Runtime ────────► Workload IAM Role（僅 S3/SQS 等業務範圍）
維運 Job ───────────────► 獨立 Role + 限定網路與 DB 權限
Root Account ───────────► 僅特殊帳號層級操作，不作日常 API 身分
```

## 五、安全控制原則：應如何設計

| 原則 | 系統設計要求 | 優先的驗證問題 |
|---|---|---|
| Zero Trust | 服務間即使走內網，也驗證身分、用途與資源授權；預設拒絕非必要流量 | 客戶容器真的無法連內部 Redis／管理 API 嗎？ |
| Least Privilege | IAM Role、Service Account、DB 帳號分工；限操作、資源、時間 | 維運清理工作為何可以讀全平台環境變數？ |
| Tenant Isolation | 客戶間資料、網路與 Secret 權限隔離；跨租戶授權必須有明確條件 | 任一 Tenant Token 能列舉其他 Tenant 嗎？ |
| Secrets Management | 集中管理可行，但分權、分租戶、縮短憑證效期及輪替 | 某服務遭入侵，能取得多少別人的 Secrets？ |
| Backup Security | 不備份非必要明文 Secrets；必要機密加密；備份及解密金鑰獨立管理 | 備份若被讀走，攻擊者能直接還原有效憑證嗎？ |
| Credential Lifecycle | 盤點來源、副本、擁有者、Scope、到期、輪替、撤銷、退役 | 停止服務後，它的 Deploy Key 是否還有效？ |
| Security Monitoring | 把 API、權限、DB、Runtime、雲端事件關聯偵測 | 有效 Admin Token 大量讀取 Secret 會告警嗎？ |
| Audit Logging | 記錄敏感讀取及修改；保留行為與身分的關聯 | 能重建讀取路徑、時間、操作者及影響範圍嗎？ |
| Incident Response | 隔離、蒐證、撤銷、重建、驗證、通知 | 新金鑰會不會又被舊備份或仍存活的工作讀到？ |

### 5.1 為何「Secrets 不集中存放」不是正確結論？

集中 Secrets Manager 可以提供一致的存取稽核、權限管理與輪替；真正要避免的是**單一權限集中取得所有 Secret 明文**。

```text
Secrets Manager
    ├── Project A Secret ◄── Role A
    ├── Project B Secret ◄── Role B
    └── Project C Secret ◄── Role C

Role A 不可讀 B、C；管理 API 的一般 Admin 也不等於全域解密權限。
```

即使換成各客戶伺服器本機儲存，也要防止該主機被入侵後的機密讀取；**降低中央洩漏範圍 ≠ 消除機密風險**。

### 5.2 備份檔是否完全不能有敏感資訊？

不必絕對禁止：某些機密設定必須在災難復原時還原。但應該：
- 預設排除不必要的 Secret、明文密碼與長期私鑰。
- 必須保存者使用加密與獨立解密權限。
- 分開管理備份存取角色、KMS 金鑰及正式環境角色。
- 稽核備份的建立、列舉、下載、還原與解密操作。
- 輪替時清查歷史備份、映像檔、Git 歷史與快取裡是否還有**有效**的憑證。

## 六、如何判斷 Container 出現非預期程序？

不是「沒看過的 Process 就是惡意」，而是**程序的執行方式與該服務職責不一致**。

例：純 HTTP Laravel PHP-FPM 服務預期只有 PHP-FPM Master/Worker；若 Worker 執行 `sh`、`curl` 並向未知位置下載檔案，屬可疑行為。但影音處理 Worker 啟動 `ffmpeg` 可能完全合理。

| 訊號 | 應檢查的資訊 |
|---|---|
| Process | `execve`／`execveat`、執行檔路徑、命令參數 |
| Parent | 由誰啟動？`php-fpm` Worker 啟動 Shell 是否預期？ |
| Identity | 執行的 UID、容器身分、是否突升為 Root |
| Container | API／Queue／Scheduler／轉檔服務職責是否相符 |
| Filesystem | 是否執行了 `/tmp` 或新增可執行檔 |
| Network | 是否連到非預期的內部 Redis、管理 API、外部主機 |
| Kubernetes | 是否發生 `pods/exec`、建 Job、RoleBinding 權限變更 |

Falco／Tracee 等 Runtime Security 工具可在節點層取得程序及部分系統呼叫事件；Kubernetes Audit Log 則記錄管理 API 操作。兩者互補。

**重要盲點：RCE 不一定產生新的 Process。** 惡意 PHP/Node.js 可在原有程序內直接呼叫 Redis 或內部 API，完全不觸發「PHP-FPM 啟動 Shell」規則。因此應先用網路隔離阻擋不合理的連線，並同步監控 API、網路及身分行為。

建立偵測規則的順序：定義服務的**允許行為** → 觀察合法工作負載 → 規則僅告警與校準 → 對明確違規行為直接拒絕。不能把「歷史上出現過」全部列為合法，以免把早已存在的攻擊行為學成基準。

## 七、Audit Log 與安全監控如何落地？

### 7.1 審計事件的必要欄位

用結構化事件記錄「敏感操作」，不把 Secret 值寫進 Log：

```json
{
  "event": "secret.read",
  "occurred_at": "2026-10-20T07:00:00Z",
  "actor_id": "user_123",
  "identity_type": "service_account",
  "credential_id": "token_456",
  "tenant_id": "tenant_1",
  "resource_type": "project_secret",
  "resource_id": "project_789",
  "action": "read",
  "outcome": "allowed",
  "source_ip": "203.0.113.10",
  "request_id": "req_001"
}
```

- **Who／Identity**：Actor、Credential、Service Account，不能只記 IP。
- **When／Where**：UTC 時間、來源 IP／工作負載，支援跨系統時間關聯。
- **What／Target**：操作、資源與 Tenant；拒絕與成功都應記錄。
- **Result**：授權結果與錯誤原因分類，不寫 Token、密碼、Cookie、Secret 值。
- **Integrity**：送往權限隔離、具保留政策且盡可能防竄改的集中稽核系統。
- **Read access**：尤其記錄 `secret.read`、大量資料匯出、備份下載，不要只留 CRUD 中的變更。

Laravel 可在 Gate／Policy、Service 層或 Middleware／Event Listener 產生安全事件；針對敏感操作應在**真正執行授權與讀取的層級**產生紀錄，避免僅依 HTTP 200 誤判業務成功。

### 7.2 依 Zeabur 攻擊鏈建立偵測點

| 偵測點 | 事件來源 | 應觸發的訊號 |
|---|---|---|
| 客戶 Container 接觸內部 Redis | CNI／網路日誌 | 被 NetworkPolicy 拒絕的跨區連線 |
| Web Worker 啟動可疑 Shell | Falco／eBPF | 特定 Web 服務啟動非預期程序 |
| 管理 Token 從異常來源使用 | API Audit | 高權限 Token 的異常 IP、服務身分及資源 |
| 列舉大量專案與 Secrets | API Audit | 同 Token 短時大量跨租戶 `secret.read` |
| AWS Root 使用 | CloudTrail → EventBridge | 非預期 `userIdentity.type = Root` |
| Kubernetes 提升權限 | Kubernetes Audit | `cluster-admin`、`RoleBinding`、`pods/exec` 異常 |
| DB 大量讀取敏感設定 | DB Audit／Query Log | 維運帳號跨專案讀取大量資料 |
| 備份下載或解密 | Cloud Audit／KMS | 非預期 Role 讀取備份、金鑰被大量使用 |
| 稽核資料消失 | Log Pipeline | Event Drop、資料收集停擺或設定被更改 |

沒有一個單一偵測規則足以涵蓋所有 RCE。建議將事件以 `request_id`、`actor_id`、`credential_id`、`tenant_id`、工作負載身分及時間區間做關聯；IP 僅作輔助線索。

**偵測、阻擋與應變不同**：例如客戶容器連平台 Redis 應先在網路層拒絕，再產生警報；不能讓非法行為成功發生後才靠監控補救。

### 7.3 如何驗證偵測能力？

- 用**測試環境**模擬錯誤權限的讀取、非預期 `pods/exec`、異常 Root API 使用與跨租戶存取。
- 驗證事件是否被記錄、規則是否觸發、通知是否送達、處置是否有效。
- 持續量測 MTTD（平均偵測時間）、MTTR（平均應變／復原時間，需在團隊內統一定義）、偵測覆蓋率與誤報率。
- CloudTrail／EventBridge 等是非同步事件管道；告警不能被當成毫秒級同步阻擋機制。
- 日誌儲存必須與遭監控的正式服務分權；否則攻擊者取得管理權限時也可能刪除鑑識紀錄。

## 八、事故應變：為什麼只輪替金鑰會失敗？

建議的基本順序：

```text
判定事件及保留證據
    ↓
隔離遭入侵主機／網段／管理介面
    ↓
追查憑證所有副本與上下游權限
    ↓
撤銷外洩舊金鑰、Session、Token、Deploy Key
    ↓
在可信任環境重新核發最小權限憑證
    ↓
修正網路與資源授權、檢查備份／Git／映像檔
    ↓
對 Root／Cluster Admin 已遭奪取的環境重新建置
    ↓
驗證告警、異常行為是否終止，持續稽核
```

實務上保全證據、緊急阻擋與輪替可能需要並行，不能機械照表延誤止損。特別是曾失去 Root／Cluster Admin 控制的主機，不能僅刪可見的 SSH Key、CronJob 或 systemd Service 就認定乾淨。

## 九、Laravel／PHP 與雲端服務安全審查 Checklist

| 優先 | 檢查項目 | 驗收標準 |
|---|---|---|
| P0 | Tenant 資料隔離 | 非本租戶身分讀不到其他租戶資源，成功／拒絕測試都有 |
| P0 | API／Service 最小權限 | Token 具 Scope、期限、可撤銷，Admin 無預設跨租戶明文解密權 |
| P0 | Network Isolation | Runtime 不能直連平台管理網段；管理 DB、Redis 僅必要來源 |
| P0 | AWS／GCP 權限 | 日常無 Root Key；CI/CD 使用 OIDC；Runtime／維運使用不同 Role |
| P0 | DB 權限分工 | Laravel Runtime、Migration、備份及維運使用不同帳號 |
| P0 | 憑證清查與撤銷 | 能找到 Git、CI/CD、Container、備份、舊系統副本並使舊憑證失效 |
| P1 | Security Audit | 管理權限變更、Secret 讀取／匯出、DB 大量讀取都有結構化紀錄 |
| P1 | 監控與告警 | Admin／Root 使用與跨租戶 Secrets 大量讀取會觸發測試警報 |
| P1 | Backup／復原 | 加密、分權、保存下載紀錄，能用可信任復原流程還原 |
| P1 | Incident Runbook | 具備封鎖、撤銷、鑑識、重建、通報、回歸測試流程 |
| P2 | Runtime／SIEM | 程序、網路、Kubernetes、DB、雲端事件可關聯並追蹤成效 |

P0 表示應優先解決**阻斷或限制攻擊範圍**的問題；P1/P2 並非不重要，實際優先順序需按資料敏感度與部署方式調整。小型系統也能先以集中 Audit Log + 少量高價值規則起步，不需要立刻建置大型 SIEM。

## 十、已知、推定與限制

- **已確認（8 月）**：NextChat RCE → 路由快取 Legacy API Key → 管理 API → AWS Root → 共享叢集 → 正式 DB 的攻擊鏈；正式資料庫的部分環境變數與帳號資訊遭讀取。
- **已確認（10 月）**：長期管理憑證遭使用、9 個區域共享叢集受入侵、101 台 GCP 主機 Root 登入、平台資料庫大量讀取、部分憑證在 8 月後未被輪替。
- **尚未確認**：10 月攻擊者最初如何取得管理憑證；是否使用 8 月取得的 GitHub App 私鑰；是否透過備份取得再次外洩的新金鑰；受影響資料庫服務是否都成功完整遭讀取。
- **不是官方承諾**：本文的 NetworkPolicy、IAM Role、Audit Schema、偵測規則、P0/P1 排序是工程設計建議，並非 Zeabur 實際已落地的完整技術架構。
- **資訊時效**：第二次為初步調查，數據與應變項目可能修正。任何涉及受害範圍的判斷應回查最新公告。

## 資料來源

1. [Zeabur：2026 年八月資安事件（2026-09-07）](https://zeabur.com/zh-TW/blogs/august-2026-security-incident)
2. [Zeabur：2026 年十月資安事件初步報告（2026-10-19）](https://zeabur.com/zh-TW/blogs/october-2026-security-incident)
3. [Zeabur：停止營運公告與服務遷移安排（2026-10-17）](https://zeabur.com/zh-TW/blogs/zeabur-is-shutting-down)
4. [Zeabur 服務狀態與事故紀錄](https://status.zeabur.com/incidents)
5. [iThome：10 月事件初步調查與未完成的憑證輪替（2026-10-20）](https://www.ithome.com.tw/news/179804)
6. [Kubernetes NetworkPolicy 官方文件](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
7. [AWS IAM Root 使用者安全建議](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html)
8. [Falco Runtime Security 文件](https://falco.org/docs/)
