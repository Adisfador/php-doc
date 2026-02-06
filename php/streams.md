# Streams - Потоки в PHP

Полный разбор потоков (streams) в PHP: wrappers, фильтры, контексты, псевдопотоки.

---

## 🎯 Что такое Streams?

**Stream** - это абстракция для работы с последовательными данными (файлы, сеть, память).

**Все I/O операции в PHP работают через streams:**
- `fopen()`, `fread()`, `fwrite()`
- `file_get_contents()`, `file_put_contents()`
- HTTP запросы
- Сетевые соединения

**Преимущества:**
- ✅ Единый интерфейс для разных источников данных
- ✅ Поддержка фильтров (шифрование, сжатие)
- ✅ Настройка через контексты
- ✅ Эффективная работа с большими файлами

---

## 🔍 Streams - фундамент всего I/O в PHP

### Да, ВСЁ держится на streams!

**CLI (Command Line):**
```php
// Любая команда в терминале
echo "Hello";           // → STDOUT stream
trigger_error("Error"); // → STDERR stream
$input = readline();    // ← STDIN stream
```

**Web (HTTP - PHP-FPM, Apache mod_php):**
```php
// HTTP Request
$_GET, $_POST, $_FILES  // ← Парсятся из php://input stream
file_get_contents('php://input'); // ← RAW request body

// HTTP Response
echo "Hello";           // → php://output stream (→ веб-сервер → браузер)
header('Location: /');  // → Headers в output stream

// Errors
error_log("Error");     // → php://stderr stream (→ error.log)
```

**Внутреннее устройство:**

```
┌─────────────────────────────────────────────────────────┐
│                    PHP Application                      │
├─────────────────────────────────────────────────────────┤
│  echo / print / var_dump                                │
│       ↓                                                 │
│  php://output (web) / STDOUT (CLI)                      │
│       ↓                                                 │
│  Output Buffer (ob_start)                               │
│       ↓                                                 │
│  FastCGI/Apache/CLI → Web Server → Browser              │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│               Входящий HTTP запрос                      │
├─────────────────────────────────────────────────────────┤
│  Browser → Web Server → PHP-FPM                         │
│       ↓                                                 │
│  php://input stream                                     │
│       ↓                                                 │
│  $_POST, $_GET (PHP парсит автоматически)               │
│       ↓                                                 │
│  Your Application Code                                  │
└─────────────────────────────────────────────────────────┘
```

### Всё, что вы используете, это streams под капотом:

```php
// ✅ Файлы
file_get_contents('file.txt')  // → file:// wrapper → filesystem stream

// ✅ HTTP клиенты
file_get_contents('http://api.example.com')  // → http:// wrapper → network stream

// ✅ Database (PDO)
new PDO('mysql:host=localhost')  // → tcp socket stream

// ✅ Sessions
session_start()  // → файлы или redis stream

// ✅ Cache (Redis, Memcached)
$redis->get('key')  // → tcp/unix socket stream

// ✅ Logs
error_log('message')  // → file stream или syslog stream

// ✅ Email
mail('to@example.com', 'Subject', 'Body')  // → SMTP socket stream

// ✅ Curl (под капотом)
curl_exec($ch)  // → network stream с SSL wrapper
```

### Практический пример - что происходит при HTTP запросе

```php
// 1. Браузер отправляет: POST /api/users HTTP/1.1
//    Content-Type: application/json
//    {"name": "John"}

// 2. Web Server (Nginx) → FastCGI → PHP-FPM

// 3. PHP читает из stream:
$rawInput = file_get_contents('php://input'); 
// php://input stream получает тело запроса

$data = json_decode($rawInput, true);
// ['name' => 'John']

// 4. Ваш код обрабатывает
$user = User::create($data);

// 5. PHP пишет в output stream:
echo json_encode(['id' => $user->id, 'name' => $user->name]);
// → php://output stream

// 6. PHP-FPM → Nginx → Браузер
// HTTP/1.1 200 OK
// Content-Type: application/json
// {"id": 1, "name": "John"}
```

### SAPI (Server API) определяет streams

```php
// PHP может работать в разных режимах:

// 1. CLI (терминал)
php script.php
// STDIN  ← клавиатура
// STDOUT → терминал
// STDERR → терминал (красным)

// 2. PHP-FPM (FastCGI)
// php://input  ← HTTP request body
// php://output → HTTP response body
// php://stderr → error.log файл

// 3. Apache mod_php
// Аналогично PHP-FPM

// 4. Built-in server (php -S)
// Комбинация CLI + HTTP streams
```

