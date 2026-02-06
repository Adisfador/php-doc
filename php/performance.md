# Performance - Производительность PHP

Полный разбор оптимизации: OpCache, JIT, profiling, memory, caching, benchmarking, Laravel optimization.

---

## ⚡ OpCache - Opcode Cache

### Что это?

**OpCache** - встроенный в PHP кэш откомпилированного bytecode.

**Как работает PHP:**
1. **Parsing** - исходный код → AST (Abstract Syntax Tree)
2. **Compilation** - AST → Opcodes (bytecode)
3. **Execution** - Opcodes выполняются Zend Engine

**Без OpCache:** каждый запрос парсит и компилирует файлы заново.  
**С OpCache:** opcodes кэшируются в памяти → **30-50% прирост производительности**.

### Настройка OpCache

**php.ini:**
```ini
[opcache]
; Включить OpCache
opcache.enable=1
opcache.enable_cli=0  ; Выключить для CLI (не нужно)

; Память для opcache (128-256MB для средних приложений)
opcache.memory_consumption=256

; Максимальное количество файлов (Laravel ~10k файлов)
opcache.max_accelerated_files=20000

; ===== PRODUCTION =====
; НЕ проверять изменения файлов (быстрее)
opcache.validate_timestamps=0

; Или проверять каждые 60 секунд (если не используешь deployment)
opcache.revalidate_freq=60

; ===== DEVELOPMENT =====
; Проверять каждый запрос
opcache.validate_timestamps=1
opcache.revalidate_freq=0

; Другие важные настройки
opcache.save_comments=1  ; Для Doctrine annotations, PHPDoc
opcache.fast_shutdown=1  ; Быстрое завершение
opcache.interned_strings_buffer=16  ; Для строк (classnames, function names)
opcache.max_wasted_percentage=10  ; Перезагрузка при фрагментации >10%
```

### Мониторинг OpCache

```php
<?php
$status = opcache_get_status();

echo "Memory usage: " . round($status['memory_usage']['used_memory'] / 1024 / 1024, 2) . " MB\n";
echo "Free memory: " . round($status['memory_usage']['free_memory'] / 1024 / 1024, 2) . " MB\n";
echo "Wasted memory: " . round($status['memory_usage']['wasted_memory'] / 1024 / 1024, 2) . " MB\n";
echo "Cached scripts: " . $status['opcache_statistics']['num_cached_scripts'] . "\n";
echo "Hits: " . $status['opcache_statistics']['hits'] . "\n";
echo "Misses: " . $status['opcache_statistics']['misses'] . "\n";
echo "Hit rate: " . round($status['opcache_statistics']['opcache_hit_rate'], 2) . "%\n";

// Hit rate должен быть >95%
```

### Очистка OpCache

```php
// Очистить весь cache
opcache_reset();

// Invaliate конкретный файл
opcache_invalidate('/path/to/file.php', true);
```

**После deployment:**
```bash
# Reload PHP-FPM
sudo systemctl reload php8.2-fpm

# Или через web (если настроен)
curl http://yoursite.com/opcache-clear.php
```

---

## 🚀 JIT - Just-In-Time Compiler (PHP 8.0+)

### Что это?

**JIT** компилирует горячие участки кода в **нативный машинный код** (минуя Zend VM).

**Когда помогает:**
- ✅ CPU-intensive вычисления (математика, алгоритмы)
- ✅ Image processing
- ✅ Криптография
- ✅ Парсинг больших данных

**Когда НЕ помогает:**
- ❌ I/O bound приложения (БД, сеть) - большинство веб-приложений!
- ❌ Laravel/Symfony (мало CPU-bound кода)

### Настройка JIT

**php.ini:**
```ini
opcache.enable=1
opcache.jit=tracing  ; или 'function'
opcache.jit_buffer_size=100M
```

**Режимы JIT:**
- `tracing` - компилирует часто выполняемые пути (recommended)
- `function` - компилирует целые функции
- `1255` - tracing JIT с агрессивной оптимизацией
- `off` - отключить JIT

### Тестирование JIT

```php
<?php
// CPU-bound задача
function fibonacci(int $n): int {
    if ($n <= 1) return $n;
    return fibonacci($n - 1) + fibonacci($n - 2);
}

$start = microtime(true);
$result = fibonacci(35);
$time = microtime(true) - $start;

echo "Result: {$result}\n";
echo "Time: " . round($time, 4) . "s\n";

// Без JIT: ~2.5s
// С JIT: ~1.8s (30% быстрее)
```

