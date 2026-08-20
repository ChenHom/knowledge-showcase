# RAG 大量文件的多層次儲存與檢索設計

#rag #retrieval #vector-database #system-design #ai-engineering #knowledge-base

## 這是什麼

這份文件說明當 RAG 系統面對大量文件時，如何設計多層次儲存、有效率寫入、有效率讀取，以及如何把檢索結果組成可靠回答。

核心目標不是「把所有文件切 chunk 丟進 vector DB」而已，而是建立一套能支援：

- 大量文件持續寫入
- 文件更新與增量索引
- 快速檢索
- 精準召回
- 可引用來源
- 權限過濾
- 回答品質穩定

的 RAG 架構。

## 核心結論

大量文件 RAG 不應只做單層 chunk embedding。比較穩定的做法是：

```text
Document layer
→ Section layer
→ Chunk layer
→ Fact / Entity layer（選配）
```

讀取時也不要一開始全庫搜尋 chunk，而應該：

```text
Query 分析
→ 先找相關文件 / 章節
→ 再找精準 chunks
→ rerank
→ context packing
→ LLM 回答
→ citation / verification
```

有效率的 RAG 系統，本質上是：

> 寫入時多存索引與摘要，讀取時先粗後細，回答時只餵最有價值的上下文。

## 一、系統目標

### 1. 有效率存入

文件進來後，系統應該能：

- 判斷文件是否已存在
- 判斷是否有更新
- 只重建變動部分
- 自動切章節與 chunk
- 建立 embedding
- 建立 metadata index
- 建立文件摘要 / 章節摘要
- 保留來源與權限資訊

### 2. 有效率讀取

使用者查詢時，系統應該能：

- 快速縮小搜尋範圍
- 避免全庫暴力搜尋
- 避免撈到語意相近但不相關的 chunk
- 支援權限過濾
- 支援時間、來源、文件類型過濾
- 支援 rerank
- 支援引用來源

### 3. 有效率回應

回答時，系統應該能：

- 控制 context 長度
- 去除重複資訊
- 保留必要上下文
- 引用來源
- 文件不足時明確說不知道
- 避免幻覺
- 支援後續追問

## 二、資料分層設計

### 1. Document Layer：文件層

文件層是整份文件的索引。

適合用來回答：

- 哪些文件可能相關？
- 這份文件屬於什麼主題？
- 文件來源、時間、版本、權限是什麼？
- 文件是否已更新？

建議欄位：

```json
{
  "doc_id": "doc_001",
  "source_type": "pdf | markdown | web | notion | google_drive",
  "source_uri": "https://...",
  "title": "退款政策文件",
  "summary": "這份文件說明退款條件、期限與例外情境。",
  "tags": ["policy", "refund", "customer-service"],
  "language": "zh-TW",
  "version": "2026-05-01",
  "content_hash": "sha256...",
  "permission_scope": ["support_team"],
  "created_at": "2026-05-07T10:00:00Z",
  "updated_at": "2026-05-07T10:00:00Z",
  "embedding": "embedding(summary + title)"
}
```

文件層用途：

- 快速找候選文件
- 做權限過濾
- 做版本管理
- 做來源追蹤
- 避免每次都從 chunk 層開始搜尋

### 2. Section Layer：章節層

章節層代表文件中的自然結構，例如：

```text
1. 退款政策
1.1 可退款條件
1.2 不可退款條件
2. 退貨流程
3. 特殊案例
```

建議欄位：

```json
{
  "section_id": "sec_001",
  "doc_id": "doc_001",
  "heading_path": ["退款政策", "可退款條件"],
  "section_title": "可退款條件",
  "section_summary": "說明哪些訂單符合退款資格。",
  "start_page": 3,
  "end_page": 5,
  "content_hash": "sha256...",
  "embedding": "embedding(section_summary + heading_path)"
}
```

章節層用途：

- 先找到相關章節，再進 chunk
- 保留上下文結構
- 避免 chunk 太碎導致語意漂移
- 支援「這份文件哪一章有答案」

### 3. Chunk Layer：片段層

chunk 是實際餵給 LLM 的主要內容。

建議欄位：

