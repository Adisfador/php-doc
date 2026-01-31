# Типизация в PHP

Детальный разбор системы типов PHP: strict_types, type hints, union/intersection types, covariance/contravariance.

---

## 🎯 Эволюция типизации в PHP

### История

**PHP 5.0 (2004)** - Type hints для классов:
```php
function process(MyClass $obj) { }
```

**PHP 5.1** - array type hint:
```php
function process(array $data) { }
```

**PHP 7.0 (2015)** - Scalar type hints + return types:
```php
function add(int $a, int $b): int {
    return $a + $b;
}
```

**PHP 7.1** - nullable types, void:
```php
function find(?int $id): ?User { }
function log(string $message): void { }
```

**PHP 7.4** - Typed properties:
```php
class User {
    public int $id;
    public string $name;
}
```

**PHP 8.0** - Union types, mixed:
```php
function process(int|string $value): int|float { }
```

**PHP 8.1** - Intersection types, never:
```php
function handle(Countable&Traversable $collection): never { }
```

**PHP 8.2** - Disjunctive Normal Form (DNF) types:
```php
function process((A&B)|C $value) { }
```

---

## 📝 strict_types Directive

### Режимы типизации

**Coercive (слабая) типизация** - по умолчанию:
```php
<?php
// БЕЗ declare(strict_types=1)

function add(int $a, int $b): int {
    return $a + $b;
}

echo add(5, 10);      // 15 ✅
echo add("5", "10");  // 15 ✅ (строки приводятся к int)
echo add(5.9, 10.1);  // 16 ✅ (float приводятся к int)
echo add("5x", 10);   // TypeError в PHP 8.0+ (раньше Warning + приведение)
```

**Strict (строгая) типизация:**
```php
<?php
declare(strict_types=1);  // ПЕРВАЯ строка после <?php

function add(int $a, int $b): int {
    return $a + $b;
}

echo add(5, 10);      // 15 ✅
echo add("5", "10");  // TypeError: Argument #1 must be of type int, string given
echo add(5.9, 10.1);  // TypeError: Argument #1 must be of type int, float given
```

### Важные нюансы strict_types

**1. Per-file, не глобально:**
```php
// file1.php
<?php
declare(strict_types=1);

function strict_add(int $a, int $b): int {
    return $a + $b;
}

// file2.php
<?php
// НЕТ declare(strict_types=1)

require 'file1.php';

strict_add("5", 10);  // TypeError! (проверка в месте вызова)
```

**2. Влияет только на вызов функции, не на объявление:**
```php
// library.php
<?php
// НЕТ strict_types

function add(int $a, int $b): int {
    return $a + $b;
}

// app.php
<?php
declare(strict_types=1);

require 'library.php';

add("5", 10);  // TypeError (strict в app.php)
```

**3. НЕ влияет на внутренние функции PHP:**
```php
<?php
declare(strict_types=1);

strlen(123);  // "3" - int приводится к string (внутренняя функция PHP)
```

**4. Рекомендация: ВСЕГДА используй strict_types=1**
```php
<?php
declare(strict_types=1);

// Все файлы в Laravel/Symfony проектах должны начинаться так
```

---

## 🔤 Scalar Types

### int - целое число

```php
<?php
declare(strict_types=1);

function factorial(int $n): int {
    if ($n <= 1) return 1;
    return $n * factorial($n - 1);
}

factorial(5);    // 120 ✅
factorial(-3);   // -6 ✅ (отрицательные разрешены)
factorial(5.0);  // TypeError (float не int)

// Range: -2147483648 to 2147483647 (32-bit)
//        -9223372036854775808 to 9223372036854775807 (64-bit)
```

### float - число с плавающей точкой

```php
function calculatePrice(float $price, float $tax): float {
    return $price * (1 + $tax);
}

calculatePrice(100.0, 0.2);  // 120.0 ✅
calculatePrice(100, 0.2);    // TypeError (int не float в strict mode)

// ⚠️ Осторожно с float сравнением!
0.1 + 0.2 === 0.3;  // false! (0.30000000000000004)

// Правильное сравнение:
abs((0.1 + 0.2) - 0.3) < PHP_FLOAT_EPSILON;  // true
```

### string - строка

