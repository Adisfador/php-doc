# HTTP и HTTPS

Протокол передачи гипертекста - основа веб-коммуникации.

---

## 🌐 Что такое HTTP

**HTTP (HyperText Transfer Protocol)** - прикладной протокол для передачи данных в сети, основа World Wide Web.

### Основные характеристики

- **Stateless** - каждый запрос независим, сервер не хранит состояние клиента
- **Client-Server** - клиент инициирует запрос, сервер отвечает
- **Text-based** (HTTP/1.x) - человекочитаемый формат
- **Request-Response** - цикл запрос-ответ
- **Application Layer** - 7 уровень OSI модели

### HTTP Message Structure

**Request:**
```http
GET /api/users/123 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: application/json
Authorization: Bearer token123

```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 145
Cache-Control: max-age=3600

{"id":123,"name":"John Doe","email":"john@example.com"}
```

---

## 📋 HTTP Methods (Verbs)

### GET - Получение ресурса

**Характеристики:**
- **Safe** - не изменяет состояние сервера
- **Idempotent** - множественные вызовы = одинаковый результат
- **Cacheable** - можно кешировать

```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Accept: application/json
```

**PHP:**
```php
// Laravel
Route::get('/users/{id}', function ($id) {
    return User::findOrFail($id);
});

// Guzzle client
$response = $client->get('https://api.example.com/users/123');
$user = json_decode($response->getBody(), true);
```

### POST - Создание ресурса

**Характеристики:**
- **NOT Safe** - изменяет состояние
- **NOT Idempotent** - повторный вызов создаст дубликат
- **NOT Cacheable** (обычно)

```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"name":"John Doe","email":"john@example.com"}
```

**PHP:**
```php
// Laravel
Route::post('/users', function (Request $request) {
    $user = User::create($request->all());
    return response()->json($user, 201);
});

// Guzzle
$response = $client->post('https://api.example.com/users', [
    'json' => [
        'name' => 'John Doe',
        'email' => 'john@example.com',
    ]
]);
```

### PUT - Полное обновление ресурса

**Характеристики:**
- **NOT Safe**
- **Idempotent** - повторные вызовы дают тот же результат
- Заменяет весь ресурс

```http
PUT /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"name":"Jane Doe","email":"jane@example.com","age":30}
```

**PHP:**
```php
Route::put('/users/{id}', function (Request $request, $id) {
    $user = User::findOrFail($id);
    $user->update($request->all());  // полная замена
    return response()->json($user);
});
```

### PATCH - Частичное обновление

**Характеристики:**
- **NOT Safe**
- **Idempotent** (должен быть)
- Обновляет только указанные поля

```http
PATCH /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"email":"newemail@example.com"}
```

**PHP:**
```php
Route::patch('/users/{id}', function (Request $request, $id) {
    $user = User::findOrFail($id);
    $user->update($request->only(['email']));  // частичное
    return response()->json($user);
});
```

**PUT vs PATCH:**
```php
// PUT - все поля обязательны
PUT /users/123
{
  "name": "John",
  "email": "john@example.com",
  "age": 30,
  "address": "123 Main St"
}

// PATCH - только изменяемые поля
PATCH /users/123
{
  "email": "newemail@example.com"
}
```

### DELETE - Удаление ресурса

**Характеристики:**
- **NOT Safe**
- **Idempotent** - повторное удаление = тот же результат (404 или 204)

```http
DELETE /api/users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer token123
```

**PHP:**
```php
Route::delete('/users/{id}', function ($id) {
    $user = User::findOrFail($id);
    $user->delete();
    return response()->noContent();  // 204
});
```

### HEAD - Получение заголовков

**Как GET, но без body. Проверка существования ресурса.**

```http
HEAD /api/users/123 HTTP/1.1
Host: api.example.com
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 145
Last-Modified: Mon, 27 Jan 2025 10:00:00 GMT

(no body)
```

**Use cases:**
- Проверка существования файла
- Получение метаданных (размер, дата изменения)
- Валидация кеша

### OPTIONS - Получение доступных методов

**Используется для CORS preflight.**

```http
OPTIONS /api/users HTTP/1.1
Host: api.example.com
Origin: https://frontend.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type
```

**Response:**
```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