```json
{
  "chunk_id": "chunk_001",
  "doc_id": "doc_001",
  "section_id": "sec_001",
  "heading_path": ["退款政策", "可退款條件"],
  "text": "使用者在付款後 7 天內，且商品未出貨時，可以申請退款...",
  "token_count": 420,
  "chunk_index": 3,
  "page_start": 4,
  "page_end": 4,
  "prev_chunk_id": "chunk_000",
  "next_chunk_id": "chunk_002",
  "content_hash": "sha256...",
  "embedding": "embedding(text)"
}
```

Chunk 設計建議：

```text
chunk size：300–800 tokens
overlap：10–20%
```

實際數字要依文件類型調整：

| 文件類型 | chunk 建議 |
|---|---|
| FAQ | 小 chunk，問題答案成對保留 |
| 法規 / 政策 | 中等 chunk，保留條文完整性 |
| 技術文件 | 按 heading / code block 切 |
| 長篇報告 | 章節摘要 + chunk 混合 |
| 表格 | 優先轉成結構化 records |

### 4. Fact / Entity Layer：事實層，選配

如果文件中有大量規則、產品規格、政策條件，可以抽成結構化 facts。

例如：

```json
{
  "fact_id": "fact_001",
  "doc_id": "doc_001",
  "section_id": "sec_001",
  "subject": "退款期限",
  "predicate": "是",
  "object": "付款後 7 天內",
  "condition": "商品尚未出貨",
  "source_chunk_id": "chunk_001",
  "confidence": 0.92
}
```

Fact Layer 適合：

- 政策查詢
- 法規條文
- 產品規格
- 價格表
- 權限規則
- 可自動驗證的條件

不適合：

- 文學內容
- 主觀評論
- 高度上下文依賴的內容

## 三、有效率的寫入設計

### 寫入流程總覽

```text
Input document
→ Normalize
→ Hash / deduplicate
→ Parse structure
→ Generate document summary
→ Split sections
→ Generate section summaries
→ Split chunks
→ Generate embeddings
→ Store metadata + vectors
→ Build indexes
```

### 1. Normalize：格式標準化

所有來源先轉成統一格式。

```text
PDF / HTML / Markdown / Word / Notion
→ canonical markdown / plain text
```

標準化時要處理：

- 移除頁首頁尾
- 移除重複導航列
- 保留 heading
- 保留表格
- 保留來源頁碼
- 保留連結
- 正規化空白與標點
- 語言偵測

### 2. Hash：避免重複寫入

每份文件都算 hash：

```text
doc_hash = sha256(normalized_content)
```

每個 section / chunk 也算 hash：

```text
section_hash = sha256(section_text)
chunk_hash = sha256(chunk_text)
```

寫入時先比較：

```text
如果 doc_hash 沒變 → 不重建
如果 section_hash 沒變 → 不重建該章節
如果 chunk_hash 沒變 → 不重算 embedding
```

這是大量文件系統省成本的關鍵。

### 3. Parse Structure：保留文件結構

不要只按 token 數硬切。

優先用文件自然結構：

```text
H1 / H2 / H3
PDF outline
章節標題
條文編號
FAQ Q/A
表格 row
```

如果文件沒有結構，再 fallback 成 token-based chunking。

### 4. Summary：寫入時預先摘要

寫入時可以先產生：

```text
document_summary
section_summary
```

這些摘要不是最終回答，而是 retrieval index。

好處：

- query 可以先對 summary embedding 搜尋
- 比直接搜長 chunk 更穩
- 可以快速縮小候選範圍

### 5. Embedding：分層建立向量

不要只 embedding chunk。

建議至少建立三種 embedding：

```text
document embedding：title + document_summary
section embedding：heading_path + section_summary
chunk embedding：chunk text
```

選配：

```text
fact embedding：subject + predicate + object + condition
```

### 6. Batch 寫入

大量文件寫入時不要一筆一筆寫。

建議：

```text
parse jobs
→ embedding batch jobs
→ vector upsert batch
→ metadata DB transaction
```

寫入批次設計：

```text
embedding batch size：依模型 API / 本機模型吞吐量調整
vector upsert batch：100–1000 records
metadata insert：transaction
```

### 7. 寫入資料庫分工

建議分開：

