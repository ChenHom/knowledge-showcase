# Claw Notify：RSSHub、Firebase 與 PWA 推播收件箱架構

#rsshub #firebase #firestore #fcm #pwa #webpush #notification-system #openclaw #automation #obsidian

## 這是什麼

Claw Notify 是一套把多個內容來源整理成個人推播收件箱的輕量架構。

核心目標不是單純「發通知」，而是把通知從聊天軟體移出來，變成一套可回查、可分類、可擴充的通知資料流：

```txt
資料來源
→ RSSHub feed
→ 本機排程腳本
→ Firebase Firestore
→ Firebase Cloud Messaging
→ PWA 收件箱
```

目前接入的來源包含：

- 阮一峰科技愛好者週刊：`https://github.com/ruanyf/weekly`
- Jacky Bing-Sheng Lee：`https://jackybingshenglee.substack.com`
- JOJO 的運動產業觀察筆記：`https://jojo4sports.substack.com`

---

## 核心結論

這套架構裡，各元件的責任要切清楚：

- **RSSHub**：抓來源、轉 RSS、統一 feed 格式。
- **crontab / notify script**：定時讀 feed、比對新舊、組通知 payload。
- **Firestore**：保存歷史通知，扮演 cache / inbox database。
- **FCM**：把新通知推到手機。
- **PWA**：讓手機查看通知列表、分類、摘要與原文連結。

Firebase 不會自己抓 RSSHub；目前是本機排程主動把資料推進 Firebase。

---

## 系統流程

### 1. 資料來源層

原始來源格式不一致：

- GitHub repo / Markdown index
- Substack RSS / feed
- 未來也可能是網站、API、GitHub release、論壇、YouTube 等

因此不要讓通知系統直接耦合每個來源的解析方式，而是先交給 RSSHub 統一。

目前本機 RSSHub 服務：

```txt
/home/hom/services/RSSHub
http://127.0.0.1:1208
```

主要 feed：

```txt
http://127.0.0.1:1208/ruanyf/weekly?limit=3&opencc=s2t
http://127.0.0.1:1208/substack/subscribe/jackybingshenglee?limit=5&opencc=s2t
http://127.0.0.1:1208/substack/subscribe/jojo4sports?limit=5&opencc=s2t
```

其中 `opencc=s2t` 用於把內容轉成正體中文。

---

### 2. RSSHub 層

RSSHub 的角色是「資料轉換層」，不是通知器。

它負責：

- 抓取來源內容
- 解析來源結構
- 轉成標準 RSS item
- 提供 title / link / pubDate / description
- 用 query parameter 控制數量與繁中轉換

它不負責：

- 判斷是否已通知過
- 推播手機
- 儲存歷史通知
- 管理已讀 / 未讀

這個切分很重要，因為 RSSHub 應保持 stateless，通知狀態交給後面的 script / database 處理。

---

### 3. 排程與新舊比對

目前由本機 `crontab` 觸發兩支腳本：

```txt
/home/hom/.openclaw/workspace/bin/ruanyf_weekly_notify.py
/home/hom/.openclaw/workspace/bin/sports_substack_notify.py
```

排程規則：

```txt
阮一峰週刊：每週五 20:00，週六 09:00 補檢
運動日報：每天 08:00
```

腳本會做：

1. 讀 RSSHub feed
2. 抽出最新 item / posts
3. 與本機 state 檔比對
4. 若沒有新內容，直接結束
5. 若有新內容，組成 Firebase notification payload
6. 呼叫 Firebase sender

狀態檔：

```txt
/home/hom/.openclaw/workspace/memory/ruanyf-weekly-notify-state.json
/home/hom/.openclaw/workspace/memory/sports-substack-notify-state.json
```

這些 state 檔的用途是避免重複通知。

---

### 4. Firebase sender

Firebase sender 位於：

```txt
/home/hom/services/openclaw-notify-inbox/scripts/firebase_sender.js
```

它接收 notify script 傳來的 JSON payload，負責兩件事：

1. 寫入 Firestore `notifications` collection
2. 讀取 Firestore `devices` collection 裡的 FCM token，送出 Web Push

這裡的設計重點是：

> 先寫入 Firestore，再送推播。