**PHP (Laravel CORS):**
```php
// config/cors.php
return [
    'paths' => ['api/*'],
    'allowed_methods' => ['*'],
    'allowed_origins' => ['https://frontend.com'],
    'allowed_headers' => ['*'],
    'max_age' => 86400,
];
```

### Summary Table

| Method | Safe | Idempotent | Cacheable | Use Case |
|--------|------|------------|-----------|----------|
| GET | ✅ | ✅ | ✅ | Read |
| POST | ❌ | ❌ | ❌ | Create |
| PUT | ❌ | ✅ | ❌ | Full update |
| PATCH | ❌ | ✅ | ❌ | Partial update |
| DELETE | ❌ | ✅ | ❌ | Delete |
| HEAD | ✅ | ✅ | ✅ | Metadata |
| OPTIONS | ✅ | ✅ | ❌ | CORS/capabilities |

---

## 🔢 HTTP Status Codes

### 1xx - Informational

**100 Continue**
```http
POST /upload HTTP/1.1
Expect: 100-continue

→ HTTP/1.1 100 Continue
```

**101 Switching Protocols** (WebSockets)
```http
GET /chat HTTP/1.1
Upgrade: websocket

→ HTTP/1.1 101 Switching Protocols
```

### 2xx - Success

**200 OK** - стандартный успех
```php
return response()->json(['message' => 'Success'], 200);
```

**201 Created** - ресурс создан
```php
$user = User::create($request->all());
return response()->json($user, 201)
    ->header('Location', "/api/users/{$user->id}");
```

**202 Accepted** - принято к обработке (async)
```php
Queue::push(new ProcessVideoJob($video));
return response()->json(['message' => 'Processing started'], 202);
```

**204 No Content** - успех, нет тела ответа
```php
$user->delete();
return response()->noContent();  // 204
```

**206 Partial Content** - частичная загрузка (Range requests)
```http
GET /video.mp4 HTTP/1.1
Range: bytes=0-1023

→ HTTP/1.1 206 Partial Content
Content-Range: bytes 0-1023/5000
```

### 3xx - Redirection

**301 Moved Permanently** - постоянный редирект
```php
return redirect('/new-url', 301);
```

**302 Found** - временный редирект
```php
return redirect('/temporary-url');  // 302 по умолчанию
```

**304 Not Modified** - кеш валиден
```php
if ($request->header('If-None-Match') === $etag) {
    return response()->noContent()->setStatusCode(304);
}
```

**307 Temporary Redirect** - сохраняет метод (POST → POST)
**308 Permanent Redirect** - постоянный с сохранением метода

### 4xx - Client Errors

**400 Bad Request** - невалидный запрос
```php
return response()->json(['error' => 'Invalid JSON'], 400);
```

**401 Unauthorized** - не авторизован
```php
if (!Auth::check()) {
    return response()->json(['error' => 'Unauthenticated'], 401);
}
```

**403 Forbidden** - нет прав доступа
```php
if (!$user->can('edit', $post)) {
    return response()->json(['error' => 'Forbidden'], 403);
}
```

**404 Not Found** - ресурс не найден
```php
$user = User::find($id);
if (!$user) {
    return response()->json(['error' => 'User not found'], 404);
}

// или
$user = User::findOrFail($id);  // автоматически 404
```

**405 Method Not Allowed** - метод не поддерживается
```http
GET /api/users
→ 200 OK

POST /api/users
→ 405 Method Not Allowed
Allow: GET, HEAD
```

**409 Conflict** - конфликт (duplicate)
```php
if (User::where('email', $email)->exists()) {
    return response()->json(['error' => 'Email already exists'], 409);
}
```

**410 Gone** - ресурс был удален навсегда
```php
return response()->json(['error' => 'Resource deleted'], 410);
```

**422 Unprocessable Entity** - валидация
```php
$validator = Validator::make($request->all(), [
    'email' => 'required|email',
]);

if ($validator->fails()) {
    return response()->json(['errors' => $validator->errors()], 422);
}
```

