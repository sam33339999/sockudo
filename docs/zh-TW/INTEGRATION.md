# Sockudo 整合指南

> 本文檔說明如何在 Golang、PHP、Laravel 及 Rust 中整合 Sockudo。
> Sockudo 實作 Pusher Protocol v7，因此可以直接使用 **Pusher 官方 SDK** 進行整合。

---

## 目錄

- [前置條件](#前置條件)
- [1. Golang 整合](#1-golang-整合)
- [2. PHP 整合](#2-php-整合)
- [3. Laravel 整合](#3-laravel-整合)
  - [3.1 Laravel Private / Presence Channel 深入教學](#31-laravel-private--presence-channel-深入教學)
- [4. Rust 整合](#4-rust-整合)
- [5. 認證端點實作](#5-認證端點實作)
- [6. 前端客戶端（所有語言通用）](#6-前端客戶端所有語言通用)

---

## 前置條件

在開始之前，請確認你已有一個運行中的 Sockudo 伺服器，且知道以下設定：

| 項目 | 範例值 | 說明 |
|------|--------|------|
| Sockudo 主機 | `localhost` | 伺服器位址 |
| Sockudo 埠號 | `6001` | 伺服器埠號 |
| App ID | `my-app-id` | 應用 ID |
| App Key | `my-app-key` | 應用公開金鑰 |
| App Secret | `my-app-secret` | 應用私密金鑰（僅後端使用，不可暴露） |

**重要原則**：
- **後端**使用 Pusher **伺服器端 SDK** 觸發事件、驗證頻道認證
- **前端**使用 Pusher **客戶端 SDK**（如 `pusher-js`）建立 WebSocket 連線
- 將 SDK 的 `host` 與 `port` 指向你的 Sockudo 伺服器（而非 Pusher 官方服務）

---

## 1. Golang 整合

### 安裝 Pusher Go SDK

```bash
go get github.com/pusher/pusher-http-go/v5
```

### 初始化客戶端

```go
package main

import (
    "fmt"
    "log"

    pusher "github.com/pusher/pusher-http-go/v5"
)

func main() {
    // 初始化 Pusher 客戶端，指向 Sockudo 伺服器
    client := pusher.Client{
        AppID:   "my-app-id",
        Key:     "my-app-key",
        Secret:  "my-app-secret",
        Host:    "localhost:6001",  // Sockudo 主機:埠號
        Secure:  false,             // 使用 HTTP（true 則使用 HTTPS）
    }

    // 觸發事件到頻道
    data := map[string]string{
        "message": "Hello from Go!",
    }
    err := client.Trigger("my-channel", "my-event", data)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("事件已觸發")
}
```

### 觸發事件到多個頻道

```go
// 同時發送到多個頻道（最多 100 個）
err := client.Trigger([]string{"channel-1", "channel-2"}, "my-event", data)
```

### 批次觸發事件

```go
events := []pusher.Event{
    {Channel: "channel-1", Name: "event-1", Data: map[string]string{"msg": "hello"}},
    {Channel: "channel-2", Name: "event-2", Data: map[string]string{"msg": "world"}},
}
err := client.TriggerBatch(events)
```

### 查詢頻道資訊

```go
// 取得頻道列表
channels, err := client.Channels(pusher.ChannelsParams{
    FilterByPrefix: "presence-",
    Info:           []string{"user_count"},
})

// 取得單一頻道資訊
channel, err := client.Channel("my-channel", pusher.ChannelParams{
    Info: []string{"subscription_count"},
})

// 取得 Presence 頻道使用者
users, err := client.GetChannelUsers("presence-room")
```

### 私有頻道認證（HTTP 處理器）

```go
package main

import (
    "net/http"

    pusher "github.com/pusher/pusher-http-go/v5"
)

var client = pusher.Client{
    AppID:  "my-app-id",
    Key:    "my-app-key",
    Secret: "my-app-secret",
    Host:   "localhost:6001",
    Secure: false,
}

func authHandler(w http.ResponseWriter, r *http.Request) {
    params, _ := io.ReadAll(r.Body)

    // 在此處加入你的權限驗證邏輯
    // 例如：檢查使用者是否有權限訂閱此頻道

    response, err := client.AuthorizePrivateChannel(params)
    if err != nil {
        w.WriteHeader(http.StatusForbidden)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(response)
}

func presenceAuthHandler(w http.ResponseWriter, r *http.Request) {
    params, _ := io.ReadAll(r.Body)

    // Presence 頻道需要提供成員資料
    presenceData := pusher.MemberData{
        UserID: "user-123",
        UserInfo: map[string]string{
            "name": "Alice",
        },
    }

    response, err := client.AuthorizePresenceChannel(params, presenceData)
    if err != nil {
        w.WriteHeader(http.StatusForbidden)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(response)
}

func main() {
    http.HandleFunc("/pusher/auth", authHandler)
    http.ListenAndServe(":8080", nil)
}
```

### Webhook 接收與驗證

```go
func webhookHandler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)

    webhook, err := client.Webhook(r.Header, body)
    if err != nil {
        // 簽名驗證失敗
        w.WriteHeader(http.StatusUnauthorized)
        return
    }

    for _, event := range webhook.Events {
        switch event.Name {
        case "channel_occupied":
            fmt.Printf("頻道 %s 已被佔用\n", event.Channel)
        case "channel_vacated":
            fmt.Printf("頻道 %s 已空閒\n", event.Channel)
        case "member_added":
            fmt.Printf("成員 %s 加入 %s\n", event.UserID, event.Channel)
        case "member_removed":
            fmt.Printf("成員 %s 離開 %s\n", event.UserID, event.Channel)
        }
    }

    w.WriteHeader(http.StatusOK)
}
```

---

## 2. PHP 整合

### 安裝 Pusher PHP SDK

```bash
composer require pusher/pusher-php-server
```

### 初始化客戶端

```php
<?php
require __DIR__ . '/vendor/autoload.php';

$pusher = new Pusher\Pusher(
    'my-app-key',       // App Key
    'my-app-secret',    // App Secret
    'my-app-id',        // App ID
    [
        'host'      => 'localhost',  // Sockudo 主機
        'port'      => 6001,         // Sockudo 埠號
        'scheme'    => 'http',       // 使用 HTTP（或 'https'）
        'useTLS'    => false,        // 不使用 TLS（與 scheme 對應）
    ]
);
```

### 觸發事件

```php
<?php
// 觸發事件到單一頻道
$pusher->trigger('my-channel', 'my-event', [
    'message' => 'Hello from PHP!'
]);

// 觸發事件到多個頻道
$pusher->trigger(['channel-1', 'channel-2'], 'my-event', [
    'message' => 'Hello!'
]);

// 排除特定 Socket（避免發送者收到自己的事件）
$pusher->trigger('my-channel', 'my-event', $data, [
    'socket_id' => '12345.67890'
]);
```

### 批次觸發事件

```php
<?php
$batch = [
    ['channel' => 'channel-1', 'name' => 'event-1', 'data' => ['msg' => 'hello']],
    ['channel' => 'channel-2', 'name' => 'event-2', 'data' => ['msg' => 'world']],
];
$pusher->triggerBatch($batch);
```

### 查詢頻道資訊

```php
<?php
// 取得頻道列表
$channels = $pusher->getChannels([
    'filter_by_prefix' => 'presence-',
    'info' => 'user_count'
]);

// 取得單一頻道資訊
$channel = $pusher->getChannelInfo('my-channel', [
    'info' => 'subscription_count'
]);

// 取得 Presence 頻道使用者
$users = $pusher->getPresenceUsers('presence-room');
```

### 私有頻道認證

```php
<?php
// 處理 /pusher/auth 端點
$socketId = $_POST['socket_id'];
$channelName = $_POST['channel_name'];

// 在此處加入你的權限驗證邏輯
// 例如：檢查 session、JWT token 等

if (strpos($channelName, 'presence-') === 0) {
    // Presence 頻道認證
    $userData = [
        'user_id' => '123',
        'user_info' => [
            'name' => 'Alice',
        ],
    ];
    $auth = $pusher->authorizePresenceChannel($channelName, $socketId, '123', $userData['user_info']);
} else {
    // 私有頻道認證
    $auth = $pusher->authorizeChannel($channelName, $socketId);
}

header('Content-Type: application/json');
echo json_encode($auth);
```

### Webhook 接收與驗證

```php
<?php
$headers = getallheaders();
$body = file_get_contents('php://input');

$webhook = $pusher->webhook($headers, $body);

foreach ($webhook->get_events() as $event) {
    switch ($event->name) {
        case 'channel_occupied':
            error_log("頻道 {$event->channel} 已被佔用");
            break;
        case 'channel_vacated':
            error_log("頻道 {$event->channel} 已空閒");
            break;
        case 'member_added':
            error_log("成員 {$event->user_id} 加入 {$event->channel}");
            break;
        case 'member_removed':
            error_log("成員 {$event->user_id} 離開 {$event->channel}");
            break;
    }
}

http_response_code(200);
```

---

## 3. Laravel 整合

Laravel 內建了 Broadcasting 功能，可以直接整合 Sockudo。

### 步驟 1：安裝依賴

```bash
composer require pusher/pusher-php-server
```

### 步驟 2：設定環境變數

在 `.env` 檔案中設定：

```env
BROADCAST_DRIVER=pusher

PUSHER_APP_ID=my-app-id
PUSHER_APP_KEY=my-app-key
PUSHER_APP_SECRET=my-app-secret
PUSHER_HOST=localhost
PUSHER_PORT=6001
PUSHER_SCHEME=http
PUSHER_APP_CLUSTER=mt1
```

### 步驟 3：設定 Broadcasting

編輯 `config/broadcasting.php`：

```php
'pusher' => [
    'driver' => 'pusher',
    'key' => env('PUSHER_APP_KEY'),
    'secret' => env('PUSHER_APP_SECRET'),
    'app_id' => env('PUSHER_APP_ID'),
    'options' => [
        'host' => env('PUSHER_HOST', 'localhost'),
        'port' => env('PUSHER_PORT', 6001),
        'scheme' => env('PUSHER_SCHEME', 'http'),
        'useTLS' => env('PUSHER_SCHEME', 'http') === 'https',
        'encrypted' => false,
    ],
],
```

### 步驟 4：建立 Event 類別

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class NewMessage implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public string $message;
    public string $userName;

    public function __construct(string $message, string $userName)
    {
        $this->message = $message;
        $this->userName = $userName;
    }

    /**
     * 定義事件廣播的頻道
     */
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('chat.room.1'),
        ];
    }

    /**
     * 自訂事件名稱（可選）
     */
    public function broadcastAs(): string
    {
        return 'message.new';
    }

    /**
     * 自訂廣播資料（可選）
     */
    public function broadcastWith(): array
    {
        return [
            'message' => $this->message,
            'user' => $this->userName,
            'timestamp' => now()->toISOString(),
        ];
    }
}
```

### 步驟 5：定義頻道授權

編輯 `routes/channels.php`：

```php
<?php

use Illuminate\Support\Facades\Broadcast;

// 私有頻道授權
Broadcast::channel('chat.room.{roomId}', function ($user, $roomId) {
    // 檢查使用者是否有權限存取此聊天室
    return $user->canAccessRoom($roomId);
});

// Presence 頻道授權
Broadcast::channel('presence.room.{roomId}', function ($user, $roomId) {
    if ($user->canAccessRoom($roomId)) {
        // 返回使用者資料（會作為 channel_data）
        return [
            'id' => $user->id,
            'name' => $user->name,
            'avatar' => $user->avatar_url,
        ];
    }
    return false;
});
```

### 步驟 6：觸發事件

```php
<?php

namespace App\Http\Controllers;

use App\Events\NewMessage;

class ChatController extends Controller
{
    public function sendMessage(Request $request)
    {
        $request->validate([
            'message' => 'required|string|max:1000',
            'room_id' => 'required|integer',
        ]);

        // 儲存訊息到資料庫
        $message = Message::create([
            'user_id' => auth()->id(),
            'room_id' => $request->room_id,
            'content' => $request->message,
        ]);

        // 廣播事件到 Sockudo
        broadcast(new NewMessage(
            $request->message,
            auth()->user()->name
        ))->toOthers();  // toOthers() 排除發送者

        return response()->json(['status' => 'sent']);
    }
}
```

### 步驟 7：前端設定（Laravel Echo）

安裝前端依賴：

```bash
npm install --save laravel-echo pusher-js
```

設定 `resources/js/echo.js`（或 `bootstrap.js`）：

```javascript
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    wsHost: import.meta.env.VITE_PUSHER_HOST ?? 'localhost',
    wsPort: import.meta.env.VITE_PUSHER_PORT ?? 6001,
    wssPort: import.meta.env.VITE_PUSHER_PORT ?? 6001,
    forceTLS: false,
    disableStats: true,
    enabledTransports: ['ws', 'wss'],
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER ?? 'mt1',
});
```

在 `.env` 中加入前端變數：

```env
VITE_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
VITE_PUSHER_HOST="${PUSHER_HOST}"
VITE_PUSHER_PORT="${PUSHER_PORT}"
VITE_PUSHER_SCHEME="${PUSHER_SCHEME}"
VITE_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```

### 步驟 8：前端監聽事件

```javascript
// 監聽私有頻道
Echo.private('chat.room.1')
    .listen('.message.new', (e) => {
        console.log('新訊息:', e.message, '來自:', e.user);
    });

// 監聽 Presence 頻道
Echo.join('presence.room.1')
    .here((users) => {
        console.log('目前在線:', users);
    })
    .joining((user) => {
        console.log('使用者加入:', user);
    })
    .leaving((user) => {
        console.log('使用者離開:', user);
    })
    .listen('.message.new', (e) => {
        console.log('新訊息:', e);
    });
```

---

### 3.1 Laravel Private / Presence Channel 深入教學

本節深入說明 Laravel 如何與 Sockudo 配合實現**私有頻道**（Private Channel）與 **Presence 頻道**的完整流程，包括認證原理、實際情境範例與疑難排解。

#### 認證原理：Sockudo 如何驗證頻道訂閱

當前端透過 `pusher-js` 訂閱 `private-*` 或 `presence-*` 頻道時，會觸發以下流程：

```
前端 (pusher-js)                  Laravel 後端                     Sockudo
      |                                |                              |
      |-- subscribe('private-xxx') --> |                              |
      |                                |                              |
      |-- POST /broadcasting/auth ---> |                              |
      |   {socket_id, channel_name}    |                              |
      |                                |-- 驗證使用者權限 ------------> |
      |                                |-- 計算 HMAC-SHA256 簽名 ----> |
      |                                |                              |
      | <---- {auth: "key:sig"} -------|                              |
      |                                                               |
      |-- pusher:subscribe {channel, auth} -------------------------> |
      |                                                  驗證簽名：    |
      |                                     expected = HMAC-SHA256(   |
      |                                       app_secret,            |
      |                                       "{socket_id}:{channel}"|
      |                                     )                         |
      |                                     比對 auth 中的 sig        |
      | <-------- subscription_succeeded -----------------------------|
```

**簽名格式**（基於 Sockudo 原始碼 `src/channel/manager.rs` 與 `src/token.rs`）：

| 頻道類型 | 簽名字串格式 | 回傳格式 |
|----------|-------------|----------|
| Private | `{socket_id}:{channel_name}` | `{"auth": "{app_key}:{hmac_hex}"}` |
| Presence | `{socket_id}:{channel_name}:{channel_data}` | `{"auth": "{app_key}:{hmac_hex}", "channel_data": "..."}` |
| Private Encrypted | `{socket_id}:{channel_name}` | `{"auth": "{app_key}:{hmac_hex}", "shared_secret": "..."}` |

其中 `hmac_hex = HMAC-SHA256(app_secret, 簽名字串)` 以十六進位編碼輸出，使用 timing-safe 比對驗證。

---

#### 範例 1：Private Channel — 使用者專屬通知

**情境**：每個使用者有自己的通知頻道 `private-user.{id}`，只有本人可訂閱。

**步驟 A：定義 Event**

```php
<?php

namespace App\Events;

use App\Models\User;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Queue\SerializesModels;

class UserNotification implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public int $userId,
        public string $title,
        public string $body,
        public string $type = 'info',  // info, warning, error
    ) {}

    /**
     * 廣播到該使用者的私有頻道
     * Laravel 會自動加上 "private-" 前綴
     */
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("user.{$this->userId}"),
        ];
    }

    public function broadcastAs(): string
    {
        return 'notification';
    }

    public function broadcastWith(): array
    {
        return [
            'title' => $this->title,
            'body' => $this->body,
            'type' => $this->type,
            'created_at' => now()->toISOString(),
        ];
    }
}
```

**步驟 B：設定頻道授權**

在 `routes/channels.php` 中：

```php
<?php

