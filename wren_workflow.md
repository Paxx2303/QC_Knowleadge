# Netflix Data Architecture — Tại Sao Chọn Công Cụ Này, Không Phải Công Cụ Kia

> Tài liệu này **không** giải thích công cụ là gì. Nó trả lời: **khi nào dùng**, **tại sao Netflix chọn nó thay vì thứ khác**, và **họ thiết kế nó thế nào để đảm bảo vận hành ổn định**.
>
> **Cách dùng:** Các số ①②③… trên sơ đồ khớp với số trong tiêu đề các mục bên dưới. Nhìn sơ đồ → thấy công cụ quan tâm → tìm số tương ứng → tra ngay mục chi tiết.

---

## Sơ đồ toàn hệ thống

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                         NETFLIX DATA ARCHITECTURE                                  ║
║                    260M+ subscribers · 190+ countries                              ║
╚══════════════════════════════════════════════════════════════════════════════════════╝

                      ┌──────────────────────────────────┐
                      │   USERS (iOS · Android · TV · Web)│
                      │   play · search · rate · login    │
                      └─────────────────┬────────────────┘
                                        │
                                        ▼
                      ┌─────────────────────────────────────────┐
                      │  ZUUL — API Gateway  ⓪                  │
                      │  Load balancing · Auth · Rate limiting   │
                      │  Request routing · TLS termination       │
                      └───────┬─────────────────┬───────────────┘
                              │                 │
              ┌───────────────┘                 └───────────────────┐
              │  OLTP requests                   Client events       │
              ▼                                                      ▼
┌─────────────────────────────────┐          ┌──────────────────────────────────────┐
│         OLTP LAYER              │          │  ⑱ KEYSTONE EVENT PIPELINE           │
│                                 │          │  Client SDK → HTTPS/protobuf batch   │
│  ① MySQL (RDS Multi-AZ)        │          │  Dedup layer 1: bloom filter         │
│    Billing · Auth · Profiles    │          │  ~1.3 trillion events / day          │
│    Content catalog              │          └──────────────────┬───────────────────┘
│    ACID · Strong consistency    │                             │
│    Sharded by user_id % N       │                             │
│                                 │                             │
│  ② Cassandra (Ring topology)   │                             │
│    Viewing history · Ratings    │                             │
│    Play position · Notif state  │                             │
│    RF=3 · LOCAL_QUORUM writes   │                             │
│    Eventual consistency         │                             │
│                                 │                             │
│  ③ EVCache (Memcached-based)   │                             │
│    Session cache · Metadata     │                             │
│    Zone-aware routing           │                             │
│    Absorbs 95% of reads         │                             │
└──────┬──────────────────────────┘                             │
       │                                                         │
       │  ⑤ Debezium CDC                                        │
       │  MySQL binlog → Kafka                                   │
       │  INSERT/UPDATE/DELETE captured                          │
       │  Sub-second latency                                     │
       │                                                         │
       └──────────────────────┐      ┌──────────────────────────┘
                              ▼      ▼
            ┌────────────────────────────────────────────┐
            │        ④ APACHE KAFKA                      │
            │   Event streaming backbone                  │
            │   ~500K msg/sec · RF=3 critical topics      │
            │   1024 partitions / major topic             │
            │   Partition key = user_id (ordering)        │
            │   Retention: 7 days → S3 archive forever    │
            │                                             │
            │   Topics:                                   │
            │   prod.viewing.play_event    [1024 parts]   │
            │   prod.billing.transaction   [RF=3]         │
            │   prod.cdc.mysql.*           [Debezium]     │
            └──────┬──────────────────────────────────────┘
                   │
       ┌───────────┼────────────────────┬───────────────────┐
       │           │                    │                   │
       ▼           ▼                    ▼                   ▼
┌──────────┐ ┌──────────────┐   ┌─────────────┐   ┌───────────────┐
│ ⑥ FLINK  │ │ ⑦ SPARK     │   │ ⑩ DRUID     │   │  S3 Raw sink  │
│ Stream   │ │ Batch ETL    │   │ Realtime     │   │  Raw archive  │
│ < 1 min  │ │ 2am–6am      │   │ analytics    │   │  forever      │
│ latency  │ │ TB-scale     │   │ < 1 sec      │   └───────────────┘
│          │ │              │   │ queries      │
│ Exactly  │ │ 400 executors│   │ 24/7 serve   │
│ -once    │ │ Spot 70%     │   │              │
│ RocksDB  │ │ EMR+Titus⑬  │   │ Pre-agg      │
│ state    │ │              │   │ + bitmap idx │
└────┬─────┘ └──────┬───────┘   └──────┬───────┘
     │              │                   │
     │              │                   │  serves realtime
     ▼              ▼                   │  dashboards
┌────────────────────────────────────┐  │
│   ⑧ AMAZON S3 + APACHE ICEBERG    │  │
│   ~1 Exabyte · Data Lake           │  │
│                                    │  │
│   /raw      → unchanged events     │  │
│   /cleaned  → validated            │  │
│   /enriched → joined with dims     │  │
│   /aggregated → pre-computed       │  │
│                                    │  │
│   Format: Parquet (columnar)       │  │
│   Iceberg: ACID · time travel      │  │
│            schema evolution        │  │
│                                    │  │
│   Nightly maintenance (UTC):       │  │
│   2am Compaction → 512MB files     │  │
│   3am Expire snapshots             │  │
│   4am Remove orphan files          │  │
│   5am Update statistics            │  │
└───────────┬────────────────────────┘  │
            │                           │
     ┌──────┴──────────┐                │
     ▼                 ▼                ▼
