# REST API - Принципы, Laravel API Resources, Best Practices

Полное руководство по созданию RESTful API в Laravel.

---

## REST Принципы

### 1. Stateless (Без состояния)

```
Каждый запрос независим и содержит всю необходимую информацию.
Сервер не хранит состояние клиента между запросами.

✅ ХОРОШО: Authorization: Bearer <token>
❌ ПЛОХО: Полагаться на сессии
```

### 2. Client-Server Architecture

```
Разделение ответственности:
- Клиент: UI, пользовательский опыт
- Сервер: данные, бизнес-логика
```

### 3. Cacheable (Кешируемость)

```
Ответы должны явно указывать, можно ли их кешировать.

Cache-Control: max-age=3600, public
ETag: "686897696a7c876b7e"
```

### 4. Layered System

```
Клиент не знает, общается ли он напрямую с сервером
или через промежуточные слои (load balancer, proxy, cache).
```

### 5. Uniform Interface

```
Единообразный интерфейс:
- Идентификация ресурсов (URI)
- Манипуляция через представления
- Самоописательные сообщения
- HATEOAS (опционально)
```

---

## HTTP Methods (Методы)

### Семантика методов

```http
GET     /users          - Получить список пользователей (безопасный, идемпотентный)
GET     /users/1        - Получить пользователя #1
POST    /users          - Создать нового пользователя (не идемпотентный)
PUT     /users/1        - Полностью заменить пользователя #1 (идемпотентный)
PATCH   /users/1        - Частично обновить пользователя #1 (идемпотентный)
DELETE  /users/1        - Удалить пользователя #1 (идемпотентный)

HEAD    /users/1        - Как GET, но только заголовки (без body)
OPTIONS /users          - Какие методы доступны для ресурса
```

### Идемпотентность

```
Идемпотентный = повторный запрос не меняет результат

GET    /users/1    ✅ Идемпотентный (всегда вернет то же)
POST   /users      ❌ НЕ идемпотентный (каждый раз новый user)
PUT    /users/1    ✅ Идемпотентный (результат тот же)
PATCH  /users/1    ✅ Идемпотентный (обычно)
DELETE /users/1    ✅ Идемпотентный (повторное удаление = 404)
```

---

## Resource Naming (Именование ресурсов)

### Правила именования

```http
✅ ХОРОШО - существительные во множественном числе
GET  /users
GET  /posts
GET  /comments

❌ ПЛОХО - глаголы
GET  /getUsers
POST /createUser

✅ ХОРОШО - вложенные ресурсы
GET  /users/1/posts
GET  /posts/5/comments

❌ ПЛОХО - слишком глубокая вложенность (больше 2 уровней)
GET  /users/1/posts/5/comments/3/likes

✅ ЛУЧШЕ
GET  /comments/3/likes

✅ ХОРОШО - фильтрация через query params
GET  /users?status=active&role=admin

❌ ПЛОХО
GET  /users/active/admin

✅ ХОРОШО - kebab-case для составных слов
GET  /order-items
GET  /user-settings

❌ ПЛОХО
GET  /orderItems
GET  /OrderItems
```

### Примеры endpoints

```http
# Пользователи
GET     /api/users              - Список пользователей
GET     /api/users/1            - Конкретный пользователь
POST    /api/users              - Создать пользователя
PUT     /api/users/1            - Обновить пользователя (все поля)
PATCH   /api/users/1            - Обновить пользователя (частично)
DELETE  /api/users/1            - Удалить пользователя

# Посты пользователя
GET     /api/users/1/posts      - Посты пользователя #1
POST    /api/users/1/posts      - Создать пост для пользователя #1

# Комментарии к посту
GET     /api/posts/5/comments   - Комментарии к посту #5
POST    /api/posts/5/comments   - Добавить комментарий к посту #5

# Не-CRUD действия (используй существительные или подресурсы)
POST    /api/users/1/follow     - Подписаться на пользователя
DELETE  /api/users/1/follow     - Отписаться
POST    /api/posts/5/publish    - Опубликовать пост
POST    /api/orders/10/cancel   - Отменить заказ
```

---

## HTTP Status Codes

### 2xx Success

```http
200 OK                    - Успешный GET, PUT, PATCH, DELETE
201 Created               - Успешный POST (создание ресурса)
                           + заголовок Location: /users/123
202 Accepted              - Запрос принят, но обработка не завершена (async)
204 No Content            - Успешно, но нет body (DELETE, некоторые PUT/PATCH)
```

### 3xx Redirection

```http
301 Moved Permanently     - Ресурс перемещен навсегда
302 Found                 - Временный редирект
304 Not Modified          - Клиент может использовать кешированную версию
```

