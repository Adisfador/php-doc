# Message Patterns

Архитектурные паттерны работы с системами обмена сообщениями (Kafka, RabbitMQ).

---

## 📮 Основные паттерны обмена сообщениями

### 1. Publish-Subscribe (Pub-Sub)

**Один publisher, множество subscribers. Все подписчики получают копию сообщения.**

```
Publisher → Topic → [Subscriber 1]
                  → [Subscriber 2]
                  → [Subscriber 3]
```

**Характеристики:**
- Fan-out (один ко многим)
- Decoupling - publisher не знает о subscribers
- Broadcasting - все получают сообщение

**RabbitMQ:**

```php
<?php

// Publisher
$channel->exchangeDeclare('user_events', 'fanout', false, true, false);

$message = json_encode(['user_id' => 123, 'event' => 'registered']);
$channel->basic_publish(
    new AMQPMessage($message),
    'user_events'  // exchange
);

// Subscriber 1: Email Service
$channel->queueDeclare('email_queue', false, true, false, false);
$channel->queueBind('email_queue', 'user_events');

$channel->basic_consume('email_queue', '', false, false, false, false, 
    function ($msg) {
        $data = json_decode($msg->body, true);
        sendWelcomeEmail($data['user_id']);
        $msg->ack();
    }
);

// Subscriber 2: Analytics Service
$channel->queueDeclare('analytics_queue', false, true, false, false);
$channel->queueBind('analytics_queue', 'user_events');

$channel->basic_consume('analytics_queue', '', false, false, false, false,
    function ($msg) {
        $data = json_decode($msg->body, true);
        trackUserRegistration($data['user_id']);
        $msg->ack();
    }
);
```

**Kafka:**

```php
<?php

// Publisher
$producer = new RdKafka\Producer($conf);
$topic = $producer->newTopic('user_events');

$message = json_encode(['user_id' => 123, 'event' => 'registered']);
$topic->produce(RD_KAFKA_PARTITION_UA, 0, $message);
$producer->flush(5000);

// Subscriber 1: Email Service (consumer group: email-service)
$conf->set('group.id', 'email-service');
$consumer1 = new RdKafka\KafkaConsumer($conf);
$consumer1->subscribe(['user_events']);

while (true) {
    $message = $consumer1->consume(1000);
    if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
        $data = json_decode($message->payload, true);
        sendWelcomeEmail($data['user_id']);
        $consumer1->commit($message);
    }
}

// Subscriber 2: Analytics Service (consumer group: analytics)
$conf->set('group.id', 'analytics');
$consumer2 = new RdKafka\KafkaConsumer($conf);
$consumer2->subscribe(['user_events']);
// ... аналогично
```

**Use cases:**
- User registration → Email + Analytics + Notification services
- Order placed → Inventory + Shipping + Accounting
- Sensor data → Dashboard + Alerting + Archive

---

### 2. Point-to-Point (Queue)

**Один producer, один consumer. Сообщение обрабатывается только одним consumer.**

```
Producer → Queue → [Consumer 1]
                   [Consumer 2] (competing)
                   [Consumer 3] (competing)

Только ОДИН consumer получит сообщение
```

**Характеристики:**
- Competing consumers - конкуренция за сообщения
- Load balancing - распределение нагрузки
- Сообщение удаляется после ack

**RabbitMQ:**

```php
<?php

// Producer
$channel->queueDeclare('image_processing', false, true, false, false);

$message = json_encode(['image_url' => 'https://example.com/photo.jpg']);
$channel->basic_publish(
    new AMQPMessage($message),
    '',  // default exchange
    'image_processing'  // routing key = queue name
);

// Consumer 1
$callback = function ($msg) {
    $data = json_decode($msg->body, true);
    processImage($data['image_url']);
    $msg->ack();
};

$channel->basic_consume('image_processing', '', false, false, false, false, $callback);
$channel->wait();

// Consumer 2, 3, ... - запускаем параллельно
// RabbitMQ распределяет сообщения round-robin
```

**Kafka (Consumer Group):**