use Illuminate\Support\Facades\Broadcast;

/*
 * Private Channel 授權
 *
 * 回呼函式接收目前已登入的 $user 和路由參數。
 * 返回 true 表示允許訂閱，返回 false 表示拒絕。
 *
 * 當 Laravel 收到 POST /broadcasting/auth 請求時：
 * 1. 透過 session/token 驗證使用者身份
 * 2. 呼叫此回呼函式檢查權限
 * 3. 如果允許 → 用 app_secret 計算 HMAC-SHA256 簽名回傳
 * 4. 如果拒絕 → 回傳 403 Forbidden
 */
Broadcast::channel('user.{userId}', function ($user, $userId) {
    // 只允許使用者訂閱自己的頻道
    return (int) $user->id === (int) $userId;
});
```

**步驟 C：觸發事件**

```php
// 在 Controller 或 Service 中
use App\Events\UserNotification;

// 發送通知給特定使用者
event(new UserNotification(
    userId: $user->id,
    title: '訂單已出貨',
    body: '您的訂單 #12345 已出貨，預計明天到達。',
    type: 'info',
));
```

**步驟 D：前端監聽**

```javascript
// 使用 Laravel Echo
// Echo.private() 會自動：
// 1. 呼叫 POST /broadcasting/auth 取得簽名
// 2. 用簽名向 Sockudo 訂閱 private-user.{id}
Echo.private(`user.${userId}`)
    .listen('.notification', (e) => {
        showToast(e.title, e.body, e.type);
    });
