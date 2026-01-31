# Laravel Caching - Драйверы, Tags, Стратегии

Полное руководство по кешированию в Laravel.

---

## Конфигурация

**config/cache.php:**

```php
<?php

return [
    'default' => env('CACHE_DRIVER', 'file'),

    'stores' => [
        'array' => [
            'driver' => 'array',
            'serialize' => false,
        ],

        'database' => [
            'driver' => 'database',
            'table' => 'cache',
            'connection' => null,
            'lock_connection' => null,
        ],

        'file' => [
            'driver' => 'file',
            'path' => storage_path('framework/cache/data'),
        ],

        'memcached' => [
            'driver' => 'memcached',
            'persistent_id' => env('MEMCACHED_PERSISTENT_ID'),
            'sasl' => [
                env('MEMCACHED_USERNAME'),
                env('MEMCACHED_PASSWORD'),
            ],
            'options' => [
                // Memcached::OPT_CONNECT_TIMEOUT => 2000,
            ],
            'servers' => [
                [
                    'host' => env('MEMCACHED_HOST', '127.0.0.1'),
                    'port' => env('MEMCACHED_PORT', 11211),
                    'weight' => 100,
                ],
            ],
        ],

        'redis' => [
            'driver' => 'redis',
            'connection' => 'cache',
            'lock_connection' => 'default',
        ],

        'dynamodb' => [
            'driver' => 'dynamodb',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
            'table' => env('DYNAMODB_CACHE_TABLE', 'cache'),
            'endpoint' => env('DYNAMODB_ENDPOINT'),
        ],

        'octane' => [
            'driver' => 'octane',
        ],
    ],

    'prefix' => env('CACHE_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_cache_'),
];
```

### Создание таблицы для database драйвера

```bash
php artisan cache:table
php artisan migrate
```

---

## Драйверы кеша

### File (файловый)

```php
// Простой, но медленный
// Хранит кеш в storage/framework/cache/data/
// ✅ Для локальной разработки
// ❌ Не подходит для production
```

### Array (в памяти)

```php
// Кеш живет только в рамках одного запроса
// ✅ Для тестов
// ❌ Не персистентный
```

### Database (БД)

```php
// Хранит в таблице cache
// ✅ Простая настройка, доступен везде
// ❌ Медленнее Redis/Memcached
```

### Redis (рекомендуется)

```php
// Быстрый, персистентный, с функциями
// ✅ Production ready, поддерживает tags
// ✅ Лучший выбор для большинства проектов

// .env
CACHE_DRIVER=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

### Memcached

```php
// Очень быстрый, но без персистентности
// ✅ Высокая скорость
// ❌ Не поддерживает tags
// ❌ Данные теряются при перезагрузке
```

---

## Базовое использование

### Получение из кеша

```php
use Illuminate\Support\Facades\Cache;

// Получить значение
$value = Cache::get('key');

// С default значением
$value = Cache::get('key', 'default');

// С closure как default
$value = Cache::get('key', function () {
    return DB::table('users')->get();
});

// Проверка существования
if (Cache::has('key')) {
    $value = Cache::get('key');
}

// Получить и удалить
$value = Cache::pull('key');

// Получить несколько значений
$values = Cache::many(['key1', 'key2', 'key3']);
// ['key1' => 'value1', 'key2' => 'value2', 'key3' => null]
```

### Сохранение в кеш

```php
// Сохранить навсегда
Cache::put('key', 'value');

// Сохранить на время (в секундах)
Cache::put('key', 'value', 3600); // 1 час

// С DateTime
Cache::put('key', 'value', now()->addMinutes(10));

// Сохранить несколько
Cache::putMany([
    'key1' => 'value1',
    'key2' => 'value2',
], 3600);

// Добавить только если ключа нет
if (Cache::add('key', 'value', 3600)) {
    // Успешно добавлено
}

// Сохранить навсегда
Cache::forever('key', 'value');

// Инкремент/Декремент (только для чисел)
Cache::increment('views'); // +1
Cache::increment('views', 5); // +5
Cache::decrement('downloads'); // -1
Cache::decrement('downloads', 3); // -3
```

### Получить или сохранить (Remember)

```php
// Если в кеше есть - вернуть, иначе выполнить closure и закешировать
$users = Cache::remember('users', 3600, function () {
    return DB::table('users')->get();
});

// Remember навсегда
$settings = Cache::rememberForever('settings', function () {
    return DB::table('settings')->get();
});

// Гибкий контроль
$value = Cache::get('key');