```php
function greet(string $name): string {
    return "Hello, {$name}!";
}

greet("John");  // "Hello, John!" ✅
greet("");      // "Hello, !" ✅ (пустая строка разрешена)
greet(null);    // TypeError

// Для деньги ИСПОЛЬЗУЙ string, НЕ float!
class Money {
    public function __construct(
        private string $amount,  // "123.45"
        private string $currency // "USD"
    ) {}
}
```

### bool - булево значение

```php
function isActive(bool $active): bool {
    return $active;
}

isActive(true);   // true ✅
isActive(false);  // false ✅
isActive(1);      // TypeError (int не bool)
isActive(null);   // TypeError

// Без strict_types:
// isActive(1);    // true (приведение)
// isActive(0);    // false
// isActive("x");  // true
```

---

## 🎨 Compound Types

### array - массив

```php
function sum(array $numbers): int {
    return array_sum($numbers);
}

sum([1, 2, 3]);           // 6 ✅
sum([]);                  // 0 ✅ (пустой массив разрешен)
sum(['a' => 1, 'b' => 2]); // 3 ✅ (ассоциативный массив)
sum(null);                // TypeError

// ⚠️ Нельзя указать тип элементов массива!
function processInts(array $ints): void {
    // Нет гарантии что $ints содержит только int
}

// Решение: PHPDoc + static analysis (Psalm, PHPStan)
/**
 * @param array<int> $ints
 */
function processInts(array $ints): void {
    foreach ($ints as $int) {
        // Psalm проверит что $int это int
    }
}
```

### object - любой объект

```php
function process(object $obj): void {
    echo $obj::class;
}

process(new stdClass());  // "stdClass" ✅
process(new User());      // "User" ✅
process([]);              // TypeError (array не object)
```

### callable - вызываемая функция

```php
function filter(array $items, callable $callback): array {
    return array_filter($items, $callback);
}

// Разные виды callable:
filter([1, 2, 3], fn($x) => $x > 1);        // closure ✅
filter([1, 2, 3], 'is_int');                // string function name ✅
filter([1, 2, 3], [User::class, 'method']); // static method ✅
filter([1, 2, 3], [$obj, 'method']);        // instance method ✅

// ⚠️ callable НЕ ПРОВЕРЯЕТ сигнатуру!
// Решение: Closure type hint (PHP 7.0+)
function filter(array $items, Closure $callback): array {
    return array_filter($items, $callback);
}
```

### iterable - массив или Traversable

```php
function iterate(iterable $items): void {
    foreach ($items as $item) {
        echo $item;
    }
}

iterate([1, 2, 3]);              // ✅ array
iterate(new ArrayIterator([1, 2, 3])); // ✅ Traversable
iterate("string");               // TypeError

// Return type
function getItems(): iterable {
    return [1, 2, 3];  // ✅ array
    // или
    yield 1;           // ✅ Generator (implements Traversable)
}
```

---

## 🔷 Special Types

### mixed - любой тип (PHP 8.0+)

```php
function process(mixed $value): mixed {
    return $value;
}

process(123);        // ✅
process("string");   // ✅
process([]);         // ✅
process(null);       // ✅
process(new User()); // ✅

// mixed = int|float|string|bool|array|object|resource|null
// Эквивалентно отсутствию type hint в PHP < 8.0
```

### void - нет возвращаемого значения (PHP 7.1+)

```php
function log(string $message): void {
    file_put_contents('log.txt', $message, FILE_APPEND);
    // НЕТ return или return без значения
}

log("test");  // ✅

function invalid(): void {
    return null;  // TypeError! (даже null запрещен)
}

// ⚠️ void ТОЛЬКО для return type, НЕ для параметров
function test(void $x): void { }  // Syntax error
```

### never - никогда не возвращает (PHP 8.1+)

```php
function redirect(string $url): never {
    header("Location: {$url}");
    exit;  // всегда завершает выполнение
}

function fail(string $message): never {
    throw new Exception($message);  // всегда бросает исключение
}

function infiniteLoop(): never {
    while (true) {
        // бесконечный цикл
    }
}

// never СТРОЖЕ чем void:
// - void может return; (без значения)
// - never НИКОГДА не возвращает управление

// Использование:
function process(mixed $value): string {
    if (!is_string($value)) {
        fail("Value must be string");  // never = код после не выполнится
    }
    
    // Здесь $value точно string (type narrowing)
    return strtoupper($value);
}
```

