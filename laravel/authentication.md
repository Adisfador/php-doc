# Laravel Authentication & Authorization

Полный разбор аутентификации и авторизации: Session, API tokens, Sanctum, JWT, Gates, Policies.

---

## 🎯 Authentication vs Authorization

**Authentication (Аутентификация)** - "Кто ты?"  
**Authorization (Авторизация)** - "Что тебе разрешено делать?"

```php
// Authentication
if (auth()->check()) {
    // Пользователь залогинен
}

// Authorization
if (auth()->user()->can('edit', $post)) {
    // Пользователь может редактировать пост
}
```

---

## 🔐 Session Authentication

### Настройка

**config/auth.php:**
```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],
],

'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model' => App\Models\User::class,
    ],
],
```

### User модель

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;
    
    protected $fillable = [
        'name',
        'email',
        'password',
    ];
    
    protected $hidden = [
        'password',
        'remember_token',
    ];
    
    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed', // Laravel 10+
    ];
}
```

### Регистрация

```php
use App\Models\User;
use Illuminate\Support\Facades\Hash;

public function register(Request $request)
{
    $validated = $request->validate([
        'name' => 'required|string|max:255',
        'email' => 'required|string|email|max:255|unique:users',
        'password' => 'required|string|min:8|confirmed',
    ]);
    
    $user = User::create([
        'name' => $validated['name'],
        'email' => $validated['email'],
        'password' => Hash::make($validated['password']),
    ]);
    
    // Автоматический логин после регистрации
    auth()->login($user);
    
    // Или
    Auth::login($user);
    
    return redirect('/dashboard');
}
```

### Вход (Login)

```php
use Illuminate\Support\Facades\Auth;

public function login(Request $request)
{
    $credentials = $request->validate([
        'email' => 'required|email',
        'password' => 'required',
    ]);
    
    // Попытка входа
    if (Auth::attempt($credentials)) {
        $request->session()->regenerate(); // Защита от session fixation
        
        return redirect()->intended('/dashboard');
    }
    
    return back()->withErrors([
        'email' => 'The provided credentials do not match our records.',
    ])->onlyInput('email');
}

// С "Remember Me"
Auth::attempt($credentials, $remember = true);

// С дополнительными условиями
Auth::attempt([
    'email' => $request->email,
    'password' => $request->password,
    'active' => 1,
]);
```

### Выход (Logout)

```php
public function logout(Request $request)
{
    Auth::logout();
    
    $request->session()->invalidate();
    $request->session()->regenerateToken();
    
    return redirect('/');
}
```

### Проверка аутентификации

```php
// Проверить залогинен ли
if (Auth::check()) {
    // Пользователь авторизован
}

// Получить текущего пользователя
$user = Auth::user();
$user = auth()->user();
$user = $request->user();

// ID пользователя
$id = Auth::id();

// Проверка гостя
if (Auth::guest()) {
    // Пользователь НЕ авторизован
}
```

### Вход за пользователя (без пароля)

```php
Auth::login($user);

// Один раз, без remember
Auth::once($credentials);

// Login by ID
Auth::loginUsingId(1);
Auth::loginUsingId(1, $remember = true);
```

---

## 🛡️ Guards (Охранники)

**Guard** определяет как пользователи аутентифицируются для каждого запроса.

### Настройка нескольких Guards

```php
// config/auth.php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],
    
    'admin' => [
        'driver' => 'session',
        'provider' => 'admins',
    ],
    
    'api' => [
        'driver' => 'sanctum',
        'provider' => 'users',
    ],
],

'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model' => App\Models\User::class,
    ],
    
    'admins' => [
        'driver' => 'eloquent',
        'model' => App\Models\Admin::class,
    ],
],
```

### Использование Guards

```php
// Web guard (по умолчанию)
Auth::attempt($credentials);

// Admin guard
Auth::guard('admin')->attempt($credentials);

if (Auth::guard('admin')->check()) {
    $admin = Auth::guard('admin')->user();
}

// Logout из admin guard
Auth::guard('admin')->logout();
```

---

## 🚪 Middleware

### auth

```php
// routes/web.php
Route::middleware('auth')->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});

// Или в контроллере
public function __construct()
{
    $this->middleware('auth');
    
    // Только для определенных методов
    $this->middleware('auth')->only(['edit', 'update']);
    $this->middleware('auth')->except(['index', 'show']);
}