┌──────────────┐ ┌──────────────────────────────┐
│ ⑨ PRESTO/   │ │  ⑪ HIVE METASTORE + METACAT  │
│    TRINO     │ │  Table catalog · Schema defs  │
│              │ │  Data lineage · Ownership     │
│ Ad-hoc SQL   │ │  KHÔNG serve queries          │
│ 9am–6pm      │ │  Luôn chạy, serve metadata    │
│              │ └──────────────────────────────┘
│ Coordinator  │
│ + Workers    │
│ r5.8xlarge   │
│ Alluxio SSD  │
│ cache 10TB   │
└──────────────┘

       ┌─────────────────────────────────────────────────────┐
       │              SUPPORTING INFRASTRUCTURE              │
       │                                                     │
       │  ⑫ MAESTRO — Workflow orchestration                │
       │     Schedule & run thousands of Spark/Flink jobs    │
       │     Step-level retry · SLA monitoring               │
       │     Dynamic parallelism · Dependency tracking       │
       │                                                     │
       │  ⑬ TITUS — Container scheduler (on AWS)            │
       │     Runs all Spark / Flink / service containers     │
       │     Spot instance optimization                      │
       │                                                     │
       │  ⑭ ATLAS — Metrics platform                        │
       │     Billions of time series · Server-side agg       │
       │     SLO-based alerting · Grafana dashboards         │
       │                                                     │
       │  ⑮ EDGAR — Distributed tracing                     │
       │     Zipkin-based · End-to-end request traces        │
       │     Latency breakdown per microservice              │
       │                                                     │
       │  ⑯ ELASTICSEARCH                                   │
       │     Content full-text search (rebuilt from MySQL)   │
       │     Operational log search (30-day retain)          │
       │                                                     │
       │  ⑰ HOLLOW — In-memory data propagation             │
       │     Content catalog pushed to all service instances │
       │     Delta snapshots · Zero DB calls for reads       │
       └─────────────────────────────────────────────────────┘
```

### Bảng chú thích nhanh

| Số | Công cụ | Vai trò trong hệ thống | Chi tiết ở mục |
|:---:|---|---|:---:|
| ⓪ | Zuul (API Gateway) | Cổng vào duy nhất, routing mọi request | — |
| ① | MySQL | OLTP: billing, auth, profiles, catalog | §1.1 |
| ② | Cassandra | OLTP: viewing history, ratings, play state | §1.2 |
| ③ | EVCache | Cache layer trước MySQL/Cassandra | §7 |
| ④ | Apache Kafka | Xương sống streaming, trung tâm kết nối mọi luồng | §3 |
| ⑤ | Debezium | CDC: chuyển MySQL binlog → Kafka tự động | §4 |
| ⑥ | Apache Flink | Stream processing: latency < 1 phút, exactly-once | §5 Flink |
| ⑦ | Apache Spark | Batch ETL, ML training: chạy đêm, TB-scale | §5 Spark |
| ⑧ | S3 + Iceberg | OLAP storage: toàn bộ data lake ~1 Exabyte | §2.2 |
| ⑨ | Presto/Trino | Ad-hoc SQL analyst queries: giây đến phút | §2.1 |
| ⑩ | Apache Druid | Realtime dashboard: < 1 giây, 24/7 | §6 |
| ⑪ | Hive Metastore + Metacat | Catalog metadata, lineage — không query | §2.1 |
| ⑫ | Apache Maestro | Orchestrate hàng nghìn Spark jobs mỗi ngày | §8 |
| ⑬ | Titus | Container scheduler cho mọi data job trên AWS | §5 |
| ⑭ | Atlas | Metrics: billions of time series, SLO alerting | §9.1 |
| ⑮ | Edgar | Distributed tracing: latency breakdown per service | §9 |
| ⑯ | Elasticsearch | Content search + operational log analytics | §4 |
| ⑰ | Hollow | In-memory catalog propagation đến mọi service | §7 |
| ⑱ | Keystone | Client event ingestion pipeline | §3 |

---

## Mục lục

1. [OLTP Layer — Lưu trữ giao dịch thời gian thực](#1-oltp-layer)
2. [OLAP Layer — Phân tích & ra quyết định](#2-olap-layer)
3. [Streaming & Event Pipeline](#3-streaming--event-pipeline)
4. [CDC — Cầu nối OLTP → OLAP](#4-cdc--cầu-nối-oltp--olap)
5. [Batch Processing & Transformation](#5-batch-processing--transformation)
6. [Realtime Analytics](#6-realtime-analytics)
7. [Caching Layer](#7-caching-layer)
8. [Workflow Orchestration](#8-workflow-orchestration)
9. [Observability Stack](#9-observability-stack)
10. [Tổng hợp quyết định công cụ](#10-tổng-hợp-quyết-định-công-cụ)

---

## 1. OLTP Layer

### 1.1 ① MySQL vs PostgreSQL vs ② Cassandra

#### Khi nào ① MySQL tốt hơn

| Tình huống | MySQL tốt hơn PostgreSQL | MySQL tốt hơn Cassandra |
|---|---|---|
| Billing, thanh toán | Cần ACID full, FK constraint | Cassandra không có FK, không có multi-row transaction |
| User profile (đọc nhiều, ghi ít) | Read replica dễ scale out | Cassandra overkill cho volume nhỏ |
| Content catalog (ít thay đổi) | Join giữa nhiều bảng metadata | Cassandra không JOIN, phải denormalize toàn bộ |
| Dữ liệu cần consistency ngay | Strong consistency mặc định | Cassandra mặc định eventual |

**Thời điểm MySQL phát huy tốt nhất:** request cần đọc/ghi < 1KB, latency < 10ms, cần ACID, và volume đủ nhỏ để fit trong một shard hoặc vài shard.

**Tại sao Netflix không dùng PostgreSQL làm default thay MySQL:**
- 2009–2012 Netflix migrate lên AWS, RDS MySQL mature hơn RDS PostgreSQL ở thời điểm đó
- Toàn bộ tooling nội bộ (backup script, replication monitor, CDC connector) đã được build xung quanh MySQL binlog — đổi sang PostgreSQL WAL cần viết lại hàng trăm tools
- Không có ROI rõ ràng: MySQL đã đủ tốt cho các workload OLTP của Netflix; tính năng PostgreSQL vượt trội (window function, JSONB, full-text) đều có thể thay thế bằng Cassandra hoặc Elasticsearch ở các use case tương ứng
- Chi phí migration cho hàng chục terabyte dữ liệu production không thể justify

**Cách Netflix thiết kế MySQL để vận hành:**

```
Mỗi MySQL cluster:
  1 Primary (Multi-AZ, r5.8xlarge: 32 vCPU, 256GB RAM)
  2–5 Read Replicas (cùng AZ và cross-AZ)
  EVCache layer bên trên (absorb 95% reads)
  Binlog format = ROW  →  bắt buộc cho CDC
  sync_binlog = 1      →  không mất dữ liệu khi crash
  innodb_flush_log_at_trx_commit = 1  →  ACID compliance