if (is_null($value)) {
    $value = // получить из БД
    Cache::put('key', $value, 3600);
}

return $value;
```

### Удаление из кеша

```php
// Удалить один ключ
Cache::forget('key');

// Удалить весь кеш
Cache::flush();

// Удалить с проверкой
if (Cache::has('key')) {
    Cache::forget('key');
}
```

---

## Cache Tags (только Redis/Memcached/Array)

```php
// Группировка кеша по тегам
Cache::tags(['users', 'premium'])->put('user:1', $user, 3600);
Cache::tags(['users'])->put('user:2', $user2, 3600);

// Получение
$user = Cache::tags(['users', 'premium'])->get('user:1');

// Удаление по тегу
Cache::tags(['users'])->flush(); // Удалит user:1 и user:2
Cache::tags(['premium'])->flush(); // Удалит только user:1

// Remember с тегами
$users = Cache::tags(['users', 'active'])->remember('active-users', 3600, function () {
    return User::where('status', 'active')->get();
});

// ⚠️ НЕ работает с file и database драйверами!
```

### Пример использования тегов

```php
class UserService
{
    public function getUser($id)
    {
        return Cache::tags(['users'])->remember("user:{$id}", 3600, function () use ($id) {
            return User::find($id);
        });
    }
    
    public function updateUser($id, array $data)
    {
        $user = User::find($id);
        $user->update($data);
        
        // Очистить кеш конкретного пользователя
        Cache::tags(['users'])->forget("user:{$id}");
        
        // Или сбросить весь кеш пользователей
        // Cache::tags(['users'])->flush();
        
        return $user;
    }
    
    public function getUserPosts($userId)
    {
        return Cache::tags(['users', 'posts'])->remember("user:{$userId}:posts", 3600, function () use ($userId) {
            return Post::where('user_id', $userId)->get();
        });
    }
}
```

---

## Atomic Locks

```php
use Illuminate\Support\Facades\Cache;

// Получить lock (блокировка)
$lock = Cache::lock('processing-order-123', 10); // 10 секунд

// Попытаться получить lock
if ($lock->get()) {
    // Lock получен, обработать заказ
    processOrder(123);
    
    // Освободить lock
    $lock->release();
} else {
    // Lock занят, заказ уже обрабатывается
}

// Автоматическое освобождение
Cache::lock('key', 10)->get(function () {
    // Эксклюзивная работа
    // Lock автоматически освободится после выполнения
});

// Block - ждать освобождения lock
$lock = Cache::lock('key', 10);

$lock->block(5, function () {
    // Ждать максимум 5 секунд
    // Если lock освободится - выполнить
});

// Или с timeout
try {
    $lock->block(5);
    
    // Lock получен
    processOrder();
    
    $lock->release();
} catch (LockTimeoutException $e) {
    // Не удалось получить lock за 5 секунд
}
```

### Пример: предотвращение race condition

```php
class OrderService
{
    public function processPayment($orderId, $amount)
    {
        $lock = Cache::lock("order-payment-{$orderId}", 10);
        
        if ($lock->get()) {
            try {
                $order = Order::find($orderId);
                
                if ($order->status === 'pending') {
                    // Обработать платеж
                    $payment = PaymentGateway::charge($amount);
                    
                    $order->update([
                        'status' => 'paid',
                        'payment_id' => $payment->id,
                    ]);
                }
            } finally {
                $lock->release();
            }
        }
    }
}
```

---

## Стратегии кеширования

### 1. Cache-Aside (Lazy Loading)

```php
// Самая популярная стратегия
public function getUser($id)
{
    return Cache::remember("user:{$id}", 3600, function () use ($id) {
        return User::find($id);
    });
}

// ✅ Просто реализовать
// ✅ Кеш наполняется по мере запросов
// ❌ Cache miss приводит к запросу в БД
```

### 2. Write-Through

```php
// При записи сразу обновляем кеш
public function updateUser($id, array $data)
{
    $user = User::find($id);
    $user->update($data);
    
    // Сразу обновить кеш
    Cache::put("user:{$id}", $user, 3600);
    
    return $user;
}

// ✅ Кеш всегда актуален
// ❌ Дополнительные операции при записи
```

### 3. Write-Behind (Write-Back)

```php
// Записывать в кеш, периодически сбрасывать в БД
public function updateUserStats($userId, $action)
{
    Cache::increment("user:{$userId}:actions");
    
    // Периодически (через команду/job) сохранять в БД
}

// Команда (запускается по расписанию)
class SyncCacheToDatabase extends Command
{
    public function handle()
    {
        // Синхронизировать кеш с БД
    }
}