### 4xx Client Errors

```http
400 Bad Request           - Невалидные данные (не прошла валидация)
401 Unauthorized          - Не аутентифицирован (нет токена или невалидный)
403 Forbidden             - Аутентифицирован, но нет прав доступа
404 Not Found             - Ресурс не найден
405 Method Not Allowed    - HTTP метод не разрешен для endpoint
409 Conflict              - Конфликт (например, duplicate email)
422 Unprocessable Entity  - Валидация не прошла (Laravel default)
429 Too Many Requests     - Rate limit превышен
```

### 5xx Server Errors

```http
500 Internal Server Error - Ошибка на сервере
502 Bad Gateway           - Ошибка прокси/шлюза
503 Service Unavailable   - Сервис временно недоступен (maintenance)
504 Gateway Timeout       - Таймаут прокси/шлюза
```

---

## Laravel API Routes

### routes/api.php

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\UserController;
use App\Http\Controllers\Api\PostController;

// Префикс /api автоматически добавляется
// Middleware 'api' применяется по умолчанию

// Простые маршруты
Route::get('/users', [UserController::class, 'index']);
Route::post('/users', [UserController::class, 'store']);

// Resource routes
Route::apiResource('users', UserController::class);
/*
GET     /api/users              index
POST    /api/users              store
GET     /api/users/{user}       show
PUT     /api/users/{user}       update
PATCH   /api/users/{user}       update
DELETE  /api/users/{user}       destroy
*/

// Вложенные ресурсы
Route::apiResource('users.posts', PostController::class);
/*
GET     /api/users/{user}/posts
POST    /api/users/{user}/posts
GET     /api/users/{user}/posts/{post}
PUT     /api/users/{user}/posts/{post}
PATCH   /api/users/{user}/posts/{post}
DELETE  /api/users/{user}/posts/{post}
*/

// Только определенные методы
Route::apiResource('users', UserController::class)
    ->only(['index', 'show']);

Route::apiResource('posts', PostController::class)
    ->except(['destroy']);

// Именование маршрутов
Route::apiResource('users', UserController::class)
    ->names([
        'index' => 'users.all',
        'show' => 'users.find',
    ]);

// Группировка с middleware
Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('posts', PostController::class);
    Route::apiResource('comments', CommentController::class);
});

// Версионирование API
Route::prefix('v1')->group(function () {
    Route::apiResource('users', 'Api\V1\UserController');
});

Route::prefix('v2')->group(function () {
    Route::apiResource('users', 'Api\V2\UserController');
});
```

---

## API Controller

### Базовый контроллер

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;

class UserController extends Controller
{
    /**
     * GET /api/users
     */
    public function index()
    {
        $users = User::all();
        
        return response()->json([
            'data' => $users,
        ]);
    }

    /**
     * POST /api/users
     */
    public function store(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8',
        ]);

        $validated['password'] = bcrypt($validated['password']);
        $user = User::create($validated);

        return response()->json([
            'data' => $user,
        ], 201); // 201 Created
    }

    /**
     * GET /api/users/{id}
     */
    public function show(User $user)
    {
        return response()->json([
            'data' => $user,
        ]);
    }

    /**
     * PUT/PATCH /api/users/{id}
     */
    public function update(Request $request, User $user)
    {
        $validated = $request->validate([
            'name' => 'sometimes|max:255',
            'email' => 'sometimes|email|unique:users,email,' . $user->id,
        ]);

        $user->update($validated);

        return response()->json([
            'data' => $user,
        ]);
    }

    /**
     * DELETE /api/users/{id}
     */
    public function destroy(User $user)
    {
        $user->delete();

        return response()->json(null, 204); // 204 No Content
    }
}
```

---

## API Resources (Трансформация данных)

### Создание Resource

```bash
php artisan make:resource UserResource
php artisan make:resource UserCollection
```

### UserResource

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * Transform the resource into an array.
     */
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
            
            // Условная загрузка отношений
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            
            // Условные поля
            'is_admin' => $this->when($this->isAdmin(), true),
            'secret' => $this->when($request->user()?->isAdmin(), $this->secret),
            
            // Merge дополнительных данных
            $this->mergeWhen($this->isPremium(), [
                'premium_expires_at' => $this->premium_expires_at,
                'premium_level' => $this->premium_level,
            ]),
        ];
    }
    
    /**
     * Customize the response
     */
    public function with($request): array
    {
        return [
            'version' => '1.0',
            'timestamp' => now()->toISOString(),
        ];
    }
}
```

### Использование в контроллере

```php
use App\Http\Resources\UserResource;
use App\Http\Resources\UserCollection;