```

Replication lag được alert khi > 5 giây. Nếu replica lag quá cao, traffic được route về primary — chấp nhận tăng load primary hơn là đọc stale data ở billing service.

---

#### Khi nào ② Cassandra tốt hơn ① MySQL

| Tình huống | Cassandra tốt hơn |
|---|---|
| Viewing history (ghi liên tục, global) | MySQL shard không đủ scale ở 260M users; Cassandra tự sharding theo ring |
| Play position (resume point) | Ghi nhiều, đọc theo user+content — partition key hoàn hảo |
| Ratings / thumbs | Write-heavy, không cần strong consistency |
| Notification state | Cần multi-region replication built-in |

**Thời điểm Cassandra phát huy tốt nhất:** ghi nhiều hơn đọc, data có dạng time-series hoặc "lookup by user + timestamp", chấp nhận eventual consistency, cần scale ngang tuyến tính.

**Tại sao Netflix không dùng MongoDB thay Cassandra:**
- Cassandra không có single point of failure (ring topology, mọi node đều equal) — MongoDB cần Primary node, fail-over mất vài giây
- Cassandra multi-datacenter replication được thiết kế từ đầu, không phải add-on
- Netflix đã có deep expertise Cassandra từ 2011; toàn bộ operational playbook, tuning guide đều được build sẵn
- MongoDB ở thời điểm Netflix chọn Cassandra (2011–2013) chưa mature về distributed operation

**Cách Netflix thiết kế Cassandra để vận hành:**

```
Ring topology:
  us-east-1:  3 AZ × 8 nodes = 24 nodes, RF=3
  eu-west-1:  3 AZ × 8 nodes = 24 nodes, RF=3
  ap-northeast: replicas

Consistency level:
  Writes: LOCAL_QUORUM  →  ghi thành công khi 2/3 nodes trong local DC confirm
  Reads:  ONE           →  đọc từ node gần nhất (latency ưu tiên)

Compaction:
  Time-series tables (viewing_history, play_events):
    → TimeWindowCompactionStrategy, window 7 days
    → Group SSTables theo thời gian → xóa TTL data hiệu quả
  Reference tables (user preferences):
    → LeveledCompactionStrategy → read performance tốt hơn

TTL mặc định 90 ngày cho viewing history:
  → Tự động xóa data cũ, không cần cron job
  → Giảm disk usage và compaction overhead
