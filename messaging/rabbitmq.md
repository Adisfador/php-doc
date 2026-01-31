# RabbitMQ (Концептуально)

## Что такое RabbitMQ

**RabbitMQ** - это message broker (брокер сообщений), реализующий протокол AMQP (Advanced Message Queuing Protocol).

### Основная задача
- Асинхронная обработка задач
- Decoupling сервисов (разделение ответственности)
- Load leveling (сглаживание нагрузки)
- Гарантия доставки сообщений

---

## Основные компоненты

```
Producer → Exchange → Queue → Consumer
```

### 1. Producer (Издатель)
- Отправляет сообщения в Exchange
- НЕ отправляет напрямую в Queue!

```php
// PHP пример
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->basic_publish(
    $message,           // Сообщение
    'logs_exchange',    // Exchange
    'info.log'         // Routing key
);
```

### 2. Exchange (Обменник)
- Принимает сообщения от Producer
- Маршрутизирует в Queue(s) по правилам
- **4 типа Exchange** (см. ниже)

### 3. Queue (Очередь)
- Хранит сообщения
- Отдает Consumer'ам
- FIFO (First In, First Out)
- Может быть durable (переживает рестарт) или temporary

### 4. Consumer (Потребитель)
- Получает сообщения из Queue
- Обрабатывает
- Отправляет ACK (подтверждение)

```php
$callback = function ($msg) {
    echo 'Received: ', $msg->body, "\n";
    $msg->ack();  // Подтверждение обработки
};

$channel->basic_consume(
    'task_queue',   // Queue name
    '',             // Consumer tag
    false,          // no_local
    false,          // no_ack (manual ack)
    false,          // exclusive
    false,          // nowait
    $callback       // Callback
);
```

### 5. Binding (Привязка)
- Связь между Exchange и Queue
- Определяет правила маршрутизации
- Использует Routing Key

```php
$channel->queue_bind(
    'error_logs_queue',  // Queue
    'logs_exchange',     // Exchange
    'error.*'            // Routing key pattern
);
```

---

## Типы Exchange

### 1. Direct Exchange

**Принцип:** Точное совпадение routing key

```
Producer → Exchange (direct) → Queue
           routing_key="error"
           
Queue1: binding_key="error"    ← получит
Queue2: binding_key="info"     ← НЕ получит
Queue3: binding_key="error"    ← получит
```

**Когда использовать:**
- Простая маршрутизация
- Распределение задач по типам

**Пример:**
```php
// Создание exchange
$channel->exchange_declare(
    'direct_logs',      // Exchange name
    'direct',           // Type
    false,              // Passive
    true,               // Durable
    false               // Auto-delete
);

// Binding
$channel->queue_bind('error_queue', 'direct_logs', 'error');
$channel->queue_bind('info_queue', 'direct_logs', 'info');

// Publish
$channel->basic_publish($msg, 'direct_logs', 'error');  // → error_queue
```

### 2. Fanout Exchange

**Принцип:** Broadcast всем привязанным очередям (игнорирует routing key)

```
Producer → Exchange (fanout) → Queue1
                            → Queue2
                            → Queue3
                            (все получат)
```

**Когда использовать:**
- Broadcasting (уведомления всем)
- Pub/Sub паттерн
- Cache invalidation на всех нодах

**Пример:**
```php
$channel->exchange_declare('logs', 'fanout', false, false, false);

$channel->queue_bind('queue1', 'logs');
$channel->queue_bind('queue2', 'logs');

// Все очереди получат
$channel->basic_publish($msg, 'logs', '');
```

### 3. Topic Exchange

**Принцип:** Pattern matching по routing key

**Wildcards:**
- `*` - одно слово
- `#` - ноль или больше слов

```
Routing key: "user.created.email"

Binding patterns:
"user.created.*"     ← Match
"user.*.*"           ← Match
"user.#"             ← Match
"*.created.#"        ← Match
"order.#"            ← No match
```

**Когда использовать:**
- Сложная маршрутизация
- Event-driven архитектура
- Подписка на категории событий

**Пример:**
```php
$channel->exchange_declare('events', 'topic', false, true, false);

// Bindings
$channel->queue_bind('email_queue', 'events', '*.*.email');
$channel->queue_bind('sms_queue', 'events', '*.*.sms');
$channel->queue_bind('all_user_events', 'events', 'user.#');

// Publishing
$channel->basic_publish($msg, 'events', 'user.created.email');
// → email_queue + all_user_events
```