**Для Laravel:**
```bash
# Обычно JIT дает 0-5% прирост для Laravel
# Не критично, но можно включить
```

---

## 📊 Profiling - Анализ производительности

### Xdebug Profiler

**Установка:**
```bash
sudo apt install php8.2-xdebug
```

**php.ini:**
```ini
[xdebug]
zend_extension=xdebug.so
xdebug.mode=profile
xdebug.output_dir=/tmp/xdebug
xdebug.profiler_output_name=cachegrind.out.%p
```

**Анализ:**
```bash
# KCachegrind (Linux)
kcachegrind /tmp/xdebug/cachegrind.out.12345

# QCachegrind (macOS/Windows)
qcachegrind /tmp/xdebug/cachegrind.out.12345

# Webgrind (веб-интерфейс)
# https://github.com/jokkedk/webgrind
```

**Trigger profiling по требованию:**
```ini
xdebug.mode=profile
xdebug.start_with_request=trigger
```

```bash
# Добавить ?XDEBUG_PROFILE=1 к URL
curl "http://localhost/slow-page?XDEBUG_PROFILE=1"
```

### Blackfire.io (рекомендуется для production)

```bash
# Установка
wget -q -O - https://packages.blackfire.io/gpg.key | sudo apt-key add -
echo "deb http://packages.blackfire.io/debian any main" | sudo tee /etc/apt/sources.list.d/blackfire.list
sudo apt update
sudo apt install blackfire-agent blackfire-php

# Конфигурация
sudo blackfire-agent --register
```

**Профилирование:**
```bash
# CLI
blackfire run php artisan some:command

# HTTP
blackfire curl http://yoursite.com/slow-endpoint

# Browser extension
# Установить Blackfire Chrome/Firefox extension
```

**Анализ:**
- Flamegraph - визуализация hot paths
- Timeline - хронология выполнения
- Recommendations - автоматические рекомендации

### Tideways

```bash
# Альтернатива Blackfire (тоже платная, но дешевле)
composer require tideways/profiler
```

### XHProf (бесплатная альтернатива)

```bash
# Установка
sudo pecl install xhprof
echo "extension=xhprof.so" | sudo tee /etc/php/8.2/mods-available/xhprof.ini
sudo phpenmod xhprof
```

**Использование:**
```php
<?php
xhprof_enable(XHPROF_FLAGS_CPU + XHPROF_FLAGS_MEMORY);

// Код для профилирования
processLargeDataset();

$data = xhprof_disable();

// Сохранить результаты
file_put_contents('/tmp/xhprof.json', json_encode($data));

// Анализ через xhprof_html (веб UI)
```

---

## 💾 Memory Optimization

### memory_limit

```ini
; php.ini
memory_limit = 256M  ; Для большинства приложений

; Для CLI задач (импорт данных)
memory_limit = 512M
```

**Мониторинг:**
```php
echo "Memory usage: " . round(memory_get_usage() / 1024 / 1024, 2) . " MB\n";
echo "Peak memory: " . round(memory_get_peak_usage() / 1024 / 1024, 2) . " MB\n";
```

### unset() больших переменных

```php
$largeArray = range(1, 1000000);

// Обработка
processData($largeArray);

// Освободить память
unset($largeArray);
```

### Generators вместо массивов

```php
// ❌ Плохо - загружает все 1M записей в память
function getAllUsers(): array {
    return DB::table('users')->get()->toArray();  // 500MB памяти
}

foreach (getAllUsers() as $user) {
    processUser($user);
}

// ✅ Хорошо - по 1000 записей в памяти одновременно
function getUsersGenerator(): \Generator {
    $offset = 0;
    $limit = 1000;
    
    while (true) {
        // Загружаем batch из 1000 записей
        $users = DB::table('users')->offset($offset)->limit($limit)->get();
        
        if ($users->isEmpty()) {
            break;
        }
        
        // Yield каждую запись из batch, затем загружаем следующие 1000
        foreach ($users as $user) {
            yield $user;
        }
        
        $offset += $limit;
    }
    // В памяти максимум 1000 записей, а не все 1M
}

foreach (getUsersGenerator() as $user) {
    processUser($user);  // Процессит по 1 записи, но грузит батчами по 1000
}

// ✅ Еще лучше - Laravel chunk
DB::table('users')->chunk(1000, function($users) {
    foreach ($users as $user) {
        processUser($user);
    }
});
```