```

**Read Repair & Hinted Handoff — thiết kế cho eventual consistency:**

- **Read Repair**: Khi đọc, coordinator hỏi nhiều replica. Nếu replica trả về giá trị khác nhau, coordinator lấy cái có timestamp mới nhất trả về client, đồng thời gửi "repair" async đến replica stale. Không cần lock, không block write.
- **Hinted Handoff**: Khi một node down, node khác lưu "hint" (bản ghi chờ giao). Khi node down online lại, nhận hints và sync. Đảm bảo write không bị mất dù node tạm thời unavailable.

---

## 2. OLAP Layer

### 2.1 ⑨ Presto/Trino vs ⑦ Spark SQL vs ⑩ Druid vs ⑪ Hive

#### Khi nào dùng cái nào

| Công cụ | Dùng khi | Không dùng khi |
|---|---|---|
| **Presto/Trino** | Ad-hoc SQL, analyst query, kết quả cần trong vài giây đến vài phút | Job chạy hàng đêm xử lý TB data — Spark hiệu quả hơn |
| **Apache Spark** | Batch ETL, ML training, transform hàng chục TB, job có dependencies phức tạp | Query ad-hoc cần kết quả nhanh — overhead Spark job startup ~2 phút |
| **Apache Druid** | Dashboard realtime, query cần < 1 giây, data mới nhất (15–30 phút lag) | Scan toàn bộ lịch sử data — Druid không tối ưu cho full-table scan lớn |
| **Apache Hive** | Lưu table metadata (Metastore), định nghĩa schema — KHÔNG dùng để query | Query production — Hive query engine chậm hơn Presto/Spark đáng kể |

**Thời điểm từng công cụ phát huy:**

- **Presto/Trino**: 9am–6pm — giờ làm việc của analyst và data scientist, ad-hoc queries
- **Spark**: 2am–6am — batch window đêm, ít traffic OLTP, Spot Instance rẻ nhất
- **Druid**: 24/7 — dashboard operations, on-call engineer xem metrics realtime
- **Hive Metastore**: Luôn chạy, chỉ serve metadata — không serve query

**Tại sao Netflix không dùng Redshift hoặc BigQuery:**
- Netflix đã có Data Lake trên S3 ở quy mô exabyte — Redshift/BigQuery yêu cầu load data vào storage riêng của chúng, tạo data duplication và ingestion pipeline phức tạp
- Presto + S3 + Iceberg cho phép Netflix giữ toàn quyền kiểm soát storage và compute độc lập — scale compute mà không trả thêm cho storage
- Redshift không hỗ trợ tốt concurrent writes từ hàng trăm Spark jobs — Iceberg trên S3 handle được
- Cost: Spot Instance cho Presto workers rẻ hơn Redshift reserved capacity ở scale exabyte

**Cách Netflix thiết kế Presto cluster để vận hành:**

```
Coordinator node:
  - Parse SQL, tạo query plan, phân phối tasks
  - Không xử lý data — chỉ orchestrate

Worker nodes:
  - r5.8xlarge (256GB RAM) để cache S3 data in memory
  - Alluxio SSD cache (~10TB/cluster) trước S3
  - Cache hit rate 70–80% cho hot tables

Query routing:
  - Cluster riêng cho: ad-hoc queries, scheduled jobs, ML queries
  - Tránh analyst ad-hoc query làm chậm scheduled production jobs

Failure handling:
  - Query fail → retry từ checkpoint (Presto task-level retry)
  - Worker node fail → coordinator redistribute tasks sang nodes còn lại
```

---

### 2.2 ⑧ Apache Iceberg vs Hive Table Format vs Delta Lake

**Tại sao Netflix chọn Iceberg thay vì Hive table format truyền thống:**

| Vấn đề với Hive format | Iceberg giải quyết như thế nào |
|---|---|
| Hàng trăm Spark jobs ghi đồng thời → corrupt data | ACID transactions: serializable isolation trên S3 |
| Thêm column vào table → phải rewrite toàn bộ Parquet files | Schema evolution: thêm column không cần rewrite |
| Không thể xem data "như ngày hôm qua" | Time travel: query snapshot cũ bằng timestamp |
| Partition pruning chậm với hàng triệu files | Hidden partitioning + metadata indexing |
| Small files problem sau nhiều write nhỏ | Compaction job merge files, transparent với readers |

**Tại sao không chọn Delta Lake (Databricks):**
- Iceberg là open standard (Apache), không bị vendor lock-in vào Databricks
- Presto và Trino support Iceberg native — Delta Lake support trong Presto kém hơn ở thời điểm Netflix ra quyết định
- Netflix contributes code vào Iceberg project — ảnh hưởng được roadmap

**Cách Netflix vận hành Iceberg — maintenance jobs chạy hàng đêm:**

```
Job 1 — Compaction (2am UTC):
  Mục tiêu: Merge nhiều small Parquet files thành files 512MB
  Tại sao: S3 đọc 1 file 512MB nhanh hơn đọc 1024 files 512KB
            vì mỗi S3 GET request có overhead latency ~5–10ms
  Sort order: Z-order(content_id, user_id) để queries filter
              theo content_id hoặc user_id đều benefit từ data locality

Job 2 — Expire Snapshots (3am UTC):
  Xóa snapshots cũ hơn 7 ngày, giữ lại 10 snapshots gần nhất
  Tại sao: Mỗi write tạo snapshot mới — tích lũy hàng nghìn snapshots
            → metadata file phình to → query planning chậm

Job 3 — Remove Orphan Files (4am UTC):
  Xóa S3 files bị "mồ côi" (Spark job crash giữa chừng để lại files)
  Tại sao: Spark atomic write = write temp file rồi rename
            Nếu crash sau write nhưng trước rename → orphan file
            Không xóa → S3 storage tăng không kiểm soát

Job 4 — Update Statistics (5am UTC):
  ANALYZE TABLE → cập nhật column statistics cho query optimizer
  Tại sao: Presto/Trino dùng statistics để chọn join order,
            decide có nên push predicate xuống S3 scan không