**429 Too Many Requests** - rate limiting
```php
// Laravel throttle middleware
Route::middleware('throttle:60,1')->group(function () {
    Route::get('/api/users', ...);
});

// Ответ при превышении:
// HTTP/1.1 429 Too Many Requests
// Retry-After: 60
```

### 5xx - Server Errors

**500 Internal Server Error** - общая ошибка сервера
```php
try {
    // some code
} catch (Exception $e) {
    Log::error($e->getMessage());
    return response()->json(['error' => 'Internal server error'], 500);
}
```

**502 Bad Gateway** - проблема с upstream сервером (nginx → PHP-FPM)

**503 Service Unavailable** - сервис временно недоступен
```php
if (Cache::get('maintenance_mode')) {
    return response()->json(['error' => 'Service unavailable'], 503)
        ->header('Retry-After', 3600);
}
```

**504 Gateway Timeout** - timeout upstream сервера

---

## 📨 HTTP Headers

### Request Headers

**Host** (обязательный в HTTP/1.1)
```http
Host: api.example.com
```

**User-Agent** - идентификация клиента
```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
```

**Accept** - желаемый формат ответа
```http
Accept: application/json
Accept: text/html, application/xml;q=0.9, */*;q=0.8
```

**Accept-Language**
```http
Accept-Language: ru-RU, ru;q=0.9, en;q=0.8
```

**Accept-Encoding** - поддержка сжатия
```http
Accept-Encoding: gzip, deflate, br
```

**Content-Type** - тип тела запроса
```http
Content-Type: application/json
Content-Type: multipart/form-data; boundary=---123
Content-Type: application/x-www-form-urlencoded
```

**Content-Length** - размер тела
```http
Content-Length: 348
```

**Authorization** - аутентификация
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Authorization: Basic dXNlcjpwYXNzd29yZA==
```

**Cookie** - отправка cookies
```http
Cookie: session_id=abc123; user_pref=dark_mode
```

**Referer** - источник перехода
```http
Referer: https://google.com/search?q=example
```

**Origin** - CORS origin
```http
Origin: https://frontend.com
```

**If-None-Match** - условный запрос (ETag)
```http
If-None-Match: "686897696a7c876b7e"
```

**If-Modified-Since** - условный запрос (дата)
```http
If-Modified-Since: Mon, 27 Jan 2025 10:00:00 GMT
```

**Range** - частичная загрузка
```http
Range: bytes=0-1023
```

### Response Headers

**Content-Type** - тип ответа
```http
Content-Type: application/json; charset=utf-8
```

**Content-Length** - размер ответа
```http
Content-Length: 1234
```

**Content-Encoding** - сжатие
```http
Content-Encoding: gzip
```

**Set-Cookie** - установка cookies
```http
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Lax
```

**Location** - редирект или созданный ресурс
```http
HTTP/1.1 201 Created
Location: /api/users/123
```

**Cache-Control** - управление кешем
```http
Cache-Control: public, max-age=3600
Cache-Control: private, no-cache, no-store, must-revalidate
Cache-Control: max-age=0, s-maxage=86400
```

**Expires** - дата истечения кеша
```http
Expires: Wed, 28 Jan 2026 10:00:00 GMT
```

**ETag** - идентификатор версии ресурса
```http
ETag: "686897696a7c876b7e"
```

**Last-Modified** - дата изменения
```http
Last-Modified: Mon, 27 Jan 2025 10:00:00 GMT
```

**Access-Control-Allow-Origin** - CORS
```http
Access-Control-Allow-Origin: https://frontend.com
Access-Control-Allow-Origin: *
```

**Access-Control-Allow-Methods**
```http
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
```

**Access-Control-Allow-Headers**
```http
Access-Control-Allow-Headers: Content-Type, Authorization
```

**Access-Control-Max-Age** - кеш preflight
```http
Access-Control-Max-Age: 86400
```

**X-RateLimit-*** - rate limiting
```http
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1643284800
```

**Retry-After** - повторить после
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

**Server** - информация о сервере
```http
Server: nginx/1.21.0
```

**X-Powered-By** - технология (лучше скрывать)
```http
X-Powered-By: PHP/8.2.0
```

**Strict-Transport-Security** (HSTS)
```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### PHP Examples