### Проверка текущего SAPI

```php
echo PHP_SAPI; // "cli", "fpm-fcgi", "apache2handler", "cli-server"

if (PHP_SAPI === 'cli') {
    // Работаем с STDIN/STDOUT/STDERR
    fwrite(STDOUT, "Running in CLI\n");
} else {
    // Работаем с php://input и php://output
    echo "Running in web context";
}
```

---

## 📦 Stream Wrappers

### Что это?

**Wrapper** - протокол доступа к данным (`file://`, `http://`, `php://`).

**Встроенные wrappers:**
```php
print_r(stream_get_wrappers());

// Array
// (
//     [0] => file
//     [1] => http
//     [2] => ftp
//     [3] => php
//     [4] => zlib
//     [5] => data
//     [6] => glob
//     [7] => phar
//     [8] => ssh2
// )
```

---

## 🔹 php:// - Псевдопотоки

### php://stdin, php://stdout, php://stderr

**Стандартные потоки ввода/вывода:**

```php
// Чтение из stdin (консольный ввод)
$name = fgets(STDIN);
// или
$name = file_get_contents('php://stdin');

// Запись в stdout
fwrite(STDOUT, "Hello\n");
// или
file_put_contents('php://stdout', "Hello\n");

// Запись в stderr
fwrite(STDERR, "Error!\n");
file_put_contents('php://stderr', "Error!\n");
```

**CLI скрипт с интерактивностью:**
```php
#!/usr/bin/env php
<?php
fwrite(STDOUT, "Enter your name: ");
$name = trim(fgets(STDIN));

if (!$name) {
    fwrite(STDERR, "Name is required!\n");
    exit(1);
}

fwrite(STDOUT, "Hello, {$name}!\n");
```

**💡 Что происходит под капотом:**

```php
// Когда вы пишете:
echo "Hello";

// PHP делает (упрощенно):
// CLI:
fwrite(STDOUT, "Hello");

// Web (PHP-FPM):
fwrite(php://output, "Hello");

// Аналогично:
var_dump($data);  // → STDOUT/php://output
print_r($array);  // → STDOUT/php://output
printf("%.2f", $number);  // → STDOUT/php://output

// Ошибки:
trigger_error("Warning");  // → STDERR/php://stderr
error_log("Error message");  // → STDERR/php://stderr или файл
```

**Даже Laravel/Symfony используют это:**
```php
// Laravel response
return response()->json($data);
// ↓ под капотом
echo json_encode($data);  // → php://output stream

// Laravel redirect
return redirect('/home');
// ↓ под капотом
header('Location: /home');  // → headers в output stream
echo '';  // → php://output stream
```

---

### php://input

**Читает RAW POST data** (тело HTTP запроса).

```php
// Получить JSON из POST body
$json = file_get_contents('php://input');
$data = json_decode($json, true);

// API endpoint
Route::post('/api/webhook', function () {
    $payload = file_get_contents('php://input');
    $signature = hash_hmac('sha256', $payload, config('app.webhook_secret'));
    
    if ($signature !== request()->header('X-Signature')) {
        abort(403);
    }
    
    $data = json_decode($payload, true);
    // Process webhook...
});
```

**⚠️ Важно:**
- `php://input` доступен только один раз (не seekable)
- Не работает с `enctype="multipart/form-data"` (файлы)
- Для повторного чтения используй `file_get_contents()` + кэширование

---

### php://output

**Пишет напрямую в output buffer.**

```php
// Прямая запись в output
file_put_contents('php://output', 'Hello, World!');

// Генерация файла для скачивания
header('Content-Type: text/csv');
header('Content-Disposition: attachment; filename="export.csv"');

$output = fopen('php://output', 'w');
fputcsv($output, ['Name', 'Email', 'Age']);
fputcsv($output, ['John', 'john@example.com', 30]);
fclose($output);
```

---

### php://temp и php://memory

**Временное хранилище в памяти.**

**php://memory** - всегда в RAM:
```php
$memory = fopen('php://memory', 'r+');
fwrite($memory, 'Temporary data');
rewind($memory);
echo fread($memory, 1024); // "Temporary data"
fclose($memory);
```

**php://temp** - в памяти до лимита, потом на диск:
```php
// По умолчанию лимит 2MB, потом создаст временный файл
$temp = fopen('php://temp', 'r+');
fwrite($temp, str_repeat('A', 3 * 1024 * 1024)); // 3MB
// Автоматически переключится на temporary file

rewind($temp);
$data = fread($temp, 1024);
fclose($temp);
```