```

---

#### 範例 2：Private Channel — 聊天室

**情境**：兩個使用者之間的一對一私聊，頻道名稱為 `private-chat.{roomId}`。

**步驟 A：定義 Event**

```php
<?php

namespace App\Events;

use App\Models\Message;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Queue\SerializesModels;

class ChatMessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public Message $message,
    ) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("chat.{$this->message->room_id}"),
        ];
    }

    public function broadcastAs(): string
    {
        return 'message.sent';
    }

    public function broadcastWith(): array
    {
        return [
            'id' => $this->message->id,
            'user_id' => $this->message->user_id,
            'user_name' => $this->message->user->name,
            'content' => $this->message->content,
            'created_at' => $this->message->created_at->toISOString(),
        ];
    }
}
```

**步驟 B：頻道授權 — 檢查使用者是否為聊天室成員**

```php
// routes/channels.php

Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
    // 查詢使用者是否為此聊天室的成員
    return \App\Models\ChatRoom::where('id', $roomId)
        ->whereHas('members', function ($query) use ($user) {
            $query->where('user_id', $user->id);
        })
        ->exists();
});
```

**步驟 C：Controller**

```php
<?php

namespace App\Http\Controllers;

use App\Events\ChatMessageSent;
use App\Models\ChatRoom;
use App\Models\Message;
use Illuminate\Http\Request;

