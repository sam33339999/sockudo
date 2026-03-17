# Sockudo 時序圖

> 本文檔使用 [Mermaid](https://mermaid.js.org/) 語法繪製時序圖，描述 Sockudo 的核心流程。

---

## 目錄

- [1. WebSocket 連線建立](#1-websocket-連線建立)
- [2. 公開頻道訂閱](#2-公開頻道訂閱)
- [3. 私有頻道認證與訂閱](#3-私有頻道認證與訂閱)
- [4. Presence 頻道訂閱](#4-presence-頻道訂閱)
- [5. 透過 HTTP API 觸發事件](#5-透過-http-api-觸發事件)
- [6. 客戶端事件傳遞](#6-客戶端事件傳遞)
- [7. Webhook 通知流程](#7-webhook-通知流程)
- [8. Delta 壓縮流程](#8-delta-壓縮流程)
- [9. 標籤過濾流程](#9-標籤過濾流程)
- [10. 多節點水平擴展（Redis 適配器）](#10-多節點水平擴展redis-適配器)
- [11. 使用者認證流程](#11-使用者認證流程)
- [12. 完整應用整合流程（以 Laravel 為例）](#12-完整應用整合流程以-laravel-為例)

---

## 1. WebSocket 連線建立

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant S as Sockudo 伺服器

    C->>S: GET /app/{appKey} (WebSocket 升級請求)
    Note over S: 驗證 appKey 是否有效
    Note over S: 驗證 Origin 是否在允許清單中
    Note over S: 檢查速率限制
    Note over S: 檢查連線數是否超過上限

    alt 驗證成功
        S->>C: HTTP 101 Switching Protocols
        S->>C: pusher:connection_established
        Note over C: data: {"socket_id":"12345.67890","activity_timeout":120}
    else appKey 無效
        S->>C: pusher:error (code: 4001)
        S->>C: 關閉連線
    else 速率限制
        S->>C: HTTP 429 Too Many Requests
    end

    loop 心跳 (每 activity_timeout 秒)
        C->>S: pusher:ping
        S->>C: pusher:pong
    end
```

---

## 2. 公開頻道訂閱

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant S as Sockudo 伺服器

    Note over C,S: 連線已建立

    C->>S: pusher:subscribe {"channel":"news"}
    Note over S: 頻道名稱不含 private- 或 presence- 前綴
    Note over S: 無需認證，直接訂閱
    S->>C: pusher_internal:subscription_succeeded {"channel":"news","data":"{}"}

    Note over S: 若此為頻道第一個訂閱者
    Note over S: 觸發 channel_occupied Webhook
```

---

## 3. 私有頻道認證與訂閱

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant A as 應用後端伺服器
    participant S as Sockudo 伺服器

    Note over C,S: 連線已建立，取得 socket_id

    C->>A: POST /pusher/auth {"socket_id":"12345.67890","channel_name":"private-chat"}
    Note over A: 檢查使用者權限
    Note over A: 計算簽名 = HMAC-SHA256(app_secret, "12345.67890:private-chat")
    A->>C: {"auth":"app-key:a1b2c3d4..."}

    C->>S: pusher:subscribe {"channel":"private-chat","auth":"app-key:a1b2c3d4..."}
    Note over S: 驗證簽名
    Note over S: expected = HMAC-SHA256(app_secret, "12345.67890:private-chat")
    Note over S: 比對簽名（timing-safe comparison）

    alt 簽名有效
        S->>C: pusher_internal:subscription_succeeded
    else 簽名無效
        S->>C: pusher:error {"code":null,"message":"Auth signature mismatch"}
    end
```

---

## 4. Presence 頻道訂閱

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant A as 應用後端伺服器
    participant S as Sockudo 伺服器
    participant O as 其他訂閱者

    Note over C,S: 連線已建立，取得 socket_id

    C->>A: POST /pusher/auth {"socket_id":"12345.67890","channel_name":"presence-room"}
    Note over A: 驗證使用者身份
    Note over A: channel_data = {"user_id":"u1","user_info":{"name":"Alice"}}
    Note over A: 簽名字串 = "12345.67890:presence-room:{channel_data}"
    Note over A: 簽名 = HMAC-SHA256(app_secret, 簽名字串)
    A->>C: {"auth":"app-key:sig...","channel_data":"{\"user_id\":\"u1\",...}"}

    C->>S: pusher:subscribe {"channel":"presence-room","auth":"app-key:sig...","channel_data":"..."}
    Note over S: 驗證簽名（含 channel_data）
    Note over S: 將使用者加入 Presence 頻道

    S->>C: pusher_internal:subscription_succeeded
    Note over C: data.presence = {"ids":["u1","u2"],"hash":{...},"count":2}

    S->>O: pusher_internal:member_added
    Note over O: data = {"user_id":"u1","user_info":{"name":"Alice"}}

    Note over C: 當 Alice 離開時
    C->>S: pusher:unsubscribe {"channel":"presence-room"}
    S->>O: pusher_internal:member_removed {"user_id":"u1"}
```

---

## 5. 透過 HTTP API 觸發事件

```mermaid
sequenceDiagram
    participant B as 應用後端
    participant S as Sockudo 伺服器
    participant C1 as 客戶端 1
    participant C2 as 客戶端 2

    Note over B: 產生事件（例如：新訊息）
    Note over B: 計算 body_md5 = MD5(request_body)
    Note over B: 組合簽名字串 = "POST\n/apps/{id}/events\n{sorted_query}"
    Note over B: auth_signature = HMAC-SHA256(app_secret, 簽名字串)

    B->>S: POST /apps/{appId}/events
    Note over B,S: Body: {"name":"new-message","channel":"chat","data":"{...}"}
    Note over B,S: Query: auth_key, auth_timestamp, auth_version, body_md5, auth_signature

    Note over S: 驗證 auth_signature
    Note over S: 驗證 auth_timestamp 是否在有效範圍內
    Note over S: 查詢 App 配置
    Note over S: 檢查事件大小限制

    S->>B: 200 OK {"ok": true}

    par 廣播給訂閱者
        S->>C1: {"event":"new-message","channel":"chat","data":"{...}"}
        S->>C2: {"event":"new-message","channel":"chat","data":"{...}"}
    end
```

---

## 6. 客戶端事件傳遞

```mermaid
sequenceDiagram
    participant C1 as 客戶端 1 (Alice)
    participant S as Sockudo 伺服器
    participant C2 as 客戶端 2 (Bob)
    participant C3 as 客戶端 3 (Carol)

    Note over C1,C3: 三個客戶端都已訂閱 private-chat

    C1->>S: {"event":"client-typing","channel":"private-chat","data":"{\"user\":\"Alice\"}"}

    Note over S: 驗證事件名稱以 "client-" 開頭
    Note over S: 驗證頻道為 private- 或 presence- 頻道
    Note over S: 驗證 enable_client_messages = true
    Note over S: 檢查客戶端事件速率限制

    par 廣播（排除發送者）
        S->>C2: {"event":"client-typing","channel":"private-chat","data":"{\"user\":\"Alice\"}"}
        S->>C3: {"event":"client-typing","channel":"private-chat","data":"{\"user\":\"Alice\"}"}
    end
    Note over C1: 發送者不會收到自己的事件
```

---

## 7. Webhook 通知流程

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant S as Sockudo 伺服器
    participant Q as 佇列（Memory/Redis/SQS）
    participant W as Webhook 端點

    C->>S: pusher:subscribe {"channel":"orders"}
    Note over S: 此為頻道的第一個訂閱者

    S->>Q: 入佇列 channel_occupied 事件
    Note over Q: 批次收集（duration: 50ms, size: 100）

    Q->>S: 批次處理就緒

    Note over S: 組合 Webhook 酬載
    Note over S: body = {"time_ms":...,"events":[{"name":"channel_occupied","channel":"orders"}]}
    Note over S: signature = HMAC-SHA256(app_secret, body)

    S->>W: POST /webhook
    Note over S,W: Headers: X-Pusher-Key, X-Pusher-Signature
    Note over S,W: Body: {"time_ms":...,"events":[...]}

    W->>S: 200 OK

    Note over W: 驗證簽名
    Note over W: expected = HMAC-SHA256(app_secret, request_body)
    Note over W: 比對 X-Pusher-Signature == expected
```

---

## 8. Delta 壓縮流程

```mermaid
sequenceDiagram
    participant B as 應用後端
    participant S as Sockudo 伺服器
    participant C as 客戶端

    Note over C: 已啟用 Delta 壓縮

    B->>S: POST /events {"channel":"ticker","data":"{\"price\":100,\"vol\":500}"}
    Note over S: 訊息 #1：完整訊息（無先前狀態）
    S->>C: {"event":"update","channel":"ticker","data":"{\"price\":100,\"vol\":500}"}
    Note over S: 儲存狀態：sequence=1

    B->>S: POST /events {"channel":"ticker","data":"{\"price\":101,\"vol\":510}"}
    Note over S: 訊息 #2：計算差異
    Note over S: delta = fossil_delta(prev_data, new_data)
    Note over S: delta 大小 < 完整訊息？使用 delta
    S->>C: pusher:delta {"sequence":2,"data":"base64_patch","channel":"ticker"}
    Note over C: 套用 delta patch 還原完整訊息

    Note over S: 每 full_message_interval 則（預設 10）
    Note over S: 強制發送完整訊息以重新同步

    B->>S: POST /events (第 10 則訊息)
    S->>C: {"event":"update","channel":"ticker","data":"{完整資料}"}
    Note over C: 重置 Delta 狀態
```

---

## 9. 標籤過濾流程

```mermaid
sequenceDiagram
    participant B as 應用後端
    participant S as Sockudo 伺服器
    participant C1 as 客戶端 1 (訂閱 type=goal)
    participant C2 as 客戶端 2 (訂閱全部)

    C1->>S: pusher:subscribe {"channel":"match:123","filter":{"eq":{"type":"goal"}}}
    S->>C1: pusher_internal:subscription_succeeded

    C2->>S: pusher:subscribe {"channel":"match:123"}
    S->>C2: pusher_internal:subscription_succeeded

    B->>S: POST /events {"channel":"match:123","data":"{...}","tags":{"type":"shot"}}
    Note over S: 評估每個訂閱者的過濾條件
    Note over S: C1 過濾條件：type == "goal" → "shot" ≠ "goal" → 不符合
    Note over S: C2 無過濾條件 → 符合
    S->>C2: {"event":"update","channel":"match:123","data":"{...}"}
    Note over C1: 未收到（不符合過濾條件）

    B->>S: POST /events {"channel":"match:123","data":"{...}","tags":{"type":"goal"}}
    Note over S: C1 過濾條件：type == "goal" → 符合
    Note over S: C2 無過濾條件 → 符合
    par
        S->>C1: {"event":"update","channel":"match:123","data":"{...}"}
        S->>C2: {"event":"update","channel":"match:123","data":"{...}"}
    end
```

---

## 10. 多節點水平擴展（Redis 適配器）

```mermaid
sequenceDiagram
    participant C1 as 客戶端 1
    participant N1 as Sockudo 節點 1
    participant R as Redis (Pub/Sub)
    participant N2 as Sockudo 節點 2
    participant C2 as 客戶端 2

    Note over C1,N1: C1 連線到節點 1
    Note over C2,N2: C2 連線到節點 2
    Note over C1,C2: 兩者都訂閱 "news" 頻道

    N1->>R: SUBSCRIBE sockudo_adapter:news
    N2->>R: SUBSCRIBE sockudo_adapter:news

    Note over N1: 收到來自後端的事件

    N1->>C1: 直接傳送給本地客戶端
    N1->>R: PUBLISH sockudo_adapter:news {event_data}

    R->>N2: 收到訊息
    N2->>C2: 傳送給本地客戶端

    Note over N1,N2: 叢集健康監控
    loop 每 heartbeat_interval_ms
        N1->>R: 發送心跳
        N2->>R: 發送心跳
    end
    Note over R: 節點逾時未心跳 → 標記為離線
```

---

## 11. 使用者認證流程

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant A as 應用後端伺服器
    participant S as Sockudo 伺服器

    Note over C,S: WebSocket 連線已建立

    C->>A: 請求使用者認證令牌
    Note over A: 驗證使用者身份
    Note over A: user_data = {"user_id":"u1","user_info":{"name":"Alice"}}
    Note over A: 簽名 = HMAC-SHA256(app_secret, JSON(user_data))
    A->>C: {"user_data":"{...}","auth":"app-key:signature"}

    C->>S: pusher:signin {"user_data":"{...}","auth":"app-key:signature"}
    Note over S: 驗證簽名
    Note over S: 解析 user_data 取得 user_id

    alt 驗證成功
        S->>C: pusher:signin_success {"user_id":"u1"}
        Note over S: 關聯 socket 與 user_id
    else 驗證失敗
        S->>C: pusher:error
    end
```

---

## 12. 完整應用整合流程（以 Laravel 為例）

```mermaid
sequenceDiagram
    participant U as 使用者瀏覽器
    participant L as Laravel 後端
    participant S as Sockudo 伺服器
    participant WS as WebSocket 連線

    Note over U: 頁面載入，初始化 Pusher SDK

    U->>S: GET /app/{appKey} (WebSocket 連線)
    S->>U: pusher:connection_established {"socket_id":"..."}

    Note over U: 訂閱私有頻道

    U->>L: POST /broadcasting/auth {"socket_id":"...","channel_name":"private-user.1"}
    Note over L: 驗證使用者身份（Laravel Auth）
    Note over L: 計算 HMAC-SHA256 簽名
    L->>U: {"auth":"app-key:signature"}

    U->>S: pusher:subscribe {"channel":"private-user.1","auth":"..."}
    S->>U: pusher_internal:subscription_succeeded

    Note over L: 使用者操作觸發事件
    U->>L: POST /api/send-message {"message":"Hello"}
    Note over L: 處理業務邏輯
    Note over L: 使用 Pusher SDK 觸發事件

    L->>S: POST /apps/{id}/events
    Note over L,S: {"name":"new-message","channel":"private-user.1","data":"{...}"}

    S->>L: 200 OK

    S->>WS: {"event":"new-message","channel":"private-user.1","data":"{...}"}
    WS->>U: 即時更新頁面
```
