# gRPC

Полный разбор gRPC: Protocol Buffers, PHP gRPC, performance, vs REST/GraphQL.

---

## 🎯 Что такое gRPC?

**gRPC** = **g**RPC **R**emote **P**rocedure **C**all - framework от Google для RPC.

**Ключевые особенности:**
- **Protocol Buffers** (protobuf) - бинарный формат вместо JSON
- **HTTP/2** - multiplexing, streaming
- **Strongly Typed** - строгая типизация
- **Code Generation** - автогенерация клиента и сервера
- **Performance** - в 5-10 раз быстрее REST

**Use Cases:**
- Microservices (внутреннее взаимодействие)
- High-performance APIs
- Real-time bidirectional streaming
- Mobile → Backend (экономия батареи)

---

## 📦 Protocol Buffers (protobuf)

### Что это?

**Protocol Buffers** - язык описания структуры данных от Google.

**Vs JSON:**

| Feature | JSON | Protobuf |
|---------|------|----------|
| **Format** | Text | Binary |
| **Size** | ~100 bytes | ~30 bytes (3x меньше) |
| **Parse Speed** | Slow | Fast (5-10x) |
| **Schema** | Нет | Да (.proto файлы) |
| **Human Readable** | Да | Нет |

### Синтаксис .proto

**user.proto:**

```protobuf
syntax = "proto3";

package user;

// Message = struct/class
message User {
  int32 id = 1;           // 1 = field number (не значение!)
  string name = 2;
  string email = 3;
  repeated Post posts = 4; // repeated = array
}

message Post {
  int32 id = 1;
  string title = 2;
  string content = 3;
  int32 user_id = 4;
}

// Service = API endpoints
service UserService {
  // Unary RPC (обычный запрос-ответ)
  rpc GetUser(GetUserRequest) returns (User);
  
  // Server streaming (сервер шлет поток данных)
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client streaming (клиент шлет поток данных)
  rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
  
  // Bidirectional streaming (двусторонний поток)
  rpc Chat(stream Message) returns (stream Message);
}

message GetUserRequest {
  int32 id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 per_page = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUsersResponse {
  int32 count = 1;
}

message Message {
  string text = 1;
  int64 timestamp = 2;
}
```

### Типы данных

**Scalar Types:**

```protobuf
int32, int64       // Целые числа
uint32, uint64     // Беззнаковые
sint32, sint64     // Знаковые (для отрицательных чисел)
fixed32, fixed64   // Фиксированный размер (быстрее для больших чисел)
float, double      // Числа с плавающей точкой
bool               // Boolean
string             // UTF-8 или ASCII
bytes              // Произвольные байты
```

**Complex Types:**

```protobuf
message Address {
  string street = 1;
  string city = 2;
  string country = 3;
}

message User {
  int32 id = 1;
  string name = 2;
  Address address = 3;        // Nested message
  repeated string tags = 4;   // Array
  map<string, string> metadata = 5; // Map/Dict
}

enum Status {
  UNKNOWN = 0; // Первое значение ВСЕГДА 0
  ACTIVE = 1;
  INACTIVE = 2;
}

message Post {
  Status status = 1;
}
```

### Опциональные поля (proto3)

```protobuf
syntax = "proto3";

message User {
  int32 id = 1;
  string name = 2;
  optional string email = 3; // Может отсутствовать
}
```

### Значения по умолчанию

```protobuf
// proto3 автоматически использует default values:
int32 -> 0
string -> ""
bool -> false
repeated -> []
```

---

## 🔧 PHP gRPC

### Установка

```bash
# 1. Установить protoc compiler
# macOS
brew install protobuf

# Ubuntu
sudo apt-get install -y protobuf-compiler

# 2. Установить PHP расширение grpc
pecl install grpc

# 3. Composer пакеты
composer require grpc/grpc
composer require google/protobuf
```

**php.ini:**
```ini
extension=grpc.so
```

### Генерация кода

**Структура:**

```
project/
├── proto/
│   └── user.proto
├── generated/  ← автогенерированный код
└── src/
    ├── Server.php
    └── Client.php
```

**Команда генерации:**

