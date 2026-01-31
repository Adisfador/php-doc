# PSR Standards - PHP Standards Recommendations

Полный разбор PSR стандартов: кодстайл, autoloading, логирование, HTTP, кэширование, DI контейнеры.

---

## 🎯 Что такое PSR?

**PSR (PHP Standards Recommendations)** - стандарты от PHP-FIG (PHP Framework Interop Group).

**Цель:** обеспечить совместимость между библиотеками и фреймворками.

**Статусы:**
- ✅ **Accepted** - принятые стандарты
- 📝 **Draft** - черновики
- ❌ **Deprecated** - устаревшие (PSR-2 заменен на PSR-12)
- 🗑️ **Abandoned** - заброшенные

**Ссылки:** [php-fig.org](https://www.php-fig.org/psr/)

---

## 📝 PSR-1: Basic Coding Standard

### Основные правила

**1. Файлы должны использовать только `<?php` или `<?=`:**
```php
<?php
// ✅ Правильно

<?
// ❌ Неправильно (short tags)
```

**2. Файлы должны быть в UTF-8 без BOM:**
```php
<?php
// Без Byte Order Mark (BOM)
```

**3. Файл должен либо объявлять символы, либо иметь side-effects, но НЕ оба:**

```php
// ✅ Правильно - только объявления
<?php
namespace App;

class User {}

// ✅ Правильно - только side-effects
<?php
ini_set('display_errors', '1');
require 'bootstrap.php';

// ❌ Неправильно - и то, и другое
<?php
ini_set('display_errors', '1');  // side-effect

class User {}  // объявление
```

**4. Namespaces и классы следуют PSR-4:**
```php
<?php
namespace Vendor\Package;

class ClassName {}
```

**5. Названия классов в StudlyCaps:**
```php
<?php
class UserController {}      // ✅
class User_Controller {}     // ❌
class userController {}      // ❌
```

**6. Константы классов в UPPER_CASE:**
```php
<?php
class Config {
    const VERSION = '1.0.0';
    const API_KEY = 'secret';
    
    const apiKey = 'secret';  // ❌
}
```

**7. Методы в camelCase:**
```php
<?php
class User {
    public function getName() {}      // ✅
    public function get_name() {}     // ❌
    public function GetName() {}      // ❌
}
```

---

## 🎨 PSR-12: Extended Coding Style

**PSR-12 заменяет PSR-2** (PSR-2 deprecated с 2019 года).

### Основные правила

**1. declare(strict_types=1) должен быть на первой строке:**
```php
<?php declare(strict_types=1);

namespace App\Models;

use App\Contracts\UserInterface;

class User implements UserInterface
{
    // ...
}
```

**2. Отступы: 4 пробела (не табы):**
```php
<?php

class User
{
    public function getName(): string
    {
        return $this->name;
    }
}
```

**3. Открывающая фигурная скобка на новой строке для классов:**
```php
<?php

class User
{
    // ✅
}

class User {  // ❌
}
```

**4. Методы: фигурная скобка на новой строке:**
```php
<?php

class User
{
    public function getName(): string
    {
        return $this->name;
    }
}
```

**5. Видимость (visibility) обязательна:**
```php
<?php

class User
{
    public string $name;         // ✅
    private int $age;            // ✅
    protected ?string $email;    // ✅
    
    string $name;                // ❌ без visibility
    
    public function getName(): string  // ✅
    {
        return $this->name;
    }
    
    function getName() {}        // ❌ без visibility
}
```

**6. use statements - по алфавиту, группировка:**
```php
<?php

namespace App\Controllers;

use App\Models\User;
use App\Services\AuthService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class UserController
{
    // ...
}
```

**7. Импорт функций и констант:**
```php
<?php

namespace App\Helpers;

use function array_filter;
use function array_map;

use const PHP_EOL;
use const PHP_VERSION;

function processArray(array $data): array
{
    return array_filter(
        array_map('trim', $data)
    );
}
```

**8. Return type declarations:**
```php
<?php

class Calculator
{
    public function add(int $a, int $b): int  // ✅
    {
        return $a + $b;
    }
    
    public function divide(int $a, int $b) : float  // ❌ пробел перед :
    {
        return $a / $b;
    }
}
```

**9. Control structures - фигурная скобка на той же строке:**
```php
<?php

if ($condition) {
    // ✅
} elseif ($other) {
    // ✅
} else {
    // ✅
}

if ($condition)
{
    // ❌
}
```

**10. Switch:**
```php
<?php

switch ($variable) {
    case 'value1':
        // code
        break;
    case 'value2':
        // code
        break;
    default:
        // code
        break;
}
```

**11. Многострочные аргументы:**
```php
<?php

$foo = new Foo(
    $arg1,
    $arg2,
    $arg3
);

$result = $object->method(
    $arg1,
    $arg2,
    $arg3
);
```

**12. Closure (анонимные функции):**
```php
<?php

$closure = function ($arg1, $arg2) use ($var1, $var2) {
    // body
};

$longArgs = function (
    $longArgument,
    $longerArgument,
    $muchLongerArgument
) use (
    $var1,
    $var2
) {
    // body
};
```

---

## 🪵 PSR-3: Logger Interface

### LoggerInterface

```php
<?php

namespace Psr\Log;

interface LoggerInterface
{
    public function emergency(string|\Stringable $message, array $context = []): void;
    public function alert(string|\Stringable $message, array $context = []): void;
    public function critical(string|\Stringable $message, array $context = []): void;
    public function error(string|\Stringable $message, array $context = []): void;
    public function warning(string|\Stringable $message, array $context = []): void;
    public function notice(string|\Stringable $message, array $context = []): void;
    public function info(string|\Stringable $message, array $context = []): void;
    public function debug(string|\Stringable $message, array $context = []): void;
    public function log($level, string|\Stringable $message, array $context = []): void;
}
```

### Уровни логирования (RFC 5424)

1. **emergency** - система непригодна
2. **alert** - нужны немедленные действия
3. **critical** - критические условия
4. **error** - ошибки
5. **warning** - предупреждения
6. **notice** - нормальные но значимые события
7. **info** - информационные сообщения
8. **debug** - отладочная информация

### Использование context

```php
<?php

use Psr\Log\LoggerInterface;

class UserService
{
    public function __construct(
        private LoggerInterface $logger
    ) {}
    
    public function createUser(string $email): void
    {
        $this->logger->info('User creation started', [
            'email' => $email
        ]);
        
        try {
            // create user
            $this->logger->info('User created successfully', [
                'user_id' => $user->id,
                'email' => $email
            ]);
        } catch (\Exception $e) {
            $this->logger->error('User creation failed', [
                'email' => $email,
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);
            throw $e;
        }
    }
}
```

### Placeholders в message

```php
<?php

$logger->info('User {username} logged in from {ip}', [
    'username' => 'john',
    'ip' => '192.168.1.1'
]);

// Вывод: "User john logged in from 192.168.1.1"
```

### LoggerAwareInterface - внедрение logger

```php
<?php

namespace Psr\Log;

interface LoggerAwareInterface
{
    public function setLogger(LoggerInterface $logger): void;
}

// Использование:
use Psr\Log\LoggerAwareInterface;
use Psr\Log\LoggerAwareTrait;

class MyService implements LoggerAwareInterface
{
    use LoggerAwareTrait;
    
    public function doSomething(): void
    {
        $this->logger->info('Doing something');
    }
}
```

### Реализации PSR-3

**Monolog (самая популярная):**
```php
<?php

use Monolog\Logger;
use Monolog\Handler\StreamHandler;

$logger = new Logger('app');
$logger->pushHandler(new StreamHandler('app.log', Logger::WARNING));

$logger->warning('Warning message', ['user_id' => 123]);
$logger->error('Error occurred', ['exception' => $e]);
```

**Laravel Log:**
```php
<?php

use Illuminate\Support\Facades\Log;

Log::info('User logged in', ['user_id' => 1]);
Log::error('Database error', ['query' => $sql]);

// Разные каналы
Log::channel('slack')->critical('Production error!');
```

---

## 🔄 PSR-4: Autoloading Standard

### Правила маппинга

**FQCN (Fully Qualified Class Name):**
```
\<NamespacePrefix>\(<SubNamespace>)*\<ClassName>
```

**Пример:**
```
\Acme\Log\Writer\File_Writer
 │    │   │      └─ ClassName
 │    │   └──────── SubNamespace
 │    └──────────── SubNamespace
 └───────────────── NamespacePrefix
```

**Маппинг:**
```
NamespacePrefix: Acme\Log → Base Directory: ./acme-log-writer/lib/
```

**Результат:**
```
\Acme\Log\Writer\File_Writer
→ ./acme-log-writer/lib/Writer/File_Writer.php
```

### Composer PSR-4

**composer.json:**
```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "Database\\Seeders\\": "database/seeders/",
            "Database\\Factories\\": "database/factories/"
        }
    }
}
```

**Примеры маппинга:**

| FQCN | File Path |
|------|-----------|
| `App\Models\User` | `src/Models/User.php` |
| `App\Http\Controllers\UserController` | `src/Http/Controllers/UserController.php` |
| `Database\Seeders\UserSeeder` | `database/seeders/UserSeeder.php` |

### Правила

1. **Namespace prefix** → **base directory**
2. `\` заменяется на `/` (directory separator)
3. Добавляется `.php`
4. **Case-sensitive** (регистрозависимо)

```php
<?php

// src/Models/User.php
namespace App\Models;

class User
{
    // ...
}

// Использование:
$user = new \App\Models\User();
```

### Несколько префиксов для одного namespace

```json
{
    "autoload": {
        "psr-4": {
            "App\\": ["src/", "lib/"]
        }
    }
}
```

Проверит:
1. `src/Models/User.php`
2. `lib/Models/User.php`

---

## 💾 PSR-6: Caching Interface

### CacheItemPoolInterface

```php
<?php

namespace Psr\Cache;

interface CacheItemPoolInterface
{
    public function getItem(string $key): CacheItemInterface;
    public function getItems(array $keys = []): iterable;
    public function hasItem(string $key): bool;
    public function clear(): bool;
    public function deleteItem(string $key): bool;
    public function deleteItems(array $keys): bool;
    public function save(CacheItemInterface $item): bool;
    public function saveDeferred(CacheItemInterface $item): bool;
    public function commit(): bool;
}
```

### CacheItemInterface

```php
<?php

namespace Psr\Cache;

interface CacheItemInterface
{
    public function getKey(): string;
    public function get(): mixed;
    public function isHit(): bool;
    public function set(mixed $value): static;
    public function expiresAt(?\DateTimeInterface $expiration): static;
    public function expiresAfter(int|\DateInterval|null $time): static;
}
```

### Использование

```php
<?php

use Psr\Cache\CacheItemPoolInterface;

class UserRepository
{
    public function __construct(
        private CacheItemPoolInterface $cache
    ) {}
    
    public function find(int $id): ?User
    {
        $item = $this->cache->getItem("user.{$id}");
        
        if ($item->isHit()) {
            return $item->get();
        }
        
        $user = $this->loadFromDatabase($id);
        
        $item->set($user);
        $item->expiresAfter(3600);  // 1 час
        $this->cache->save($item);
        
        return $user;
    }
    
    public function save(User $user): void
    {
        $this->saveToDatabase($user);
        
        // Invalidate cache
        $this->cache->deleteItem("user.{$user->id}");
    }
}
```

### Deferred save (batch)

```php
<?php

$pool = $container->get(CacheItemPoolInterface::class);

foreach ($users as $user) {
    $item = $pool->getItem("user.{$user->id}");
    $item->set($user);
    $item->expiresAfter(3600);
    
    $pool->saveDeferred($item);  // не сохраняет сразу
}

$pool->commit();  // сохранить все разом (эффективнее)
```

### Реализации PSR-6

- **Symfony Cache** (рекомендуется)
- **Laravel Cache** (адаптер)
- **Stash**

---

## 🌐 PSR-7: HTTP Message Interface

### RequestInterface

```php
<?php

namespace Psr\Http\Message;

interface RequestInterface extends MessageInterface
{
    public function getRequestTarget(): string;
    public function withRequestTarget(string $requestTarget): static;
    public function getMethod(): string;
    public function withMethod(string $method): static;
    public function getUri(): UriInterface;
    public function withUri(UriInterface $uri, bool $preserveHost = false): static;
}
```

### ServerRequestInterface (для HTTP сервера)

```php
<?php

namespace Psr\Http\Message;

interface ServerRequestInterface extends RequestInterface
{
    public function getServerParams(): array;
    public function getCookieParams(): array;
    public function withCookieParams(array $cookies): static;
    public function getQueryParams(): array;
    public function withQueryParams(array $query): static;
    public function getUploadedFiles(): array;
    public function withUploadedFiles(array $uploadedFiles): static;
    public function getParsedBody(): null|array|object;
    public function withParsedBody($data): static;
    public function getAttributes(): array;
    public function getAttribute(string $name, $default = null): mixed;
    public function withAttribute(string $name, $value): static;
    public function withoutAttribute(string $name): static;
}
```

### ResponseInterface

```php
<?php

namespace Psr\Http\Message;

interface ResponseInterface extends MessageInterface
{
    public function getStatusCode(): int;
    public function withStatus(int $code, string $reasonPhrase = ''): static;
    public function getReasonPhrase(): string;
}
```

### Immutability - все методы `with*()` возвращают новый объект

```php
<?php

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;

function handle(ServerRequestInterface $request): ResponseInterface
{
    // ❌ Не работает (объект immutable)
    $request->withAttribute('user_id', 123);
    
    // ✅ Правильно
    $request = $request->withAttribute('user_id', 123);
    
    // Цепочка вызовов
    $response = $response
        ->withStatus(200)
        ->withHeader('Content-Type', 'application/json')
        ->withBody($stream);
    
    return $response;
}
```

### Использование с Guzzle

```php
<?php

use GuzzleHttp\Client;
use GuzzleHttp\Psr7\Request;

$client = new Client();

$request = new Request(
    'POST',
    'https://api.example.com/users',
    ['Content-Type' => 'application/json'],
    json_encode(['name' => 'John'])
);

$response = $client->send($request);

echo $response->getStatusCode();  // 200
echo $response->getBody();        // JSON response
```

### Laravel PSR-7 adapter

```php
<?php

use Symfony\Bridge\PsrHttpMessage\Factory\PsrHttpFactory;
use Illuminate\Http\Request;

// Laravel Request → PSR-7
$psr7Request = app(PsrHttpFactory::class)->createRequest(
    Request::capture()
);

// PSR-7 → Laravel Response
$laravelResponse = app(PsrHttpFactory::class)->createResponse(
    $psr7Response
);
```

---

## 📦 PSR-11: Container Interface

### ContainerInterface

```php
<?php

namespace Psr\Container;

interface ContainerInterface
{
    public function get(string $id): mixed;
    public function has(string $id): bool;
}
```

### Использование

```php
<?php

use Psr\Container\ContainerInterface;

class UserController
{
    public function __construct(
        private ContainerInterface $container
    ) {}
    
    public function index(): void
    {
        if ($this->container->has(UserRepository::class)) {
            $repo = $this->container->get(UserRepository::class);
            $users = $repo->all();
        }
    }
}
```

### Laravel Container реализует PSR-11

```php
<?php

use Illuminate\Container\Container;
use Psr\Container\ContainerInterface;

$container = Container::getInstance();

// PSR-11 методы
$container->has(UserRepository::class);  // true
$service = $container->get(UserRepository::class);

// Laravel методы (расширение PSR-11)
$container->bind(Interface::class, Implementation::class);
$container->singleton(Service::class);
$container->make(UserController::class);
```

### Exceptions

```php
<?php

namespace Psr\Container;

interface NotFoundExceptionInterface extends ContainerExceptionInterface
{
    // Entry was not found in the container
}

interface ContainerExceptionInterface extends \Throwable
{
    // Base exception
}
```

**Использование:**
```php
<?php

use Psr\Container\NotFoundExceptionInterface;
use Psr\Container\ContainerInterface;

try {
    $service = $container->get('NonExistentService');
} catch (NotFoundExceptionInterface $e) {
    // Service not found
}
```

---

## 🔄 PSR-15: HTTP Server Request Handlers

### RequestHandlerInterface

```php
<?php

namespace Psr\Http\Server;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

interface RequestHandlerInterface
{
    public function handle(ServerRequestInterface $request): ResponseInterface;
}
```

### MiddlewareInterface

```php
<?php

namespace Psr\Http\Server;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

interface MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface;
}
```

### Пример middleware

```php
<?php

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class AuthMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        $token = $request->getHeader('Authorization')[0] ?? null;
        
        if (!$token) {
            return new Response(401, [], 'Unauthorized');
        }
        
        $user = $this->validateToken($token);
        $request = $request->withAttribute('user', $user);
        
        // Передать следующему middleware/handler
        return $handler->handle($request);
    }
}
```

### Pipeline (цепочка middleware)

```php
<?php

