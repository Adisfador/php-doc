# Apache Kafka

Распределенная платформа потоковой передачи данных для обработки событий в реальном времени.

---

## 🎯 Что такое Kafka

**Apache Kafka** - распределенная система обмена сообщениями (distributed streaming platform) для высокопроизводительной обработки потоков событий.

### Основные возможности

1. **Messaging System** - публикация и подписка на потоки сообщений
2. **Storage System** - надежное хранение потоков событий
3. **Stream Processing** - обработка потоков в реальном времени

### Ключевые характеристики

- **Высокая пропускная способность** - миллионы сообщений в секунду
- **Горизонтальная масштабируемость** - добавление новых брокеров
- **Fault-tolerance** - репликация данных между брокерами
- **Низкая задержка** - обработка в миллисекундах
- **Постоянное хранение** - сообщения сохраняются на диске
- **Гарантии доставки** - at-least-once, at-most-once, exactly-once

---

## 📊 Основные концепции

### Topics (Темы)

**Topic** - категория или поток, в который публикуются сообщения.

```
Topic: user_events

| Partition 0 | Partition 1 | Partition 2 |
|-------------|-------------|-------------|
| msg 0       | msg 1       | msg 2       |
| msg 3       | msg 4       | msg 5       |
| msg 6       | msg 7       | msg 8       |
```

**Характеристики:**
- Имя уникально в кластере
- Multi-subscriber - много потребителей
- Append-only log - добавление в конец
- Retention policy - время хранения сообщений

### Partitions (Разделы)

**Partition** - упорядоченная, неизменяемая последовательность сообщений.

```
Topic: orders (3 partitions)

Partition 0:  [msg0] → [msg3] → [msg6] → [msg9]
              offset: 0     1      2      3

Partition 1:  [msg1] → [msg4] → [msg7] → [msg10]
              offset: 0     1      2      3

Partition 2:  [msg2] → [msg5] → [msg8] → [msg11]
              offset: 0     1      2      3
```

**Зачем partitions:**
- **Параллелизм** - каждый partition обрабатывается независимо
- **Масштабирование** - распределение по разным брокерам
- **Ordering** - гарантия порядка внутри partition
- **Load balancing** - распределение нагрузки

### Brokers (Брокеры)

**Broker** - сервер Kafka, хранящий и обслуживающий данные.

```
Kafka Cluster

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Broker 0   │  │  Broker 1   │  │  Broker 2   │
│             │  │             │  │             │
│ Part 0 (L)  │  │ Part 0 (F)  │  │ Part 1 (L)  │
│ Part 1 (F)  │  │ Part 2 (L)  │  │ Part 0 (F)  │
│ Part 2 (F)  │  │ Part 1 (F)  │  │ Part 2 (F)  │
└─────────────┘  └─────────────┘  └─────────────┘

L = Leader (читает/пишет)
F = Follower (replica)
```

**Характеристики:**
- Каждый broker хранит часть данных
- Leader для partition обрабатывает все чтения/записи
- Followers реплицируют данные
- Zookeeper/KRaft координирует брокеры

### Producers (Производители)

**Producer** - приложение, публикующее сообщения в topics.

```php
// Producer отправляет сообщение
Producer → Topic (orders) → Partition (по ключу)
```

**Ключевые возможности:**
- Выбор partition (по ключу, round-robin, custom)
- Батчинг сообщений
- Compression (gzip, snappy, lz4, zstd)
- Acknowledgments (acks)

### Consumers (Потребители)

**Consumer** - приложение, читающее сообщения из topics.

```
Consumer читает с offset

Partition 0:  [msg0] → [msg1] → [msg2] → [msg3]
                         ↑
                      offset 1
                  (consumer читает msg1)
```

**Характеристики:**
- Pull model - consumer сам забирает сообщения
- Offset tracking - позиция в partition
- Consumer group для параллелизма

### Consumer Groups (Группы потребителей)

**Consumer Group** - набор consumers, совместно потребляющих topic.

```
Topic: orders (3 partitions)

Consumer Group: order-processors

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Consumer 1   │  │ Consumer 2   │  │ Consumer 3   │
│ Partition 0  │  │ Partition 1  │  │ Partition 2  │
└──────────────┘  └──────────────┘  └──────────────┘

Каждый partition назначен ОДНОМУ consumer в группе
```

**Правила:**
- Каждый partition читается только одним consumer в группе
- Если consumers > partitions, лишние простаивают
- Если consumers < partitions, один consumer читает несколько partitions
- Разные groups читают все сообщения независимо

### Offsets (Смещения)

**Offset** - уникальный идентификатор сообщения в partition.

```
Partition 0:
┌──────┬──────┬──────┬──────┬──────┐
│ msg0 │ msg1 │ msg2 │ msg3 │ msg4 │
└──────┴──────┴──────┴──────┴──────┘
offset: 0      1      2      3      4

Consumer commit offset: 3
→ следующее чтение с offset 3 (msg3)
```

**Управление offset:**
- **Auto-commit** - автоматически через интервал
- **Manual commit** - вручную после обработки
- **Stored in Kafka** - в специальном topic `__consumer_offsets`

### Replication (Репликация)

**Replication Factor** - количество копий каждого partition.