// С guard
Route::middleware('auth:admin')->group(function () {
    Route::get('/admin', [AdminController::class, 'index']);
});
```

### guest

```php
// Только для гостей (не залогиненных)
Route::middleware('guest')->group(function () {
    Route::get('/login', [LoginController::class, 'show']);
    Route::get('/register', [RegisterController::class, 'show']);
});
```

### verified

```php
// Требует подтвержденный email
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});
```

### password.confirm

```php
// Требует подтверждения пароля перед доступом
Route::middleware(['auth', 'password.confirm'])->group(function () {
    Route::get('/settings/security', [SettingsController::class, 'security']);
});
```

---

## 🎫 API Token Authentication (Sanctum)

### Установка

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

### Настройка

**app/Http/Kernel.php:**
```php
'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    'throttle:api',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```

**config/sanctum.php:**
```php
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
    '%s%s',
    'localhost,localhost:3000,127.0.0.1,127.0.0.1:8000,::1',
    env('APP_URL') ? ','.parse_url(env('APP_URL'), PHP_URL_HOST) : ''
))),
```

### User модель

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;
}
```

### Выдача токенов

```php
use App\Models\User;
use Illuminate\Support\Facades\Hash;

public function login(Request $request)
{
    $request->validate([
        'email' => 'required|email',
        'password' => 'required',
    ]);
    
    $user = User::where('email', $request->email)->first();
    
    if (!$user || !Hash::check($request->password, $user->password)) {
        return response()->json(['message' => 'Invalid credentials'], 401);
    }
    
    // Создать токен
    $token = $user->createToken('mobile-app')->plainTextToken;
    
    return response()->json([
        'token' => $token,
        'user' => $user,
    ]);
}

// С abilities (scopes)
$token = $user->createToken('token-name', ['post:create', 'post:update'])->plainTextToken;

// Срок действия
$token = $user->createToken('token-name', ['*'], now()->addWeek())->plainTextToken;
```

### Использование токенов

```php
// В клиенте (frontend)
fetch('/api/user', {
    headers: {
        'Authorization': 'Bearer ' + token,
        'Accept': 'application/json',
    }
})
```

```php
// API routes
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', function (Request $request) {
        return $request->user();
    });
    
    Route::post('/posts', [PostController::class, 'store']);
});
```

### Проверка abilities

```php
Route::middleware('auth:sanctum')->post('/posts', function (Request $request) {
    // Проверить ability
    if (!$request->user()->tokenCan('post:create')) {
        return response()->json(['message' => 'Forbidden'], 403);
    }
    
    // ...
});

// Или через middleware
Route::middleware(['auth:sanctum', 'ability:post:create,post:update'])
    ->post('/posts', [PostController::class, 'store']);

// Хотя бы один из abilities
Route::middleware(['auth:sanctum', 'abilities:post:create,post:update'])
    ->post('/posts', [PostController::class, 'store']);
```

### Управление токенами

```php
// Получить все токены пользователя
$user->tokens;

// Получить текущий токен
$token = $request->user()->currentAccessToken();

// Удалить текущий токен
$request->user()->currentAccessToken()->delete();

// Удалить все токены
$user->tokens()->delete();

// Удалить конкретный токен
$user->tokens()->where('name', 'mobile-app')->delete();
```

### SPA Authentication

```php
// sanctum.php
'stateful' => ['localhost:3000', 'myapp.com'],

// CORS settings для SPA
// config/cors.php
'paths' => ['api/*', 'sanctum/csrf-cookie'],
'supports_credentials' => true,
```

**Frontend (Vue/React):**
```javascript
// 1. Получить CSRF cookie
await axios.get('/sanctum/csrf-cookie');

// 2. Login
await axios.post('/login', {
    email: 'test@example.com',
    password: 'password'
});

// 3. Делать authenticated запросы
const response = await axios.get('/api/user');
```

---

## 🔑 JWT Authentication

### Установка (tymon/jwt-auth)

```bash
composer require tymon/jwt-auth
php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
php artisan jwt:secret
```

### Настройка

**config/auth.php:**
```php
'guards' => [
    'api' => [
        'driver' => 'jwt',
        'provider' => 'users',
    ],
],
```

### User модель

```php
use Tymon\JWTAuth\Contracts\JWTSubject;

class User extends Authenticatable implements JWTSubject
{
    public function getJWTIdentifier()
    {
        return $this->getKey();
    }
    
    public function getJWTCustomClaims()
    {
        return [
            'role' => $this->role,
            'email' => $this->email,
        ];
    }
}
```

### Login & получение JWT

```php
use Tymon\JWTAuth\Facades\JWTAuth;