class Pipeline implements RequestHandlerInterface
{
    private array $middleware = [];
    
    public function pipe(MiddlewareInterface $middleware): void
    {
        $this->middleware[] = $middleware;
    }
    
    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $middleware = array_shift($this->middleware);
        
        if (!$middleware) {
            return $this->defaultHandler->handle($request);
        }
        
        return $middleware->process($request, $this);
    }
}

// Использование:
$pipeline = new Pipeline();
$pipeline->pipe(new AuthMiddleware());
$pipeline->pipe(new LoggingMiddleware());
$pipeline->pipe(new RateLimitMiddleware());

$response = $pipeline->handle($request);
```

---

## 💨 PSR-16: Simple Cache

**Упрощенная альтернатива PSR-6** (проще API).

### CacheInterface

```php
<?php

namespace Psr\SimpleCache;

interface CacheInterface
{
    public function get(string $key, mixed $default = null): mixed;
    public function set(string $key, mixed $value, null|int|\DateInterval $ttl = null): bool;
    public function delete(string $key): bool;
    public function clear(): bool;
    public function getMultiple(iterable $keys, mixed $default = null): iterable;
    public function setMultiple(iterable $values, null|int|\DateInterval $ttl = null): bool;
    public function deleteMultiple(iterable $keys): bool;
    public function has(string $key): bool;
}
```

### Использование (проще чем PSR-6)

```php
<?php

