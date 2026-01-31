# Laravel Queues & Jobs - Асинхронная обработка задач

Очереди позволяют отложить выполнение времязатратных задач (отправка email, обработка изображений, API запросы) на потом, что ускоряет отклик приложения.

---

## Зачем нужны очереди?

### Без очередей

```php
public function register(Request $request)
{
    $user = User::create($request->all());
    
    // Синхронное выполнение - пользователь ждет!
    Mail::to($user)->send(new WelcomeEmail()); // 2 секунды
    $this->resizeAvatar($user->avatar); // 3 секунды
    $this->notifySlack("New user: {$user->email}"); // 1 секунда
    
    return response()->json($user); // Ответ через 6 секунд!
}
```

### С очередями

```php
public function register(Request $request)
{
    $user = User::create($request->all());
    
    // Асинхронное выполнение - в очередь!
    SendWelcomeEmail::dispatch($user);
    ResizeAvatar::dispatch($user->avatar);
    NotifySlack::dispatch("New user: {$user->email}");
    
    return response()->json($user); // Ответ мгновенно!
}
```

---

## Создание Job

### Генерация

```bash
php artisan make:job SendWelcomeEmail
```

### Базовая структура

```php
<?php

namespace App\Jobs;

use App\Models\User;
use App\Mail\WelcomeEmail;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public User $user
    ) {}

    public function handle(): void
    {
        // Основная логика джоба
        Mail::to($this->user)->send(new WelcomeEmail($this->user));
    }
}
```

### Отправка в очередь

```php
// 1. Dispatch (рекомендуется)
SendWelcomeEmail::dispatch($user);

// 2. Dispatch if (условно)
SendWelcomeEmail::dispatchIf($condition, $user);
SendWelcomeEmail::dispatchUnless($condition, $user);

// 3. Dispatch синхронно (для тестов)
SendWelcomeEmail::dispatchSync($user);

// 4. Dispatch после database commit
SendWelcomeEmail::dispatch($user)->afterCommit();

// 5. Chain (цепочка джобов)
SendWelcomeEmail::dispatch($user)
    ->chain([
        new ProcessAvatar($user),
        new NotifyAdmin($user),
    ]);
```

---

## Конфигурация Queue

### Драйверы очередей

**config/queue.php:**

```php
return [
    'default' => env('QUEUE_CONNECTION', 'sync'),

    'connections' => [
        // Синхронно (без очереди, для разработки)
        'sync' => [
            'driver' => 'sync',
        ],

        // Database (простой, но медленный)
        'database' => [
            'driver' => 'database',
            'table' => 'jobs',
            'queue' => 'default',
            'retry_after' => 90,
            'after_commit' => false,
        ],

        // Redis (рекомендуется для production)
        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
            'queue' => env('REDIS_QUEUE', 'default'),
            'retry_after' => 90,
            'block_for' => null,
            'after_commit' => false,
        ],

        // SQS (AWS)
        'sqs' => [
            'driver' => 'sqs',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'prefix' => env('SQS_PREFIX', 'https://sqs.us-east-1.amazonaws.com/your-account-id'),
            'queue' => env('SQS_QUEUE', 'default'),
            'suffix' => env('SQS_SUFFIX'),
            'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
        ],
    ],
];
```

### Миграция для database драйвера

```bash
php artisan queue:table
php artisan migrate
```

---

## Queue Priority (Приоритеты)

### Несколько очередей

```php
// Критические задачи
SendWelcomeEmail::dispatch($user)->onQueue('high');

// Обычные задачи
ProcessReport::dispatch($data)->onQueue('default');

// Фоновые задачи
CleanupOldFiles::dispatch()->onQueue('low');
```

### Запуск worker с приоритетами

```bash
# Обрабатывает high, потом default, потом low
php artisan queue:work --queue=high,default,low

# Только high очередь
php artisan queue:work --queue=high

# Несколько worker'ов
php artisan queue:work --queue=high &
php artisan queue:work --queue=default &
php artisan queue:work --queue=low &
```

---

## Job Middleware

### Rate Limiting

```php
<?php

use Illuminate\Queue\Middleware\RateLimited;

class ProcessPodcast implements ShouldQueue
{
    public function middleware(): array
    {
        return [
            // Максимум 10 джобов в минуту
            new RateLimited('podcasts'),
        ];
    }

    public function handle(): void
    {
        // Обработка подкаста
    }
}
```