```php
<?php

// Producer
$topic = $producer->newTopic('image_processing');
$topic->produce(RD_KAFKA_PARTITION_UA, 0, json_encode(['image_url' => '...']));

// Consumers в одной группе - point-to-point
$conf->set('group.id', 'image-processors');  // ОДНА группа

// Consumer 1
$consumer1 = new RdKafka\KafkaConsumer($conf);
$consumer1->subscribe(['image_processing']);

// Consumer 2, 3 - в той же группе
// Каждый partition обрабатывается только одним consumer в группе
```

**Use cases:**
- Image processing queue
- Email sending queue
- Background job processing
- Task distribution

---

### 3. Request-Reply (RPC)

**Синхронная коммуникация: отправить запрос, дождаться ответа.**

```
Client → Request Queue → Server
       ← Reply Queue   ←
```

**RabbitMQ:**

```php
<?php

// Server (RPC Server)
class RpcServer
{
    private $channel;
    
    public function __construct($channel)
    {
        $this->channel = $channel;
        $this->channel->queueDeclare('rpc_queue', false, false, false, false);
    }
    
    public function start(): void
    {
        $this->channel->basic_consume(
            'rpc_queue',
            '',
            false,
            false,
            false,
            false,
            [$this, 'onRequest']
        );
        
        $this->channel->wait();
    }
    
    public function onRequest($req): void
    {
        $n = intval($req->body);
        $result = $this->fibonacci($n);
        
        // Отправить ответ в reply_to queue
        $msg = new AMQPMessage(
            (string) $result,
            ['correlation_id' => $req->get('correlation_id')]
        );
        
        $this->channel->basic_publish($msg, '', $req->get('reply_to'));
        $req->ack();
    }
    
    private function fibonacci(int $n): int
    {
        if ($n <= 1) return $n;
        return $this->fibonacci($n - 1) + $this->fibonacci($n - 2);
    }
}

// Client (RPC Client)
class RpcClient
{
    private $channel;
    private $callbackQueue;
    private $response;
    private $correlationId;
    
    public function __construct($channel)
    {
        $this->channel = $channel;
        
        // Создать временную очередь для ответов
        list($this->callbackQueue, ,) = $this->channel->queueDeclare(
            '',     // random name
            false,
            false,
            true,   // exclusive
            false
        );
        
        // Слушать ответы
        $this->channel->basic_consume(
            $this->callbackQueue,
            '',
            false,
            false,
            false,
            false,
            [$this, 'onResponse']
        );
    }
    
    public function call(int $n): int
    {
        $this->response = null;
        $this->correlationId = uniqid();
        
        // Отправить запрос
        $msg = new AMQPMessage(
            (string) $n,
            [
                'correlation_id' => $this->correlationId,
                'reply_to' => $this->callbackQueue,
            ]
        );
        
        $this->channel->basic_publish($msg, '', 'rpc_queue');
        
        // Ждать ответ
        while ($this->response === null) {
            $this->channel->wait();
        }
        
        return intval($this->response);
    }
    
    public function onResponse($rep): void
    {
        if ($rep->get('correlation_id') === $this->correlationId) {
            $this->response = $rep->body;
        }
    }
}

// Использование
$client = new RpcClient($channel);
echo "Fibonacci(10) = ", $client->call(10), "\n";
```

**Use cases:**
- Microservice communication (Order Service → Inventory Service)
- Remote calculations
- Synchronous API calls через message broker

**Недостатки:**
- Tight coupling
- Blocking - ждем ответа
- Лучше использовать HTTP/gRPC для RPC

---

### 4. Routing Patterns

#### Content-Based Routing

**Маршрутизация на основе содержимого сообщения.**

**RabbitMQ (Topic Exchange):**