```
Topic: payments, replication-factor: 3

Partition 0:
  Broker 0: Leader
  Broker 1: Follower (replica)
  Broker 2: Follower (replica)

Partition 1:
  Broker 1: Leader
  Broker 0: Follower
  Broker 2: Follower
```

**ISR (In-Sync Replicas):**
- Replicas, которые синхронизированы с leader
- Producer может требовать acks от всех ISR
- При падении leader, новый выбирается из ISR

---

## 🆚 Kafka vs RabbitMQ

### Сравнение

| Аспект | Kafka | RabbitMQ |
|--------|-------|----------|
| **Тип** | Distributed streaming platform | Message broker |
| **Модель** | Publish-Subscribe (log-based) | Flexible routing (queue/exchange) |
| **Хранение** | Persistent log (дни/недели/forever) | Сообщение удаляется после ack |
| **Пропускная способность** | Очень высокая (millions/sec) | Средняя (thousands/sec) |
| **Latency** | Низкая (ms) | Очень низкая (microseconds) |
| **Ordering** | Гарантирован в partition | Гарантирован в queue |
| **Consumers** | Pull (consumer забирает) | Push (broker отправляет) |
| **Scalability** | Горизонтальная (partitions) | Ограниченная |
| **Use cases** | Event sourcing, stream processing, logs | Task queues, RPC, routing |
| **Complexity** | Высокая | Средняя |

### Когда Kafka

✅ **Event sourcing** - хранение всех событий домена
✅ **Stream processing** - обработка потоков в реальном времени
✅ **Log aggregation** - централизованные логи приложений
✅ **Metrics collection** - сбор метрик с множества источников
✅ **CDC (Change Data Capture)** - отслеживание изменений в БД
✅ **High throughput** - миллионы сообщений в секунду
✅ **Long retention** - нужно хранить сообщения долго
✅ **Replay events** - переобработка событий

### Когда RabbitMQ

✅ **Task queues** - распределение задач между workers
✅ **RPC (Request-Reply)** - синхронная коммуникация
✅ **Complex routing** - маршрутизация по правилам
✅ **Priority queues** - обработка по приоритету
✅ **Low latency** - критична минимальная задержка
✅ **Simple use cases** - CRUD операции, простые очереди
✅ **Message TTL** - сообщения с временем жизни
✅ **Dead letter queues** - обработка failed messages

---

## 🔧 PHP и Kafka

### Библиотеки

**1. php-rdkafka (рекомендуется)**

```bash
# Установка расширения
pecl install rdkafka

# php.ini
extension=rdkafka.so
```

**2. Composer packages**

```bash
composer require kwn/php-rdkafka
# или
composer require enqueue/rdkafka
```

### Producer - базовый пример

```php
<?php

use RdKafka\Producer;
use RdKafka\ProducerTopic;
use RdKafka\Conf;

// Конфигурация
$conf = new Conf();
$conf->set('metadata.broker.list', 'localhost:9092');
$conf->set('client.id', 'php-producer');

// Создать producer
$producer = new Producer($conf);

// Topic
$topic = $producer->newTopic('user_events');

// Отправить сообщение
$message = json_encode([
    'user_id' => 123,
    'event' => 'user_registered',
    'timestamp' => time(),
]);

$topic->produce(
    RD_KAFKA_PARTITION_UA,  // partition: UA = unassigned (auto)
    0,                       // msgflags
    $message,                // payload
    'user:123'               // key (для партиционирования)
);

// Poll для callbacks и доставки
$producer->poll(0);

// Flush - дождаться доставки всех сообщений
$producer->flush(10000);  // timeout 10 секунд

echo "Message sent!\n";
```

### Producer с partition key

```php
<?php

// Сообщения с одинаковым ключом попадут в один partition
// → гарантия порядка для событий одного пользователя

$users = [123, 456, 789];

foreach ($users as $userId) {
    $message = json_encode([
        'user_id' => $userId,
        'action' => 'login',
        'timestamp' => time(),
    ]);
    
    // Key = user_id → все события юзера в одном partition
    $topic->produce(
        RD_KAFKA_PARTITION_UA,
        0,
        $message,
        "user:{$userId}"  // ключ партиционирования
    );
}

$producer->poll(0);
$producer->flush(10000);
```

### Producer с delivery callback

```php
<?php

// Callback при доставке/ошибке
$conf->setDrMsgCb(function ($kafka, $message) {
    if ($message->err) {
        echo "Delivery failed: {$message->errstr()}\n";
    } else {
        echo "Message delivered to partition {$message->partition}, offset {$message->offset}\n";
    }
});

$producer = new Producer($conf);
$topic = $producer->newTopic('orders');

$topic->produce(RD_KAFKA_PARTITION_UA, 0, 'Order #123', 'order:123');

// Poll для вызова callbacks
for ($i = 0; $i < 10; $i++) {
    $producer->poll(100);
}
```

### Consumer - базовый пример