**Настройка лимита:**
```php
// php://temp/maxmemory:5242880 - 5MB лимит
$temp = fopen('php://temp/maxmemory:5242880', 'r+');
```

**Use case - CSV генерация:**
```php
function generateCsv(array $data): string
{
    $temp = fopen('php://temp', 'r+');
    
    foreach ($data as $row) {
        fputcsv($temp, $row);
    }
    
    rewind($temp);
    $csv = stream_get_contents($temp);
    fclose($temp);
    
    return $csv;
}
```

**⚠️ Важные нюансы php://temp vs реальный файл:**

```php
// 1️⃣ php://temp - АВТОМАТИЧЕСКИ переключается на файл при превышении лимита
$temp = fopen('php://temp', 'r+'); // Лимит 2MB по умолчанию
fwrite($temp, str_repeat('A', 3 * 1024 * 1024)); // 3MB
// PHP автоматически создаст временный файл в sys_get_temp_dir()

// 2️⃣ Контроль лимита памяти
$temp = fopen('php://temp/maxmemory:10485760', 'r+'); // 10MB лимит
// До 10MB - в RAM, после - на диск

// 3️⃣ Реальный временный файл (вручную)
$tempFile = tempnam(sys_get_temp_dir(), 'csv_');
$handle = fopen($tempFile, 'r+');
fwrite($handle, 'data');
fclose($handle);
unlink($tempFile); // Нужно удалять вручную!
```

**Когда использовать что:**

```php
// ✅ php://temp - небольшие данные (до нескольких MB)
function generateSmallCsv(array $data): string
{
    $temp = fopen('php://temp', 'r+');
    foreach ($data as $row) {
        fputcsv($temp, $row);
    }
    rewind($temp);
    $csv = stream_get_contents($temp);
    fclose($temp); // Автоматически удаляется
    return $csv;
}

// ✅ Прямая запись в файл - большие данные (сотни MB/GB)
function generateLargeCsv(array $data, string $outputPath): void
{
    $handle = fopen($outputPath, 'w');
    foreach ($data as $row) {
        fputcsv($handle, $row);
    }
    fclose($handle);
    // Файл остается на диске
}

// ✅ Потоковая запись в HTTP response - экспорт без сохранения
function streamCsvDownload(array $data): void
{
    header('Content-Type: text/csv');
    header('Content-Disposition: attachment; filename="export.csv"');
    
    $output = fopen('php://output', 'w');
    foreach ($data as $row) {
        fputcsv($output, $row); // Пишет сразу в браузер
    }
    fclose($output);
    // Не использует память/диск для хранения!
}

// ✅ Замена существующего файла атомарно
function replaceFileAtomically(string $path, array $data): void
{
    // Создать временный файл рядом с целевым
    $tempFile = $path . '.tmp.' . uniqid();
    
    $handle = fopen($tempFile, 'w');
    foreach ($data as $row) {
        fputcsv($handle, $row);
    }
    fclose($handle);
    
    // Атомарная замена (одна операция)
    rename($tempFile, $path);
    // Если rename упадет - оригинал не тронут!
}
```

**Сравнение производительности:**

```php
// Тест: 100,000 строк CSV

// ❌ ХУДШИЙ: в память → stream_get_contents
$temp = fopen('php://memory', 'r+'); // Всегда в RAM!
foreach ($rows as $row) {
    fputcsv($temp, $row); // 50MB в памяти
}
rewind($temp);
$csv = stream_get_contents($temp); // Дополнительно 50MB
// Итого: 100MB пик памяти

// ⚠️ СРЕДНИЙ: php://temp (автопереключение)
$temp = fopen('php://temp', 'r+'); // 2MB в RAM
foreach ($rows as $row) {
    fputcsv($temp, $row); // После 2MB → временный файл
}
rewind($temp);
$csv = stream_get_contents($temp); // Читает файл → в память
// Итого: 2MB + временный файл + 50MB при чтении

// ✅ ЛУЧШИЙ: прямая запись в файл
$handle = fopen('output.csv', 'w');
foreach ($rows as $row) {
    fputcsv($handle, $row); // Сразу на диск
}
fclose($handle);
// Итого: минимум памяти, один файл

// ✅ ИДЕАЛЬНЫЙ: stream в HTTP response
$output = fopen('php://output', 'w');
foreach ($rows as $row) {
    fputcsv($output, $row); // Сразу клиенту
}
// Итого: почти 0 памяти, 0 файлов!
```

**Что происходит под капотом:**