use Psr\SimpleCache\CacheInterface;

class UserRepository
{
    public function __construct(
        private CacheInterface $cache
    ) {}
    
    public function find(int $id): ?User
    {
        $key = "user.{$id}";
        
        // Получить с default значением
        $user = $this->cache->get($key);
        
        if ($user === null) {
            $user = $this->loadFromDatabase($id);
            $this->cache->set($key, $user, 3600);  // TTL в секундах
        }
        
        return $user;
    }
    
    public function findMultiple(array $ids): array
    {
        $keys = array_map(fn($id) => "user.{$id}", $ids);
        
        return $this->cache->getMultiple($keys);
    }
}
```

### PSR-6 vs PSR-16

| Feature | PSR-6 | PSR-16 |
|---------|-------|--------|
| API сложность | Сложный | Простой |
| Deferred save | ✅ | ❌ |
| Batch operations | Через getItems | getMultiple/setMultiple |
| Use case | Сложные сценарии | Простое кэширование |

**Когда использовать:**
- **PSR-16** - простое кэширование (90% случаев)
- **PSR-6** - нужен deferred save или сложная логика

---

## 🏭 PSR-17: HTTP Factories

**Создание PSR-7 объектов.**

### RequestFactoryInterface

```php
<?php

namespace Psr\Http\Message;

