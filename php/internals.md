# Внутреннее устройство PHP (Deep Dive)

## Zend Engine

### Архитектура

```
┌─────────────────────────────────────┐
│      PHP Code (source.php)          │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│      Lexer (Tokenization)           │
│      Превращает код в токены        │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│      Parser (Parsing)               │
│      Создает AST (Abstract Syntax   │
│      Tree) из токенов               │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│      Compiler (Compilation)         │
│      AST → OpCodes (байт-код)       │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│      OpCode Cache (OpCache)         │
│      Кеширование скомпилированного  │
│      байт-кода                      │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│      Zend VM (Execution)            │
│      Выполнение OpCodes             │
└─────────────────────────────────────┘
```

### Цикл выполнения PHP скрипта

1. **Module Initialization (MINIT)**
   - Запускается один раз при старте PHP
   - Инициализация расширений
   - Регистрация констант, классов

2. **Request Initialization (RINIT)**
   - Запускается для каждого запроса
   - Инициализация суперглобалов ($_GET, $_POST и т.д.)
   - Сброс состояния

3. **Execution**
   - Компиляция и выполнение скрипта

4. **Request Shutdown (RSHUTDOWN)**
   - Завершение запроса
   - Запись сессий, закрытие соединений

5. **Module Shutdown (MSHUTDOWN)**
   - Один раз при завершении PHP
   - Очистка ресурсов расширений

---

## OpCodes и Zend VM

### OpCodes

OpCode = операция + операнды

```php
<?php
$a = 1 + 2;
```

**Компилируется в:**
```
ASSIGN $a, (ADD 1, 2)

// Детально:
L0:   ASSIGN CV0($a) int(3)
L1:   RETURN int(1)
```

**Просмотр OpCodes:**
```bash
php -d opcache.opt_debug_level=0x10000 script.php
```