### 4. Headers Exchange

**Принцип:** Маршрутизация по заголовкам сообщения (не routing key)

```php
// Binding с условиями
$args = new AMQPTable([
    'x-match' => 'all',  // или 'any'
    'format' => 'pdf',
    'type' => 'report'
]);

$channel->queue_bind('pdf_reports', 'headers_exchange', '', false, $args);

// Publishing с headers
$headers = new AMQPTable(['format' => 'pdf', 'type' => 'report']);
$msg = new AMQPMessage($body, ['application_headers' => $headers]);
$channel->basic_publish($msg, 'headers_exchange');
```

**Когда использовать:**
- Сложные условия маршрутизации
- Многомерная фильтрация

---

## Свойства сообщений

### Delivery Mode
```php
$msg = new AMQPMessage($body, [
    'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT  // 2
]);
```
- **1** - Non-persistent (в памяти, быстро)
- **2** - Persistent (на диск, надежно)

### Priority
```php
$msg = new AMQPMessage($body, ['priority' => 10]);
```
- Приоритет 0-255
- Queue должна быть создана с `x-max-priority`

### TTL (Time To Live)
```php
// На уровне сообщения
$msg = new AMQPMessage($body, ['expiration' => '60000']);  // ms

// На уровне очереди
$args = new AMQPTable(['x-message-ttl' => 60000]);
$channel->queue_declare('my_queue', false, true, false, false, false, $args);
```

### Headers
```php
$headers = new AMQPTable([
    'user_id' => 123,
    'retry_count' => 3
]);

$msg = new AMQPMessage($body, ['application_headers' => $headers]);
```

---

## Паттерны работы

### 1. Work Queue (Task Queue)

Распределение задач между workers

```
Producer → Queue → Worker1
                 → Worker2
                 → Worker3
```

**Fair Dispatch** (равномерное распределение):
```php
// Не отправлять новое сообщение worker'у, пока он не обработал текущее
$channel->basic_qos(
    null,   // Prefetch size
    1,      // Prefetch count
    null    // Global
);
```

### 2. Publish/Subscribe

Broadcasting сообщений

```php
// Publisher
$channel->exchange_declare('news', 'fanout');
$channel->basic_publish($msg, 'news');

// Subscriber
$queue = $channel->queue_declare('', false, false, true, false);
$queueName = $queue[0];  // Temporary queue
$channel->queue_bind($queueName, 'news');
$channel->basic_consume($queueName, '', false, true, false, false, $callback);
```

### 3. RPC (Remote Procedure Call)

Запрос-ответ через RabbitMQ

```
Client → request_queue → Server
      ← reply_queue    ←
```

```php
// Client
$correlationId = uniqid();
$msg = new AMQPMessage($request, [
    'correlation_id' => $correlationId,
    'reply_to' => 'amq.rabbitmq.reply-to'  // Direct reply-to
]);

$channel->basic_publish($msg, '', 'rpc_queue');

// Server
$callback = function ($req) {
    $result = processRequest($req->body);
    
    $msg = new AMQPMessage($result, [
        'correlation_id' => $req->get('correlation_id')
    ]);
    
    $req->getChannel()->basic_publish($msg, '', $req->get('reply_to'));
    $req->ack();
};
```

### 4. Dead Letter Exchange (DLQ)

Обработка неудачных сообщений

```php
// Создание DLX
$channel->exchange_declare('dlx', 'direct', false, true, false);
$channel->queue_declare('dead_letters', false, true, false, false);
$channel->queue_bind('dead_letters', 'dlx', 'rejected');

// Main queue с DLX
$args = new AMQPTable([
    'x-dead-letter-exchange' => 'dlx',
    'x-dead-letter-routing-key' => 'rejected'
]);

$channel->queue_declare('main_queue', false, true, false, false, false, $args);
```

**Сообщение попадает в DLQ:**
- Reject с requeue=false
- Превышен TTL
- Превышена длина очереди

### 5. Delayed Messages

```php
// Используя DLX + TTL
$args = new AMQPTable([
    'x-message-ttl' => 5000,  // 5 секунд
    'x-dead-letter-exchange' => 'main_exchange'
]);

$channel->queue_declare('delay_queue', false, true, false, false, false, $args);

// Отправка в delay_queue → через 5 сек попадет в main_exchange
```

---

## Acknowledgment (подтверждение)