```

---

## 3. Streaming & Event Pipeline

### 3.1 ④ Apache Kafka + ⑱ Keystone — Tại sao không dùng RabbitMQ, SQS, hay Pulsar

**Tại sao Kafka:**

| Tiêu chí | Kafka | RabbitMQ | AWS SQS |
|---|---|---|---|
| Throughput | 500K+ msg/sec per broker | ~50K msg/sec | ~3K msg/sec standard |
| Retention | 7 ngày, replay bất kỳ lúc nào | Message bị xóa sau khi consumer đọc | Message bị xóa |
| Fan-out | 1 topic → N consumer groups đọc độc lập | Exchange routing phức tạp | Cần SNS + nhiều SQS queues |
| Ordering | Đảm bảo trong cùng partition | Không đảm bảo với nhiều consumers | Không đảm bảo (standard queue) |
| Replay | Có — rewind offset về bất kỳ thời điểm nào | Không | Không |

**Khi nào Kafka phát huy tốt nhất tại Netflix:**
- Cần nhiều downstream systems đọc cùng 1 stream (Druid, Flink, S3 Sink, Elasticsearch) — mỗi cái là một consumer group độc lập, không ảnh hưởng nhau
- Cần replay: khi một consumer group (ví dụ Druid) bị down vài giờ, sau khi restore nó replay từ offset cũ — không mất event
- Cần ordering per user: partition key = user_id → mọi event của user đi vào cùng partition → Flink stateful processing đúng thứ tự

**Cách Netflix thiết kế Kafka để vận hành:**

```
Partition strategy:
  Topic prod.viewing.play_event: 1024 partitions
  Partition key = user_id → hash → partition number
  Lý do: 1024 = số consumer thread tối đa có thể chạy song song
          Ordering per-user được đảm bảo
          Flink stateful job có thể aggregate per-user đúng thứ tự

Replication:
  RF = 3 cho critical topics (billing, auth events)
  RF = 2 cho non-critical (impression events)
  min.insync.replicas = 2 → write chỉ thành công khi 2 replicas confirm

Producer config:
  acks = 1 (leader ack only) cho high-throughput, low-latency events
  acks = all cho billing events (durability ưu tiên hơn latency)
  compression = lz4 → balance CPU vs bandwidth
  batch.size = 32KB, linger.ms = 5 → buffer 5ms để gom batch

Retention:
  7 ngày trong Kafka → sau đó S3 archive (vĩnh viễn)
  Lý do 7 ngày: đủ để recovery từ bất kỳ incident nào trong tuần
                không giữ lâu hơn vì disk cost của Kafka brokers cao
```

---

## 4. CDC — Cầu Nối OLTP → OLAP

### 4.1 ⑤ Debezium vs Custom Polling vs Dual Write

**Tại sao không dùng Dual Write (application tự gửi event khi write DB):**

```
Vấn đề với Dual Write:
  1. Developer quên gửi event → silent data loss
  2. DB write thành công nhưng Kafka publish fail → inconsistency
  3. Mỗi service phải tự implement fanout logic → duplicated code
  4. Không capture được changes từ DB migration scripts hoặc admin tools

Tại sao không Polling (cron job SELECT WHERE updated_at > last_check):
  1. Cần updated_at column trên mọi table — không phải table nào cũng có
  2. DELETE không được capture (row đã biến mất)
  3. Polling interval = minimum lag — poll mỗi 1 phút = lag 1 phút
  4. Tăng load trên production DB theo scale của data
```

**Tại sao Debezium + MySQL Binlog:**
- Binlog là feature core của MySQL, không phải add-on — zero overhead trên write path (MySQL ghi binlog anyway cho replication)
- Capture INSERT, UPDATE, DELETE đầy đủ
- Sub-second latency — ngay khi transaction commit, binlog event xuất hiện
- Debezium đọc binlog như một MySQL replica — không query production DB

**Khi nào CDC lag là chấp nhận được vs không:**

| Pipeline | Max lag chấp nhận | Lý do |
|---|---|---|
| Billing transaction → fraud detection | < 30 giây | Fraud xảy ra ngay sau transaction |
| User profile update → Elasticsearch | < 5 phút | Search index stale 5 phút không ảnh hưởng UX đáng kể |
| Content catalog → Analytics | < 1 giờ | Analyst không cần data ngay lập tức |
| MySQL → S3 Data Lake | < 6 giờ | Batch analytics job chạy đêm |

**Cách Netflix vận hành Debezium:**

```
Debezium chạy trong Kafka Connect cluster (không phải trực tiếp trên DB server)
  → Đọc binlog qua MySQL replication protocol
  → Convert mỗi binlog event thành Kafka message (format Avro)
  → Publish vào topic: prod.cdc.mysql.{database}.{table}

Disaster recovery:
  Debezium lưu binlog position (file + offset) vào Kafka topic riêng
  Nếu Debezium crash → restart từ saved position → không replay duplicate
  Nếu MySQL primary fail → Debezium kết nối lại với replica (sau promotion)

Monitoring:
  Alert khi CDC lag > 5 phút (so sánh binlog timestamp vs Kafka message timestamp)
  Alert khi Debezium connector status = FAILED
  Alert khi schema thay đổi unexpected (auto-detect từ binlog)