class UserController extends Controller
{
    public function index()
    {
        $users = User::with('posts')->paginate(15);
        
        return UserResource::collection($users);
        // или
        return new UserCollection($users);
    }

    public function show(User $user)
    {
        return new UserResource($user->load('posts'));
    }

    public function store(Request $request)
    {
        $user = User::create($request->validated());
        
        return (new UserResource($user))
            ->response()
            ->setStatusCode(201)
            ->header('X-Custom-Header', 'Value');
    }
}
```

### Collection Resource

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    /**
     * Transform the resource collection into an array.
     */
    public function toArray($request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total_users' => $this->collection->count(),
                'active_users' => $this->collection->where('status', 'active')->count(),
            ],
            'links' => [
                'self' => route('users.index'),
            ],
        ];
    }
}
```

### Пагинация с Resources

```php
public function index()
{
    $users = User::paginate(15);
    
    return UserResource::collection($users);
}

// Ответ:
{
    "data": [...],
    "links": {
        "first": "http://example.com/api/users?page=1",
        "last": "http://example.com/api/users?page=10",
        "prev": null,
        "next": "http://example.com/api/users?page=2"
    },
    "meta": {
        "current_page": 1,
        "from": 1,
        "last_page": 10,
        "per_page": 15,
        "to": 15,
        "total": 150
    }
}
```

---

## Обработка ошибок

### Custom Exception Handler

**app/Exceptions/Handler.php:**

```php
<?php

namespace App\Exceptions;

use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Throwable;

class Handler extends ExceptionHandler
{
    public function register(): void
    {
        $this->renderable(function (Throwable $e, $request) {
            if ($request->is('api/*')) {
                return $this->handleApiException($e);
            }
        });
    }

    protected function handleApiException(Throwable $exception)
    {
        $statusCode = 500;
        $message = 'Internal Server Error';

        if ($exception instanceof ModelNotFoundException) {
            $statusCode = 404;
            $message = 'Resource not found';
        } elseif ($exception instanceof NotFoundHttpException) {
            $statusCode = 404;
            $message = 'Endpoint not found';
        } elseif ($exception instanceof ValidationException) {
            $statusCode = 422;
            return response()->json([
                'message' => 'Validation failed',
                'errors' => $exception->errors(),
            ], $statusCode);
        }

        return response()->json([
            'message' => $message,
            'error' => config('app.debug') ? $exception->getMessage() : null,
            'trace' => config('app.debug') ? $exception->getTraceAsString() : null,
        ], $statusCode);
    }
}
```

### Формат ошибок

```json
// 404 Not Found
{
    "message": "Resource not found"
}

// 422 Validation Error
{
    "message": "Validation failed",
    "errors": {
        "email": [
            "The email field is required.",
            "The email must be a valid email address."
        ],
        "password": [
            "The password must be at least 8 characters."
        ]
    }
}

// 500 Internal Server Error (debug mode)
{
    "message": "Internal Server Error",
    "error": "Division by zero",
    "trace": "..."
}
```

---

## Фильтрация, Сортировка, Поиск

### Query Parameters

```http
GET /api/users?status=active&role=admin&sort=-created_at&search=john&page=2
```

### Контроллер

```php
class UserController extends Controller
{
    public function index(Request $request)
    {
        $query = User::query();
        
        // Фильтрация
        if ($request->has('status')) {
            $query->where('status', $request->status);
        }
        
        if ($request->has('role')) {
            $query->where('role', $request->role);
        }
        
        // Поиск
        if ($request->has('search')) {
            $query->where('name', 'like', '%' . $request->search . '%')
                ->orWhere('email', 'like', '%' . $request->search . '%');
        }
        
        // Сортировка
        $sortField = $request->get('sort', 'created_at');
        $sortDirection = 'asc';
        
        if (str_starts_with($sortField, '-')) {
            $sortField = substr($sortField, 1);
            $sortDirection = 'desc';
        }
        
        $query->orderBy($sortField, $sortDirection);
        
        // Пагинация
        $users = $query->paginate($request->get('per_page', 15));
        
        return UserResource::collection($users);
    }
}
```

### Query Builder Package (spatie/laravel-query-builder)

```bash
composer require spatie/laravel-query-builder
```

```php
use Spatie\QueryBuilder\QueryBuilder;

class UserController extends Controller
{
    public function index(Request $request)
    {
        $users = QueryBuilder::for(User::class)
            ->allowedFilters(['status', 'role', 'email'])
            ->allowedSorts('name', 'created_at', 'email')
            ->allowedIncludes('posts', 'comments')
            ->paginate($request->get('per_page', 15));
        
        return UserResource::collection($users);
    }
}

// Использование:
// GET /api/users?filter[status]=active&filter[role]=admin&sort=-created_at&include=posts
```

