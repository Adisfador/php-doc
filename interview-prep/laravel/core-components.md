# Laravel Core Components - Routing, Middleware, Controllers

Ключевые компоненты Laravel для обработки HTTP запросов.

---

## Routing (Маршрутизация)

### Базовые маршруты

```php
<?php

use Illuminate\Support\Facades\Route;

// Простые маршруты
Route::get('/users', function () {
    return 'Users list';
});

Route::post('/users', function () {
    return 'Create user';
});

Route::put('/users/{id}', function ($id) {
    return "Update user $id";
});

Route::patch('/users/{id}', function ($id) {
    return "Partial update user $id";
});

Route::delete('/users/{id}', function ($id) {
    return "Delete user $id";
});

// Несколько HTTP методов
Route::match(['get', 'post'], '/form', function () {
    //
});

// Любой HTTP метод
Route::any('/any', function () {
    //
});

// Redirect
Route::redirect('/old-url', '/new-url');
Route::redirect('/old-url', '/new-url', 301); // Permanent

Route::permanentRedirect('/old-url', '/new-url'); // 301

// View без контроллера
Route::view('/welcome', 'welcome');
Route::view('/welcome', 'welcome', ['name' => 'John']); // С данными
```

### Параметры маршрута

```php
// Обязательный параметр
Route::get('/user/{id}', function ($id) {
    return "User $id";
});

// Несколько параметров
Route::get('/posts/{post}/comments/{comment}', function ($post, $comment) {
    return "Post $post, Comment $comment";
});

// Опциональный параметр
Route::get('/user/{name?}', function ($name = 'Guest') {
    return "Hello, $name";
});

// С значением по умолчанию
Route::get('/user/{name?}', function ($name = 'John') {
    return $name;
});

// Регулярные выражения (constraints)
Route::get('/user/{id}', function ($id) {
    //
})->where('id', '[0-9]+');

Route::get('/user/{name}', function ($name) {
    //
})->where('name', '[A-Za-z]+');

Route::get('/user/{id}/{name}', function ($id, $name) {
    //
})->where(['id' => '[0-9]+', 'name' => '[a-z]+']);

// Предопределенные паттерны
Route::get('/user/{id}', function ($id) {
    //
})->whereNumber('id');

Route::get('/user/{name}', function ($name) {
    //
})->whereAlpha('name');

Route::get('/user/{name}', function ($name) {
    //
})->whereAlphaNumeric('name');

Route::get('/user/{id}', function ($id) {
    //
})->whereUuid('id');

Route::get('/user/{id}', function ($id) {
    //
})->whereUlid('id');

// Глобальные constraints (RouteServiceProvider)
public function boot()
{
    Route::pattern('id', '[0-9]+');
}
```

### Именованные маршруты

```php
// Назначение имени
Route::get('/user/profile', function () {
    //
})->name('profile');

// Использование
$url = route('profile'); // /user/profile

// С параметрами
Route::get('/user/{id}/profile', function ($id) {
    //
})->name('profile');

$url = route('profile', ['id' => 1]); // /user/1/profile

// В редиректах
return redirect()->route('profile');
return redirect()->route('profile', ['id' => 1]);

// В Blade шаблонах
<a href="{{ route('profile') }}">Profile</a>

// Проверка текущего маршрута
if (request()->routeIs('profile')) {
    // Текущий маршрут - profile
}

if (request()->routeIs('user.*')) {
    // Любой маршрут начинающийся с user.
}
```

### Группы маршрутов

```php
// Префикс
Route::prefix('admin')->group(function () {
    Route::get('/users', function () {
        // /admin/users
    });
    Route::get('/posts', function () {
        // /admin/posts
    });
});

// Имя (prefix для имен)
Route::name('admin.')->group(function () {
    Route::get('/users', function () {
        //
    })->name('users'); // admin.users
});

// Middleware
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', function () {
        //
    });
});

// Комбинированная группа
Route::prefix('admin')
    ->name('admin.')
    ->middleware('auth')
    ->group(function () {
        Route::get('/users', function () {
            // /admin/users, имя: admin.users, middleware: auth
        })->name('users');
    });

// Controller namespace (устарело в Laravel 8+)
// Теперь используем полные имена классов
Route::get('/users', [UserController::class, 'index']);

// Subdomain routing
Route::domain('{account}.example.com')->group(function () {
    Route::get('/user/{id}', function ($account, $id) {
        //
    });
});
```

### Route Model Binding