### null - только null (PHP 8.2+)

```php
function alwaysNull(): null {
    return null;  // единственное валидное значение
}

// Редко используется, в основном для переопределения методов
class Parent {
    public function get(): mixed {
        return "value";
    }
}

class Child extends Parent {
    public function get(): null {  // сужение типа
        return null;
    }
}
```

---

## ❓ Nullable Types

### Синтаксис ?Type (PHP 7.1+)

```php
function find(int $id): ?User {
    $user = DB::find($id);
    return $user ?: null;  // может вернуть User или null
}

$user = find(1);  // User|null
if ($user !== null) {
    echo $user->name;
}

// Параметры тоже могут быть nullable
function greet(?string $name = null): string {
    return $name !== null ? "Hello, {$name}" : "Hello, Guest";
}

greet("John");  // "Hello, John" ✅
greet(null);    // "Hello, Guest" ✅
greet();        // "Hello, Guest" ✅ (default)
```

### Nullable vs Default null

```php
// ⚠️ Разница между nullable и default null
function test1(?string $name): void { }
test1();  // TypeError: Too few arguments

function test2(?string $name = null): void { }
test2();  // ✅ (default = null)

// Рекомендация: если nullable, добавляй default = null
function create(?string $name = null, ?int $age = null): User {
    return new User($name ?? 'Guest', $age ?? 18);
}

create();              // ✅
create("John");        // ✅
create("John", 30);    // ✅
create(null, null);    // ✅
```

---

## 🔀 Union Types (PHP 8.0+)

### Синтаксис Type1|Type2

```php
function process(int|string $value): int|float {
    if (is_int($value)) {
        return $value * 2;
    }
    
    return (float) $value * 2;
}

process(10);     // 20 (int) ✅
process("10");   // 20.0 (float) ✅
process(10.5);   // TypeError (float не в union)

// Множественные типы
function format(int|float|string|null $value): string {
    if ($value === null) {
        return 'N/A';
    }
    
    return (string) $value;
}

// Union с классами
function save(User|Admin|Guest $entity): void {
    $entity->save();
}
```

### Union с nullable (?Type = Type|null)

```php
// Эквивалентны:
function test1(?string $name): void { }
function test2(string|null $name): void { }

// НО рекомендуется ?Type (короче)
function find(int $id): ?User { }  // ✅ предпочтительнее
function find(int $id): User|null { }  // ⚠️ работает, но длиннее
```

### Ограничения Union Types

```php
// ⚠️ Нельзя void|null, never|int и т.д.
function test(): void|null { }  // Syntax error

// ⚠️ Нельзя дублировать типы
function test(int|int $x): void { }  // Syntax error

// ⚠️ Нельзя смешивать ?Type и |null
function test(?int|string $x): void { }  // Syntax error
// Правильно:
function test(int|string|null $x): void { }
```

---

## ⚡ Intersection Types (PHP 8.1+)

### Синтаксис Type1&Type2

```php
interface Loggable {
    public function log(): void;
}

interface Cacheable {
    public function cache(): void;
}

// Объект ДОЛЖЕН реализовывать ОБА интерфейса
function process(Loggable&Cacheable $obj): void {
    $obj->log();
    $obj->cache();
}

class User implements Loggable, Cacheable {
    public function log(): void { }
    public function cache(): void { }
}

process(new User());  // ✅

// ⚠️ Intersection ТОЛЬКО для классов/интерфейсов, НЕ для scalar
function test(int&string $x): void { }  // Syntax error (невозможно)
```

### Traversable&Countable - частый пример

```php
function iterate(Traversable&Countable $collection): void {
    echo "Count: " . count($collection) . "\n";
    
    foreach ($collection as $item) {
        echo $item;
    }
}

$array = new ArrayObject([1, 2, 3]);  // implements Traversable, Countable
iterate($array);  // ✅

iterate([1, 2, 3]);  // TypeError (array не Traversable&Countable, только Countable)
```

---

## 🎭 DNF Types (PHP 8.2+)

### Disjunctive Normal Form - (A&B)|C