```php
<?php

// Объявить topic exchange
$channel->exchangeDeclare('logs', 'topic', false, true, false);

// Producer отправляет с routing key
$routingKeys = [
    'user.created.email',
    'user.created.sms',
    'order.placed.email',
    'order.shipped.sms',
];

foreach ($routingKeys as $key) {
    $message = json_encode(['event' => $key]);
    $channel->basic_publish(new AMQPMessage($message), 'logs', $key);
}

// Consumer 1: все email уведомления
$channel->queueDeclare('email_queue', false, true, false, false);
$channel->queueBind('email_queue', 'logs', '*.*.email');  // pattern

// Consumer 2: все user события
$channel->queueDeclare('user_queue', false, true, false, false);
$channel->queueBind('user_queue', 'logs', 'user.*.*');

// Consumer 3: конкретное событие
$channel->queueDeclare('order_placed_queue', false, true, false, false);
$channel->queueBind('order_placed_queue', 'logs', 'order.placed.*');
```

**Routing patterns:**
- `*` - один элемент: `user.*.email` → `user.created.email`
- `#` - ноль или более: `user.#` → `user.created.email`, `user.created`

**RabbitMQ (Headers Exchange):**

```php
<?php

// Маршрутизация по headers
$channel->exchangeDeclare('events', 'headers', false, true, false);

// Producer
$properties = [
    'application_headers' => [
        'type' => 'order',
        'priority' => 'high',
        'region' => 'EU',
    ]
];

$channel->basic_publish(
    new AMQPMessage($message, $properties),
    'events'
);

// Consumer: все high priority заказы
$channel->queueBind('high_priority_queue', 'events', '', [
    'x-match' => 'all',  // all | any
    'type' => 'order',
    'priority' => 'high',
]);
```

#### Message Filter

**Consumer фильтрует ненужные сообщения.**

```php
<?php

$channel->basic_consume('all_events', '', false, false, false, false,
    function ($msg) {
        $data = json_decode($msg->body, true);
        
        // Фильтр: обрабатываем только high priority
        if ($data['priority'] !== 'high') {
            $msg->ack();  // skip
            return;
        }
        
        processHighPriorityEvent($data);
        $msg->ack();
    }
);
```

#### Recipient List

**Отправить сообщение списку получателей (динамический).**

```php
<?php

class RecipientListRouter
{
    private $channel;
    
    public function route(array $message, array $recipients): void
    {
        foreach ($recipients as $recipient) {
            $this->channel->basic_publish(
                new AMQPMessage(json_encode($message)),
                '',
                $recipient  // queue name
            );
        }
    }
}

// Использование
$router = new RecipientListRouter($channel);

$message = ['order_id' => 123, 'status' => 'shipped'];

// Динамически определяем получателей
$recipients = [];
if ($order->requiresEmailNotification()) {
    $recipients[] = 'email_queue';
}
if ($order->requiresSmsNotification()) {
    $recipients[] = 'sms_queue';
}
if ($order->isPremium()) {
    $recipients[] = 'premium_queue';
}

$router->route($message, $recipients);
```

#### Splitter

**Разбить составное сообщение на части.**

```php
<?php

// Получили batch заказов
$batchMessage = [
    'orders' => [
        ['id' => 1, 'amount' => 100],
        ['id' => 2, 'amount' => 200],
        ['id' => 3, 'amount' => 300],
    ]
];

// Splitter: отправить каждый заказ отдельно
foreach ($batchMessage['orders'] as $order) {
    $channel->basic_publish(
        new AMQPMessage(json_encode($order)),
        '',
        'single_order_queue'
    );
}
```

#### Aggregator

**Собрать связанные сообщения в одно.**

```php
<?php

class OrderAggregator
{
    private $orders = [];
    
    public function consume($channel): void
    {
        $channel->basic_consume('order_items', '', false, false, false, false,
            function ($msg) {
                $item = json_decode($msg->body, true);
                $orderId = $item['order_id'];
                
                // Собрать все items заказа
                $this->orders[$orderId][] = $item;
                
                // Когда все items собраны (correlation ID)
                if ($this->isComplete($orderId)) {
                    $completeOrder = [
                        'order_id' => $orderId,
                        'items' => $this->orders[$orderId],
                        'total' => $this->calculateTotal($orderId),
                    ];
                    
                    // Отправить полный заказ
                    $this->publishCompleteOrder($completeOrder);
                    
                    unset($this->orders[$orderId]);
                }
                
                $msg->ack();
            }
        );
    }
    
    private function isComplete(int $orderId): bool
    {
        // Проверка по correlation_id или count
        return count($this->orders[$orderId]) >= 3;  // пример
    }
}
```

