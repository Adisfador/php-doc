# WebSockets - Real-time коммуникация

WebSocket - это протокол полнодуплексной связи поверх TCP, позволяющий двустороннюю коммуникацию между клиентом и сервером в реальном времени.

---

## Зачем нужны WebSockets?

### HTTP - однонаправленный протокол

```
Client                    Server
  |                          |
  |--------- Request ------->|
  |                          | (обработка)
  |<-------- Response -------|
  |                          |
  | (соединение закрывается) |
```

**Проблемы HTTP для real-time:**
- Клиент всегда инициирует запрос
- Сервер не может отправить данные по своей инициативе
- Новое TCP соединение для каждого запроса (overhead)
- HTTP заголовки добавляют ~1KB overhead

### Решения до WebSockets

#### 1. Polling (опрос)
```javascript
// Клиент постоянно спрашивает: "Есть обновления?"
setInterval(() => {
    fetch('/api/messages/new')
        .then(response => response.json())
        .then(data => updateUI(data));
}, 3000); // Каждые 3 секунды
```

**Минусы:**
- Много лишних запросов
- Задержка до 3 секунд
- Нагрузка на сервер

#### 2. Long Polling
```javascript
function longPoll() {
    fetch('/api/messages/wait') // Сервер держит соединение открытым
        .then(response => response.json())
        .then(data => {
            updateUI(data);
            longPoll(); // Новый запрос сразу после ответа
        });
}
```

**Минусы:**
- Все еще HTTP overhead
- Сложность на сервере

#### 3. Server-Sent Events (SSE)
```javascript
const eventSource = new EventSource('/api/stream');
eventSource.onmessage = (event) => {
    console.log(event.data);
};
```

**Минусы:**
- Только от сервера к клиенту (однонаправленный)
- HTTP/1.1 ограничение на количество соединений

### WebSocket - идеальное решение

```
Client                    Server
  |                          |
  |--- HTTP Upgrade Request ->|
  |<-- 101 Switching Protocols|
  |                          |
  |<===== WebSocket Stream ====>| Постоянное двустороннее соединение
  |                          |
  | Данные туда-сюда         |
  | без HTTP overhead        |
```

---

## Как работает WebSocket

### 1. Handshake (переход с HTTP на WebSocket)

**Клиент отправляет HTTP запрос:**
```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

**Сервер отвечает:**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**После этого:**
- HTTP соединение "апгрейдится" до WebSocket
- Используется тот же TCP сокет
- Данные передаются в бинарных фреймах, не HTTP

### 2. Data Framing

WebSocket передает данные во фреймах:

```
Frame structure:
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

**Opcodes:**
- `0x0` - continuation frame
- `0x1` - text frame
- `0x2` - binary frame
- `0x8` - connection close
- `0x9` - ping
- `0xA` - pong

---

## WebSockets в PHP

### Вариант 1: Ratchet (ReactPHP)

```php
<?php
// composer require cboden/ratchet

use Ratchet\MessageComponentInterface;
use Ratchet\ConnectionInterface;

class ChatServer implements MessageComponentInterface
{
    protected $clients;

    public function __construct()
    {
        $this->clients = new \SplObjectStorage;
    }

    public function onOpen(ConnectionInterface $conn)
    {
        // Новое подключение
        $this->clients->attach($conn);
        echo "New connection! ({$conn->resourceId})\n";
    }

    public function onMessage(ConnectionInterface $from, $msg)
    {
        // Получено сообщение
        echo "Message from {$from->resourceId}: {$msg}\n";

        // Отправить всем клиентам
        foreach ($this->clients as $client) {
            if ($from !== $client) {
                $client->send($msg);
            }
        }
    }

    public function onClose(ConnectionInterface $conn)
    {
        // Соединение закрыто
        $this->clients->detach($conn);
        echo "Connection {$conn->resourceId} has disconnected\n";
    }

    public function onError(ConnectionInterface $conn, \Exception $e)
    {
        echo "Error: {$e->getMessage()}\n";
        $conn->close();
    }
}

// Запуск сервера
$server = IoServer::factory(
    new HttpServer(
        new WsServer(
            new ChatServer()
        )
    ),
    8080
);

$server->run();
```