```php
// (Loggable&Cacheable)|string - объект с обоими интерфейсами ИЛИ строка
function process((Loggable&Cacheable)|string $value): void {
    if (is_string($value)) {
        echo $value;
    } else {
        $value->log();
        $value->cache();
    }
}

class User implements Loggable, Cacheable { }

process(new User());  // ✅
process("text");      // ✅
process(new stdClass());  // TypeError

// Сложные DNF типы
function handle((A&B)|(C&D)|E $value): void { }

// ⚠️ Нельзя (A|B)&C - только (A&B)|C форма
```

---

## 📦 Typed Properties (PHP 7.4+)

### Объявление типов свойств

```php
class User {
    // Typed properties (PHP 7.4+)
    public int $id;
    public string $name;
    public ?string $email = null;
    public array $roles = [];
    
    // Union types в свойствах (PHP 8.0+)
    public int|string $identifier;
    
    // Readonly properties (PHP 8.1+)
    public readonly int $id;
    
    // Readonly + constructor promotion (PHP 8.0+)
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public ?string $email = null,
    ) {}
}

$user = new User(1, "John");
$user->id = 2;  // Error: Cannot modify readonly property (PHP 8.1+)
```

### Uninitialized Properties

```php
class User {
    public int $id;  // БЕЗ значения по умолчанию
}

$user = new User();
echo $user->id;  // Error: Typed property must not be accessed before initialization

// Инициализация обязательна:
class User {
    public int $id;
    
    public function __construct() {
        $this->id = 1;  // инициализация
    }
}

// Или default value:
class User {
    public int $id = 0;  // с дефолтом
}
```

### Nullable Properties

```php
class User {
    public ?string $email;  // nullable БЕЗ default = UNINITIALIZED
}

$user = new User();
echo $user->email;  // Error: must not be accessed before initialization

// Решение 1: default = null
class User {
    public ?string $email = null;
}

$user = new User();
echo $user->email;  // null ✅

// Решение 2: инициализация в конструкторе
class User {
    public ?string $email;
    
    public function __construct(?string $email = null) {
        $this->email = $email;
    }
}
```

---

## 🔄 Type Coercion (приведение типов)

### Явное приведение (Casting)

```php
$int = (int) "123";        // 123
$int = (int) "123.45";     // 123 (обрезает дробную часть)
$int = (int) "123abc";     // 123 (парсит до первого нечислового символа)
$int = (int) "abc";        // 0
$int = (int) true;         // 1
$int = (int) false;        // 0

$float = (float) "123.45"; // 123.45
$float = (float) "123";    // 123.0

$string = (string) 123;    // "123"
$string = (string) true;   // "1"
$string = (string) false;  // "" (пустая строка!)
$string = (string) null;   // "" (пустая строка)

$bool = (bool) 1;          // true
$bool = (bool) 0;          // false
$bool = (bool) "";         // false
$bool = (bool) "0";        // false
$bool = (bool) "false";    // true! (непустая строка)
$bool = (bool) [];         // false
$bool = (bool) [1];        // true

$array = (array) "string"; // ["string"]
$array = (array) null;     // []
$array = (array) 123;      // [123]

$object = (object) ["a" => 1]; // stdClass {a: 1}
```

### Функции приведения

```php
// intval(), floatval(), strval(), boolval()
intval("123");      // 123
intval("123.45");   // 123
floatval("123.45"); // 123.45
strval(123);        // "123"
boolval(1);         // true

// settype() - изменяет переменную
$var = "123";
settype($var, "int");
echo $var;  // 123 (int)
```

---

## 🔍 Type Checking

### Функции проверки типов

```php
is_int($var);      // int
is_float($var);    // float (is_double - алиас)
is_string($var);   // string
is_bool($var);     // bool
is_array($var);    // array
is_object($var);   // object
is_null($var);     // null
is_resource($var); // resource
is_callable($var); // callable
is_iterable($var); // array|Traversable
is_countable($var);// array|Countable (PHP 7.3+)

is_numeric($var);  // число или числовая строка ("123")
is_scalar($var);   // int|float|string|bool

// Проверка класса
$user instanceof User;          // true если $user это User или наследник
is_a($user, User::class);       // аналогично instanceof
is_subclass_of($user, User::class); // true только для наследников
```

### get_debug_type() - PHP 8.0+