---

### 5. Message Transformation

#### Message Translator

**Конвертация между форматами.**

```php
<?php

class MessageTranslator
{
    public function jsonToXml(string $json): string
    {
        $data = json_decode($json, true);
        $xml = new SimpleXMLElement('<root/>');
        array_walk_recursive($data, [$xml, 'addChild']);
        return $xml->asXML();
    }
    
    public function xmlToJson(string $xml): string
    {
        $obj = simplexml_load_string($xml);
        return json_encode($obj);
    }
}

// Consumer: читает JSON, пишет XML
$channel->basic_consume('json_queue', '', false, false, false, false,
    function ($msg) use ($translator, $channel) {
        $xml = $translator->jsonToXml($msg->body);
        
        $channel->basic_publish(
            new AMQPMessage($xml),
            '',
            'xml_queue'
        );
        
        $msg->ack();
    }
);
```

#### Envelope Wrapper

**Добавление метаданных к сообщению.**

```php
<?php

class EnvelopeWrapper
{
    public function wrap(array $payload): array
    {
        return [
            'envelope' => [
                'message_id' => uniqid(),
                'timestamp' => time(),
                'version' => '1.0',
                'source' => 'order-service',
            ],
            'payload' => $payload,
        ];
    }
    
    public function unwrap(array $envelope): array
    {
        // Валидация envelope
        if (!isset($envelope['envelope']['message_id'])) {
            throw new Exception('Invalid envelope');
        }
        
        return $envelope['payload'];
    }
}

// Producer
$wrapper = new EnvelopeWrapper();
$message = $wrapper->wrap(['order_id' => 123]);
$channel->basic_publish(new AMQPMessage(json_encode($message)), '', 'orders');

// Consumer
$message = json_decode($msg->body, true);
$payload = $wrapper->unwrap($message);
```

#### Content Enricher

**Обогащение сообщения дополнительными данными.**

```php
<?php

class ContentEnricher
{
    public function enrich(array $order): array
    {
        // Обогатить заказ данными пользователя из БД
        $user = User::find($order['user_id']);
        $order['user'] = [
            'name' => $user->name,
            'email' => $user->email,
            'vip_status' => $user->is_vip,
        ];
        
        // Обогатить данными продуктов
        foreach ($order['items'] as &$item) {
            $product = Product::find($item['product_id']);
            $item['product_name'] = $product->name;
            $item['category'] = $product->category;
        }
        
        return $order;
    }
}

// Consumer: enricher
$channel->basic_consume('raw_orders', '', false, false, false, false,
    function ($msg) use ($enricher, $channel) {
        $order = json_decode($msg->body, true);
        $enrichedOrder = $enricher->enrich($order);
        
        // Отправить обогащенный заказ
        $channel->basic_publish(
            new AMQPMessage(json_encode($enrichedOrder)),
            '',
            'enriched_orders'
        );
        
        $msg->ack();
    }
);
```

#### Normalizer

**Преобразование к каноническому формату.**

```php
<?php

class OrderNormalizer
{
    public function normalize(array $order, string $source): array
    {
        // Разные источники → единый формат
        return match($source) {
            'shopify' => [
                'order_id' => $order['id'],
                'customer_email' => $order['customer']['email'],
                'items' => array_map(fn($item) => [
                    'sku' => $item['sku'],
                    'quantity' => $item['quantity'],
                    'price' => $item['price'],
                ], $order['line_items']),
            ],
            'woocommerce' => [
                'order_id' => $order['order_id'],
                'customer_email' => $order['billing']['email'],
                'items' => array_map(fn($item) => [
                    'sku' => $item['product_sku'],
                    'quantity' => $item['qty'],
                    'price' => $item['total'],
                ], $order['items']),
            ],
            default => throw new Exception("Unknown source: {$source}"),
        };
    }
}
```

---

### 6. Reliability Patterns

#### At-Least-Once Delivery

**Гарантия доставки, возможны дубликаты.**