class ChatController extends Controller
{
    public function sendMessage(Request $request, ChatRoom $room)
    {
        // 確認使用者是成員（Policy 或 middleware）
        $this->authorize('sendMessage', $room);

        $request->validate([
            'content' => 'required|string|max:5000',
        ]);

        $message = Message::create([
            'room_id' => $room->id,
            'user_id' => auth()->id(),
            'content' => $request->content,
        ]);

        $message->load('user');

        // 廣播給聊天室中的其他人
        broadcast(new ChatMessageSent($message))->toOthers();

        return response()->json($message);
    }
}
```

**步驟 D：前端**

```javascript
// 訂閱聊天室頻道
const chatChannel = Echo.private(`chat.${roomId}`);

// 監聽新訊息
chatChannel.listen('.message.sent', (e) => {
    appendMessage(e);
});

// 使用 client event 顯示「對方正在輸入」
// 注意：需在 Sockudo 的 App 配置中設定 enable_client_messages: true
chatChannel.whisper('typing', { user: currentUser.name });

// 監聽對方的輸入狀態
chatChannel.listenForWhisper('typing', (e) => {
    showTypingIndicator(e.user);
});
```

> **注意**：`whisper()` 是 Laravel Echo 對 Pusher client event 的封裝。它會發送 `client-typing` 事件。這需要 Sockudo 的 App 配置中 `enable_client_messages: true`。

---

#### 範例 3：Presence Channel — 在線使用者列表

**情境**：顯示聊天室中誰在線上，即時更新上線/離線狀態。

**步驟 A：定義 Event（可選，Presence 頻道本身就有成員事件）**

Presence 頻道的成員加入/離開事件由 Sockudo 自動管理，**不需要額外定義 Event 類別**。但你仍可以在 Presence 頻道上廣播自訂事件：

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Queue\SerializesModels;

class UserStatusChanged implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public int $roomId,
        public int $userId,
        public string $status,  // 'online', 'away', 'busy'
    ) {}

    public function broadcastOn(): array
    {
        return [
            new PresenceChannel("room.{$this->roomId}"),
        ];
    }

    public function broadcastAs(): string
    {
        return 'status.changed';
    }
}
```

