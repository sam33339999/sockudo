# Sockudo 繁體中文文檔

> **版本：** 3.1.0 | **協議版本：** Pusher Protocol v7

Sockudo 是一個用 Rust 編寫的高效能即時 WebSocket 伺服器，實作了 [Pusher Protocol](https://pusher.com/docs/channels/library_auth_reference/pusher-websockets-protocol/)，支援水平擴展、多種後端適配器以及即時通訊功能。

---

## 📖 文檔目錄

| 文檔 | 說明 |
|------|------|
| [完整配置參考](./CONFIGURATION.md) | 所有配置項目、環境變數、預設值的完整說明 |
| [API 參考](./API_REFERENCE.md) | HTTP REST API 端點與 WebSocket 協議詳細文檔 |
| [整合指南](./INTEGRATION.md) | Golang、PHP、Laravel、Rust 客戶端整合教學 |
| ↳ [Laravel Private / Presence Channel](./INTEGRATION.md#31-laravel-private--presence-channel-深入教學) | Laravel 私有頻道與 Presence 頻道深入教學 |
| [時序圖](./SEQUENCE_DIAGRAMS.md) | WebSocket 連線、頻道訂閱、認證等流程的時序圖 |

---

## 🚀 快速開始

### 1. 安裝與啟動

```bash
# 使用 Docker 快速啟動
make quick-start

# 或從原始碼編譯（僅本地功能，編譯最快）
cargo build --release

# 啟動伺服器
./target/release/sockudo --config config/config.json
```

### 2. 最小配置範例

建立 `config.json`：

```json
{
  "host": "0.0.0.0",
  "port": 6001,
  "app_manager": {
    "driver": "memory",
    "array": {
      "apps": [
        {
          "id": "my-app-id",
          "key": "my-app-key",
          "secret": "my-app-secret",
          "max_connections": 1000,
          "enable_client_messages": true,
          "enabled": true,
          "max_client_events_per_second": 100
        }
      ]
    }
  }
}
```

### 3. 驗證伺服器是否啟動

```bash
curl http://localhost:6001/up
# 回應：OK
```

---

## 🏗️ 功能特色

- **Pusher Protocol v7 相容**：可直接使用現有 Pusher SDK
- **多種後端適配器**：Redis、Redis Cluster、NATS、記憶體
- **多種應用管理器**：記憶體、MySQL、PostgreSQL、DynamoDB、ScyllaDB
- **Delta 壓縮**：支援 Fossil Delta 與 xdelta3 演算法，節省 60-90% 頻寬
- **標籤過濾**：伺服器端標籤過濾，減少不必要的訊息傳輸
- **水平擴展**：支援多節點叢集部署
- **Prometheus 監控**：內建指標收集與匯出
- **Unix Socket 支援**：適用於反向代理部署

---

## 📋 配置優先順序

配置值的載入優先順序（由高到低）：

1. **環境變數** — 最高優先權
2. **配置檔案** (`--config config.json`)
3. **程式內建預設值** — 最低優先權

---

## 🔧 Cargo Feature Flags

Sockudo 使用 Cargo feature flags 控制編譯的後端：

| Feature | 說明 |
|---------|------|
| `local`（預設） | 僅包含本地/記憶體實作，無外部依賴 |
| `full` | 啟用所有後端 |
| `redis` | Redis 適配器、快取、佇列、速率限制器 |
| `redis-cluster` | Redis Cluster 支援（包含 `redis`） |
| `nats` | NATS 適配器 |
| `mysql` | MySQL 應用管理器 |
| `postgres` | PostgreSQL 應用管理器 |
| `dynamodb` | DynamoDB 應用管理器 |
| `scylla` | ScyllaDB 應用管理器 |
| `sqs` | AWS SQS 佇列 |
| `lambda` | AWS Lambda Webhook 支援 |

```bash
# 預設編譯（最快）
cargo build --release

# 僅使用 Redis
cargo build --release --no-default-features --features "local,redis"

# 全部功能
cargo build --release --features full
```