```php
<?php

use RdKafka\Consumer;
use RdKafka\ConsumerTopic;
use RdKafka\Conf;

// Конфигурация
$conf = new Conf();
$conf->set('metadata.broker.list', 'localhost:9092');
$conf->set('group.id', 'php-consumer-group');
$conf->set('auto.offset.reset', 'earliest');  // earliest | latest

// Создать consumer
$consumer = new Consumer($conf);

// Topic
$topic = $consumer->newTopic('user_events');

// Начать чтение с partition 0, offset stored
$topic->consumeStart(0, RD_KAFKA_OFFSET_STORED);

echo "Waiting for messages...\n";

while (true) {
    // Poll сообщение с timeout 1 секунда
    $message = $topic->consume(0, 1000);
    
    switch ($message->err) {
        case RD_KAFKA_RESP_ERR_NO_ERROR:
            // Успешно получено сообщение
            echo "Received: {$message->payload}\n";
            echo "Partition: {$message->partition}, Offset: {$message->offset}\n";
            
            // Обработать сообщение
            processMessage(json_decode($message->payload, true));
            
            break;
            
        case RD_KAFKA_RESP_ERR__PARTITION_EOF:
            // Достигнут конец partition (нет новых сообщений)
            echo "End of partition reached\n";
            break;
            
        case RD_KAFKA_RESP_ERR__TIMED_OUT:
            // Timeout - нет сообщений
            echo "Timeout\n";
            break;
            
        default:
            // Ошибка
            throw new Exception($message->errstr(), $message->err);
    }
}

function processMessage(array $event): void
{
    echo "Processing: {$event['event']} for user {$event['user_id']}\n";
}
```

### High-Level Consumer (рекомендуется)

```php
<?php

use RdKafka\KafkaConsumer;
use RdKafka\Conf;

// Конфигурация
$conf = new Conf();
$conf->set('metadata.broker.list', 'localhost:9092');
$conf->set('group.id', 'my-consumer-group');
$conf->set('auto.offset.reset', 'earliest');
$conf->set('enable.auto.commit', 'false');  // manual commit

// Создать high-level consumer
$consumer = new KafkaConsumer($conf);

// Подписаться на topics (можно несколько)
$consumer->subscribe(['user_events', 'order_events']);

echo "Waiting for messages...\n";

while (true) {
    $message = $consumer->consume(1000);  // timeout 1 сек
    
    switch ($message->err) {
        case RD_KAFKA_RESP_ERR_NO_ERROR:
            // Обработать сообщение
            $data = json_decode($message->payload, true);
            processEvent($data);
            
            // Commit offset после успешной обработки
            $consumer->commit($message);
            
            echo "Processed: {$message->topic} [{$message->partition}] offset {$message->offset}\n";
            break;
            
        case RD_KAFKA_RESP_ERR__PARTITION_EOF:
            echo "No more messages\n";
            break;
            
        case RD_KAFKA_RESP_ERR__TIMED_OUT:
            echo "Timed out\n";
            break;
            
        default:
            throw new Exception($message->errstr(), $message->err);
    }
}

function processEvent(array $data): void
{
    // Бизнес-логика
    echo "Event: {$data['event']}\n";
}
```

---

## 🎨 Laravel интеграция

### Laravel Queue Driver

```bash
composer require enqueue/laravel-queue
```

**config/queue.php:**

```php
'connections' => [
    'kafka' => [
        'driver' => 'kafka',
        'queue' => 'default',
        'brokers' => env('KAFKA_BROKERS', 'localhost:9092'),
        'group_id' => env('KAFKA_CONSUMER_GROUP', 'laravel'),
        'consumer' => [
            'enable.auto.commit' => 'false',
            'auto.offset.reset' => 'earliest',
        ],
    ],
],
```

**Job dispatch:**

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class ProcessUserEvent implements ShouldQueue
{
    use InteractsWithQueue, Queueable;
    
    public function __construct(
        public int $userId,
        public string $event,
    ) {}
    
    public function handle(): void
    {
        // Обработка события
        echo "Processing {$this->event} for user {$this->userId}\n";
    }
}

// Dispatch на Kafka
ProcessUserEvent::dispatch(123, 'user_registered')
    ->onQueue('user_events');
```

**Worker:**

```bash
php artisan queue:work kafka --queue=user_events
```

### Laravel Kafka Package (mateusjunges/laravel-kafka)

```bash
composer require mateusjunges/laravel-kafka
```

**Publish:**

```php
<?php

use Junges\Kafka\Facades\Kafka;
use Junges\Kafka\Message\Message;

// Простая публикация
Kafka::publishOn('user_events')
    ->withMessage(new Message(
        body: ['user_id' => 123, 'event' => 'login']
    ))
    ->send();

// С ключом (партиционирование)
Kafka::publishOn('orders')
    ->withMessage(
        new Message(
            body: ['order_id' => 456, 'amount' => 100],
            key: 'order:456'
        )
    )
    ->send();

// Batch
$messages = [
    new Message(body: ['user_id' => 1]),
    new Message(body: ['user_id' => 2]),
    new Message(body: ['user_id' => 3]),
];

Kafka::publishOn('user_events')
    ->withMessages($messages)
    ->send();
```

**Consume:**

```php
<?php

use Junges\Kafka\Facades\Kafka;
use Junges\Kafka\Contracts\KafkaConsumerMessage;