```php
<?php

// Producer: wait for ack
$conf->set('acks', 'all');  // Kafka
$producer->flush(10000);

// Consumer: commit ПОСЛЕ обработки
$message = $consumer->consume(1000);
processMessage($message);  // может быть вызвано дважды!
$consumer->commit($message);

// ✅ Идемпотентная обработка
DB::table('orders')->updateOrInsert(
    ['order_id' => $data['order_id']],  // unique key
    $data
);

// ❌ НЕ идемпотентная
DB::table('counters')->increment('total');  // дубликат = +2!
```

#### At-Most-Once Delivery

**Быстрая доставка, возможна потеря.**

```php
<?php

// Consumer: commit ДО обработки
$message = $consumer->consume(1000);
$consumer->commit($message);  // commit сразу
processMessage($message);     // crash → потеря сообщения
```

#### Exactly-Once Delivery

**Гарантия ровно один раз (сложно!).**

```php
<?php

// Kafka: idempotent producer + transactions
$conf->set('enable.idempotence', 'true');
$conf->set('transactional.id', 'my-transactional-id');

$producer = new RdKafka\Producer($conf);
$producer->initTransactions(10000);

$producer->beginTransaction();

try {
    $topic->produce(RD_KAFKA_PARTITION_UA, 0, $message1);
    $topic->produce(RD_KAFKA_PARTITION_UA, 0, $message2);
    
    $producer->commitTransaction(10000);
} catch (Exception $e) {
    $producer->abortTransaction(10000);
}

// Consumer: read_committed
$conf->set('isolation.level', 'read_committed');
```

**Альтернатива: Идемпотентность + Deduplication:**

```php
<?php

class DeduplicatingConsumer
{
    private $cache;
    
    public function consume($message): void
    {
        $messageId = $message->key;  // unique ID
        
        // Проверить дубликат
        if ($this->cache->has($messageId)) {
            echo "Duplicate message: {$messageId}\n";
            return;  // skip
        }
        
        // Обработать
        processMessage($message);
        
        // Сохранить ID на 1 час (TTL)
        $this->cache->put($messageId, true, 3600);
    }
}
```

#### Retry Pattern

**Повторная обработка при ошибках.**

```php
<?php

class RetryableConsumer
{
    private const MAX_RETRIES = 3;
    
    public function consume($message): void
    {
        $retries = 0;
        
        while ($retries < self::MAX_RETRIES) {
            try {
                $this->process($message);
                $message->ack();
                return;
                
            } catch (TransientException $e) {
                // Временная ошибка - retry
                $retries++;
                $delay = $this->calculateBackoff($retries);
                usleep($delay * 1000);
                
                echo "Retry {$retries}/{self::MAX_RETRIES}: {$e->getMessage()}\n";
                
            } catch (PermanentException $e) {
                // Постоянная ошибка - DLQ
                $this->sendToDeadLetterQueue($message, $e);
                $message->ack();
                return;
            }
        }
        
        // Max retries exceeded
        $this->sendToDeadLetterQueue($message, new Exception('Max retries'));
        $message->ack();
    }
    
    private function calculateBackoff(int $retry): int
    {
        // Exponential backoff: 100ms, 200ms, 400ms, 800ms
        return 100 * pow(2, $retry - 1);
    }
}
```

#### Circuit Breaker

**Предотвращение каскадных сбоев.**

```php
<?php

class CircuitBreaker
{
    private const FAILURE_THRESHOLD = 5;
    private const TIMEOUT = 60;  // секунд
    
    private int $failures = 0;
    private ?int $openedAt = null;
    private string $state = 'closed';  // closed | open | half-open
    
    public function call(callable $action)
    {
        if ($this->state === 'open') {
            if (time() - $this->openedAt > self::TIMEOUT) {
                $this->state = 'half-open';
                echo "Circuit breaker: half-open\n";
            } else {
                throw new Exception('Circuit breaker is OPEN');
            }
        }
        
        try {
            $result = $action();
            
            // Success
            if ($this->state === 'half-open') {
                $this->state = 'closed';
                $this->failures = 0;
                echo "Circuit breaker: closed\n";
            }
            
            return $result;
            
        } catch (Exception $e) {
            $this->failures++;
            
            if ($this->failures >= self::FAILURE_THRESHOLD) {
                $this->state = 'open';
                $this->openedAt = time();
                echo "Circuit breaker: OPEN\n";
            }
            
            throw $e;
        }
    }
}

// Использование
$breaker = new CircuitBreaker();

$channel->basic_consume('orders', '', false, false, false, false,
    function ($msg) use ($breaker) {
        try {
            $breaker->call(function () use ($msg) {
                // Вызов внешнего API (может упасть)
                $this->externalApi->processOrder($msg->body);
            });
            
            $msg->ack();
            
        } catch (Exception $e) {
            // Circuit breaker open - skip message
            $msg->nack(false, true);  // requeue
        }
    }
);
```