**步驟 B：頻道授權 — 返回使用者資訊**

Presence 頻道的授權回呼與 Private 不同：**返回陣列**（使用者資料）表示授權成功，返回 `false` 表示拒絕。

```php
// routes/channels.php

/*
 * Presence Channel 授權
 *
 * 返回值的差異：
 * - Private Channel: 返回 true/false
 * - Presence Channel: 返回 array（使用者資料）或 false
 *
 * 返回的陣列會成為 channel_data 中的 user_info，
 * Laravel 會自動加入 user_id 欄位。
 *
 * Sockudo 收到訂閱時，簽名字串格式為：
 *   "{socket_id}:presence-room.{roomId}:{channel_data_json}"
 * 其中 channel_data_json = {"user_id":"123","user_info":{"name":"Alice",...}}
 */
Broadcast::channel('room.{roomId}', function ($user, $roomId) {
    if ($user->canAccessRoom($roomId)) {
        // 返回的資料會傳給所有頻道中的其他成員
        // 這些資料可以在前端透過 member.info 取得
        return [
            'id' => $user->id,
            'name' => $user->name,
            'avatar' => $user->avatar_url,
            'role' => $user->getRoleInRoom($roomId),
        ];
    }

    return false;  // 拒絕訂閱
});
```

**步驟 C：前端 — 完整 Presence 頻道使用**

```javascript
const presenceChannel = Echo.join(`room.${roomId}`);

// ===== 成員管理事件（由 Sockudo 自動觸發） =====

// 1. 訂閱成功 — 取得目前所有在線成員
presenceChannel.here((members) => {
    // members 是一個陣列，包含所有在線成員的資料
    // 對應 Sockudo 回傳的 pusher_internal:subscription_succeeded
    // data.presence.hash 中的資料
    console.log('在線成員:', members);
    // [
    //   { id: 1, name: 'Alice', avatar: '...', role: 'admin' },
    //   { id: 2, name: 'Bob', avatar: '...', role: 'member' },
    // ]
    updateOnlineUserList(members);
});

// 2. 新成員加入 — 對應 Sockudo 的 pusher_internal:member_added
presenceChannel.joining((member) => {
    console.log('加入:', member);
    // { id: 3, name: 'Carol', avatar: '...', role: 'member' }
    addToOnlineList(member);
    showNotification(`${member.name} 已上線`);
});

// 3. 成員離開 — 對應 Sockudo 的 pusher_internal:member_removed
presenceChannel.leaving((member) => {
    console.log('離開:', member);
    removeFromOnlineList(member);
    showNotification(`${member.name} 已離線`);
});

// ===== 自訂事件（由後端 broadcast() 觸發） =====
presenceChannel.listen('.status.changed', (e) => {
    updateUserStatus(e.userId, e.status);
});

// ===== Client Event — 打字指示器 =====
// whisper 不經過後端，直接由 Sockudo 轉發給其他成員
presenceChannel.whisper('typing', {
    user: currentUser.name,
});

presenceChannel.listenForWhisper('typing', (e) => {
    showTypingIndicator(e.user);
});
```

---

#### 範例 4：Private Encrypted Channel — 端對端加密

**情境**：需要端對端加密的敏感通訊，即使 Sockudo 伺服器也無法讀取訊息內容。