```
┌─────────────────────────────────────────────────┐
│         php://temp (размер растет)              │
├─────────────────────────────────────────────────┤
│  0 - 2MB:    в памяти (быстро)                  │
│  2MB+:       PHP создает файл в /tmp            │
│              (например: /tmp/php8X42aF)         │
│  fclose():   автоматически удаляет файл         │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│      Реальный временный файл (tempnam)          │
├─────────────────────────────────────────────────┤
│  Создаете:   tempnam() → /tmp/csv_ABC123        │
│  Пишете:     fwrite() → всегда на диск          │
│  Закрываете: fclose() → файл ОСТАЕТСЯ           │
│  Удаляете:   unlink() - ВРУЧНУЮ!                │
└─────────────────────────────────────────────────┘
```

**Проверить, где php://temp:**

```php
$temp = fopen('php://temp', 'r+');
fwrite($temp, str_repeat('A', 3 * 1024 * 1024)); // 3MB

$meta = stream_get_meta_data($temp);
print_r($meta);
// [uri] => /tmp/php8X42aF  ← реальный файл!
// [wrapper_type] => plainfile

fclose($temp);
// Файл /tmp/php8X42aF автоматически удален
```

**Резюме:**

| Метод | Память | Диск | Auto cleanup | Когда использовать |
|-------|--------|------|--------------|-------------------|
| `php://memory` | ВСЁ! | Нет | ✅ | Маленькие данные (<1MB) |
| `php://temp` | До 2MB | Авто | ✅ | Средние данные (1-50MB) |
| `tempnam()` | Мало | Да | ❌ Manual | Нужен контроль файла |
| Прямой файл | Мало | Да | ❌ | Большие данные (50MB+) |
| `php://output` | ~0 | Нет | ✅ | HTTP streaming |

**Вывод:** `php://temp` автоматически превращается в реальный файл при превышении лимита (2MB), поэтому для больших данных разницы нет! Но для **очень больших** данных лучше писать сразу в целевой файл или стримить в `php://output`.

---

### php://filter

**Применяет фильтры к потокам.**

**Синтаксис:**
```
php://filter/read=<filter>/resource=<path>
php://filter/write=<filter>/resource=<path>
```

**Base64 encode файла:**
```php
$encoded = file_get_contents('php://filter/convert.base64-encode/resource=file.txt');
```

**Base64 decode:**
```php
file_put_contents(
    'php://filter/convert.base64-decode/resource=output.bin',
    $base64Data
);
```

**Множественные фильтры (цепочка):**
```php
$data = file_get_contents(
    'php://filter/read=string.toupper|convert.base64-encode/resource=file.txt'
);
// Сначала toupper, потом base64
```

**ROT13 шифрование:**
```php
$encrypted = file_get_contents(
    'php://filter/read=string.rot13/resource=secret.txt'
);
```

---

### data:// - Data URI scheme

**Встраивание данных в URL.**

```php
// Текст
$data = file_get_contents('data://text/plain,Hello World');
echo $data; // "Hello World"

// Base64
$data = file_get_contents('data://text/plain;base64,SGVsbG8gV29ybGQ=');
echo $data; // "Hello World"

// HTML
$html = file_get_contents('data://text/html,<h1>Title</h1>');
```

**Use case - тестирование:**
```php
// Создать "файл" без создания реального файла
$image = imagecreatefromstring(
    file_get_contents('data://image/png;base64,' . $base64Image)
);
```

---

### glob:// - Поиск файлов

```php
// Найти все .txt файлы
foreach (glob('*.txt') as $file) {
    echo $file . "\n";
}

// Через stream
$files = opendir('glob://path/to/*.{jpg,png}', GLOB_BRACE);
while (false !== ($file = readdir($files))) {
    echo $file . "\n";
}
closedir($files);
```

---

## 🌐 http:// и https://

**HTTP/HTTPS запросы через fopen:**

```php
// Простой GET
$html = file_get_contents('https://example.com');

// С параметрами
$html = file_get_contents('https://api.example.com/users?page=1');
```

**⚠️ По умолчанию:**
- Метод: GET
- Timeout: default_socket_timeout (60s)
- Не следует редиректам автоматически
- Не отправляет User-Agent

**Для настройки используй Stream Context!**

---

## ⚙️ Stream Contexts - Контексты

### Что это?

**Context** - набор опций и параметров для stream.

### Создание контекста

```php
$context = stream_context_create([
    'http' => [
        'method' => 'POST',
        'header' => "Content-Type: application/json\r\n",
        'content' => json_encode(['name' => 'John']),
        'timeout' => 10,
    ]
]);

$response = file_get_contents('https://api.example.com/users', false, $context);
```