```bash
protoc --proto_path=proto \
  --php_out=generated \
  --grpc_out=generated \
  --plugin=protoc-gen-grpc=$(which grpc_php_plugin) \
  proto/user.proto
```

**Генерируется:**

```
generated/
├── User/
│   ├── User.php
│   ├── Post.php
│   ├── GetUserRequest.php
│   └── UserServiceClient.php  ← Client
└── GPBMetadata/
    └── User.php
```

---

## 🖥️ gRPC Server (PHP)

**ВАЖНО:** PHP не поддерживает native gRPC server. Варианты:
1. **RoadRunner** - рекомендуется ✅
2. **Spiral Framework** - built-in gRPC
3. **Go/Node.js server** + PHP handlers

### RoadRunner gRPC Server

**Установка:**

```bash
composer require spiral/roadrunner-grpc
vendor/bin/rr get-binary

# .rr.yaml
grpc:
  listen: tcp://127.0.0.1:9001
  proto:
    - "proto/user.proto"

server:
  command: "php worker.php"
```

**worker.php:**

```php
<?php

use Spiral\Goridge\StreamRelay;
use Spiral\RoadRunner\Worker;
use Spiral\RoadRunner\GRPC\Server;
use User\UserServiceInterface;

require __DIR__ . '/vendor/autoload.php';

$worker = Worker::create();
$server = new Server($worker);

// Register service
$server->registerService(UserServiceInterface::class, new UserService());

$server->serve();
```

**UserService.php:**

```php
<?php

namespace App\Services;

use User\GetUserRequest;
use User\User;
use User\UserServiceInterface;
use Spiral\RoadRunner\GRPC\ContextInterface;

class UserService implements UserServiceInterface
{
    public function GetUser(ContextInterface $ctx, GetUserRequest $request): User
    {
        // Получить из БД
        $userData = \App\Models\User::find($request->getId());
        
        if (!$userData) {
            throw new \Exception('User not found');
        }
        
        // Создать protobuf объект
        $user = new User();
        $user->setId($userData->id);
        $user->setName($userData->name);
        $user->setEmail($userData->email);
        
        return $user;
    }
    
    public function ListUsers(ContextInterface $ctx, ListUsersRequest $request)
    {
        $users = \App\Models\User::paginate(
            $request->getPerPage(),
            ['*'],
            'page',
            $request->getPage()
        );
        
        foreach ($users as $userData) {
            $user = new User();
            $user->setId($userData->id);
            $user->setName($userData->name);
            $user->setEmail($userData->email);
            
            // Streaming response
            yield $user;
        }
    }
}
```

**Запуск:**

```bash
./rr serve
```

---

## 📱 gRPC Client (PHP)

**client.php:**

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use User\GetUserRequest;
use User\UserServiceClient;
use Grpc\ChannelCredentials;

// Создать клиент
$client = new UserServiceClient(
    '127.0.0.1:9001',
    [
        'credentials' => ChannelCredentials::createInsecure(),
    ]
);

// Простой запрос (Unary)
$request = new GetUserRequest();
$request->setId(1);

list($user, $status) = $client->GetUser($request)->wait();

if ($status->code !== \Grpc\STATUS_OK) {
    echo "Error: " . $status->details . PHP_EOL;
    exit(1);
}

echo "User: " . $user->getName() . " (" . $user->getEmail() . ")" . PHP_EOL;

// Server streaming
$request = new ListUsersRequest();
$request->setPage(1);
$request->setPerPage(10);

$stream = $client->ListUsers($request);

foreach ($stream->responses() as $user) {
    echo "User: " . $user->getName() . PHP_EOL;
}

$stream->waitForStatus();
```

---

## 🔄 Типы RPC

### 1. Unary RPC (обычный запрос-ответ)

**Как REST.**

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}
```

**Server:**

```php
public function GetUser(ContextInterface $ctx, GetUserRequest $request): User
{
    $user = new User();
    // ... заполнить данными
    return $user;
}
```

**Client:**

```php
$request = new GetUserRequest();
$request->setId(1);

list($user, $status) = $client->GetUser($request)->wait();
```

### 2. Server Streaming

**Сервер отправляет поток данных клиенту.**