$consumer = Kafka::createConsumer(['user_events'])
    ->withConsumerGroupId('my-group')
    ->withHandler(function (KafkaConsumerMessage $message) {
        $data = $message->getBody();
        
        // Обработать сообщение
        processEvent($data);
    })
    ->build();

$consumer->consume();
```

**Consumer в команде:**

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Junges\Kafka\Facades\Kafka;

class ConsumeUserEvents extends Command
{
    protected $signature = 'kafka:consume-users';
    
    public function handle(): void
    {
        $this->info('Starting Kafka consumer...');
        
        Kafka::createConsumer(['user_events'])
            ->withConsumerGroupId('laravel-consumers')
            ->withAutoCommit()
            ->withHandler(function ($message) {
                $data = $message->getBody();
                $this->info("Event: {$data['event']}");
                
                // Dispatch Laravel job
                ProcessUserEvent::dispatch($data['user_id'], $data['event']);
            })
            ->build()
            ->consume();
    }
}
```

---

## 🎯 Use Cases

### 1. Event Sourcing

**Хранение всех доменных событий в Kafka.**

```php
<?php

// Domain Events
class UserRegistered
{
    public function __construct(
        public int $userId,
        public string $email,
        public DateTime $occurredAt,
    ) {}
}

class OrderPlaced
{
    public function __construct(
        public int $orderId,
        public int $userId,
        public float $amount,
        public DateTime $occurredAt,
    ) {}
}

// Event Store (Kafka)
class KafkaEventStore
{
    private ProducerTopic $topic;
    
    public function __construct(private Producer $producer)
    {
        $this->topic = $producer->newTopic('domain_events');
    }
    
    public function append(object $event): void
    {
        $message = json_encode([
            'type' => get_class($event),
            'data' => serialize($event),
            'occurred_at' => time(),
        ]);
        
        // Ключ = aggregate ID для ordering
        $key = $this->extractAggregateId($event);
        
        $this->topic->produce(
            RD_KAFKA_PARTITION_UA,
            0,
            $message,
            $key
        );
        
        $this->producer->flush(5000);
    }
    
    private function extractAggregateId(object $event): string
    {
        if ($event instanceof UserRegistered) {
            return "user:{$event->userId}";
        }
        if ($event instanceof OrderPlaced) {
            return "order:{$event->orderId}";
        }
        return 'unknown';
    }
}

// Использование
$eventStore = new KafkaEventStore($producer);

$event = new UserRegistered(
    userId: 123,
    email: 'user@example.com',
    occurredAt: new DateTime()
);

$eventStore->append($event);  // В Kafka навсегда
```

**Replay events - восстановление состояния:**

```php
<?php

class UserProjection
{
    private array $users = [];
    
    public function rebuild(KafkaConsumer $consumer): void
    {
        // Читать с начала topic
        $consumer->subscribe(['domain_events']);
        
        while (true) {
            $message = $consumer->consume(1000);
            
            if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
                $event = json_decode($message->payload, true);
                $this->apply($event);
            }
        }
    }
    
    private function apply(array $event): void
    {
        switch ($event['type']) {
            case 'UserRegistered':
                $data = unserialize($event['data']);
                $this->users[$data->userId] = [
                    'email' => $data->email,
                    'registered_at' => $data->occurredAt,
                ];
                break;
        }
    }
}
```

### 2. CQRS (Command Query Responsibility Segregation)

**Write model пишет в БД и публикует события в Kafka. Query model подписан на Kafka и обновляет read-модель.**

```php
<?php

// Write Side (Command Handler)
class CreateOrderHandler
{
    public function __construct(
        private OrderRepository $repository,
        private Producer $kafka,
    ) {}
    
    public function handle(CreateOrderCommand $command): void
    {
        // 1. Создать Order в write DB
        $order = Order::create([
            'user_id' => $command->userId,
            'items' => $command->items,
            'total' => $command->total,
        ]);
        
        $this->repository->save($order);
        
        // 2. Опубликовать событие в Kafka
        $event = [
            'type' => 'OrderCreated',
            'order_id' => $order->id,
            'user_id' => $order->user_id,
            'total' => $order->total,
            'created_at' => $order->created_at,
        ];
        
        $topic = $this->kafka->newTopic('order_events');
        $topic->produce(
            RD_KAFKA_PARTITION_UA,
            0,
            json_encode($event),
            "order:{$order->id}"
        );
        
        $this->kafka->flush(5000);
    }
}

// Read Side (Consumer)
class OrderQueryModelUpdater
{
    public function consume(KafkaConsumer $consumer): void
    {
        $consumer->subscribe(['order_events']);
        
        while (true) {
            $message = $consumer->consume(1000);
            
            if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
                $event = json_decode($message->payload, true);
                
                if ($event['type'] === 'OrderCreated') {
                    // Обновить read model (Redis, Elasticsearch, денормализованная таблица)
                    Redis::hset("user:{$event['user_id']}:orders", $event['order_id'], json_encode($event));
                    
                    // Обновить счетчик
                    Redis::incr("user:{$event['user_id']}:order_count");
                }
                
                $consumer->commit($message);
            }
        }
    }
}
```

### 3. Log Aggregation

**Централизованное хранение логов с множества приложений.**