> **注意**：加密頻道使用 `private-encrypted-` 前綴。Sockudo 會自動偵測此前綴並停用 Delta 壓縮（因為加密後的資料無法有效壓縮）。

```php
// Laravel Event — 使用加密頻道
public function broadcastOn(): array
{
    // 頻道名稱加上 'private-encrypted-' 前綴
    // Laravel 的 EncryptedPrivateChannel 會自動處理
    return [
        new \Illuminate\Broadcasting\EncryptedPrivateChannel("medical.{$this->patientId}"),
    ];
}
```

```php
// routes/channels.php
Broadcast::channel('medical.{patientId}', function ($user, $patientId) {
    // 只有患者本人或其醫生可訂閱
    return $user->id === (int) $patientId
        || $user->isDoctorOf($patientId);
});
```

```javascript
// 前端 — 加密頻道的用法與普通私有頻道相同
// pusher-js 會自動處理加密/解密
Echo.encryptedPrivate(`medical.${patientId}`)
    .listen('.record.updated', (e) => {
        console.log('病歷更新:', e);
    });
```

---

#### 疑難排解

**問題 1：訂閱私有頻道時出現 403 Forbidden**

```
原因：Laravel 的認證端點拒絕了請求
排查步驟：
1. 確認使用者已登入（session 或 token 認證有效）
2. 確認 routes/channels.php 中的回呼邏輯正確
3. 檢查 Laravel 的 CSRF 保護（API 路由可能不需要）
```

解決方案 — 確認認證路由已正確註冊：

```php
// app/Providers/BroadcastServiceProvider.php
// Laravel 11+ 使用 bootstrap/app.php

use Illuminate\Support\Facades\Broadcast;

// 確認這行已啟用（預設在 BroadcastServiceProvider 中）
Broadcast::routes(['middleware' => ['web']]);

// 如果前端使用 API token（如 Sanctum），改為：
Broadcast::routes(['middleware' => ['auth:sanctum']]);
```

**問題 2：Sockudo 回傳 `Auth signature mismatch`**

```
原因：Laravel 計算的簽名與 Sockudo 預期的不一致
排查步驟：
1. 確認 .env 中的 PUSHER_APP_KEY 和 PUSHER_APP_SECRET 
   與 Sockudo 配置中的 App key/secret 完全一致
2. 確認沒有多餘的空格或換行符號
3. 確認 Sockudo 配置中的 App 已啟用（enabled: true）
```

**問題 3：前端 `Echo.private()` 沒有觸發認證請求**

```
原因：Laravel Echo 的認證端點配置不正確
排查步驟：
1. 確認 Echo 配置中的 authEndpoint 或 channelAuthorization.endpoint 正確
2. 開啟瀏覽器開發者工具 Network 頁面，查看是否有 POST 請求到 /broadcasting/auth
3. 如果使用 Sanctum，確認 CSRF cookie 已取得
```

前端認證端點設定：

```javascript
// Laravel Echo 預設使用 /broadcasting/auth
// 如果需要自訂認證端點：
window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    wsHost: import.meta.env.VITE_PUSHER_HOST ?? 'localhost',
    wsPort: import.meta.env.VITE_PUSHER_PORT ?? 6001,
    forceTLS: false,
    disableStats: true,
    enabledTransports: ['ws', 'wss'],
    cluster: 'mt1',

    // 自訂認證設定
    channelAuthorization: {
        endpoint: '/broadcasting/auth',
        transport: 'ajax',
        headers: {
            // Laravel Sanctum SPA 認證
            'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]')?.content,
            // 或使用 Bearer Token
            // 'Authorization': `Bearer ${token}`,
        },
    },
});
```

**問題 4：Presence 頻道的 `here()` 收到空陣列**

```
原因：channels.php 中的回呼返回了 true 而非陣列
解決：Presence 頻道的回呼必須返回一個陣列（使用者資料），而非 boolean。
```

```php
// ❌ 錯誤 — 這是 Private Channel 的寫法
Broadcast::channel('room.{roomId}', function ($user, $roomId) {
    return true;  // 不會包含使用者資料
});

// ✅ 正確 — Presence Channel 必須返回陣列
Broadcast::channel('room.{roomId}', function ($user, $roomId) {
    return [
        'id' => $user->id,
        'name' => $user->name,
    ];
});
```

---

#### 頻道類型快速對照表

| 特性 | Public | Private | Presence | Private Encrypted |
|------|--------|---------|----------|-------------------|
| Laravel 類別 | `Channel` | `PrivateChannel` | `PresenceChannel` | `EncryptedPrivateChannel` |
| 頻道前綴 | 無 | `private-` | `presence-` | `private-encrypted-` |
| 需要認證 | ❌ | ✅ | ✅ | ✅ |
| 成員追蹤 | ❌ | ❌ | ✅ | ❌ |
| Client Event | ❌ | ✅ | ✅ | ❌ |
| 端對端加密 | ❌ | ❌ | ❌ | ✅ |
| channels.php 返回值 | — | `true`/`false` | `array`/`false` | `true`/`false` |
| Echo 訂閱方法 | `Echo.channel()` | `Echo.private()` | `Echo.join()` | `Echo.encryptedPrivate()` |
| Sockudo 簽名字串 | — | `{sid}:{ch}` | `{sid}:{ch}:{data}` | `{sid}:{ch}` |
| Delta 壓縮 | ✅ | ✅ | ✅ | ❌ 自動停用 |