因此即使推播失敗，通知資料仍會留在 inbox 裡，手機打開 PWA 還是能回查。

---

## Firebase 資料模型

Firebase project：

```txt
openclaw-notify-inbox-hom
```

PWA：

```txt
https://openclaw-notify-inbox-hom.web.app
```

本機 Firebase 專案：

```txt
/home/hom/services/openclaw-notify-inbox
```

Firestore database：

```txt
(default)
location: asia-east1
```

### notifications

`notifications` 是主要歷史資料表，每一筆代表一則可回查通知。

建議欄位：

```json
{
  "type": "tech_weekly",
  "source": "ruanyf_weekly",
  "sourceName": "阮一峰科技愛好者週刊",
  "title": "科技愛好者週刊（第 394 期）：第二次 API 開放浪潮",
  "summary": "摘要文字...",
  "url": "https://...",
  "publishedAt": "2026-04-23T23:46:46Z",
  "createdAt": "2026-05-06T...",
  "tags": ["tech", "weekly"],
  "read": false
}
```

目前類型：

```txt
tech_weekly    阮一峰週刊
sports_jacky   Jacky Bing-Sheng Lee
sports_jojo    JOJO 的運動產業觀察筆記
```

### devices

`devices` 保存手機 / 瀏覽器註冊的 FCM token。

PWA 按下「啟用推播」後，前端會取得 token 並寫入這個 collection。

典型欄位：

```json
{
  "token": "FCM token",
  "platform": "iPhone",
  "userAgent": "...",
  "createdAt": "...",
  "updatedAt": "..."
}
```

---

## PWA 接收端

Claw Notify 是 Firebase Hosting 上的一個 PWA。

目前 UI 先保持簡單：

- 標題：`Claw Notify`
- 類型篩選
- 重新整理
- 未啟用時顯示「啟用推播」按鈕
- 已啟用時隱藏推播按鈕
- 通知列表顯示來源、時間、標題、摘要、原文連結

PWA 相關檔案：

```txt
/home/hom/services/openclaw-notify-inbox/public/index.html
/home/hom/services/openclaw-notify-inbox/public/app.js
/home/hom/services/openclaw-notify-inbox/public/firebase-messaging-sw.js
/home/hom/services/openclaw-notify-inbox/public/manifest.json
```

Web Push 注意事項：

- Telegram 內建瀏覽器不支援 Firebase Web Messaging。
- iPhone 需要用 Safari 開啟，加入主畫面後從 PWA 啟用推播。
- Android 通常可直接用 Chrome 啟用。
- 若 Web Push token 取得失敗，可能需要在 Firebase Console 產生 VAPID key，填入 `public/firebase-config.js`。

---

## 為什麼不用 Telegram 當通知中心

Telegram 很適合聊天，但不適合做可治理的通知資料庫。

主要問題：

- 通知會混在對話裡
- 不好分類
- 不好回查
- 不適合擴充已讀 / 未讀 / source filter
- 多來源通知容易變成聊天噪音

改成 Firebase + PWA 後：

- Telegram 回到聊天用途
- 通知有自己的 inbox
- Firestore 保存歷史資料
- FCM 負責即時提醒
- PWA 負責查詢與閱讀

---

## 目前部署與運維位置

### RSSHub

```txt
/home/hom/services/RSSHub
systemd user service: rsshub-ruanyf.service
http://127.0.0.1:1208
```

### Notify scripts

```txt
/home/hom/.openclaw/workspace/bin/ruanyf_weekly_notify.py
/home/hom/.openclaw/workspace/bin/sports_substack_notify.py
```

### Firebase / PWA

```txt
/home/hom/services/openclaw-notify-inbox
https://openclaw-notify-inbox-hom.web.app
```

### Firebase sender

```txt
/home/hom/services/openclaw-notify-inbox/scripts/firebase_sender.js
```

### 本機 state / logs

```txt
/home/hom/.openclaw/workspace/memory/ruanyf-weekly-notify-state.json
/home/hom/.openclaw/workspace/memory/sports-substack-notify-state.json
/home/hom/.openclaw/workspace/memory/ruanyf-weekly-notify.log
/home/hom/.openclaw/workspace/memory/sports-substack-notify.log
```

---

## 這套架構的取捨

### 優點