**Конфигурация rate limiter:**

```php
// app/Providers/AppServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot()
{
    RateLimiter::for('podcasts', function ($job) {
        return Limit::perMinute(10);
    });
}
```

### Without Overlapping (предотвращение дублирования)

```php
use Illuminate\Queue\Middleware\WithoutOverlapping;

class ProcessOrder implements ShouldQueue
{
    public function __construct(
        public Order $order
    ) {}

    public function middleware(): array
    {
        return [
            // Не запускать несколько джобов для одного заказа одновременно
            (new WithoutOverlapping($this->order->id))
                ->releaseAfter(60) // Освободить блокировку через 60 сек
                ->expireAfter(180), // Истечение блокировки через 3 мин
        ];
    }
}
```

### Throttling

```php
use Illuminate\Queue\Middleware\ThrottlesExceptions;

class ProcessVideo implements ShouldQueue
{
    public function middleware(): array
    {
        return [
            // Максимум 3 исключения за 5 минут
            (new ThrottlesExceptions(3, 5))
                ->backoff(5), // Повторить через 5 минут
        ];
    }
}
```

---

## Retry Logic (Повторные попытки)

### Автоматические повторы

```php
class SendEmail implements ShouldQueue
{
    // Количество попыток
    public $tries = 3;

    // Или максимальное время
    public $timeout = 120; // 2 минуты

    // Задержка между попытками (секунды)
    public $backoff = [10, 30, 60]; // 10с, 30с, 60с

    // Или экспоненциальный backoff
    public function backoff(): int
    {
        return 10 * $this->attempts(); // 10, 20, 30, 40...
    }

    public function handle(): void
    {
        // Может упасть - будет повтор
        Mail::send(...);
    }

    // Вызывается после всех неудачных попыток
    public function failed(\Throwable $exception): void
    {
        // Логирование
        Log::error("Failed to send email after {$this->tries} attempts", [
            'exception' => $exception->getMessage(),
        ]);

        // Уведомление админа
        Notification::route('slack', config('slack.webhook'))
            ->notify(new JobFailedNotification($exception));
    }
}
```

### Ручной retry

```php
public function handle(): void
{
    try {
        $this->processOrder();
    } catch (RateLimitException $e) {
        // Повторить через 30 секунд
        return $this->release(30);
    } catch (TemporaryException $e) {
        // Повторить немедленно
        return $this->release();
    } catch (PermanentException $e) {
        // Удалить из очереди
        $this->fail($e);
    }
}
```

---

## Failed Jobs (Упавшие задачи)

### Таблица для failed jobs

```bash
php artisan queue:failed-table
php artisan migrate
```

### Просмотр failed jobs

```bash
# Список
php artisan queue:failed

# Детали конкретного
php artisan queue:failed 5
```

### Retry failed jobs

```bash
# Повторить конкретный
php artisan queue:retry 5

# Повторить все
php artisan queue:retry all

# Повторить последние 100
php artisan queue:retry --range=1-100
```

### Удаление failed jobs

```bash
# Удалить конкретный
php artisan queue:forget 5

# Очистить все
php artisan queue:flush
```

### Отключение failed jobs

```php
class ProcessVideo implements ShouldQueue
{
    // Не сохранять в failed_jobs при падении
    public $failOnTimeout = false;
}
```

---

## Job Batching (Пакетная обработка)

### Создание батча

```php
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

$batch = Bus::batch([
    new ProcessCsv(1, 1000),
    new ProcessCsv(1001, 2000),
    new ProcessCsv(2001, 3000),
])->then(function (Batch $batch) {
    // Все джобы успешно завершены
    Log::info('All jobs completed');
})->catch(function (Batch $batch, Throwable $e) {
    // Первый джоб упал
    Log::error('First batch job failed');
})->finally(function (Batch $batch) {
    // Батч завершен (успех или нет)
    Log::info('Batch finished');
})->name('Import CSV')
  ->onQueue('imports')
  ->dispatch();

return $batch->id;
```

### Добавление джобов в существующий батч

```php
class ProcessCsv implements ShouldQueue
{
    use Batchable;

    public function handle(): void
    {
        if ($this->batch()->cancelled()) {
            // Батч отменен
            return;
        }

        // Обработка данных
        $moreJobs = $this->processChunk();

        // Динамически добавить еще джобы
        if ($moreJobs) {
            $this->batch()->add($moreJobs);
        }
    }
}
```