**Запуск:**
```bash
php websocket-server.php
```

### Вариант 2: Swoole (высокая производительность)

```php
<?php
// composer require swoole/swoole

$server = new Swoole\WebSocket\Server("0.0.0.0", 9501);

$server->on("start", function ($server) {
    echo "Swoole WebSocket Server started at ws://127.0.0.1:9501\n";
});

$server->on("open", function ($server, $request) {
    echo "Connection opened: {$request->fd}\n";
});

$server->on("message", function ($server, $frame) {
    echo "Received: {$frame->data}\n";
    
    // Отправить всем подключенным клиентам
    foreach ($server->connections as $fd) {
        if ($server->isEstablished($fd)) {
            $server->push($fd, $frame->data);
        }
    }
});

$server->on("close", function ($server, $fd) {
    echo "Connection closed: {$fd}\n";
});

$server->start();
```

---

## Laravel + WebSockets

### Laravel Reverb (Laravel 11+, официальный пакет)

```bash
php artisan reverb:install
php artisan reverb:start
```

**config/reverb.php:**
```php
return [
    'apps' => [
        [
            'id' => env('REVERB_APP_ID'),
            'key' => env('REVERB_APP_KEY'),
            'secret' => env('REVERB_APP_SECRET'),
            'options' => [
                'host' => env('REVERB_HOST', '0.0.0.0'),
                'port' => env('REVERB_PORT', 8080),
                'scheme' => env('REVERB_SCHEME', 'http'),
            ],
        ],
    ],
];
```

### Laravel Echo + Pusher/Reverb

**Broadcasting Event:**
```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class MessageSent implements ShouldBroadcast
{
    use InteractsWithSockets, SerializesModels;

    public function __construct(
        public string $message,
        public int $userId
    ) {}

    public function broadcastOn(): Channel
    {
        return new Channel('chat');
    }

    public function broadcastAs(): string
    {
        return 'message.sent';
    }

    public function broadcastWith(): array
    {
        return [
            'message' => $this->message,
            'user_id' => $this->userId,
            'timestamp' => now()->toISOString(),
        ];
    }
}
```

**Отправка события:**
```php
use App\Events\MessageSent;

// В контроллере или сервисе
broadcast(new MessageSent($message, $userId));
```

**Клиент (JavaScript):**
```javascript
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT,
    forceTLS: false,
    enabledTransports: ['ws', 'wss'],
});

// Подписка на канал
Echo.channel('chat')
    .listen('.message.sent', (event) => {
        console.log('New message:', event.message);
        addMessageToUI(event);
    });
```

---

## Авторизация WebSocket соединений

### Проблема
WebSocket соединение не имеет автоматической аутентификации через cookies/sessions как HTTP.

### Решение 1: Query параметры (небезопасно!)

```javascript
// ❌ ПЛОХО: токен в URL
const ws = new WebSocket('ws://example.com?token=abc123');
```

**Проблема:** Токен попадет в логи сервера!

### Решение 2: Авторизация при handshake

```php
<?php
// Ratchet с авторизацией

use Ratchet\ConnectionInterface;
use Ratchet\MessageComponentInterface;

class AuthenticatedChatServer implements MessageComponentInterface
{
    protected $clients;
    protected $authenticatedUsers = [];

    public function onOpen(ConnectionInterface $conn)
    {
        // Получить токен из query string
        $queryString = $conn->httpRequest->getUri()->getQuery();
        parse_str($queryString, $queryParams);
        
        $token = $queryParams['token'] ?? null;
        
        // Валидация токена
        if (!$token || !$this->validateToken($token)) {
            $conn->send(json_encode(['error' => 'Unauthorized']));
            $conn->close();
            return;
        }
        
        $userId = $this->getUserIdFromToken($token);
        $this->authenticatedUsers[$conn->resourceId] = $userId;
        $this->clients->attach($conn);
        
        echo "User {$userId} connected\n";
    }

    private function validateToken(string $token): bool
    {
        // Проверка JWT или session токена
        try {
            $decoded = JWT::decode($token, $key, ['HS256']);
            return true;
        } catch (\Exception $e) {
            return false;
        }
    }

    public function onMessage(ConnectionInterface $from, $msg)
    {
        $userId = $this->authenticatedUsers[$from->resourceId];
        
        // Теперь знаем кто отправил сообщение
        $data = json_encode([
            'user_id' => $userId,
            'message' => $msg,
            'timestamp' => time(),
        ]);
        
        foreach ($this->clients as $client) {
            $client->send($data);
        }
    }
}
```