---

## 4. Rust 整合

Rust 目前沒有官方的 Pusher 伺服器端 SDK，但可以直接呼叫 Sockudo 的 HTTP API。

### 方法 1：使用 HTTP API 直接呼叫

#### 依賴

在 `Cargo.toml` 中加入：

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
hmac = "0.12"
sha2 = "0.10"
hex = "0.4"
md5 = "0.7"
```

#### 實作 Pusher API 客戶端

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;
use std::collections::BTreeMap;
use std::time::{SystemTime, UNIX_EPOCH};

type HmacSha256 = Hmac<Sha256>;

pub struct SockudoClient {
    app_id: String,
    key: String,
    secret: String,
    host: String,
    port: u16,
    secure: bool,
    http_client: reqwest::Client,
}

impl SockudoClient {
    pub fn new(app_id: &str, key: &str, secret: &str, host: &str, port: u16) -> Self {
        Self {
            app_id: app_id.to_string(),
            key: key.to_string(),
            secret: secret.to_string(),
            host: host.to_string(),
            port,
            secure: false,
            http_client: reqwest::Client::new(),
        }
    }

    fn base_url(&self) -> String {
        let scheme = if self.secure { "https" } else { "http" };
        format!("{}://{}:{}", scheme, self.host, self.port)
    }

    fn sign(&self, method: &str, path: &str, params: &BTreeMap<String, String>) -> String {
        let query_string: String = params
            .iter()
            .map(|(k, v)| format!("{}={}", k, v))
            .collect::<Vec<_>>()
            .join("&");

        let string_to_sign = format!("{}\n{}\n{}", method, path, query_string);

        let mut mac = HmacSha256::new_from_slice(self.secret.as_bytes())
            .expect("HMAC can take key of any size");
        mac.update(string_to_sign.as_bytes());
        hex::encode(mac.finalize().into_bytes())
    }

    /// 觸發事件
    pub async fn trigger(
        &self,
        channel: &str,
        event: &str,
        data: &serde_json::Value,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let path = format!("/apps/{}/events", self.app_id);

        let body = serde_json::json!({
            "name": event,
            "channel": channel,
            "data": serde_json::to_string(data)?
        });

        let body_string = serde_json::to_string(&body)?;
        let body_md5 = format!("{:x}", md5::compute(&body_string));

        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)?
            .as_secs()
            .to_string();

        let mut params = BTreeMap::new();
        params.insert("auth_key".to_string(), self.key.clone());
        params.insert("auth_timestamp".to_string(), timestamp);
        params.insert("auth_version".to_string(), "1.0".to_string());
        params.insert("body_md5".to_string(), body_md5);

        let signature = self.sign("POST", &path, &params);
        params.insert("auth_signature".to_string(), signature);

        let url = format!("{}{}", self.base_url(), path);

        let response = self
            .http_client
            .post(&url)
            .query(&params)
            .header("Content-Type", "application/json")
            .body(body_string)
            .send()
            .await?;

        if !response.status().is_success() {
            let status = response.status();
            let text = response.text().await?;
            return Err(format!("API error {}: {}", status, text).into());
        }

        Ok(())
    }

    /// 產生私有頻道認證簽名
    pub fn authorize_channel(&self, socket_id: &str, channel: &str) -> serde_json::Value {
        let string_to_sign = format!("{}:{}", socket_id, channel);

        let mut mac = HmacSha256::new_from_slice(self.secret.as_bytes())
            .expect("HMAC can take key of any size");
        mac.update(string_to_sign.as_bytes());
        let signature = hex::encode(mac.finalize().into_bytes());

        serde_json::json!({
            "auth": format!("{}:{}", self.key, signature)
        })
    }

    /// 產生 Presence 頻道認證簽名
    pub fn authorize_presence_channel(
        &self,
        socket_id: &str,
        channel: &str,
        user_id: &str,
        user_info: &serde_json::Value,
    ) -> serde_json::Value {
        let channel_data = serde_json::json!({
            "user_id": user_id,
            "user_info": user_info
        });
        let channel_data_str = serde_json::to_string(&channel_data).unwrap();
        let string_to_sign = format!("{}:{}:{}", socket_id, channel, channel_data_str);

        let mut mac = HmacSha256::new_from_slice(self.secret.as_bytes())
            .expect("HMAC can take key of any size");
        mac.update(string_to_sign.as_bytes());
        let signature = hex::encode(mac.finalize().into_bytes());

        serde_json::json!({
            "auth": format!("{}:{}", self.key, signature),
            "channel_data": channel_data_str
        })
    }
}
```