### HTTP контекст - опции

```php
$context = stream_context_create([
    'http' => [
        // Метод
        'method' => 'POST',
        
        // Заголовки (строка или массив)
        'header' => [
            'Content-Type: application/json',
            'Authorization: Bearer token123',
            'User-Agent: My App/1.0',
        ],
        
        // Тело запроса
        'content' => json_encode(['key' => 'value']),
        
        // Timeout
        'timeout' => 30,
        
        // Следовать редиректам
        'follow_location' => true,
        'max_redirects' => 5,
        
        // Игнорировать ошибки (не выбрасывать warning на 404/500)
        'ignore_errors' => true,
        
        // Proxy
        'proxy' => 'tcp://proxy.example.com:8080',
        'request_fulluri' => true, // Для proxy
    ]
]);

$response = file_get_contents('https://api.example.com', false, $context);

// Получить response headers
$headers = $http_response_header; // Автоматически создается!
print_r($headers);
```

### SSL контекст

```php
$context = stream_context_create([
    'ssl' => [
        // Проверка сертификата
        'verify_peer' => true,
        'verify_peer_name' => true,
        
        // CA bundle
        'cafile' => '/path/to/cacert.pem',
        
        // Самоподписанный сертификат (для разработки)
        'verify_peer' => false,
        'verify_peer_name' => false,
        
        // Клиентский сертификат
        'local_cert' => '/path/to/client.pem',
        'local_pk' => '/path/to/client.key',
        'passphrase' => 'secret',
        
        // Криптография
        'ciphers' => 'HIGH:!aNULL:!MD5',
        'crypto_method' => STREAM_CRYPTO_METHOD_TLSv1_2_CLIENT,
    ]
]);

$response = file_get_contents('https://secure-api.com', false, $context);
```

### Практический пример - API клиент

```php
class ApiClient
{
    private string $baseUrl;
    private string $token;
    
    public function __construct(string $baseUrl, string $token)
    {
        $this->baseUrl = $baseUrl;
        $this->token = $token;
    }
    
    public function post(string $endpoint, array $data): array
    {
        $context = stream_context_create([
            'http' => [
                'method' => 'POST',
                'header' => [
                    'Content-Type: application/json',
                    "Authorization: Bearer {$this->token}",
                ],
                'content' => json_encode($data),
                'timeout' => 10,
                'ignore_errors' => true,
            ]
        ]);
        
        $response = file_get_contents($this->baseUrl . $endpoint, false, $context);
        
        // Проверка статуса
        if (!preg_match('/HTTP\/\d\.\d\s+(\d+)/', $http_response_header[0], $matches)) {
            throw new Exception('Invalid response');
        }
        
        $statusCode = (int)$matches[1];
        
        if ($statusCode >= 400) {
            throw new Exception("API Error: {$statusCode}");
        }
        
        return json_decode($response, true);
    }
}

// Использование
$client = new ApiClient('https://api.example.com', 'token123');
$user = $client->post('/users', ['name' => 'John', 'email' => 'john@example.com']);
```

---

## 🔧 Stream Filters - Фильтры

### Что это?

**Filter** - преобразование данных на лету (чтение/запись).

### Встроенные фильтры

```php
print_r(stream_get_filters());

// Array
// (
//     [0] => convert.iconv.*
//     [1] => string.rot13
//     [2] => string.toupper
//     [3] => string.tolower
//     [4] => string.strip_tags
//     [5] => convert.base64-encode
//     [6] => convert.base64-decode
//     [7] => zlib.deflate
//     [8] => zlib.inflate
//     [9] => mcrypt.*
//     [10] => mdecrypt.*
// )
```

### Применение фильтров

```php
// Открыть файл
$handle = fopen('file.txt', 'r');

// Добавить фильтр
stream_filter_append($handle, 'string.toupper', STREAM_FILTER_READ);

// Читать с фильтром
while ($line = fgets($handle)) {
    echo $line; // ВСЕ В UPPERCASE
}

fclose($handle);
```

### Фильтры сжатия

```php
// Сжать файл при записи
$output = fopen('output.gz', 'w');
stream_filter_append($output, 'zlib.deflate', STREAM_FILTER_WRITE, [
    'level' => 9, // Максимальное сжатие
]);

fwrite($output, 'Large data...');
fclose($output);

// Распаковать при чтении
$input = fopen('output.gz', 'r');
stream_filter_append($input, 'zlib.inflate', STREAM_FILTER_READ);

$data = stream_get_contents($input);
fclose($input);
```