### Решение 3: Laravel Broadcasting авторизация

**routes/channels.php:**
```php
<?php

use Illuminate\Support\Facades\Broadcast;

// Private канал - только аутентифицированные
Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
    // Проверка: имеет ли пользователь доступ к комнате
    return $user->canAccessRoom($roomId);
});

// Presence канал - кто онлайн
Broadcast::channel('presence-chat.{roomId}', function ($user, $roomId) {
    if ($user->canAccessRoom($roomId)) {
        return [
            'id' => $user->id,
            'name' => $user->name,
            'avatar' => $user->avatar_url,
        ];
    }
});
```

**JavaScript:**
```javascript
// Подписка на приватный канал (требует авторизацию)
Echo.private('chat.1')
    .listen('MessageSent', (e) => {
        console.log(e.message);
    });

// Presence канал - видно кто онлайн
Echo.join('presence-chat.1')
    .here((users) => {
        console.log('Currently online:', users);
    })
    .joining((user) => {
        console.log(user.name + ' joined');
    })
    .leaving((user) => {
        console.log(user.name + ' left');
    })
    .listen('MessageSent', (e) => {
        console.log(e.message);
    });
```

**Laravel автоматически:**
1. Проверит CSRF токен
2. Проверит сессию пользователя
3. Выполнит callback из `routes/channels.php`
4. Вернет разрешение/отказ

---

## Security Best Practices

### 1. Всегда используй WSS (WebSocket Secure)

```javascript
// ✅ ХОРОШО: зашифровано
const ws = new WebSocket('wss://example.com');

// ❌ ПЛОХО: незашифровано
const ws = new WebSocket('ws://example.com');
```

**Nginx конфиг для WSS:**
```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location /ws {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 2. Rate Limiting

```php
<?php

class RateLimitedChatServer implements MessageComponentInterface
{
    private $messageLimits = []; // [userId => [timestamp, timestamp, ...]]

    public function onMessage(ConnectionInterface $from, $msg)
    {
        $userId = $this->authenticatedUsers[$from->resourceId];
        
        // Проверка: не больше 10 сообщений в минуту
        $now = time();
        $userMessages = $this->messageLimits[$userId] ?? [];
        $userMessages = array_filter($userMessages, fn($t) => $t > $now - 60);
        
        if (count($userMessages) >= 10) {
            $from->send(json_encode([
                'error' => 'Rate limit exceeded. Max 10 messages per minute.'
            ]));
            return;
        }
        
        $userMessages[] = $now;
        $this->messageLimits[$userId] = $userMessages;
        
        // Обработка сообщения...
    }
}
```

### 3. Валидация данных

```php
public function onMessage(ConnectionInterface $from, $msg)
{
    // Всегда валидируй входящие данные!
    $data = json_decode($msg, true);
    
    if (!isset($data['type']) || !isset($data['payload'])) {
        $from->send(json_encode(['error' => 'Invalid message format']));
        return;
    }
    
    // Санитизация
    $message = htmlspecialchars($data['payload'], ENT_QUOTES, 'UTF-8');
    
    // Валидация длины
    if (strlen($message) > 1000) {
        $from->send(json_encode(['error' => 'Message too long']));
        return;
    }
    
    // Обработка...
}
```

### 4. Origin проверка

```php
public function onOpen(ConnectionInterface $conn)
{
    $origin = $conn->httpRequest->getHeader('Origin')[0] ?? '';
    
    $allowedOrigins = ['https://example.com', 'https://app.example.com'];
    
    if (!in_array($origin, $allowedOrigins)) {
        $conn->close();
        return;
    }
    
    // Продолжить...
}
```

### 5. Heartbeat/Ping-Pong

```php
<?php