```php
<?php

// Monolog Handler для Kafka
use Monolog\Logger;
use Monolog\Handler\AbstractProcessingHandler;

class KafkaHandler extends AbstractProcessingHandler
{
    private ProducerTopic $topic;
    
    public function __construct(
        Producer $producer,
        string $topic = 'application_logs',
        int $level = Logger::DEBUG
    ) {
        parent::__construct($level);
        $this->topic = $producer->newTopic($topic);
    }
    
    protected function write(array $record): void
    {
        $message = json_encode([
            'app' => 'my-app',
            'level' => $record['level_name'],
            'message' => $record['message'],
            'context' => $record['context'],
            'timestamp' => $record['datetime']->format('Y-m-d H:i:s'),
        ]);
        
        $this->topic->produce(
            RD_KAFKA_PARTITION_UA,
            0,
            $message
        );
    }
}

// Laravel config/logging.php
'channels' => [
    'kafka' => [
        'driver' => 'monolog',
        'handler' => KafkaHandler::class,
        'with' => [
            'producer' => app(Producer::class),
            'topic' => 'laravel_logs',
        ],
    ],
],

// Использование
Log::channel('kafka')->info('User registered', ['user_id' => 123]);
```

### 4. Real-time Analytics

**Потоковая обработка событий пользователей.**

```php
<?php

// Producer: отправка событий аналитики
class AnalyticsTracker
{
    public function __construct(private Producer $kafka) {}
    
    public function track(string $event, array $properties): void
    {
        $message = json_encode([
            'event' => $event,
            'properties' => $properties,
            'timestamp' => microtime(true),
        ]);
        
        $topic = $this->kafka->newTopic('analytics_events');
        $topic->produce(RD_KAFKA_PARTITION_UA, 0, $message);
        $this->kafka->poll(0);
    }
}

// Использование
$tracker = new AnalyticsTracker($producer);

$tracker->track('page_view', [
    'user_id' => 123,
    'page' => '/products/laptop',
    'referrer' => 'google.com',
]);

$tracker->track('purchase', [
    'user_id' => 123,
    'product_id' => 456,
    'amount' => 999.99,
]);

// Consumer: агрегация в реальном времени
class RealTimeStatsAggregator
{
    private array $stats = [];
    
    public function consume(KafkaConsumer $consumer): void
    {
        $consumer->subscribe(['analytics_events']);
        
        while (true) {
            $message = $consumer->consume(100);
            
            if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
                $event = json_decode($message->payload, true);
                
                // Агрегировать в памяти (или Redis)
                $minute = date('Y-m-d H:i', $event['timestamp']);
                $this->stats[$minute][$event['event']] ??= 0;
                $this->stats[$minute][$event['event']]++;
                
                // Каждые 60 секунд - сохранить в БД
                $this->flushIfNeeded();
            }
        }
    }
}
```

### 5. CDC (Change Data Capture)

**Отслеживание изменений в БД через Kafka Connect + Debezium.**

```yaml
# Debezium MySQL Connector
{
  "name": "mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "localhost",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "secret",
    "database.server.id": "184054",
    "database.server.name": "mydb",
    "table.include.list": "mydb.users,mydb.orders",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes"
  }
}
```

**Consumer изменений:**

```php
<?php

// Kafka получает CDC события от Debezium
$consumer = new KafkaConsumer($conf);
$consumer->subscribe(['mydb.users', 'mydb.orders']);

while (true) {
    $message = $consumer->consume(1000);
    
    if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
        $change = json_decode($message->payload, true);
        
        // Debezium формат
        $operation = $change['op'];  // c = create, u = update, d = delete
        $before = $change['before'] ?? null;
        $after = $change['after'] ?? null;
        
        match ($operation) {
            'c' => handleInsert($after),
            'u' => handleUpdate($before, $after),
            'd' => handleDelete($before),
        };
        
        $consumer->commit($message);
    }
}

function handleInsert(array $data): void
{
    // Синхронизировать в Elasticsearch
    Elasticsearch::index('users', $data['id'], $data);
}

function handleUpdate(array $before, array $after): void
{
    Elasticsearch::update('users', $after['id'], $after);
}

function handleDelete(array $data): void
{
    Elasticsearch::delete('users', $data['id']);
}
```

### 6. Microservices Communication

**Асинхронная коммуникация между микросервисами.**

```php
<?php

// Order Service: публикует событие
class OrderService
{
    public function placeOrder(PlaceOrderRequest $request): Order
    {
        // 1. Создать заказ
        $order = Order::create([
            'user_id' => $request->userId,
            'items' => $request->items,
            'total' => $request->total,
        ]);
        
        // 2. Опубликовать событие OrderPlaced
        $this->kafka->publish('order_events', [
            'type' => 'OrderPlaced',
            'order_id' => $order->id,
            'user_id' => $order->user_id,
            'total' => $order->total,
        ]);
        
        return $order;
    }
}

// Payment Service: слушает OrderPlaced
class PaymentService
{
    public function consumeOrders(): void
    {
        $consumer = Kafka::createConsumer(['order_events'])
            ->withHandler(function ($message) {
                $event = $message->getBody();
                
                if ($event['type'] === 'OrderPlaced') {
                    // Обработать платеж
                    $payment = $this->processPayment($event['order_id'], $event['total']);
                    
                    // Опубликовать событие PaymentProcessed
                    Kafka::publishOn('payment_events')
                        ->withMessage(['type' => 'PaymentProcessed', 'order_id' => $event['order_id']])
                        ->send();
                }
            })
            ->build();
            
        $consumer->consume();
    }
}

// Inventory Service: слушает PaymentProcessed
class InventoryService
{
    public function consumePayments(): void
    {
        $consumer = Kafka::createConsumer(['payment_events'])
            ->withHandler(function ($message) {
                $event = $message->getBody();
                
                if ($event['type'] === 'PaymentProcessed') {
                    // Зарезервировать товар
                    $this->reserveItems($event['order_id']);
                }
            })
            ->build();
            
        $consumer->consume();
    }
}
```