### Отслеживание прогресса

```php
$batch = Bus::findBatch($batchId);

return [
    'total' => $batch->totalJobs,
    'processed' => $batch->processedJobs(),
    'pending' => $batch->pendingJobs,
    'failed' => $batch->failedJobs,
    'progress' => $batch->progress(), // Процент
    'finished' => $batch->finished(),
    'cancelled' => $batch->cancelled(),
];
```

### Отмена батча

```php
$batch = Bus::findBatch($batchId);
$batch->cancel();
```

---

## Job Chaining (Цепочки)

### Последовательное выполнение

```php
SendWelcomeEmail::withChain([
    new ProcessAvatar($user),
    new UpdateStatistics($user),
    new NotifyAdmin($user),
])->dispatch($user);

// Или
SendWelcomeEmail::dispatch($user)->chain([
    new ProcessAvatar($user),
    new UpdateStatistics($user),
]);
```

### Условные цепочки

```php
class ProcessPayment implements ShouldQueue
{
    public function handle(): void
    {
        $result = $this->processPayment();

        if ($result->success) {
            // Продолжить цепочку
            $this->next([
                new SendReceipt($result),
                new UpdateInventory($result),
            ]);
        } else {
            // Остановить цепочку
            $this->delete();
        }
    }

    private function next(array $jobs): void
    {
        foreach ($jobs as $job) {
            dispatch($job);
        }
    }
}
```

---

## Monitoring & Debugging

### Laravel Horizon (для Redis)

```bash
composer require laravel/horizon
php artisan horizon:install
php artisan horizon
```

**Dashboard:** `http://your-app.test/horizon`

**Features:**
- Real-time мониторинг очередей
- Метрики (throughput, runtime, failed jobs)
- Управление worker'ами
- Retry failed jobs

**config/horizon.php:**

```php
return [
    'environments' => [
        'production' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['high', 'default', 'low'],
                'balance' => 'auto',
                'processes' => 10,
                'tries' => 3,
                'timeout' => 300,
            ],
        ],

        'local' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'simple',
                'processes' => 3,
                'tries' => 3,
            ],
        ],
    ],
];
```

### Queue Monitoring Events

```php
use Illuminate\Queue\Events\JobProcessed;
use Illuminate\Queue\Events\JobFailed;
use Illuminate\Support\Facades\Queue;

// В AppServiceProvider
public function boot()
{
    Queue::after(function (JobProcessed $event) {
        // Джоб успешно обработан
        Log::info("Job processed: {$event->job->resolveName()}");
    });

    Queue::failing(function (JobFailed $event) {
        // Джоб упал
        Log::error("Job failed: {$event->job->resolveName()}", [
            'exception' => $event->exception->getMessage(),
        ]);
    });
}
```

### Custom Queue Logger

```php
class QueueMetricsJob implements ShouldQueue
{
    private $startTime;

    public function handle(): void
    {
        $this->startTime = microtime(true);

        try {
            // Основная логика
            $this->process();

            $this->logSuccess();
        } catch (\Throwable $e) {
            $this->logFailure($e);
            throw $e;
        }
    }

    private function logSuccess(): void
    {
        $duration = microtime(true) - $this->startTime;

        Log::info('Job completed', [
            'job' => static::class,
            'duration' => $duration,
            'memory' => memory_get_peak_usage(true),
            'attempts' => $this->attempts(),
        ]);
    }

    private function logFailure(\Throwable $e): void
    {
        Log::error('Job failed', [
            'job' => static::class,
            'error' => $e->getMessage(),
            'attempts' => $this->attempts(),
        ]);
    }
}
```

---

## Production Best Practices

### 1. Supervisor для queue workers

**supervisord.conf:**

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/path/to/worker.log
stopwaitsecs=3600
```

**Команды:**

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
sudo supervisorctl stop laravel-worker:*
sudo supervisorctl restart laravel-worker:*
```

### 2. Graceful Restart

```bash
# После deploy - gracefully перезапустить worker'ы
php artisan queue:restart
```

Worker'ы завершат текущие джобы и перезапустятся.

### 3. Max Time vs Timeout

