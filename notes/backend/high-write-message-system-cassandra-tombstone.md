# 高寫入量訊息系統，如何用 Cassandra 做可擴充儲存，以及它會怎麼用 tombstone 反咬你一口

## 文章主題

這篇是 Discord 在 **2017-01-13** 發的工程文，講的是他們當時如何把聊天訊息儲存架構，從 **MongoDB** 遷移到 **Cassandra**，以支撐已經超過 **每日 1.2 億則訊息** 的規模。這篇重點不是「聊天產品怎麼做」，而是「大量訊息寫入與隨機讀取，要怎麼設計資料庫與資料模型」。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

## 重點整理

### 1. 為什麼要從 MongoDB 換掉

Discord 早期把所有資料放在單一 MongoDB replica set，方便快速開發。但到 **2015 年 11 月左右，累積訊息達 1 億則** 後，資料與索引已經塞不進 RAM，延遲開始不可預測，系統撐不住。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

### 2. 他們選 Cassandra 的理由

Discord 的訊息讀寫特性是：

- 讀取非常隨機
    
- 讀寫比大約 **50/50**
    
- 需要線性擴充
    
- 需要自動故障切換
    
- 要低維護
    
- 要穩定且效能可預測
    
- 不想靠 Redis/Memcached 之類額外快取去硬撐
    

在這些條件下，他們判斷 **Cassandra** 最符合需求。因為它能靠加節點擴充、容忍節點故障，且相同分區內資料會連續存放，適合這類訊息查詢模式。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

### 3. 資料模型怎麼設計

原本 MongoDB 的查詢索引是 `(channel_id, created_at)`。  
搬到 Cassandra 後，Discord 改用 `(channel_id, message_id)`，其中 `message_id` 是 **Snowflake**，具時間排序特性，因此可以做範圍掃描。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

但這還不夠。因為一個 channel 可能存在很多年，分區會越長越肥。於是他們再把訊息按時間做 **bucket 分桶**，大約每桶放 **10 天** 的訊息，讓單一 partition 盡量控制在 **100MB 以下**。最後主鍵變成：

`((channel_id, bucket), message_id)`

這樣查近期訊息時，就掃最近幾個 bucket，避免單一大分區把 Cassandra 拖死。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

### 4. 上線後踩到的一個坑：最終一致性

Discord 採用 Cassandra，也等於接受它的 **eventual consistency**。  
Cassandra 的寫入本質偏向 **upsert**，而且以欄位為單位採 **last write wins**。這帶來一個競態問題：

- 一個人同時編輯訊息
    
- 另一個人同時刪掉訊息
    

結果可能留下只剩主鍵和部分欄位的「殘缺資料列」。Discord 的做法是：用必要欄位（例如 `author_id`）當檢查點，只要缺值就視為壞資料直接刪除。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

### 5. Tombstone 才是真正的地雷

Cassandra 刪除資料不是立刻物理刪除，而是寫入 **tombstone**。之後讀取會略過它，等到 compaction 和保留時間到期才真正清掉。更麻煩的是：

- 寫 `null`
    
- 刪欄位
    

兩者都會產生 tombstone。  
Discord 發現自己的 schema 有 **16 個欄位**，但平均每則訊息只有 **4 個欄位有值**，等於平常一直在白白寫大量 tombstone。於是他們改成：**只寫非 null 欄位**。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

### 6. 最慘事故怎麼發生

正式切主後大約 6 個月，Cassandra 有次突然卡死，GC 持續出現 **10 秒 stop-the-world**。追查後發現有一個 channel 幾乎被 API 刪光了，只剩 **1 則訊息**，但底下其實堆了 **數百萬個 tombstone**。使用者一讀這個 channel，Cassandra 就得掃過海量 tombstone，JVM 垃圾回收直接爆炸。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

他們最後做了兩件事：

- 把 tombstone 保留時間從 **10 天降到 2 天**
    
- 在查詢邏輯中記錄「空 bucket」，之後避免再掃這些空桶
    

這就是工程世界經典場景：資料看起來只剩 1 筆，實際上墓地塞滿。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

### 7. 成果

測試期間他們觀察到：

- 寫入延遲低於 **1ms**
    
- 讀取延遲低於 **5ms**
    

而且即使跳到一個擁有數百萬則訊息的 channel 中，一年前的訊息位置也能穩定讀取。當時他們最終運行的是 **12 節點、replication factor 3** 的 Cassandra 叢集。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))

---

## 這篇文章最值得記住的 5 件事

1. **先求可用，再求可擴充**：Discord 一開始用 MongoDB 不是錯，是刻意為了快速驗證產品。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))
    
2. **Cassandra 不是萬靈丹，資料模型決定生死**：分區鍵、排序鍵、bucket 設計錯了，後面全是災難。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))
    
3. **eventual consistency 不是口號，是會炸出競態條件的現實**。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))
    
4. **tombstone 是 Cassandra 常見陷阱**：大量刪除、寫 null、不當查詢，都可能把系統拖進 GC 地獄。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))
    
5. **分散式系統的痛點常不在平均情況，而在少數極端 case**。Discord 真正出事，不是因為正常流量，而是因為一個被大量刪訊息的特殊 channel。([Discord](https://discord.com/blog/how-discord-stores-billions-of-messages "How Discord Stores Billions of Messages"))
    

---

ref: [# Discord 如何處理一天數億的訊息](https://tachunwu.github.io/posts/discord-cassandra/?fbclid=IwAR0Kafo0LEdqflpMGRRWPk7ZX-9ESI9uOdjTNqZk1zE5DA69jQ5VKDs6Crc)


#backend #backend/database #backend/database/cassandra #backend/database/mongodb #backend/distributed-systems #backend/data-modeling #backend/scalability #backend/storage #backend/eventual-consistency