public function login(Request $request)
{
    $credentials = $request->only('email', 'password');
    
    if (!$token = auth('api')->attempt($credentials)) {
        return response()->json(['error' => 'Unauthorized'], 401);
    }
    
    return $this->respondWithToken($token);
}

protected function respondWithToken($token)
{
    return response()->json([
        'access_token' => $token,
        'token_type' => 'bearer',
        'expires_in' => auth('api')->factory()->getTTL() * 60,
    ]);
}
```

### Защита routes

```php
Route::middleware('auth:api')->group(function () {
    Route::get('/user', function (Request $request) {
        return $request->user();
    });
});
```

### Refresh token

```php
public function refresh()
{
    return $this->respondWithToken(auth('api')->refresh());
}
```

### Logout

```php
public function logout()
{
    auth('api')->logout();
    
    return response()->json(['message' => 'Successfully logged out']);
}
```

### Получение payload

```php
$user = auth('api')->user();

$payload = auth('api')->payload();
$payload->get('role');
```

---

## 🔓 Authorization (Авторизация)

### Gates

**app/Providers/AuthServiceProvider.php:**
```php
use Illuminate\Support\Facades\Gate;

public function boot()
{
    // Простой Gate
    Gate::define('update-post', function (User $user, Post $post) {
        return $user->id === $post->user_id;
    });
    
    // Gate без модели
    Gate::define('viewAdminPanel', function (User $user) {
        return $user->isAdmin();
    });
    
    // Gate с опциональным пользователем
    Gate::define('before', function (?User $user, string $ability) {
        if ($user && $user->isSuper Admin()) {
            return true; // Разрешить всё
        }
    });
    
    // Response с сообщением
    Gate::define('update-post', function (User $user, Post $post) {
        return $user->id === $post->user_id
            ? Response::allow()
            : Response::deny('You do not own this post.');
    });
}
```

### Проверка Gates

```php
use Illuminate\Support\Facades\Gate;

// Authorize (throw 403 если false)
Gate::authorize('update-post', $post);

// Проверка
if (Gate::allows('update-post', $post)) {
    // Разрешено
}

if (Gate::denies('update-post', $post)) {
    // Запрещено
}

// Любой из
if (Gate::any(['update-post', 'delete-post'], $post)) {
    // ...
}

// Все должны быть true
if (Gate::none(['update-post', 'delete-post'], $post)) {
    // ...
}

// Inspect response
$response = Gate::inspect('update-post', $post);

if ($response->allowed()) {
    // Разрешено
} else {
    echo $response->message();
}
```

### Проверка для другого пользователя

```php
if (Gate::forUser($user)->allows('update-post', $post)) {
    // ...
}
```

### В Blade

```blade
@can('update-post', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan

@cannot('delete-post', $post)
    <p>You cannot delete this post.</p>
@endcannot

@can('update', $post)
    <!-- ... -->
@elsecan('create', Post::class)
    <!-- ... -->
@endcan
```

---

## 📋 Policies

**Создание:**
```bash
php artisan make:policy PostPolicy
php artisan make:policy PostPolicy --model=Post
```

**app/Policies/PostPolicy.php:**
```php
<?php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    // Перед всеми проверками
    public function before(User $user, string $ability)
    {
        if ($user->isAdmin()) {
            return true; // Админ может всё
        }
    }
    
    // Viewне требует $post (для списка)
    public function viewAny(User $user)
    {
        return true;
    }
    
    // View конкретного поста
    public function view(?User $user, Post $post)
    {
        return $post->published || ($user && $user->id === $post->user_id);
    }
    
    // Create
    public function create(User $user)
    {
        return $user->email_verified_at !== null;
    }
    
    // Update
    public function update(User $user, Post $post)
    {
        return $user->id === $post->user_id;
    }
    
    // Delete
    public function delete(User $user, Post $post)
    {
        return $user->id === $post->user_id;
    }
    
    // Restore (soft deleted)
    public function restore(User $user, Post $post)
    {
        return $user->id === $post->user_id;
    }
    
    // Force Delete
    public function forceDelete(User $user, Post $post)
    {
        return $user->isAdmin();
    }
}
```

### Регистрация Policies

**app/Providers/AuthServiceProvider.php:**
```php
protected $policies = [
    Post::class => PostPolicy::class,
];