```php
// Автоматический binding по ID
Route::get('/users/{user}', function (User $user) {
    return $user->email;
});
// /users/1 -> найдет User::find(1)

// Custom key (не id, а например slug)
Route::get('/posts/{post:slug}', function (Post $post) {
    return $post->title;
});
// /posts/my-first-post -> Post::where('slug', 'my-first-post')->firstOrFail()

// В модели (глобально для всех роутов)
public function getRouteKeyName()
{
    return 'slug';
}

// Scoped bindings (вложенные модели)
Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
    return $post;
});
// Проверит что post принадлежит user
// SELECT * FROM posts WHERE id = ? AND user_id = ?

// Явное указание scope
Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    return $post;
})->scopeBindings();

// Кастомный binding (в RouteServiceProvider)
public function boot()
{
    Route::bind('user', function ($value) {
        return User::where('name', $value)->firstOrFail();
    });
}

// Или через model()
Route::model('user', User::class);
```

### Fallback маршруты

```php
// 404 fallback
Route::fallback(function () {
    return 'Page not found';
});

// ВАЖНО: должен быть последним в списке маршрутов
```

### Rate Limiting

```php
// app/Providers/RouteServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

protected function configureRateLimiting()
{
    // Глобальный лимит
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });
    
    // Для конкретной группы
    RateLimiter::for('uploads', function (Request $request) {
        return $request->user()->isPremium()
            ? Limit::none()
            : Limit::perMinute(10);
    });
}

// Применение в маршрутах
Route::middleware(['throttle:api'])->group(function () {
    Route::get('/user', function () {
        //
    });
});

// С кастомным лимитером
Route::middleware(['throttle:uploads'])->group(function () {
    //
});

// Настройка прямо в маршруте
Route::get('/user', function () {
    //
})->middleware('throttle:60,1'); // 60 запросов в минуту
```

---

## Middleware (Посредники)

### Создание Middleware

```bash
php artisan make:middleware EnsureTokenIsValid
```

**app/Http/Middleware/EnsureTokenIsValid.php:**

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class EnsureTokenIsValid
{
    /**
     * Handle an incoming request.
     */
    public function handle(Request $request, Closure $next)
    {
        // Действия ДО передачи запроса дальше
        if ($request->input('token') !== 'my-secret-token') {
            return redirect('home');
        }

        $response = $next($request);

        // Действия ПОСЛЕ обработки запроса
        // Модифицировать $response если нужно

        return $response;
    }
}
```

### Регистрация Middleware

**app/Http/Kernel.php:**

```php
// Глобальные middleware (применяются ко всем запросам)
protected $middleware = [
    \App\Http\Middleware\TrustProxies::class,
    \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
    // ...
];

// Группы middleware
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
        'throttle:api',
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ],
];

// Именованные middleware (для отдельных маршрутов)
protected $middlewareAliases = [
    'auth' => \App\Http\Middleware\Authenticate::class,
    'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
    'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
    'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
    // Ваши middleware
    'token.valid' => \App\Http\Middleware\EnsureTokenIsValid::class,
];
```

### Применение Middleware

```php
// К отдельному маршруту
Route::get('/profile', function () {
    //
})->middleware('auth');

// Несколько middleware
Route::get('/profile', function () {
    //
})->middleware(['auth', 'verified']);

// К контроллеру
Route::get('/profile', [UserController::class, 'show'])->middleware('auth');

// Ко всем маршрутам контроллера (в конструкторе контроллера)
class UserController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
        
        // Только для определенных методов
        $this->middleware('log')->only('index');
        
        // Кроме определенных методов
        $this->middleware('auth')->except(['index', 'show']);
    }
}

// К группе маршрутов
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', function () {
        //
    });
});

// Middleware с параметрами
Route::get('/post/{id}', function ($id) {
    //
})->middleware('role:editor');

// В middleware
public function handle(Request $request, Closure $next, $role)
{
    if (! $request->user()->hasRole($role)) {
        abort(403);
    }

    return $next($request);
}
```

### Terminable Middleware

```php
// Выполнится ПОСЛЕ отправки ответа клиенту
namespace App\Http\Middleware;

use Closure;

class TerminatingMiddleware
{
    public function handle($request, Closure $next)
    {
        return $next($request);
    }

    public function terminate($request, $response)
    {
        // Логирование, очистка, отправка метрик и т.д.
        // НЕ блокирует ответ клиенту
    }
}
```

### Встроенные Middleware

```php
// auth - аутентификация
Route::middleware('auth')->group(...);

// guest - только для неаутентифицированных
Route::middleware('guest')->group(...);