#### Dead Letter Queue (DLQ)

**Очередь для failed messages.**

```php
<?php

// RabbitMQ: DLQ с TTL
$channel->queueDeclare('main_queue', false, true, false, false, false, [
    'x-dead-letter-exchange' => ['S', 'dlx'],
    'x-message-ttl' => ['I', 60000],  // 60 sec TTL
]);

$channel->exchangeDeclare('dlx', 'direct', false, true, false);
$channel->queueDeclare('dead_letter_queue', false, true, false, false);
$channel->queueBind('dead_letter_queue', 'dlx', 'main_queue');

// Consumer
$channel->basic_consume('main_queue', '', false, false, false, false,
    function ($msg) {
        try {
            processMessage($msg->body);
            $msg->ack();
            
        } catch (Exception $e) {
            // Reject → DLQ (через TTL или сразу)
            $msg->nack(false, false);  // don't requeue
            
            // Или отправить вручную
            $this->channel->basic_publish(
                new AMQPMessage($msg->body, [
                    'headers' => [
                        'x-original-error' => $e->getMessage(),
                        'x-failed-at' => time(),
                    ]
                ]),
                'dlx',
                'main_queue'
            );
        }
    }
);

// DLQ consumer (manual inspection/retry)
$channel->basic_consume('dead_letter_queue', '', false, false, false, false,
    function ($msg) {
        echo "Failed message: {$msg->body}\n";
        $headers = $msg->get('application_headers');
        echo "Error: {$headers['x-original-error']}\n";
        
        // Manual retry или log
        $msg->ack();
    }
);
```

**Kafka DLQ:**

```php
<?php

class KafkaDLQHandler
{
    private $dlqProducer;
    
    public function consume($consumer): void
    {
        while (true) {
            $message = $consumer->consume(1000);
            
            if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
                try {
                    processMessage($message->payload);
                    $consumer->commit($message);
                    
                } catch (Exception $e) {
                    // Отправить в DLQ topic
                    $this->sendToDLQ($message, $e);
                    $consumer->commit($message);  // commit original
                }
            }
        }
    }
    
    private function sendToDLQ($message, Exception $e): void
    {
        $dlqTopic = $this->dlqProducer->newTopic($message->topic_name . '_dlq');
        
        $dlqMessage = json_encode([
            'original_topic' => $message->topic_name,
            'original_partition' => $message->partition,
            'original_offset' => $message->offset,
            'original_payload' => $message->payload,
            'error' => $e->getMessage(),
            'failed_at' => time(),
        ]);
        
        $dlqTopic->produce(RD_KAFKA_PARTITION_UA, 0, $dlqMessage);
        $this->dlqProducer->flush(5000);
    }
}
```

---

### 7. Message Expiration (TTL)

**Сообщения с временем жизни.**

**RabbitMQ:**

```php
<?php

// Per-message TTL
$properties = ['expiration' => '60000'];  // 60 секунд
$channel->basic_publish(
    new AMQPMessage($message, $properties),
    '',
    'queue_name'
);

// Per-queue TTL
$channel->queueDeclare('temp_queue', false, true, false, false, false, [
    'x-message-ttl' => ['I', 60000],  // все сообщения живут 60 сек
]);
```

**Use cases:**
- Temporary offers
- Session tokens
- Cache invalidation

---

### 8. Priority Queues

**Обработка сообщений по приоритету.**