// Или auto-discovery (Laravel найдет сам)
public function boot()
{
    Gate::guessPolicyNamesUsing(function ($modelClass) {
        return 'App\\Policies\\' . class_basename($modelClass) . 'Policy';
    });
}
```

### Использование Policies

```php
// В контроллере
public function update(Request $request, Post $post)
{
    $this->authorize('update', $post);
    
    // Если не разрешено - 403 автоматом
    
    $post->update($request->all());
    
    return redirect()->route('posts.show', $post);
}

// Без exception
if ($request->user()->can('update', $post)) {
    // ...
}

if ($request->user()->cannot('delete', $post)) {
    abort(403);
}

// Для другого пользователя
if ($user->can('update', $post)) {
    // ...
}

// В модели через enum
use App\Models\Post;

if ($request->user()->can('update', Post::class)) {
    // Can create any post
}
```

### Middleware

```php
// routes/web.php
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->middleware('can:update,post');

// Для класса (create)
Route::post('/posts', [PostController::class, 'store'])
    ->middleware('can:create,App\Models\Post');
```

### В Blade

```blade
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan

@cannot('delete', $post)
    <p>You cannot delete this post.</p>
@endcannot
```

### Resource Controllers

```php
public function __construct()
{
    $this->authorizeResource(Post::class, 'post');
}

// Автоматически применит policies:
// index → viewAny
// show → view
// create → create
// store → create
// edit → update
// update → update
// destroy → delete
```

---

## 🔐 Password Confirmation

### Middleware

```php
Route::middleware(['auth', 'password.confirm'])->group(function () {
    Route::get('/settings/security', [SettingsController::class, 'security']);
});
```

### Подтверждение вручную

```php
use Illuminate\Support\Facades\Hash;

if (Hash::check($request->password, $request->user()->password)) {
    $request->session()->passwordConfirmed();
    
    // Password подтвержден на 3 часа
}
```

### В Blade

```blade
<form method="POST" action="/confirm-password">
    @csrf
    <input type="password" name="password">
    <button>Confirm</button>
</form>
```

---

## ✉️ Email Verification

### Миграция

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable(); // ← важно
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
```

### User модель

```php
use Illuminate\Contracts\Auth\MustVerifyEmail;

class User extends Authenticatable implements MustVerifyEmail
{
    // ...
}
```

### Routes

```php
// routes/web.php
use Illuminate\Foundation\Auth\EmailVerificationRequest;
use Illuminate\Http\Request;

// Страница напоминания
Route::get('/email/verify', function () {
    return view('auth.verify-email');
})->middleware('auth')->name('verification.notice');

// Обработка ссылки подтверждения
Route::get('/email/verify/{id}/{hash}', function (EmailVerificationRequest $request) {
    $request->fulfill();
    
    return redirect('/dashboard');
})->middleware(['auth', 'signed'])->name('verification.verify');

// Повторная отправка
Route::post('/email/verification-notification', function (Request $request) {
    $request->user()->sendEmailVerificationNotification();
    
    return back()->with('message', 'Verification link sent!');
})->middleware(['auth', 'throttle:6,1'])->name('verification.send');
```

### Защита routes

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});
```

### Кастомизация письма

**app/Notifications/VerifyEmail.php:**
```php
use Illuminate\Auth\Notifications\VerifyEmail as VerifyEmailNotification;
use Illuminate\Notifications\Messages\MailMessage;

class VerifyEmail extends VerifyEmailNotification
{
    protected function buildMailMessage($url)
    {
        return (new MailMessage)
            ->subject('Verify Email Address')
            ->line('Click the button below to verify your email address.')
            ->action('Verify Email Address', $url);
    }
}

// В User модели
public function sendEmailVerificationNotification()
{
    $this->notify(new \App\Notifications\VerifyEmail);
}
```

---

## 🔁 Password Reset

### Миграция

```php
Schema::create('password_reset_tokens', function (Blueprint $table) {
    $table->string('email')->primary();
    $table->string('token');
    $table->timestamp('created_at')->nullable();
});
```

### Routes

```php
use Illuminate\Auth\Events\PasswordReset;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Password;
use Illuminate\Support\Str;

// Форма запроса сброса
Route::get('/forgot-password', [PasswordResetController::class, 'request'])
    ->name('password.request');

// Отправка ссылки
Route::post('/forgot-password', function (Request $request) {
    $request->validate(['email' => 'required|email']);
    
    $status = Password::sendResetLink($request->only('email'));
    
    return $status === Password::RESET_LINK_SENT
        ? back()->with(['status' => __($status)])
        : back()->withErrors(['email' => __($status)]);
})->name('password.email');