```

---

## 5. Batch Processing & Transformation

### 5.1 ⑦ Apache Spark vs ⑥ Flink — Khi nào dùng cái nào (chạy trên ⑬ Titus)

| Tiêu chí | Spark | Flink |
|---|---|---|
| **Latency acceptable** | Phút đến giờ | Giây đến phút |
| **Data freshness cần** | T+1 (ngày hôm trước) | Near-realtime (< 1 phút) |
| **Processing model** | Micro-batch hoặc batch | True streaming |
| **State size** | Lớn tùy ý (spill to disk) | Phụ thuộc RocksDB state backend |
| **Startup overhead** | 1–3 phút (Spark job launch) | Luôn chạy, không startup overhead |

**Khi nào Spark tốt hơn Flink tại Netflix:**
- Daily content performance report (chạy lúc 4am, xử lý toàn bộ data ngày hôm qua)
- ML model training (cần toàn bộ 260M users' viewing history — TB range data)
- Historical backfill khi schema thay đổi
- Bất kỳ job nào cần join với large dimension table (content catalog, user demographics)

**Khi nào Flink tốt hơn Spark tại Netflix:**
- Realtime view count per content (cho operations dashboard)
- Fraud detection (cần phản ứng trong < 30 giây)
- Realtime deduplication của events trong cùng session
- Bất kỳ pipeline nào có SLA latency < 5 phút

**Cách Netflix vận hành Spark jobs:**

```
Infrastructure: Amazon EMR trên Titus (container scheduler)
  Driver:    8 vCPU, 32GB RAM (1 container)
  Executors: 400 × (4 vCPU, 16GB RAM) = 1600 vCPU tổng

Spot Instance strategy:
  70–80% executors = Spot Instances (70% cheaper)
  20–30% executors = On-Demand (đảm bảo job hoàn thành)
  Khi Spot bị reclaim: Spark restart chỉ task đó từ shuffle files
  → Job không cần restart từ đầu

Failure handling:
  spark.task.maxFailures = 4 (retry task 4 lần trước khi fail job)
  spark.stage.maxConsecutiveAttempts = 8
  Checkpoint to S3 mỗi 30 phút cho long-running jobs (> 2 giờ)

Output strategy:
  Không ghi trực tiếp vào Iceberg table trong production
  Ghi vào staging table → validate row count + schema → swap atomically
  Lý do: Nếu job fail giữa chừng, production table không bị corrupt
```

**Cách Netflix vận hành Flink jobs:**

```
State backend: RocksDB (spill to SSD khi state > RAM)
  Lý do: Dedup state cần lưu event_id của hàng tỷ events/ngày
         Không thể fit trong memory thuần

Checkpointing:
  Mỗi 30 giây → snapshot toàn bộ operator state → S3
  Nếu Flink job crash → restore từ checkpoint → tối đa mất 30 giây data
  Kết hợp với Kafka exactly-once → không duplicate, không mất

Exactly-once guarantee:
  Flink checkpoint + Kafka transaction:
    1. Flink commit checkpoint → Kafka transaction commit atomically
    2. Nếu fail trước commit → transaction rollback → retry từ checkpoint
    3. Kết quả: mỗi event được xử lý đúng 1 lần dù có failures
```

---

## 6. Realtime Analytics

### 6.1 ⑩ Apache Druid vs ClickHouse vs ⑯ Elasticsearch cho realtime analytics

**Tại sao Netflix chọn Druid:**

| Tiêu chí | Druid | ClickHouse | Elasticsearch |
|---|---|---|---|
| Kafka ingestion native | Có, built-in | Cần connector thêm | Cần Logstash/connector |
| Pre-aggregation | Có (roll-up khi ingest) | Không (lưu raw, query-time agg) | Không |
| Bitmap index cho dimensions | Có → filter cực nhanh | Không (dùng sparse index) | Có (inverted index) |
| Time-series partition | Native (segment per time interval) | Native | Cần custom mapping |
| Sub-second query SLA | Đạt được với pre-agg + bitmap | Phụ thuộc data volume | Phụ thuộc, không guarantee |

**Khi nào Druid phát huy tốt nhất:**
- Query pattern cố định (dashboard), không phải ad-hoc
- Data có dạng time-series với dimensions có cardinality thấp (device_type, region, content_genre)
- Cần kết quả < 1 giây với data fresh trong 15–30 phút

**Khi nào Druid không phù hợp:**
- Ad-hoc JOIN giữa nhiều tables — Druid không hỗ trợ tốt
- Scan toàn bộ lịch sử data (years) — nên dùng Presto + S3
- High-cardinality dimensions (user_id, content_id) trong GROUP BY — pre-agg không hiệu quả

**Cách Netflix thiết kế Druid để vận hành:**

```
Segment architecture:
  Data được chia thành Segments theo time interval (ví dụ: mỗi 1 giờ)
  Mỗi segment chứa: timestamp + dimensions (string) + metrics (numeric)
  Mỗi segment được lưu vào S3 (deep storage) VÀ loaded vào RAM của Historical nodes

Node roles:
  Realtime/MiddleManager nodes: Ingest từ Kafka, serve data < 24 giờ
  Historical nodes:             Serve data > 24 giờ từ S3 segments
  Broker nodes:                 Route query đến đúng nodes, merge kết quả
  Coordinator:                  Quản lý segment assignment, load balancing

Pre-aggregation (Roll-up) tại ingest:
  Thay vì lưu 1M raw play events/phút →
  Lưu 1 record per (minute, content_id, device_type, region):
    {count: 12400, unique_users_approx: 11200, sum_watch_ms: 744000000}
  → Giảm storage 100x–1000x
  → Query GROUP BY chỉ cần sum pre-aggregated metrics, không scan raw rows

Approximate counting:
  unique_users dùng HyperLogLog (±2% error) thay vì exact count
  → Giảm storage và query time đáng kể
  → Chấp nhận được với dashboards (không cần exact figure đến đơn vị)