### SplFixedArray для больших массивов

```php
// Обычный array
$array = [];
for ($i = 0; $i < 100000; $i++) {
    $array[] = $i;
}
// ~9MB

// SplFixedArray
$fixed = new SplFixedArray(100000);
for ($i = 0; $i < 100000; $i++) {
    $fixed[$i] = $i;
}
// ~7MB (20-30% экономия памяти)
```

### WeakMap для caches (PHP 8.0+)

```php
// ❌ Обычный cache может утечь память
class UserRepository {
    private array $cache = [];
    
    public function find(int $id): User {
        if (!isset($this->cache[$id])) {
            $this->cache[$id] = User::query()->find($id);
        }
        return $this->cache[$id];
    }
}
// Объекты User остаются в памяти даже если больше не нужны

// ✅ WeakMap автоматически очищает неиспользуемые объекты
class ImageProcessor {
    private WeakMap $processedImages;
    
    public function __construct() {
        $this->processedImages = new WeakMap();
    }
    
    public function process(Image $image) {
        // Используем САМ объект как ключ
        if (!isset($this->processedImages[$image])) {
            $result = $this->heavyProcessing($image);
            $this->processedImages[$image] = $result;
        }
        
        return $this->processedImages[$image];
    }
}

// Использование:
$image = new Image('photo.jpg');
$processor->process($image);  // Обрабатывает
$processor->process($image);  // Берет из кэша (тот же объект!)

unset($image);  // Когда Image удаляется → запись из WeakMap тоже удаляется
```

---

## 🗄️ Caching Strategies

### 1. OpCode Cache (OpCache)

Кэширует скомпилированный PHP bytecode.

### 2. Data Cache (Redis/Memcached)

```php
use Illuminate\Support\Facades\Cache;

// Cache-Aside (Lazy Loading)
$users = Cache::remember('users:all', 3600, function() {
    return User::all();  // Только если нет в кэше
});

// Tags (только Redis/Memcached)
Cache::tags(['users', 'premium'])->put('john', $user, 3600);
Cache::tags(['users'])->flush();  // Очистить все с тегом 'users'

// Atomic locks
$lock = Cache::lock('process:import', 10);

if ($lock->get()) {
    try {
        // Critical section
        importData();
    } finally {
        $lock->release();
    }
}
```

### 3. Query Cache (избегать в MySQL 8+)

**MySQL Query Cache удален в 8.0+**

Вместо этого:
```php
// ✅ Application-level caching
$products = Cache::remember('products:featured', 3600, function() {
    return Product::where('featured', true)->get();
});
```

### 4. HTTP Cache (Varnish/CDN)

```php
// Cache-Control headers
return response($content)
    ->header('Cache-Control', 'public, max-age=3600');

// Laravel Response Cache
composer require spatie/laravel-responsecache

// Кэширует целые HTTP responses
```

### 5. Full Page Cache

```php
// Laravel Varnish
// https://github.com/spatie/laravel-varnish

// Очистка cache
Varnish::flush('http://yoursite.com/products');
```

---

## 🎯 Code Optimization

### 1. Избегать premature optimization

> "Premature optimization is the root of all evil" - Donald Knuth

```php
// ❌ Не оптимизируй без измерений
function calculate() {
    // Преждевременная микро-оптимизация
}

// ✅ Сначала measure, потом optimize
$start = microtime(true);
calculate();
echo microtime(true) - $start;  // 0.0001s - не стоит оптимизировать
```

### 2. Database Query Optimization

```php
// ❌ N+1 Query Problem
$posts = Post::all();  // 1 query
foreach ($posts as $post) {
    echo $post->user->name;  // N queries
}

// ✅ Eager Loading
$posts = Post::with('user')->get();  // 2 queries
foreach ($posts as $post) {
    echo $post->user->name;
}

// ✅ Lazy Eager Loading
$posts = Post::all();
$posts->load('user');  // 1 дополнительный query для всех users
```

**Select только нужные колонки:**
```php
// ❌ Загружает все поля
User::all();

// ✅ Только нужные
User::select('id', 'name', 'email')->get();
```

