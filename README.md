# PHẦN B: AUTOMATION WORKFLOW VỚI N8N

> **Hệ thống RAG + Automation cho WeupBook**  Triển khai với n8n  
> Kết hợp với kiến trúc microservices đã thiết kế ở Phần A

---

## Mục Lục

1. [Tổng quan N8N trong hệ thống](#1-tổng-quan-n8n-trong-hệ-thống)
2. [Workflow 1  Document Indexing Pipeline](#2-workflow-1--document-indexing-pipeline)
3. [Workflow 2  RAG Query & Response](#3-workflow-2--rag-query--response)
4. [Workflow 3  User Matching & Notification](#4-workflow-3--user-matching--notification)
5. [Workflow 4  Scheduled Re-indexing](#5-workflow-4--scheduled-re-indexing)
6. [Workflow 5  Error Handling & Alerting](#6-workflow-5--error-handling--alerting)
7. [N8N Node Configuration Chi tiết](#7-n8n-node-configuration-chi-tiết)
8. [Deployment N8N](#8-deployment-n8n)

---

## 1. Tổng quan N8N trong hệ thống

### N8N đóng vai trò **Automation Orchestrator**  thay thế code logic phức tạp bằng visual workflow

```mermaid
graph TB
    subgraph "External Triggers"
        S3[" AWS S3<br/>File Upload Event"]
        SCHED[" Cron Scheduler<br/>Scheduled Tasks"]
        WEBHOOK[" Webhook<br/>API Calls"]
        USER[" User Action<br/>Frontend Events"]
    end

    subgraph "N8N Automation Layer"
        direction TB
        WF1[" Workflow 1<br/>Document Indexing Pipeline"]
        WF2[" Workflow 2<br/>RAG Query & Response"]
        WF3[" Workflow 3<br/>User Matching & Notification"]
        WF4[" Workflow 4<br/>Scheduled Re-indexing"]
        WF5[" Workflow 5<br/>Error Handling & Alerting"]
    end

    subgraph "Microservices"
        DOC["Document Service<br/>:8001"]
        EMBED["Embedding Service<br/>:8002"]
        ASSIST["Assistant Service<br/>:8003"]
    end

    subgraph "Data Layer"
        S3S["S3 Storage"]
        QDRANT["Qdrant :6333"]
        MONGO["MongoDB :27017"]
        POSTGRES["PostgreSQL :5432"]
        REDIS["Redis Cache"]
    end

    subgraph "Notification Channels"
        EMAIL[" Email<br/>SendGrid"]
        ZALO[" Zalo OA"]
        SLACK[" Slack"]
        INAPP[" In-App"]
    end

    S3 --> WF1
    SCHED --> WF4
    WEBHOOK --> WF2
    USER --> WF2

    WF1 --> DOC
    WF1 --> EMBED
    WF1 --> WF3

    WF2 --> ASSIST
    WF2 --> EMBED

    WF3 --> EMAIL
    WF3 --> ZALO
    WF3 --> SLACK
    WF3 --> INAPP

    WF4 --> WF1
    WF5 --> SLACK

    DOC --> S3S
    DOC --> POSTGRES
    EMBED --> QDRANT
    ASSIST --> MONGO
    ASSIST --> POSTGRES

    style WF1 fill:#FFC6C6
    style WF2 fill:#E6F3FF
    style WF3 fill:#E6FFE6
    style WF4 fill:#FFF3E6
    style WF5 fill:#FFE6F3
```

### Lý do chọn N8N

| Tiêu chí | N8N | Custom Code | Apache Airflow |
|----------|-----|-------------|----------------|
| **Setup time** |  Nhanh (visual) |  Chậm |  Chậm |
| **Self-hosted** |  Có |  Có |  Có |
| **HTTP integrations** |  Built-in | Manual | Manual |
| **Error handling** |  Visual retry | Manual code | DAG-based |
| **Cost** | Free (self-host) | Dev time | Dev time |
| **Webhook support** |  Native | Manual | Plugin |
| **AI nodes** |  LangChain built-in | Manual | Manual |

---

## 2. Workflow 1  Document Indexing Pipeline

> **Trigger**: S3 upload event  Index tài liệu vào Qdrant

### Sơ đồ tổng quan

```mermaid
flowchart TD
    T1([" START\nS3 Webhook Trigger"])

    N1[" HTTP Request Node\nReceive S3 Event\nPOST /webhook/s3-upload"]

    N2[" IF Node\nValidate Event\ncheck event_type & file_type"]

    N3[" AWS S3 Node\nDownload File\nfrom bucket/key"]

    N4[" Code Node\nDetect File Type\nPDF / DOCX / TXT / VTT"]

    N5A[" HTTP Request\nCall Document Service\nPOST :8001/api/parse/pdf"]
    N5B[" HTTP Request\nCall Document Service\nPOST :8001/api/parse/docx"]
    N5C[" HTTP Request\nCall Document Service\nPOST :8001/api/parse/txt"]
    N5D[" HTTP Request\nCall Document Service\nPOST :8001/api/parse/vtt"]

    N6[" Code Node\nText Cleaning\nNormalize + Remove noise"]

    N7[" Code Node\nChunking\nsize=500, overlap=100"]

    N8[" PostgreSQL Node\nSave Chunks\nINSERT document_chunks"]

    N9[" Split In Batches Node\nBatch size = 32"]

    N10[" HTTP Request Node\nCall Embedding Service\nPOST :8002/api/embed/batch"]

    N11[" IF Node\nEmbedding Success?\ncheck status == 200"]

    N12[" HTTP Request Node\nUpsert to Qdrant\nPOST :6333/collections/upsert"]

    N13[" PostgreSQL Node\nUpdate Status = indexed\nUPDATE documents"]

    N14[" Execute Workflow Node\nTrigger WF3\nUser Matching & Notification"]

    ERR[" Error Handler\nLog + Retry\nSend Slack Alert"]

    T1 --> N1 --> N2
    N2 -->|" Valid"| N3
    N2 -->|" Invalid"| ERR

    N3 --> N4
    N4 -->|PDF| N5A
    N4 -->|DOCX| N5B
    N4 -->|TXT| N5C
    N4 -->|VTT| N5D

    N5A & N5B & N5C & N5D --> N6
    N6 --> N7 --> N8 --> N9
    N9 --> N10 --> N11

    N11 -->|" Success"| N12
    N11 -->|" Failed"| ERR

    N12 --> N13 --> N14

    style T1 fill:#00C853,color:#fff
    style ERR fill:#FF5252,color:#fff
    style N14 fill:#E6FFE6
```

### Chi tiết từng node

```mermaid
graph LR
    subgraph "Node 1: S3 Webhook Trigger"
        W1["Trigger Type: Webhook\nMethod: POST\nPath: /webhook/s3-upload\nAuthentication: Header Auth\nX-Webhook-Secret: {secret}"]
    end

    subgraph "Node 2: Validate Event"
        W2["IF Condition:\n$json.event_type == 's3:ObjectCreated:Put'\nAND\n$json.content_type IN ['application/pdf',\n'text/plain', 'application/vnd.openxml...']"]
    end

    subgraph "Node 3: S3 Download"
        W3["AWS S3 Node\nOperation: Download\nBucket: {{ $json.bucket }}\nKey: {{ $json.key }}\nReturn: Binary Data"]
    end
```

### N8N JSON Config (Node: Embedding Batch)

```json
{
  "name": "Call Embedding Service",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "method": "POST",
    "url": "http://embedding-service:8002/api/embed/batch",
    "authentication": "genericCredentialType",
    "sendBody": true,
    "bodyParameters": {
      "parameters": [
        {
          "name": "texts",
          "value": "={{ $json.chunks.map(c => c.text) }}"
        },
        {
          "name": "normalize",
          "value": true
        }
      ]
    },
    "options": {
      "timeout": 30000,
      "retry": {
        "enabled": true,
        "maxTries": 3,
        "waitBetweenTries": 2000
      }
    }
  }
}
```

---

## 3. Workflow 2  RAG Query & Response

> **Trigger**: User gửi câu hỏi từ Frontend  Trả về Answer + Recommendations

```mermaid
flowchart TD
    T2([" START\nWebhook Trigger\nPOST /webhook/ask"])

    N1[" HTTP Request Node\nReceive Question\n{user_id, question, session_id}"]

    N2[" MongoDB Node\nGet User Profile\ndb.users.findOne({user_id})"]

    N3A[" Redis Node\nCheck Semantic Cache\nGET cache:{question_hash}"]

    N3B[" IF Node\nCache Hit?\nsimilarity >= 0.95"]

    N4[" HTTP Request\nEmbed Question\nPOST :8002/api/embed/single"]

    N5[" HTTP Request\nVector Search Qdrant\nPOST :6333/collections/search\n+ metadata filters"]

    N6[" PostgreSQL Node\nGet Document Metadata\nSELECT FROM documents\nWHERE doc_id IN (...)"]

    N7[" Code Node\nBuild RAG Context\nTop-5 chunks assembly"]

    N8[" Code Node\nBuild Adaptive Prompt\nSwitch by user.level:\nbeginner / intermediate / advanced"]

    N9[" HTTP Request\nCall Claude API\nPOST anthropic/v1/messages"]

    N10[" Code Node\nScore & Rank Documents\n0.5*relevance + 0.3*level\n+ 0.2*interest"]

    N11[" Redis Node\nCache Answer\nSET cache:{hash} TTL=86400"]

    N12[" PostgreSQL Node\nLog Interaction\nINSERT user_interactions"]

    N13[" Respond to Webhook\nReturn JSON Response\n{answer, documents, confidence}"]

    CACHED[" Return Cached Answer\nSkip LLM call\nSave ~$0.03/query"]

    T2 --> N1 --> N2
    N2 --> N3A --> N3B

    N3B -->|" Cache Hit"| CACHED
    N3B -->|" Miss"| N4

    CACHED --> N12

    N4 --> N5 --> N6 --> N7 --> N8 --> N9 --> N10 --> N11 --> N12 --> N13

    style T2 fill:#00C853,color:#fff
    style CACHED fill:#FFD700
    style N9 fill:#FFE6E6
    style N13 fill:#E6FFE6
```

### Adaptive Prompt Builder  Code Node

```mermaid
graph TD
    A["Input: user.level + context_chunks + question"]

    B{"user.level?"}

    C[" beginner"]
    D[" intermediate"]
    E[" advanced"]

    F["Prompt Template BEGINNER:\n- Dùng ngôn ngữ đơn giản\n- Thêm ví dụ thực tế\n- Giải thích từng bước\n- Tránh thuật ngữ kỹ thuật\n- Dùng phép so sánh, ẩn dụ"]

    G["Prompt Template INTERMEDIATE:\n- Cân bằng kỹ thuật & dễ hiểu\n- Đề cập khái niệm chính\n- Use cases thực tế\n- Một số ký hiệu toán học"]

    H["Prompt Template ADVANCED:\n- Giải thích chuyên sâu\n- Toán học & công thức\n- Trích dẫn research papers\n- Edge cases & tradeoffs\n- Độ sâu lý thuyết"]

    I["Final Prompt:\nSystem + Context + Question\n Claude API"]

    A --> B
    B -->|beginner| C --> F --> I
    B -->|intermediate| D --> G --> I
    B -->|advanced| E --> H --> I
```

---

## 4. Workflow 3  User Matching & Notification

> **Trigger**: Sau khi document được index xong  Tìm users phù hợp  Gửi thông báo

```mermaid
flowchart TD
    T3([" START\nExecute Workflow Trigger\nfrom WF1 completion"])

    N1[" Input Node\nReceive Document Info\n{doc_id, topic, level, course_id, tags}"]

    N2[" MongoDB Node\nQuery Matching Users\ndb.users.find({...filters})"]

    N3[" IF Node\nUsers Found?\ncount > 0"]

    N4[" Split In Batches\nProcess 50 users/batch\nAvoid memory overload"]

    N5[" Code Node\nBuild Recommendation\nPersonalize message per user"]

    N6[" Switch Node\nRoute by Channel\ncheck user.notification_preferences"]

    N7A[" SendGrid Node\nSend Email\nHTML Template"]
    N7B[" HTTP Request\nSend Zalo OA\nPOST zalo-api/message"]
    N7C[" Slack Node\nSend Slack Message\nFormatted blocks"]
    N7D[" PostgreSQL Node\nSave In-App Notification\nINSERT notifications"]

    N8[" PostgreSQL Node\nLog Delivery Status\nINSERT user_recommendations"]

    N9[" Code Node\nAggregate Results\ncount sent/failed per channel"]

    N10[" Slack Node\nSend Summary Report\n#ops-notifications channel"]

    SKIP[" No Action\nNo matching users\nLog to POSTGRES"]

    T3 --> N1 --> N2 --> N3

    N3 -->|" Found"| N4
    N3 -->|" None"| SKIP

    N4 --> N5 --> N6

    N6 -->|email| N7A
    N6 -->|zalo| N7B
    N6 -->|slack| N7C
    N6 -->|always| N7D

    N7A & N7B & N7C & N7D --> N8 --> N9 --> N10

    style T3 fill:#00C853,color:#fff
    style N7A fill:#E8F5E9
    style N7B fill:#E3F2FD
    style N7C fill:#F3E5F5
    style N7D fill:#FFF8E1
```

### MongoDB Query Node  Tìm Matching Users

```mermaid
graph LR
    subgraph "MongoDB Query Logic"
        Q["Filter Conditions:\n\n1. level >= doc.level\n   (user đủ trình độ)\n\n2. doc.topic IN user.interests\n   (đúng sở thích)\n\n3. doc.course_id IN user.enrolled_courses\n   (đang học khóa này)\n\n4. notifications_enabled = true\n   (cho phép nhận thông báo)\n\n5. NOT recently recommended\n   (chưa gợi ý trong 7 ngày)\n\nLimit: 1000 users/batch"]
    end
```

### Email Template Node (SendGrid)

```mermaid
graph TD
    A["SendGrid Node Config"]
    B["To: {{ $json.user.email }}"]
    C["Subject:  Tài liệu mới phù hợp với bạn!"]
    D["Template ID: d-xxxxx (HTML template)"]
    E["Dynamic Data:\n- user_name: {{ $json.user.name }}\n- doc_title: {{ $json.doc.title }}\n- doc_level: {{ $json.doc.level }}\n- reading_time: {{ $json.doc.estimated_reading_time }}\n- doc_url: {{ $json.doc.url }}\n- topic_tag: {{ $json.doc.topic }}"]

    A --> B & C & D & E
```

---

## 5. Workflow 4  Scheduled Re-indexing

> **Trigger**: Cron job hàng đêm  Kiểm tra & re-index tài liệu lỗi/cũ

```mermaid
flowchart TD
    T4([" START\n Cron Trigger\nEvery day at 2:00 AM"])

    N1[" PostgreSQL Node\nQuery Failed Documents\nSELECT * FROM documents\nWHERE status IN ('failed', 'pending')\nAND created_at < NOW() - INTERVAL '1 hour'"]

    N2[" IF Node\nFailed docs found?\ncount > 0"]

    N3[" PostgreSQL Node\nQuery Outdated Documents\nSELECT * FROM documents\nWHERE version < current_model_version\nAND indexed_at < NOW() - 30 days"]

    N4[" Merge Node\nCombine failed + outdated\nDeduplicate by doc_id"]

    N5[" Split In Batches\n10 docs per batch"]

    N6[" AWS S3 Node\nFetch Original File\nfrom s3://bucket/key"]

    N7[" Execute Workflow\nCall WF1\nDocument Indexing Pipeline"]

    N8[" PostgreSQL Node\nUpdate re-index log\nINSERT reindex_history"]

    N9[" Code Node\nBuild Summary Report\n{total, success, failed}"]

    N10[" SendGrid Node\nSend Daily Report Email\nTo: ops-team@weupbook.com"]

    N11[" Slack Node\nPost to #ops-daily\nMarkdown summary"]

    SKIP2[" Skip\nAll documents healthy\nLog: no action needed"]

    T4 --> N1 --> N2

    N2 -->|" Found failures"| N3
    N2 -->|" All ok"| SKIP2

    N3 --> N4 --> N5 --> N6 --> N7 --> N8 --> N9 --> N10 & N11

    style T4 fill:#FF6F00,color:#fff
    style SKIP2 fill:#E8F5E9
```

---

## 6. Workflow 5  Error Handling & Alerting

> **Trigger**: Bất kỳ workflow nào thất bại  Alert team + Auto retry

```mermaid
flowchart TD
    T5([" ERROR TRIGGER\nN8N Error Workflow\nAny workflow failure"])

    N1[" Input Node\nCapture Error Details\n{workflow_id, node_name,\nerror_message, timestamp,\nretry_count, input_data}"]

    N2[" Code Node\nClassify Error Type\nTimeout / Network / Parse\n/ Auth / Rate Limit / Unknown"]

    N3{"Error Type?"}

    N4A[" Wait Node\nTimeout Error\nWait 30 seconds\nthen retry"]

    N4B[" HTTP Request\nNetwork Error\nRetry with backoff\n1s  2s  4s  8s"]

    N4C[" Code Node\nParse Error\nLog raw data\nSkip + continue"]

    N4D[" Code Node\nAuth Error\nRefresh token\nor alert immediately"]

    N4E[" Wait Node\nRate Limit\nWait until\nreset window"]

    N5[" IF Node\nRetry Attempts < 3?"]

    N6A[" Execute Workflow\nRetry Original Workflow\nwith same input data"]

    N6B[" Critical Alert\nMax retries exceeded\nEscalate to human"]

    N7[" Slack Node\n#ops-alerts channel\nFormatted error message\nwith severity level"]

    N8[" SendGrid Node\nEmail Alert\nops-team@weupbook.com\nOnly for CRITICAL errors"]

    N9[" PostgreSQL Node\nLog Error to DB\nINSERT error_logs\n{workflow, error, status}"]

    T5 --> N1 --> N2 --> N3

    N3 -->|Timeout| N4A
    N3 -->|Network| N4B
    N3 -->|Parse| N4C
    N3 -->|Auth| N4D
    N3 -->|Rate Limit| N4E

    N4A & N4B & N4C & N4D & N4E --> N5

    N5 -->|"< 3"| N6A
    N5 -->|">= 3"| N6B

    N6B --> N7 --> N8 --> N9

    style T5 fill:#FF5252,color:#fff
    style N6B fill:#FF5252,color:#fff
    style N7 fill:#FFE0B2
```

### Error Classification Matrix

```mermaid
graph TD
    subgraph "Error Severity Levels"
        P1[" CRITICAL\n- Qdrant down\n- PostgreSQL down\n- Claude API down\nAction: Immediate Slack + Email\nRetry: 3x then escalate"]

        P2[" HIGH\n- Embedding timeout\n- S3 access denied\n- MongoDB slow\nAction: Slack alert\nRetry: 3x exponential backoff"]

        P3[" MEDIUM\n- Single notification fail\n- Parse error 1 doc\n- Cache miss anomaly\nAction: Log only\nRetry: 2x then skip"]

        P4[" LOW\n- Scheduled task delay\n- Minor formatting issue\nAction: Log to DB\nRetry: None"]
    end
```

---

## 7. N8N Node Configuration Chi tiết

### Credential Setup trong N8N

```mermaid
graph LR
    subgraph "N8N Credentials Store"
        C1["AWS S3\n- Access Key ID\n- Secret Access Key\n- Region: ap-southeast-1"]

        C2["PostgreSQL\n- Host: postgres:5432\n- DB: weupbook\n- User/Pass: ***"]

        C3["MongoDB\n- Connection String\n- mongodb://mongo:27017\n- Database: weupbook"]

        C4["Anthropic Claude\n- API Key: sk-ant-***\n- Model: claude-sonnet-4"]

        C5["SendGrid\n- API Key: SG.***\n- From: noreply@weupbook.com"]

        C6["Slack\n- Bot Token: xoxb-***\n- Signing Secret: ***"]

        C7["Redis\n- Host: redis:6379\n- Password: ***"]

        C8["Qdrant\n- Host: http://qdrant:6333\n- API Key: ***"]
    end
```

### Environment Variables cho N8N

```bash
# .env file cho N8N deployment

# N8N Core
N8N_HOST=0.0.0.0
N8N_PORT=5678
N8N_PROTOCOL=https
N8N_ENCRYPTION_KEY=your-32-char-secret-key

# Database (N8N internal)
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n_user
DB_POSTGRESDB_PASSWORD=n8n_password

# Webhook
WEBHOOK_URL=https://n8n.weupbook.com
N8N_WEBHOOK_BASE_URL=https://n8n.weupbook.com/webhook

# Queue Mode (for scaling)
EXECUTIONS_MODE=queue
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379
```

---

## 8. Deployment N8N

### Docker Compose  N8N Service

```yaml
version: "3.8"

services:
  #  N8N Automation Engine 
  n8n:
    image: n8nio/n8n:latest
    container_name: weupbook-n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.weupbook.com
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n_db
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASS}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
    volumes:
      - n8n_data:/home/node/.n8n
      - ./n8n-workflows:/home/node/.n8n/workflows
    networks:
      - weupbook-network
    depends_on:
      - postgres
      - redis

  #  N8N Worker (for queue mode) 
  n8n-worker:
    image: n8nio/n8n:latest
    container_name: weupbook-n8n-worker
    restart: always
    command: worker
    environment:
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n_db
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    networks:
      - weupbook-network
    depends_on:
      - n8n
      - redis

volumes:
  n8n_data:

networks:
  weupbook-network:
    driver: bridge
```

### Kiến trúc Scaling N8N

```mermaid
graph TB
    subgraph "Production N8N Setup"
        LB[" Load Balancer\nnginx / AWS ALB"]

        subgraph "N8N Main (UI + Trigger)"
            N8N1["N8N Instance 1\n:5678\nWebhook handler\nWorkflow editor"]
        end

        subgraph "N8N Workers (Execution)"
            W1["Worker 1\nDocument workflows"]
            W2["Worker 2\nNotification workflows"]
            W3["Worker 3\nScheduled workflows"]
        end

        subgraph "Queue"
            BULL["Bull Queue\nRedis :6379\nJob management"]
        end
    end

    subgraph "Monitoring"
        PROM["Prometheus\nMetrics"]
        GRAF["Grafana\nDashboard"]
    end

    LB --> N8N1
    N8N1 --> BULL
    BULL --> W1 & W2 & W3

    W1 & W2 & W3 --> PROM --> GRAF

    style N8N1 fill:#FF6D00,color:#fff
    style W1 fill:#FFA040
    style W2 fill:#FFA040
    style W3 fill:#FFA040
```

---

## Tổng kết: Workflow Integration Map

```mermaid
flowchart LR
    subgraph "TRIGGERS"
        T1[" S3 Upload"]
        T2[" User Query"]
        T3[" Cron 2AM"]
        T4[" WF1 Completion"]
        T5[" Any Error"]
    end

    subgraph "N8N WORKFLOWS"
        WF1["WF1\nDocument Indexing\n~3-5 min/doc"]
        WF2["WF2\nRAG Query\n< 2 sec"]
        WF3["WF3\nNotification\n< 1 min"]
        WF4["WF4\nRe-indexing\nNightly batch"]
        WF5["WF5\nError Handler\nReal-time"]
    end

    subgraph "OUTCOMES"
        O1[" Document Indexed\nin Qdrant"]
        O2[" Answer + Docs\nto User"]
        O3[" Notifications\nSent to Users"]
        O4[" Failed Docs\nRecovered"]
        O5[" Team Alerted\nIssues Resolved"]
    end

    T1 --> WF1 --> O1
    WF1 --> WF3

    T2 --> WF2 --> O2

    T4 --> WF3 --> O3

    T3 --> WF4 --> O4

    T5 --> WF5 --> O5

    style WF1 fill:#FFC6C6
    style WF2 fill:#E6F3FF
    style WF3 fill:#E6FFE6
    style WF4 fill:#FFF3E6
    style WF5 fill:#FFE6F3
```

---

>  **Ghi chú**: Toàn bộ workflow N8N có thể export/import dưới dạng JSON và version-control trên Git. Mỗi workflow nên có documentation đi kèm và unit test với N8N's built-in execution history.
