# Sockudo 完整配置參考

> 本文檔基於 Sockudo v3.1.0 原始碼撰寫，涵蓋所有可配置項目。

---

## 目錄

- [1. 配置方式](#1-配置方式)
- [2. 核心設定](#2-核心設定)
- [3. 適配器配置 (adapter)](#3-適配器配置-adapter)
- [4. 應用管理器配置 (app_manager)](#4-應用管理器配置-app_manager)
- [5. 應用設定 (App)](#5-應用設定-app)
- [6. 資料庫配置 (database)](#6-資料庫配置-database)
- [7. 快取配置 (cache)](#7-快取配置-cache)
- [8. 佇列配置 (queue)](#8-佇列配置-queue)
- [9. 速率限制器配置 (rate_limiter)](#9-速率限制器配置-rate_limiter)
- [10. 指標配置 (metrics)](#10-指標配置-metrics)
- [11. WebSocket 配置 (websocket)](#11-websocket-配置-websocket)
- [12. SSL/TLS 配置 (ssl)](#12-ssltls-配置-ssl)
- [13. Unix Socket 配置 (unix_socket)](#13-unix-socket-配置-unix_socket)
- [14. CORS 配置 (cors)](#14-cors-配置-cors)
- [15. Delta 壓縮配置 (delta_compression)](#15-delta-壓縮配置-delta_compression)
- [16. 標籤過濾配置 (tag_filtering)](#16-標籤過濾配置-tag_filtering)
- [17. Webhook 配置 (webhooks)](#17-webhook-配置-webhooks)
- [18. 清理配置 (cleanup)](#18-清理配置-cleanup)
- [19. 叢集健康配置 (cluster_health)](#19-叢集健康配置-cluster_health)
- [20. 日誌配置 (logging)](#20-日誌配置-logging)
- [21. 其他設定](#21-其他設定)
- [22. 完整配置檔案範例](#22-完整配置檔案範例)
- [23. 環境變數快速參考表](#23-環境變數快速參考表)

---

## 1. 配置方式

Sockudo 支援兩種配置方式：

### JSON 配置檔案

透過 `--config` 參數指定：

```bash
./sockudo --config /path/to/config.json
# 或使用簡寫
./sockudo -c config.json
```

### 環境變數

直接設定環境變數，或使用 `.env` 檔案：

```bash
PORT=6001 HOST=0.0.0.0 ./sockudo
```

### 優先順序

**環境變數** > **配置檔案** > **程式預設值**

---

## 2. 核心設定

最上層的伺服器配置項目：

| 配置項目 | JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|----------|------|--------|------|
| 伺服器主機 | `host` | `HOST` | String | `"0.0.0.0"` | 伺服器綁定位址 |
| 伺服器埠號 | `port` | `PORT` | u16 | `6001` | 伺服器監聽埠號 |
| 除錯模式 | `debug` | `DEBUG` 或 `DEBUG_MODE` | bool | `false` | 啟用除錯模式（`DEBUG` 優先於 `DEBUG_MODE`） |
| 運行模式 | `mode` | `ENVIRONMENT` | String | `"production"` | `development`、`staging` 或 `production` |
| 路徑前綴 | `path_prefix` | — | String | `"/"` | HTTP 路徑前綴（已定義但目前未套用於路由） |
| 關閉寬限期 | `shutdown_grace_period` | `SHUTDOWN_GRACE_PERIOD` | u64 | `10` | 優雅關閉等待秒數 |
| 使用者認證逾時 | `user_authentication_timeout` | `USER_AUTHENTICATION_TIMEOUT` | u64 | `3600` | 使用者認證 session 逾時（秒） |
| WebSocket 最大酬載 | `websocket_max_payload_kb` | `WEBSOCKET_MAX_PAYLOAD_KB` | u32 | `64` | WebSocket 訊息最大大小（KB） |
| 活動逾時 | `activity_timeout` | — | u64 | `120` | 連線閒置逾時（秒） |

---

## 3. 適配器配置 (adapter)

適配器用於管理多節點間的訊息同步。

### 適配器驅動

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `adapter.driver` | `ADAPTER_DRIVER` | String | `"local"` | `local`、`redis`、`redis-cluster`、`nats` |
| `adapter.enable_socket_counting` | `ADAPTER_ENABLE_SOCKET_COUNTING` | bool | `true` | 啟用 socket 計數（關閉可提升效能） |
| `adapter.buffer_multiplier_per_cpu` | — | usize | `64` | 每 CPU 核心的緩衝倍數 |

### Redis 適配器 (`adapter.redis`)

| JSON 欄位 | 類型 | 預設值 | 說明 |
|----------|------|--------|------|
| `adapter.redis.requests_timeout` | u64 | `5000` | 請求逾時（毫秒） |
| `adapter.redis.prefix` | String | `"sockudo_adapter:"` | Redis key 前綴 |
| `adapter.redis.cluster_mode` | bool | `false` | 是否使用叢集模式 |

### Redis Cluster 適配器 (`adapter.cluster`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `adapter.cluster.nodes` | `REDIS_CLUSTER_NODES` | Vec\<String\> | `[]` | 叢集節點列表 |
| `adapter.cluster.prefix` | — | String | `"sockudo_adapter:"` | Key 前綴 |
| `adapter.cluster.request_timeout_ms` | — | u64 | `5000` | 請求逾時（毫秒） |
| `adapter.cluster.use_connection_manager` | — | bool | `true` | 使用連線管理器 |
| `adapter.cluster.use_sharded_pubsub` | — | bool | `false` | 使用分片 Pub/Sub（需 Redis 7.0+） |

### NATS 適配器 (`adapter.nats`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `adapter.nats.servers` | `NATS_SERVERS` | Vec\<String\> | `["nats://localhost:4222"]` | NATS 伺服器列表（逗號分隔） |
| `adapter.nats.prefix` | — | String | 視 feature 而定 | Subject 前綴 |
| `adapter.nats.request_timeout_ms` | `NATS_REQUEST_TIMEOUT_MS` | u64 | `5000` | 請求逾時（毫秒） |
| `adapter.nats.username` | `NATS_USERNAME` | String? | `null` | 認證用戶名 |
| `adapter.nats.password` | `NATS_PASSWORD` | String? | `null` | 認證密碼 |
| `adapter.nats.token` | `NATS_TOKEN` | String? | `null` | 認證令牌 |
| `adapter.nats.connection_timeout_ms` | `NATS_CONNECTION_TIMEOUT_MS` | u64 | `5000` | 連線逾時（毫秒） |

---

## 4. 應用管理器配置 (app_manager)

管理應用程式（App）的元資料儲存方式。

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `app_manager.driver` | `APP_MANAGER_DRIVER` | String | `"memory"` | `memory`、`mysql`、`pgsql`、`dynamodb`、`scylladb` |

### 應用元資料快取 (`app_manager.cache`)

| JSON 欄位 | 類型 | 預設值 | 說明 |
|----------|------|--------|------|
| `app_manager.cache.enabled` | bool | `true` | 啟用應用元資料快取 |
| `app_manager.cache.ttl` | u64 | `300` | 快取 TTL（秒） |

### ScyllaDB 設定 (`app_manager.scylladb`)

| JSON 欄位 | 類型 | 預設值 | 說明 |
|----------|------|--------|------|
| `app_manager.scylladb.nodes` | Vec\<String\> | `["127.0.0.1:9042"]` | ScyllaDB 節點列表 |
| `app_manager.scylladb.keyspace` | String | `"sockudo"` | Keyspace 名稱 |
| `app_manager.scylladb.table_name` | String | `"applications"` | 表名 |
| `app_manager.scylladb.username` | String? | `null` | 認證用戶名 |
| `app_manager.scylladb.password` | String? | `null` | 認證密碼 |
| `app_manager.scylladb.replication_class` | String | `"SimpleStrategy"` | 複製策略 |
| `app_manager.scylladb.replication_factor` | u32 | `3` | 複製因子 |

---

## 5. 應用設定 (App)

每個應用在 `app_manager.array.apps` 陣列中定義。以下為所有可配置欄位：

| JSON 欄位 | 環境變數（預設應用） | 類型 | 說明 |
|----------|---------------------|------|------|
| `id` | `SOCKUDO_DEFAULT_APP_ID` | String | 應用 ID（必填） |
| `key` | `SOCKUDO_DEFAULT_APP_KEY` | String | 公開金鑰（必填） |
| `secret` | `SOCKUDO_DEFAULT_APP_SECRET` | String | 私密金鑰（必填，務必保密） |
| `enabled` | `SOCKUDO_DEFAULT_APP_ENABLED` | bool | 是否啟用此應用（必填） |
| `max_connections` | `SOCKUDO_DEFAULT_APP_MAX_CONNECTIONS` | u32 | 最大並行 WebSocket 連線數（必填） |
| `enable_client_messages` | `SOCKUDO_ENABLE_CLIENT_MESSAGES` | bool | 允許客戶端對客戶端訊息（必填） |
| `max_client_events_per_second` | `SOCKUDO_DEFAULT_APP_MAX_CLIENT_EVENTS_PER_SECOND` | u32 | 客戶端事件速率限制（必填） |
| `max_backend_events_per_second` | `SOCKUDO_DEFAULT_APP_MAX_BACKEND_EVENTS_PER_SECOND` | u32? | 後端事件速率限制 |
| `max_read_requests_per_second` | `SOCKUDO_DEFAULT_APP_MAX_READ_REQUESTS_PER_SECOND` | u32? | 讀取 API 速率限制 |
| `max_presence_members_per_channel` | `SOCKUDO_DEFAULT_APP_MAX_PRESENCE_MEMBERS_PER_CHANNEL` | u32? | 每頻道最大 Presence 成員數 |
| `max_presence_member_size_in_kb` | `SOCKUDO_DEFAULT_APP_MAX_PRESENCE_MEMBER_SIZE_IN_KB` | u32? | 成員資料最大大小（KB） |
| `max_channel_name_length` | `SOCKUDO_DEFAULT_APP_MAX_CHANNEL_NAME_LENGTH` | u32? | 頻道名稱最大長度 |
| `max_event_channels_at_once` | `SOCKUDO_DEFAULT_APP_MAX_EVENT_CHANNELS_AT_ONCE` | u32? | 單次事件最大頻道數 |
| `max_event_name_length` | `SOCKUDO_DEFAULT_APP_MAX_EVENT_NAME_LENGTH` | u32? | 事件名稱最大長度 |
| `max_event_payload_in_kb` | `SOCKUDO_DEFAULT_APP_MAX_EVENT_PAYLOAD_IN_KB` | u32? | 事件酬載最大大小（KB） |
| `max_event_batch_size` | `SOCKUDO_DEFAULT_APP_MAX_EVENT_BATCH_SIZE` | u32? | 批次事件最大數量 |
| `enable_user_authentication` | `SOCKUDO_DEFAULT_APP_ENABLE_USER_AUTHENTICATION` | bool? | 啟用使用者認證 |
| `enable_watchlist_events` | `SOCKUDO_DEFAULT_APP_ENABLE_WATCHLIST_EVENTS` | bool? | 啟用 Watchlist 事件 |
| `allowed_origins` | `SOCKUDO_DEFAULT_APP_ALLOWED_ORIGINS` | Vec\<String\>? | 允許的來源（支援萬用字元，逗號分隔） |
| `webhooks` | — | Vec\<Webhook\>? | Webhook 端點配置 |
| `channel_delta_compression` | — | Map? | 每頻道 Delta 壓縮配置 |

### Webhook 端點配置

每個 Webhook 端點：

```json
{
  "url": "https://your-server.com/webhook",
  "event_types": ["channel_occupied", "channel_vacated", "member_added", "member_removed"],
  "filter": {
    "channel_name_prefixes": ["private-"],
    "channel_name_suffixes": [],
    "channel_name_patterns": []
  },
  "headers": {
    "X-Custom-Header": "value"
  },
  "lambda": {
    "function_name": "my-function",
    "region": "us-east-1",
    "async": true
  }
}
```

### 每頻道 Delta 壓縮配置

```json
{
  "channel_delta_compression": {
    "ticker:*": {
      "enabled": true,
      "algorithm": "Fossil",
      "conflation_key": "symbol",
      "max_messages_per_key": 10,
      "max_conflation_keys": 1000,
      "enable_tags": false
    }
  }
}
```

---

## 6. 資料庫配置 (database)

### MySQL 設定 (`database.mysql`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `database.mysql.host` | `DATABASE_MYSQL_HOST` | String | `"localhost"` | 主機位址 |
| `database.mysql.port` | `DATABASE_MYSQL_PORT` | u16 | `3306` | 埠號 |
| `database.mysql.username` | `DATABASE_MYSQL_USERNAME` | String | `"root"` | 使用者名稱 |
| `database.mysql.password` | `DATABASE_MYSQL_PASSWORD` | String | `""` | 密碼 |
| `database.mysql.database` | `DATABASE_MYSQL_DATABASE` | String | `"sockudo"` | 資料庫名稱 |
| `database.mysql.table_name` | `DATABASE_MYSQL_TABLE_NAME` | String | `"applications"` | 資料表名稱 |
| `database.mysql.pool_min` | `DATABASE_MYSQL_POOL_MIN` | u32? | `null` | 連線池最小連線數 |
| `database.mysql.pool_max` | `DATABASE_MYSQL_POOL_MAX` | u32? | `null` | 連線池最大連線數 |
| `database.mysql.connection_pool_size` | — | u32 | `10` | 連線池大小 |
| `database.mysql.cache_ttl` | — | u64 | `300` | 快取 TTL（秒） |
| `database.mysql.cache_cleanup_interval` | — | u64 | `60` | 快取清理間隔（秒） |
| `database.mysql.cache_max_capacity` | — | u64 | `100` | 快取最大容量 |

### PostgreSQL 設定 (`database.postgres`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `database.postgres.host` | `DATABASE_POSTGRES_HOST` | String | `"localhost"` | 主機位址 |
| `database.postgres.port` | `DATABASE_POSTGRES_PORT` | u16 | `5432` | 埠號 |
| `database.postgres.username` | `DATABASE_POSTGRES_USERNAME` | String | `"root"` | 使用者名稱 |
| `database.postgres.password` | `DATABASE_POSTGRES_PASSWORD` | String | `""` | 密碼 |
| `database.postgres.database` | `DATABASE_POSTGRES_DATABASE` | String | `"sockudo"` | 資料庫名稱 |
| `database.postgres.pool_min` | `DATABASE_POSTGRES_POOL_MIN` | u32? | `null` | 連線池最小連線數 |
| `database.postgres.pool_max` | `DATABASE_POSTGRES_POOL_MAX` | u32? | `null` | 連線池最大連線數 |

### Redis 設定 (`database.redis`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `database.redis.host` | `DATABASE_REDIS_HOST` | String | `"127.0.0.1"` | 主機位址 |
| `database.redis.port` | `DATABASE_REDIS_PORT` | u16 | `6379` | 埠號 |
| `database.redis.db` | `DATABASE_REDIS_DB` | u32 | `0` | 資料庫編號 |
| `database.redis.username` | `DATABASE_REDIS_USERNAME` | String? | `null` | 使用者名稱（Redis ACL） |
| `database.redis.password` | `DATABASE_REDIS_PASSWORD` | String? | `null` | 密碼 |
| `database.redis.key_prefix` | `DATABASE_REDIS_KEY_PREFIX` | String | `"sockudo:"` | Key 前綴 |
| `database.redis.name` | — | String | `"mymaster"` | Sentinel 主節點名稱 |

**Redis URL 覆寫**：設定 `REDIS_URL` 環境變數可覆寫所有 Redis 連線設定。

### Redis Sentinel 設定 (`database.redis.sentinels`)

```json
{
  "database": {
    "redis": {
      "sentinels": [
        { "host": "sentinel1.example.com", "port": 26379 },
        { "host": "sentinel2.example.com", "port": 26379 }
      ],
      "sentinel_password": "sentinel-auth-password",
      "name": "mymaster",
      "password": "redis-master-password"
    }
  }
}
```

連線 URL 格式：`redis+sentinel://[sentinelpass@]host1:port1,host2:port2/master-name/db[?password=masterpass]`

### Redis Cluster 設定 (`database.redis.cluster`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `database.redis.cluster.nodes` | `REDIS_CLUSTER_NODES` | Vec\<ClusterNode\> | `[]` | 叢集節點列表 |
| — | `DATABASE_REDIS_CLUSTER_USERNAME` | String? | `null` | 叢集認證用戶名 |
| — | `DATABASE_REDIS_CLUSTER_PASSWORD` | String? | `null` | 叢集認證密碼 |
| — | `DATABASE_REDIS_CLUSTER_USE_TLS` | bool | `false` | 使用 TLS |

**叢集節點格式**：支援 `host:port`、`redis://host:port`、`rediss://host:port`、`[::1]:6379`（IPv6）。

### DynamoDB 設定 (`database.dynamodb`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `database.dynamodb.region` | `DATABASE_DYNAMODB_REGION` | String | `"us-east-1"` | AWS 區域 |
| `database.dynamodb.table_name` | `DATABASE_DYNAMODB_TABLE_NAME` | String | `"sockudo-applications"` | 資料表名稱 |
| `database.dynamodb.endpoint_url` | `DATABASE_DYNAMODB_ENDPOINT_URL` | String? | `null` | 自訂端點（用於本地測試） |
| — | `AWS_ACCESS_KEY_ID` | String? | `null` | AWS 存取金鑰 |
| — | `AWS_SECRET_ACCESS_KEY` | String? | `null` | AWS 私密金鑰 |

### 連線池設定 (`database_pooling`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `database_pooling.enabled` | `DATABASE_POOLING_ENABLED` | bool | `true` | 啟用連線池 |
| `database_pooling.min` | `DATABASE_POOL_MIN` | u32 | `2` | 最小連線數 |
| `database_pooling.max` | `DATABASE_POOL_MAX` | u32 | `10` | 最大連線數 |

---

## 7. 快取配置 (cache)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `cache.driver` | `CACHE_DRIVER` | String | `"memory"` | `memory`、`redis`、`redis-cluster`、`none` |

### 記憶體快取 (`cache.memory`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `cache.memory.ttl` | `CACHE_TTL_SECONDS` | u64 | `300` | 快取 TTL（秒） |
| `cache.memory.cleanup_interval` | `CACHE_CLEANUP_INTERVAL` | u64 | `60` | 清理間隔（秒） |
| `cache.memory.max_capacity` | `CACHE_MAX_CAPACITY` | u64 | `10000` | 最大快取項目數 |

### Redis 快取 (`cache.redis`)

| JSON 欄位 | 類型 | 預設值 | 說明 |
|----------|------|--------|------|
| `cache.redis.prefix` | String? | `"sockudo_cache:"` | Redis key 前綴 |
| `cache.redis.url_override` | String? | `null` | Redis URL 覆寫 |
| `cache.redis.cluster_mode` | bool | `false` | 使用叢集模式 |

---

## 8. 佇列配置 (queue)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `queue.driver` | `QUEUE_DRIVER` | String | `"memory"` | `memory`、`redis`、`redis-cluster`、`sqs`、`none` |

### Redis 佇列 (`queue.redis`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `queue.redis.concurrency` | `QUEUE_REDIS_CONCURRENCY` | u32 | `5` | 並行處理數 |
| `queue.redis.prefix` | `QUEUE_REDIS_PREFIX` | String? | `"sockudo_queue:"` | Key 前綴 |
| `queue.redis.url_override` | — | String? | `null` | Redis URL 覆寫 |
| `queue.redis.cluster_mode` | — | bool | `false` | 使用叢集模式 |

### Redis Cluster 佇列 (`queue.redis_cluster`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `queue.redis_cluster.concurrency` | `REDIS_CLUSTER_QUEUE_CONCURRENCY` | u32 | `5` | 並行處理數 |
| `queue.redis_cluster.prefix` | `REDIS_CLUSTER_QUEUE_PREFIX` | String? | `"sockudo_queue:"` | Key 前綴 |
| `queue.redis_cluster.nodes` | — | Vec\<String\> | `["redis://127.0.0.1:6379"]` | 叢集節點列表 |
| `queue.redis_cluster.request_timeout_ms` | — | u64 | `5000` | 請求逾時（毫秒） |

### SQS 佇列 (`queue.sqs`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `queue.sqs.region` | `QUEUE_SQS_REGION` | String | `"us-east-1"` | AWS 區域 |
| `queue.sqs.visibility_timeout` | `QUEUE_SQS_VISIBILITY_TIMEOUT` | i32 | `30` | 可見性逾時（秒） |
| `queue.sqs.max_messages` | `QUEUE_SQS_MAX_MESSAGES` | i32 | `10` | 每次拉取最大訊息數 |
| `queue.sqs.wait_time_seconds` | `QUEUE_SQS_WAIT_TIME_SECONDS` | i32 | `5` | 長輪詢等待時間（秒） |
| `queue.sqs.concurrency` | `QUEUE_SQS_CONCURRENCY` | u32 | `5` | 並行處理數 |
| `queue.sqs.fifo` | `QUEUE_SQS_FIFO` | bool | `false` | 使用 FIFO 佇列 |
| `queue.sqs.endpoint_url` | `QUEUE_SQS_ENDPOINT_URL` | String? | `null` | 自訂端點（用於 LocalStack） |
| `queue.sqs.message_group_id` | — | String? | `"default"` | FIFO 訊息群組 ID |

---

## 9. 速率限制器配置 (rate_limiter)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `rate_limiter.enabled` | `RATE_LIMITER_ENABLED` | bool | `true` | 啟用速率限制 |
| `rate_limiter.driver` | `RATE_LIMITER_DRIVER` | String | `"memory"` | `memory`、`redis`、`redis-cluster` |

### API 速率限制 (`rate_limiter.api_rate_limit`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `rate_limiter.api_rate_limit.max_requests` | `RATE_LIMITER_API_MAX_REQUESTS` | u32 | `100` | 最大請求數 |
| `rate_limiter.api_rate_limit.window_seconds` | `RATE_LIMITER_API_WINDOW_SECONDS` | u64 | `60` | 時間窗口（秒） |
| `rate_limiter.api_rate_limit.trust_hops` | `RATE_LIMITER_API_TRUST_HOPS` | u32? | `0` | 信任的代理跳數 |

### WebSocket 速率限制 (`rate_limiter.websocket_rate_limit`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `rate_limiter.websocket_rate_limit.max_requests` | `RATE_LIMITER_WS_MAX_REQUESTS` | u32 | `20` | 最大連線嘗試數 |
| `rate_limiter.websocket_rate_limit.window_seconds` | `RATE_LIMITER_WS_WINDOW_SECONDS` | u64 | `60` | 時間窗口（秒） |

---

## 10. 指標配置 (metrics)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `metrics.enabled` | `METRICS_ENABLED` | bool | `true` | 啟用指標收集 |
| `metrics.driver` | `METRICS_DRIVER` | String | `"prometheus"` | 指標驅動（目前僅支援 Prometheus） |
| `metrics.host` | `METRICS_HOST` | String | `"0.0.0.0"` | 指標伺服器主機 |
| `metrics.port` | `METRICS_PORT` | u16 | `9601` | 指標伺服器埠號 |
| `metrics.prometheus.prefix` | `METRICS_PROMETHEUS_PREFIX` | String | `"sockudo_"` | Prometheus 指標前綴 |

---

## 11. WebSocket 配置 (websocket)

進階 WebSocket 連線設定，用於控制背壓（backpressure）與緩衝行為。

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `websocket.max_messages` | `WEBSOCKET_MAX_MESSAGES` | usize? | `1000` | 每連線最大緩衝訊息數（`null` 或 `"none"` 停用） |
| `websocket.max_bytes` | `WEBSOCKET_MAX_BYTES` | usize? | `null` | 每連線最大緩衝位元組（例：`1048576` = 1MB） |
| `websocket.disconnect_on_buffer_full` | `WEBSOCKET_DISCONNECT_ON_BUFFER_FULL` | bool | `true` | 緩衝滿時斷開連線（`false` 則丟棄訊息） |
| `websocket.max_message_size` | `WEBSOCKET_MAX_MESSAGE_SIZE` | usize | `67108864` | 最大訊息大小（64MB） |
| `websocket.max_frame_size` | `WEBSOCKET_MAX_FRAME_SIZE` | usize | `16777216` | 最大 WebSocket Frame 大小（16MB） |
| `websocket.write_buffer_size` | `WEBSOCKET_WRITE_BUFFER_SIZE` | usize | `16384` | 寫入緩衝區大小（16KB） |
| `websocket.max_backpressure` | `WEBSOCKET_MAX_BACKPRESSURE` | usize | `1048576` | 最大背壓位元組（1MB） |
| `websocket.auto_ping` | `WEBSOCKET_AUTO_PING` | bool | `true` | 自動 Ping |
| `websocket.ping_interval` | `WEBSOCKET_PING_INTERVAL` | u32 | `30` | Ping 間隔（秒） |
| `websocket.idle_timeout` | `WEBSOCKET_IDLE_TIMEOUT` | u32 | `120` | 閒置逾時（秒） |
| `websocket.compression` | `WEBSOCKET_COMPRESSION` | String | `"disabled"` | 壓縮模式 |

**壓縮模式選項**：`disabled`、`dedicated`、`shared`、`window256b`、`window1kb`、`window2kb`、`window4kb`、`window8kb`、`window16kb`、`window32kb`

### 緩衝限制模式

| 模式 | 設定方式 | 效能特性 |
|------|----------|----------|
| 僅訊息計數（預設，最快） | 設定 `max_messages`，`max_bytes = null` | 零額外開銷 |
| 僅位元組大小 | 設定 `max_bytes`，`max_messages = null` | 原子計數器（~1-2ns/訊息） |
| 雙重限制 | 同時設定兩者 | 原子計數器 + 通道容量檢查 |

---

## 12. SSL/TLS 配置 (ssl)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `ssl.enabled` | `SSL_ENABLED` | bool | `false` | 啟用 SSL |
| `ssl.cert_path` | `SSL_CERT_PATH` | String | `""` | 憑證檔案路徑 |
| `ssl.key_path` | `SSL_KEY_PATH` | String | `""` | 私鑰檔案路徑 |
| `ssl.passphrase` | — | String? | `null` | 私鑰密碼 |
| `ssl.ca_path` | — | String? | `null` | CA 憑證路徑 |
| `ssl.redirect_http` | `SSL_REDIRECT_HTTP` | bool | `false` | HTTP 重導向至 HTTPS |
| `ssl.http_port` | `SSL_HTTP_PORT` | u16? | `80` | HTTP 重導向埠號 |

---

## 13. Unix Socket 配置 (unix_socket)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `unix_socket.enabled` | `UNIX_SOCKET_ENABLED` | bool | `false` | 啟用 Unix Socket |
| `unix_socket.path` | `UNIX_SOCKET_PATH` | String | `"/var/run/sockudo/sockudo.sock"` | Socket 檔案路徑 |
| `unix_socket.permission_mode` | `UNIX_SOCKET_PERMISSION_MODE` | String | `"660"` | 檔案權限（八進位） |

> **注意**：Unix Socket 僅在 Unix-like 系統（Linux、macOS）上可用。在 Windows 上此配置會被忽略。

---

## 14. CORS 配置 (cors)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `cors.credentials` | `CORS_CREDENTIALS` | bool | `true` | 允許攜帶認證 |
| `cors.origin` | `CORS_ORIGINS` | Vec\<String\> | `["*"]` | 允許的來源 |
| `cors.methods` | `CORS_METHODS` | Vec\<String\> | `["GET", "POST", "OPTIONS"]` | 允許的方法 |
| `cors.allowed_headers` | `CORS_HEADERS` | Vec\<String\> | `["Authorization", "Content-Type", "X-Requested-With", "Accept"]` | 允許的標頭 |

---

## 15. Delta 壓縮配置 (delta_compression)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `delta_compression.enabled` | `DELTA_COMPRESSION_ENABLED` | bool | `true` | 啟用 Delta 壓縮 |
| `delta_compression.algorithm` | `DELTA_COMPRESSION_ALGORITHM` | String | `"fossil"` | 演算法：`fossil` 或 `xdelta3` |
| `delta_compression.full_message_interval` | `DELTA_COMPRESSION_FULL_MESSAGE_INTERVAL` | u32 | `10` | 每 N 則訊息發送完整訊息 |
| `delta_compression.min_message_size` | `DELTA_COMPRESSION_MIN_MESSAGE_SIZE` | usize | `100` | 最小壓縮大小（位元組） |
| `delta_compression.max_state_age_secs` | `DELTA_COMPRESSION_MAX_STATE_AGE_SECS` | u64 | `300` | 狀態最大保留時間（秒） |
| `delta_compression.max_channel_states_per_socket` | — | usize | `100` | 每 Socket 最大頻道狀態數 |
| `delta_compression.cluster_coordination` | `DELTA_COMPRESSION_CLUSTER_COORDINATION` | bool | `false` | 啟用叢集協調 |
| `delta_compression.omit_delta_algorithm` | `DELTA_COMPRESSION_OMIT_DELTA_ALGORITHM` | bool | `false` | 省略演算法欄位（節省 ~20-30 bytes/訊息） |

> **重要**：加密頻道（`private-encrypted-*`）會自動停用 Delta 壓縮，因為加密後的資料無法有效壓縮。

---

## 16. 標籤過濾配置 (tag_filtering)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `tag_filtering.enabled` | `TAG_FILTERING_ENABLED` | bool | `false` | 啟用標籤過濾（預設停用以相容 Pusher） |
| `tag_filtering.enable_tags` | `TAG_FILTERING_ENABLE_TAGS` | bool | `true` | 在訊息中包含標籤 |

> **注意**：`enable_tags: false` 會在伺服器端過濾後移除標籤再傳送。

---

## 17. Webhook 配置 (webhooks)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `webhooks.batching.enabled` | `WEBHOOK_BATCHING_ENABLED` | bool | `true` | 啟用 Webhook 批次處理 |
| `webhooks.batching.duration` | `WEBHOOK_BATCHING_DURATION` | u64 | `50` | 批次收集時間（毫秒） |
| `webhooks.batching.size` | — | usize | `100` | 批次最大事件數 |

---

## 18. 清理配置 (cleanup)

控制 WebSocket 斷線後的非同步清理行為。

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `cleanup.async_enabled` | `CLEANUP_ASYNC_ENABLED` | bool | `true` | 啟用非同步清理 |
| `cleanup.fallback_to_sync` | `CLEANUP_FALLBACK_TO_SYNC` | bool | `true` | 非同步失敗時回退同步清理 |
| `cleanup.queue_buffer_size` | `CLEANUP_QUEUE_BUFFER_SIZE` | usize | `50000` | 清理佇列緩衝大小 |
| `cleanup.batch_size` | `CLEANUP_BATCH_SIZE` | usize | `25` | 批次清理數量 |
| `cleanup.batch_timeout_ms` | `CLEANUP_BATCH_TIMEOUT_MS` | u64 | `50` | 批次逾時（毫秒） |
| `cleanup.worker_threads` | `CLEANUP_WORKER_THREADS` | String | `"auto"` | 工作執行緒數（`"auto"` = 25% CPU 核心數，最少 1，最多 4） |
| `cleanup.max_retry_attempts` | `CLEANUP_MAX_RETRY_ATTEMPTS` | u32 | `2` | 最大重試次數 |

---

## 19. 叢集健康配置 (cluster_health)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `adapter.cluster_health.enabled` | — | bool | `true` | 啟用叢集健康監控 |
| `adapter.cluster_health.heartbeat_interval_ms` | — | u64 | `10000` | 心跳間隔（毫秒） |
| `adapter.cluster_health.node_timeout_ms` | — | u64 | `30000` | 節點逾時（毫秒） |
| `adapter.cluster_health.cleanup_interval_ms` | — | u64 | `10000` | 清理間隔（毫秒） |

> **建議**：`heartbeat_interval_ms` 應 ≤ `node_timeout_ms / 3`。

---

## 20. 日誌配置 (logging)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `logging.colors_enabled` | `LOG_COLORS_ENABLED` | bool | `true` | 啟用彩色輸出 |
| `logging.include_target` | `LOG_INCLUDE_TARGET` | bool | `true` | 包含模組目標 |
| — | `LOG_OUTPUT_FORMAT` | String | `"human"` | 日誌格式（`human` 或 `json`，**必須透過環境變數設定**） |

> **重要**：JSON 格式（`LOG_OUTPUT_FORMAT=json`）只能透過環境變數在啟動時設定，無法在配置檔案中配置。這是 tracing 訂閱者的技術限制。

---

## 21. 其他設定

### 頻道限制 (`channel_limits`)

| JSON 欄位 | 類型 | 預設值 | 說明 |
|----------|------|--------|------|
| `channel_limits.max_name_length` | u32 | `200` | 頻道名稱最大長度 |
| `channel_limits.cache_ttl` | u64 | `3600` | 頻道快取 TTL（秒） |

### 事件限制 (`event_limits`)

| JSON 欄位 | 類型 | 預設值 | 說明 |
|----------|------|--------|------|
| `event_limits.max_channels_at_once` | u32 | `100` | 單次事件最大頻道數 |
| `event_limits.max_name_length` | u32 | `200` | 事件名稱最大長度 |
| `event_limits.max_payload_in_kb` | u32 | `100` | 事件酬載最大大小（KB） |
| `event_limits.max_batch_size` | u32 | `10` | 批次事件最大數量 |

### Presence 配置 (`presence`)

| JSON 欄位 | 類型 | 預設值 | 說明 |
|----------|------|--------|------|
| `presence.max_members_per_channel` | u32 | `100` | 每頻道最大成員數 |
| `presence.max_member_size_in_kb` | u32 | `2` | 成員資料最大大小（KB） |

### HTTP API 配置 (`http_api`)

| JSON 欄位 | 類型 | 預設值 | 說明 |
|----------|------|--------|------|
| `http_api.request_limit_in_mb` | u32 | `10` | HTTP 請求最大大小（MB） |
| `http_api.accept_traffic.memory_threshold` | f64 | `0.90` | 記憶體使用量閾值（90%） |

### 實例配置 (`instance`)

| JSON 欄位 | 環境變數 | 類型 | 預設值 | 說明 |
|----------|----------|------|--------|------|
| `instance.process_id` | `INSTANCE_PROCESS_ID` | String | UUID 自動產生 | 唯一程序識別碼 |

---

## 22. 完整配置檔案範例

```json
{
  "debug": false,
  "host": "0.0.0.0",
  "port": 6001,
  "mode": "production",
  "shutdown_grace_period": 10,
  "websocket_max_payload_kb": 64,
  "user_authentication_timeout": 3600,
  "activity_timeout": 120,

  "adapter": {
    "driver": "local",
    "enable_socket_counting": true,
    "cluster_health": {
      "enabled": true,
      "heartbeat_interval_ms": 10000,
      "node_timeout_ms": 30000,
      "cleanup_interval_ms": 10000
    }
  },

  "app_manager": {
    "driver": "memory",
    "array": {
      "apps": [
        {
          "id": "my-app-id",
          "key": "my-app-key",
          "secret": "my-app-secret",
          "max_connections": 100000,
          "enable_client_messages": true,
          "enable_user_authentication": false,
          "enabled": true,
          "max_client_events_per_second": 1000,
          "allowed_origins": ["*"]
        }
      ]
    },
    "cache": {
      "enabled": true,
      "ttl": 300
    }
  },

  "database": {
    "redis": {
      "host": "127.0.0.1",
      "port": 6379,
      "db": 0,
      "key_prefix": "sockudo:"
    }
  },

  "cache": {
    "driver": "memory",
    "memory": {
      "ttl": 300,
      "cleanup_interval": 60,
      "max_capacity": 10000
    }
  },

  "queue": {
    "driver": "memory"
  },

  "rate_limiter": {
    "enabled": true,
    "driver": "memory",
    "api_rate_limit": {
      "max_requests": 100,
      "window_seconds": 60
    },
    "websocket_rate_limit": {
      "max_requests": 20,
      "window_seconds": 60
    }
  },

  "metrics": {
    "enabled": true,
    "driver": "prometheus",
    "host": "0.0.0.0",
    "port": 9601,
    "prometheus": {
      "prefix": "sockudo_"
    }
  },

  "ssl": {
    "enabled": false
  },

  "cors": {
    "credentials": true,
    "origin": ["*"],
    "methods": ["GET", "POST", "OPTIONS"],
    "allowed_headers": ["Authorization", "Content-Type", "X-Requested-With", "Accept"]
  },

  "channel_limits": {
    "max_name_length": 200,
    "cache_ttl": 3600
  },

  "event_limits": {
    "max_channels_at_once": 100,
    "max_name_length": 200,
    "max_payload_in_kb": 100,
    "max_batch_size": 10
  },

  "presence": {
    "max_members_per_channel": 100,
    "max_member_size_in_kb": 2
  },

  "http_api": {
    "request_limit_in_mb": 10
  },

  "webhooks": {
    "batching": {
      "enabled": true,
      "duration": 50,
      "size": 100
    }
  },

  "cleanup": {
    "async_enabled": true,
    "fallback_to_sync": true,
    "queue_buffer_size": 50000,
    "batch_size": 25,
    "batch_timeout_ms": 50,
    "worker_threads": "auto",
    "max_retry_attempts": 2
  },

  "delta_compression": {
    "enabled": true,
    "algorithm": "fossil",
    "full_message_interval": 10,
    "min_message_size": 100,
    "max_state_age_secs": 300,
    "max_channel_states_per_socket": 100,
    "cluster_coordination": false,
    "omit_delta_algorithm": false
  },

  "tag_filtering": {
    "enabled": false,
    "enable_tags": true
  },

  "websocket": {
    "max_messages": 1000,
    "max_bytes": null,
    "disconnect_on_buffer_full": true,
    "max_message_size": 67108864,
    "max_frame_size": 16777216,
    "write_buffer_size": 16384,
    "max_backpressure": 1048576,
    "auto_ping": true,
    "ping_interval": 30,
    "idle_timeout": 120,
    "compression": "disabled"
  },

  "logging": {
    "colors_enabled": true,
    "include_target": true
  }
}
```

---

## 23. 環境變數快速參考表

### 核心設定

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `HOST` | 伺服器主機 | `0.0.0.0` |
| `PORT` | 伺服器埠號 | `6001` |
| `DEBUG` / `DEBUG_MODE` | 除錯模式 | `false` |
| `ENVIRONMENT` | 運行模式 | `production` |
| `SHUTDOWN_GRACE_PERIOD` | 關閉寬限期（秒） | `10` |
| `INSTANCE_PROCESS_ID` | 程序 ID | UUID |

### 驅動選擇

| 環境變數 | 說明 | 選項 |
|----------|------|------|
| `ADAPTER_DRIVER` | 適配器驅動 | `local`, `redis`, `redis-cluster`, `nats` |
| `APP_MANAGER_DRIVER` | 應用管理器 | `memory`, `mysql`, `pgsql`, `dynamodb`, `scylladb` |
| `CACHE_DRIVER` | 快取驅動 | `memory`, `redis`, `redis-cluster`, `none` |
| `QUEUE_DRIVER` | 佇列驅動 | `memory`, `redis`, `redis-cluster`, `sqs`, `none` |
| `RATE_LIMITER_DRIVER` | 速率限制器 | `memory`, `redis`, `redis-cluster` |

### 預設應用設定

| 環境變數 | 說明 |
|----------|------|
| `SOCKUDO_DEFAULT_APP_ID` | 應用 ID |
| `SOCKUDO_DEFAULT_APP_KEY` | 公開金鑰 |
| `SOCKUDO_DEFAULT_APP_SECRET` | 私密金鑰 |
| `SOCKUDO_DEFAULT_APP_ENABLED` | 啟用 |
| `SOCKUDO_DEFAULT_APP_MAX_CONNECTIONS` | 最大連線數 |
| `SOCKUDO_ENABLE_CLIENT_MESSAGES` | 客戶端訊息 |
| `SOCKUDO_DEFAULT_APP_MAX_CLIENT_EVENTS_PER_SECOND` | 客戶端事件速率 |
| `SOCKUDO_DEFAULT_APP_MAX_BACKEND_EVENTS_PER_SECOND` | 後端事件速率 |
| `SOCKUDO_DEFAULT_APP_MAX_READ_REQUESTS_PER_SECOND` | 讀取速率 |
| `SOCKUDO_DEFAULT_APP_ALLOWED_ORIGINS` | 允許的來源 |

### 資料庫

| 環境變數 | 說明 |
|----------|------|
| `DATABASE_REDIS_HOST` | Redis 主機 |
| `DATABASE_REDIS_PORT` | Redis 埠號 |
| `DATABASE_REDIS_PASSWORD` | Redis 密碼 |
| `DATABASE_REDIS_DB` | Redis 資料庫 |
| `REDIS_URL` | Redis 連線字串覆寫 |
| `REDIS_CLUSTER_NODES` | Redis Cluster 節點 |
| `DATABASE_MYSQL_HOST` | MySQL 主機 |
| `DATABASE_MYSQL_PORT` | MySQL 埠號 |
| `DATABASE_MYSQL_USERNAME` | MySQL 用戶名 |
| `DATABASE_MYSQL_PASSWORD` | MySQL 密碼 |
| `DATABASE_MYSQL_DATABASE` | MySQL 資料庫 |
| `DATABASE_POSTGRES_HOST` | PostgreSQL 主機 |
| `DATABASE_POSTGRES_PORT` | PostgreSQL 埠號 |

### WebSocket

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `WEBSOCKET_MAX_PAYLOAD_KB` | 最大酬載（KB） | `64` |
| `WEBSOCKET_MAX_MESSAGES` | 最大緩衝訊息數 | `1000` |
| `WEBSOCKET_MAX_BYTES` | 最大緩衝位元組 | — |
| `WEBSOCKET_DISCONNECT_ON_BUFFER_FULL` | 緩衝滿時斷線 | `true` |
| `WEBSOCKET_COMPRESSION` | 壓縮模式 | `disabled` |
| `WEBSOCKET_AUTO_PING` | 自動 Ping | `true` |
| `WEBSOCKET_PING_INTERVAL` | Ping 間隔（秒） | `30` |
| `WEBSOCKET_IDLE_TIMEOUT` | 閒置逾時（秒） | `120` |

### SSL/TLS

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `SSL_ENABLED` | 啟用 SSL | `false` |
| `SSL_CERT_PATH` | 憑證路徑 | — |
| `SSL_KEY_PATH` | 私鑰路徑 | — |
| `SSL_REDIRECT_HTTP` | HTTP 重導向 | `false` |

### 速率限制

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `RATE_LIMITER_ENABLED` | 啟用 | `true` |
| `RATE_LIMITER_API_MAX_REQUESTS` | API 最大請求數 | `100` |
| `RATE_LIMITER_API_WINDOW_SECONDS` | API 窗口（秒） | `60` |
| `RATE_LIMITER_WS_MAX_REQUESTS` | WS 最大連線數 | `20` |
| `RATE_LIMITER_WS_WINDOW_SECONDS` | WS 窗口（秒） | `60` |

### 日誌

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `LOG_OUTPUT_FORMAT` | 日誌格式 | `human` |
| `LOG_COLORS_ENABLED` | 彩色輸出 | `true` |
| `LOG_INCLUDE_TARGET` | 包含模組目標 | `true` |