**Indexes:**
```php
// Migration
Schema::table('posts', function (Blueprint $table) {
    $table->index('user_id');
    $table->index('created_at');
    $table->index(['user_id', 'created_at']);  // Composite index
});

// Analyze EXPLAIN
DB::enableQueryLog();
User::where('email', 'john@example.com')->get();
dd(DB::getQueryLog());
```

### 3. String Operations

```php
// ✅ Single quotes быстрее (нет парсинга переменных)
$str = 'Hello, World!';  // Быстрее
$str = "Hello, World!";  // Медленнее (парсит переменные)

// ✅ sprintf для множества переменных
// Медленнее
$str = 'Hello, ' . $name . '! You have ' . $count . ' messages.';

// Быстрее
$str = sprintf('Hello, %s! You have %d messages.', $name, $count);

// ✅ Precompiled regex
$pattern = '/\d+/';
for ($i = 0; $i < 1000; $i++) {
    preg_match($pattern, $string);  // Regex компилируется 1 раз
}
```

### 4. Reduce function calls

```php
// ❌ Медленно
for ($i = 0; $i < count($array); $i++) {  // count() вызывается каждую итерацию
    echo $array[$i];
}

// ✅ Быстрее
$count = count($array);
for ($i = 0; $i < $count; $i++) {
    echo $array[$i];
}

// ✅ Еще лучше
foreach ($array as $item) {
    echo $item;
}
```

### 5. Algorithm Complexity

```php
// ❌ O(n²) - вложенные циклы
foreach ($users as $user) {
    foreach ($posts as $post) {
        if ($post->user_id === $user->id) {
            // ...
        }
    }
}

// ✅ O(n) - hash map
$postsByUser = [];
foreach ($posts as $post) {
    $postsByUser[$post->user_id][] = $post;
}

foreach ($users as $user) {
    $userPosts = $postsByUser[$user->id] ?? [];
    // ...
}
```

---

## 📏 Benchmarking

### ab (Apache Bench)

```bash
# 1000 запросов, 10 одновременно
ab -n 1000 -c 10 http://localhost/

# Вывод:
# Requests per second: 250 [#/sec]
# Time per request: 40ms [mean]
# 50%: 35ms
# 95%: 80ms
# 99%: 120ms
```

### wrk (HTTP benchmarking)

```bash
# 30 секунд, 12 потоков, 400 соединений
wrk -t12 -c400 -d30s http://localhost/

# С Lua скриптом для POST
wrk -t12 -c400 -d30s -s post.lua http://localhost/api/users
```

### Siege (load testing)

```bash
# 100 одновременных пользователей, 1 минута
siege -c100 -t1M http://localhost/

# Из файла с URL
siege -c50 -t30s -f urls.txt
```

### JMeter (GUI)

```bash
# Скачать: https://jmeter.apache.org/
# Создать Thread Group
# Добавить HTTP Request Sampler
# View Results Tree / Summary Report
```

### Laravel-specific

```bash
# Laravel Telescope (dev)
composer require laravel/telescope --dev
php artisan telescope:install

# Анализ requests, queries, cache, exceptions
http://localhost/telescope

# Laravel Debugbar
composer require barryvdh/laravel-debugbar --dev

# Показывает queries, timeline, memory
```

---

## ⚡ Laravel Performance Optimization

### 1. Config Caching

```bash
# Production
php artisan config:cache

# Объединяет все config в один файл
# НЕ используй env() вне config файлов!

# ❌ Не работает после config:cache
$value = env('API_KEY');

# ✅ Правильно
$value = config('services.api_key');
```

### 2. Route Caching

```bash
php artisan route:cache

# ⚠️ Не работает с Closures в routes
# ❌
Route::get('/', function() { return view('welcome'); });

# ✅ Используй Controllers
Route::get('/', [HomeController::class, 'index']);
```

### 3. View Caching

```bash
php artisan view:cache

# Прекомпилирует все Blade шаблоны
```

### 4. Event Caching (Laravel 8.6+)

```bash
php artisan event:cache
```

### 5. Optimize Command (все в одном)

```bash
php artisan optimize

# = config:cache + route:cache + view:cache + event:cache
```

### 6. Eager Loading

```php
// ❌ N+1 Problem
$users = User::all();
foreach ($users as $user) {
    echo $user->profile->bio;  // N queries
}

// ✅ Eager Loading
$users = User::with('profile')->get();  // 2 queries

// ✅ Nested Eager Loading
$users = User::with('posts.comments')->get();

// ✅ Conditional Eager Loading
$users = User::when($includeProfile, function($query) {
    $query->with('profile');
})->get();
```

