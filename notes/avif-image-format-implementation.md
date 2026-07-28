# AVIF 圖片格式落地：能力檢測、CDN 快取與異步轉碼設計

#avif #web-performance #image-optimization #cdn #async-processing #frontend #backend #webp #lcp #obsidian

## 來源

- SegmentFault：下一代圖片格式 AVIF 在 vivo 社區的落地實踐  
  https://segmentfault.com/a/1190000047749433
- 整理日期：2026-05-08

---

## 這是什麼

這篇筆記整理一套在內容型網站落地 AVIF 圖片格式的工程方案。

核心不是單純「把圖片改成 AVIF」，而是建立一條安全、可回退、可快取、可觀測的圖片交付鏈路：

```txt
前端能力檢測
→ 選擇最佳圖片格式
→ CDN 優先命中
→ 後端檢查衍生檔是否存在
→ 不存在則觸發異步轉碼
→ 前端先降級 WebP / JPG
→ 轉碼完成後後續請求命中 AVIF
```

這種做法適合圖片流量大、首屏圖片多、弱網體驗重要，且已經使用 WebP 但還想進一步壓縮體積的網站。

---

## 核心結論

AVIF 的價值在於更好的壓縮效率，但真正能安全上線的關鍵是四件事：

1. **能力檢測**：不要靠 user-agent 猜瀏覽器支援度，而是實測瀏覽器能不能解碼 AVIF。
2. **前端降級**：AVIF 載入失敗時自動退回 WebP，再不行退回 JPG，避免圖片破圖。
3. **CDN 快取**：同一張圖轉碼後要能被 CDN 邊緣快取，避免重複回源與重複計算。
4. **異步轉碼**：不要在使用者請求鏈路上即時轉 AVIF，因為 AVIF 編碼成本高；首次 miss 時應排背景任務，使用者先看 fallback。

一句話：**AVIF 是格式優化，落地時其實是在做一套圖片衍生檔生成與交付系統。**

---

## 為什麼選 AVIF

AVIF 基於 AV1 編碼，通常在相同主觀畫質下比 JPEG / WebP 有更好的壓縮率。

vivo 社區的案例中：

- 原本已用 WebP 把圖片體積約減少 50%。
- 導入 AVIF 後，希望在 WebP 基礎上再進一步降低體積。
- 目標是 PSNR >= 35 dB 時，AVIF 比 WebP 再縮小 20% 以上。
- 實測中 AVIF 相比 JPG 體積優化約 77%–90%。
- AVIF 相比 WebP 進一步減少約 30%–40%。
- 線上 A/B 後，LCP 約改善 15%–25%，首屏圖片載入效率提升 30%+，總頻寬下降 30%+。

但 AVIF 不是所有場景都必然更好。小圖、透明 PNG、文字海報、動畫圖，都需要另外測試收益與畫質。

---

## 能力檢測是什麼

能力檢測是前端確認「目前瀏覽器真的能顯示 AVIF」。

不要只看瀏覽器名稱或 user-agent，因為：

- UA 容易被偽裝或截斷。
- WebView / PWA / 內建瀏覽器支援狀態可能不同。
- 同一瀏覽器不同版本、不同平台支援度不同。
- 有些環境看起來支援，但實際 decode 可能失敗。

更可靠的做法是：建立一個 `Image()`，塞一張極小的 AVIF base64 測試圖，讓瀏覽器實際解碼，然後檢查圖片尺寸是否符合預期。

---

## 能力檢測怎麼做到

概念範例：

```js
let avifSupportPromise;

function supportsAvif() {
  if (avifSupportPromise) return avifSupportPromise;

  avifSupportPromise = new Promise(resolve => {
    const img = new Image();
    img.onload = img.onerror = () => {
      resolve(img.height === 2);
    };
    img.src = 'data:image/avif;base64,...'; // 極小 AVIF 測試圖
  });

  return avifSupportPromise;
}
```

關鍵點：

- `Image()` 會走瀏覽器自己的圖片解碼能力。
- `onload` 且尺寸正確，才視為支援。
- `onerror` 或尺寸不對，視為不支援。
- 結果要快取，不能每張圖都測一次。

---

## 怎樣做才有效率

推薦策略：