```protobuf
service UserService {
  rpc ListUsers(ListUsersRequest) returns (stream User);
}
```

**Server:**

```php
public function ListUsers(ContextInterface $ctx, ListUsersRequest $request)
{
    $users = User::all();
    
    foreach ($users as $userData) {
        $user = new User();
        $user->setId($userData->id);
        $user->setName($userData->name);
        
        yield $user; // Streaming
    }
}
```

**Client:**

```php
$stream = $client->ListUsers($request);

foreach ($stream->responses() as $user) {
    echo $user->getName() . PHP_EOL;
}
```

### 3. Client Streaming

**Клиент отправляет поток данных серверу.**

```protobuf
service UserService {
  rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
}
```

**Client:**

```php
$call = $client->CreateUsers();

for ($i = 0; $i < 10; $i++) {
    $request = new CreateUserRequest();
    $request->setName("User $i");
    $request->setEmail("user$i@example.com");
    
    $call->write($request);
}

list($response, $status) = $call->wait();
echo "Created: " . $response->getCount() . " users" . PHP_EOL;
```

### 4. Bidirectional Streaming

**Двусторонний поток (чат).**

```protobuf
service ChatService {
  rpc Chat(stream Message) returns (stream Message);
}
```

**Use Case:** Real-time chat, gaming, live updates.

---

## 🔐 Authentication & Security

### Metadata (Headers)

**Client:**

```php
$client = new UserServiceClient('127.0.0.1:9001', [
    'credentials' => ChannelCredentials::createInsecure(),
]);

$metadata = [
    'authorization' => ['Bearer ' . $token],
    'user-id' => ['123'],
];

list($user, $status) = $client->GetUser($request, $metadata)->wait();
```

**Server:**

```php
public function GetUser(ContextInterface $ctx, GetUserRequest $request): User
{
    $metadata = $ctx->getValue('authorization');
    
    if (!$metadata || !$this->validateToken($metadata[0])) {
        throw new \Exception('Unauthorized');
    }
    
    // ...
}
```

### TLS (SSL)

**Server:**

```yaml
# .rr.yaml
grpc:
  listen: tcp://127.0.0.1:9001
  tls:
    key: /path/to/server.key
    cert: /path/to/server.crt
```

**Client:**

```php
$client = new UserServiceClient('127.0.0.1:9001', [
    'credentials' => ChannelCredentials::createSsl(
        file_get_contents('/path/to/ca.crt')
    ),
]);
```

---

## ⚡ Performance

### Benchmarks (REST vs gRPC)

**Payload: 1000 пользователей**

| Protocol | Size | Parse Time | Total Time |
|----------|------|------------|------------|
| REST (JSON) | 100 KB | 50 ms | 200 ms |
| gRPC (protobuf) | 30 KB | 5 ms | 50 ms |

**gRPC ~4x быстрее REST.**

### Причины:

1. **Binary format** - protobuf меньше и быстрее JSON
2. **HTTP/2** - multiplexing (несколько запросов в одном соединении)
3. **Schema** - no validation overhead (уже проверено на compile time)

### Оптимизация

**1. Connection Pooling:**

```php
// Переиспользуй клиент
$client = new UserServiceClient('127.0.0.1:9001', [
    'credentials' => ChannelCredentials::createInsecure(),
]);

// НЕ создавай новый клиент для каждого запроса
```

**2. Streaming для больших данных:**

```protobuf
// ❌ Плохо - весь список сразу
rpc GetAllUsers(Empty) returns (UserList);

// ✅ Хорошо - streaming
rpc ListUsers(ListUsersRequest) returns (stream User);
```

**3. Compression:**

```php
$client = new UserServiceClient('127.0.0.1:9001', [
    'credentials' => ChannelCredentials::createInsecure(),
    'grpc.default_compression_algorithm' => 2, // GZIP
]);
```

---

## 🆚 gRPC vs REST vs GraphQL

