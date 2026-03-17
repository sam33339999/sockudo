# Sockudo API 參考文檔

> 本文檔基於 Sockudo v3.1.0 原始碼撰寫，涵蓋所有 HTTP REST API 端點與 WebSocket 協議。

---

## 目錄

- [1. HTTP REST API](#1-http-rest-api)
  - [1.1 認證機制](#11-認證機制)
  - [1.2 觸發事件](#12-觸發事件)
  - [1.3 批次觸發事件](#13-批次觸發事件)
  - [1.4 取得頻道列表](#14-取得頻道列表)
  - [1.5 取得單一頻道資訊](#15-取得單一頻道資訊)
  - [1.6 取得 Presence 頻道使用者](#16-取得-presence-頻道使用者)
  - [1.7 終止使用者連線](#17-終止使用者連線)
  - [1.8 健康檢查](#18-健康檢查)
  - [1.9 伺服器資源使用量](#19-伺服器資源使用量)
  - [1.10 Prometheus 指標](#110-prometheus-指標)
- [2. WebSocket 協議](#2-websocket-協議)
  - [2.1 建立連線](#21-建立連線)
  - [2.2 連線事件](#22-連線事件)
  - [2.3 頻道訂閱](#23-頻道訂閱)
  - [2.4 頻道類型](#24-頻道類型)
  - [2.5 客戶端事件](#25-客戶端事件)
  - [2.6 使用者認證](#26-使用者認證)
  - [2.7 Delta 壓縮事件](#27-delta-壓縮事件)
  - [2.8 錯誤處理](#28-錯誤處理)
- [3. Webhook 事件](#3-webhook-事件)
- [4. 協議常數](#4-協議常數)

---

## 1. HTTP REST API

所有 API 端點的基礎路徑為 `http://{host}:{port}`（預設 `http://localhost:6001`）。

### 1.1 認證機制

REST API 使用 **Pusher 相容的 HMAC-SHA256 簽名認證**。每個請求必須包含以下查詢參數：

| 參數 | 類型 | 說明 |
|------|------|------|
| `auth_key` | String | 應用的公開金鑰（App Key） |
| `auth_timestamp` | String | Unix 時間戳（秒） |
| `auth_version` | String | 認證版本，固定為 `"1.0"` |
| `body_md5` | String | 請求 Body 的 MD5 雜湊（POST 請求且有 Body 時必填） |
| `auth_signature` | String | HMAC-SHA256 簽名 |

#### 簽名計算方式

```
簽名字串 = "{HTTP_METHOD}\n{PATH}\n{SORTED_QUERY_STRING}"
```

其中 `SORTED_QUERY_STRING` 為**除 `auth_signature` 外**所有查詢參數，按照 key 的字母順序排列。

```
auth_signature = HMAC-SHA256(app_secret, 簽名字串).hex()
```

#### 範例

```
# 簽名字串
POST\n/apps/my-app-id/events\nauth_key=my-app-key&auth_timestamp=1700000000&auth_version=1.0&body_md5=abc123...

# 最終 URL
POST /apps/my-app-id/events?auth_key=my-app-key&auth_timestamp=1700000000&auth_version=1.0&body_md5=abc123&auth_signature=def456...
```

---

### 1.2 觸發事件

向一個或多個頻道發送事件。

```
POST /apps/{appId}/events
```

**認證**：必要（Pusher API 簽名）

**請求 Body**：

```json
{
  "name": "my-event",
  "channel": "my-channel",
  "data": "{\"message\": \"Hello\"}",
  "socket_id": "12345.67890",
  "info": "subscription_count,user_count",
  "tags": {"type": "notification", "priority": "high"},
  "delta": true
}
```

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `name` | String | ✅ | 事件名稱 |
| `channel` | String | ✅* | 目標頻道名稱（與 `channels` 二擇一） |
| `channels` | Array\<String\> | ✅* | 目標頻道列表（與 `channel` 二擇一） |
| `data` | String | ✅ | 事件資料（JSON 字串） |
| `socket_id` | String | ❌ | 排除指定 Socket 不接收此事件 |
| `info` | String | ❌ | 請求額外資訊，逗號分隔：`subscription_count`、`user_count`、`cache` |
| `tags` | Object | ❌ | 標籤（用於標籤過濾功能） |
| `delta` | bool | ❌ | `true` 強制使用 Delta 壓縮，`false` 強制發送完整訊息 |

**成功回應**（200 OK）：

```json
{"ok": true}
```

若請求了 `info` 參數：

```json
{
  "channels": {
    "my-channel": {
      "subscription_count": 15,
      "user_count": 8
    }
  }
}
```

**錯誤回應**：

| 狀態碼 | 說明 |
|--------|------|
| 400 | 無效的請求參數 |
| 401 | 認證失敗 |
| 404 | 應用不存在 |
| 413 | 酬載超過大小限制 |

---

### 1.3 批次觸發事件

一次觸發多個事件。

```
POST /apps/{appId}/batch_events
```

**認證**：必要

**請求 Body**：

```json
{
  "batch": [
    {
      "name": "event-1",
      "channel": "channel-1",
      "data": "{\"msg\": \"hello\"}"
    },
    {
      "name": "event-2",
      "channels": ["channel-2", "channel-3"],
      "data": "{\"msg\": \"world\"}",
      "info": "subscription_count"
    }
  ]
}
```

> **注意**：批次中的事件按照**順序**處理，以維持 Delta 壓縮狀態的一致性。

**成功回應**（200 OK）：

```json
{}
```

若任何事件請求了 `info`：

```json
{
  "batch": [
    {},
    {
      "channels": {
        "channel-2": {"subscription_count": 5}
      }
    }
  ]
}
```

---

### 1.4 取得頻道列表

列出所有已佔用的頻道。

```
GET /apps/{appId}/channels
```

**認證**：必要

**查詢參數**：

| 參數 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `filter_by_prefix` | String | ❌ | 依前綴過濾頻道（例：`presence-`、`private-`） |
| `info` | String | ❌ | 請求額外資訊：`subscription_count`、`user_count`、`cache` |

> **限制**：`user_count` 資訊僅在 `filter_by_prefix=presence-` 時可用。

**成功回應**（200 OK）：

```json
{
  "channels": {
    "public-news": {},
    "presence-chat": {
      "user_count": 3,
      "subscription_count": 5
    }
  }
}
```

---

### 1.5 取得單一頻道資訊

取得特定頻道的詳細資訊。

```
GET /apps/{appId}/channels/{channelName}
```

**認證**：必要

**查詢參數**：

| 參數 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `info` | String | ❌ | 請求額外資訊：`subscription_count`、`user_count`、`cache` |

**成功回應**（200 OK）：

```json
{
  "occupied": true,
  "subscription_count": 15,
  "user_count": 8,
  "cache": {
    "data": "{\"last_message\": \"hello\"}",
    "ttl": 3600
  }
}
```

| 欄位 | 類型 | 說明 |
|------|------|------|
| `occupied` | bool | 頻道是否有訂閱者 |
| `subscription_count` | u32 | 訂閱數（需透過 `info` 請求） |
| `user_count` | u32 | 使用者數（僅 Presence 頻道，需透過 `info` 請求） |
| `cache` | Object | 快取資料（需透過 `info` 請求） |

---

### 1.6 取得 Presence 頻道使用者

列出 Presence 頻道中的所有使用者。

```
GET /apps/{appId}/channels/{channelName}/users
```

**認證**：必要

> **限制**：僅適用於 `presence-` 前綴的頻道。

**成功回應**（200 OK）：

```json
{
  "users": [
    {"id": "user-123"},
    {"id": "user-456"}
  ]
}
```

---

### 1.7 終止使用者連線

斷開指定使用者的所有 WebSocket 連線。

```
POST /apps/{appId}/users/{userId}/terminate_connections
```

**認證**：必要

**請求 Body**：空

**成功回應**（200 OK）：

```json
{"ok": true}
```

---

### 1.8 健康檢查

檢查伺服器健康狀態。

```
GET /up
GET /up/{appId}
```

**認證**：不需要

**成功回應**：

| 狀態碼 | 回應文字 | 說明 |
|--------|----------|------|
| 200 | `OK` | 所有系統正常 |
| 200 | `DEGRADED` | 非關鍵系統異常（如佇列） |
| 503 | `ERROR` | 關鍵系統異常（如適配器、快取管理器） |
| 404 | `NOT_FOUND` | 應用不存在 |

**回應標頭**：`X-Health-Check: {status}`

---

### 1.9 伺服器資源使用量

取得伺服器的記憶體使用資訊。

```
GET /usage
```

**認證**：不需要

**成功回應**（200 OK）：

```json
{
  "memory": {
    "free": 1024000000,
    "used": 2048000000,
    "total": 3072000000,
    "percent": 66.67
  }
}
```

> **注意**：記憶體值的單位為**位元組（bytes）**。

---

### 1.10 Prometheus 指標

匯出 Prometheus 格式的指標資料。

```
GET /metrics
```

> **注意**：此端點在獨立的指標伺服器上運行（預設埠號 `9601`，非主伺服器埠號 `6001`）。

**認證**：不需要

**Content-Type**：`text/plain; version=0.0.4; charset=utf-8`

---

## 2. WebSocket 協議

Sockudo 實作 **Pusher Protocol v7**。

### 2.1 建立連線

透過 WebSocket 升級建立連線。

```
GET /app/{appKey}
```

**查詢參數**（可選）：

| 參數 | 類型 | 說明 |
|------|------|------|
| `protocol` | String | 協議版本 |
| `client` | String | 客戶端名稱 |
| `version` | String | 客戶端版本 |

**連線成功後**，伺服器會立即發送：

```json
{
  "event": "pusher:connection_established",
  "data": "{\"socket_id\":\"12345.67890\",\"activity_timeout\":120}"
}
```

| 欄位 | 說明 |
|------|------|
| `socket_id` | 連線的唯一識別碼（格式：`{數字}.{數字}`） |
| `activity_timeout` | 活動逾時秒數（客戶端應在此時間內發送 ping） |

---

### 2.2 連線事件

#### Ping/Pong 心跳

客戶端應定期發送 ping 以保持連線：

```json
// 客戶端 → 伺服器
{"event": "pusher:ping", "data": {}}

// 伺服器 → 客戶端
{"event": "pusher:pong", "data": {}}
```

伺服器也會自動發送 ping（可透過 `websocket.auto_ping` 配置）。客戶端必須在 **30 秒**內回應 pong。

---

### 2.3 頻道訂閱

#### 訂閱頻道

```json
// 客戶端 → 伺服器
{
  "event": "pusher:subscribe",
  "data": {
    "channel": "my-channel",
    "auth": "app-key:signature",
    "channel_data": "{\"user_id\":\"123\",\"user_info\":{\"name\":\"Alice\"}}",
    "filter": {"eq": {"type": "important"}},
    "delta": true
  }
}
```

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `channel` | String | ✅ | 頻道名稱 |
| `auth` | String | 僅私有頻道 | 認證簽名（格式：`{app_key}:{hmac_hex}`） |
| `channel_data` | String | 僅 Presence | 成員資料 JSON 字串 |
| `filter` | Object | ❌ | 標籤過濾條件 |
| `delta` | bool/String | ❌ | 啟用 Delta 壓縮 |

#### 私有頻道認證簽名

簽名字串格式：

```
# 私有頻道
{socket_id}:{channel_name}

# Presence 頻道（含 channel_data）
{socket_id}:{channel_name}:{channel_data}
```

簽名 = `HMAC-SHA256(app_secret, 簽名字串).hex()`

客戶端傳送格式 = `"{app_key}:{簽名}"`

#### 訂閱成功回應

**公開/私有頻道**：

```json
{
  "event": "pusher_internal:subscription_succeeded",
  "channel": "my-channel",
  "data": "{}"
}
```

**Presence 頻道**：

```json
{
  "event": "pusher_internal:subscription_succeeded",
  "channel": "presence-room",
  "data": "{\"presence\":{\"ids\":[\"user-1\",\"user-2\"],\"hash\":{\"user-1\":{\"name\":\"Alice\"},\"user-2\":null},\"count\":2}}"
}
```

#### 取消訂閱

```json
// 客戶端 → 伺服器
{
  "event": "pusher:unsubscribe",
  "data": {
    "channel": "my-channel"
  }
}
```

---

### 2.4 頻道類型

| 類型 | 前綴 | 認證 | Presence | 快取 | 說明 |
|------|------|------|----------|------|------|
| 公開頻道 | 無 | ❌ | ❌ | ❌ | 任何人可訂閱 |
| 私有頻道 | `private-` | ✅ | ❌ | ❌ | 需認證才能訂閱 |
| Presence 頻道 | `presence-` | ✅ | ✅ | ❌ | 追蹤在線成員 |
| 加密私有頻道 | `private-encrypted-` | ✅ | ❌ | ❌ | 端對端加密 |
| 快取頻道 | 可配置 | ❌ | ❌ | ✅ | 保留最後一則事件 |

**頻道名稱限制**：
- 最大長度：200 字元
- 允許字元：`a-zA-Z0-9_\-=@,.;`

#### Presence 頻道事件

成員加入：

```json
{
  "event": "pusher_internal:member_added",
  "channel": "presence-room",
  "data": "{\"user_id\":\"new-user\",\"user_info\":{\"name\":\"Bob\"}}"
}
```

成員離開：

```json
{
  "event": "pusher_internal:member_removed",
  "channel": "presence-room",
  "data": "{\"user_id\":\"departed-user\"}"
}
```

#### 快取頻道

訂閱快取頻道時，如果有快取資料，伺服器會自動發送最後一則事件。如果沒有快取：

```json
{
  "event": "pusher:cache_miss",
  "channel": "cache-channel",
  "data": "{\"channel\":\"cache-channel\"}"
}
```

---

### 2.5 客戶端事件

在私有或 Presence 頻道上，客戶端可以直接向其他客戶端發送事件：

```json
// 客戶端 → 伺服器 → 其他客戶端
{
  "event": "client-typing",
  "channel": "private-chat",
  "data": "{\"user\":\"Alice\"}"
}
```

**規則**：
- 事件名稱**必須**以 `client-` 前綴開頭
- 僅在**私有頻道**或 **Presence 頻道**上可用
- 需啟用 `enable_client_messages` 配置
- 發送者不會收到自己的事件
- 受速率限制控制

---

### 2.6 使用者認證

啟用 `enable_user_authentication` 後，客戶端可進行使用者級別認證：

```json
// 客戶端 → 伺服器
{
  "event": "pusher:signin",
  "data": {
    "user_data": "{\"user_id\":\"123\",\"user_info\":{\"name\":\"Alice\"}}",
    "auth": "app-key:signature"
  }
}

// 伺服器 → 客戶端（成功）
{
  "event": "pusher:signin_success",
  "data": "{\"user_id\":\"123\"}"
}
```

---

### 2.7 Delta 壓縮事件

#### 啟用 Delta 壓縮

連線後全域啟用：

```json
// 客戶端 → 伺服器
{
  "event": "pusher:enable_delta_compression",
  "data": {"algorithm": "fossil"}
}

// 伺服器 → 客戶端
{
  "event": "pusher:delta_compression_enabled",
  "data": "{\"algorithm\":\"fossil\"}"
}
```

#### Delta 訊息

當啟用 Delta 壓縮後，伺服器可能發送差異訊息：

```json
{
  "event": "pusher:delta",
  "channel": "ticker",
  "data": "{\"sequence\":124,\"data\":\"base64_encoded_patch\",\"conflation_key\":\"BTC\"}"
}
```

#### 快取同步

當 Delta 狀態不一致時：

```json
{
  "event": "pusher:delta_cache_sync",
  "channel": "ticker",
  "data": "{\"channel\":\"ticker\",\"version\":123}"
}
```

---

### 2.8 錯誤處理

伺服器在發生錯誤時會發送：

```json
{
  "event": "pusher:error",
  "data": "{\"code\":4100,\"message\":\"Buffer full, reconnect with backoff\"}"
}
```

常見錯誤碼：

| 錯誤碼 | 說明 |
|--------|------|
| 4001 | 應用不存在 |
| 4009 | 連線已建立但應用被停用 |
| 4100 | 緩衝區滿（需重新連線） |
| 4200 | Pong 回應逾時 |
| 4301 | 不支援的協議版本 |

---

## 3. Webhook 事件

### 傳遞方式

Sockudo 使用 HTTP POST 將事件通知傳送到配置的 Webhook 端點。

**HTTP 標頭**：

| 標頭 | 說明 |
|------|------|
| `Content-Type` | `application/json` |
| `X-Pusher-Key` | 應用的公開金鑰 |
| `X-Pusher-Signature` | 請求 Body 的 HMAC-SHA256 簽名（十六進位） |

**簽名驗證**：

```
expected = HMAC-SHA256(app_secret, request_body).hex()
valid = (X-Pusher-Signature == expected)
```

### 酬載格式

```json
{
  "time_ms": 1700000000000,
  "events": [
    {
      "name": "channel_occupied",
      "channel": "my-channel"
    }
  ]
}
```

### 事件類型

| 事件名稱 | 觸發時機 | 資料欄位 |
|----------|----------|----------|
| `channel_occupied` | 第一個 Socket 訂閱頻道 | `name`、`channel` |
| `channel_vacated` | 最後一個 Socket 取消訂閱頻道 | `name`、`channel` |
| `member_added` | 使用者的第一個 Socket 加入 Presence 頻道 | `name`、`channel`、`user_id`、`data` |
| `member_removed` | 使用者的最後一個 Socket 離開 Presence 頻道 | `name`、`channel`、`user_id` |

### 傳遞保證

- **至少一次**（At-Least-Once）：請在接收端實作冪等性
- 支援批次處理（透過 `webhooks.batching` 配置）
- 失敗時會重試

---

## 4. 協議常數

| 常數 | 值 | 說明 |
|------|-----|------|
| 協議版本 | `7` | Pusher Protocol 版本 |
| 活動逾時 | `120` 秒 | 可透過配置覆寫 |
| Pong 逾時 | `30` 秒 | 固定值 |
| 頻道名稱最大長度 | `200` 字元 | 可透過配置覆寫 |
| 事件名稱最大長度 | `200` 字元 | 可透過配置覆寫 |
| 保留事件前綴 | `pusher:`、`pusher_internal:` | 系統保留，不可由客戶端使用 |
| 客戶端事件前綴 | `client-` | 客戶端發送事件的必要前綴 |