```php
// Laravel Response Headers
return response()->json($data)
    ->header('X-Custom-Header', 'value')
    ->withHeaders([
        'Cache-Control' => 'no-cache, private',
        'X-RateLimit-Limit' => 60,
    ]);

// Cache-Control
return response()->json($data)
    ->header('Cache-Control', 'public, max-age=3600');

// ETag
$etag = md5(json_encode($data));
return response()->json($data)
    ->setEtag($etag);

// Conditional GET
if ($request->getETags() && in_array($etag, $request->getETags())) {
    return response('', 304);
}

// CORS
return response()->json($data)
    ->header('Access-Control-Allow-Origin', 'https://frontend.com')
    ->header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');

// Reading request headers
$userAgent = $request->header('User-Agent');
$token = $request->bearerToken();  // Authorization: Bearer
$accept = $request->header('Accept');
```

---

## 🔐 HTTPS (HTTP Secure)

### SSL/TLS

**HTTPS = HTTP + SSL/TLS шифрование**

**Цели:**
1. **Confidentiality** - шифрование данных
2. **Integrity** - защита от изменения
3. **Authentication** - проверка подлинности сервера

### TLS Handshake

```
Client                          Server
  |                               |
  |------- ClientHello --------->|
  |   (supported ciphers,         |
  |    TLS versions)              |
  |                               |
  |<------ ServerHello ----------|
  |   (selected cipher,           |
  |    certificate,               |
  |    public key)                |
  |                               |
  |-- Certificate Verify ------->|
  |                               |
  |<---- Server Finished ---------|
  |                               |
  |------ Client Finished ------->|
  |                               |
  |<====== Encrypted data ======>|
```

**Шаги:**
1. **ClientHello** - клиент отправляет поддерживаемые cipher suites
2. **ServerHello** - сервер выбирает cipher suite и отправляет сертификат
3. **Key Exchange** - обмен ключами (RSA или Diffie-Hellman)
4. **Finished** - handshake завершен
5. **Application Data** - передача зашифрованных данных

### Certificates

**SSL Certificate содержит:**
- **Subject** - домен (CN: example.com)
- **Issuer** - CA (Certificate Authority)
- **Public Key** - публичный ключ
- **Validity** - срок действия
- **Signature** - подпись CA

**Типы:**
1. **DV (Domain Validation)** - только домен
2. **OV (Organization Validation)** - организация
3. **EV (Extended Validation)** - расширенная проверка (зеленая строка)

**Получение сертификата:**

```bash
# Let's Encrypt (бесплатно)
sudo certbot --nginx -d example.com -d www.example.com

# Сертификат автоматически обновляется
sudo certbot renew --dry-run
```

### HSTS (HTTP Strict Transport Security)

**Заставить браузер использовать только HTTPS.**

```php
// Laravel Middleware
return $response->header(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
);
```

**Nginx:**
```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

**HSTS Preload:**
- Браузеры имеют встроенный список HSTS доменов
- Сабмит на https://hstspreload.org/

---

## 🚀 HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.1 (1997)

**Характеристики:**
- Text-based protocol
- Sequential requests (Head-of-Line Blocking)
- Multiple connections для параллелизма (6-8)
- Keep-Alive connections

**Problems:**
```
HTTP/1.1 - 6 connections

Conn 1: [HTML ████████]
Conn 2: [CSS  ██████  ]
Conn 3: [JS   ████████████]
Conn 4: [Img1 ████]
Conn 5: [Img2 ████]
Conn 6: [Img3 ████]

Head-of-Line Blocking: большой файл блокирует connection
```

### HTTP/2 (2015)

**Улучшения:**
1. **Binary Protocol** - бинарный вместо текстового
2. **Multiplexing** - множественные запросы в одном соединении
3. **Header Compression** (HPACK) - сжатие заголовков
4. **Server Push** - сервер отправляет ресурсы без запроса
5. **Stream Prioritization** - приоритеты потоков

**Multiplexing:**
```
HTTP/2 - 1 connection, multiple streams