```text
Metadata DB：Postgres / MySQL / TiDB
Vector DB：pgvector / Milvus / Qdrant / Weaviate / TiDB Vector
Object Storage：原始文件 / normalized markdown
```

儲存分工：

| 類型 | 存哪裡 |
|---|---|
| 原始文件 | object storage / filesystem |
| normalized text | object storage / DB |
| metadata | relational DB |
| vector | vector DB |
| chunk text | DB 或 object storage |
| access control | metadata DB |
| processing state | job table |

## 四、有效率的讀取設計

### 讀取流程總覽

```text
User query
→ Query understanding
→ Permission filter
→ Document / section retrieval
→ Chunk retrieval
→ Rerank
→ Context packing
→ Answer generation
→ Citation
```

### 1. Query Understanding

先判斷問題類型：

```text
fact lookup：查明確事實
policy reasoning：查政策並判斷
summary：總結某文件
comparison：比較多文件
troubleshooting：找解法
exploratory：探索式問題
```

不同問題走不同 retrieval 策略。

| 問題類型 | Retrieval 策略 |
|---|---|
| 明確事實 | fact / chunk 優先 |
| 政策判斷 | section + chunk + fact |
| 文件摘要 | document + section |
| 比較問題 | 多 document retrieval |
| debug / 排錯 | chunk + rerank |

### 2. Permission Filter

權限一定要在 retrieval 前做，不要等 LLM 看到後再要求它不要說。

```text
WHERE permission_scope includes current_user
```

這是安全底線。

### 3. 粗到細檢索

不要直接全庫 chunk search。

建議：

```text
Step 1：document / section search 找候選範圍
Step 2：只在候選範圍內做 chunk search
Step 3：rerank chunks
```

範例：

```text
query → 找 top 10 sections
→ 在這 10 個 sections 的 chunks 裡找 top 30 chunks
→ rerank 成 top 6 chunks
→ 組 context
```

### 4. Hybrid Search

純 vector search 容易漏掉精確詞，例如：

- 訂單編號
- API 名稱
- 錯誤碼
- 法條編號
- 函式名稱

所以建議 hybrid：

```text
vector search + keyword / BM25 search
```

再合併結果。

### 5. Rerank

初步 retrieval 找到的是候選，不一定是最好。

Rerank 可以用：

- cross-encoder reranker
- LLM rerank
- heuristic score
- metadata boost

排序因素：

```text
semantic similarity
keyword match
document freshness
source authority
section relevance
user permission
query intent match
```

### 6. Context Packing

不要把 top chunks 直接全部塞進 prompt。

要做：

- 去重
- 合併相鄰 chunk
- 保留 heading path
- 保留來源
- 控制 token budget
- 優先高分 chunk
- 保留相反或例外條款

Context 格式建議：

```text
[Source 1]
doc: 退款政策 v2026
section: 退款政策 > 可退款條件
page: 4
text: ...

[Source 2]
doc: 退款政策 v2026
section: 退款政策 > 不可退款條件
page: 5
text: ...
```

## 五、有效率的回應設計

### 回答 prompt 原則

System prompt 要明確限制：

```text
你只能根據提供的 sources 回答。
如果 sources 沒有答案，請說不知道。
回答時要引用來源。
不要補充 sources 以外的資訊。
若來源互相衝突，請指出衝突。
```

### 回答格式

建議根據問題類型輸出。

#### 事實查詢

```text
答案：
...

來源：
- 文件 A，第 3 頁
```

#### 政策判斷

```text
判斷：
可以 / 不可以 / 需要更多資訊

理由：
1. ...
2. ...

來源：
- ...
```

#### 比較問題

```text
比較結果：
- A：...
- B：...

差異：
...

來源：
...
```

### 回答後驗證

重要場景可以多一步 verification：

```text
Answer
→ Check whether every claim is supported by sources
→ Remove unsupported claims
→ Add missing citation
```

尤其適合：

- 法務
- 醫療
- 金融
- 公司政策
- 對外客服

## 六、整體架構草圖