class ChatServerWithHeartbeat implements MessageComponentInterface
{
    private $loop;
    private $clients;

    public function __construct(LoopInterface $loop)
    {
        $this->loop = $loop;
        $this->clients = new \SplObjectStorage;
        
        // Ping каждые 30 секунд
        $this->loop->addPeriodicTimer(30, function () {
            foreach ($this->clients as $client) {
                $client->send(json_encode(['type' => 'ping']));
            }
        });
    }

    public function onMessage(ConnectionInterface $from, $msg)
    {
        $data = json_decode($msg, true);
        
        if ($data['type'] === 'pong') {
            // Клиент жив
            $this->clients[$from] = time();
            return;
        }
        
        // Обычное сообщение...
    }
}
```

---

## Когда использовать WebSockets?

### ✅ Используй WebSockets для:
- Chat приложения
- Real-time уведомления
- Live обновления (биржевые котировки, спортивные результаты)
- Multiplayer игры
- Collaborative editing (Google Docs-подобное)
- Live dashboard / monitoring
- Video/Audio streaming signaling (WebRTC)

### ❌ НЕ используй WebSockets для:
- Обычных REST API запросов
- Загрузки файлов
- SEO-критичного контента (поисковики не индексируют WS)
- Когда достаточно Server-Sent Events (односторонние обновления)

---

## Масштабирование WebSocket серверов

### Проблема: Sticky Sessions

```
User connects -> Server 1 (WebSocket connection)
User sends message -> Load Balancer -> Server 2 (connection lost!)
```

### Решение 1: Sticky Sessions в Load Balancer

**Nginx:**
```nginx
upstream websocket {
    ip_hash; # Одинаковый IP всегда на один сервер
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

### Решение 2: Redis Pub/Sub

```php
<?php

use Predis\Client;

class DistributedChatServer implements MessageComponentInterface
{
    private $redis;
    private $clients;

    public function __construct()
    {
        $this->redis = new Client();
        $this->clients = new \SplObjectStorage;
        
        // Подписка на Redis канал
        $this->redis->subscribe(['chat-messages'], function ($redis, $channel, $message) {
            // Отправить сообщение всем клиентам этого сервера
            foreach ($this->clients as $client) {
                $client->send($message);
            }
        });
    }

    public function onMessage(ConnectionInterface $from, $msg)
    {
        // Публикация в Redis - все серверы получат
        $this->redis->publish('chat-messages', $msg);
    }
}
```

Теперь сообщение отправится ВСЕМ клиентам на ВСЕХ серверах!

---

## Debugging WebSockets

### Chrome DevTools

1. Открой DevTools → Network → WS (WebSockets)
2. Увидишь все WebSocket соединения
3. Можешь смотреть отправленные/полученные сообщения

### wscat - CLI tool

```bash
npm install -g wscat

# Подключение
wscat -c ws://localhost:8080

# Отправка сообщений
> Hello, WebSocket!

# Получение ответов
< Message received: Hello, WebSocket!
```

---

## Сравнение: HTTP vs WebSocket

| Характеристика | HTTP | WebSocket |
|---------------|------|-----------|
| Направление | Клиент → Сервер | Двустороннее |
| Overhead | ~1KB заголовков | ~2 байта фрейма |
| Latency | Высокая (новое TCP) | Низкая (постоянное) |
| Сложность | Простой | Сложнее |
| Масштабирование | Легко (stateless) | Сложнее (stateful) |
| Использование | API, веб-страницы | Real-time обновления |

**Вывод:** Используй WebSockets только когда действительно нужен real-time. Для остального - обычный HTTP.