Connection:
  Stream 1: [HTML ████████]
  Stream 2: [CSS  ██████  ]
  Stream 3: [JS   ████████████]
  Stream 4: [Img1 ████]
  Stream 5: [Img2 ████]
  Stream 6: [Img3 ████]

Параллельно в одном TCP соединении!
```

**Server Push:**
```http
GET /index.html HTTP/2

→ Response:
  - index.html
  - PUSH: /style.css (сервер отправляет без запроса)
  - PUSH: /script.js
```

**Nginx HTTP/2:**
```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # Server Push
    location / {
        http2_push /style.css;
        http2_push /script.js;
    }
}
```

**Laravel (автоматически работает через HTTP/2 если nginx/Apache настроен):**
```php
// Link header для Server Push hint
return response()->view('welcome')
    ->header('Link', '</css/app.css>; rel=preload; as=style');
```

### HTTP/3 (2022)

**Основано на QUIC (Quick UDP Internet Connections).**

**Улучшения:**
1. **UDP вместо TCP** - устранение Head-of-Line Blocking на уровне транспорта
2. **Built-in TLS 1.3** - шифрование по умолчанию
3. **0-RTT Connection** - переподключение без handshake
4. **Improved congestion control** - лучше для мобильных сетей
5. **Connection migration** - смена IP без разрыва (WiFi → 4G)

**TCP Head-of-Line Blocking (HTTP/2 проблема):**
```
TCP packet loss → все streams блокируются пока не retransmit

[Stream 1] [Stream 2] [Stream 3]
     ▼          ▼          ▼
[Packet 1] [Packet 2] [❌ Lost] ← блокирует ВСЁ
```

**QUIC/HTTP/3 решение:**
```
UDP - независимые streams, потеря одного не блокирует другие

[Stream 1] [Stream 2] [Stream 3]
     ✅         ✅         ❌ (только stream 3 ждет)
```

**Поддержка:**
```bash
# Cloudflare автоматически включает HTTP/3
# Nginx QUIC (экспериментально)
# Apache HTTP/3 support (в разработке)
```

### Сравнение

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Protocol | Text | Binary | Binary |
| Transport | TCP | TCP | QUIC (UDP) |
| Connections | Multiple | Single | Single |
| Multiplexing | ❌ | ✅ | ✅ |
| Header Compression | ❌ | ✅ (HPACK) | ✅ (QPACK) |
| Server Push | ❌ | ✅ | ✅ |
| TLS | Optional | Mandatory | Built-in |
| Head-of-Line | TCP + HTTP | TCP only | ❌ None |
| Connection Migration | ❌ | ❌ | ✅ |
| 0-RTT | ❌ | ❌ | ✅ |

---

## 🍪 Cookies

### Set-Cookie

```http
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=3600
```

**Attributes:**
- **Domain** - домен (по умолчанию текущий)
- **Path** - путь (по умолчанию /)
- **Expires** - дата истечения
- **Max-Age** - время жизни в секундах (приоритет над Expires)
- **Secure** - только HTTPS
- **HttpOnly** - недоступна JavaScript (защита от XSS)
- **SameSite** - CSRF защита

### SameSite

```http
SameSite=Strict   # никогда не отправляется cross-site
SameSite=Lax      # отправляется при GET навигации (default Chrome 80+)
SameSite=None     # всегда отправляется (требует Secure)
```

**Example:**
```
User на site-a.com → клик на site-b.com

SameSite=Strict:  cookie НЕ отправляется
SameSite=Lax:     cookie отправляется (GET navigation)
SameSite=None:    cookie отправляется (если Secure)
```

**Laravel:**
```php
// config/session.php
'same_site' => 'lax',

// Set cookie
return response('Hello')
    ->cookie('name', 'value', $minutes, '/', null, true, true, false, 'lax');
    //                                    secure  httponly  raw    samesite

