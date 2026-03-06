# C. Tối Ưu Hệ Thống (Optimization)

Phần này trình bày các chiến lược tối ưu khi hệ thống mở rộng với **nhiều user và tài liệu**, tập trung vào 3 mục tiêu:

- Giảm chi phí token LLM
- Tăng tốc độ phản hồi
- Tránh gợi ý trùng lặp, quá chung chung nhưng vẫn cá nhân hóa

---

## 1. Tối ưu chi phí token

### 1.1 Prompt Caching

Phần System Prompt và Context tài liệu thường **lặp lại giữa các request**. Sử dụng **Prompt Caching** (Anthropic / OpenAI) để cache phần tĩnh, chỉ tính phí phần động.

```mermaid
flowchart LR

classDef action  fill:#2c2c3e,stroke:#444,color:#fff
classDef service fill:#1a4a6b,stroke:#0d3b57,color:#fff
classDef db      fill:#1565C0,stroke:#0d47a1,color:#fff
classDef done    fill:#1e8449,stroke:#196f3d,color:#fff
classDef cache   fill:#6d4c41,stroke:#4e342e,color:#fff

Q([User Query]):::action

Q --> SPLIT[Tach prompt\nStatic vs Dynamic]:::action

SPLIT --> STATIC[System Prompt\n+ Context Chunks\ncache_control: ephemeral]:::cache

SPLIT --> DYNAMIC[User Question\n+ User Profile\nkhong cache]:::action

STATIC --> LLM[LLM API\nChỉ tính phí\nphần dynamic]:::service

DYNAMIC --> LLM

LLM --> ANS([Response]):::done
```

Kết quả: giảm **60–80% chi phí token** cho các câu hỏi cùng tài liệu.

---

### 1.2 Vector Cache — Tránh Embedding Trùng Lặp

Trước khi gọi Embedding Service, kiểm tra cache. Nếu chunk đã được embed rồi thì dùng lại.

```mermaid
flowchart LR

classDef action  fill:#2c2c3e,stroke:#444,color:#fff
classDef service fill:#1a4a6b,stroke:#0d3b57,color:#fff
classDef db      fill:#1565C0,stroke:#0d47a1,color:#fff
classDef cache   fill:#6d4c41,stroke:#4e342e,color:#fff
classDef condition fill:#3d3d3d,stroke:#555,color:#fff

CHUNK[Chunk Text]:::action --> HASH[MD5 Hash\ncủa chunk]:::action

HASH --> CHECK{Cache\nhit?}:::condition

CHECK -- hit --> REDIS[(Redis\nReturn vector)]:::cache
CHECK -- miss --> EMBED[Embedding\nService]:::service

EMBED --> STORE[Lưu vào Redis\nTTL = 7 ngày]:::cache
EMBED --> QDRANT[(Qdrant)]:::db
REDIS --> QDRANT
```

Áp dụng: **Redis** làm vector cache với key = `md5(chunk_text)`.

---

### 1.3 Batching Embedding Request

Thay vì gọi API từng chunk một, gom nhiều chunk vào 1 request.

| Cách | Số API call cho 100 chunks |
|---|---|
| Gọi tuần tự | 100 calls |
| Batch size = 32 | 4 calls |
| Tiết kiệm | 96% số lượng request |

---

### 1.4 Chọn Model Phù Hợp Theo Task

Không phải task nào cũng cần model lớn.

| Task | Model đề xuất | Lý do |
|---|---|---|
| Phân loại câu hỏi (Router) | claude-haiku / gpt-4o-mini | Đơn giản, cần nhanh |
| Tóm tắt gợi ý tài liệu | claude-haiku / gpt-4o-mini | Output ngắn |
| Trả lời câu hỏi phức tạp | claude-sonnet / gpt-4o | Cần chất lượng cao |
| Reranking | Cross-encoder nhỏ (local) | Không cần gọi LLM API |

---

## 2. Tối ưu tốc độ phản hồi

### 2.1 Streaming Response

Thay vì chờ LLM trả toàn bộ output, dùng **streaming** để frontend hiển thị từng token ngay khi có.

```mermaid
flowchart LR

classDef action  fill:#2c2c3e,stroke:#444,color:#fff
classDef service fill:#1a4a6b,stroke:#0d3b57,color:#fff
classDef done    fill:#1e8449,stroke:#196f3d,color:#fff

Q([User Query]):::action --> RAG[Retrieval\nHybrid Search]:::action
RAG --> LLM[LLM API\nstream=true]:::service
LLM -- token stream --> FE[Frontend\nhiển thị ngay\ntừng token]:::done
```

Cảm nhận tốc độ tăng đáng kể dù latency thực tế không đổi.

---

### 2.2 Parallel Retrieval

