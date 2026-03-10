# Swoole, RoadRunner и Laravel Octane

PHP как **долгоживущий процесс** вместо традиционной модели "запрос → процесс → смерть".

---

## Проблема традиционной модели PHP

```
Каждый запрос при PHP-FPM:
  1. Взять worker из пула
  2. Загрузить bootstrap (autoload, конфиг, DI контейнер)
  3. Обработать запрос
  4. Уничтожить всё и вернуть worker в пул

Bootstrap Laravel ~ 50-150ms на холодном старте
```

При высокой нагрузке это **главный bottleneck** — 60-80% времени запроса может уходить на инициализацию.

---

## Идея Application Server

```
Persistent process модель:
  1. Приложение запускается ОДИН РАЗ
  2. Bootstrap выполняется ОДИН РАЗ
  3. Каждый запрос:
     - Получить → обработать → ответить (без инициализации)
  4. Процесс живёт часами/днями
```

Аналог: как работают Node.js, Go, Java Spring — они не умирают между запросами.

---

## Swoole

**Swoole** — расширение PHP (C extension), добавляющее:
- Асинхронный HTTP/WebSocket сервер прямо в PHP
- Корутины (coroutines)
- Connection pooling
- Таймеры, Worker процессы

### Как работает

```
┌──────────────────────────────────────────┐
│           Swoole HTTP Server             │
│  ┌──────────┐  ┌──────────┐             │
│  │ Worker 1 │  │ Worker 2 │  ...        │
│  │(coroutine│  │(coroutine│             │
│  │ based)   │  │ based)   │             │
│  └──────────┘  └──────────┘             │
│    Приложение загружено один раз         │
└──────────────────────────────────────────┘
          ↑
   Прямые TCP соединения (нет Nginx/FPM)
```

### Базовый пример

```php
$server = new Swoole\HTTP\Server('0.0.0.0', 9501);

$server->on('request', function ($request, $response) {
    $response->header('Content-Type', 'text/plain');
    $response->end('Hello World');
});

$server->start();
```

### Корутины в Swoole

```php
// Параллельные запросы к БД без блокировки
Co\run(function () {
    $results = [];

    $wg = new Co\WaitGroup();

    $wg->add();
    go(function () use (&$results, $wg) {
        $results['users'] = Co\MySQL::query('SELECT * FROM users');
        $wg->done();
    });

    $wg->add();
    go(function () use (&$results, $wg) {
        $results['orders'] = Co\MySQL::query('SELECT * FROM orders');
        $wg->done();
    });

    $wg->wait();
    // оба запроса выполнились параллельно
});
```

### Проблемы Swoole

⚠️ **Memory leaks** — статические переменные, синглтоны, глобальный стейт живут между запросами  
⚠️ **Установка** — требует компиляции C extension  
⚠️ **Несовместимость** — не все PHP библиотеки работают в async режиме  
⚠️ **Debugging** — сложнее, чем обычный PHP

---

## RoadRunner

**RoadRunner** — Application Server написанный на **Go**, обрабатывает HTTP и передаёт запросы в PHP worker через бинарный протокол.

### Архитектура

```
┌────────────────────────────────────────────────┐
│              RoadRunner (Go)                   │
│   HTTP Server, балансировка, worker pool       │
└──────────────┬─────────────────────────────────┘
               │ PSR-7 запросы через pipe/socket
               ↓
┌──────────────────────────────────────────────┐
│            PHP Workers (persistent)          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Worker 1 │  │ Worker 2 │  │ Worker 3 │   │
│  │(PHP proc)│  │(PHP proc)│  │(PHP proc)│   │
│  └──────────┘  └──────────┘  └──────────┘   │
│    Bootstrap выполнен один раз               │
└──────────────────────────────────────────────┘
```

### Установка

```bash
# composer
composer require spiral/roadrunner

# скачать бинарник rr
./vendor/bin/rr get-binary

# запуск
./rr serve
```

### .rr.yaml конфиг

```yaml
version: "3"

server:
  command: "php worker.php"
  pool:
    num_workers: 4
    max_jobs: 64        # перезапустить worker после N запросов (защита от утечек)
    supervisor:
      max_worker_memory: 100  # MB, перезапустить если превысил

http:
  address: "0.0.0.0:8080"
  pool:
    num_workers: 4
```

### PHP Worker

```php
<?php
// worker.php

use Spiral\RoadRunner\Worker;
use Spiral\RoadRunner\Http\PSR7Worker;
use Nyholm\Psr7\Factory\Psr17Factory;

require 'vendor/autoload.php';

// Bootstrap один раз здесь
$worker = Worker::create();
$factory = new Psr17Factory();
$psr7 = new PSR7Worker($worker, $factory, $factory, $factory);

// Бесконечный цикл обработки запросов
while ($request = $psr7->waitRequest()) {
    try {
        $response = handleRequest($request); // твоя логика
        $psr7->respond($response);
    } catch (\Throwable $e) {
        $psr7->getWorker()->error((string) $e);
    }
}
```

