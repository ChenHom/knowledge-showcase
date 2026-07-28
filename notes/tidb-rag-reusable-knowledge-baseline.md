# TiDB RAG：從自架技能包到可重用知識基線

## 這是什麼

這份知識的主題，不是單指 TiDB，也不是單指向量資料庫，而是整理一條較務實的自架 RAG 思路：

> **TiDB / TiKV / PD 很適合扮演 metadata 與資料層，但不應被誤當成完整 retrieval engine。**

如果目標是做一套可實際運作的 RAG stack，比較穩的方向通常是：
- TiDB：metadata / relational data
- Qdrant：vector retrieval
- MinIO：檔案物件儲存
- rag-api / rag-worker：upload / ingest / search / reindex / delete workflow

也就是說，TiDB-based RAG 的價值，不是「TiDB 一個全包」，而是它在整套資料與治理架構裡扮演很好的核心資料底座。

---

## 核心觀點

### 1. TiDB 不是完整 retrieval engine
這是最重要的一點。

TiDB / TiKV / PD 可以很好地承接：
- document metadata
- ingestion bookkeeping
- task / collection state
- 結構化查詢與管理面資料

但在 query-time retrieval 這件事上，若要做語意搜尋與向量近鄰查詢，仍應搭配專門的 vector engine，例如 Qdrant。

所以比較務實的理解應是：

- **TiDB 擅長資料管理與 relational/transactional 部分**
- **Qdrant 擅長 embedding retrieval**
- **MinIO 擅長原始檔案儲存**

把這三者拆開，整體系統會比硬把所有責任塞到單一元件更穩。

---

### 2. 最實用的 stack 不是「最純」，而是最能先打通閉環
一開始做自架 RAG 時，最容易卡死在：
- embedding provider
- ingestion pipeline
- parser/chunking
- collection / dimension mismatch
- upload/search/reindex/delete 全流程沒打通

比較穩的做法不是一開始就追求最完整模型，而是：

1. 先用 mock embedding 打通系統閉環
2. 確認 upload / ingest / search / reindex / delete 都能走通
3. 再切到 OpenAI-compatible embeddings 或其他正式 embedding provider

這樣可以把問題切開：
- 先驗證 pipeline
- 再驗證 embedding 品質

---

### 3. embedding model 切換時，最容易踩的是維度
這是 RAG 系統常見但容易被忽略的坑。

當你更換 embedding model 時，不能只改模型名稱，還要同步檢查：
- `EMBEDDING_DIMENSIONS`
- Qdrant collection 的 vector 維度

如果維度不一致，就必須：
- 重建 collection
- 或換新的 collection 名稱

否則 ingestion/search 很容易出現難查的錯誤。

---

### 4. parser / chunking 不該一開始就把結構壓扁
對 RAG 品質來說，parser/chunking 的策略很重要。

比較穩的原則是：

#### Markdown / HTML
- 保留原始結構
- 再做 structured chunking
- 不要先壓平成純文字再反推標題

#### PDF
- 至少做 page-aware parsing
- 保留 `page_no`
- 第一版可以先以每頁為 section
- 之後再談 OCR / 表格 / 多欄修正

這樣做的原因很簡單：

> 如果一開始就把文件結構打爛，後面的 retrieval 與 answer grounding 會一起變差。

---

## 如果要做成可重用技能包，應該包含什麼

先前這條知識曾被整理成一個 `tidb-rag-stack` skill。從可重用角度看，這種 skill 應包含：
- architecture
- deployment runbook
- config examples
- API contract
- usage examples
- troubleshooting
- templates
- evals

但後來更務實的判斷是：

> **先停在已具備半自動可執行 skill 的程度，不急著再補成更一致的 CLI 或更完整防呆。**

也就是說，目前最值得保留的是這條知識方向，而不是急著把所有最後一哩都產品化。

---

## 與一般聊天式 AI workflow 的關係

這類 TiDB RAG 知識還帶出一個更大的觀點：

### 模型能力不等於系統能力
光有模型，不等於你就有一套可穩定 ingest、search、reindex、delete 的 RAG 系統。

真正可用的系統還需要：
- state
- verification
- repair
- runbook
- troubleshooting path
- artifact governance

也因為這個觀點，後面才延伸出更完整的 `agent-system-foundation` 思路。

---

## 現在最值得記住的實務結論

如果你之後再做一套 TiDB-based RAG，最值得先記住的是這幾條：

1. **TiDB 不要誤當成完整 retrieval engine**
2. **務實架構是 TiDB + Qdrant + MinIO + API/worker**
3. **先用 mock embedding 打通閉環，再切正式 embedding provider**
4. **切 embedding model 時，一定檢查 dimension 與 collection 相容性**
5. **Markdown / HTML 要保留結構；PDF 至少做 page-aware parsing**
6. **先把可執行 skill 做到夠用，不要急著把最後一哩產品化**

---

## 一句話總結

TiDB-based 自架 RAG 的真正價值，不是讓 TiDB 一個人包辦所有 retrieval，而是把它放在資料與治理骨架的正確位置，搭配 vector engine、object storage 與穩定的 ingest/search workflow，形成一條務實、可維運、可逐步演進的 RAG 基線。

#AI #RAG #TiDB #Qdrant #MinIO #Embeddings #KnowledgeManagement #SelfHosted #VectorSearch #harness