### Base64 фильтры

```php
// Encode при записи
$output = fopen('encoded.txt', 'w');
stream_filter_append($output, 'convert.base64-encode', STREAM_FILTER_WRITE);
fwrite($output, 'Secret data');
fclose($output);

// Decode при чтении
$input = fopen('encoded.txt', 'r');
stream_filter_append($input, 'convert.base64-decode', STREAM_FILTER_READ);
$decoded = stream_get_contents($input);
fclose($input);
```

### Цепочка фильтров

```php
$handle = fopen('file.txt', 'r');

// Применить несколько фильтров последовательно
stream_filter_append($handle, 'string.toupper', STREAM_FILTER_READ);
stream_filter_append($handle, 'string.rot13', STREAM_FILTER_READ);

// Данные проходят: file → toupper → rot13 → output
```

### Удаление фильтра

```php
$handle = fopen('file.txt', 'r');
$filter = stream_filter_append($handle, 'string.toupper', STREAM_FILTER_READ);

// Читать с фильтром
$data1 = fread($handle, 100);

// Удалить фильтр
stream_filter_remove($filter);

// Читать без фильтра
$data2 = fread($handle, 100);
```

---

## 🎨 Кастомные Stream Filters

### Создание своего фильтра

```php
class MyUppercaseFilter extends php_user_filter
{
    public function filter($in, $out, &$consumed, $closing): int
    {
        while ($bucket = stream_bucket_make_writeable($in)) {
            $bucket->data = strtoupper($bucket->data);
            $consumed += $bucket->datalen;
            stream_bucket_append($out, $bucket);
        }
        
        return PSFS_PASS_ON;
    }
}

// Регистрация
stream_filter_register('custom.uppercase', MyUppercaseFilter::class);

// Использование
$handle = fopen('file.txt', 'r');
stream_filter_append($handle, 'custom.uppercase', STREAM_FILTER_READ);
echo stream_get_contents($handle); // ВСЕ В UPPERCASE
```

### Фильтр с параметрами

```php
class ReplaceFilter extends php_user_filter
{
    private string $search;
    private string $replace;
    
    public function onCreate(): bool
    {
        // $this->params - параметры из stream_filter_append
        if (isset($this->params['search']) && isset($this->params['replace'])) {
            $this->search = $this->params['search'];
            $this->replace = $this->params['replace'];
            return true;
        }
        return false;
    }
    
    public function filter($in, $out, &$consumed, $closing): int
    {
        while ($bucket = stream_bucket_make_writeable($in)) {
            $bucket->data = str_replace($this->search, $this->replace, $bucket->data);
            $consumed += $bucket->datalen;
            stream_bucket_append($out, $bucket);
        }
        
        return PSFS_PASS_ON;
    }
}

stream_filter_register('custom.replace', ReplaceFilter::class);

$handle = fopen('file.txt', 'r');
stream_filter_append($handle, 'custom.replace', STREAM_FILTER_READ, [
    'search' => 'old',
    'replace' => 'new',
]);
```

---

## 📝 Кастомные Stream Wrappers

### Создание своего wrapper

```php
class VariableStream
{
    private int $position;
    private string $data;
    
    public function stream_open(string $path, string $mode): bool
    {
        // var://myvar
        $varName = str_replace('var://', '', $path);
        
        if (!isset($GLOBALS[$varName])) {
            return false;
        }
        
        $this->data = $GLOBALS[$varName];
        $this->position = 0;
        
        return true;
    }
    
    public function stream_read(int $count): string
    {
        $result = substr($this->data, $this->position, $count);
        $this->position += strlen($result);
        return $result;
    }
    
    public function stream_eof(): bool
    {
        return $this->position >= strlen($this->data);
    }
    
    public function stream_stat(): array
    {
        return [
            'size' => strlen($this->data),
        ];
    }
    
    // Другие методы...
}

// Регистрация
stream_wrapper_register('var', VariableStream::class);

// Использование
$myvar = 'Hello from variable!';
$content = file_get_contents('var://myvar');
echo $content; // "Hello from variable!"
```

### Wrapper для кэширования HTTP