// Get cookie
$value = $request->cookie('name');
```

---

## 🗃️ Caching

### Cache-Control Directives

**Response directives:**
```http
Cache-Control: public                # кешируется proxies + browser
Cache-Control: private               # только browser
Cache-Control: no-cache              # revalidate перед использованием
Cache-Control: no-store              # НЕ кешировать
Cache-Control: max-age=3600          # кеш валиден 1 час
Cache-Control: s-maxage=86400        # shared cache (CDN) - 1 день
Cache-Control: must-revalidate       # revalidate после истечения
Cache-Control: immutable             # никогда не меняется (fonts, hashed assets)
```

**Combinations:**
```http
# Приватный кеш на 1 час
Cache-Control: private, max-age=3600

# Публичный кеш: браузер 1 час, CDN 1 день
Cache-Control: public, max-age=3600, s-maxage=86400

# Никогда не кешировать (API)
Cache-Control: no-cache, no-store, must-revalidate

# Статические ассеты (с хешем в имени)
Cache-Control: public, max-age=31536000, immutable
```

### Conditional Requests

**ETag (Entity Tag):**
```http
# Response
HTTP/1.1 200 OK
ETag: "686897696a7c876b7e"
Content-Type: application/json

{"data": "..."}

# Повторный запрос
GET /api/data HTTP/1.1
If-None-Match: "686897696a7c876b7e"

# Если не изменилось
HTTP/1.1 304 Not Modified
ETag: "686897696a7c876b7e"
(no body - экономия bandwidth)
```

**Last-Modified:**
```http
# Response
HTTP/1.1 200 OK
Last-Modified: Mon, 27 Jan 2025 10:00:00 GMT

# Повторный запрос
GET /api/data HTTP/1.1
If-Modified-Since: Mon, 27 Jan 2025 10:00:00 GMT

# Если не изменилось
HTTP/1.1 304 Not Modified
```

**Laravel:**
```php
// ETag
Route::get('/api/data', function () {
    $data = Cache::remember('data', 3600, fn() => Data::all());
    $etag = md5(json_encode($data));
    
    if (request()->header('If-None-Match') === "\"{$etag}\"") {
        return response('', 304);
    }
    
    return response()->json($data)
        ->setEtag($etag)
        ->setPublic()
        ->setMaxAge(3600);
});

// Last-Modified
Route::get('/posts/{id}', function ($id) {
    $post = Post::findOrFail($id);
    $lastModified = $post->updated_at;
    
    $response = response()->json($post);
    $response->setLastModified($lastModified);
    
    if ($response->isNotModified(request())) {
        return $response;  // 304
    }
    
    return $response;
});
```

### Vary Header

**Кеш зависит от заголовка.**

```http
Vary: Accept-Encoding

# Разные кеши для:
Accept-Encoding: gzip
Accept-Encoding: br
Accept-Encoding: identity
```

```http
Vary: Accept-Language

# Разные кеши для:
Accept-Language: en-US
Accept-Language: ru-RU
```

**Laravel:**
```php
return response()->json($data)
    ->header('Vary', 'Accept-Language');
```

---

## 🔄 Connection Management

### Keep-Alive (HTTP/1.1)

**Переиспользование TCP соединения для множественных запросов.**

```http
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

**Without Keep-Alive:**
```
Request 1: [TCP Handshake] → [HTTP Request/Response] → [TCP Close]
Request 2: [TCP Handshake] → [HTTP Request/Response] → [TCP Close]
Request 3: [TCP Handshake] → [HTTP Request/Response] → [TCP Close]
```

**With Keep-Alive:**
```
[TCP Handshake]
  → Request 1 & Response
  → Request 2 & Response
  → Request 3 & Response
[TCP Close after timeout or max]
```

**PHP:**
```php
// Guzzle keep-alive
$client = new GuzzleHttp\Client([
    'curl' => [
        CURLOPT_TCP_KEEPALIVE => 1,
        CURLOPT_TCP_KEEPIDLE => 120,
    ]
]);
```

---

## 🌐 CORS (Cross-Origin Resource Sharing)

### Same-Origin Policy