### Manual ACK
```php
$channel->basic_consume('queue', '', false, false, false, false, function ($msg) {
    try {
        processMessage($msg->body);
        $msg->ack();  // Успешно
    } catch (Exception $e) {
        $msg->nack(false, true);  // Requeue
        // или
        $msg->reject(true);  // Requeue
    }
});
```

### Auto ACK (не рекомендуется)
```php
$channel->basic_consume('queue', '', false, true, false, false, $callback);
//                                           ^^^^ no_ack=true
```

### Publisher Confirms
```php
$channel->confirm_select();

$channel->basic_publish($msg, 'exchange', 'routing_key');

$channel->wait_for_pending_acks_returns();  // Ждем подтверждения от RabbitMQ
```

---

## Гарантии доставки

### At-Most-Once
- Auto ACK
- Сообщение может быть потеряно

### At-Least-Once
- Manual ACK + requeue on failure
- Дубликаты возможны

### Exactly-Once
- Idempotent consumers (проверка на дубликаты)
- Deduplication на стороне приложения

```php
// Пример idempotent consumer
function handleMessage($msg) {
    $messageId = $msg->get('message_id');
    
    if (Redis::exists("processed:$messageId")) {
        $msg->ack();  // Уже обработано
        return;
    }
    
    DB::transaction(function () use ($msg, $messageId) {
        processMessage($msg->body);
        Redis::set("processed:$messageId", 1, 'EX', 3600);
    });
    
    $msg->ack();
}
```

---

## Laravel интеграция

### Конфигурация

```php
// config/queue.php
'connections' => [
    'rabbitmq' => [
        'driver' => 'rabbitmq',
        'host' => env('RABBITMQ_HOST', 'localhost'),
        'port' => env('RABBITMQ_PORT', 5672),
        'user' => env('RABBITMQ_USER', 'guest'),
        'password' => env('RABBITMQ_PASSWORD', 'guest'),
        'vhost' => env('RABBITMQ_VHOST', '/'),
        'queue' => env('RABBITMQ_QUEUE', 'default'),
        'options' => [
            'exchange' => [
                'name' => 'default',
                'type' => 'direct',
            ],
        ],
    ],
],
```

### Dispatch Job

```php
// Job
class ProcessOrder implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public function __construct(
        public Order $order
    ) {}
    
    public function handle()
    {
        // Обработка
    }
}

// Отправка
ProcessOrder::dispatch($order);

// С задержкой
ProcessOrder::dispatch($order)->delay(now()->addMinutes(5));

// На конкретную очередь
ProcessOrder::dispatch($order)->onQueue('high-priority');
```

---

## Мониторинг

### Management UI
- http://localhost:15672
- Логин: guest/guest
- Очереди, exchanges, connections, channels

### CLI
```bash
# Список очередей
rabbitmqctl list_queues

# Статистика
rabbitmqctl list_queues name messages_ready messages_unacknowledged

# Список exchanges
rabbitmqctl list_exchanges

# Список bindings
rabbitmqctl list_bindings
```

---

## Когда использовать RabbitMQ

### ✅ Хорошо подходит для:
- Асинхронная обработка (emails, notifications)
- Decoupling microservices
- Load leveling
- RPC между сервисами
- Event-driven architecture

### ❌ Не подходит для:
- Real-time требования (используйте WebSockets)
- Огромные сообщения (>128MB, используйте S3 + reference)
- Строгий порядок обработки (используйте Kafka)
- High throughput streaming (используйте Kafka)

---

## RabbitMQ vs Kafka

| Критерий | RabbitMQ | Kafka |
|----------|----------|-------|
| **Модель** | Message broker | Distributed log |
| **Гарантии порядка** | На уровне очереди | На уровне partition |
| **Throughput** | ~20K msg/sec | ~1M msg/sec |
| **Retention** | До обработки | Configurable (дни/недели) |
| **Use case** | Task queues, RPC | Event streaming, logs |
| **Replay** | Сложно | Легко (offset) |

---

## Ключевые вопросы для интервью

- Объясните разницу между типами Exchange
- Как работает маршрутизация в Topic Exchange?
- Что такое Dead Letter Exchange и когда его использовать?
- Как обеспечить exactly-once delivery?
- В чем разница между ACK и NACK?
- Что такое prefetch и зачем он нужен?
- Как реализовать delayed messages?
- Когда использовать RabbitMQ, а когда Kafka?
- Что такое durable queue и persistent messages?
- Как работает Publisher Confirms?