```php
class CachedHttpWrapper
{
    private $handle;
    private string $cacheDir = '/tmp/http_cache';
    
    public function stream_open(string $path, string $mode): bool
    {
        $cacheKey = md5($path);
        $cachePath = "{$this->cacheDir}/{$cacheKey}";
        
        // Проверить кэш
        if (file_exists($cachePath) && (time() - filemtime($cachePath)) < 3600) {
            $this->handle = fopen($cachePath, 'r');
            return true;
        }
        
        // Загрузить и кэшировать
        $realPath = str_replace('cachedhttp://', 'http://', $path);
        $data = file_get_contents($realPath);
        
        if (!is_dir($this->cacheDir)) {
            mkdir($this->cacheDir, 0777, true);
        }
        
        file_put_contents($cachePath, $data);
        $this->handle = fopen($cachePath, 'r');
        
        return true;
    }
    
    public function stream_read(int $count): string
    {
        return fread($this->handle, $count);
    }
    
    public function stream_eof(): bool
    {
        return feof($this->handle);
    }
    
    // ... другие методы
}

stream_wrapper_register('cachedhttp', CachedHttpWrapper::class);

// Первый раз - загрузит
$html = file_get_contents('cachedhttp://example.com');

// Второй раз - из кэша
$html = file_get_contents('cachedhttp://example.com');
```

---

## 🎯 Stream Metadata

### stream_get_meta_data()

```php
$handle = fopen('http://example.com', 'r');
$meta = stream_get_meta_data($handle);

print_r($meta);
// Array
// (
//     [timed_out] => 
//     [blocked] => 1
//     [eof] => 
//     [wrapper_type] => http
//     [stream_type] => tcp_socket/ssl
//     [mode] => r
//     [unread_bytes] => 0
//     [seekable] => 
//     [uri] => http://example.com
// )
```

### Полезные проверки

```php
$meta = stream_get_meta_data($handle);

// Timeout
if ($meta['timed_out']) {
    throw new Exception('Connection timed out');
}

// EOF
if ($meta['eof']) {
    echo "Reached end of stream\n";
}

// Seekable
if (!$meta['seekable']) {
    echo "Cannot seek in this stream\n";
}
```

---

## 📊 Статистика потоков

### stat() vs fstat()

```php
// stat - по имени файла
$info = stat('file.txt');

// fstat - по handle
$handle = fopen('file.txt', 'r');
$info = fstat($handle);

print_r($info);
// Array
// (
//     [size] => 1024
//     [atime] => 1234567890  // Last access
//     [mtime] => 1234567890  // Last modification
//     [ctime] => 1234567890  // Last change (metadata)
// )
```

---

## 🔒 File Locking

### flock() - блокировка файлов

```php
$handle = fopen('counter.txt', 'r+');

// Эксклюзивная блокировка (LOCK_EX)
if (flock($handle, LOCK_EX)) {
    $count = (int)fread($handle, 100);
    $count++;
    
    ftruncate($handle, 0);
    rewind($handle);
    fwrite($handle, (string)$count);
    
    flock($handle, LOCK_UN); // Разблокировать
}

fclose($handle);
```

**Типы блокировок:**
```php
LOCK_SH  // Shared lock (чтение) - несколько процессов могут читать
LOCK_EX  // Exclusive lock (запись) - только один процесс
LOCK_UN  // Unlock
LOCK_NB  // Non-blocking (не ждать, вернуть false если заблокирован)
```

**Non-blocking:**
```php
if (flock($handle, LOCK_EX | LOCK_NB)) {
    // Получили блокировку
} else {
    echo "File is locked by another process\n";
}
```

---

## 🎓 Best Practices

### 1. Всегда закрывай streams

```php
// ❌ Плохо
file_get_contents('large-file.txt');

// ✅ Хорошо для больших файлов
$handle = fopen('large-file.txt', 'r');
while (!feof($handle)) {
    $chunk = fread($handle, 8192);
    processChunk($chunk);
}
fclose($handle);
```

### 2. Используй stream context для HTTP

```php
// ❌ Плохо - нет timeout, user-agent
file_get_contents('https://api.example.com');

// ✅ Хорошо
$context = stream_context_create([
    'http' => [
        'timeout' => 10,
        'user_agent' => 'MyApp/1.0',
        'ignore_errors' => true,
    ]
]);
file_get_contents('https://api.example.com', false, $context);
```

### 3. php://temp для временных данных

```php
// ❌ Плохо - создает реальный файл
$tempFile = tempnam(sys_get_temp_dir(), 'prefix');
file_put_contents($tempFile, $data);
// ... используем
unlink($tempFile);

// ✅ Хорошо - в памяти
$temp = fopen('php://temp', 'r+');
fwrite($temp, $data);
rewind($temp);
// ... используем
fclose($temp); // Автоматически освобождается
```

### 4. Фильтры для преобразований