// verified - email подтвержден
Route::middleware('verified')->group(...);

// throttle - rate limiting
Route::middleware('throttle:60,1')->group(...); // 60 req/min

// can - авторизация (policies)
Route::middleware('can:update,post')->group(...);

// signed - подписанные URL
Route::middleware('signed')->group(...);

// password.confirm - подтверждение пароля
Route::middleware('password.confirm')->group(...);
```

---

## Controllers (Контроллеры)

### Создание Controller

```bash
# Обычный контроллер
php artisan make:controller UserController

# Resource контроллер (CRUD методы)
php artisan make:controller UserController --resource

# API resource контроллер (без create/edit)
php artisan make:controller UserController --api

# С моделью
php artisan make:controller UserController --model=User --resource

# Invokable контроллер (один метод __invoke)
php artisan make:controller ShowProfile --invokable
```

### Базовый контроллер

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function index()
    {
        $users = User::all();
        return view('users.index', compact('users'));
    }

    public function show($id)
    {
        $user = User::findOrFail($id);
        return view('users.show', compact('user'));
    }

    public function create()
    {
        return view('users.create');
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|max:255',
            'email' => 'required|email|unique:users',
        ]);

        $user = User::create($validated);

        return redirect()->route('users.show', $user)
            ->with('success', 'User created successfully');
    }

    public function edit($id)
    {
        $user = User::findOrFail($id);
        return view('users.edit', compact('user'));
    }

    public function update(Request $request, $id)
    {
        $user = User::findOrFail($id);

        $validated = $request->validate([
            'name' => 'required|max:255',
            'email' => 'required|email|unique:users,email,' . $user->id,
        ]);

        $user->update($validated);

        return redirect()->route('users.show', $user)
            ->with('success', 'User updated successfully');
    }

    public function destroy($id)
    {
        $user = User::findOrFail($id);
        $user->delete();

        return redirect()->route('users.index')
            ->with('success', 'User deleted successfully');
    }
}
```

### Resource Routes

```php
// Один маршрут создает все CRUD endpoints
Route::resource('users', UserController::class);

/*
Создаст маршруты:
GET     /users              index   users.index
GET     /users/create       create  users.create
POST    /users              store   users.store
GET     /users/{user}       show    users.show
GET     /users/{user}/edit  edit    users.edit
PUT     /users/{user}       update  users.update
DELETE  /users/{user}       destroy users.destroy
*/

// Частичный resource
Route::resource('users', UserController::class)->only(['index', 'show']);
Route::resource('users', UserController::class)->except(['destroy']);

// API resource (без create и edit)
Route::apiResource('users', UserController::class);

// Несколько resources
Route::resources([
    'posts' => PostController::class,
    'comments' => CommentController::class,
]);

// Вложенные resources
Route::resource('posts.comments', CommentController::class);
/*
GET     /posts/{post}/comments              index
GET     /posts/{post}/comments/create       create
POST    /posts/{post}/comments              store
GET     /posts/{post}/comments/{comment}    show
...
*/

// Shallow nesting (без дублирования {post} где не нужно)
Route::resource('posts.comments', CommentController::class)->shallow();
/*
GET     /posts/{post}/comments              index
POST    /posts/{post}/comments              store
GET     /comments/{comment}                 show
PUT     /comments/{comment}                 update
DELETE  /comments/{comment}                 destroy
*/
```

### Route Model Binding в контроллере

```php
// Автоматический inject модели
public function show(User $user)
{
    // Laravel автоматически находит User::findOrFail($id)
    return view('users.show', compact('user'));
}

public function update(Request $request, User $user)
{
    $user->update($request->validated());
    return redirect()->route('users.show', $user);
}

// Scoped binding (с проверкой владения)
public function update(Request $request, Post $post, Comment $comment)
{
    // Проверяет что comment принадлежит post
    $comment->update($request->validated());
    return redirect()->route('posts.show', $post);
}
```

### Single Action Controller (Invokable)

```php
<?php

namespace App\Http\Controllers;

class ShowProfile extends Controller
{
    public function __invoke($id)
    {
        $user = User::findOrFail($id);
        return view('profile', compact('user'));
    }
}

// В маршрутах
Route::get('/profile/{id}', ShowProfile::class);
```

### Dependency Injection в контроллере

```php
use App\Services\UserService;
use Illuminate\Http\Request;

class UserController extends Controller
{
    protected $userService;

    // Constructor injection
    public function __construct(UserService $userService)
    {
        $this->userService = $userService;
    }

    // Method injection
    public function store(Request $request, UserService $userService)
    {
        $user = $userService->createUser($request->validated());
        return redirect()->route('users.show', $user);
    }
}
```