| Feature | REST | GraphQL | gRPC |
|---------|------|---------|------|
| **Format** | JSON | JSON | Protobuf (binary) |
| **Protocol** | HTTP/1.1 | HTTP/1.1 | HTTP/2 |
| **Performance** | Slow | Medium | Fast |
| **Payload Size** | Large | Medium | Small |
| **Browser Support** | ✅ Yes | ✅ Yes | ❌ No (need proxy) |
| **Streaming** | ❌ SSE only | ✅ Subscriptions | ✅ Native |
| **Code Generation** | ❌ No | ❌ No | ✅ Yes |
| **Learning Curve** | Low | Medium | High |
| **Use Case** | Public APIs | Web/Mobile apps | Microservices |

**Когда использовать gRPC:**
- ✅ **Microservices** (внутреннее взаимодействие)
- ✅ **High performance** требуется
- ✅ **Bidirectional streaming** (real-time)
- ✅ **Polyglot** environment (разные языки)

**Когда НЕ использовать gRPC:**
- ❌ **Browser clients** (REST/GraphQL лучше)
- ❌ **Public API** (REST проще для сторонних разработчиков)
- ❌ **Simple CRUD** (overkill)

---

## 🏗️ Real-World Example: Order Service

### Proto Definition

**order.proto:**

```protobuf
syntax = "proto3";

package order;

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);
  rpc CancelOrder(CancelOrderRequest) returns (Order);
}

message Order {
  int32 id = 1;
  int32 user_id = 2;
  repeated OrderItem items = 3;
  double total = 4;
  OrderStatus status = 5;
  int64 created_at = 6;
}

message OrderItem {
  int32 product_id = 1;
  string product_name = 2;
  int32 quantity = 3;
  double price = 4;
}

enum OrderStatus {
  PENDING = 0;
  PAID = 1;
  SHIPPED = 2;
  DELIVERED = 3;
  CANCELLED = 4;
}

message CreateOrderRequest {
  int32 user_id = 1;
  repeated OrderItem items = 2;
}

message GetOrderRequest {
  int32 id = 1;
}

message ListOrdersRequest {
  int32 user_id = 1;
  int32 page = 2;
  int32 per_page = 3;
}

message CancelOrderRequest {
  int32 id = 1;
}
```

### Server Implementation

```php
<?php

namespace App\Services;

use Order\CreateOrderRequest;
use Order\GetOrderRequest;
use Order\ListOrdersRequest;
use Order\CancelOrderRequest;
use Order\Order;
use Order\OrderItem;
use Order\OrderStatus;
use Order\OrderServiceInterface;
use Spiral\RoadRunner\GRPC\ContextInterface;

class OrderService implements OrderServiceInterface
{
    public function CreateOrder(ContextInterface $ctx, CreateOrderRequest $request): Order
    {
        $userId = $request->getUserId();
        $items = $request->getItems();
        
        // Calculate total
        $total = 0;
        foreach ($items as $item) {
            $total += $item->getPrice() * $item->getQuantity();
        }
        
        // Save to DB
        $orderModel = \App\Models\Order::create([
            'user_id' => $userId,
            'total' => $total,
            'status' => 'pending',
        ]);
        
        foreach ($items as $item) {
            $orderModel->items()->create([
                'product_id' => $item->getProductId(),
                'product_name' => $item->getProductName(),
                'quantity' => $item->getQuantity(),
                'price' => $item->getPrice(),
            ]);
        }
        
        // Return protobuf object
        return $this->toProto($orderModel);
    }
    
    public function GetOrder(ContextInterface $ctx, GetOrderRequest $request): Order
    {
        $orderModel = \App\Models\Order::with('items')->findOrFail($request->getId());
        return $this->toProto($orderModel);
    }
    
    public function ListOrders(ContextInterface $ctx, ListOrdersRequest $request)
    {
        $orders = \App\Models\Order::where('user_id', $request->getUserId())
            ->with('items')
            ->paginate($request->getPerPage(), ['*'], 'page', $request->getPage());
        
        foreach ($orders as $orderModel) {
            yield $this->toProto($orderModel);
        }
    }
    
    public function CancelOrder(ContextInterface $ctx, CancelOrderRequest $request): Order
    {
        $orderModel = \App\Models\Order::findOrFail($request->getId());
        $orderModel->update(['status' => 'cancelled']);
        
        return $this->toProto($orderModel);
    }
    
    private function toProto(\App\Models\Order $orderModel): Order
    {
        $order = new Order();
        $order->setId($orderModel->id);
        $order->setUserId($orderModel->user_id);
        $order->setTotal($orderModel->total);
        $order->setStatus($this->mapStatus($orderModel->status));
        $order->setCreatedAt($orderModel->created_at->timestamp);
        
        foreach ($orderModel->items as $itemModel) {
            $item = new OrderItem();
            $item->setProductId($itemModel->product_id);
            $item->setProductName($itemModel->product_name);
            $item->setQuantity($itemModel->quantity);
            $item->setPrice($itemModel->price);
            
            $order->getItems()[] = $item;
        }
        
        return $order;
    }
    
    private function mapStatus(string $status): int
    {
        return match($status) {
            'pending' => OrderStatus::PENDING,
            'paid' => OrderStatus::PAID,
            'shipped' => OrderStatus::SHIPPED,
            'delivered' => OrderStatus::DELIVERED,
            'cancelled' => OrderStatus::CANCELLED,
            default => OrderStatus::PENDING,
        };
    }
}
```

