# API Authentication - JWT, OAuth 2.0, Laravel Sanctum, Passport

Полное руководство по аутентификации API.

---

## Типы аутентификации API

### 1. Token-based (Токены)

```
Клиент отправляет токен в каждом запросе.

Authorization: Bearer <token>

✅ Stateless (без сессий)
✅ Подходит для SPA, мобильных приложений
```

### 2. Session-based (Сессии)

```
Используется Cookie с session ID.

Cookie: laravel_session=eyJpdiI6...

❌ Stateful (требует хранения на сервере)
✅ Подходит для традиционных веб-приложений
```

### 3. API Keys

```
Простой ключ для идентификации приложения.

X-API-Key: abc123def456

✅ Простота
❌ Низкая безопасность (один ключ для всех)
```

### 4. OAuth 2.0

```
Делегированная авторизация для third-party приложений.

Authorization: Bearer <access_token>

✅ Стандарт индустрии
✅ Разграничение прав (scopes)
❌ Сложная реализация
```

---

## JWT (JSON Web Token)

### Структура JWT

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

Header.Payload.Signature

Header:
{
  "alg": "HS256",    // Алгоритм подписи
  "typ": "JWT"       // Тип токена
}

Payload (Claims):
{
  "sub": "1234567890",           // Subject (user ID)
  "name": "John Doe",            // Custom claim
  "iat": 1516239022,             // Issued at (timestamp)
  "exp": 1516242622,             // Expiration (timestamp)
  "iss": "https://example.com",  // Issuer
  "aud": "https://api.example.com" // Audience
}

Signature:
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### Установка tymon/jwt-auth

```bash
composer require tymon/jwt-auth
php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
php artisan jwt:secret
```

**config/jwt.php:**

```php
return [
    'secret' => env('JWT_SECRET'),
    'ttl' => env('JWT_TTL', 60), // Время жизни в минутах
    'refresh_ttl' => env('JWT_REFRESH_TTL', 20160), // 2 недели
    'algo' => env('JWT_ALGO', 'HS256'),
];
```

### Модель User

```php
<?php

namespace App\Models;

use Tymon\JWTAuth\Contracts\JWTSubject;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable implements JWTSubject
{
    /**
     * Get the identifier that will be stored in the subject claim of the JWT.
     */
    public function getJWTIdentifier()
    {
        return $this->getKey();
    }

    /**
     * Return a key value array, containing any custom claims to be added to the JWT.
     */
    public function getJWTCustomClaims()
    {
        return [
            'email' => $this->email,
            'role' => $this->role,
        ];
    }
}
```

### Auth Controller

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Tymon\JWTAuth\Facades\JWTAuth;

class AuthController extends Controller
{
    /**
     * POST /api/auth/login
     */
    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        if (!$token = auth()->attempt($credentials)) {
            return response()->json([
                'error' => 'Invalid credentials'
            ], 401);
        }

        return $this->respondWithToken($token);
    }

    /**
     * POST /api/auth/register
     */
    public function register(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed',
        ]);

        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => bcrypt($validated['password']),
        ]);

        $token = auth()->login($user);

        return $this->respondWithToken($token);
    }

    /**
     * GET /api/auth/me
     */
    public function me()
    {
        return response()->json(auth()->user());
    }

    /**
     * POST /api/auth/logout
     */
    public function logout()
    {
        auth()->logout();

        return response()->json(['message' => 'Successfully logged out']);
    }

    /**
     * POST /api/auth/refresh
     */
    public function refresh()
    {
        return $this->respondWithToken(auth()->refresh());
    }

    protected function respondWithToken($token)
    {
        return response()->json([
            'access_token' => $token,
            'token_type' => 'bearer',
            'expires_in' => auth()->factory()->getTTL() * 60, // секунды
        ]);
    }
}
```

### Routes

```php
// routes/api.php
Route::prefix('auth')->group(function () {
    Route::post('login', [AuthController::class, 'login']);
    Route::post('register', [AuthController::class, 'register']);
    
    Route::middleware('auth:api')->group(function () {
        Route::get('me', [AuthController::class, 'me']);
        Route::post('logout', [AuthController::class, 'logout']);
        Route::post('refresh', [AuthController::class, 'refresh']);
    });
});