---

## 📋 Topics и Partitions

### Создание Topic

```bash
# CLI
kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --topic user_events \
    --partitions 3 \
    --replication-factor 2

# Параметры:
# --partitions - количество партиций (параллелизм)
# --replication-factor - копии данных (fault tolerance)
```

**Выбор количества partitions:**

```
Partitions = max(throughput, consumers)

# Пример:
# Throughput: 100 MB/s
# Consumer throughput: 10 MB/s
# Partitions >= 100 / 10 = 10

# Рекомендации:
# - Начать с 3-5 partitions
# - Можно увеличить (нельзя уменьшить!)
# - Слишком много partitions → overhead
```

### Partition Strategy

**1. Key-based (default):**

```php
// Сообщения с одинаковым ключом → один partition
$topic->produce(RD_KAFKA_PARTITION_UA, 0, $message, "user:123");
$topic->produce(RD_KAFKA_PARTITION_UA, 0, $message, "user:123");  // → тот же partition

// Hash(key) % partitions_count = partition_number
// "user:123" → hash → partition 1
```

**2. Round-robin (без ключа):**

```php
// Нет ключа → round-robin распределение
$topic->produce(RD_KAFKA_PARTITION_UA, 0, $message1, null);  // → partition 0
$topic->produce(RD_KAFKA_PARTITION_UA, 0, $message2, null);  // → partition 1
$topic->produce(RD_KAFKA_PARTITION_UA, 0, $message3, null);  // → partition 2
```

**3. Custom partitioner:**

```php
// Указать конкретный partition
$topic->produce(
    2,  // конкретный partition ID
    0,
    $message
);
```

### Partition Key Best Practices

```php
<?php

// ✅ Хорошие ключи партиционирования

// 1. User ID - события пользователя упорядочены
$key = "user:{$userId}";

// 2. Order ID - события заказа упорядочены
$key = "order:{$orderId}";

// 3. Device ID - события устройства
$key = "device:{$deviceId}";

// 4. Tenant ID (multi-tenancy)
$key = "tenant:{$tenantId}";

// ❌ Плохие ключи

// 1. Timestamp - все сообщения в один partition
$key = time();

// 2. Random - нет ordering
$key = uniqid();

// 3. Constant - все в один partition
$key = 'constant';
```

---

## 👥 Consumer Groups

### Scaling Consumers

```
Topic: orders (4 partitions)

Scenario 1: 2 consumers in group
┌──────────────────────┐
│ Consumer 1           │
│ Partition 0, 1       │
└──────────────────────┘
┌──────────────────────┐
│ Consumer 2           │
│ Partition 2, 3       │
└──────────────────────┘

Scenario 2: 4 consumers in group (optimal)
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ C1   │ │ C2   │ │ C3   │ │ C4   │
│ P0   │ │ P1   │ │ P2   │ │ P3   │
└──────┘ └──────┘ └──────┘ └──────┘

Scenario 3: 6 consumers (2 idle!)
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ C1   │ │ C2   │ │ C3   │ │ C4   │ │ C5   │ │ C6   │
│ P0   │ │ P1   │ │ P2   │ │ P3   │ │ idle │ │ idle │
└──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘
```

**Правило:** consumers <= partitions для эффективности.

### Offset Management

**Auto-commit (по умолчанию):**

```php
$conf->set('enable.auto.commit', 'true');
$conf->set('auto.commit.interval.ms', '5000');  // каждые 5 секунд

// Риск: сообщение committed до обработки → потеря при сбое
```

**Manual commit (рекомендуется):**

```php
$conf->set('enable.auto.commit', 'false');

$consumer = new KafkaConsumer($conf);
$consumer->subscribe(['orders']);

while (true) {
    $message = $consumer->consume(1000);
    
    if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
        try {
            // Обработать
            processOrder($message->payload);
            
            // Commit ПОСЛЕ успешной обработки
            $consumer->commit($message);
            
        } catch (Exception $e) {
            // Ошибка - НЕ commit, переобработаем
            Log::error("Failed to process: {$e->getMessage()}");
        }
    }
}
```

**Commit strategies:**

```php
// 1. Commit каждое сообщение (медленно, но безопасно)
$consumer->commit($message);

// 2. Commit batch (быстрее, риск потери batch)
$processed = 0;
while (true) {
    $message = $consumer->consume(1000);
    process($message);
    $processed++;
    
    if ($processed >= 100) {
        $consumer->commit($message);  // commit каждые 100 сообщений
        $processed = 0;
    }
}

// 3. Commit с async (не ждать подтверждения)
$consumer->commitAsync($message);
```