```php
// ❌ Плохо - загружает весь файл в память
$content = file_get_contents('huge.txt');
$uppercase = strtoupper($content);
file_put_contents('output.txt', $uppercase);

// ✅ Хорошо - потоковая обработка
$input = fopen('huge.txt', 'r');
$output = fopen('output.txt', 'w');

stream_filter_append($input, 'string.toupper', STREAM_FILTER_READ);
stream_copy_to_stream($input, $output);

fclose($input);
fclose($output);
```

---

## 🎓 Для собеседования: ключевые точки

1. **Stream Wrappers** - `file://`, `http://`, `php://` - протоколы доступа к данным
2. **php://input** - RAW POST data, доступен один раз, не работает с multipart
3. **php://temp** - в памяти до лимита, потом на диск (по умолчанию 2MB)
4. **php://memory** - всегда в RAM
5. **php://filter** - применение фильтров (`php://filter/read=convert.base64-encode/resource=file.txt`)
6. **Stream Context** - настройка HTTP (метод, заголовки, timeout), SSL (сертификаты)
7. **Stream Filters** - преобразование на лету (zlib, base64, string.toupper)
8. **Custom Wrappers** - `stream_wrapper_register()`, реализация `stream_open/read/write`
9. **Custom Filters** - `stream_filter_register()`, наследование `php_user_filter`
10. **flock()** - блокировка файлов (LOCK_SH, LOCK_EX, LOCK_NB)

### ❓ Популярные вопросы на собеседованиях

**Q: "Что происходит, когда вы пишете `echo 'Hello';`?"**

A: 
```
CLI:   echo → STDOUT stream → терминал
Web:   echo → php://output stream → output buffer → FastCGI → Nginx → браузер
```

**Q: "Вся ли коммуникация в PHP идет через streams?"**

A: **Да!** Любой I/O в PHP:
- HTTP запросы/ответы → `php://input` / `php://output`
- Файлы → `file://` wrapper
- Database → TCP socket streams (PDO)
- Redis/Memcached → network streams
- Email → SMTP socket stream
- Logs → `php://stderr` или file stream

**Q: "В чем разница между STDOUT и php://output?"**

A:
- `STDOUT` - для CLI (терминал)
- `php://output` - для веб (HTTP response)
- PHP автоматически выбирает нужный в зависимости от SAPI
- `echo` использует правильный stream автоматически

**Q: "Можно ли читать php://input дважды?"**

A: **Нет!** `php://input` - не seekable stream, можно прочитать только один раз:
```php
// ✅ Правильно
$input = file_get_contents('php://input');
$data1 = $input;
$data2 = $input; // Используем переменную

// ❌ Неправильно
$data1 = file_get_contents('php://input');
$data2 = file_get_contents('php://input'); // Пусто!
```

**Q: "Зачем нужен Stream Context?"**

A: Для настройки поведения stream без изменения кода:
- HTTP: метод, headers, timeout, proxy
- SSL: сертификаты, шифрование
- FTP: пассивный режим, timeout
```php
$ctx = stream_context_create(['http' => ['timeout' => 5]]);
file_get_contents('http://slow-api.com', false, $ctx);
```

**Q: "В чем преимущество stream filters?"**

A: **Потоковая обработка** без загрузки всего файла в память:
```php
// ❌ Плохо: 1GB файл → 1GB RAM
$data = file_get_contents('huge.txt');
$uppercase = strtoupper($data);

// ✅ Хорошо: обработка кусками
$in = fopen('huge.txt', 'r');
$out = fopen('output.txt', 'w');
stream_filter_append($in, 'string.toupper', STREAM_FILTER_READ);
stream_copy_to_stream($in, $out); // Малое использование памяти
```

### 🎯 Главная концепция

**Streams - это универсальный интерфейс для ВСЕХ I/O операций в PHP.**

Неважно, работаете ли вы с:
- Файлами
- HTTP API
- Базой данных  
- Кешем (Redis)
- Логами
- WebSockets
- Email

**Всё это - streams под капотом!**

```php
// Одинаковый API для разных источников:
$local = fopen('file.txt', 'r');         // Локальный файл
$http = fopen('http://api.com', 'r');    // HTTP
$memory = fopen('php://memory', 'r+');   // Память
$custom = fopen('custom://data', 'r');   // Ваш wrapper

// И для всех работают одни и те же функции:
fread($local, 1024);
fread($http, 1024);
fread($memory, 1024);
fread($custom, 1024);
```

**Главное:** Streams - это фундаментальная абстракция PHP для работы с любыми последовательными данными (файлы, сеть, память). Фильтры и контексты позволяют гибко настраивать обработку.