// Защищенные маршруты
Route::middleware('auth:api')->group(function () {
    Route::apiResource('posts', PostController::class);
});
```

### Использование в клиенте

```javascript
// Login
const response = await fetch('/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        email: 'user@example.com',
        password: 'password'
    })
});

const data = await response.json();
localStorage.setItem('access_token', data.access_token);

// Authenticated request
const postsResponse = await fetch('/api/posts', {
    headers: {
        'Authorization': `Bearer ${localStorage.getItem('access_token')}`
    }
});

// Refresh token
const refreshResponse = await fetch('/api/auth/refresh', {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${localStorage.getItem('access_token')}`
    }
});
```

---

## Laravel Sanctum (рекомендуется для SPA)

### Установка

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

### Конфигурация

**config/sanctum.php:**

```php
return [
    'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
        '%s%s',
        'localhost,localhost:3000,127.0.0.1,127.0.0.1:8000,::1',
        env('APP_URL') ? ','.parse_url(env('APP_URL'), PHP_URL_HOST) : ''
    ))),

    'guard' => ['web'],

    'expiration' => null, // Токены не истекают (можно установить в минутах)

    'middleware' => [
        'verify_csrf_token' => App\Http\Middleware\VerifyCsrfToken::class,
        'encrypt_cookies' => App\Http\Middleware\EncryptCookies::class,
    ],
];
```

**app/Http/Kernel.php:**

```php
'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    'throttle:api',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```

### Модель User

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

### API Token Authentication (для мобильных приложений)

```php
class AuthController extends Controller
{
    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            return response()->json([
                'message' => 'Invalid credentials'
            ], 401);
        }

        // Создать токен
        $token = $user->createToken('api-token')->plainTextToken;

        return response()->json([
            'access_token' => $token,
            'token_type' => 'Bearer',
        ]);
    }

    public function logout(Request $request)
    {
        // Удалить текущий токен
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logged out']);
    }

    public function logoutAll(Request $request)
    {
        // Удалить все токены пользователя
        $request->user()->tokens()->delete();

        return response()->json(['message' => 'Logged out from all devices']);
    }
}
```

### Abilities (Права доступа)

```php
// Создать токен с правами
$token = $user->createToken('api-token', ['posts:read', 'posts:create'])->plainTextToken;

// Проверка прав
Route::middleware(['auth:sanctum', 'ability:posts:create'])->group(function () {
    Route::post('/posts', [PostController::class, 'store']);
});

// Или в контроллере
if ($request->user()->tokenCan('posts:create')) {
    // Разрешено
}

// Несколько прав
Route::middleware(['auth:sanctum', 'abilities:posts:read,posts:create'])->group(function () {
    //
});
```

### SPA Authentication (Cookie-based)

```javascript
// 1. Получить CSRF cookie
await fetch('/sanctum/csrf-cookie', {
    credentials: 'include'
});

// 2. Login
await fetch('/login', {
    method: 'POST',
    credentials: 'include',
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
    },
    body: JSON.stringify({
        email: 'user@example.com',
        password: 'password'
    })
});

// 3. Authenticated requests (Cookie автоматически отправляется)
await fetch('/api/user', {
    credentials: 'include'
});
```

---

## Laravel Passport (OAuth 2.0)

### Установка

```bash
composer require laravel/passport
php artisan migrate
php artisan passport:install
```

### Конфигурация

**app/Models/User.php:**

```php
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

**config/auth.php:**

```php
'guards' => [
    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],
```

**app/Providers/AuthServiceProvider.php:**

```php
use Laravel\Passport\Passport;

public function boot()
{
    Passport::tokensExpireIn(now()->addDays(15));
    Passport::refreshTokensExpireIn(now()->addDays(30));
    Passport::personalAccessTokensExpireIn(now()->addMonths(6));
    
    // Scopes
    Passport::tokensCan([
        'posts-read' => 'Read posts',
        'posts-create' => 'Create posts',
        'posts-update' => 'Update posts',
        'posts-delete' => 'Delete posts',
    ]);
    
    Passport::setDefaultScope([
        'posts-read',
    ]);
}
```

### OAuth 2.0 Flows

#### 1. Password Grant (для первых лиц)

```php
// POST /oauth/token
{
    "grant_type": "password",
    "client_id": "client-id",
    "client_secret": "client-secret",
    "username": "user@example.com",
    "password": "password",
    "scope": "posts-read posts-create"
}

// Response
{
    "token_type": "Bearer",
    "expires_in": 31536000,
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "refresh_token": "def50200e95..."
}
```

#### 2. Authorization Code Grant (для third-party)

```php
// 1. Redirect пользователя на авторизацию
Route::get('/redirect', function () {
    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => 'posts-read',
    ]);

    return redirect('http://passport-app.test/oauth/authorize?' . $query);
});