interface RequestFactoryInterface
{
    public function createRequest(string $method, $uri): RequestInterface;
}
```

### ResponseFactoryInterface

```php
<?php

namespace Psr\Http\Message;

interface ResponseFactoryInterface
{
    public function createResponse(int $code = 200, string $reasonPhrase = ''): ResponseInterface;
}
```

### Использование

```php
<?php

use Psr\Http\Message\RequestFactoryInterface;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\StreamFactoryInterface;

class ApiClient
{
    public function __construct(
        private RequestFactoryInterface $requestFactory,
        private ResponseFactoryInterface $responseFactory,
        private StreamFactoryInterface $streamFactory
    ) {}
    
    public function createRequest(string $method, string $uri, array $data = []): RequestInterface
    {
        $request = $this->requestFactory->createRequest($method, $uri);
        
        if (!empty($data)) {
            $body = $this->streamFactory->createStream(json_encode($data));
            $request = $request
                ->withBody($body)
                ->withHeader('Content-Type', 'application/json');
        }
        
        return $request;
    }
}
```

---

## 🌐 PSR-18: HTTP Client

### ClientInterface

```php
<?php

namespace Psr\Http\Client;

use Psr\Http\Message\RequestInterface;
use Psr\Http\Message\ResponseInterface;

interface ClientInterface
{
    public function sendRequest(RequestInterface $request): ResponseInterface;
}
```

### Использование

```php
<?php

