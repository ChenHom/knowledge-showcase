# 大量資料處理效能優化：從 1100 萬筆事件 Replay 到每秒 45000 筆

#database #performance #event-sourcing #batch-processing #php #mysql #optimization

## 這是什麼

這份筆記整理自 Brent Roose 的文章〈Processing 11 million rows〉：作者在搬遷部落格 analytics 系統時，需要重新 replay 超過 1,100 萬筆事件，並逐步把處理速度從約 `30 events/sec` 優化到約 `45,000 events/sec`。

原文：<https://stitcher.io/blog/processing-11-million-rows>

這份文件不是逐字翻譯，而是把文章中的效能優化知識點整理成可回查、可套用的工程筆記。

## 核心結論

大量資料處理的效能優化，核心不是單一技巧，而是持續減少幾類成本：

- 減少不必要的資料庫排序。
- 減少重複讀取同一批資料。
- 減少 ORM / framework abstraction 的 per-row overhead。
- 避免大 offset pagination。
- 減少 database round trip。
- 減少 commit / fsync 次數。
- 用 profiler 找真正瓶頸，而不是只靠直覺。
- 每次只改一件事，並記錄吞吐量變化。

文章中的效能變化大致如下：

| 步驟 | 優化內容 | 效能 |
|---|---|---:|
| Baseline | 原始版本 | 30/sec |
| 1 | 移除 `createdAt` 排序 | 6,700/sec |
| 2 | 反轉 projector/event loop | 6,800/sec |
| 3 | 避開 ORM | 7,800/sec |
| 4 | 改 while loop、調整 batch size | 8,400/sec |
| 5 | 修正 framework scalar binding overhead | 14,000/sec |
| 6 | Buffered inserts | 19,000/sec |
| 7 | Transaction | 45,000/sec |

## 背景 / 問題

作者的網站使用 server-side anonymous analytics。每次有人造訪網站時，系統會記錄事件，之後由不同 projectors 產生統計資料，例如：

- 每日造訪數
- 每月造訪數
- 每週熱門文章
- 每年統計
- 圖表資料

這是典型 event sourcing 場景：歷史事件完整保存，未來若新增統計方式，只要 replay 舊事件即可重建資料。

問題是五年下來累積超過 1,100 萬筆 visit events。當作者要把舊 Laravel 系統搬到 Tempest 並重建 projectors 時，原始 replay command 每秒只能處理約 30 筆事件。若有多個 projectors，總耗時可能達數十小時。

## 優化流程

### 1. 先建立 baseline

效能優化前，先量測原始速度。

原始流程大致是：

```php
foreach ($projectors as $projector) {
    $projector->clear();

    StoredEvent::select()
        ->orderBy('createdAt ASC')
        ->chunk(500, function ($events) use ($projector) {
            foreach ($events as $event) {
                $projector->replay($event);
            }
        });
}
```

問題包括：

- 每個 projector 都重新讀一次全部 events。
- 對 `createdAt` 排序。
- 使用 ORM hydration。
- 寫入可能是一筆一筆送。
- 沒有明確 transaction。

初始吞吐量約：

```text
30 events/sec
```

重點：不要靠感覺猜瓶頸。先取得 baseline，後續每一步才知道是否真的有效。

### 2. 移除不必要排序

原本 query 依 `createdAt ASC` 排序：

```php
->orderBy('createdAt ASC')
```

但事件本來就是依序寫入資料庫，若沒有額外需求，這個排序可能不必要。尤其當排序欄位沒有 index 時，對大量資料排序非常昂貴。

移除後速度大幅提升：

```text
30/sec → 6,700/sec
```

經驗：

- 大量資料處理時，`ORDER BY` 必須非常小心。
- 若必須排序，排序欄位應該有合適 index。
- 若資料本身已按插入順序排列，可以考慮用 indexed `id` 取代時間欄位排序。

### 3. 反轉 loop，避免重複讀資料

原始流程是：

```text
for 每個 projector:
    讀全部 events
    for 每個 event:
        projector replay event
```

如果有 12 個 projectors，就會把 1,100 萬筆事件讀 12 次。

改成：

```text
讀一批 events
for 每個 projector:
    for 每個 event:
        projector replay event
```

概念範例：

```php
StoredEvent::select()
    ->chunk(500, function ($events) use ($projectors) {
        foreach ($projectors as $projector) {
            foreach ($events as $event) {
                $projector->replay($event);
            }
        }
    });
```

單一 projector 的速度提升有限：

```text
6,700/sec → 6,800/sec
```

但在多 projector 場景，這個改法可以避免大量重複 I/O。

經驗：如果資料量遠大於處理器數量，應優先讓大量資料只被讀一次。

### 4. 避開 ORM

ORM 對一般開發很方便，但大量批次處理時，每一筆 row 都被轉成 model 會產生明顯成本。