### Offset Reset

```php
// auto.offset.reset - что делать если нет сохраненного offset

// earliest - читать с начала topic
$conf->set('auto.offset.reset', 'earliest');

// latest - читать только новые сообщения (default)
$conf->set('auto.offset.reset', 'latest');

// none - ошибка если нет offset
$conf->set('auto.offset.reset', 'none');
```

### Rebalancing

**Происходит когда:**
- Consumer присоединяется к группе
- Consumer покидает группу (graceful или crash)
- Partitions добавлены в topic

```
Before rebalance:
C1 → P0, P1
C2 → P2, P3

[C3 joins group]
→ Rebalancing...

After rebalance:
C1 → P0
C2 → P1, P2
C3 → P3
```

**Graceful shutdown:**

```php
$running = true;

pcntl_signal(SIGTERM, function () use (&$running) {
    $running = false;
});

while ($running) {
    $message = $consumer->consume(1000);
    // process...
}

// Graceful shutdown
$consumer->close();  // triggers rebalance
```

---

## ⚙️ Конфигурация

### Producer Configuration

```php
$conf = new Conf();

// Brokers
$conf->set('metadata.broker.list', 'localhost:9092');

// Acknowledgments (важно!)
$conf->set('acks', 'all');  // all | 1 | 0
// all - wait for all ISR (самое надежное, медленнее)
// 1 - wait for leader (default)
// 0 - fire and forget (быстро, ненадежно)

// Retries
$conf->set('retries', '3');
$conf->set('retry.backoff.ms', '100');

// Compression (экономия bandwidth)
$conf->set('compression.type', 'snappy');  // none | gzip | snappy | lz4 | zstd

// Batching (производительность)
$conf->set('batch.size', '16384');  // bytes
$conf->set('linger.ms', '10');  // wait 10ms for batching

// Idempotence (exactly-once semantics)
$conf->set('enable.idempotence', 'true');
```

### Consumer Configuration

```php
$conf = new Conf();

// Brokers
$conf->set('metadata.broker.list', 'localhost:9092');

// Consumer group
$conf->set('group.id', 'my-consumer-group');

// Offset management
$conf->set('enable.auto.commit', 'false');  // manual commit
$conf->set('auto.offset.reset', 'earliest');

// Session timeout (broker считает consumer мертвым)
$conf->set('session.timeout.ms', '30000');  // 30 sec

// Heartbeat
$conf->set('heartbeat.interval.ms', '3000');  // 3 sec

// Max poll interval (время между poll)
$conf->set('max.poll.interval.ms', '300000');  // 5 min

// Fetch size
$conf->set('fetch.min.bytes', '1');
$conf->set('fetch.max.wait.ms', '500');
```

### Broker Configuration

**server.properties:**

```properties
# Cluster
broker.id=0
listeners=PLAINTEXT://localhost:9092
zookeeper.connect=localhost:2181

# Partitions
num.partitions=3
default.replication.factor=2

# Logs
log.dirs=/var/kafka-logs
log.retention.hours=168  # 7 days
log.retention.bytes=-1  # unlimited
log.segment.bytes=1073741824  # 1GB

# Compression
compression.type=producer  # inherit from producer

# ISR
min.insync.replicas=1
```

---

## 🎬 Kafka Streams

**Lightweight библиотека для обработки потоков (Java/Scala, нет для PHP).**

Альтернативы для PHP:
1. Собственная обработка с Consumer
2. Apache Flink (PHP клиент через REST API)
3. ksqlDB (SQL over Kafka streams)

**Пример обработки в PHP:**

```php
<?php

// Stateful stream processing в PHP
class WordCountProcessor
{
    private array $counts = [];
    
    public function process(KafkaConsumer $consumer): void
    {
        $consumer->subscribe(['text_stream']);
        
        while (true) {
            $message = $consumer->consume(1000);
            
            if ($message->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
                $text = $message->payload;
                $words = explode(' ', strtolower($text));
                
                foreach ($words as $word) {
                    $this->counts[$word] = ($this->counts[$word] ?? 0) + 1;
                }
                
                // Publish results
                $this->publishResults();
                $consumer->commit($message);
            }
        }
    }
    
    private function publishResults(): void
    {
        // Publish top 10 words to output topic
        arsort($this->counts);
        $top10 = array_slice($this->counts, 0, 10, true);
        
        // ... publish to output_topic
    }
}
```

---

## 🔌 Kafka Connect

**Framework для интеграции Kafka с внешними системами.**

### Source Connectors (БД → Kafka)

- **Debezium** (MySQL, PostgreSQL, MongoDB CDC)
- **JDBC Source** (любая SQL БД)
- **File Source** (чтение файлов)

### Sink Connectors (Kafka → БД/хранилище)

- **JDBC Sink** (любая SQL БД)
- **Elasticsearch Sink**
- **S3 Sink** (AWS S3)
- **Redis Sink**

**Пример: MySQL → Kafka → Elasticsearch**