### Типы ответов контроллера

```php
class UserController extends Controller
{
    // View
    public function index()
    {
        return view('users.index', ['users' => User::all()]);
    }

    // JSON
    public function apiIndex()
    {
        return User::all(); // Автоматическое преобразование в JSON
        // или явно
        return response()->json(User::all());
    }

    // Redirect
    public function store(Request $request)
    {
        $user = User::create($request->validated());
        
        return redirect('/users');
        return redirect()->route('users.show', $user);
        return redirect()->back();
        return redirect()->back()->withInput();
        return redirect()->route('users.index')->with('success', 'Created!');
    }

    // Download
    public function download()
    {
        return response()->download(storage_path('app/file.pdf'));
        return response()->download($path, 'custom-name.pdf');
    }

    // File (показать в браузере)
    public function show()
    {
        return response()->file(storage_path('app/file.pdf'));
    }

    // Stream
    public function stream()
    {
        return response()->streamDownload(function () {
            echo 'CSV content...';
        }, 'export.csv');
    }

    // Custom response
    public function custom()
    {
        return response('Hello', 200)
            ->header('Content-Type', 'text/plain')
            ->cookie('name', 'value', 60);
    }
}
```

---

## Request (Запросы)

### Получение данных из запроса

```php
use Illuminate\Http\Request;

Route::post('/users', function (Request $request) {
    // Все данные
    $all = $request->all();
    
    // Конкретное поле
    $name = $request->input('name');
    $name = $request->name; // Альтернатива
    
    // С значением по умолчанию
    $name = $request->input('name', 'Guest');
    
    // Вложенные данные
    $email = $request->input('user.email');
    
    // Массивы
    $names = $request->input('users.*.name');
    
    // Только определенные поля
    $data = $request->only(['name', 'email']);
    $data = $request->only('name', 'email');
    
    // Все кроме определенных
    $data = $request->except(['password', 'password_confirmation']);
    
    // Query string (?search=term)
    $search = $request->query('search');
    $allQuery = $request->query(); // Все параметры
    
    // Проверка наличия
    if ($request->has('name')) {
        // Поле присутствует (даже если пустое)
    }
    
    if ($request->filled('name')) {
        // Поле присутствует И не пустое
    }
    
    if ($request->missing('name')) {
        // Поля нет
    }
    
    // Boolean значения
    $archived = $request->boolean('archived');
    // '1', 'true', 'on', 'yes' -> true
    // остальное -> false
});
```

### Old Input (старые значения после редиректа)

```php
// Сохранить input для следующего запроса
return redirect()->back()->withInput();
return redirect()->back()->withInput($request->except('password'));

// В Blade получить старое значение
<input type="text" name="name" value="{{ old('name') }}">

// В контроллере
$oldName = $request->old('name');
```

### Cookies

```php
// Получить cookie
$value = $request->cookie('name');

// Установить cookie в ответе
return response('Hello')
    ->cookie('name', 'value', $minutes);

return response('Hello')
    ->cookie('name', 'value', $minutes, $path, $domain, $secure, $httpOnly);

// Удалить cookie
return response('Goodbye')
    ->withoutCookie('name');
```

### Files (Файлы)

```php
// Получить загруженный файл
$file = $request->file('avatar');

// Проверка наличия
if ($request->hasFile('avatar')) {
    //
}

// Валидация
if ($request->file('avatar')->isValid()) {
    //
}

// Информация о файле
$path = $file->path();
$extension = $file->extension();
$size = $file->getSize();
$mimeType = $file->getMimeType();
$originalName = $file->getClientOriginalName();

// Сохранение файла
$path = $request->avatar->store('avatars');
$path = $request->avatar->store('avatars', 's3'); // На S3
$path = $request->avatar->storeAs('avatars', 'filename.jpg');

// Или через Storage facade
use Illuminate\Support\Facades\Storage;

$path = Storage::putFile('avatars', $request->file('avatar'));
$path = Storage::putFileAs('avatars', $request->file('avatar'), 'filename.jpg');
```

### Request методы