**Origin = scheme + host + port**

```
https://example.com:443/page

Same origin:
  https://example.com:443/other  ✅
  https://example.com/page       ✅ (default port 443)

Different origin:
  http://example.com             ❌ (scheme)
  https://api.example.com        ❌ (subdomain)
  https://example.com:8080       ❌ (port)
```

### CORS Flow

**Simple Request (GET, HEAD, POST with simple headers):**
```http
# Request
GET /api/users HTTP/1.1
Host: api.example.com
Origin: https://frontend.com

# Response
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://frontend.com
```

**Preflight Request (PUT, DELETE, custom headers):**
```http
# Preflight (OPTIONS)
OPTIONS /api/users/123 HTTP/1.1
Host: api.example.com
Origin: https://frontend.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: Authorization

# Preflight Response
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400

# Actual Request
DELETE /api/users/123 HTTP/1.1
Host: api.example.com
Origin: https://frontend.com
Authorization: Bearer token
```

**Laravel:**
```php
// config/cors.php
return [
    'paths' => ['api/*'],
    
    'allowed_methods' => ['*'],
    
    'allowed_origins' => [
        'https://frontend.com',
        'https://app.example.com',
    ],
    
    'allowed_origins_patterns' => [
        '/^https:\/\/.*\.example\.com$/',  // любой subdomain
    ],
    
    'allowed_headers' => ['*'],
    
    'exposed_headers' => ['X-Custom-Header'],
    
    'max_age' => 86400,
    
    'supports_credentials' => true,  // cookies
];

// Middleware
Route::middleware('cors')->group(function () {
    Route::get('/api/users', ...);
});
```

**With Credentials (cookies):**
```http
# Request
GET /api/profile HTTP/1.1
Origin: https://frontend.com
Cookie: session=abc123

# Response
Access-Control-Allow-Origin: https://frontend.com  # НЕ *
Access-Control-Allow-Credentials: true

# JavaScript
fetch('https://api.example.com/profile', {
    credentials: 'include'  // send cookies
});
```

---

## 📊 Content Negotiation

### Accept Headers

```http
Accept: application/json
Accept: text/html, application/xml;q=0.9, */*;q=0.8
Accept-Language: ru-RU, ru;q=0.9, en;q=0.8
Accept-Encoding: gzip, deflate, br
```

**Quality values (q):**
- `1.0` - наивысший приоритет (по умолчанию)
- `0.9, 0.8, ...` - убывающий приоритет
- `0` - неприемлемо

**Laravel:**
```php
Route::get('/data', function (Request $request) {
    if ($request->wantsJson()) {
        return response()->json($data);
    }
    
    if ($request->accepts('application/xml')) {
        return response($xmlData)->header('Content-Type', 'application/xml');
    }
    
    return view('data', compact('data'));
});

// Язык
$locale = $request->getPreferredLanguage(['en', 'ru', 'de']);
App::setLocale($locale);
```

---

## 🎓 Для собеседования: ключевые точки

1. **HTTP Methods** - GET (safe, idempotent), POST (not idempotent), PUT (idempotent), PATCH, DELETE
2. **Status Codes** - 2xx success, 3xx redirect, 4xx client error (400/401/403/404/422), 5xx server error
3. **Headers** - Cache-Control, Authorization, Content-Type, CORS headers
4. **HTTPS** - TLS handshake, certificates, HSTS
5. **HTTP/2** - multiplexing, server push, binary protocol
6. **HTTP/3** - QUIC (UDP), 0-RTT, устранение Head-of-Line Blocking
7. **Caching** - ETag, Last-Modified, 304 Not Modified, Cache-Control directives
8. **CORS** - preflight (OPTIONS), Access-Control-Allow-Origin, credentials
9. **Cookies** - HttpOnly (XSS защита), Secure (HTTPS), SameSite (CSRF защита)
10. **Idempotency** - GET/PUT/DELETE idempotent, POST не idempotent

**Главное:** Понимай разницу между методами (safe/idempotent), статус коды (особенно 4xx), кеширование (ETag/Cache-Control), CORS flow (preflight).