---

## Версионирование API

### 1. URL Versioning (рекомендуется)

```http
GET /api/v1/users
GET /api/v2/users
```

```php
// routes/api.php
Route::prefix('v1')->namespace('Api\V1')->group(function () {
    Route::apiResource('users', 'UserController');
});

Route::prefix('v2')->namespace('Api\V2')->group(function () {
    Route::apiResource('users', 'UserController');
});
```

### 2. Header Versioning

```http
GET /api/users
Accept: application/vnd.api.v1+json
```

```php
Route::middleware('api.version:1')->group(function () {
    Route::apiResource('users', 'Api\V1\UserController');
});

// Middleware
class ApiVersion
{
    public function handle($request, Closure $next, $version)
    {
        if ($request->header('Accept') !== "application/vnd.api.v{$version}+json") {
            return response()->json(['error' => 'Invalid API version'], 400);
        }
        
        return $next($request);
    }
}
```

### 3. Query Parameter (не рекомендуется)

```http
GET /api/users?version=1
```

---

## Rate Limiting

### Конфигурация

```php
// app/Providers/RouteServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

protected function configureRateLimiting()
{
    // По IP
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->ip());
    });
    
    // По пользователю
    RateLimiter::for('api', function (Request $request) {
        return $request->user()
            ? Limit::perMinute(100)->by($request->user()->id)
            : Limit::perMinute(20)->by($request->ip());
    });
    
    // Разные лимиты для премиум
    RateLimiter::for('api', function (Request $request) {
        return $request->user()?->isPremium()
            ? Limit::none()
            : Limit::perMinute(60);
    });
}
```

### Применение

```php
// routes/api.php
Route::middleware('throttle:api')->group(function () {
    Route::apiResource('users', UserController::class);
});

// Кастомный лимит
Route::middleware('throttle:10,1')->group(function () {
    // 10 запросов в минуту
});
```

### Headers в ответе

```http
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 59
Retry-After: 54
```

---

## CORS (Cross-Origin Resource Sharing)

### Конфигурация

```php
// config/cors.php
return [
    'paths' => ['api/*', 'sanctum/csrf-cookie'],
    
    'allowed_methods' => ['*'],
    
    'allowed_origins' => ['*'], // Production: ['https://frontend.com']
    
    'allowed_origins_patterns' => [],
    
    'allowed_headers' => ['*'],
    
    'exposed_headers' => [],
    
    'max_age' => 0,
    
    'supports_credentials' => true,
];
```

### Middleware

```php
// app/Http/Kernel.php
protected $middleware = [
    \Fruitcake\Cors\HandleCors::class, // Должен быть первым
    // ...
];
```

---

## Best Practices

### 1. Используй API Resources для трансформации

```php
// ✅ ХОРОШО
return new UserResource($user);

// ❌ ПЛОХО - возвращать модель напрямую
return $user;
```

### 2. Версионируй API через URL

```php
// ✅ ХОРОШО
GET /api/v1/users

// ❌ ПЛОХО
GET /api/users?version=1
```

### 3. Используй правильные HTTP коды

```php
// ✅ ХОРОШО
return response()->json($user, 201); // Создание
return response()->json(null, 204); // Удаление

// ❌ ПЛОХО
return response()->json(['success' => true], 200);
```

### 4. Валидация через Form Requests

```php
// ✅ ХОРОШО
public function store(StoreUserRequest $request)
{
    $user = User::create($request->validated());
}

// ❌ ПЛОХО - валидация в контроллере
```

### 5. Единый формат ответов

```json
// Success
{
    "data": {...},
    "meta": {...},
    "links": {...}
}

// Error
{
    "message": "Error message",
    "errors": {...}
}
```

### 6. Пагинация для списков

```php
// ✅ ХОРОШО
$users = User::paginate(15);

// ❌ ПЛОХО
$users = User::all(); // Может вернуть миллионы записей
```

### 7. Eager Loading для избежания N+1

```php
// ✅ ХОРОШО
$users = User::with('posts')->get();

// ❌ ПЛОХО
$users = User::all(); // N+1 при обращении к $user->posts
```

### 8. Фильтрация и поиск через query params

```http
GET /api/users?status=active&search=john&sort=-created_at
```

### 9. Rate Limiting

```php
Route::middleware('throttle:60,1')->group(function () {
    // 60 запросов в минуту
});
```

### 10. Документация API

Используй Swagger/OpenAPI, Postman, или Laravel Scribe:

```bash
composer require knuckleswtf/scribe
php artisan scribe:generate
```