可把 ORM query：

```php
StoredEvent::select()
```

改為更接近資料庫的 query builder / raw query：

```php
query('stored_events')->select()
```

概念範例：

```php
query('stored_events')
    ->select()
    ->chunk(1500, function (array $rows) use ($projectors) {
        $events = array_map(function ($row) {
            return $row['eventClass']::unserialize($row['payload']);
        }, $rows);

        foreach ($projectors as $projector) {
            foreach ($events as $event) {
                $projector->replay($event);
            }
        }
    });
```

速度提升：

```text
6,800/sec → 7,800/sec
```

經驗：ORM 是方便性工具，不是免費工具。對數百萬筆以上的批次處理，應評估是否改用 raw query / query builder。

### 5. 自己控制 batch loop，並實測 batch size

Framework 的 `chunk()` helper 也可能有額外成本。作者改用 `while` loop 自己控制批次讀取，並測試不同 batch size，最後找到在他的環境中較好的數值。

概念範例：

```php
$offset = 0;
$limit = 1500;

while (true) {
    $rows = query('stored_events')
        ->select()
        ->limit($limit)
        ->offset($offset)
        ->all();

    if ($rows === []) {
        break;
    }

    // process rows...

    $offset += $limit;
}
```

速度提升：

```text
7,800/sec → 8,400/sec
```

經驗：

- Batch size 需要實測。
- 太小會增加 round trip。
- 太大可能增加 memory pressure 或降低 cache locality。
- 不同資料庫、資料形狀、硬體環境的最佳值不同。

### 6. 不要迷信直覺優化：serialization 嘗試反而更慢

事件 payload 需要 unserialize。作者猜測若跳過 unserialize、改從原始 visits table 手動建立 event 物件，可能會更快。

實測結果反而更慢。

經驗：

- 看似合理的優化不一定有效。
- PHP 內建或成熟框架功能可能已被高度優化。
- 每個改動都要實測，不要只靠直覺。

### 7. 用 profiler 找出真正瓶頸

作者使用 Xdebug profiler 後發現：每處理一批 events，Tempest 的 reflection 相關工具被呼叫大量次數。

原因是 framework 的 database layer 在處理 query binding 時，會檢查每個值是否需要 serializer。這對 object 可能合理，但對 string / number 這類 scalar value 沒必要。

優化方向是先短路處理 scalar value：

```php
if ($value instanceof Query) {
    $value = $value->execute();
} elseif (is_string($value) || is_numeric($value)) {
    // scalar value，直接保留，不進 serializer factory
} elseif ($serializer = $serializerFactory->forValue($value)) {
    $value = $serializer->serialize($value);
}
```

速度提升：

```text
8,400/sec → 14,000/sec
```

經驗：

- 真正瓶頸常常藏在 framework 或 library 的通用路徑裡。
- Profiler 可以揭露肉眼很難看出的 per-row overhead。
- 熱路徑中的 reflection、dynamic dispatch、serializer lookup 都要特別小心。

### 8. 避免大 offset pagination

長時間 replay 時，作者發現一開始速度快，但跑越久越慢。記憶體沒有持續增加，因此不像 memory leak。

問題可能出在 offset pagination：

```sql
LIMIT 1500 OFFSET 9000000
```

資料庫可能仍需掃過前面大量 rows，只是最後丟掉。offset 越大，後面越慢。

改成 keyset pagination / seek pagination：

```php
$lastId = 0;
$limit = 1500;

while (true) {
    $rows = query('stored_events')
        ->select()
        ->where('id > ?', $lastId)
        ->orderBy('id ASC')
        ->limit($limit)
        ->all();

    if ($rows === []) {
        break;
    }

    // process rows...

    $lastId = end($rows)['id'];
}
```

經驗：

- 大量資料分頁避免大 offset。
- 優先使用 indexed 欄位，例如 `id > last_id`。
- 這個改法不一定提高初始峰值，但能讓長時間處理更穩定。

### 9. Buffered inserts：減少 database round trip

如果 projector 每處理一筆 event 就寫一次資料庫，會產生大量 round trips。

可把寫入先 buffer 起來，等一批處理完再一起送出。

概念 interface：

```php
interface BufferedProjector extends Projector
{
    public function persist(): void;
}
```

概念 trait：

```php
trait BuffersQueries
{
    private array $queries = [];

    protected function buffer(string $sql): void
    {
        $this->queries[] = $sql;
    }

    public function persist(): void
    {
        if ($this->queries === []) {
            return;
        }

        query()->executeMany($this->queries);

        $this->queries = [];
    }
}
```

Projector 不立即寫 DB，而是先 buffer：