```text
                 ┌─────────────────────┐
                 │   Document Sources   │
                 │ PDF / Web / MD / DB  │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Normalize / Parse    │
                 │ clean + structure    │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Hash / Deduplicate   │
                 │ doc/section/chunk    │
                 └──────────┬──────────┘
                            │
                            ▼
       ┌────────────────────────────────────────┐
       │ Multi-level Index Builder              │
       │ doc summary / section summary / chunks │
       └──────────┬───────────────┬─────────────┘
                  │               │
                  ▼               ▼
       ┌────────────────┐  ┌──────────────────┐
       │ Metadata DB    │  │ Vector DB         │
       │ docs/sections  │  │ embeddings        │
       │ chunks/facts   │  │ doc/sec/chunk vec │
       └────────────────┘  └──────────────────┘


Query Flow

User Query
   │
   ▼
Query Understanding
   │
   ▼
Permission Filter
   │
   ▼
Document / Section Retrieval
   │
   ▼
Chunk Retrieval + Hybrid Search
   │
   ▼
Rerank
   │
   ▼
Context Packing
   │
   ▼
LLM Answer with Citation
   │
   ▼
Verification / Final Response
```

## 七、實作優先順序

### MVP

先做：

```text
documents
sections
chunks
metadata
chunk embeddings
section summary embeddings
basic hybrid search
citation
```

不要一開始就做太複雜。

### 第二階段

再加：

```text
document summary embedding
reranker
incremental indexing
permission filter
query classification
context packing optimizer
```

### 第三階段

最後才加：

```text
fact extraction
knowledge graph
multi-hop retrieval
answer verification
eval dataset
online feedback loop
```

## 八、常見錯誤

### 1. 只做 chunk embedding

結果：

- 找不到整體脈絡
- chunk 太碎
- 回答缺上下文

### 2. chunk 太大

結果：

- embedding 不準
- context 塞不下
- rerank 困難

### 3. chunk 太小

結果：

- 語意不完整
- 回答容易斷章取義

### 4. 沒有 metadata

結果：

- 不能按文件、時間、權限過濾
- 不能引用來源
- 不能 incremental update

### 5. 權限過濾放太晚

錯誤做法：

```text
先 retrieve，再叫 LLM 不要說
```

正確做法：

```text
先 permission filter，再 retrieve
```

### 6. 沒有 eval

RAG 看起來能答，不代表答得準。

至少要測：

```text
retrieval precision
retrieval recall
answer faithfulness
citation correctness
latency
```

## 九、可以落地的資料表草案

### documents

```sql
CREATE TABLE documents (
  doc_id TEXT PRIMARY KEY,
  source_type TEXT,
  source_uri TEXT,
  title TEXT,
  summary TEXT,
  language TEXT,
  version TEXT,
  content_hash TEXT,
  permission_scope JSON,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

### sections

```sql
CREATE TABLE sections (
  section_id TEXT PRIMARY KEY,
  doc_id TEXT REFERENCES documents(doc_id),
  heading_path JSON,
  section_title TEXT,
  section_summary TEXT,
  start_page INT,
  end_page INT,
  content_hash TEXT,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

### chunks

```sql
CREATE TABLE chunks (
  chunk_id TEXT PRIMARY KEY,
  doc_id TEXT REFERENCES documents(doc_id),
  section_id TEXT REFERENCES sections(section_id),
  heading_path JSON,
  text TEXT,
  token_count INT,
  chunk_index INT,
  page_start INT,
  page_end INT,
  prev_chunk_id TEXT,
  next_chunk_id TEXT,
  content_hash TEXT,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

### embeddings

```sql
CREATE TABLE embeddings (
  embedding_id TEXT PRIMARY KEY,
  owner_type TEXT, -- document | section | chunk | fact
  owner_id TEXT,
  embedding_model TEXT,
  embedding VECTOR,
  created_at TIMESTAMP
);
```

## 十、總結

大量文件 RAG 的關鍵不是「切 chunk + embedding」而已，而是：

```text
寫入時：
保留結構、建立摘要、分層 embedding、支援增量更新

讀取時：
先粗後細、權限前置、hybrid search、rerank、context packing

回應時：
來源約束、citation、必要時 verification
```

一句話：

> 好的 RAG 系統，是把文件整理成可檢索、可追溯、可更新、可控權限的多層知識索引，再用先粗後細的方式把最有用的上下文交給 LLM。