```

---

## 7. Caching Layer & Data Propagation

### 7.1 ③ EVCache vs Redis vs Memcached thuần · ⑰ Hollow

**Tại sao Netflix dùng EVCache (Memcached-based) thay Redis:**

| Tiêu chí | EVCache | Redis |
|---|---|---|
| Multi-AZ replication | Built-in (zone-aware routing) | Cần Redis Cluster + custom config |
| Data structures | Key-Value only | Rich (List, Set, Sorted Set, Stream) |
| Throughput | Cực cao (Memcached multi-threaded) | Cao (Redis single-threaded per shard) |
| Memory efficiency | Cao (Memcached dùng ít overhead hơn) | Thấp hơn một chút |
| Persistence | Không (cache only) | Optional (RDB/AOF) |

**Khi nào EVCache tốt hơn Redis tại Netflix:**
- Session cache (stateless, không cần persistence — nếu mất session user login lại)
- Content metadata cache (read-heavy, simple key-value lookup)
- Rate limiting counters (nếu dùng Redis thì chọn Redis vì atomic INCR)

**Khi nào Netflix dùng Redis thay ③ EVCache:**
- Leaderboard (cần Sorted Set)
- Pub/Sub messaging giữa microservices
- Distributed lock (SETNX pattern)
- Services mới build sau 2018 — Redis đã mature, team lựa chọn tùy context

**Cách Netflix thiết kế ③ EVCache để vận hành:**

```
Zone-aware routing:
  Client trong us-east-1a → ưu tiên đọc từ EVCache node trong us-east-1a
  → Giảm cross-AZ latency (từ ~1ms xuống ~0.1ms)
  → Giảm cross-AZ data transfer cost

Replication:
  Write đồng thời đến ALL AZs trong region
  Read từ local AZ trước, fallback sang AZ khác nếu local node down
  → Tolerates single AZ failure mà không cần failover

Cache-aside pattern (application-managed):
  Read: Try EVCache → hit: return → miss: read DB → populate EVCache → return
  Write: Write DB → invalidate EVCache key (không update — avoid race condition)
  Lý do invalidate thay vì update:
    Nếu DB write thành công nhưng cache update fail → stale cache
    Invalidate + read-through đảm bảo next read luôn lấy fresh data từ DB

TTL strategy:
  Session data: 30 phút (auto expire nếu user inactive)
  Content metadata: 5 phút (catalog thay đổi không thường xuyên)
  User preferences: 10 phút
  Không set TTL vô hạn → tránh memory leak và stale data vĩnh viễn
```

---

## 8. Workflow Orchestration

### 8.1 ⑫ Apache Maestro vs Airflow vs Luigi

**Tại sao Netflix tự build Maestro thay vì dùng Airflow:**

| Tiêu chí | Maestro (Netflix) | Apache Airflow |
|---|---|---|
| Scale (DAGs) | Hàng triệu jobs/ngày | Hàng nghìn DAGs (scheduler bottleneck) |
| Parameterized workflows | Native (foreach, conditional) | Cần custom plugins |
| Versioning | Workflow versioning built-in | Không native |
| Step-level retry | Fine-grained (retry chỉ step fail) | Task-level retry, re-run toàn DAG |
| Integration với Titus | Native | Cần custom operator |

**Khi nào Maestro phát huy:**
- Hàng nghìn Spark jobs mỗi ngày với dependencies phức tạp
- Workflow cần dynamic parallelism (16 jobs song song, mỗi job xử lý 1 content category)
- Cần audit trail đầy đủ: job nào chạy khi nào, input/output là gì, ai trigger

**Cách Netflix thiết kế Maestro để vận hành:**

```
Step dependency tracking:
  Step chỉ start khi tất cả upstream steps = SUCCESS
  Nếu step fail sau N retries → workflow halt, alert on-call
  Upstream data freshness check trước khi step start:
    Nếu input table stale quá ngưỡng → step wait hoặc fail fast
    Tránh tình huống chạy job với data thiếu → ra kết quả sai

Parameterized execution:
  workflow_date = ${YESTERDAY}  →  inject tự động
  16 parallel Spark jobs = foreach content_category in [action, comedy, ...]
  → Maestro tự spawn 16 step instances, track riêng từng cái

Retry strategy:
  Step fail → retry sau 5 phút, tối đa 3 lần
  Sau 3 lần fail → page on-call engineer
  Không retry toàn bộ workflow — chỉ retry step fail
  → Tiết kiệm compute, không chạy lại steps đã succeed

SLA monitoring:
  Mỗi workflow có expected_completion_time
  Nếu vượt quá 20% → alert "job chạy chậm hơn bình thường"
  Trước khi downstream systems cần data (ví dụ 8am) → đảm bảo job xong lúc 7am
```

---

## 9. Observability Stack

### 9.1 ⑭ Atlas (Metrics) vs Prometheus vs CloudWatch · ⑮ Edgar (Tracing)

**Tại sao Netflix build Atlas thay vì dùng Prometheus:**

| Tiêu chí | Atlas (Netflix OSS) | Prometheus |
|---|---|---|
| Scale | Hàng tỷ time series | Hàng triệu (Thanos cần cho scale lớn hơn) |
| Aggregation | Server-side (giảm network) | Client-side pull model |
| Expression language | Powerful stack-based DSL | PromQL (simpler) |
| Long-term storage | Native (custom binary format) | Cần Thanos/Cortex add-on |
| Multi-region | Federated aggregation built-in | Phức tạp hơn |

**Khi nào Atlas phát huy tốt nhất:**
- Aggregating metrics từ hàng nghìn microservice instances (Atlas tự aggregate trước khi store)
- Query "average latency của tất cả instances của service X trong 30 ngày qua" — Prometheus gặp khó ở long-term storage

**Cách Netflix thiết kế observability để vận hành databases:**

```
Symptom-based alerting (không phải cause-based):
  SAI:  "CPU > 80% on mysql-shard-5"
          → False positive khi CPU cao nhưng user không bị ảnh hưởng
  ĐÚNG: "P99 latency /api/profile > 500ms trong 5 phút liên tiếp"
          → Trực tiếp đo user experience