```php
// Лучше чем gettype() (более информативно)
echo get_debug_type(123);        // "int"
echo get_debug_type("test");     // "string"
echo get_debug_type(new User()); // "User" (имя класса)
echo get_debug_type(null);       // "null"
echo get_debug_type([]);         // "array"
echo get_debug_type(true);       // "bool"

// gettype() (старый, менее информативный)
echo gettype(123);        // "integer" (не "int")
echo gettype(true);       // "boolean" (не "bool")
echo gettype(new User()); // "object" (не имя класса)
```

---

## 🎯 Variance - Ковариантность и Контравариантность

### Covariance (ковариантность) return types - PHP 7.4+

**Правило:** Дочерний класс может вернуть **более специфичный тип**.

```php
class Animal {}
class Dog extends Animal {}

class AnimalFactory {
    public function create(): Animal {
        return new Animal();
    }
}

class DogFactory extends AnimalFactory {
    // Ковариантность: Dog более специфичен чем Animal ✅
    public function create(): Dog {
        return new Dog();
    }
}

// Liskov Substitution Principle (SOLID):
// DogFactory можно использовать везде где AnimalFactory
function test(AnimalFactory $factory): Animal {
    return $factory->create();  // всегда Animal или наследник
}

test(new DogFactory());  // ✅
```

**До PHP 7.4 - covariance запрещена:**
```php
class DogFactory extends AnimalFactory {
    public function create(): Dog {  // Fatal error в PHP < 7.4
        return new Dog();
    }
}
```

### Contravariance (контравариантность) parameter types - PHP 7.4+

**Правило:** Дочерний класс может принимать **менее специфичный тип**.

```php
class Dog {}
class Puppy extends Dog {}

class DogFeeder {
    public function feed(Dog $dog): void {
        // кормим Dog
    }
}

class PuppyFeeder extends DogFeeder {
    // Контравариантность: можем принять любую Dog (не только Puppy) ✅
    public function feed(Dog $dog): void {  // можно даже убрать тип совсем
        // кормим Dog или Puppy
    }
}

// ⚠️ НЕЛЬЗЯ сузить тип параметра:
class WrongPuppyFeeder extends DogFeeder {
    public function feed(Puppy $puppy): void {  // Fatal error!
        // сужение типа запрещено
    }
}
```

**Почему contravariance безопасна:**
```php
function test(PuppyFeeder $feeder, Dog $dog): void {
    $feeder->feed($dog);  // передаем Dog
}

// Если бы WrongPuppyFeeder разрешили (feed(Puppy)):
test(new WrongPuppyFeeder(), new Dog());  
// ОШИБКА: передали Dog, а метод ожидает Puppy!

// С контравариантностью (feed(Dog)):
test(new PuppyFeeder(), new Dog());  // ✅ Dog подходит
```

---

## 🛡️ Static Analysis Tools

### Psalm

```php
// composer require --dev vimeo/psalm

/** @var array<int, string> */
$users = ["Alice", "Bob"];

/** @param array<User> $users */
function process(array $users): void {
    foreach ($users as $user) {
        // Psalm знает что $user это User
        echo $user->name;
    }
}

// Проверить типы:
// vendor/bin/psalm

// Psalm levels (1-8, 1 самый строгий):
// psalm.xml
<psalm errorLevel="1">
```

### PHPStan

```php
// composer require --dev phpstan/phpstan

/** @param array<int, User> $users */
function process(array $users): void {
    foreach ($users as $user) {
        // PHPStan проверит что $user это User
        echo $user->name;
    }
}

// Проверить:
// vendor/bin/phpstan analyse src

// PHPStan levels (0-9, 9 самый строгий):
// phpstan.neon
level: 9
```

### Generics (через PHPDoc)

```php
/**
 * @template T
 * @param class-string<T> $class
 * @return T
 */
function create(string $class): object {
    return new $class();
}

$user = create(User::class);  // PHPStan/Psalm знают что $user это User
echo $user->name;  // ✅ autocomplete работает

// Коллекции с дженериками:
/**
 * @template T
 */
class Collection {
    /** @var array<T> */
    private array $items = [];
    
    /** @param T $item */
    public function add($item): void {
        $this->items[] = $item;
    }
    
    /** @return array<T> */
    public function all(): array {
        return $this->items;
    }
}

/** @var Collection<User> */
$users = new Collection();
$users->add(new User());  // ✅
$users->add(new Post());  // ❌ PHPStan/Psalm error
```