// 2. После подтверждения - callback с code
Route::get('/callback', function (Request $request) {
    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'authorization_code',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'code' => $request->code,
    ]);

    return $response->json();
});
```

#### 3. Client Credentials Grant (server-to-server)

```php
// POST /oauth/token
{
    "grant_type": "client_credentials",
    "client_id": "client-id",
    "client_secret": "client-secret",
    "scope": "posts-read"
}
```

#### 4. Refresh Token

```php
// POST /oauth/token
{
    "grant_type": "refresh_token",
    "refresh_token": "def50200e95...",
    "client_id": "client-id",
    "client_secret": "client-secret",
    "scope": "posts-read"
}
```

### Защита маршрутов

```php
Route::middleware('auth:api')->group(function () {
    Route::get('/user', function (Request $request) {
        return $request->user();
    });
});

// С проверкой scope
Route::middleware(['auth:api', 'scope:posts-create'])->group(function () {
    Route::post('/posts', [PostController::class, 'store']);
});

// Любой из scopes
Route::middleware(['auth:api', 'scopes:posts-read,posts-create'])->group(function () {
    //
});
```

### Personal Access Tokens

```php
// Создать персональный токен
$token = $user->createToken('Token Name', ['posts-read'])->accessToken;

// Revoke токен
$user->tokens->each(function ($token) {
    $token->revoke();
});
```

---

## OAuth 2.0 Детально

### Grant Types

#### 1. Authorization Code

```
Самый безопасный flow для веб-приложений.

User -> Client App -> Authorization Server (авторизация)
     <- Redirect с code
Client App -> Authorization Server (обмен code на token)
          <- Access Token + Refresh Token

✅ Для веб-приложений с backend
✅ Refresh tokens
```

#### 2. Implicit (устарел)

```
Упрощенный flow без client secret.

User -> Client App -> Authorization Server
     <- Redirect с Access Token в URL fragment

❌ Устарел (небезопасен)
✅ Заменен на Authorization Code + PKCE
```

#### 3. Resource Owner Password Credentials

```
Клиент напрямую получает username/password.

Client App -> Authorization Server (username + password)
           <- Access Token

✅ Только для first-party приложений
❌ Не для third-party
```

#### 4. Client Credentials

```
Server-to-server аутентификация.

Service A -> Authorization Server (client credentials)
          <- Access Token

✅ Для микросервисов, API-to-API
❌ Нет пользовательского контекста
```

#### 5. Refresh Token

```
Обновление access token без повторной аутентификации.

Client App -> Authorization Server (refresh token)
           <- New Access Token + Refresh Token

✅ Продление сессии без ввода пароля
```

### PKCE (Proof Key for Code Exchange)

```php
// Для мобильных/SPA приложений без client secret

// 1. Генерация code_verifier и code_challenge
$codeVerifier = Str::random(128);
$codeChallenge = base64_encode(hash('sha256', $codeVerifier, true));

// 2. Authorization request
$query = http_build_query([
    'client_id' => 'client-id',
    'redirect_uri' => 'myapp://callback',
    'response_type' => 'code',
    'scope' => 'posts-read',
    'code_challenge' => $codeChallenge,
    'code_challenge_method' => 'S256',
]);