use Psr\Http\Client\ClientInterface;
use Psr\Http\Message\RequestFactoryInterface;

class ApiService
{
    public function __construct(
        private ClientInterface $client,
        private RequestFactoryInterface $requestFactory
    ) {}
    
    public function fetchUser(int $id): array
    {
        $request = $this->requestFactory
            ->createRequest('GET', "https://api.example.com/users/{$id}")
            ->withHeader('Accept', 'application/json');
        
        $response = $this->client->sendRequest($request);
        
        if ($response->getStatusCode() !== 200) {
            throw new \RuntimeException('API request failed');
        }
        
        return json_decode((string) $response->getBody(), true);
    }
}
```

### Guzzle реализует PSR-18

```php
<?php

use GuzzleHttp\Client;
use Psr\Http\Client\ClientInterface;

$client = new Client();  // implements PSR-18

$request = new Request('GET', 'https://api.github.com/users/laravel');
$response = $client->sendRequest($request);

echo $response->getStatusCode();
echo $response->getBody();
```

---

## 📚 Сравнительная таблица PSR

| PSR | Название | Статус | Описание |
|-----|----------|--------|----------|
| PSR-1 | Basic Coding Standard | Accepted | Базовые правила кодирования |
| PSR-2 | Coding Style Guide | **Deprecated** | Заменен на PSR-12 |
| PSR-3 | Logger Interface | Accepted | Интерфейс логирования |
| PSR-4 | Autoloading Standard | Accepted | Автозагрузка классов |
| PSR-6 | Caching Interface | Accepted | Интерфейс кэширования (сложный) |
| PSR-7 | HTTP Message Interface | Accepted | HTTP запросы/ответы (immutable) |
| PSR-11 | Container Interface | Accepted | DI контейнер |
| PSR-12 | Extended Coding Style | Accepted | Расширенный кодстайл |
| PSR-15 | HTTP Server Request Handlers | Accepted | Middleware |
| PSR-16 | Simple Cache | Accepted | Упрощенное кэширование |
| PSR-17 | HTTP Factories | Accepted | Фабрики для PSR-7 |
| PSR-18 | HTTP Client | Accepted | HTTP клиент |

---

## 🎓 Best Practices

### 1. Всегда используй PSR-4 autoloading

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

### 2. Следуй PSR-12 для кодстайла

```php
<?php declare(strict_types=1);