---

## 🎓 Best Practices для типизации

### 1. Всегда используй strict_types

```php
<?php
declare(strict_types=1);

// КАЖДЫЙ файл должен начинаться так
```

### 2. Всегда указывай типы (параметры + return)

```php
// ❌ Плохо (без типов)
function calculate($a, $b) {
    return $a + $b;
}

// ✅ Хорошо
function calculate(int $a, int $b): int {
    return $a + $b;
}
```

### 3. Используй nullable + default для опциональных

```php
// ❌ Плохо
function greet(?string $name): string {
    return $name ? "Hello, {$name}" : "Hello";
}
greet();  // Error: Too few arguments

// ✅ Хорошо
function greet(?string $name = null): string {
    return $name ? "Hello, {$name}" : "Hello";
}
greet();  // ✅
```

### 4. Предпочитай специфичные типы

```php
// ❌ Плохо (слишком широкий тип)
function process(mixed $value): mixed { }

// ✅ Хорошо (точный тип)
function process(User $user): int { }

// ⚠️ Union только если реально нужны несколько типов
function process(int|string $value): void { }
```

### 5. Используй readonly для иммутабельности (PHP 8.1+)

```php
class User {
    public function __construct(
        public readonly int $id,
        public readonly string $name,
    ) {}
}

$user = new User(1, "John");
$user->id = 2;  // Error!
```

### 6. PHPDoc для массивов и дженериков

```php
/**
 * @param array<int, User> $users
 * @return array<string, int>
 */
function processUsers(array $users): array {
    $result = [];
    foreach ($users as $user) {
        $result[$user->name] = $user->id;
    }
    return $result;
}
```

### 7. Static analysis в CI/CD

```yaml
# .github/workflows/ci.yml
- name: PHPStan
  run: vendor/bin/phpstan analyse

- name: Psalm
  run: vendor/bin/psalm
```

---

## 🔥 Частые ошибки и решения

### 1. Забыли strict_types

```php
<?php
// БЕЗ declare(strict_types=1)

function add(int $a, int $b): int {
    return $a + $b;
}

add("5", "10");  // 15 (строки приводятся, БАГ не обнаружен)

// ✅ Решение: всегда добавляй
declare(strict_types=1);
```

### 2. Nullable без default

```php
function find(?int $id): void { }
find();  // TypeError

// ✅ Решение:
function find(?int $id = null): void { }
```

### 3. Использование array без PHPDoc

```php
function process(array $users): void {
    foreach ($users as $user) {
        echo $user->name;  // IDE не знает что $user это User
    }
}

// ✅ Решение:
/** @param array<User> $users */
function process(array $users): void { }
```

### 4. Return type не соответствует реальности

```php
function find(int $id): User {
    return DB::find($id);  // может вернуть null!
}

// ✅ Решение:
function find(int $id): ?User {
    return DB::find($id);
}
```

---

## 📊 Сравнение типов в разных версиях PHP

| Feature | PHP 5 | PHP 7.0 | PHP 7.1 | PHP 7.4 | PHP 8.0 | PHP 8.1 | PHP 8.2 |
|---------|-------|---------|---------|---------|---------|---------|---------|
| Class type hints | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| array type hint | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Scalar types | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Return types | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Nullable (?Type) | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| void | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Typed properties | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Union types | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| mixed | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Intersection types | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| never | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| readonly | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| DNF types | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## 🎓 Для собеседования: ключевые точки

1. **strict_types=1** - объяснить разницу с coercive mode
2. **Scalar types** - int, float, string, bool (с PHP 7.0)
3. **Nullable types** - ?Type = Type|null (с PHP 7.1)
4. **Union types** - int|string (с PHP 8.0)
5. **Intersection types** - Countable&Traversable (с PHP 8.1)
6. **never vs void** - never никогда не возвращает (exit/throw)
7. **Covariance/Contravariance** - return сужается, параметры расширяются
8. **Typed properties** - PHP 7.4+, readonly в PHP 8.1+
9. **Static analysis** - Psalm, PHPStan для массивов и дженериков
10. **Best practice** - всегда strict_types, типы везде, PHPDoc для массивов