```php
public function onPageVisited(PageVisited $event): void
{
    $date = $event->visitedAt->format('Y-m-d');

    $this->buffer("
        INSERT INTO visits_per_day (`date`, `count`)
        VALUES ('$date', 1)
        ON DUPLICATE KEY UPDATE `count` = `count` + 1
    ");
}
```

Replay loop 每批 flush：

```php
foreach ($projectors as $projector) {
    foreach ($events as $event) {
        $projector->replay($event);
    }

    if ($projector instanceof BufferedProjector) {
        $projector->persist();
    }
}
```

速度提升：

```text
14,000/sec → 19,000/sec
```

注意：上面的 SQL 是概念範例。實務上應使用 prepared statements、bulk insert builder 或安全的 escaping，避免 SQL injection。

### 10. Transaction：減少 commit / fsync 成本

最後的巨大提升來自 transaction。

如果沒有明確 transaction，每次 insert/update 都可能成為 implicit transaction。對 InnoDB 這類資料庫來說，頻繁 commit 會導致大量 fsync 成本。

概念上：

```text
20,000 inserts = 20,000 commits = 很多次 fsync
```

包進 transaction 後：

```text
20,000 inserts = 1 transaction = 1 commit
```

概念範例：

```php
$database->withinTransaction(function () use ($projectors, $events) {
    foreach ($projectors as $projector) {
        foreach ($events as $event) {
            $projector->replay($event);
        }

        if ($projector instanceof BufferedProjector) {
            $projector->persist();
        }
    }
});
```

速度提升：

```text
19,000/sec → 45,000/sec
```

經驗：大量寫入時，transaction 常常是最有力的優化之一。

## 可套用的優化 checklist

大量資料處理前後，可以用這份 checklist 檢查：

```text
[ ] 有沒有建立 baseline？
[ ] 有沒有每一步記錄吞吐量？
[ ] Query 是否有不必要 ORDER BY？
[ ] ORDER BY 欄位是否有 index？
[ ] 是否用了大 OFFSET？
[ ] 能不能改成 id > last_id？
[ ] 是否重複撈同一批資料？
[ ] 能不能資料讀一次，多個處理器共用？
[ ] ORM hydration 是否是瓶頸？
[ ] Batch size 是否實測過？
[ ] 是否一筆一筆寫 DB？
[ ] 能不能 buffer writes？
[ ] 能不能 bulk insert / bulk update？
[ ] 是否包 transaction？
[ ] 有沒有用 profiler 找真正瓶頸？
```

## 常見踩坑

### 不必要排序

大量資料上的未索引排序，可能是最昂貴的操作之一。若資料本身已按寫入順序排列，先確認排序是否真的必要。

### Offset pagination

`OFFSET` 在後期會越來越慢。大量掃描應優先用 keyset pagination：

```sql
WHERE id > :last_id
ORDER BY id ASC
LIMIT :limit
```

### ORM per-row overhead

ORM 會做 hydration、mapping、型別處理、可能還有 event hooks。對一般 CRUD 很方便，但在千萬級資料處理中可能成為熱路徑成本。

### 小 query 太多

即使單次 query 很快，幾百萬次 round trip 也會非常慢。應考慮 buffer、bulk execute、bulk insert、bulk update。

### Commit 太頻繁

大量 insert/update 若沒有 transaction，可能會被 implicit commit 拖垮。明確 transaction 可以大幅降低 fsync 成本。

### 沒有 profiler

直覺只能找到一部分問題。Framework 內部的 reflection、serializer lookup、binding handling 等熱路徑瓶頸，通常需要 profiler 才看得到。

## 工程思維

### 每次只改一件事

效能優化要能歸因。一次改太多，速度變快也不知道是哪個改動有效；速度變慢也不知道是哪個改動造成問題。

### 先解最大瓶頸

移除排序從 30/sec 到 6,700/sec，比許多微優化有效得多。先找高成本操作，再處理細節。

### 不要過度抽象

大量資料處理通常是熱路徑。熱路徑上要盡量簡單、直接、可預測。

### 對方便性工具保持警覺

ORM、chunk helper、serializer factory、reflection 都可能在大量資料處理中變成成本。不是不能用，而是要知道何時該繞過。

### 寫入優化常比讀取優化更關鍵

讀取可以透過 index、分批、減少排序改善；寫入則要注意 round trip、bulk operation、transaction、lock contention、index 更新成本。

## 延伸應用

這些原則不只適用 PHP / MySQL，也適用於多數大量資料處理場景：

- Event sourcing replay
- Analytics aggregation
- Data migration
- Backfill jobs
- ETL / ELT pipeline
- Reindexing
- Materialized view rebuild
- Audit log replay
- Historical report recomputation

可概括為一句話：

> 大量資料處理要減少排序、減少抽象層、減少 round trip、減少 commit、避免大 offset，並用 profiler 驗證每一步。