Или через [VLD extension](https://github.com/derickr/vld):
```bash
php -d vld.active=1 -d vld.execute=0 script.php
```

### Типы операндов

| Тип | Описание | Пример |
|-----|----------|--------|
| **IS_CONST** | Константа | `1`, `"hello"` |
| **IS_TMP_VAR** | Временная переменная | `~0` (результат выражения) |
| **IS_VAR** | Переменная | `$0` (переменная из функции) |
| **IS_CV** | Compiled Variable | `!0` (переменная из текущего scope) |
| **IS_UNUSED** | Не используется | |

### Zend VM Execution

**Три режима выполнения:**

1. **CALL mode** (по умолчанию)
```c
while (1) {
    switch (opcode) {
        case ZEND_ADD: /* ... */ break;
        case ZEND_SUB: /* ... */ break;
        // ...
    }
}
```

2. **GOTO mode** (быстрее)
```c
ZEND_ADD:
    /* execute ADD */
    goto *opcode_handlers[next_opcode];

ZEND_SUB:
    /* execute SUB */
    goto *opcode_handlers[next_opcode];
```

3. **HYBRID mode** (оптимизация горячих путей)

---

## OpCache

### Как работает

1. **При первом запросе:**
   - PHP компилирует скрипт в OpCodes
   - OpCache сохраняет OpCodes в shared memory
   - Ключ = полный путь к файлу

2. **При последующих запросах:**
   - Проверка timestamp файла (если `opcache.validate_timestamps=1`)
   - Если файл не изменился → использование закешированных OpCodes
   - Пропускается лексинг, парсинг, компиляция

### Структура в памяти

```
┌────────────────────────────────────────┐
│     Shared Memory Segment              │
├────────────────────────────────────────┤
│  Hash Table (filepath → opcodes)       │
│  ┌──────────────────────────────────┐  │
│  │ /var/www/index.php → OpArray     │  │
│  │ /var/www/app.php → OpArray       │  │
│  └──────────────────────────────────┘  │
├────────────────────────────────────────┤
│  Interned Strings (деудупликация)     │
│  "hello" → одна копия в памяти        │
├────────────────────────────────────────┤
│  OpCode arrays                         │
└────────────────────────────────────────┘
```

### Настройки OpCache

```ini
; Включить OpCache
opcache.enable=1

; Размер памяти для OpCache (128 MB)
opcache.memory_consumption=128

; Количество файлов
opcache.max_accelerated_files=10000

; Проверять изменения файлов
opcache.validate_timestamps=1

; Частота проверки (секунды)
opcache.revalidate_freq=2

; Interned strings buffer
opcache.interned_strings_buffer=16

; JIT (PHP 8.0+)
opcache.jit_buffer_size=100M
opcache.jit=1255
```

### Мониторинг OpCache

```php
// Статистика
$status = opcache_get_status();
print_r($status);

// Сброс кеша
opcache_reset();

// Инвалидация файла
opcache_invalidate('/path/to/file.php', true);
```

---

## JIT (Just-In-Time Compilation) PHP 8.0+

### Как работает JIT

```
PHP Code → OpCodes → JIT → Machine Code (x86/ARM)
                     ↓
            Профилирование горячих участков
```

**JIT компилирует:**
- Горячие циклы (loops)
- Математические операции
- Типизированный код

**JIT НЕ помогает:**
- I/O операции (DB queries, file operations)
- Большинство веб-приложений (узкое место - DB)
- Код с динамической типизацией

### Режимы JIT

Формат: `opcache.jit=CRTO`

- **C** - CPU-specific optimization (0-5)
- **R** - JIT register allocation (0-2)
- **T** - JIT trigger (0-5)
  - 0 = отключен
  - 3 = tracing (по умолчанию)
  - 4 = function
  - 5 = tracing + function
- **O** - Optimization level (0-5)

**Примеры:**
```ini
opcache.jit=1255  # Максимальная оптимизация
opcache.jit=tracing  # То же самое
opcache.jit=off  # Отключено
```

---

## Memory Management

### Zend Memory Manager (ZMM)

**Цели:**
- Быстрое выделение/освобождение памяти
- Избегание фрагментации
- Автоматическая очистка после запроса

### Структура памяти

```
┌────────────────────────────────────┐
│   Heap (OS memory)                 │
├────────────────────────────────────┤
│   Chunks (2 MB)                    │
│   ┌────────────────────┐           │
│   │  Pages (4 KB)      │           │
│   │  ┌──────────────┐  │           │
│   │  │  Slots       │  │           │
│   │  │  (8, 16, 32..│  │           │
│   │  │   bytes)     │  │           │
│   │  └──────────────┘  │           │
│   └────────────────────┘           │
└────────────────────────────────────┘
```

**Размеры аллокаций:**
- **Small** (< 3KB): Slab allocator (быстро)
- **Large** (≥ 3KB): Direct allocation

### Reference Counting и Garbage Collector

**Reference Counting:**
```php
$a = "hello";  // refcount=1
$b = $a;       // refcount=2
unset($a);     // refcount=1
unset($b);     // refcount=0 → освобождение памяти
```

**Проблема циклических ссылок:**
```php
$a = [];
$a[0] = &$a;  // Циклическая ссылка
unset($a);    // refcount > 0, но недостижимо!
```

**Garbage Collector (PHP 5.3+):**
- Запускается при превышении порога
- Находит циклические ссылки
- Освобождает память

```php
// Запуск GC вручную
gc_collect_cycles();

// Статистика
print_r(gc_status());

// Отключить GC (не рекомендуется)
gc_disable();
```

---

## Copy-on-Write (COW)

```php
$a = range(1, 1000000);  // Выделение памяти
$b = $a;                 // НЕ копируется! Только refcount++

$b[0] = 999;            // Теперь КОПИРУЕТСЯ (separation)
```

**Внутренняя структура zval:**
```c
struct _zval_struct {
    zend_value value;   // Значение
    union {
        uint32_t type_info;
        struct {
            uint8_t type;          // IS_STRING, IS_ARRAY, etc.
            uint8_t type_flags;    // IS_TYPE_REFCOUNTED, etc.
            uint8_t extra;
        } v;
    } u1;
    uint32_t u2;  // Extra data
};
```

---

## Функции и стек вызовов

### Call Stack

```php
function a() {
    b();
}

function b() {
    c();
}

function c() {
    debug_print_backtrace();
}

a();
```

**Вывод:**
```
#0  c() called at [script.php:7]
#1  b() called at [script.php:3]
#2  a() called at [script.php:11]
```

### Stack Frame

При вызове функции создается stack frame:
- **Local variables**
- **Arguments**
- **Return address**
- **$this** (для методов)

### Recursion и Stack Overflow

```php
function factorial($n) {
    if ($n <= 1) return 1;
    return $n * factorial($n - 1);
}

// PHP имеет ограничение на глубину стека
factorial(100000); // Fatal error: Maximum function nesting level reached
```

**Решения:**
- Tail recursion (PHP не оптимизирует)
- Итеративный подход
- Увеличение `xdebug.max_nesting_level` (если используется Xdebug)

---

## Строки (Strings)

### Структура строки (zend_string)

```c
struct _zend_string {
    zend_refcounted_h gc;      // Refcount
    zend_ulong        h;       // Hash value (для ключей массивов)
    size_t            len;     // Длина
    char              val[1];  // Сами данные (flexible array member)
};
```

### Interned Strings

- Глобальный пул строк
- Одинаковые строки хранятся один раз
- Используется в OpCache и для ключей массивов

```php
$a = "hello";
$b = "hello";
// В памяти одна копия "hello"
```

### String Optimization

```php
// Медленно: конкатенация создает промежуточные строки
$result = '';
for ($i = 0; $i < 10000; $i++) {
    $result .= $i;  // Копирование на каждой итерации
}

// Быстро: массив + implode
$parts = [];
for ($i = 0; $i < 10000; $i++) {
    $parts[] = $i;
}
$result = implode('', $parts);
```

---

## Массивы (Arrays)

### Структура массива (HashTable)

```c
struct _zend_array {
    zend_refcounted_h gc;
    union {
        struct {
            uint32_t flags;
            uint32_t nNumOfElements;    // Количество элементов
            uint32_t nTableSize;        // Размер таблицы (степень 2)
            uint32_t nInternalPointer;
            zend_long nNextFreeElement; // Следующий числовой ключ
            // ...
        } v;
    } u;
    Bucket *arData;  // Массив buckets
};

typedef struct _Bucket {
    zval val;           // Значение
    zend_ulong h;       // Hash числового ключа или строки
    zend_string *key;   // Строковый ключ (или NULL для числового)
} Bucket;
```

### Типы массивов

1. **Packed Array** (оптимизированный)
```php
$a = [1, 2, 3, 4];  // Последовательные числовые ключи
```
- Меньше памяти
- Быстрее доступ

2. **Hash Array**
```php
$a = ['key' => 'value', 10 => 'ten'];
```
- Полноценная hash table

### Внутренний порядок

```php
$a = [];
$a[2] = 'two';
$a[0] = 'zero';
$a[1] = 'one';

foreach ($a as $k => $v) {
    echo "$k: $v\n";
}
// Вывод: 2, 0, 1 (порядок вставки!)
```

---

## Расширения (Extensions)

### Типы расширений

1. **Core extensions** (встроены в PHP)
   - SPL, PDO, JSON, etc.

2. **Bundled extensions** (поставляются с PHP, но требуют включения)
   - MySQLi, GD, Curl

3. **PECL extensions** (устанавливаются отдельно)
   - Redis, MongoDB, Xdebug

### Lifecycle расширения

```c
// Инициализация модуля (при запуске PHP)
PHP_MINIT_FUNCTION(myext) {
    // Регистрация констант, классов
    return SUCCESS;
}

// Инициализация запроса
PHP_RINIT_FUNCTION(myext) {
    // Подготовка к запросу
    return SUCCESS;
}

// Завершение запроса
PHP_RSHUTDOWN_FUNCTION(myext) {
    // Очистка после запроса
    return SUCCESS;
}

// Завершение модуля
PHP_MSHUTDOWN_FUNCTION(myext) {
    // Очистка при выключении PHP
    return SUCCESS;
}
```

---

## Autoloading

### SPL Autoload

```php
spl_autoload_register(function ($class) {
    $file = str_replace('\\', '/', $class) . '.php';
    if (file_exists($file)) {
        require $file;
    }
});
```

**Внутренняя работа:**
1. PHP не находит класс
2. Вызывает зарегистрированные autoloaders
3. Если класс найден → возвращается к выполнению
4. Иначе → Fatal error

### Composer Autoload

Генерирует оптимизированные autoload файлы:

```php
// vendor/composer/autoload_classmap.php
return [
    'App\\User' => $baseDir . '/app/User.php',
    'App\\Post' => $baseDir . '/app/Post.php',
];
```

**Типы:**
- **classmap** - самый быстрый (прямая карта)
- **PSR-4** - стандарт autoload по namespace
- **PSR-0** - устаревший
- **files** - всегда включаются

---

## Profiling и отладка

### Xdebug

```ini
[xdebug]
zend_extension=xdebug.so
xdebug.mode=debug,profile
xdebug.output_dir=/tmp/xdebug
xdebug.profiler_output_name=cachegrind.out.%t
```

### Blackfire

```bash
blackfire run php script.php
```

### Встроенные функции

```php
// Время выполнения
$start = microtime(true);
// код
echo microtime(true) - $start;

// Память
echo memory_get_usage();        // Текущее использование
echo memory_get_peak_usage();   // Пиковое использование
```

---

## Ключевые вопросы для интервью

- Объясните цикл выполнения PHP скрипта от получения запроса до ответа
- Как работает OpCache и почему он ускоряет PHP?
- Что такое JIT и когда он помогает?
- Как PHP управляет памятью? Что такое reference counting?
- Почему циклические ссылки - проблема и как их решает GC?
- Что такое Copy-on-Write?
- Как устроен массив в PHP на низком уровне?
- Чем Packed Array отличается от Hash Array?
- Что такое Interned Strings?
- Как работает autoloading в PHP?