### Преимущества RoadRunner

✅ **Не требует** изменений в PHP коде (в отличие от Swoole)  
✅ **Go HTTP сервер** — очень быстрый и стабильный  
✅ **Плагины** — Kafka consumer, gRPC, WebSockets, Temporal  
✅ **max_jobs** — автоматический перезапуск для защиты от утечек памяти

---

## Laravel Octane

**Laravel Octane** — официальный пакет Laravel для работы с Swoole или RoadRunner. Решает проблему совместимости Laravel с persistent процессами.

### Установка

```bash
composer require laravel/octane

php artisan octane:install
# выбрать: swoole или roadrunner

# запуск с RoadRunner
php artisan octane:start --server=roadrunner --port=8000

# запуск с Swoole
php artisan octane:start --server=swoole --port=8000
```

### Что делает Octane

```
Обычный Laravel (FPM):            Laravel Octane:
  Request → bootstrap → handle      bootstrap() — ОДИН РАЗ
  Request → bootstrap → handle      loop {
  Request → bootstrap → handle        clone(app) → handle → reset
  ...                                }
```

Octane **клонирует** Application контейнер для каждого запроса — изоляция стейта без перезагрузки.

### Производительность

| Сервер | RPS (простой маршрут) |
|---|---|
| PHP-FPM (Nginx) | ~1,000–2,000 |
| Octane + RoadRunner | ~5,000–8,000 |
| Octane + Swoole | ~8,000–15,000 |

*Реальные цифры зависят от приложения, железа и нагрузки.*

### Octane Cache — сохранение между запросами

```php
// Данные живут в памяти worker'а между запросами
// Быстрее Redis — нет сетевого вызова

Octane::table('users')->set('1', ['name' => 'Ivan']);
$user = Octane::table('users')->get('1');
```

### Что НЕЛЬЗЯ хранить в статических переменных

```php
// ❌ ОПАСНО — живёт между запросами, стейт протекает
class UserRepository
{
    private static array $cache = [];  // будет расти вечно

    public static function find(int $id): User
    {
        if (!isset(self::$cache[$id])) {
            self::$cache[$id] = User::find($id);
        }
        return self::$cache[$id];
    }
}

// ✅ ПРАВИЛЬНО — использовать Octane::table() или request-scoped DI
```

### Listeners для сброса стейта

```php
// ServiceProvider — сброс данных между запросами
use Laravel\Octane\Facades\Octane;

Octane::tick('clear-metrics', function () {
    // выполняется каждые N секунд
})->seconds(60);

// Сброс на каждый запрос
$this->app->scoped(MyService::class, function () {
    return new MyService(); // новый экземпляр на каждый запрос
});
```

---

## FrankenPHP

Новый вариант (2023+) — **Go сервер** встроенный прямо в PHP через CGO.

```
FrankenPHP = Caddy (Go web server) + PHP runtime в одном бинарнике
```

```bash
# Запуск Laravel без nginx/fpm
frankenphp php-server --root public/

# Или через Docker
docker run -v .:/app -p 80:80 dunglas/frankenphp
```

✅ Поддерживает Early Hints, HTTP/3, автоматический HTTPS  
✅ Worker mode (как Octane) — один bootstrap  
⚠️ Ещё молодой проект

---

## Сравнение подходов

| | PHP-FPM | RoadRunner | Swoole | Octane |
|---|---|---|---|---|
| Сложность setup | Низкая | Средняя | Высокая | Средняя |
| Производительность | Базовая | Высокая | Очень высокая | Высокая/Очень высокая |
| Memory leaks риск | Нет | Низкий (max_jobs) | Высокий | Средний |
| Совместимость | ✅ Всё | ✅ Всё | ⚠️ Не всё | ✅ Laravel |
| Корутины/async | ❌ | ❌ | ✅ | ❌ (Swoole) / ❌ (RR) |
| Prod готовность | ✅ | ✅ | ✅ | ✅ |

---

## Когда использовать

```
PHP-FPM:
  - Стандартные приложения
  - Команда не знакома с persistent PHP
  - Нужна максимальная совместимость

RoadRunner + Octane:
  - Laravel проект с высокой нагрузкой
  - Нужно ускорение без переписывания кода
  - Хочется Go стабильности HTTP слоя

Swoole + Octane:
  - Максимальная производительность
  - WebSockets, async задачи
  - Команда знает подводные камни

FrankenPHP:
  - Хочется минимальной инфраструктуры
  - Эксперименты, новые проекты
```

---

## Ключевые вопросы для интервью

- Чем persistent PHP процесс отличается от PHP-FPM?
- Почему статические переменные опасны в Swoole/Octane?
- Как Octane решает проблему state между запросами?
- Зачем в RoadRunner нужен параметр `max_jobs`?
- В чём разница между Swoole корутинами и async/await в Node.js?
- Когда НЕ стоит использовать Octane?