1. App 初始化時測一次。
2. 把結果存在全域狀態或 Promise，避免重複測。
3. 同一個 session 內直接重用結果。
4. 若要跨 session 快取，可放 `localStorage`，但要有版本或 TTL。

範例：

```js
async function getImageFormat() {
  const cached = localStorage.getItem('img-format:v1');
  if (cached) return cached;

  const ok = await supportsAvif();
  const format = ok ? 'avif' : 'webp';

  localStorage.setItem('img-format:v1', format);
  return format;
}
```

更保守的做法是只長期快取成功結果，失敗結果短期快取即可。原因是某些 WebView、PWA 或初始化時機問題可能造成偶發檢測失敗，不一定代表永久不支援。

也可以由後端根據 `Accept` header 判斷格式：

```txt
Accept: image/avif,image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
```

但實務上前端能力檢測仍有價值，因為它更接近「這個 runtime 是否真的能 decode」。大型系統也可以採混合策略：後端看 `Accept` 做初步協商，前端再用 decode test 與 onerror fallback 保底。

---

## URL 與圖片格式選擇

前端取得格式後，組出穩定 URL：

```js
function getOptimizedImageUrl(imageId, format) {
  return `https://cdn.example.com/images/${imageId}.${format}`;
}
```

也可以用 query string：

```txt
https://cdn.example.com/images/abc?format=avif&w=800&q=75
```

設計重點：

- URL 必須穩定，方便 CDN 快取。
- cache key 要包含格式、尺寸、品質參數。
- 原圖 ID、尺寸、格式、quality 要能推導出唯一衍生檔。
- 不要讓同一張衍生圖有太多等價 URL，否則 CDN 命中率會下降。

---

## 前端降級策略

AVIF 請求失敗時，前端應該自動退回 WebP；WebP 也失敗時，再退回 JPG。

概念範例：

```html
<img id="cover" src="/images/abc.avif" />

<script>
const img = document.querySelector('#cover');

img.onerror = function () {
  const current = this.src;

  if (current.endsWith('.avif')) {
    this.src = current.replace(/\.avif$/, '.webp');
    return;
  }

  if (current.endsWith('.webp')) {
    this.src = current.replace(/\.webp$/, '.jpg');
  }
};
</script>
```

實際產品中可以封裝成 Image component，統一處理：

- AVIF / WebP / JPG fallback。
- placeholder / skeleton。
- lazy loading。
- loading error telemetry。
- 首屏圖片 preload。

---

## 異步轉碼如何處理

AVIF 編碼通常比 WebP 慢，不適合在使用者請求當下即時轉碼。比較穩的方式是「miss 時背景補貨」。

流程：

```txt
Browser
  ↓ request abc.avif
CDN
  ↓ cache miss
Image API
  ├─ AVIF exists? yes → return avif
  └─ AVIF exists? no
      ├─ enqueue convert job
      └─ return 404 / 202 / fallback

Worker
  ↓ consume job
Original image storage
  ↓ convert to AVIF
AVIF storage
  ↓