**RabbitMQ:**

```php
<?php

// Объявить очередь с приоритетами (0-255)
$channel->queueDeclare('priority_queue', false, true, false, false, false, [
    'x-max-priority' => ['I', 10],
]);

// Отправить с приоритетом
$highPriority = new AMQPMessage('URGENT', ['priority' => 10]);
$lowPriority = new AMQPMessage('Normal', ['priority' => 1]);

$channel->basic_publish($highPriority, '', 'priority_queue');
$channel->basic_publish($lowPriority, '', 'priority_queue');

// Consumer получит highPriority первым
```

**Use cases:**
- VIP orders
- Critical alerts
- Express delivery

---

### 9. Idempotency

**Повторная обработка = тот же результат.**

```php
<?php

class IdempotentOrderProcessor
{
    public function process(array $order): void
    {
        $orderId = $order['id'];
        
        // ✅ Идемпотентно: upsert
        DB::table('orders')->updateOrInsert(
            ['id' => $orderId],
            [
                'user_id' => $order['user_id'],
                'total' => $order['total'],
                'status' => 'processing',
                'updated_at' => now(),
            ]
        );
        
        // ✅ Идемпотентно: проверка перед действием
        $existingPayment = Payment::where('order_id', $orderId)->first();
        if (!$existingPayment) {
            Payment::create(['order_id' => $orderId, 'amount' => $order['total']]);
        }
        
        // ❌ НЕ идемпотентно
        DB::table('order_count')->increment('count');  // +1 при каждом вызове!
        
        // ✅ Идемпотентно: increment с условием
        if (!Cache::has("order_counted:{$orderId}")) {
            DB::table('order_count')->increment('count');
            Cache::put("order_counted:{$orderId}", true, 3600);
        }
    }
}
```

**Техники:**
1. **Unique constraints** в БД
2. **Deduplication cache** (Redis)
3. **State machines** - только валидные переходы
4. **Check before action** - проверка существования

---

### 10. Saga Pattern

**Distributed transactions через компенсирующие транзакции.**

#### Choreography Saga

**Каждый сервис слушает события и публикует свои.**

```php
<?php

// Order Service: начало саги
class OrderService
{
    public function placeOrder(array $orderData): void
    {
        // 1. Создать заказ
        $order = Order::create($orderData);
        
        // 2. Опубликовать OrderCreated
        Kafka::publish('order_events', [
            'type' => 'OrderCreated',
            'order_id' => $order->id,
            'user_id' => $order->user_id,
            'items' => $order->items,
        ]);
    }
}

// Payment Service: слушает OrderCreated
class PaymentService
{
    public function consumeOrderEvents(): void
    {
        Kafka::consume(['order_events'], function ($event) {
            if ($event['type'] === 'OrderCreated') {
                try {
                    // Обработать платеж
                    $payment = $this->processPayment($event['order_id']);
                    
                    // Успех → PaymentCompleted
                    Kafka::publish('payment_events', [
                        'type' => 'PaymentCompleted',
                        'order_id' => $event['order_id'],
                        'payment_id' => $payment->id,
                    ]);
                    
                } catch (PaymentFailedException $e) {
                    // Ошибка → PaymentFailed
                    Kafka::publish('payment_events', [
                        'type' => 'PaymentFailed',
                        'order_id' => $event['order_id'],
                        'reason' => $e->getMessage(),
                    ]);
                }
            }
        });
    }
}

// Inventory Service: слушает PaymentCompleted
class InventoryService
{
    public function consumePaymentEvents(): void
    {
        Kafka::consume(['payment_events'], function ($event) {
            if ($event['type'] === 'PaymentCompleted') {
                try {
                    // Резервировать товар
                    $this->reserveItems($event['order_id']);
                    
                    Kafka::publish('inventory_events', [
                        'type' => 'ItemsReserved',
                        'order_id' => $event['order_id'],
                    ]);
                    
                } catch (OutOfStockException $e) {
                    // Нет товара → компенсация
                    Kafka::publish('inventory_events', [
                        'type' => 'ReservationFailed',
                        'order_id' => $event['order_id'],
                    ]);
                }
            }
        });
    }
}

// Order Service: слушает PaymentFailed/ReservationFailed для компенсации
class OrderService
{
    public function consumeFailureEvents(): void
    {
        Kafka::consume(['payment_events', 'inventory_events'], function ($event) {
            if (in_array($event['type'], ['PaymentFailed', 'ReservationFailed'])) {
                // Компенсирующая транзакция - отменить заказ
                $order = Order::find($event['order_id']);
                $order->update(['status' => 'cancelled']);
                
                Kafka::publish('order_events', [
                    'type' => 'OrderCancelled',
                    'order_id' => $event['order_id'],
                    'reason' => $event['type'],
                ]);
                
                // Если платеж был успешен, вернуть деньги
                if ($event['type'] === 'ReservationFailed') {
                    Kafka::publish('payment_events', [
                        'type' => 'RefundRequested',
                        'order_id' => $event['order_id'],
                    ]);
                }
            }
        });
    }
}
```