Thực hiện Semantic Search và Keyword Search **song song** thay vì tuần tự.

```mermaid
flowchart LR

classDef action    fill:#2c2c3e,stroke:#444,color:#fff
classDef service   fill:#1a4a6b,stroke:#0d3b57,color:#fff
classDef condition fill:#3d3d3d,stroke:#555,color:#fff
classDef done      fill:#1e8449,stroke:#196f3d,color:#fff

Q([User Query]):::action

Q --> SEM[Semantic Search\nQdrant dense]:::service
Q --> KW[Keyword Search\nBM25 sparse]:::service

SEM --> MERGE[Merge + Normalize\nRRF Score]:::action
KW --> MERGE

MERGE --> RERANK[Cross-Encoder\nRerank Top 5]:::action
RERANK --> CONTEXT([Final Context]):::done
```

Giảm latency retrieval từ ~400ms xuống ~200ms (chạy song song với `asyncio.gather`).

---

### 2.3 Query Rewriting

Câu hỏi ngắn hoặc mơ hồ của user thường cho retrieval kém. Rewrite trước khi search.

```mermaid
flowchart LR

classDef action    fill:#2c2c3e,stroke:#444,color:#fff
classDef service   fill:#1a4a6b,stroke:#0d3b57,color:#fff
classDef condition fill:#3d3d3d,stroke:#555,color:#fff
classDef cache     fill:#6d4c41,stroke:#4e342e,color:#fff
classDef done      fill:#1e8449,stroke:#196f3d,color:#fff

Q([User: viet code\nkhong biet bat dau]):::action

Q --> RW[Query Rewriting\nLLM mini model]:::service

RW --> Q2[Huong dan lap trinh\ncho nguoi moi bat dau\nPython co ban]:::action

Q2 --> CACHE{Cache\nhit?}:::condition

CACHE -- hit --> RET[(Redis\nCached result)]:::cache
CACHE -- miss --> SEARCH[Hybrid Search\nQdrant]:::service

SEARCH --> RERANK[Rerank\n+ Context]:::action
RET --> RERANK
RERANK --> LLM([LLM Generate]):::done
```

---

### 2.4 Kết quả tổng hợp tối ưu tốc độ

| Kỹ thuật | Tác động |
|---|---|
| Streaming | Thời gian đến token đầu tiên giảm ~70% |
| Parallel retrieval | Latency retrieval giảm ~50% |
| Query rewriting | Chất lượng context tăng, giảm hallucination |
| Vector cache (Redis) | Cache hit loại bỏ hoàn toàn embedding latency |

---

## 3. Tránh gợi ý trùng lặp và cá nhân hóa

### 3.1 Notification Deduplication

Trước khi gửi gợi ý, kiểm tra user đã nhận tài liệu này chưa.

```mermaid
flowchart LR

classDef action    fill:#2c2c3e,stroke:#444,color:#fff
classDef db        fill:#1565C0,stroke:#0d47a1,color:#fff
classDef condition fill:#3d3d3d,stroke:#555,color:#fff
classDef error     fill:#922b21,stroke:#7b241c,color:#fff
classDef done      fill:#1e8449,stroke:#196f3d,color:#fff

MATCH[User matched\nvoi doc moi]:::action

MATCH --> CHECK{Da gui\ngoi y nay\nchua?}:::condition

CHECK -- da gui --> SKIP[Skip\nKhong gui trung]:::error

CHECK -- chua gui --> WINDOW{Trong\ncooldown\n24h?}:::condition

WINDOW -- yes --> SKIP

WINDOW -- no --> SEND[Gui thong bao]:::done

SEND --> LOG[(Log DB\nuser_id + doc_id\nsent_at)]:::db
```

Logic kiểm tra trong PostgreSQL:

```sql
SELECT 1 FROM notification_log
WHERE user_id = $1
  AND doc_id  = $2
  AND sent_at > NOW() - INTERVAL '7 days'
LIMIT 1;
```

---

### 3.2 Diversity Injection — Tránh Gợi Ý Quá Giống Nhau

Khi gợi ý nhiều tài liệu, đảm bảo **đa dạng chủ đề** thay vì chỉ gợi ý cùng 1 topic.

```mermaid
flowchart LR

classDef action    fill:#2c2c3e,stroke:#444,color:#fff
classDef condition fill:#3d3d3d,stroke:#555,color:#fff
classDef done      fill:#1e8449,stroke:#196f3d,color:#fff

TOPK[Top 20 chunks\ntừ Qdrant]:::action

TOPK --> GROUP[Group by topic]:::action

GROUP --> PICK[Chọn top 1-2\ntừ mỗi topic\nMaximal Marginal Relevance]:::condition

PICK --> RERANK[Rerank final\ntheo relevance + diversity]:::action

RERANK --> RESULT([Top 5 diverse chunks]):::done
```