CDN cache on next request
```

使用者第一次看到的可能是 WebP，背景轉碼完成後，下一次或下一位使用者就會吃到 AVIF。

---

## 異步轉碼的後端設計

一個實用的後端可以拆成：

```txt
Object Storage：保存原圖與衍生圖
Image API：處理圖片請求、檢查衍生檔狀態、觸發任務
Queue：保存轉碼任務
Worker：執行轉碼
Metadata DB：保存圖片衍生檔狀態
CDN：快取衍生圖
```

狀態表可以長這樣：

```txt
image_id
format
width
quality
status: pending | done | failed
source_etag
output_url
error_message
retry_count
created_at
updated_at
```

圖片請求時：

1. 根據 `image_id + format + width + quality` 算出衍生檔 key。
2. 檢查 storage 是否已有該檔。
3. 有就回傳或 redirect 到 CDN URL。
4. 沒有就檢查狀態表。
5. 若沒有 pending 任務，建立任務。
6. 回傳 404 / 202 / fallback，讓前端降級。

---

## 任務去重與鎖

異步轉碼最重要的是避免同一張圖被大量併發請求觸發重複轉碼。

可用 Redis lock：

```txt
SETNX convert:abc:avif:800:q75 1 EX 300
```

拿到鎖才 enqueue 任務；拿不到鎖代表已經有人排隊或正在轉。

也可以靠資料庫唯一索引：

```txt
UNIQUE(image_id, format, width, quality, source_etag)
```

如果 insert pending 失敗，就代表任務已存在，不再重複排。

---

## Worker 轉碼流程

Worker 的基本流程：

```txt
1. 取出 queue job
2. 下載原圖
3. 檢查圖片類型、尺寸、是否適合轉 AVIF
4. 執行轉碼
5. 可選：計算 PSNR / SSIM 或抽樣做品質檢查
6. 上傳 AVIF 到 storage
7. 更新 metadata status = done
8. 清除 lock
```

失敗時：

- 記錄錯誤。
- retry 次數有限制。
- 對永久失敗的圖片標記 `failed`，避免每次請求都重試。
- 可設定冷卻時間，例如 24 小時後才允許再試。

---

## 品質與轉碼參數

AVIF 常見會用 CRF / quality 類參數控制體積與畫質。vivo 案例提到 CRF=32、CRF=35 的調優比較，CRF35 體積更小但 PSNR 略降。

上線前應該分層測試：

- 1MB 以下小圖。
- 1–5MB 中圖。
- 5MB 以上大圖。
- JPG 原圖。
- PNG 原圖。
- 透明圖。
- 文字海報。
- 人物 / 風景 / 商品圖。

不要只看平均壓縮率，因為不同圖片類型收益差很多。

---

## 什麼圖不一定適合 AVIF

以下類型要特別小心：

- 很小的 icon：轉碼收益低，額外格式反而增加管理成本。
- 透明 PNG：要確認 alpha 支援、邊緣品質與體積收益。
- 文字海報：壓縮過度可能讓字邊糊掉。
- 動圖：AVIF animation 支援與相容性要另外評估。
- 已高度壓縮的 WebP / JPEG：再轉可能收益有限或畫質劣化。

可建立規則：

```txt
小於 20KB：不轉
寬高太小：不轉
透明 PNG：保守參數或跳過
文字比例高：較高品質參數
大圖：優先轉 AVIF
```

---

## 上線觀測指標

要判斷 AVIF 是否真的有效，至少觀測：

- AVIF 請求比例。
- AVIF CDN 命中率。
- AVIF 轉碼成功率。
- AVIF fallback 率。
- 圖片載入錯誤率。
- LCP / FCP / CLS。
- 首屏圖片載入時間。
- 圖片總頻寬。
- Worker queue length。
- 轉碼平均耗時與 P95 / P99。
- 轉碼失敗原因分布。

如果 AVIF fallback 率很高，可能代表：

- 能力檢測錯誤。
- 衍生檔還沒轉完。
- CDN / storage path 設計錯。
- 某些瀏覽器宣稱支援但 decode 有問題。

---

## 最小可行版本

第一版不要做太滿，可以先做：

```txt
前端：AVIF 能力檢測 + onerror fallback
後端：AVIF 不存在就 enqueue 任務
Worker：轉 AVIF 寫回 storage
CDN：快取轉好的 AVIF
觀測：記錄 fallback 與轉碼成功率
```

這樣已經能驗證核心收益。

等第一版穩定後，再補：

- 多尺寸衍生圖。
- 品質參數動態調整。
- PSNR / SSIM 抽樣檢查。
- 灰度發布。
- 按圖片類型決策是否轉碼。
- 轉碼任務優先級。
- 熱門圖片預轉碼。

---

## 與傳統即時轉碼的差異

不建議：

```txt
使用者請求圖片 → 後端立即轉 AVIF → 等轉完才回應
```

原因：

- AVIF 編碼慢，會拉高 TTFB。
- 高併發下 CPU 壓力不可控。
- 熱點圖片可能被重複轉碼。
- 使用者體驗不穩。

建議：

```txt
使用者請求 AVIF → 沒有就排背景任務 → 使用者先看 WebP → 下次命中 AVIF
```

這個策略把 CPU 成本移出請求鏈路，用 CDN 快取攤平成本。

---

## 一句話設計準則

AVIF 落地不要只想「改副檔名」。應該把它當成圖片服務能力升級：

```txt
正確檢測支援度
穩定產生衍生檔
用 CDN 放大轉碼收益
用 fallback 保證可用性
用觀測數據決定是否擴大
```