### 7. Chunk для больших выборок

```php
// ❌ Загружает все 1M записей в память
User::all()->each(function($user) {
    processUser($user);
});

// ✅ По 1000 за раз
User::chunk(1000, function($users) {
    foreach ($users as $user) {
        processUser($user);
    }
});

// ✅ Или lazy()
User::lazy()->each(function($user) {
    processUser($user);
});
```

### 8. Cursor Pagination

```php
// ❌ Offset pagination (медленно для больших offset)
$users = User::paginate(10);  // page 1000 = OFFSET 10000

// ✅ Cursor pagination
$users = User::cursorPaginate(10);
// WHERE (created_at, id) > (last_created_at, last_id)
```

### 9. Query Optimization

```php
// exists() вместо count()
// ❌ Медленно
if (User::where('email', $email)->count() > 0) {}

// ✅ Быстрее (stops после первой найденной записи)
if (User::where('email', $email)->exists()) {}

// select() только нужные поля
User::select('id', 'name', 'email')->get();

// Index hints (MySQL)
DB::table('users')->from('users USE INDEX (idx_email)')
    ->where('email', $email)
    ->get();
```

### 10. Queue Heavy Tasks

```php
// ❌ Медленный response (3s генерация отчета)
public function generateReport(Request $request)
{
    $report = $this->reportService->generate();  // 3s
    return response()->json($report);
}

// ✅ Async processing
public function generateReport(Request $request)
{
    GenerateReportJob::dispatch($request->user());
    
    return response()->json(['message' => 'Report generation started']);
}
```

### 11. Octane (Swoole/RoadRunner)

```bash
composer require laravel/octane

# Swoole
php artisan octane:install --server=swoole

# RoadRunner
php artisan octane:install --server=roadrunner

# Запуск
php artisan octane:start --workers=4 --max-requests=500
```

**Прирост производительности:**
- Традиционный PHP-FPM: ~100 req/s
- Laravel Octane: ~1000-2000 req/s (10-20x быстрее)

**Особенности:**
- Application остается в памяти
- ⚠️ Не использовать глобальные переменные
- ⚠️ Memory leaks = проблема

### 12. Horizon (Queue Monitoring)

```bash
composer require laravel/horizon
php artisan horizon:install

# Запуск
php artisan horizon

# Dashboard
http://localhost/horizon
```

---

## 🎓 Performance Checklist

### Production Deployment

- [ ] `APP_DEBUG=false`
- [ ] `APP_ENV=production`
- [ ] OpCache включен (`opcache.validate_timestamps=0`)
- [ ] `composer install --no-dev --optimize-autoloader`
- [ ] `php artisan optimize` (config + route + view + event cache)
- [ ] Redis/Memcached для cache
- [ ] Queue worker для тяжелых задач
- [ ] CDN для статики
- [ ] Gzip/Brotli compression
- [ ] HTTP/2
- [ ] Database indexes
- [ ] Eager loading (нет N+1)
- [ ] HTTPS + HSTS header
- [ ] Security headers

### Monitoring

- [ ] New Relic / Datadog APM
- [ ] Laravel Telescope (staging)
- [ ] Blackfire.io (profiling)
- [ ] Sentry (errors)
- [ ] Logs (ELK stack / CloudWatch)

---

## 🎓 Для собеседования: ключевые точки

1. **OpCache** - кэш bytecode, 30-50% прирост, validate_timestamps=0 в production
2. **JIT** - для CPU-bound кода, мало пользы для Laravel
3. **Profiling** - Xdebug/Blackfire для поиска bottlenecks
4. **Memory** - generators/chunk для больших данных, WeakMap для cache
5. **Caching** - OpCache, data cache (Redis), HTTP cache (Varnish)
6. **N+1 Problem** - eager loading with(), lazy eager load()
7. **Database** - indexes, select нужные поля, exists() вместо count()
8. **Laravel optimization** - config/route/view cache, Octane для высокой нагрузки
9. **Benchmarking** - ab, wrk, Siege для load testing
10. **Production checklist** - OpCache, optimize, no-dev, Redis, CDN

**Главное:** Measure → Analyze → Optimize → Verify (не оптимизируй без измерений).