// ✅ Высокая производительность записи
// ❌ Риск потери данных при сбое
// ❌ Сложная реализация
```

### 4. Refresh-Ahead

```php
// Обновлять кеш до истечения TTL
public function getPopularPosts()
{
    $posts = Cache::get('popular-posts');
    
    if (is_null($posts)) {
        $posts = $this->loadPopularPosts();
        Cache::put('popular-posts', $posts, 3600);
    } else {
        // Если кеш скоро истечет - обновить в фоне
        $ttl = Cache::getStore()->getRedis()->ttl('popular-posts');
        
        if ($ttl < 300) { // Осталось меньше 5 минут
            dispatch(new RefreshPopularPostsCache());
        }
    }
    
    return $posts;
}

// ✅ Минимальное количество cache miss
// ❌ Сложнее реализовать
```

---

## Кеширование Eloquent запросов

### Query Cache (кастомный trait)

```php
// app/Traits/Cacheable.php
namespace App\Traits;

use Illuminate\Support\Facades\Cache;

trait Cacheable
{
    public function scopeCacheQuery($query, $key, $ttl = 3600)
    {
        $cacheKey = $this->getCacheKey($key, $query);
        
        return Cache::remember($cacheKey, $ttl, function () use ($query) {
            return $query->get();
        });
    }
    
    protected function getCacheKey($key, $query)
    {
        return sprintf(
            '%s:%s:%s',
            $this->getTable(),
            $key,
            md5($query->toSql() . serialize($query->getBindings()))
        );
    }
    
    public static function bootCacheable()
    {
        // Очищать кеш при изменении
        static::created(function ($model) {
            $model->clearModelCache();
        });
        
        static::updated(function ($model) {
            $model->clearModelCache();
        });
        
        static::deleted(function ($model) {
            $model->clearModelCache();
        });
    }
    
    public function clearModelCache()
    {
        Cache::tags([$this->getTable()])->flush();
    }
}

// Использование
class User extends Model
{
    use Cacheable;
}

// В контроллере
$activeUsers = User::where('status', 'active')
    ->cacheQuery('active-users', 3600);
```

### Model Cache (пакет genealabs/laravel-model-caching)

```bash
composer require genealabs/laravel-model-caching
```

```php
use GeneaLabs\LaravelModelCaching\Traits\Cachable;

class User extends Model
{
    use Cachable;
}

// Автоматическое кеширование запросов
$users = User::where('status', 'active')->get(); // Кешируется
$user = User::find(1); // Кешируется

// Отключить кеш для конкретного запроса
$users = User::disableCache()->get();
```

---

## Cache Events

```php
use Illuminate\Support\Facades\Cache;
use Illuminate\Cache\Events\CacheHit;
use Illuminate\Cache\Events\CacheMissed;
use Illuminate\Cache\Events\KeyForgotten;
use Illuminate\Cache\Events\KeyWritten;

// В EventServiceProvider
protected $listen = [
    CacheHit::class => [
        function (CacheHit $event) {
            // $event->key
            // $event->value
            Log::info("Cache hit: {$event->key}");
        },
    ],
    
    CacheMissed::class => [
        function (CacheMissed $event) {
            Log::info("Cache missed: {$event->key}");
        },
    ],
    
    KeyWritten::class => [
        function (KeyWritten $event) {
            // $event->key
            // $event->value
            // $event->seconds
        },
    ],
    
    KeyForgotten::class => [
        function (KeyForgotten $event) {
            Log::info("Cache key forgotten: {$event->key}");
        },
    ],
];
```

---

## Кеширование представлений (View Cache)

```php
// В Blade
@cache('sidebar', 3600)
    <div class="sidebar">
        @foreach($categories as $category)
            <a href="{{ route('category', $category) }}">
                {{ $category->name }}
            </a>
        @endforeach
    </div>
@endcache

// Или вручную
$html = Cache::remember('sidebar-html', 3600, function () {
    return view('partials.sidebar')->render();
});
```

---

## Response Cache (HTTP Cache)

```bash
composer require spatie/laravel-responsecache
```

```php
// config/responsecache.php
return [
    'enabled' => env('RESPONSE_CACHE_ENABLED', true),
    'cache_lifetime_in_seconds' => 60 * 60 * 24 * 7, // 1 неделя
];

// Middleware
Route::middleware('cacheResponse')->group(function () {
    Route::get('/posts', [PostController::class, 'index']);
});