```php
class LongRunningJob implements ShouldQueue
{
    // Максимальное время выполнения ОДНОЙ попытки
    public $timeout = 600; // 10 минут

    // После этого worker завершится (и supervisor перезапустит)
    // php artisan queue:work --max-time=3600
}
```

### 4. Memory Limit

```bash
# Worker перезапустится если память > 128MB
php artisan queue:work --memory=128
```

### 5. Database Transactions

```php
class ProcessOrder implements ShouldQueue
{
    // Отправить в очередь только после commit транзакции
    public $afterCommit = true;

    // Или при dispatch
    ProcessOrder::dispatch($order)->afterCommit();
}
```

### 6. Unique Jobs (Laravel 10+)

```php
use Illuminate\Contracts\Queue\ShouldBeUnique;

class ProcessReport implements ShouldQueue, ShouldBeUnique
{
    public function __construct(
        public Report $report
    ) {}

    // Уникальный ID джоба
    public function uniqueId(): string
    {
        return $this->report->id;
    }

    // Время уникальности (секунды)
    public function uniqueFor(): int
    {
        return 3600; // 1 час
    }
}
```

Если джоб с таким `uniqueId` уже в очереди - новый не добавится.

### 7. Encrypted Jobs (для чувствительных данных)

```php
use Illuminate\Contracts\Queue\ShouldBeEncrypted;

class ProcessPayment implements ShouldQueue, ShouldBeEncrypted
{
    public function __construct(
        public string $cardNumber,
        public string $cvv
    ) {}
}
```

Джоб будет зашифрован в очереди.

---

## Testing

### Fake Queue

```php
use Illuminate\Support\Facades\Queue;

public function test_order_creates_jobs()
{
    Queue::fake();

    // Действие
    $response = $this->post('/orders', $orderData);

    // Проверка
    Queue::assertPushed(ProcessOrder::class);
    
    Queue::assertPushed(ProcessOrder::class, function ($job) use ($order) {
        return $job->order->id === $order->id;
    });

    Queue::assertPushed(ProcessOrder::class, 1); // Ровно 1 раз
    
    Queue::assertNotPushed(CancelOrder::class);
    
    Queue::assertPushedOn('high', ProcessOrder::class);
}
```

### Синхронное выполнение в тестах

```php
// phpunit.xml
<env name="QUEUE_CONNECTION" value="sync"/>

// Или в конкретном тесте
Config::set('queue.default', 'sync');
```

---

## Common Patterns

### 1. Idempotent Jobs (идемпотентность)

```php
class ProcessPayment implements ShouldQueue
{
    public function handle(): void
    {
        // Проверка: не обработан ли уже
        if ($this->order->isPaid()) {
            return; // Уже обработано
        }

        DB::transaction(function () {
            $this->payment->process();
            $this->order->markAsPaid();
        });
    }
}
```

### 2. Job с прогрессом

```php
class ImportCsv implements ShouldQueue
{
    use Batchable;

    public function handle(): void
    {
        $rows = $this->getCsvRows();
        $total = count($rows);

        foreach ($rows as $index => $row) {
            $this->processRow($row);

            // Обновить прогресс
            $this->updateProgress($index + 1, $total);
        }
    }

    private function updateProgress(int $current, int $total): void
    {
        Cache::put("import.{$this->importId}.progress", [
            'current' => $current,
            'total' => $total,
            'percentage' => round(($current / $total) * 100, 2),
        ], 3600);
    }
}
```

### 3. Distributed Lock

```php
use Illuminate\Support\Facades\Cache;

class ProcessResource implements ShouldQueue
{
    public function handle(): void
    {
        $lock = Cache::lock("process-resource-{$this->resourceId}", 60);

        if ($lock->get()) {
            try {
                // Критическая секция
                $this->process();
            } finally {
                $lock->release();
            }
        } else {
            // Не удалось получить lock - повторить позже
            $this->release(10);
        }
    }
}
```

---

## Когда использовать очереди?

### ✅ Используй очереди для:
- Отправка email/SMS
- Обработка изображений/видео
- Генерация PDF отчетов
- API запросы к внешним сервисам
- Импорт/экспорт данных
- Сложные вычисления
- Уведомления
- Логирование/аналитика

### ❌ НЕ используй очереди для:
- Критичных для UX операций (пользователь ждет результат)
- Операций требующих немедленного ответа
- Простых database операций
- Задач < 100ms

**Правило:** Если операция > 200ms - подумай об очереди!