namespace App\Services;

use App\Contracts\ServiceInterface;

class UserService implements ServiceInterface
{
    public function __construct(
        private UserRepository $repository
    ) {}
    
    public function findById(int $id): ?User
    {
        return $this->repository->find($id);
    }
}
```

### 3. Внедряй зависимости через интерфейсы

```php
<?php

use Psr\Log\LoggerInterface;
use Psr\Cache\CacheItemPoolInterface;

class UserService
{
    public function __construct(
        private LoggerInterface $logger,
        private CacheItemPoolInterface $cache
    ) {}
}
```

### 4. Используй PSR-16 для простого кэширования

```php
<?php

use Psr\SimpleCache\CacheInterface;

class ProductService
{
    public function __construct(
        private CacheInterface $cache
    ) {}
    
    public function getFeatured(): array
    {
        return $this->cache->get('products.featured', function() {
            return $this->loadFeatured();
        });
    }
}
```

---

## 🎓 Для собеседования: ключевые точки

1. **PSR-1** - <?php only, UTF-8, namespace + class names follow PSR-4
2. **PSR-12** - strict_types, 4 spaces, visibility required, opening brace rules
3. **PSR-3** - Logger interface, 8 levels (emergency → debug), context array
4. **PSR-4** - namespace → directory mapping, САМЫЙ ВАЖНЫЙ для собеса!
5. **PSR-6** - Cache pool + items, deferred save для batch
6. **PSR-7** - HTTP messages immutable, with*() методы
7. **PSR-11** - Container has() + get(), Laravel Container реализует
8. **PSR-16** - Simple cache, проще PSR-6, для большинства случаев
9. **PSR-15** - Middleware process() с RequestHandler

**Главное:** PSR обеспечивают совместимость между фреймворками, Laravel следует PSR (особенно PSR-3, PSR-4, PSR-11).

**Критично знать:** PSR-4 (autoloading), PSR-3 (logging), PSR-12 (code style).
