# Архитектура и жизненный цикл Laravel

## Request Lifecycle

### Полный цикл обработки запроса

```
HTTP Request
    ↓
public/index.php
    ↓
Bootstrap (autoload, app creation)
    ↓
Kernel (HTTP/Console)
    ↓
Service Providers (register + boot)
    ↓
Middleware Pipeline
    ↓
Router → Controller/Closure
    ↓
Response
    ↓
Middleware (terminate methods)
    ↓
HTTP Response
```

### Детальный разбор

#### 1. Entry Point (public/index.php)

```php
// Composer autoload
require __DIR__.'/../vendor/autoload.php';

// Создание Application
$app = require_once __DIR__.'/../bootstrap/app.php';

// Создание Kernel
$kernel = $app->make(Kernel::class);

// Обработка запроса
$response = $kernel->handle(
    $request = Request::capture()
);

// Отправка ответа
$response->send();

// Terminate
$kernel->terminate($request, $response);
```

#### 2. Application Bootstrap (bootstrap/app.php)

```php
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);

// Регистрация core bindings
$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);

return $app;
```

#### 3. HTTP Kernel

```php
// app/Http/Kernel.php
class Kernel extends HttpKernel
{
    protected $middleware = [
        // Global middleware (выполняется для всех запросов)
        \App\Http\Middleware\TrustProxies::class,
        \Illuminate\Http\Middleware\HandleCors::class,
        \App\Http\Middleware\PreventRequestsDuringMaintenance::class,
        \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
        \App\Http\Middleware\TrimStrings::class,
        \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
    ];

    protected $middlewareGroups = [
        'web' => [
            \App\Http\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \App\Http\Middleware\VerifyCsrfToken::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],
        'api' => [
            \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            'throttle:api',
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],
    ];

    protected $routeMiddleware = [
        // Route-specific middleware
        'auth' => \App\Http\Middleware\Authenticate::class,
        'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
        'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
        // ...
    ];
}
```

**Метод handle:**
```php
public function handle($request)
{
    // 1. Bootstrap приложения
    $this->bootstrap();
    
    // 2. Пропуск через middleware pipeline
    return (new Pipeline($this->app))
        ->send($request)
        ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
        ->then($this->dispatchToRouter());
}
```

---

## Service Container (IoC Container)

### Основа Laravel

Service Container - это мощный инструмент для управления зависимостями и выполнения инверсии управления (IoC).

### Binding (привязка)

```php
// Простая привязка
app()->bind(UserRepositoryInterface::class, EloquentUserRepository::class);

// Singleton (одна инстанция на весь request)
app()->singleton(ApiService::class, function ($app) {
    return new ApiService(
        $app->make(HttpClient::class),
        config('services.api.key')
    );
});

// Instance (готовый объект)
$service = new MyService();
app()->instance(MyService::class, $service);

// Scoped (singleton в пределах HTTP запроса, новый в queue job)
app()->scoped(ConnectionManager::class);

// Bind if (только если еще не привязано)
app()->bindIf(ServiceInterface::class, ServiceImplementation::class);
```

### Resolving (извлечение)

```php
// Через app()
$service = app(UserService::class);

// Через make()
$service = app()->make(UserService::class);

// Через resolve()
$service = resolve(UserService::class);

// Через dependency injection (автоматически)
class UserController extends Controller
{
    public function __construct(
        private UserService $userService  // Автоматически resolve
    ) {}
}
```

### Automatic Dependency Injection

```php
class UserService
{
    public function __construct(
        private UserRepository $repository,
        private Mailer $mailer,
        private EventDispatcher $events
    ) {}
    // Все зависимости автоматически resolved из container
}

// При вызове:
$service = app(UserService::class);
// Laravel автоматически создаст UserRepository, Mailer, EventDispatcher
```

### Contextual Binding

```php
// Разные реализации для разных классов
app()->when(PhotoController::class)
    ->needs(Filesystem::class)
    ->give(function () {
        return Storage::disk('local');
    });

app()->when(VideoController::class)
    ->needs(Filesystem::class)
    ->give(function () {
        return Storage::disk('s3');
    });
```

### Tagging

```php
// Регистрация с тегами
app()->tag([CpuReport::class, MemoryReport::class], 'reports');

// Получение всех tagged
foreach (app()->tagged('reports') as $report) {
    $report->generate();
}
```

### Extending Services

```php
// Расширение уже зарегистрированного сервиса
app()->extend(PaymentGateway::class, function ($service, $app) {
    return new CachedPaymentGateway($service, $app->make(Cache::class));
});
```

---

## Service Providers