- 資料來源與推播通道解耦
- Firestore 可保存歷史通知
- PWA 可依類型瀏覽與回查
- RSSHub 可繼續接更多來源
- Telegram 不再被通知洗版

### 限制

- Firebase 不會自己抓資料；依賴本機 cron 正常運作
- 本機 RSSHub 掛掉時，排程抓不到 feed
- Web Push 在 iOS 上需要 PWA / Safari 條件
- Firebase sender 目前是輕量自製腳本，不是完整後台服務

---

## 後續可擴充方向

- 加入已讀 / 未讀同步
- 加入通知搜尋
- 加入 source 設定頁
- 加入摘要品質分級
- 將排程狀態寫入 Firestore，例如 `jobs` collection
- 改成 Cloud Run / Cloud Scheduler，讓 Firebase/GCP 端自行拉 RSSHub 或來源 API
- 將 RSSHub 也容器化部署到雲端，降低本機依賴
- 加入通知失敗重試與 dead-letter queue

---

## 可重用設計原則

這套流程可抽象成：

```txt
Source Adapter → Normalized Feed → Dedup State → Notification Store → Push Channel → Reader UI
```

其中最重要的原則是：

1. **來源解析與通知推送分離**
2. **先持久化，再推播**
3. **推播只做提醒，完整內容放 inbox**
4. **每個來源都要有 stable id / guid，避免重複通知**
5. **本機 state 負責排程 dedup，Firestore 負責使用者可見歷史**

這樣未來要新增來源時，只要新增 feed route 與 payload mapping，不需要重寫整套推播系統。

---

## 補充：摘要生成與防回歸設計

後續修正後，Claw Notify 不應把 RSS 原文片段直接塞進推播 body。推播應顯示整理後的 `summary`，完整內容與原文連結留在 inbox 裡。

目前摘要流程應維持：

```txt
RSS item title / description
→ notify_summary.py
→ ollama-presets/scripts/ollama-article
→ clean_article_text / forbidden pattern cleanup
→ Firebase payload.summary
→ Firestore notifications.summary
→ FCM push body = doc.summary
→ PWA 顯示 summary
```

### 設計重點

1. **摘要器走 skill，不直接打 Ollama API**  
   本機腳本應呼叫：

   ```txt
   /home/hom/.agents/skills/ollama-presets/scripts/ollama-article
   ```

   不要在通知腳本中直接呼叫：

   ```txt
   http://127.0.0.1:11434/api/generate
   ```

2. **fallback 也不能回退成原文截斷**  
   如果 Ollama 摘要失敗，fallback 應使用 title-based summary，而不是把 RSS description 前幾百字直接塞出去。

3. **清理原文雜訊**  
   summary cleaner 至少要移除：

   - `http://` / `https://`
   - `pan.baidu`
   - `提取碼`
   - `複製這段內容`
   - 下載鏈接 / raw link noise

4. **推播 body 使用 `doc.summary`**  
   Firebase sender 寫入 Firestore 後，送 FCM 時 body 應優先使用 `doc.summary`，不要用原始 description。

5. **只推最新 device token**  
   如果同一支手機 / PWA 反覆註冊 device，sender 應排序 `updatedAt` / `createdAt` 後只取最新一筆，避免同一則通知連續響多次。

---

## 補充：Claw Notify Live Eval

Claw Notify 的 regression eval 不只檢查程式碼，也要檢查 Firestore live state。

目前第一版 live eval 會看：

- 最新 Firestore `notifications`
  - `summary` 不可空白
  - 不可含 URL
  - 不可含 `pan.baidu`
  - 不可含 `提取碼`
  - 不可含「複製這段內容」
  - 不可過長
  - 不可像 title + raw excerpt
- Firestore `devices`
  - 至少有一筆 device
  - sender 程式仍維持只推最新 token

對這類通知系統來說，live eval 很重要，因為程式碼看起來正確，不代表實際 Firestore 裡的新資料沒有退化。

可用 runner：

```bash
cd /home/hom/.openclaw/workspace
python3 evals/runners/system_regression_eval.py
```

這個 eval 的定位是：

> 把「新的通知還是沒有摘要」這種真實踩過的問題，變成以後每次都能自動抓到的防回歸檢查。