**Saga flow:**
```
OrderCreated
    → PaymentCompleted
        → ItemsReserved
            → OrderCompleted ✅

OR

OrderCreated
    → PaymentFailed
        → OrderCancelled ❌

OR

OrderCreated
    → PaymentCompleted
        → ReservationFailed
            → RefundRequested
                → OrderCancelled ❌
```

#### Orchestration Saga

**Центральный оркестратор управляет сагой.**

```php
<?php

class OrderSagaOrchestrator
{
    private const STEPS = [
        'payment',
        'inventory',
        'shipping',
    ];
    
    public function execute(int $orderId): void
    {
        $saga = Saga::create(['order_id' => $orderId]);
        
        try {
            // 1. Payment
            $this->executePayment($orderId);
            $saga->markCompleted('payment');
            
            // 2. Inventory
            $this->reserveInventory($orderId);
            $saga->markCompleted('inventory');
            
            // 3. Shipping
            $this->createShipment($orderId);
            $saga->markCompleted('shipping');
            
            // Успех
            $this->completeOrder($orderId);
            
        } catch (Exception $e) {
            // Откат выполненных шагов
            $this->compensate($saga, $e);
        }
    }
    
    private function compensate(Saga $saga, Exception $e): void
    {
        $completedSteps = $saga->completed_steps;
        
        // Откат в обратном порядке
        if (in_array('shipping', $completedSteps)) {
            $this->cancelShipment($saga->order_id);
        }
        
        if (in_array('inventory', $completedSteps)) {
            $this->releaseInventory($saga->order_id);
        }
        
        if (in_array('payment', $completedSteps)) {
            $this->refundPayment($saga->order_id);
        }
        
        $this->cancelOrder($saga->order_id, $e->getMessage());
    }
}
```

---

## 🎓 Best Practices

1. **Idempotency** - всегда обрабатывай сообщения идемпотентно
2. **Dead Letter Queues** - для failed messages
3. **Monitoring** - трекай lag, throughput, errors
4. **Schema versioning** - обратная совместимость
5. **Graceful shutdown** - commit offsets перед остановкой
6. **Retry с backoff** - не забивай систему
7. **Circuit breaker** - защита от каскадных сбоев
8. **Message TTL** - для временных данных
9. **Correlation ID** - для трейсинга distributed requests
10. **Saga pattern** - для распределенных транзакций

---

## 🎓 Для собеседования: ключевые точки

1. **Pub-Sub** - fan-out, broadcasting, decoupling
2. **Point-to-Point** - queue, competing consumers, load balancing
3. **Request-Reply** - RPC через MQ, correlation ID
4. **Routing** - topic exchange, headers, content-based
5. **Reliability** - at-least-once, idempotency, DLQ, retry
6. **Saga** - choreography vs orchestration, compensating transactions
7. **Idempotency** - unique constraints, deduplication cache
8. **Circuit breaker** - предотвращение cascade failures
9. **Message transformation** - translator, enricher, normalizer
10. **Kafka vs RabbitMQ** - streaming vs messaging, retention

**Главное:** Понимай когда какой паттерн использовать, идемпотентность критична для at-least-once delivery.