Công thức **MMR (Maximal Marginal Relevance)**:

```
MMR_score = λ · relevance(chunk, query)
          - (1-λ) · max_similarity(chunk, selected_chunks)

λ = 0.7  →  ưu tiên relevance
λ = 0.5  →  cân bằng relevance và diversity
```

---

### 3.3 Cá Nhân Hóa Theo Lịch Sử Học

Dùng `learning_history` của user để **lọc nội dung đã học** và **ưu tiên nội dung tiếp theo**.

```mermaid
flowchart LR

classDef action    fill:#2c2c3e,stroke:#444,color:#fff
classDef db        fill:#1565C0,stroke:#0d47a1,color:#fff
classDef condition fill:#3d3d3d,stroke:#555,color:#fff
classDef done      fill:#1e8449,stroke:#196f3d,color:#fff

PROFILE[(User Profile\nlevel · interests\nlearning_history)]:::db

PROFILE --> FILTER[Loai tru\ntai lieu da hoc]:::action

FILTER --> SCORE[Tinh diem uu tien\ntheo prerequisite graph]:::action

SCORE --> RANK[Xep hang tai lieu\ntheo lo trinh hoc]:::action

RANK --> PROMPT[Inject vao prompt\nLevel-aware context]:::action

PROMPT --> LLM([LLM Generate\nGoi y ca nhan hoa]):::done
```

Ví dụ scoring logic:

```
priority_score =
  0.4 · semantic_similarity(doc, user_interests)
+ 0.3 · prerequisite_match(doc, completed_courses)
+ 0.2 · level_match(doc.level, user.level)
+ 0.1 · recency_boost(doc.upload_date)
```

---

### 3.4 Level-Aware Prompt — Tránh Trả Lời Quá Chung Chung

Prompt được điều chỉnh chặt theo `user.level` để tránh output generic.

```
System Prompt (Beginner):
  - Giai thich bang vi du thuc te, tranh thuat ngu ky thuat
  - Moi khai niem can co 1 vi du code cu the
  - Gioi han do dai: ngan gon, de hieu
  - KHONG duoc: giai thich truu tuong, bo qua buoc co ban

System Prompt (Intermediate):
  - Co the dung thuat ngu ky thuat, nhung can giai thich ngan
  - Tap trung vao best practice va common pitfalls
  - So sanh cac cach tiep can khac nhau

System Prompt (Advanced):
  - Di thang vao van de, khong giai thich co ban
  - Tap trung vao edge cases, performance, trade-off
  - Trich dan nguon tham khao neu co
```

---

## 4. Tổng hợp chiến lược tối ưu

```mermaid
flowchart LR

classDef action    fill:#2c2c3e,stroke:#444,color:#fff
classDef service   fill:#1a4a6b,stroke:#0d3b57,color:#fff
classDef db        fill:#1565C0,stroke:#0d47a1,color:#fff
classDef cache     fill:#6d4c41,stroke:#4e342e,color:#fff
classDef done      fill:#1e8449,stroke:#196f3d,color:#fff

Q([User Query]):::action

Q --> RW[Query Rewriting\nmini model]:::service
Q --> DEDUP[Dedup Check\nRedis + Postgres]:::cache

RW --> PAR1[Semantic Search]:::service
RW --> PAR2[Keyword Search]:::service

PAR1 --> MERGE[Merge + MMR\nDiversity Injection]:::action
PAR2 --> MERGE

MERGE --> RERANK[Cross-Encoder\nRerank Top 5]:::action

RERANK --> CACHE_CHK{Prompt\nCache hit?}:::cache

CACHE_CHK -- hit --> STREAM[LLM Streaming\ncached context]:::service
CACHE_CHK -- miss --> BUILD[Build Prompt\nLevel-aware]:::action

BUILD --> STREAM

DEDUP --> FILTER[Filter\nda xem / da gui]:::action
FILTER --> STREAM

STREAM --> FE([Frontend\nStreaming response\n+ suggested docs JSON]):::done
```

---

## Kết luận Phần C

| Mục tiêu | Kỹ thuật | Kết quả kỳ vọng |
|---|---|---|
| Giảm chi phí token | Prompt Caching + Batching + Model routing | Giảm 60-80% chi phí LLM |
| Tăng tốc phản hồi | Streaming + Parallel retrieval + Vector cache | Latency giảm 50%, TTFT giảm 70% |
| Tránh trùng lặp | Deduplication log + Cooldown window | Loại bỏ 100% gợi ý đã gửi |
| Cá nhân hóa sâu | MMR + Priority scoring + Level-aware prompt | Gợi ý phù hợp lo trinh, không generic |