// Форма сброса
Route::get('/reset-password/{token}', function ($token) {
    return view('auth.reset-password', ['token' => $token]);
})->name('password.reset');

// Обработка сброса
Route::post('/reset-password', function (Request $request) {
    $request->validate([
        'token' => 'required',
        'email' => 'required|email',
        'password' => 'required|min:8|confirmed',
    ]);
    
    $status = Password::reset(
        $request->only('email', 'password', 'password_confirmation', 'token'),
        function ($user, $password) {
            $user->forceFill([
                'password' => Hash::make($password)
            ])->setRememberToken(Str::random(60));
            
            $user->save();
            
            event(new PasswordReset($user));
        }
    );
    
    return $status === Password::PASSWORD_RESET
        ? redirect()->route('login')->with('status', __($status))
        : back()->withErrors(['email' => [__($status)]]);
})->name('password.update');
```

---

## 🌐 Social Authentication (Socialite)

### Установка

```bash
composer require laravel/socialite
```

### Настройка

```php
// config/services.php
'github' => [
    'client_id' => env('GITHUB_CLIENT_ID'),
    'client_secret' => env('GITHUB_CLIENT_SECRET'),
    'redirect' => env('GITHUB_REDIRECT_URL'),
],

'google' => [
    'client_id' => env('GOOGLE_CLIENT_ID'),
    'client_secret' => env('GOOGLE_CLIENT_SECRET'),
    'redirect' => env('GOOGLE_REDIRECT_URL'),
],
```

### Routes

```php
use Laravel\Socialite\Facades\Socialite;

Route::get('/auth/github/redirect', function () {
    return Socialite::driver('github')->redirect();
});

Route::get('/auth/github/callback', function () {
    $githubUser = Socialite::driver('github')->user();
    
    $user = User::updateOrCreate([
        'github_id' => $githubUser->id,
    ], [
        'name' => $githubUser->name,
        'email' => $githubUser->email,
        'github_token' => $githubUser->token,
        'github_refresh_token' => $githubUser->refreshToken,
    ]);
    
    Auth::login($user);
    
    return redirect('/dashboard');
});
```

### Получение данных пользователя

```php
$user = Socialite::driver('github')->user();

$user->token; // OAuth 2.0 access token
$user->refreshToken; // OAuth 2.0 refresh token
$user->expiresIn; // Секунд до истечения
$user->getId(); // ID пользователя у провайдера
$user->getNickname(); // Username
$user->getName(); // Полное имя
$user->getEmail(); // Email
$user->getAvatar(); // URL аватара
```

### Stateless (API)

```php
return Socialite::driver('github')->stateless()->user();
```

---

## 🎓 Best Practices

### 1. Всегда хешируй пароли

```php
// ✅ ХОРОШО
use Illuminate\Support\Facades\Hash;

$user->password = Hash::make($request->password);

// ❌ ПЛОХО
$user->password = bcrypt($request->password); // Устарело
```

### 2. Используй Route Model Binding + Policy

```php
// В контроллере
public function update(Request $request, Post $post)
{
    $this->authorize('update', $post);
    
    // ...
}
```

### 3. Regenerate session после логина

```php
$request->session()->regenerate(); // Защита от session fixation
```

### 4. Используй throttle для login

```php
Route::post('/login', [LoginController::class, 'login'])
    ->middleware('throttle:5,1'); // 5 попыток в минуту
```

### 5. Всегда проверяй abilities для Sanctum

```php
if (!$request->user()->tokenCan('post:create')) {
    return response()->json(['message' => 'Forbidden'], 403);
}
```

---

## 🎓 Для собеседования: ключевые точки

1. **Guards** - определяют КАК аутентифицировать (session, token, jwt)
2. **Providers** - определяют ГДЕ брать пользователей (eloquent, database)
3. **Gates vs Policies** - Gates для простых проверок, Policies для моделей
4. **Sanctum** - API tokens для SPA/mobile, abilities/scopes
5. **JWT vs Sanctum** - JWT stateless, Sanctum с БД, более легкий
6. **Auth::attempt()** - проверка credentials + логин
7. **$user->can()** - проверка через Policy
8. **Gate::allows()** - проверка через Gate
9. **middleware auth** - защита routes, можно указать guard
10. **email verification** - MustVerifyEmail interface, verified middleware

**Главное:** Authentication проверяет личность, Authorization проверяет права. Guards управляют КАК, Policies управляют ЧТО разрешено.