### Анатомия Service Provider

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register - регистрация bindings
     * Вызывается ДО boot() всех провайдеров
     */
    public function register(): void
    {
        // Регистрация в container
        $this->app->singleton(ApiService::class, function ($app) {
            return new ApiService(
                config('services.api.url'),
                config('services.api.key')
            );
        });
        
        // Merge config
        $this->mergeConfigFrom(
            __DIR__.'/../../config/custom.php', 'custom'
        );
    }

    /**
     * Boot - выполнение инициализации
     * Вызывается ПОСЛЕ register() всех провайдеров
     * Здесь доступны все зарегистрированные сервисы
     */
    public function boot(): void
    {
        // Маршруты
        if ($this->app->routesAreCached()) {
            // ...
        }
        
        // View composers
        View::composer('profile', function ($view) {
            $view->with('count', User::count());
        });
        
        // Validators
        Validator::extend('custom_rule', function ($attribute, $value, $parameters, $validator) {
            return $value === 'valid';
        });
        
        // Events
        Event::listen(OrderShipped::class, SendShipmentNotification::class);
        
        // Commands (для Console Kernel)
        if ($this->app->runningInConsole()) {
            $this->commands([
                CustomCommand::class,
            ]);
        }
        
        // Migrations
        $this->loadMigrationsFrom(__DIR__.'/../../database/migrations');
        
        // Translations
        $this->loadTranslationsFrom(__DIR__.'/../../lang', 'custom');
        
        // Views
        $this->loadViewsFrom(__DIR__.'/../../resources/views', 'custom');
        
        // Publish assets
        $this->publishes([
            __DIR__.'/../../config/custom.php' => config_path('custom.php'),
        ], 'config');
    }
}
```

### Deferred Providers

Для оптимизации загрузки:

```php
class RiakServiceProvider extends ServiceProvider
{
    /**
     * Indicates if loading of the provider is deferred.
     */
    protected $defer = true;  // Laravel 9
    // или implements DeferrableProvider в Laravel 10+

    public function register(): void
    {
        $this->app->singleton(Connection::class, function ($app) {
            return new Connection(config('riak'));
        });
    }

    /**
     * Какие сервисы предоставляет этот провайдер
     */
    public function provides(): array
    {
        return [Connection::class];
    }
}
```

- Provider не загружается при каждом запросе
- Загружается только когда запрашивается `Connection::class`
- Ускоряет bootstrap

### Регистрация провайдеров

```php
// config/app.php
'providers' => [
    // Laravel Framework Service Providers
    Illuminate\Auth\AuthServiceProvider::class,
    Illuminate\Broadcasting\BroadcastServiceProvider::class,
    // ...

    // Package Service Providers
    Laravel\Tinker\TinkerServiceProvider::class,

    // Application Service Providers
    App\Providers\AppServiceProvider::class,
    App\Providers\AuthServiceProvider::class,
    App\Providers\EventServiceProvider::class,
    App\Providers\RouteServiceProvider::class,
],
```

---

## Facades

### Что такое Facade

Facade предоставляет статический интерфейс к объектам в Service Container.

```php
// Без Facade
app('cache')->get('key');
app(Illuminate\Cache\CacheManager::class)->get('key');

// С Facade
Cache::get('key');
```

### Анатомия Facade

```php
namespace Illuminate\Support\Facades;

class Cache extends Facade
{
    /**
     * Возвращает имя binding в container
     */
    protected static function getFacadeAccessor()
    {
        return 'cache';
    }
}
```

**Как это работает:**

1. `Cache::get('key')` вызывается
2. PHP вызывает `__callStatic` (нет статического метода get)
3. Facade::__callStatic ищет `getFacadeAccessor()` → `'cache'`
4. Достает из container: `app('cache')`
5. Вызывает на нем метод: `app('cache')->get('key')`

### Создание собственного Facade

```php
// 1. Создаем класс
namespace App\Services;

class PaymentService
{
    public function charge($amount)
    {
        return "Charged: $amount";
    }
}

// 2. Регистрируем в Service Provider
public function register()
{
    $this->app->singleton('payment', function () {
        return new \App\Services\PaymentService();
    });
}

// 3. Создаем Facade
namespace App\Facades;

use Illuminate\Support\Facades\Facade;

class Payment extends Facade
{
    protected static function getFacadeAccessor()
    {
        return 'payment';
    }
}

// 4. Alias в config/app.php (опционально)
'aliases' => [
    'Payment' => App\Facades\Payment::class,
],

// 5. Используем
Payment::charge(100);
```

### Real-time Facades

```php
// Вместо создания facade вручную:
use Facades\App\Services\PaymentService;

PaymentService::charge(100);
// Laravel автоматически создаст facade для PaymentService
```

### Facade vs Dependency Injection

**Facade:**
```php
class UserController extends Controller
{
    public function index()
    {
        $users = Cache::remember('users', 60, function () {
            return User::all();
        });
    }
}
```

**Dependency Injection:**
```php
class UserController extends Controller
{
    public function __construct(
        private CacheManager $cache
    ) {}
    