SLO-based error budget:
  MySQL user profile SLO = 99.95% availability/tháng
  = 22 phút downtime/tháng được phép
  Alert khi burn rate = 14× faster than normal (tiêu hết budget trong < 2 ngày)
  Alert severity = CRITICAL → page on-call immediately

Database-specific thresholds:
  MySQL replication lag > 5 giây → WARNING → investigate
  MySQL replication lag > 30 giây → CRITICAL → route all traffic to primary
  Cassandra read P99 > 50ms → WARNING
  Cassandra pending compactions > 100 → WARNING (disk pressure)
  Kafka consumer lag growing (không stable) → alert relevant team
  Druid segment loading > 30 phút → alert (Historical node pressure)
```

### 9.2 Chaos Engineering — Tại sao Netflix chủ động phá hệ thống

**Nguyên lý:** Biết hệ thống sẽ fail. Thà fail trong môi trường được kiểm soát (Chaos Monkey) còn hơn bị surprise lúc 3am.

**Áp dụng cụ thể với databases:**

| Chaos experiment | Kết quả expect | Nếu fail → sửa gì |
|---|---|---|
| Kill MySQL read replica | Traffic tự route sang replica khác, latency tăng nhẹ | Shard router không có health check → thêm circuit breaker |
| Block Cassandra node 10 phút | RF=3 + LOCAL_QUORUM → vẫn serve, hints tích lũy | Nếu reads fail → consistency level quá strict → điều chỉnh |
| Tăng Kafka consumer lag nhân tạo | Alert fire trong < 5 phút | Nếu không alert → threshold sai, metric missing |
| Inject S3 slowness 500ms | Presto queries timeout gracefully, không hang | Nếu hang → thiếu query timeout config |

---

## 10. Tổng Hợp Quyết Định Công Cụ

### Ma trận: Chọn công cụ theo tình huống

| Tình huống | Công cụ chọn | Tại sao KHÔNG chọn các công cụ khác |
|---|---|---|
| Billing transaction | MySQL (ACID) | Cassandra: không có multi-row transaction; MongoDB: đã chọn MySQL từ trước |
| 260M users' viewing history | Cassandra | MySQL: không scale ngang đến mức này; DynamoDB: vendor lock-in |
| Session cache | EVCache | Redis: EVCache đã được invest sâu, zone-aware routing tốt hơn Redis Cluster setup ban đầu |
| Ad-hoc analyst query | Presto/Trino | Spark: startup overhead 2 phút không phù hợp; Redshift: phải load data riêng |
| Daily batch ETL (TB) | Apache Spark | Flink: batch throughput của Spark cao hơn; Presto: không tối ưu cho write-heavy batch |
| Realtime dashboard (< 1 giây) | Apache Druid | Presto: không đạt sub-second SLA ở scale; ClickHouse: cần re-evaluate khi Netflix ra quyết định |
| Stream processing (< 1 phút) | Apache Flink | Spark Streaming: micro-batch latency không đủ thấp; Kafka Streams: thiếu tính năng stateful complex |
| OLTP → OLAP bridge | Debezium + Kafka | Dual write: unreliable; Polling: missing DELETEs, tăng DB load |
| OLAP storage | S3 + Iceberg | Redshift: lock-in, cost ở exabyte scale; Hive format: không có ACID |
| Table metadata | Hive Metastore + Metacat | Glue catalog: ít control hơn; custom solution: không cần thiết |
| Job orchestration | Apache Maestro | Airflow: scheduler bottleneck ở hàng triệu jobs/ngày |
| Metrics platform | Atlas | Prometheus+Thanos: thêm complexity ở scale hàng tỷ time series |
| Distributed tracing | Edgar (internal Zipkin-based) | Jaeger: đủ dùng nhưng Netflix cần tích hợp sâu với internal infrastructure |
| Realtime anomaly detection | Flink + Atlas alert | Spark: latency quá cao cho fraud detection |

### Nguyên tắc quyết định Netflix dùng xuyên suốt

1. **Chọn eventual consistency trừ khi bắt buộc strong consistency** — billing và auth cần strong; viewing history, ratings không cần → giảm latency và tăng availability

2. **Tách storage khỏi compute** — S3 cho storage, Presto/Spark cho compute → scale độc lập, dùng Spot Instance cho compute

3. **Không share database giữa services** — mỗi microservice có DB riêng → schema change không cascade, autoscale độc lập

4. **Thiết kế cho failure, không phải để tránh failure** — RF=3, Multi-AZ, circuit breaker, Chaos Engineering → assume failure xảy ra, thiết kế để recover tự động

5. **Open source khi đã stable, giữ internal khi còn competitive advantage** — Hollow, Maestro, Delta, EVCache đều đã được open-source sau khi mature nội bộ

---

*Tổng hợp từ Netflix Tech Blog, QCon talks, Spark Summit, và các paper kỹ thuật công khai của Netflix Engineering.*