```json
{
  "name": "mysql-source",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "secret",
    "database.server.name": "mydb",
    "table.include.list": "mydb.products",
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "$3"
  }
}

{
  "name": "elasticsearch-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "connection.url": "http://elasticsearch:9200",
    "topics": "products",
    "type.name": "_doc",
    "key.ignore": "false"
  }
}
```

---

## 📊 Мониторинг

### Метрики Producer

```php
// Kafka metrics через JMX или rdkafka metadata
$metadata = $producer->getMetadata(true, null, 5000);

foreach ($metadata->getBrokers() as $broker) {
    echo "Broker {$broker->getId()}: {$broker->getHost()}:{$broker->getPort()}\n";
}

foreach ($metadata->getTopics() as $topic) {
    echo "Topic {$topic->getTopic()}: {$topic->getPartitions()->count()} partitions\n";
}
```

### Ключевые метрики

**Broker:**
- `UnderReplicatedPartitions` - партиции без достаточного количества реплик
- `OfflinePartitionsCount` - недоступные партиции
- `ActiveControllerCount` - должен быть 1
- `RequestHandlerAvgIdlePercent` - нагрузка на broker

**Producer:**
- `record-send-rate` - сообщений в секунду
- `record-error-rate` - ошибки отправки
- `request-latency-avg` - средняя задержка

**Consumer:**
- `records-lag` - отставание consumer от конца partition
- `records-consumed-rate` - сообщений в секунду
- `commit-latency-avg` - задержка commit

### Kafka Manager / CMAK

```bash
# UI для управления Kafka
docker run -d \
  -p 9000:9000 \
  -e ZK_HOSTS="zookeeper:2181" \
  hlebalbau/kafka-manager:stable
```

### Prometheus + Grafana

**JMX Exporter для Kafka:**

```yaml
# kafka-jmx-exporter.yml
lowercaseOutputName: true
rules:
  - pattern: kafka.server<type=(.+), name=(.+)><>Value
    name: kafka_server_$1_$2
```

**Grafana dashboard:** ID 721 (Kafka Overview)

---

## 🎓 Best Practices

### 1. Partition Keys

```php
// ✅ DO: Используй ключи для ordering
$topic->produce(RD_KAFKA_PARTITION_UA, 0, $message, "user:{$userId}");

// ❌ DON'T: Случайные ключи
$topic->produce(RD_KAFKA_PARTITION_UA, 0, $message, uniqid());
```

### 2. Idempotent Consumers

```php
// Обрабатывай сообщения идемпотентно (повторная обработка = тот же результат)

// ✅ DO: Upsert с уникальным ключом
DB::table('events')->updateOrInsert(
    ['event_id' => $message->key],  // unique constraint
    ['data' => $message->payload]
);

// ❌ DON'T: Increment без проверки
DB::table('counters')->increment('count');  // дублирование при retry!
```

### 3. Error Handling

```php
// ✅ DO: Retry с exponential backoff
$retries = 0;
$maxRetries = 3;

while ($retries < $maxRetries) {
    try {
        processMessage($message);
        $consumer->commit($message);
        break;
    } catch (Exception $e) {
        $retries++;
        $delay = pow(2, $retries) * 100;  // 200ms, 400ms, 800ms
        usleep($delay * 1000);
        
        if ($retries >= $maxRetries) {
            // Dead Letter Queue
            $this->sendToDLQ($message, $e);
        }
    }
}
```

### 4. Schema Evolution

```php
// Используй schema registry (Confluent Schema Registry)

// v1
$schema = [
    'user_id' => 'int',
    'name' => 'string',
];

// v2 - добавили поле (backward compatible)
$schema = [
    'user_id' => 'int',
    'name' => 'string',
    'email' => 'string',  // новое поле с default
];

// Consumer v1 игнорирует email
// Consumer v2 читает email
```

### 5. Monitoring и Alerting

```php
// Трекай consumer lag
$lag = $this->getConsumerLag($topic, $partition);

if ($lag > 10000) {
    Alert::send("High lag: {$lag} messages behind");
}

// Мониторь failed messages
if ($failedCount > 100) {
    Alert::send("Too many failed messages");
}
```

### 6. Graceful Shutdown

```php
declare(ticks = 1);

$running = true;

pcntl_signal(SIGTERM, function () use (&$running) {
    echo "Shutting down gracefully...\n";
    $running = false;
});

while ($running) {
    $message = $consumer->consume(1000);
    // process...
}

$consumer->close();  // commit offsets, leave group
echo "Shutdown complete\n";
```

---

## 🎓 Для собеседования: ключевые точки

1. **Kafka = distributed streaming platform** с высокой пропускной способностью
2. **Topics и Partitions** - параллелизм и ordering
3. **Consumer Groups** - масштабирование consumers, load balancing
4. **Offsets** - позиция в partition, commit strategies
5. **Replication** - fault tolerance через leader/follower
6. **Kafka vs RabbitMQ** - streaming vs messaging, retention, throughput
7. **Use cases** - event sourcing, CQRS, log aggregation, CDC, real-time analytics
8. **PHP integration** - rdkafka extension, Laravel packages
9. **Idempotency** - обработка сообщений должна быть идемпотентной
10. **At-least-once delivery** - сообщения могут дублироваться (commit после обработки)

**Главное:** Понимай partition key для ordering, consumer groups для scaling, manual commit для надежности.