// Или на уровне контроллера
class PostController extends Controller
{
    public function __construct()
    {
        $this->middleware('cacheResponse:3600'); // 1 час
    }
}

// Очистка кеша
use Spatie\ResponseCache\Facades\ResponseCache;

ResponseCache::clear(); // Весь кеш
ResponseCache::forget('/posts'); // Конкретный URL
```

---

## Cache Warming (Прогрев кеша)

```php
// Artisan команда для прогрева
namespace App\Console\Commands;

use Illuminate\Console\Command;

class WarmCache extends Command
{
    protected $signature = 'cache:warm';
    
    public function handle()
    {
        $this->info('Warming cache...');
        
        // Прогреть основные данные
        Cache::remember('settings', 86400, function () {
            return Setting::all()->pluck('value', 'key');
        });
        
        Cache::remember('categories', 3600, function () {
            return Category::with('posts')->get();
        });
        
        Cache::remember('popular-posts', 3600, function () {
            return Post::orderBy('views', 'desc')->limit(10)->get();
        });
        
        $this->info('Cache warmed successfully!');
    }
}

// Запуск после deploy
php artisan cache:warm
```

---

## Best Practices

### 1. Используй Cache Tags для группировки

```php
// ✅ ХОРОШО
Cache::tags(['users', 'premium'])->put('user:1', $user);

// При изменении очистить все связанное
Cache::tags(['users'])->flush();
```

### 2. Правильное время жизни (TTL)

```php
// Часто меняющиеся данные - короткий TTL
Cache::put('active-sessions', $sessions, 60); // 1 минута

// Редко меняющиеся - длинный TTL
Cache::put('settings', $settings, 86400); // 1 день

// Статические данные - forever
Cache::forever('countries', $countries);
```

### 3. Cache Keys по соглашению

```php
// ✅ ХОРОШО - структурированные ключи
Cache::put('user:1:profile', $profile);
Cache::put('post:123:comments', $comments);
Cache::put('category:tech:posts', $posts);

// ❌ ПЛОХО
Cache::put('u1', $profile);
Cache::put('comments', $comments);
```

### 4. Очистка кеша при изменениях

```php
// В модели
class Post extends Model
{
    protected static function booted()
    {
        static::saved(function ($post) {
            Cache::tags(['posts'])->flush();
            Cache::forget("post:{$post->id}");
        });
    }
}

// Или в Observer
class PostObserver
{
    public function saved(Post $post)
    {
        Cache::tags(['posts'])->flush();
    }
}
```

### 5. Remember вместо get/put

```php
// ❌ ПЛОХО
$users = Cache::get('users');

if (is_null($users)) {
    $users = User::all();
    Cache::put('users', $users, 3600);
}

// ✅ ХОРОШО
$users = Cache::remember('users', 3600, function () {
    return User::all();
});
```

### 6. Используй Redis для production

```php
// .env
CACHE_DRIVER=redis

// ✅ Быстро, поддерживает tags, персистентный
// ✅ Можно мониторить через Redis CLI
```

### 7. Мониторинг кеша

```php
// Логировать cache miss
Cache::missing(function ($key) {
    Log::info("Cache missed: {$key}");
});

// Метрики
$hitRate = Cache::get('cache-hits') / (Cache::get('cache-hits') + Cache::get('cache-misses'));
```

### 8. Не кешируй всё подряд

```php
// ❌ НЕ кешируй:
// - Редко запрашиваемые данные
// - Очень часто меняющиеся данные
// - Пользовательские данные (разные для каждого)

// ✅ Кешируй:
// - Дорогие запросы (JOIN, COUNT)
// - Популярный контент
// - Статические данные (настройки, переводы)
```

### 9. Cache Aside Pattern (рекомендуется)

```php
public function getUser($id)
{
    // 1. Проверить кеш
    $user = Cache::get("user:{$id}");
    
    if ($user) {
        return $user;
    }
    
    // 2. Загрузить из БД
    $user = User::find($id);
    
    // 3. Сохранить в кеш
    Cache::put("user:{$id}", $user, 3600);
    
    return $user;
    
    // Или короче:
    return Cache::remember("user:{$id}", 3600, fn() => User::find($id));
}
```

---

## Команды Artisan

```bash
# Очистить весь кеш
php artisan cache:clear

# Очистить кеш конфигурации
php artisan config:clear

# Очистить кеш маршрутов
php artisan route:clear

# Очистить кеш представлений
php artisan view:clear

# Кешировать конфигурацию (для production)
php artisan config:cache

# Кешировать маршруты
php artisan route:cache

# Кешировать события
php artisan event:cache
```