#### 使用範例

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = SockudoClient::new(
        "my-app-id",
        "my-app-key",
        "my-app-secret",
        "localhost",
        6001,
    );

    // 觸發事件
    let data = serde_json::json!({
        "message": "Hello from Rust!",
        "timestamp": "2024-01-01T00:00:00Z"
    });
    client.trigger("my-channel", "my-event", &data).await?;
    println!("事件已觸發");

    Ok(())
}
```

### 方法 2：使用 WebSocket 客戶端

若需要在 Rust 中接收即時事件，可使用 WebSocket 客戶端：

```toml
[dependencies]
tokio-tungstenite = "0.24"
futures-util = "0.3"
url = "2"
```

```rust
use futures_util::{SinkExt, StreamExt};
use tokio_tungstenite::connect_async;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "ws://localhost:6001/app/my-app-key";

    let (ws_stream, _) = connect_async(url).await?;
    println!("WebSocket 已連線");

    let (mut write, mut read) = ws_stream.split();

    // 接收連線確認
    if let Some(msg) = read.next().await {
        let msg = msg?;
        println!("收到: {}", msg);
        // 解析 pusher:connection_established 取得 socket_id
    }

    // 訂閱公開頻道
    let subscribe_msg = serde_json::json!({
        "event": "pusher:subscribe",
        "data": {
            "channel": "my-channel"
        }
    });
    write
        .send(tokio_tungstenite::tungstenite::Message::Text(
            subscribe_msg.to_string(),
        ))
        .await?;

    // 持續接收事件
    while let Some(msg) = read.next().await {
        let msg = msg?;
        println!("收到事件: {}", msg);
    }

    Ok(())
}
```

---

## 5. 認證端點實作

無論使用哪種語言，你都需要在後端提供一個**認證端點**供客戶端在訂閱私有/Presence 頻道時呼叫。

### 認證流程

```
1. 客戶端嘗試訂閱 private-xxx 或 presence-xxx 頻道
2. Pusher SDK 自動呼叫你的認證端點（預設 POST /pusher/auth）
3. 你的後端驗證使用者權限
4. 你的後端計算 HMAC-SHA256 簽名並回傳
5. Pusher SDK 使用此簽名完成訂閱
```

### 簽名計算

**私有頻道**：

```
string_to_sign = "{socket_id}:{channel_name}"
signature = HMAC-SHA256(app_secret, string_to_sign).hex()
response = {"auth": "{app_key}:{signature}"}
```

**Presence 頻道**：

```
channel_data = JSON.stringify({"user_id": "...", "user_info": {...}})
string_to_sign = "{socket_id}:{channel_name}:{channel_data}"
signature = HMAC-SHA256(app_secret, string_to_sign).hex()
response = {"auth": "{app_key}:{signature}", "channel_data": channel_data}
```

---

## 6. 前端客戶端（所有語言通用）

無論後端使用哪種語言，前端都使用相同的 `pusher-js` SDK。

### 安裝

```bash
npm install pusher-js
```

### 基本設定

```javascript
import Pusher from 'pusher-js';

const pusher = new Pusher('my-app-key', {
    wsHost: 'localhost',
    wsPort: 6001,
    wssPort: 6001,
    forceTLS: false,
    disableStats: true,
    enabledTransports: ['ws', 'wss'],
    cluster: 'mt1',

    // 認證端點（訂閱私有/Presence 頻道時使用）
    channelAuthorization: {
        endpoint: '/pusher/auth',  // 你的後端認證端點
        transport: 'ajax',
        headers: {
            // 如有需要，加入認證標頭
            'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]')?.content,
        },
    },
});
```

### 訂閱頻道

```javascript
// 公開頻道（無需認證）
const publicChannel = pusher.subscribe('news');
publicChannel.bind('breaking-news', (data) => {
    console.log('突發新聞:', data);
});

// 私有頻道（需認證）
const privateChannel = pusher.subscribe('private-user.123');
privateChannel.bind('notification', (data) => {
    console.log('通知:', data);
});

// Presence 頻道（需認證 + 成員追蹤）
const presenceChannel = pusher.subscribe('presence-room.1');
presenceChannel.bind('pusher:subscription_succeeded', (members) => {
    console.log('在線成員數:', members.count);
    members.each((member) => {
        console.log('成員:', member.id, member.info);
    });
});
presenceChannel.bind('pusher:member_added', (member) => {
    console.log('新成員:', member.id);
});
presenceChannel.bind('pusher:member_removed', (member) => {
    console.log('成員離開:', member.id);
});
```

### 發送客戶端事件

```javascript
// 在私有或 Presence 頻道上發送客戶端事件
const channel = pusher.subscribe('private-chat');
channel.trigger('client-typing', { user: 'Alice' });
```

### 連線狀態監控

```javascript
pusher.connection.bind('state_change', (states) => {
    console.log('連線狀態:', states.previous, '->', states.current);
});

pusher.connection.bind('connected', () => {
    console.log('已連線，Socket ID:', pusher.connection.socket_id);
});

pusher.connection.bind('disconnected', () => {
    console.log('連線已斷開');
});

pusher.connection.bind('error', (err) => {
    console.error('連線錯誤:', err);
});
```