// 3. Token request
Http::asForm()->post('/oauth/token', [
    'grant_type' => 'authorization_code',
    'client_id' => 'client-id',
    'redirect_uri' => 'myapp://callback',
    'code' => $code,
    'code_verifier' => $codeVerifier,
]);
```

---

## Сравнение: Sanctum vs Passport vs JWT

### Laravel Sanctum

```
✅ Простая настройка
✅ Идеально для SPA (Cookie-based)
✅ API tokens для мобильных
✅ Легкий вес
❌ Нет OAuth 2.0
❌ Нет refresh tokens из коробки

Когда использовать:
- SPA + Laravel backend (Cookie)
- Мобильное приложение (API tokens)
- Простая аутентификация без third-party
```

### Laravel Passport

```
✅ Полная реализация OAuth 2.0
✅ Все grant types
✅ Scopes (разграничение прав)
✅ Third-party приложения
❌ Сложная настройка
❌ Больше overhead

Когда использовать:
- OAuth 2.0 сервер
- Third-party приложения
- Сложные требования к безопасности
```

### JWT (tymon/jwt-auth)

```
✅ Stateless
✅ Легкий вес
✅ Работает везде
❌ Сложнее отозвать токен
❌ Не стандарт Laravel

Когда использовать:
- Микросервисы
- Нужен stateless
- Cross-domain аутентификация
```

---

## Безопасность

### 1. HTTPS обязательно

```php
// В production всегда HTTPS
if (!$request->secure() && app()->environment('production')) {
    return redirect()->secure($request->getRequestUri());
}
```

### 2. Token Storage

```javascript
// ✅ ХОРОШО - HttpOnly Cookie (для SPA)
// Не доступен JavaScript, защита от XSS

// ⚠️ ОСТОРОЖНО - localStorage/sessionStorage
// Доступен JavaScript, уязвим к XSS
localStorage.setItem('token', token);

// ❌ ПЛОХО - Cookie без HttpOnly
document.cookie = `token=${token}`;
```

### 3. Rate Limiting

```php
Route::middleware('throttle:5,1')->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
});
// 5 попыток в минуту
```

### 4. Token Expiration

```php
// Короткий TTL для access token
'ttl' => 15, // 15 минут

// Длинный TTL для refresh token
'refresh_ttl' => 20160, // 2 недели
```

### 5. CSRF Protection (для SPA)

```php
// CSRF токен в cookie
Route::get('/sanctum/csrf-cookie', function () {
    return response()->json(['message' => 'CSRF cookie set']);
});

// В requests включать CSRF token
fetch('/api/posts', {
    headers: {
        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content
    }
});
```

### 6. Token Blacklist

```php
// При logout добавить токен в blacklist
public function logout(Request $request)
{
    $token = $request->user()->token();
    $token->revoke();
    
    // Или для JWT
    JWTAuth::invalidate(JWTAuth::getToken());
}
```

### 7. IP Whitelist

```php
public function login(Request $request)
{
    $allowedIPs = ['192.168.1.1', '10.0.0.1'];
    
    if (!in_array($request->ip(), $allowedIPs)) {
        return response()->json(['error' => 'Forbidden'], 403);
    }
}
```

---

## Best Practices

### 1. Используй Sanctum для SPA

```php
// Простота + безопасность
return Sanctum::actingAs($user)->get('/api/user');
```

### 2. Используй Passport для OAuth 2.0

```php
// Если нужны third-party приложения
Passport::tokensCan([...]);
```

### 3. Короткий TTL для access tokens

```php
'ttl' => 15, // 15 минут
'refresh_ttl' => 20160, // 2 недели
```

### 4. Rate Limiting на login

```php
Route::middleware('throttle:5,1')->post('/login', ...);
```

### 5. Логирование аутентификации

```php
public function login(Request $request)
{
    Log::info('Login attempt', [
        'email' => $request->email,
        'ip' => $request->ip(),
        'user_agent' => $request->userAgent(),
    ]);
}
```

### 6. Два фактора аутентификации

```bash
composer require pragmarx/google2fa-laravel
```

### 7. Revoke старых токенов

```php
// При login удалить старые токены
$user->tokens()->delete();
$token = $user->createToken('api-token')->plainTextToken;
```

### 8. HTTPS в production

```php
if (app()->environment('production')) {
    URL::forceScheme('https');
}
```