```php
// HTTP метод
$method = $request->method();

if ($request->isMethod('post')) {
    //
}

// URL информация
$url = $request->url(); // Без query string
$fullUrl = $request->fullUrl(); // С query string
$path = $request->path(); // users/1

// Проверка пути
if ($request->is('admin/*')) {
    //
}

if ($request->routeIs('admin.*')) {
    //
}

// IP адрес
$ip = $request->ip();

// Headers
$value = $request->header('X-Header-Name');
$value = $request->header('X-Header-Name', 'default');

if ($request->hasHeader('X-Header-Name')) {
    //
}

// Bearer token
$token = $request->bearerToken();

// Content negotiation
if ($request->expectsJson()) {
    return response()->json([...]);
}

if ($request->accepts(['text/html', 'application/json'])) {
    //
}
```

---

## Form Requests (Валидация)

### Создание Form Request

```bash
php artisan make:request StoreUserRequest
```

**app/Http/Requests/StoreUserRequest.php:**

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreUserRequest extends FormRequest
{
    /**
     * Авторизация запроса
     */
    public function authorize(): bool
    {
        // Можно проверить права доступа
        return $this->user()->can('create', User::class);
        
        // Или просто разрешить всем
        return true;
    }

    /**
     * Правила валидации
     */
    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|string|min:8|confirmed',
            'age' => 'required|integer|min:18|max:120',
            'role' => 'required|in:user,admin,editor',
        ];
    }

    /**
     * Кастомные сообщения об ошибках
     */
    public function messages(): array
    {
        return [
            'name.required' => 'Имя обязательно для заполнения',
            'email.unique' => 'Пользователь с таким email уже существует',
            'password.min' => 'Пароль должен быть минимум 8 символов',
        ];
    }

    /**
     * Кастомные названия атрибутов
     */
    public function attributes(): array
    {
        return [
            'email' => 'электронная почта',
            'name' => 'имя пользователя',
        ];
    }

    /**
     * Подготовка данных перед валидацией
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => Str::slug($this->name),
        ]);
    }

    /**
     * Действия после валидации
     */
    protected function passedValidation(): void
    {
        // Логика после успешной валидации
    }
}
```

### Использование Form Request

```php
use App\Http\Requests\StoreUserRequest;

class UserController extends Controller
{
    public function store(StoreUserRequest $request)
    {
        // Данные уже провалидированы!
        $validated = $request->validated();
        
        $user = User::create($validated);
        
        return redirect()->route('users.show', $user);
    }
}
```

### Обработка ошибок валидации

```php
// В Blade шаблоне
@if ($errors->any())
    <div class="alert alert-danger">
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif

@error('email')
    <div class="alert-danger">{{ $message }}</div>
@enderror

<input type="text" name="email" value="{{ old('email') }}">

// В контроллере (manual validation)
$validated = $request->validate([
    'name' => 'required|max:255',
    'email' => 'required|email',
]);

// С кастомными сообщениями
$validated = $request->validate(
    ['email' => 'required|email'],
    ['email.required' => 'Email обязателен']
);

// Или через Validator facade
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'name' => 'required',
    'email' => 'required|email',
]);

if ($validator->fails()) {
    return redirect()->back()
        ->withErrors($validator)
        ->withInput();
}

$validated = $validator->validated();
```

---

## Response (Ответы)

### Типы ответов

```php
// Простая строка
return 'Hello World';

// HTML View
return view('welcome', ['name' => 'John']);

// JSON
return response()->json([
    'name' => 'John',
    'age' => 30,
]);

// JSON с кодом
return response()->json(['error' => 'Not found'], 404);

// JSONP
return response()->jsonp('callback', ['name' => 'John']);

// Redirect
return redirect('/dashboard');
return redirect()->route('profile');
return redirect()->back();
return redirect()->away('https://example.com'); // Внешний URL

// Download
return response()->download($pathToFile);
return response()->download($pathToFile, $name, $headers);

// File
return response()->file($pathToFile);

// Custom response
return response($content)
    ->header('Content-Type', 'text/plain')
    ->header('X-Custom-Header', 'Value');

// No content (204)
return response()->noContent();

// Created (201)
return response()->json($user, 201);
```

### Response Macros

```php
// В AppServiceProvider
use Illuminate\Support\Facades\Response;

Response::macro('success', function ($data) {
    return Response::json([
        'success' => true,
        'data' => $data,
    ]);
});

// Использование
return response()->success(['name' => 'John']);
```

---

## Best Practices

1. **Используй Form Requests** для сложной валидации вместо валидации в контроллере
2. **Resource контроллеры** для CRUD операций
3. **Single Responsibility** - один контроллер = один ресурс
4. **Route Model Binding** вместо ручного `findOrFail`
5. **Middleware** для повторяющейся логики (auth, logging, CORS)
6. **Named routes** вместо хардкода URL
7. **API Resources** для трансформации данных в JSON