    public function index()
    {
        $users = $this->cache->remember('users', 60, function () {
            return User::all();
        });
    }
}
```

**Когда использовать что:**
- **Facades**: быстрое прототипирование, простые скрипты, глобальные helpers
- **DI**: тестируемый код, четкие зависимости, SOLID принципы

---

## Pipeline

Pipeline - паттерн для пропуска объекта через серию "труб" (pipes).

### Middleware Pipeline

```php
return (new Pipeline($this->app))
    ->send($request)
    ->through([
        Middleware1::class,
        Middleware2::class,
        Middleware3::class,
    ])
    ->then(function ($request) {
        return $this->router->dispatch($request);
    });
```

**Как это работает:**

```php
// Middleware1
public function handle($request, Closure $next)
{
    // До обработки
    echo "Before 1\n";
    
    $response = $next($request);
    
    // После обработки
    echo "After 1\n";
    
    return $response;
}

// Middleware2
public function handle($request, Closure $next)
{
    echo "Before 2\n";
    $response = $next($request);
    echo "After 2\n";
    return $response;
}

// Вывод:
// Before 1
// Before 2
// [Router handles request]
// After 2
// After 1
```

### Собственный Pipeline

```php
use Illuminate\Pipeline\Pipeline;

$result = app(Pipeline::class)
    ->send($data)
    ->through([
        ValidateData::class,
        SanitizeData::class,
        TransformData::class,
    ])
    ->thenReturn();  // Возвращает результат последней pipe
```

**Пример Pipe:**
```php
class ValidateData
{
    public function handle($data, Closure $next)
    {
        if (!isset($data['email'])) {
            throw new ValidationException('Email required');
        }
        
        return $next($data);
    }
}
```

---

## Bootstrap Process

### Bootstrappers

Kernel запускает bootstrappers в определенном порядке:

```php
// Illuminate\Foundation\Http\Kernel
protected $bootstrappers = [
    \Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class,
    \Illuminate\Foundation\Bootstrap\LoadConfiguration::class,
    \Illuminate\Foundation\Bootstrap\HandleExceptions::class,
    \Illuminate\Foundation\Bootstrap\RegisterFacades::class,
    \Illuminate\Foundation\Bootstrap\RegisterProviders::class,
    \Illuminate\Foundation\Bootstrap\BootProviders::class,
];
```

1. **LoadEnvironmentVariables** - загружает .env файл
2. **LoadConfiguration** - загружает config файлы
3. **HandleExceptions** - настраивает error/exception handling
4. **RegisterFacades** - регистрирует facade aliases
5. **RegisterProviders** - вызывает register() всех Service Providers
6. **BootProviders** - вызывает boot() всех Service Providers

---

## Ключевые концепции

### 1. Application Instance

Singleton для всего приложения:
```php
$app = app();  // Всегда один и тот же объект
```

### 2. Contracts vs Facades

**Contract** (интерфейс):
```php
use Illuminate\Contracts\Cache\Repository as CacheContract;

class UserService
{
    public function __construct(
        private CacheContract $cache
    ) {}
}
```

**Facade** (статический прокси):
```php
use Illuminate\Support\Facades\Cache;

Cache::put('key', 'value');
```

### 3. Макросы

Расширение классов Laravel:

```php
// В Service Provider
use Illuminate\Support\Str;

Str::macro('partNumber', function () {
    return static::upper(static::random(10));
});

// Использование
Str::partNumber();  // 'ABCDEF1234'
```

---

## Производительность

### Config Caching

```bash
php artisan config:cache
```
- Создает `bootstrap/cache/config.php`
- Объединяет все config файлы
- `env()` НЕ работает вне config файлов после кеширования!

### Route Caching

```bash
php artisan route:cache
```
- Сериализует все маршруты
- Closures НЕ могут быть сериализованы (используйте контроллеры)

### Event Caching

```bash
php artisan event:cache
```

### View Caching

Автоматически компилируется в `storage/framework/views/`

### Optimization

```bash
# Все оптимизации сразу
php artisan optimize

# Очистка оптимизаций
php artisan optimize:clear
```

---

## Ключевые вопросы для интервью

- Опишите полный жизненный цикл HTTP запроса в Laravel
- Что такое Service Container и как он работает?
- В чем разница между bind, singleton, и scoped?
- Когда вызываются register() и boot() в Service Providers?
- Как работают Facades под капотом?
- Что такое Pipeline и где используется?
- Что происходит при вызове `Cache::get('key')`?
- Как работает автоматический dependency injection?
- Что такое deferred service providers и зачем они нужны?
- Чем отличается contextual binding от обычного?