### Client Usage

```php
<?php

use Order\CreateOrderRequest;
use Order\OrderItem;
use Order\OrderServiceClient;
use Grpc\ChannelCredentials;

$client = new OrderServiceClient('127.0.0.1:9001', [
    'credentials' => ChannelCredentials::createInsecure(),
]);

// Create order
$item1 = new OrderItem();
$item1->setProductId(1);
$item1->setProductName('Product 1');
$item1->setQuantity(2);
$item1->setPrice(99.99);

$item2 = new OrderItem();
$item2->setProductId(2);
$item2->setProductName('Product 2');
$item2->setQuantity(1);
$item2->setPrice(49.99);

$request = new CreateOrderRequest();
$request->setUserId(1);
$request->setItems([$item1, $item2]);

list($order, $status) = $client->CreateOrder($request)->wait();

if ($status->code === \Grpc\STATUS_OK) {
    echo "Order created: #{$order->getId()}, Total: \${$order->getTotal()}" . PHP_EOL;
} else {
    echo "Error: " . $status->details . PHP_EOL;
}
```

---

## 🧪 Testing

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use Order\CreateOrderRequest;
use Order\OrderItem;
use Order\OrderServiceClient;
use Grpc\ChannelCredentials;

class OrderGrpcTest extends TestCase
{
    private OrderServiceClient $client;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $this->client = new OrderServiceClient('127.0.0.1:9001', [
            'credentials' => ChannelCredentials::createInsecure(),
        ]);
    }
    
    public function test_creates_order()
    {
        $item = new OrderItem();
        $item->setProductId(1);
        $item->setProductName('Test Product');
        $item->setQuantity(1);
        $item->setPrice(99.99);
        
        $request = new CreateOrderRequest();
        $request->setUserId(1);
        $request->setItems([$item]);
        
        list($order, $status) = $this->client->CreateOrder($request)->wait();
        
        $this->assertEquals(\Grpc\STATUS_OK, $status->code);
        $this->assertEquals(99.99, $order->getTotal());
        $this->assertDatabaseHas('orders', [
            'user_id' => 1,
            'total' => 99.99,
        ]);
    }
}
```

---

## 🎓 Для собеседования: ключевые точки

1. **gRPC** - RPC framework от Google, HTTP/2 + protobuf
2. **Protocol Buffers** - бинарный формат, 3x меньше JSON, 5-10x быстрее
3. **Типы RPC** - Unary/Server Streaming/Client Streaming/Bidirectional
4. **PHP** - не native server, нужен RoadRunner/Spiral
5. **Performance** - ~4x быстрее REST благодаря binary + HTTP/2
6. **Use Case** - microservices, high-performance, streaming
7. **vs REST** - быстрее, но сложнее, не для браузеров
8. **vs GraphQL** - gRPC для backend-to-backend, GraphQL для frontend-to-backend
9. **Security** - metadata для auth, TLS для encryption
10. **Trade-offs** - performance vs complexity, не для public APIs

**Главное:** gRPC идеален для microservices благодаря высокой производительности и строгой типизации.
